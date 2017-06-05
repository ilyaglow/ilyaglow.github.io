---
layout: post
title: Creating a Post
---

To make your first blog post, just create a new markdown file in the `_posts` directory with the title format `YYYY-MM-DD-title.md`. Posts follow the simple [markdown syntax](https://github.com/adam-p/markdown-here/wiki/Markdown-Cheatsheet) and can include a wide array of multimedia formats.

# This is a header

And this is a simple paragraph.

## Smaller header

This is how you **bold** & *italicize* text.

Here is some `inline code`.

```golang
var hostKey ssh.PublicKey
// An SSH client is represented with a ClientConn.
//
// To authenticate with the remote server you must pass at least one aaaaaaaaaaaaaa
// implementation of AuthMethod via the Auth field in ClientConfig,
// and provide a HostKeyCallback.
config := &ssh.ClientConfig{
    User: "username",
    Auth: []ssh.AuthMethod{
        ssh.Password("yourpassword"),
    },
    HostKeyCallback: ssh.FixedHostKey(hostKey),
}
client, err := ssh.Dial("tcp", "yourserver.com:22", config)
if err != nil {
    log.Fatal("Failed to dial: ", err)
}

// Each ClientConn can support multiple interactive sessions,
// represented by a Session.
session, err := client.NewSession()
if err != nil {
    log.Fatal("Failed to create session: ", err)
}
defer session.Close()

// Once a Session is created, you can execute a single command on
// the remote side using the Run method.
var b bytes.Buffer
session.Stdout = &b
if err := session.Run("/usr/bin/whoami"); err != nil {
    log.Fatal("Failed to run: " + err.Error())
}
fmt.Println(b.String())
```

# Another header

And here's a list:

* Item 1
* Item 2
* Item 3

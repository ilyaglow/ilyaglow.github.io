---
layout: post
title: Rapid analyzing Sonar HTTP datasets
categories: threatintel
tags: [threatintel, recon, sonar, scans]
---

Sometimes you need to get threat intelligence data as fast as possible and Rapid7's Project Sonar Opendata can give great inside here.

The only problem is that you can't just grep the http response body with lovely [jq](https://stedolan.github.io/jq/) easily, because the data field of the resulting JSON is base64 encoded:
```json
{
  "data": "SFRUUC8xLjAgNDAwIEJhZCBSZXF1ZXN0DQpTZXJ2ZXI6IEFrYW1haUdIb3N0DQpNaW1lLVZlcnNpb246IDEuMA0KQ29udGVudC1UeXBlOiB0ZXh0L2h0bWwNCkNvbnRlbnQtTGVuZ3RoOiAyMDgNCkV4cGlyZXM6IE1vbiwgMjMgQXByIDIwMTggMDc6NDA6MjkgR01UDQpEYXRlOiBNb24sIDIzIEFwciAyMDE4IDA3OjQwOjI5IEdNVA0KQ29ubmVjdGlvbjogY2xvc2UNCg0KPEhUTUw+PEhFQUQ+CjxUSVRMRT5JbnZhbGlkIFVSTDwvVElUTEU+CjwvSEVBRD48Qk9EWT4KPEgxPkludmFsaWQgVVJMPC9IMT4KVGhlIHJlcXVlc3RlZCBVUkwgIiYjOTE7bm8mIzMyO1VSTCYjOTM7IiwgaXMgaW52YWxpZC48cD4KUmVmZXJlbmNlJiMzMjsmIzM1OzkmIzQ2OzFmMzEzMjE3JiM0NjsxNTI0NDY5MjI5JiM0NjsyNDFiOGFmCjwvQk9EWT48L0hUTUw+Cg==",
  "host": "REDACTED",
  "ip": "REDACTED",
  "path": "/",
  "port": 80,
  "vhost": "REDACTED"
}
```

Okay, probably you can grep it by using a decent bash script, but I think I have a better option :)

_Update_:

Apparently the latest `jq` version [has](https://github.com/stedolan/jq/issues/47) `@base64d` filter so there is definitely another way to do the same thing using only jq and it's nice to have multiple [options](https://hub.docker.com/r/ilyaglow/jq).

# My solution

To circumvent this issue the quick and dirty [sonargrep](https://github.com/ilyaglow/sonargrep) app was written. It accepts gzipped data from the stdin and tries to find host entries that hold the data you need.

You can install sonargrep with the following command (assuming Go is already installed):
```
go install github.com/ilyaglow/sonargrep
```

# Usage

Here is how to get a list of Wordpress-related IPs:
```
curl -L -s https://opendata.rapid7.com/sonar.https/2018-04-24-1524531601-https_get_443.json.gz \
    | sonargrep -w wordpress -i \
    | jq -r '.ip'
```

The example above achieves:
* Greps records that contain wordpress (case insensitive) in their http response body from `sonar.https` dataset.
* Extracts IPs using jq.
* All this without saving a 50GB file to the disk.

_Update_:
I've made a docker for the latest jq github version, so you can do the same task like this:
```
alias jq="sudo docker run -i --rm ilyaglow/jq"
curl -L -s https://opendata.rapid7.com/sonar.https/2018-04-24-1524531601-https_get_443.json.gz \
    | gunzip \
    | jq -r 'select((.data | @base64d) | match(".*wordpress.*", "i")) | .ip'
```

# Flow

Basically, here is how my opendata research flow with [sonargrep](https://github.com/ilyaglow/sonargrep) now looks like:

* Spin up a digital ocean droplet
* Start grepping
* Wait for an hour or something, while playing with the results that come on the fly
* Kill the droplet

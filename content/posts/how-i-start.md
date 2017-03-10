+++
date = "2017-03-10T00:32:25-05:00"
title = "How I Start: Writing Servers for Existing Clients"
+++

I find again and again that I end up creating a new server for clients which already exist. I have now gone through this process with multiple projects, including [mdm](https://github.com/micromdm/micromdm), [SCEP](github.com/micromdm/scep), osquery and others. Today, I want to take a moment and describe how I use Go to prototype a new server around a client that I'm not familiar with, or have no documentation for. 

I always end up throwing out this early prototype, but it's an important part of the process and deserves to be documented on it's own. <br />
A few days ago, I started a new project - [moroz](https://github.com/groob/moroz), a server for [Santa](https://github.com/google/santa), a popular macOS security tool. I'll use Santa as the client in this example.

<!--more-->

# Why Go? 
There are many tools out there, especially if you're dealiing with http - ngrok, nginx, mitmproxy, wireshark etc are all good examples of tools one might use to see what exactly a client is trying to send to a server. But sometimes, you just want full control over the process, so you need to write your own tool. Plus, the end goal is to write a server, and doing the exploratory work with the same programming language will help reveal what libraries you might need and give clues as to the architecture you'll end up with. <br />
Well, turns out [Go](https://github.com/golang/go), with it's powerful [`io` interfaces](https://medium.com/go-walkthrough/go-walkthrough-io-package-8ac5e95a9fbd#.78mymhfbb), and [rich standard library](https://golang.org/pkg/#stdlib) is a fantastic partner for exploring the unknown. 

# Configuring the client
  
The first step is to point the client at a URL or IP that you control, most likely something like `localhost:8080` on your PC. [ngrok](https://ngrok.com/) is a fantastic option here as it offers valid TLS and DNS that a client like an iOS device can connect to with no extra hassle. 

For santa, I configured the server URL to `https://santa:8080/`:
```
sudo defaults write /var/db/santa/config.plist SyncBaseURL https://santa:8080/
```

I then added an entry to my `/etc/hosts` file, 

```
sudo echo "127.0.0.1 santa" >> /etc/hosts
```

and created a new certificate with the `CN=santa`. 

```
openssl genrsa -out server.key 2048
openssl rsa -in server.key -out server.key
openssl req -sha256 -new -key server.key -out server.csr -subj "/CN=santa"
openssl x509 -req -sha256 -days 365 -in server.csr -signkey server.key -out server.crt

# add certificate roots to client config 
sudo defaults write /var/db/santa/config.plist ServerAuthRootsFile $(pwd)/server.crt
```

Note: I could've used `localhost` as the CN, but that prevents other clients for connecting and usually is more trouble than it's worth. I frequently use [my own example here](https://gist.github.com/groob/01a40417e0176bd9ea8d473c8e381daa) to generate a full CA chain with roots and intermediaries. 


We're now ready to connect the client to a server:
```
sudo santactl sync
```

```
Missing Machine Owner.
HTTP Response: -1004 Could not connect to the server.
Preflight failed, aborting run
```

Ok, so we still need a server. Let's write one.

# Your firsts HTTP(s) server
Here's a hello world http example in Go, except that returning `hello world`, it will print the request to stdout. 
You can copy/paste the example and run it with `go run main.go` to get started with most projects.

```
package main

import (
	"fmt"
	"log"
	"net/http"
	"net/http/httputil"
)

// an http handler which will dump the request to stdout, showing
// the requests method, path and headers.
func dumpHandler(w http.ResponseWriter, r *http.Request) {
	// setting the second argument to true also dumps the request
	// body.
	dump, err := httputil.DumpRequest(r, true)
	if err != nil {
		log.Println(err)
		return
	}
	fmt.Println(string(dump))
}

// register the http handler, then start
// a https server on port 8080.
func main() {
	http.HandleFunc("/", dumpHandler)
	log.Fatal(http.ListenAndServeTLS(":8080",
		"server.crt", "server.key", nil))
}
```

Start the server with `go run main.go` and run the sync command again:

```
sudo santactl sync
Sync completed successfully
```

This time the client connects to the server and there's some output. The fact that the communication was succesful is partly because we're returning `200 OK` every time, and partly because santa isn't expecting a specific response. 
Often here the client will crash or return an error. We're just looking for _some_ communication to help figure out what to do next. 

```
POST /preflight/FA01680E-98CA-5557-8F59-7716ECFEE964 HTTP/2.0
Host: santa:8080
Accept: */*
Accept-Encoding: gzip, deflate
Accept-Language: en-us
Content-Encoding: zlib
Content-Length: 167
Content-Type: application/json
User-Agent: santactl-sync/0.9.16

x�U��
4�>��Y��2���
POST /ruledownload/FA01680E-98CA-5557-8F59-7716ECFEE964 HTTP/2.0
Host: santa:8080
Accept: */*
Accept-Encoding: gzip, deflate
Accept-Language: en-us
Content-Encoding: zlib
Content-Length: 10
Content-Type: application/json
User-Agent: santactl-sync/0.9.16

x���u�
POST /postflight/FA01680E-98CA-5557-8F59-7716ECFEE964 HTTP/2.0
Host: santa:8080
Accept: */*
Accept-Encoding: gzip, deflate
Accept-Language: en-us
Content-Length: 0
Content-Type: application/json
User-Agent: santactl-sync/0.9.16
```

Looks like we got three `POST` requests to three different endpoints. We now have a rough idea of what the `sync` command will do. 

The request body we get looks like junk, but the headers are helpful. 
```
Content-Encoding: zlib
Content-Type: application/json
```

The server is sending JSON, but it's encoded with zlib. The `encoding/zlib` package is part of the standard library, so this change should be easy. First, we'll set the  second argument in `DumpRequest` to `false`. We now want to retain the request body and handle it ourselves
```
out, err := httputil.DumpRequest(r, false)
```

Next, the request body, will need to be turned into a zlib reader, which will decode content when we print it. 

```
	zr, err := zlib.NewReader(r.Body)
```

And we'll need to print the reader to stdout:

```
io.Copy(os.Stdout, zr)
```

re-running `sync` now shows json output we can read
```
POST /preflight/FA01680E-98CA-5557-8F59-7716ECFEE964 HTTP/2.0
Host: santa:8080
Accept: */*
Accept-Encoding: gzip, deflate
Accept-Language: en-us
Content-Encoding: zlib
Content-Length: 167
Content-Type: application/json
User-Agent: santactl-sync/0.9.16


{"os_build":"16D32","santa_version":"0.9.16",
"hostname":"kl.groob.io","os_version":"10.12.3",
"certificate_rule_count":3,
"client_mode":"MONITOR",
"serial_num":"C02RY6G811ZL",
"binary_rule_count":5,"
primary_user":""}
```

Here's the modified `dumpHandler`:  
```
func dumpHandler(w http.ResponseWriter, r *http.Request) {
	dump, err := httputil.DumpRequest(r, false)
	if err != nil {
		log.Println(err)
		return
	}
	defer r.Body.Close()
	fmt.Println(string(dump))
	zr, err := zlib.NewReader(r.Body)
	if err != nil {
		log.Println(err)
		return
	}
	defer zr.Close()
	io.Copy(os.Stdout, zr)
}
```

*Note*: instead of Copying to `stdout`, we could've also copied to a file, a buffer or another network stream. The [io](https://golang.org/pkg/io/) and [ioutil](https://golang.org/pkg/io/ioutil/) packages offer a lot of options for working with input/output. 

# Responding to the client

Now that the client communicated with the server, we can figure out what to send back as a response. 
This part might be tricky depending on what information you have available. For SCEP I read the [RFC](https://tools.ietf.org/html/draft-gutmann-scep-05) and spend many hours figuring out how to encode a valid payload that would not crash my mac. Santa has good test coverage, so I was able to [find what I needed](https://github.com/google/santa/blob/a2a660d483d5bcd27c3c5ee9e581657af60f9400/Tests/LogicTests/Resources/sync_ruledownload_batch1.json) in a  few minutes. 

We can use the `w http.ResponseWriter`, the first parameter of the http handler to write back a respoinse to the client. 

For example:
```
w.Write([]byte("hello world"))
```

or with JSON:

```
w.Write([]byte(`{"msg" : "hello world" }`))
```

Santa is a project that can blacklist/whitelist binary execution, so that's what we'll try to do. I wrote a new handler, which returns a hardcoded rule JSON. I used the sha256 for Firefox:

```
func ruleDownload(w http.ResponseWriter, r *http.Request) {
	rules := []byte(`{"rules": 
	[{"rule_type": "BINARY", "policy": "BLACKLIST", 
	"sha256": "2dc104631939b4bdf5d6bccab76e166e37fe5e1605340cf68dab919df58b8eda", 
	"custom_msg": "hi there"}]}`)
	w.Write(rules)
}
```

The handler also needs to be registered in `func main` along with the dump handler.
```
	http.HandleFunc("/", dumpHandler)
	http.HandleFunc("/ruledownload/", ruleDownload)
```

Now `sync` gives a different response:

```
Uploaded 2 events
Added 1 rules
Sync completed successfully
```

And opening Firefox:

{{<figure src="/how-i-start-santa/firefox.png" title="" >}}

Success! 

This was a relatively simple API to explore. Not all of them will be as easy, and some might even require diving into tools like Hopper, gdb and `tcpdump` to see what the client is doing, but this is a start.
I [saved](https://gist.github.com/groob/4b1e4995b110bfd4af0e2f49397d7ea7) the full example of tinkering with santa.


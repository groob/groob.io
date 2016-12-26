+++
date = "2016-12-26T11:36:25-05:00"
title = "Accepting Github Webhooks with Go"
draft = false
tags = ["golang","tutorial", "dev"]

+++

Recently I had to write some automation scripts which ran whenever certain events occured in a Github repository. To do so, I wrote a custom HTTP server which accepted Github [Webhooks](https://developer.github.com/webhooks/) and triggered my script. Github has a simple guide using [sinatra](https://developer.github.com/webhooks/configuring/#writing-the-server), but I used the Go `net/http` library to write my server. This tutorial will show you how to build your own server to accept Github webhooks.

<!--more-->

# Introduction

Covered in this tutorial: 

- Using the `net/http` library to build a web server.
- Using the `encoding/json` to decode a JSON payload.
- Using the `github.com/google/go-github` to parse webhooks.

Assumptions:

- Go installed (see the [Installation Instructions](https://golang.org/doc/install)).
- Basic knowledge of Go (see the [Go Tour](https://tour.golang.org/welcome/1)).
- Knowledge of Go code organization/workspaces (see [How to Write Go Code](https://golang.org/doc/code.html#Organization)).

# Using ngrok

To deliver webhooks, Github needs a publicly reachable address or DNS name. We will use `ngrok` to temporarily expose the development server to the internet.

First, [download](https://ngrok.com/download) ngrok.
Then, expose port `8080` to the web. We will use port `8080` to run our Go webserver.

```
~/Downloads/ngrok http 8080
```

You'll see a new screen with a random `Forwarding` https address. Mine is `https://11f018ee.ngrok.io`, but yours will have a different subdomain. Copy this address, we will use it to configure the github webhook URL.

Keep the `ngrok` session running. If you restart it, you'll get a completely different URL, requiring you to reconfigure the Github settings.

# Configure Github to send webhooks

1. In a repo's settings(you must own the repository) page, click the "Webhook" section.
2. Click the `Add Webhook` button.
3. In the `Payload URL` field, type your `ngrok` https address, and add `/webhook` as the path.  
Example: `https://11f018ee.ngrok.io/webhook`  
For now, leave the `Secret` field blank. We will return to it later in the tutorial.  
Select the list of events you're interested in. In my case, I selected the *Watch* category, which will trigger a Webhook whenever someone stars the repository.

{{<figure src="/go-github-webhook/manage_webhook01.png" title="Add Repository Webhook" >}}

Now that both Github and `ngrok` are configured, your computer is all set to receive webhooks.
`ngrok` has a introspection feature which allows us to see all the requests we receive.  
Go to http://127.0.0.1:4040/inspect/http to view the data requests as they're sent by Github.
A feature that will prove useful during development will be the *Replay* button. 
You're unlikely to get everything right the first time, and here we can play the request as needed. 

{{<figure src="/go-github-webhook/ngrok_inspect01.png" title="Inspect ngrok traffic" >}}

# Writing the Web Server

We're finally ready to start coding. The Go `net/http` package allows us to create an HTTP Handler with a function that has the following two arguments:
```
func handler(w http.ResponseWriter, r *http.Request) {
 // your code here  
}
```

Inside the handler, we can access the `r *http.Request` properties like `r.URL.Path`, `r.Body` and so on.

Here's the full program:

```Go
package main

import (
	"fmt"
	"io"
	"log"
	"net/http"
	"os"
)

func handleWebhook(w http.ResponseWriter, r *http.Request) {
	fmt.Printf("headers: %v\n", r.Header)

	_, err := io.Copy(os.Stdout, r.Body)
	if err != nil {
		log.Println(err)
		return
	}
}

func main() {
    log.Println("server started")
	http.HandleFunc("/webhook", handleWebhook)
	log.Fatal(http.ListenAndServe(":8080", nil))
}
```

The `http.HandleFunc` line of the `main()` function tells the http library to handle http requests at the `/webhook` path using the `handleWebhook` function.

The `http.ListenAndServe` line starts a new webserver on `0.0.0.0:8080`. `http.ListenAndServe` can return an error, so we wrap it with `log.Fatal()` to capture and log the error if it happens.

Our simple `handleWebhook` function will print the request headers from [`*http.Request`](https://golang.org/pkg/net/http/#Request) and also copy the JSON content to stdout using the [`io.Copy`](https://golang.org/pkg/io/#Copy) method.

Go ahead and try running the server. Use `go run main.go` to start the server, then open the [inspect](http://127.0.0.1:4040/inspect/http) window in ngrok and replay one of your respnses. You should get a `200 OK` response, and the output on the terminal. 


# Decoding JSON using a map

The `encoding/json` package makes it very easy to encode/decode JSON objects into Go data types. 
Usually, I'd choose to decode a know JSON object into a struct, like 

```
type webhook struct {
	Action     string
	Repository struct {
		ID       string
		FullName string
	}
}
```

but for now, lets use a [map](https://tour.golang.org/moretypes/19). We can rewrite the `handleWebhook` function to decode the JSON.

```Go
func handleWebhook(w http.ResponseWriter, r *http.Request) {
	webhookData := make(map[string]interface{})
	err := json.NewDecoder(r.Body).Decode(&webhookData)
	if err != nil {
		http.Error(w, err.Error(), http.StatusInternalServerError)
		return
	}
	fmt.Println("got webhook payload: ")
	for k, v := range webhookData {
		fmt.Printf("%s : %v\n", k, v)
	}
}
```

```Go
	webhookData := make(map[string]interface{})
```

Here we create a new value, `webhookData`, which is a map of string keys to arbitrary data types.
Then, we create a JSON decoder from the HTTP request body and `Decode` it into the `webhookData` map.

```Go
	err := json.NewDecoder(r.Body).Decode(&webhookData)
	if err != nil {
		http.Error(w, err.Error(), http.StatusInternalServerError)
		return
    }
```

Notice that we handle any possible errors, as they [come up](https://blog.golang.org/error-handling-and-go).  In this particular case, we respond to the client with an HTTP error and `return` to stop execution.

# Introducing the go-github package

As you've likely noticed by now, the Github webhook JSON objects are quite large, and there's [quite a few](https://developer.github.com/v3/activity/events/types/) event types as well. Writing out each struct ourselves could be laborious. Luckily, there's an excellent library for the github API - [`github.com/google/go-github`](https://godoc.org/github.com/google/go-github/github).

We'll use the `ParseWebhook` function from the `github` package in our handler, instead of decoding the JSON ourselves.

First, install the Go package to your workspace.

```bash
go get -u github.com/google/go-github/github
```

Import the github package, and begin rewriting the handler. We need to read the payload into a `[]byte` buffer.
```Go
	payload, err := ioutil.ReadAll(r.Body)
	if err != nil {
		log.Printf("error reading request body: err=%s\n", err)
		return
	}
	defer r.Body.Close()

```

Next, we can parse the webhook into an event.

```Go
	event, err := github.ParseWebHook(github.WebHookType(r), payload)
	if err != nil {
		log.Printf("could not parse webhook: err=%s\n", err)
		return
	}

```

The event type is `interface{}`, so we'll need to use the switch statement to get the concrete type.
Inside the switch statement, we can handle each event type separately.

```Go
	switch e := event.(type) {
	case *github.PushEvent:
		// this is a commit push, do something with it
	case *github.PullRequestEvent:
		// this is a pull request, do something with it
	case *github.WatchEvent:
		// https://developer.github.com/v3/activity/events/types/#watchevent
		// someone starred our repository
		if e.Action != nil && *e.Action == "starred" {
			fmt.Printf("%s starred repository %s\n",
				*e.Sender.Login, *e.Repo.FullName)
		}
	default:
		log.Printf("unknown event type %s\n", github.WebHookType(r))
		return
	}
```

Note that because many of the fields in the `go-github` structs are pointers, you should check to make sure they're not `nil`, before accessing them, otherwise you'll get a `panic`. 

# Adding Webhook Security

If you go back to the Github webhook settings, and update the *Secret* field, Github will send that field as a header - `X-Hub-Signature`.
Github has some [recommendations](https://developer.github.com/webhooks/securing/#validating-payloads-from-github) for how you should validate the secret.

Here you can write your own helper, by getting the header with
```Go
payloadSecret := r.Header.Get("X-Hub-Signature")
```

Go-github already implements the recommended way of validating payloads. We can replace the original
`ioutil.ReadAll(r.Body)` line with

```Go
payload, err := github.ValidatePayload(r, []byte("my-secret-key"))
```

Putting it all together, we get an updated handler function.

```
func handleWebhook(w http.ResponseWriter, r *http.Request) {
	payload, err := github.ValidatePayload(r, []byte("my-secret-key"))
	if err != nil {
		log.Printf("error validating request body: err=%s\n", err)
		return
	}
	defer r.Body.Close()

	event, err := github.ParseWebHook(github.WebHookType(r), payload)
	if err != nil {
		log.Printf("could not parse webhook: err=%s\n", err)
		return
	}

	switch e := event.(type) {
	case *github.PushEvent:
		// this is a commit push, do something with it
	case *github.PullRequestEvent:
		// this is a pull request, do something with it
	case *github.WatchEvent:
		// https://developer.github.com/v3/activity/events/types/#watchevent
		// someone starred our repository
		if e.Action != nil && *e.Action == "starred" {
			fmt.Printf("%s starred repository %s\n",
				*e.Sender.Login, *e.Repo.FullName)
		}
	default:
		log.Printf("unknown event type %s\n", github.WebHookType(r))
		return
	}
}
```

# What next?

You should now have a simple webserver capable of accepting any Github webhook event. You're only limited by your imagination when it comes to choosing how you want to process these requests. For example, you could create a github bot which checks pull requests to see if the submitter signed a CLA, or deploys some code. 

Here's an [example gist](https://gist.github.com/groob/a70af7e7ed3567eb6b6801f65c7a1393) of something I wrote a few months ago to deploy each branch and pull request in a repository and send a slack notification on success.
At work, I used a similar webhook handler in a "repomonitor" service, which saves information about each commit, pull request and CI status update to Google Cloud Datastore and creates [Pubsub](https://cloud.google.com/pubsub/docs/) events. It has since become a core part of our deployment infrastructure.


# Additional Reading

- [JSON and Go](https://blog.golang.org/json-and-go) Additional info on encoding/decoding JSON.
- [Share Memory By Communicating](https://golang.org/doc/codewalk/sharemem/) (similar to the DeploymentMonitor in the gist above)

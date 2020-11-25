# Episode 2 - Containers

Date: November 25, 2020

Video URL: https://www.youtube.com/watch?v=AHDrejEv0SM

Web page for episode: TBD

Outline:

  - Why does Kubernetes use containers?
  - How do you build a container?
  - Build a simple Go app
  - Run the Go app directly
  - Run the Go app in a container
  - Push the Go app to an image registry

## Instructions for 'Hello Go' app

There is a very simple Go-based web app that responds to HTTP requests on port 8180 in [`cmd/hello/hello.go`](cmd/hello/hello.go).

After [installing Go](https://golang.org/doc/install), you can run the app directly with the command:

    go run cmd/hello/hello.go

Or you can build the Go command `hello` binary using:

    go build cmd/hello/hello.go

And then run it and monitor requests (access `localhost:8180/some-path-here` in a browser):

```
$ ./hello
2028/10/24 17:30:36 Starting webserver on :8180
2028/10/24 17:30:59 Received request for path: /some-path-here
```

After you're finished, you can remove the binary with `rm hello`.

## Build the 'Hello Go' Docker container image

TODO.

    docker build -t geerlingguy/kube101-go .
    docker run --rm -p 8180:8180 geerlingguy/kube101-go

TODO.

## Push the container image to a private Docker registry

TODO.

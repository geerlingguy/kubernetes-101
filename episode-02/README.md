# Episode 2 - Containers

Date: November 25, 2020

Video URL: https://www.youtube.com/watch?v=AHDrejEv0SM

Web page for episode: https://kube101.jeffgeerling.com/2020/episode-2-containers

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

Next up, we want to set up a container build environment that can build the Go application and then also run it (but without all the Go language cruft) in a trimmed down container image.

There is a `Dockerfile` in this directory containing a multi-stage Docker build layout which first builds the Go app using the official `golang` Docker image, then builds the final container based on Alpine Linux (using the official `alpine` Docker image).

To build the container, run:

    docker build -t geerlingguy/kube101-go .

Once the container is built, you can see it in your list of `docker images`, and you can run it with the command:

    docker run --rm -p 8180:8180 geerlingguy/kube101-go

## Push the container image to a private Docker registry

When you're satisfied the container image works correctly, go ahead and push it up to a Docker registry.

For my example, I'm pushing it to a private Docker Hub repository named `geerlingguy/kube101-go`:

    docker push geerlingguy/kube101-go

> Note: Pushing to a registry typically requires authentication. Please read the documentation for a guide on how to make sure you are authenticated to your Docker Hub (or other provider) account.
>
> Also, it's likely you won't be able to push to my namespace, so you might want to try using your own namespace instead of `geerlingguy` ;-)

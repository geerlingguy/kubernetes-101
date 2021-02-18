# Episode 1 - Hello, Kubernetes!

Date: November 18, 2020

Video URL: https://www.youtube.com/watch?v=IcslsH7OoYo

Web page for episode: https://kube101.jeffgeerling.com/2020/episode-1-hello-kubernetes

Outline:

  - Introduction to Kubernetes
  - Why Kubernetes? Why _not_ Kubernetes?
  - Installing Minikube
  - Running our first app on Kubernetes
  - Where to get help, and where to find community

## Instructions for Minikube

In the episode, Jeff showed how to use [Minikube](https://minikube.sigs.k8s.io/docs/) and [`kubectl`](https://kubernetes.io/docs/tasks/tools/install-kubectl/) to build your first local Kubernetes cluster:

  1. Install Minikube: `brew install minikube` (on a Mac with [Homebrew](https://brew.sh))
  2. Install kubectl: `brew install kubectl`
  3. Start a Minikube cluster: `minikube start`

Then you can check on the cluster's state, to make sure all the nodes—in this case, just one `master` node—are running and ready:

```
$ kubectl get nodes
NAME       STATUS   ROLES    AGE   VERSION
minikube   Ready    master   91s   v1.19.4
```

Now that we have a running cluster, it's time to deploy a lightweight application to it, just to make sure it's working:

```
$ kubectl create deployment hello-k8s --image=geerlingguy/kube101:intro
deployment.apps/hello-k8s created
```

It seems like it's deploying correctly, so the next step is to make it so we can access the deployed application from outside the cluster. By default, Kubernetes sets up an internal network, but does not expose any of your applications to the outside world. Here's how to 'expose' the deployment to the outside using a Kubernetes service:

```
$ kubectl expose deployment hello-k8s --type=NodePort --port=80
service/hello-k8s exposed
```

We'll get into what `NodePort` means later, but for now, we should be able to access the deployment from our computer. Minikube has a handy command that will open up the service in a web browser directly:

```
$ minikube service hello-k8s
<should launch your web browser>
```

But you could also find the IP address for the cluster using `minikube ip`, then pair that with the high-numbered port that is returned when you run `kubectl get services hello-k8s`.

When you're finished using the cluster, run `minikube halt` to stop it, or `minikube delete` to delete the cluster.

## Building the example Docker image

There is a `Dockerfile` in this directory, which is used by GitHub Actions to build the [`geerlingguy/kube101:intro` image on Docker Hub](https://hub.docker.com/repository/docker/geerlingguy/kube101).

That image is used in this episode of Kubernetes 101 to demonstrate a simple Kubernetes deployment.

If you want to build the image on your own, locally, you can run:

    docker build -t geerlingguy/kube101:intro .

And to run the image on its own, run:

    docker run -d -p 80:80 geerlingguy/kube101:intro

Once it's running, you can access the demo page:

    http://localhost

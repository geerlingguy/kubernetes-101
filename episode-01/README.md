# Episode 1 - Hello, Kubernetes!

Date: November 18, 2020

Video URL: https://www.youtube.com/watch?v=IcslsH7OoYo

Outline:

  - Introduction to Kubernetes
  - Why Kubernetes? Why _not_ Kubernetes?
  - Installing Minikube
  - Running our first app on Kubernetes
  - Where to get help, and where to find community.

## Building the example Docker image

Build the image:

    docker build -t geerlingguy/kube101:intro

Run the image:

    docker run -d -p 80:80 geerlingguy/kube101:intro

Access the demo:

    http://localhost

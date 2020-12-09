# Episode 4 - Real-world apps

Date: December 9, 2020

Video URL: https://www.youtube.com/watch?v=mrxA8g3w6ic

Outline:

  - Deploying Drupal on a traditional LAMP server
  - Deploying Drupal into Kubernetes using Helm
  - Deploying Drupal into a multi-node Kubernetes cluster
  - Scaling Drupal horizontally reveals some problems:
    - Persistent files
    - Persistent database
    - How do we configure DNS and TLS?

## Installing Drupal on a Traditional LAMP server

See the example and README inside the [`traditional-lamp-setup`](traditional-lamp-setup/) directory.

### Automating the Installation

I should note that I do not set up servers using the technique mentioned in the README above. Instead, I use tools like Ansible to automate every aspect of server provisioning and setup.

But as with traditional architecture, you should not dive into the deep end of Kubernetes without first understanding the basics. You shouldn't blindly run a 1,000 line Ansible playbook without understanding how file permissions, app configurations, and database connections work.

And you can't just skip over all that stuff when you're starting out in Kubernetes either!

## Installing Drupal on Kubernetes using Bitnami's Helm Chart

On that theme, there is a very popular packaging system for Kubernetes called Helm. Helm is a convenient and fairly common way to build templates for applications and deploy and update them in clusters.

But one common pitfall I see is developers new to Kubernetes picking up very good—but very complex—Helm charts, customizing them a little bit, and running their applications this way.

I do want to show you how to deploy Drupal using Helm (which I'll do shortly), but I am intentionally not going to go deep into Helm beyond the basics, because I would rather teach how the underlying components work together before I recommend using complex templates and magic tools to manage resources in Kubernetes!

### Install Helm

Install Helm; you can do this using `brew install helm` on a Mac, or follow the [Helm install instructions](https://helm.sh/docs/intro/install/) otherwise.

### Install the Drupal Chart

This guide assumes you have a Kubernetes cluster running somewhere (e.g. by running `minikube start`:

```
# Add Bitnami's Helm Chart repository.
helm repo add bitnami https://charts.bitnami.com/bitnami

# Install Drupal in your default namespace with the release name 'mysite'.
helm install mysite bitnami/drupal
```

Now, at this point, the Helm Chart will output a message saying you can monitor the Service it sets up for Drupal until a 'LoadBalancer' IP address is assigned, using the command:

```
kubectl get svc --namespace default -w mysite-drupal
```

That's great, but out of the box, Minikube, by virtue of its architecture, doesn't have a Load Balancing layer like Amazon ELBs or Linode's NodeBalancers built into it. Therefore you'll be waiting forever!

### Exposing a LoadBalancer in Minikube

So we'll use a little Minikube trick to enable a LoadBalancer, at least temporarily. Open a separate Terminal window and run the command:

```
minikube tunnel
```

This command will take a second to start, then ask for your account password to elevate privileges and create a new LoadBalancer on your machine for services like Drupal to use.

And if you run that command and watch the original terminal window, you'll notice it assigns an External IP once the LoadBalancer is running. Nice!

Now you can grab that IP address, paste it in a browser window, and access your Drupal site!

### Changing Chart Options

I won't go through every option available with Helm in this episode, but I will mention that well-maintained Charts like this Drupal chart have configurable options that allow you to control almost any aspect of what is deployed, including things like the Service type.

This Chart's documentation includes a [large selection of configurable Parameters](https://github.com/bitnami/charts/tree/master/bitnami/drupal/#installing-the-chart) which you can specify in a Helm values file.

So, if you wanted, you could change the release from using a LoadBalancer to a NodePort by setting `service.type` to `NodePort`.

### Cleaning Up

You can remove the Drupal site with:

```
helm uninstall mysite
```

## Drupal Directly in Kubernetes - Let's Do it [Mostly] Right!

Now let's work on deploying Drupal into Kubernetes _the hard way_.

Some of the magic the Helm chart covered up:

  1. Choosing or building and managing container images for Apache, MySQL, and PHP.
  2. Generating a Drupal codebase.
  3. Connecting all these services together correctly.

We are going to run through how to do everything step by step, and hopefully have a running Drupal installation by the end of this episode!

I'm going to switch things up and deploy to a real Kubernetes cluster, in this case a cluster running on Linode (since they are still offering a [free $100 credit using this link](https://linode.com/geerling)).

### Deploying the Drupal Kubernetes Manifests

Please see the [README inside the k8s-manifests](k8s-manifests/README.md) for further instructions.

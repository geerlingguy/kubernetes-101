# Episode 7 - Hello, Operator!

Date: January 20, 2021

Video URL: https://www.youtube.com/watch?v=Q7G6DBaIJ1c

Outline:

  - What are Kubernetes Operators?
  - Popular operators you may already know
  - How can we build our own operator?
  - Where can we find other operators?

## What are Operators?

As we've seen in the past few episodes, real-world applications like Drupal require a good deal of effort to deploy and maintain in a Kubernetes cluster.

There are many application concerns we haven't covered in depth, like running updates on the database schema, or performing routine operations outside of cron, like queue management or data backups.

Sometimes you can plug other tools into your Kubernetes cluster to solve some of these puzzles, but _what if there was a way to tell Kubernetes how to manage everything for you?_

Well there is, and that mechanism is an **Operator**.

> We're going to throw around a lot of Kubernetes jargon in the next few paragraphs! Don't worry if the connections between all these things is crystal clear yet. A few years into my own Kubernetes journey, I'm constantly referencing documentation to make sure I'm using the terms correctly!

### The Concept

Much like a human operator, a Kubernetes Operator is tasked with the management of a given application (or a whole bunch of instances of that application) inside a cluster.

The operator knows how to deploy the app, how to update it, and how to fix it if there are issues.

Digging down a couple layers into how Kubernetes works, Kubernetes uses **controllers** running in the cluster to watch the state of the cluster (watched resources), and make changes until the cluster is in the state it desires.

An analogy from the Kubernetes documentation makes a lot of sense here: a controller is like a thermostat. You set a temperature on the thermostat, then thermostat works to inform the heating system (the Kubernetes cluster) gets to the final state where that temperature is reached.

Operators are a standard way of building controllers for **Custom Resources**, extensions to the Kubernetes API beyond the standard resources like Pods, Deployments, and Services.

In the context of this series, we would probably consider building a **Custom Resource Definition** (CRD) named "HelloGo", or "Drupal", and then we could build an Operator to Custom Resources that conform to that CRD.

### The Execution

So taking Drupal as an example, an Operator would have a Custom Resource Definition.

This CRD defines the specification for individual Custom Resources (CRs) of Drupal, and at a minimum, you'd probably want to include data fields like:

  - `databaseEngine`
  - `filePvcType`

And anything else you might want to customize or tweak in an instance of Drupal running inside your Kubernetes cluster.

Then you could create one or more Custom Resources of `kind: Drupal`, and then the Operator would be called upon to create them if they don't exist, or manage them when any part of the instance changes.

As an example, if you set `databaseEngine: aws_aurora`, you could write some logic in your Operator that makes sure a database exists for the Drupal application in Amazon Aurora, then connects the Drupal site to that database.

Alternatively, if you set `databaseEngine: local_mariadb`, you could have the Operator ensure a MariaDB database exists inside the Kubernetes cluster, and then connect the Drupal site to _that_ database.

What if you set up your Drupal site with `aws_aurora`, then switched it to `local_mariadb`? Well, if you manage the site via an Operator, you could even build logic into that Operator to manage full database migrations!

The real power of managing applications like Drupal via Operator—especially if you manage more than one instance—is the ability to maintain the application deployment and maintenance logic in the Operator, and control instances with simple declarative YAML, just like other Kubernetes primatives.

For example, compare the following hypothetical Drupal Custom Resource to the longer examples used in episodes 4, 5, and 6:

```yaml
apiVersion: "drupal.example.com/v1"
kind: Drupal
metadata:
  name: my-drupal-site
spec:
  databaseEngine: local_mariadb
  version: 1.2.3
  filePvcType: efs
```

A lot simpler to manage! Now imagine you're building a platform that's running 300 different Drupal sites. Would you rather build the automation into one Operator that can manage 300 instances, or have to build automation that runs outside of Kubernetes to do the same thing?

### Why _not_ use an Operator?

There are some valid reasons for sticking with primatives, especially if you don't have more complex needs, like running many instances of an application in one or more clusters.

In many cases, you might only need to run one instance of an application or microservice, and it's not something that would benefit from an extra layer of automation.

Especially early on, when something is in development, it is easier to iterate using Kubernetes resources directly, rather than to build and maintain an Operator, which then manages those resources for you.

And in some cases, Operators are just one extra layer of automation that you might not want to maintain.

## Popular Kubernetes Operators

Even if you don't build your own operator, though, there's a good chance you'll end up using one or more in your Kubernetes clusters.

Almost every cluster I've ever built needed Prometheus for monitoring, and the standard way to install Prometheus and Alertmanager in a Kubernetes cluster is to use the [Prometheus Operator](https://github.com/prometheus-operator/prometheus-operator).

There are many other operators available, like the [Argo CD Operator](https://argocd-operator.readthedocs.io/en/latest/) for deploying one of the most popular Continuous Deployment tools in Kubernetes, or the [OpenEBS Operator](https://docs.openebs.io/docs/next/installation.html#installation-through-kubectl).

Operators aren't really centrally visible, like Docker images on [Docker Hub](https://hub.docker.com), or Helm charts on [Artifact Hub](https://artifacthub.io).

But there is one central location that's aggregating a large number of operators, and that's [OperatorHub.io](https://operatorhub.io).

Currently, OperatorHub lists almost 200 operators, but I know there are hundreds more that aren't listed there. Usually I find them through a direct Google or GitHub search.

Some are better maintained than others, though. The only way to get a good grasp on whether an existing operator is right for you is to install it in a test cluster.

## Build your own Operator

So what if there isn't a good operator for the software you're running in your cluster? Or what if your application needs a custom operator?

Early on, building an Operator required fairly deep knowledge of the Go programming language and Kubernetes APIs, but thanks to a lot of work by the community, there are now a variety of ways you can build operators—even if you don't know Go at all!

The two main ways I've seen used for building operators are [Kubebuilder](https://github.com/kubernetes-sigs/kubebuilder) and the [Operator SDK](https://sdk.operatorframework.io).

The main difference between them used to be that Kubebuilder only helped build Go-based operators, requiring knowledge of Go, whereas Operator SDK worked with Go, Ansible, or Helm-based operators.

But in the past year, the Operator SDK upstreamed some of it's customizations into Kubebuilder, so Go-based operators built by Kubebuilder and Operator SDK are practically identical.

Operator SDK is still the only easy way to build non-Go operators with Ansible or Helm, though.

There are some other projects that help with building operators, or operator-like tooling, like [KUDO](https://kudo.dev), but I won't cover them in this episode.

There are some tradeoffs if you're not using Go to build your operator with Operator SDK, though. The Operator SDK shows a graph of ['operator capability levels'](https://sdk.operatorframework.io/docs/overview/#operator-capability-level), and this shows different types of operators are better at different levels of automation.

Helm-based operators are great for basic app install and upgrades.

Ansible-based operators can also perform more app management, including metrics integrations, config management, and external integrations.

Go-based operators can do everything, with the highest amount of granularity and available performance tuning.

But Go-based operators are the most difficult to set up and maintain if you don't already know the Go programming language. And Ansible might be more complex than what you require too, if you just want to be able to install and upgrade many instances of your application via Helm.

### Building an Operator with Operator SDK

Since I'm most familiar with Ansible, though, I'm going to demonstrate building a custom operator in Ansible, using the guide from the Operator SDK.

#### Installing Operator SDK

First, you need to make sure Operator SDK is installed on your system. As with everything else on my Mac, I use homebrew to install it:

```
brew install operator-sdk
```

You can also download the release binary from GitHub if you want.

#### Building an Ansible Operator

The first step to building an operator is to create a project directory, and initialize an Operator SDK project inside:

```
mkdir memcached-operator
cd memcached-operator
operator-sdk init --plugins=ansible --domain=example.com
```

Next you need to create an API for Kubernetes with a role for the API to run in the cluster:

```
operator-sdk create api --group cache --version v1 --kind Memcached --generate-role
```

Next you need to create an Ansible task to actually manage a Memcached instance whenever a new Memcached Custom Resource (CR) is added.

So open `roles/memcached/tasks/main.yml`—this is the task file that is run when the Ansible Operator identifies a new or changed resource. Add the following inside:

```yaml
---
- name: Manage a memcached deployment with {{ size }} replicas.
  community.kubernetes.k8s:
    definition:
      kind: Deployment
      apiVersion: apps/v1
      metadata:
        name: '{{ ansible_operator_meta.name }}-memcached'
        namespace: '{{ ansible_operator_meta.namespace }}'
      spec:
        replicas: "{{ size }}"
        selector:
          matchLabels:
            app: memcached
        template:
          metadata:
            labels:
              app: memcached
          spec:
            containers:
            - name: memcached
              command:
              - memcached
              - -m=64
              - -o
              - modern
              - -v
              image: "docker.io/memcached:1.4.36-alpine"
              ports:
                - containerPort: 11211
```

Next, modify the `spec` inside `config/samples/cache_v1_memcached.yaml` so it provides a custom `size` value for the Ansible operator—and delete the existing `foo: bar` entry:

```yaml
spec:
  size: 3
```

To be complete, you should also set an Ansible role default for the size variable in case the user didn't set one, but I'm skipping that step here.

The operator runs inside a container image, so before you can start using the operator, you have to build the docker image and push it somewhere your Kubernetes cluster can pull it from:

```
make docker-build docker-push IMG=ttl.sh/example-memcached:1h
```

> [ttl.sh](https://ttl.sh) allows you to push ephemeral container images to a public repository for testing purposes. Note that the image will be automatically removed after an hour. For real-world usage, you should push the operator image to a persistent registry!

> Currently there's [a bug](https://github.com/operator-framework/operator-sdk/issues/4403) in the Operator SDK that prevents the generated Makefile from working on macOS Big Sur. The fix is to comment out the `SHELL` line in the Makefile before building.

#### Running an Ansible Operator

First, make sure you have a Kubernetes cluster running somewhere to which you have access as a `cluster-admin`. The easiest thing for testing is to use Kind or Minikube. In my case, I'll start up a Minikube cluster:

```
minikube start
```

Then, install the Operator's CRD into the cluster:

```
make install
```

And finally, deploy the Operator into the cluster, so it can start watching for custom resources:

```
make deploy IMG=ttl.sh/example-memcached:1h
```

After it's deployed, you can check on the new operator pod running in the cluster, which should be in the `memcached-operator-system` namespace:

```
kubectl get pods -n memcached-operator-system
```

Once it's `Running`, you can start deploying instances of your application and the Operator will start managing them!

#### Create a CR, and let the Operator Operate!

Now deploy the Memcached Custom Resource sample modified earlier:

```
kubectl apply -f config/samples/cache_v1_memcached.yaml
```

And watch the Operator perform a 'reconciliation', making sure the CR is in the proper state:

```
kubectl logs deployment.apps/memcached-operator-controller-manager -n memcached-operator-system -c manager
```

After you're done, you can clean everything up by deleting the CR, then un-deploying the operator and CRDs:

```
kubectl delete -f config/samples/cache_v1_memcached.yaml
make undeploy
```

That example was fairly simple, but the idea is you can add Ansible tasks to create whatever resources are necessary, manage connections between them, run necessary installation or database update tasks, and keep things running correctly whenever a change occurs in the cluster.

### Any language, including Python or Rust!

Since Operators are basically Kubernetes Controllers that interact with the Kubernetes API directly, you can write them in any language.

Besides Go, Ansible, and Helm, there are also guides and frameworks to assist with building a [controller in Rust](http://technosophos.com/2019/08/07/writing-a-kubernetes-controller-in-rust.html), or [an operator in Python using Kopf](https://github.com/nolar/kopf).

## Conclusion

There are hundreds of Kubernetes Operators that you can look at for inspiration. I even maintain a [Drupal Operator](https://github.com/geerlingguy/drupal-operator) that could be used to manage multiple Drupal instances in a cluster!

Operators are not the solution for every application, but they do help in many ways, especially if you have a lot of application lifecycle management to automate in a cluster, or if you run many instances of an application.

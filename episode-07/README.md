# Episode 7 - Hello, Operator!

Date: January 13, 2021

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

### Why not use an Operator?

There are some valid reasons for sticking with primatives, especially if you don't have more complex needs, like running many instances of an application in one or more clusters.

TODO: Reasons.

## Popular Kubernetes Operators

TODO:

  - prometheus operator
  - Argo CD operator
  - OpenEBS operator

## Build your own Operator

TODO:

  - Kubebuilder vs Operator SDK?

### Operator Framework / Operator SDK

TODO.

### Any language, including Rust!

TODO:

  - http://technosophos.com/2019/08/07/writing-a-kubernetes-controller-in-rust.html

## Finding other operators

TODO:

  - Operator Hub
  - My own operators... (Drupal, MariaDB, etc.)

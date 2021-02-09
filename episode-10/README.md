# Episode 10 - Monitoring - Lens, Prometheus, and Grafana

Date: February 10, 2021

Video URL: https://www.youtube.com/watch?v=zW-E8THfvPY

Outline:

  - Cluster visibility with Lens
  - Metrics monitoring with Prometheus and Grafana

## Monitoring Kubernetes

In episode 6, we briefly touched on monitoring logs in Kubernetes, but didn't go too deep into that topic.

In the entire Kubernetes 101 series, I've avoided visualization tools and abstractions, mostly because I find learning the basics—the command line interface, the basic architecture—is better to do _before_ you start visualizing your cluster.

In this way, the visualizations make more sense. Many people start their Kubernetes journey by installing some sort of dashboard. Some people can learn the system architecture that way, but I'm not one of those people.

But it _is_ useful to have some abstracted visualizations of the cluster and resource consumption, because we, the operators of these clusters, are humans, and our brains often need visual cues to help us identify anomalies or monitor the overall state of our applications.

So in this episode, I'm going to cover three tools that I've found essential to monitoring almost every production Kubernetes cluster I've built.

[**Lens**](https://k8slens.dev) is billed as an 'IDE' for Kubernetes development, but I think of it as a handy dashboard, one that's easier to use than the traditional web-based dashboards Kubernetes has included.

[**Prometheus**](https://prometheus.io) is a metrics monitoring tool, which is the _de facto_ standard for monitoring and alerting on system metrics in Kubernetes clusters. It can also be used outside of Kubernetes, but is less popular outside the 'cloud native' context.

[**Grafana**](https://grafana.com/oss/grafana/) was forked originally from Kibana 3, but focuses mainly on real-time metrics monitoring dashboards. Most of the fancy-looking monitoring dashboards you've seen in screenshots are built with Grafana. The CNCF even [uses Grafana to plot out project activity](https://k8s.devstats.cncf.io/d/12/dashboards?orgId=1&from=now-7d&to=now-1h&refresh=15m).

Lens is often used by individual Kubernetes developers, and can even be used by other members of a team to inspect deployments and explore application logs, following the permissions they have in the cluster.

Prometheus and Grafana are often used together to gather cluster and custom application metrics, and display them in user-friendly dashboards.

### Two Clusters to Monitor

To help with the examples in this chapter, I've created two Kubernetes clusters:

  1. A local Minikube cluster following the example from chapter 1, with a kubeconfig file in `~/.kube/config`.
  2. A Linode Kubernetes Engine cluster running Drupal, following the example from chapters 4 and 5, with a kubeconfig file in `~/.kube/linode.yaml`.

## Cluster Visibility with Lens

If you work with multiple clusters, managing the contexts in the command line with `kubectl` can get annoying.

Luckily, the open source [Lens](https://k8slens.dev) app, available for Mac, Windows, or Linux, makes it easier to browse multiple clusters and dig into any resource you have the permission to manage.

Lens was originally created by a team at Kontena, but after acquiring Kontenta in early 2020, Mirantis also acquired ownership of the Lens app later in 2020.

The app's source is still maintained under the MIT license, though, so I wouldn't worry about the license being swapped like some other vendors have done recently.

### Install Lens

The easiest way to get lens is to download a binary from the Lens website, at [https://k8slens.dev](https://k8slens.dev).

On my Mac, I prefer to install things via Homebrew if possible, and Lens is available as a cask, so I can install it with:

```
$ brew install --cask lens
```

Lens is also available as a Snap for Linux, if you prefer installing things that way.

### Inspect your clusters with Lens

Once installed, you can open Lens, and start adding clusters.

Click the '+' icon, and browse to a kubeconfig file. I added two clusters:

  1. Local Minikube
     - kubeconfig: `~/.kube/config`
     - context: `minikube`
  2. Linode cluster
     - kubeconfig: `~/.kube/linode.yaml`
     - context: `lke18701-ctx` (will vary)

After adding both clusters, I was able to see node information and browse all the various resources in the cluster.

### Explore Pod Logs

One of my favorite features of Lens is the ability to jump into any Pod's logs quickly through the UI.

If you go to the Workloads > Pods section, and select any Pod, you can click on the 'Logs' icon, and choose any of the containers in that Pod, to see it's logs.

While viewing the logs, you can switch containers, show timestamps (for containers that don't log timestamps in their own output), and even save the log file locally for deeper inspection or archive.

### Log into Nodes and Pods

Another handy feature is the ability to log into any Kubernetes Node or Pod to which you have access.

> **WAIT!?** You may ask... how can Lens allow users to log into a Node?
>
> Well, if you didn't know this already, prepare for your mind to be blown: if you give someone admin privileges in a typical Kubernetes cluster, that user will be able to use Linux kernel namespace features to manage the _nodes themselves_ as root.
>
> This is a _feature_, not a bug, though there are more attempts lately to run Kubernetes in a more locked-down fashion.
>
> But the mechanism Lens uses is fairly simple: when you request shell access to a Node, it runs a Pod on that node with `privileged` access, and runs the command `nsenter -t 1 -m -u -i -n sleep 14000`, thereby allowing that Pod to access resources on the node as root.
>
> Read up on [Pod Security Policies](https://kubernetes.io/docs/concepts/policy/pod-security-policy/) to learn about one way to mitigate attack vectors through privileged Pods.
>
> If you weren't strict about granting admin-level access in your clusters before, you should probably think about being more strict! Anybody with that level of access effectively has root-level access to all nodes in your cluster.

You can visit any Node or Pod, and click the 'Node shell' or 'Pod shell' icon to be dropped into a terminal session inside that Node or Pod.

One major caveat: if the container you're logging into is running a minimal base image like Alpine or Scratch, it might not have a full-fledged `bash` environment like you're used to, so the ability to debug anything inside the container may be limited.

### Visit web services in a browser

Another handy feature is the ability for Lens to automatically create proxy connections between your cluster and your local computer, so you can open web services in your browser.

Just find a service exposed in your cluster under Network > Services, then click on the exposed TCP port. After a few seconds, it should open up in your browser.

### Manage resources

You can edit any resource to which you have access by clicking the 'Edit' icon. You can scale up and down Deployments, you can create new resources with the '+' icon... and you can even open up a Terminal session on your local computer with `kubectl` configured and ready to go by clicking `+`, then selecting 'Terminal session'.

You might notice, however, that Lens shows a blank section on the main Cluster overview, with the warning:

> Metrics are not available due to missing or invalid Prometheus configuration.

Wouldn't it be nice to have those dashboards filled in, too?

While features like Horizontal Pod Autoscaling and `kubectl top` require metrics-server to be running in your cluster, Lens relies on Prometheus for metrics data, so now's a good time to install it.

## Prometheus and Grafana

Prometheus was created at SoundCloud in 2012, when they wanted to both simplify and scale their metrics monitoring beyond what they could do with StatsD and Graphite.

The project's governance was transferred to the CNCF in 2016, and was the second 'graduated' project after Kubernetes itself in 2018.

Grafana is maintained by Grafana Labs, and has thousands of integrations that make it the most flexible open source dashboard visualization and alerting tool. It doesn't only work with Prometheus, but most people looking for an open source monitoring stack for Kubernetes use it along with Prometheus.

### Install Prometheus and Grafana using Helm

For the past few years, an effort was made to build a 'first class' out of the box experience for Kubernetes users integrating Prometheus and Grafana in almost any Kubernetes cluster, and the fruit of that labor is the `kube-prometheus-stack` Helm chart, based on the [kube-prometheus](https://github.com/prometheus-operator/kube-prometheus) project.

This Helm chart installs the following in your cluster:

  - kube-state-metrics (gathers metrics from cluster resources)
  - Prometheus Node Exporter (gathers metrics from Kubernetes nodes)
  - Grafana
  - Grafana dashboards and Prometheus rules for Kubernetes monitoring

To install it, first add the Prometheus Community Helm repo and run `helm repo update`:

```
$ helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
$ helm repo update
```

Then install the stack into the `monitoring` namespace:

```
$ kubectl create namespace monitoring
$ helm install prometheus --namespace monitoring prometheus-community/kube-prometheus-stack
```

Watch the progress in Lens, or via `kubectl`:

```
$ kubectl get deployments -n monitoring -w
```

### Access Grafana

Once deployed, you can access Grafana using the default `admin` account and the default password `prom-operator`.

The Grafana password is stored in the `prometheus-grafana` secret, which you can view with the following command:

```
$ kubectl get secret -n monitoring prometheus-grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo
```

> You can change the password via the parameters passed to the Helm chart when you install it. You should override this password if you're running Grafana in production!

To access Grafana in your browser, run:

```
$ kubectl port-forward -n monitoring service/prometheus-grafana 3000:80
```

Then open your browser and visit `http://localhost:3000/` and log in with the password you found from the earlier command.

But wait! If our cluster already has an Ingress Controller and cert-manager, it's even easier to access Grafana. You can add an Ingress resource for Grafana just like we did for Drupal in episode 6, and Nginx and cert-manager will work in tandem to give you a nice, friendly URL and TLS certificate for HTTPS access.

See the example [`grafana-ingress-tls.yml`](k8s-manifests/grafana-ingress-tls.yml) manifest—which is very similar to the Drupal Ingress manifest—and apply it to the cluster with:

```
$ kubectl apply -f grafana-ingress-tls.yml
```

After a couple minutes, a cert should be acquired, and you can access Grafana at a friendlier URL: `https://grafana.kube101.jeffgeerling.com/`

### Grafana Dashboards

If you go to Dashboards > Manage, you'll see a list of the default dashboards that ship with the kube-prometheus-stack, including a number of dashboards for Compute Resources, Networking, and even Nodes.

Click on one of the dashboards (e.g. 'Nodes') to view the dashboard, and you should see live data going back to the time when Prometheus was installed.

The 'Cluster' dashboard is helpful for a quick overview of which namespaces are consuming the most resources.

### Maintaining Grafana

Grafana has a full fledged access control system, a plugin system, custom dashboards, and more.

In that sense, it's a stateful application, and like Drupal, care must be taken if you want to persist your configuration and data like user accounts.

It's beyond the scope of this episode, but if you are running Grafana in a production environment, you should install the monitoring stack with persistence enabled (the Grafana chart uses a `persistence.enabled` parameter that is set to `false` by default), and make sure you take backups or snapshots of the persistent volume Grafana uses.

## Back to Lens and Conclusion

With Prometheus installed, you should also be able to see cluster metrics in Lens now, though you may need to restart the App before metrics start coming in.

Now that you have more insights into your cluster's resource usage, you can make decisions about how and when to scale more easily, and also consider adding alerts either via AlertManager or Grafana which can be delivered through email, chat apps, or even services like PagerDuty.

Getting the right alerts set up for your own applications is often a trial-and-error process. In my experience, applications run differently on every cloud provider, and some elevated metrics that are concerning on one provider might never be a problem on another provider.

Don't get too bogged down in using a tool like Lens, though. Especially early on in your Kubernetes journey, a tool that abstracts away the basic commands to manage cluster resources can be detrimental to your actual understanding of Kubernetes!

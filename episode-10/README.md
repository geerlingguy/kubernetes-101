# Episode 10 - Monitoring - Lens, Prometheus, and Grafana

Date: February 10, 2021

Video URL: https://www.youtube.com/watch?v=zW-E8THfvPY

Outline:

  - Cluster visibility with Lens
  - Metrics monitoring with Prometheus and Grafana

## Monitoring Kubernetes

TODO.

### Two Clusters to Monitor

To help with the examples in this chapter, I've created two Kubernetes clusters:

  1. A local Minikube cluster following the example from chapter TODO, with a kubeconfig file in `TODO`.
  2. A Linode Kubernetes Engine cluster, following the example from chapter TODO, with a kubeconfig file in `TODO`.

## Cluster Visibility with Lens

TODO.

### Install Lens

TODO.

Download: https://k8slens.dev

TODO.

```
brew install --cask lens
```

TODO.

### Inspect your clusters with Lens

TODO.

## Prometheus and Grafana

TODO.

### Install Prometheus and Grafana using Helm

TODO.

```
helm install stable/prometheus-operator --namespace monitoring --name prometheus
```

TODO.

### Other installation methods

TODO:

  - https://github.com/prometheus-operator/kube-prometheus
  - https://github.com/carlosedp/cluster-monitoring (see Turing Pi)

### Access Grafana

TODO.

### Create Custom Grafana Charts

TODO.

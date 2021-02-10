# Kubernetes 101

[![CI](https://github.com/geerlingguy/kubernetes-101/workflows/CI/badge.svg?branch=master&event=push)](https://github.com/geerlingguy/kubernetes-101/actions?query=workflow%3ACI)

A YouTube live streaming series and book by Jeff Geerling.

This repository contains code examples from the series and the code that powers the [Kubernetes 101 website](https://kube101.jeffgeerling.com).

## Episodes

Click on an episode title to see the resources for that episode:

  - [Episode 1 - Hello, Kubernetes!](episode-01): November 18, 2020
  - [Episode 2 - Containers](episode-02): November 25, 2020
  - [Episode 3 - Deploying Apps](episode-03): December 2, 2020
  - [Episode 4 - Real-world apps](episode-04): December 9, 2020
  - [Episode 5 - Scaling Drupal in K8s](episode-05): December 16, 2020
  - [Episode 6 - DNS, TLS, Cron, Logging](episode-06): January 6, 2021
  - [Episode 7 - Hello, Operator!](episode-07): January 20, 2021
  - [Episode 8 - Kube, Meet Pi](episode-08): January 27, 2021
  - [Episode 9 - GitOps and Lagoon](episode-09): February 3, 2021
  - [Episode 10 - Monitoring with Lens, Prometheus, and Grafana](episode-10): February 10, 2021

## Kubernetes 101 Book

Check out the book inspired by this series, [Kubernetes 101](https://www.kubernetes101book.com).

## Ansible for Kubernetes

Check out Jeff Geerling's book on automating cloud-native infrastructure, [Ansible for Kubernetes](https://www.ansibleforkubernetes.com).

## Static Jekyll site

This GitHub project includes a Jekyll static site configuration inside the `site` folder, which is used by a GitHub Actions workflow to build the static site displayed at [https://kube101.jeffgeerling.com](https://kube101.jeffgeerling.com).

### Building the site locally

Make sure you have Ruby and Bundler installed, then:

    cd site
    bundle exec jekyll serve

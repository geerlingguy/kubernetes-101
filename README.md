# Kubernetes 101

A YouTube live streaming series by Jeff Geerling.

This repository contains code examples from the series and the code that powers the [Kubernetes 101 website](https://kube101.jeffgeerling.com).

## Episodes

Click on an episode title to see the resources for that episode:

  - [Episode 1 - Hello, Kubernetes!](episode-01): November 18, 2020
  - [Episode 2 - Containers](episode-02): November 18, 2020
  - [Episode 3 - Deploying Apps](episode-03): November 18, 2020
  - [Episode 4 - Real-world apps](episode-04): November 18, 2020
  - [Episode 5 - Scaling a Drupal Deployment](episode-05): November 18, 2020
  - [Episode 6 - Hello, Operator!](episode-06): January 6, 2021
  - [Episode 7 - Kube, Meet Pi](episode-07): January 13, 2021
  - [Episode 8 - TBD](episode-08): January 20, 2021
  - [Episode 9 - TBD](episode-09): January 27, 2021
  - [Episode 10 - TBD](episode-10): February 3, 2021

## Ansible for Kubernetes

Check out Jeff Geerling's book, [Ansible for Kubernetes](https://www.ansibleforkubernetes.com).

## Static Jekyll site

This GitHub project includes a Jekyll static site configuration inside the `site` folder, which is used by a GitHub Actions workflow to build the static site displayed at [https://kube101.jeffgeerling.com](http://kube101.jeffgeerling.com).

### Building the site locally

Make sure you have Ruby and Bundler installed, then:

    cd site
    bundle exec jekyll serve

# Episode 6 - DNS, TLS, Cron, Logging

Date: January 6, 2021 (first episode of 2021)

Video URL: https://www.youtube.com/watch?v=E1_uINjq2As

Outline:

  - DNS for Drupal
  - Automating certificates for Drupal Ingress
  - Working with Drupal's Cron system
  - Figuring out what to do with Drupal's internal logs

## Setting things up from Episode 5

Everything in this episode builds on the scalable Drupal configuration we set up in Episode 5. You will need it running to be able to set up everything in this episode.

  1. Deploy the `nfs-server` from Episode 5.
  2. Create a Kubernetes cluster on Linode with at least 3 worker nodes.
  3. Deploy nfs-client-provisioner into the Kubernetes cluster:

     ```
     helm repo add ckotzbauer https://ckotzbauer.github.io/helm-charts
     helm install --set nfs.server=192.168.148.123 --set nfs.path=/home/nfs ckotzbauer/nfs-client-provisioner --version 1.0.2 --generate-name
     ```

  4. Deploy metrics-server into the Kubernetes cluster:

     ```
     helm repo add bitnami https://charts.bitnami.com/bitnami
     helm install -f values/metrics-server.yml metrics-server bitnami/metrics-server
     ```

  5. Deploy Drupal and MariaDB into the cluster:

     ```
     cd k8s-manifests
     kubectl create namespace drupal
     kubectl apply -f mariadb.yml -f drupal.yml
     ```

## DNS and Ingress setup for Drupal

Up to this point, we've been accessing Drupal using a NodePort, with an IP address and port, like `http://69.164.207.70:32270/`.

But that's not a very user-friendly URL for a website or web application.

We really need to get DNS to point to the site, so we can have a friendly site name, like `ep6.kube101.jeffgeerling.com`.

There are a few different ways to do this, and the easiest would be to switch Drupal's Service to a LoadBalancer, which would provision a cloud load balancer—on Linode, a NodeBalancer, but something like an ELB on AWS.

This is easy, but if you plan on deploying many sites and publicly-accessible HTTP apps to your cluster, it can get expensive _fast_, because each Load balancer will incur an hourly or monthly fee.

So the preferred way to control DNS tying into backend applications is to use what Kubernetes calls 'Ingress'.

Instead of a Load Balancer per application, you set up one Load Balancer for an Ingress Controller, which runs software like Nginx, HAProxy, Traefik, or Envoy, and that Ingress Controller proxies web requests to backend applications like Drupal.

### Set up an NGINX Ingress Controller

In this case, mostly because it's the webserver and proxy I'm most familiar with, I'm going to set up Nginx, and there's a simple Helm chart that sets up Nginx as a cluster Ingress Controller:

```
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm install ingress-nginx ingress-nginx/ingress-nginx
```

> Notice I mentioned "a" cluster Ingress Controller, and not "the" cluster Ingress Controller? You can have multiple Ingress Controllers in your cluster, if you'd like!

Get the IP address of the `nginx-ingress-controller` using the command:

```
kubectl get svc ingress-nginx-controller
```

Now go to your DNS provider, and add an A record pointing a domain or subdomain at the "EXTERNAL-IP" address of the Ingress Controller, for example:

```
ep6.kube101.jeffgeerling.com --> A 45.79.240.122
```

At this point, if you visit the URL, you'll get a 404 Not Found from Nginx (assuming everything was updated correctly and the DNS change has propagated!).

### Set up Ingress for Drupal

To connect the domain name to the Drupal backend, we need to add an Ingress record that tells Nginx to route requests for `ep6.kube101.jeffgeerling.com` to the `drupal` service on port 80.

And the [`k8s-manifests/drupal-ingress.yml`](k8s-manifests/drupal-ingress.yml) file does just that!

So go ahead and take a look at it, then deploy it into the cluster with:

```
kubectl apply -f drupal-ingress.yml
```

After a few seconds, you should be able to access the site (e.g. http://ep6.kube101.jeffgeerling.com/), and you should get Drupal.

Much better now!

And since the Ingress is pointed at a Kubernetes Service in front of three or more Drupal Pods, it will make sure all the requests to Drupal are load-balanced for better scalability.

### External DNS Integration

There are a few things relating to Ingress I won't cover in this 101 series, and one of them that may be pertinent, depending on your DNS provider, is integration between Kubernetes and External DNS providers for easy DNS configuration for Services.

The [External DNS](https://github.com/kubernetes-sigs/external-dns) integration integrates Kubernetes with a large number of external DNS providers, including AWS Route 53, Google Cloud DNS, Linode DNS, CloudFlare, and more. This makes it very easy to manage domain integration into your Kubernetes apps, but it assumes you have your DNS records configured in one of the supported services.

In the case of my example today, I used Name.com, which does not have an API that integrates with Kubernetes, so I wasn't able to demonstrate External DNS.

Instead, I manually grabbed the IP address of the NodeBalancer Linode provisioned for me, and added an A record in Name.com's frontend interface.

At a certain scale, managing DNS manually is very inefficient!

## Set up TLS with cert-manager and Let's Encrypt

Speaking of things hard to manage manually, most web applications should be set up so they are secured using TLS encryption, meaning you access them over encrypted HTTPS and not via plaintext HTTP.

And the easiest way—and the way I have always set up my own Kubernetes clusters—to configure certificates automatically for your Ingress resources is to use [cert-manager](https://cert-manager.io).

Most of the time I use it with Let's Encrypt to get valid public certificates, but you can also use cert-manager with many other setups, including private CAs for internal applications, depending on your needs.

Installing cert-manager is easy enough.

First, create a namespace:

```
kubectl create namespace cert-manager
```

Then install the CRDs (Custom Resource Definitions) for cert-manager resources:

```
kubectl apply -f https://github.com/jetstack/cert-manager/releases/download/v1.1.0/cert-manager.crds.yaml
```

Then add the `jetstack` Helm repository and install `cert-manager` from Jetstack's chart, into the `cert-manager` namespace:

```
helm repo add jetstack https://charts.jetstack.io
helm install cert-manager jetstack/cert-manager -n cert-manager --version v1.1.0
```

Check that it's running:

```
kubectl get pods -n cert-manager
```

Once it's running, you'll need to set up a ClusterIssuer inside the cluster that will tie cert-manager to a CA service—in our case, LetsEncrypt. Look at the configuration of the `ClusterIssuer` inside the [`k8s-manifests/cluster-issuer.yml`](k8s-manifests/cluster-issuer.yml) file. There are many other options you could choose, including using Let's Encrypt's staging environment for a non-production cluster.

Create the `letsencrypt-prod` ClusterIssuer:

```
kubectl apply -f cluster-issuer.yml
```

Then deploy an updated Drupal Ingress manifest, which is the same as the ingress resource added earlier, but with TLS settings using the `letsencrypt-prod` ClusterIssuer we just created:

```
kubectl apply -f drupal-ingress-tls.yml
```

It will take cert-manager a minute or two to perform it's HTTP01 challenge in the background. You can monitor progress by viewing the cert-manager pod logs:

```
kubectl logs -f -n cert-manager -l app=cert-manager
```

Now you should be able to refresh the Drupal site in your browser and see a secure connection, with a valid LetsEncrypt certificate! (In my case, issued by `DST Root CA X3 -> R3`).

## Keeping Drupal Happy with a CronJob

Drupal is configured out of the box to use a special module, called "Automated Cron", that runs its internal maintenance jobs on a regular cycle.

However, there is a downside to the way this works—it relies on external web requests to operate.

Drupal waits until an a certain amount of time passes—by default, 3 hours—and it waits for a web request to be made, and then after it serves that request, it runs a cron job through that same request cycle.

One problem is that you don't have control over external web requests, so if you want to run Drupal's cron more frequently, say every 5 minutes, you can't rely on having steady traffic at all hours of the day.

Another problem is this ties cron runs to a web process that is tuned for external access performance, and might not even have the resources to allow Drupal to complete all its maintenance tasks efficiently.

Kubernetes has a solution in the form of a CronJob resource. Kubernetes Jobs are one-off Pod definitions which run a Pod to completion, keeping the log output for later inspection.

Kubernetes CronJobs are an efficient way of managing repetitive Jobs, like running Drupal's cron on whatever schedule you want.

I wrote an entire blog post on this topic in 2018: [Running Drupal Cron Jobs in Kubernetes](https://www.jeffgeerling.com/blog/2018/running-drupal-cron-jobs-kubernetes), but I'll dive into how to do everything on our example Drupal site in this episode.

Make sure the URL to the Drupal site's cron URL is correct (you can find this under Configuration > System > Cron) inside [`k8s-manifests/drupal-cronjob.yml`](k8s-manifests/drupal-cronjob.yml), then deploy the CronJob into the cluster:

```
kubectl apply -f drupal-cronjob.yml
```

Then verify the CronJob exists in the Drupal namespace with:

```
kubectl get cronjob -n drupal
```

After you see a Job run successfully (you can monitor cron-triggered Jobs with `kubectl get job -n drupal`), check in Drupal's admin UI to make sure Cron runs are being detected.

There are a few things to be aware of when using Kubernetes CronJobs:

  1. The Successful and Failed Job History Limits allow you to control how many Jobs are kept before Kubernetes prunes old Jobs, for both successful and failed runs. It may be useful to increase these limits, especially if you need to debug CronJob problems.
  2. For most Jobs, it is a good idea to disable concurrency using the 'Forbid' concurrency policy. But there are use cases where you might want to change that to 'Allow' or 'Replace'.
  3. When you start hitting frequent Job failures, you could run into some weird issues... at least I have in the past. Sometimes I've had to re-create a CronJob to get it to start running its schedule again after too many Job failures in a row.
  4. Beware of having too many Jobs on your cluster. If you run hundreds of sites and have a CronJob for each, running every minute, this can overload your Kubernetes control plane! See my [50,000 Kubernets Jobs for 50K subscribers](https://www.youtube.com/watch?v=O1iEBzY7-ok) video for more on that!

Once you can verify the Kubernetes CronJob is running Drupal's cron successfully, it would be a good idea to disable the now-redundant Automated Cron module in Drupal!

## Monitoring Drupal's Logs

Now, I had originally planned on showing a nice, simple logging setup for Drupal that I could demo in a few minutes in this episode.

But logging is not rainbows and sunshine. It wasn't easy to do centralized logging with multiple servers _before_ Kubernetes was a thing, and it's not easy with multiple containers inside Kubernetes either.

I have to punt on this topic because dealing with logs—even with one simple site—is not a quick and easy problem to solve.

Especially if you want to handle logs in a scalable and secure manner!

That said, here are a few potential solutions that I have either used in the past or think would be good for many use cases:

### Using an External SaaS Log Aggregator

If you have the budget, this is going to be the easiest option. Services like [Sumo Logic](https://www.sumologic.com), [DataDog](https://www.datadoghq.com/product/log-management/), and [Elastic](https://www.elastic.co/observability) provide cloud log aggregation, monitoring, and search dashboards.

They are relatively easy to set up, they have pre-made ingest plugins available for most applications (e.g. [DataDog Logs HTTP for Drupal](https://www.drupal.org/project/datadog)) and for Kubernetes itself, and they can usually scale to thousands of applications without a hassle.

But they do cost a _lot_ (usually). That's the number one downside, but for many companies, the ease of integration and flexibility make it worthwhile.

### Running your own ELK Stack

If you want to save on the costs of a hosted service, running your own ELK stack is a perfectly reasonable option. Especially for smaller clusters (where you aren't dealing in many gigabytes of logs per day), it is not impossible to manage an ELK stack with a small team.

However, maintenance is not free, and the Elastic stack (Elasticsearch, Logstash, and Kibana) require an investment in time to set up the stack, and maintain it.

And it's not lightweight, either—in my experience I've had to allocate a lot of resources to get an Elasticsearch cluster to run well beyond one or two medium-traffic sites dumping all their log data, in addition to sometimes-noisy Kubernetes services.

### Relying on a Service Mesh

So-called 'Service Meshes' like Istio or Google's Anthos sometimes have their own logging integrations built-in.

I don't typically recommend a Service Mesh layer on top of a Kubernetes cluster, because I kind of see it like spraying a firehose of 'all the things' on top of a cluster, and in reality, you don't necessarily need 'all the things' that a Service Mesh provides (they sure add to your cluster operation costs though!).

It's often easier to inject specific sidecar containers—that is, containers that run alongside your application containers in the same Pod—to do specific purposes like extract log files or stream specific data to or from your application.

### Using your cloud provider's solution

Many cloud providers have integrated logging in their platform. You're already paying for it (most likely), so why not use it?

Some of the solutions are not as robust as the alternatives, but the primary purpose for central logging is to be able to audit your applications and identify problems, and the basics are usually covered well.

Speaking of cloud providers offering integrated logging, the Kubernetes 101 series sponsor, [amazee.io](https://www.amazee.io), maintains their own [Lagoon Logs](https://www.drupal.org/project/lagoon_logs) project that integrates Drupal logs with Lagoon's logging platform, through Logstash.

This is one argument in favor of relying on hosting partners who are invested in the software you are using—they can offer deeper integration and insights into the apps you host with them!

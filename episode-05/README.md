# Episode 5 - Scaling Drupal in K8s

Date: December 16, 2020 (last episode of 2020)

Video URL: https://www.youtube.com/watch?v=euZdS5b2siA

Outline:

  - Setting up a shared filesystem to scale Drupal's deployment
  - Setting up Horizontal Pod Autoscaling
  - Options for High-Availability Databases
  - Pre-Christmas Q&A and Book Giveaway!

## Fixing the scalability issue with Drupal Pods

Last episode, we tried increasing the replica count beyond 1 Pod for the Drupal Deployment, but got a "Multi-Attach error", because the default block storage that was connected to the Drupal pod doesn't have the ability to be attached to multiple Pods at the same time.

But if you want to scale up Drupal, Drupal has to be able to read and write files to a shared, persistent filesystem!

The problem is you can only use the 'ReadWriteOnce' storage mode for normal cloud storage, like Amazon EBS volumes, or Linode Volumes.

So to make 'ReadWriteMany' volumes available, we have to set up a different storage provisioning system for your cluster.

### Shared Storage Options

In many ways, the simplest and most reliable option is to use a shared filesystem service from your cloud provider, for example, Amazon EFS.

Running a filesystem cluster, whether it's running with Ceph, Gluster, NFS, or some other storage technology, can be very difficult. The configuration is complex, there are many moving parts, backups can be tricky, and data can easily be lost if you don't know what you're doing!

Cloud systems like Amazon EFS make it so you click a button and have shared storage available, so I usually recommend it if you can use it in your own cloud environment.

But what if you're running a bare metal Kubernetes cluster on your own servers? Or if you're on a cloud environment that doesn't have something like EFS—for example, with Linode currently?

### Rook and Ceph

Initially, I was going to set up this episode's demo using [Rook](https://rook.io) to manage an in-cluster CephFS clustered filesystem.

But after spending almost an entire workday trying to find a way to easily (and more importantly for this live series, _demonstrably_) deploy a Rook/CephFS cluster into Kubernetes—either in Minikube, Linode Kubernetes Engine, or even Amazon's EKS, I decided the complexity was not worth it.

However, I am a strong believer in Rook and CephFS, and they make a potent combo for cloud-provider-agnostic flexible storage options. If you have the time and inclination, it is worth learning how they work, getting them running, and seeing if Rook might be the storage provisioner you should use for your cluster—especially if you're building on bare metal!

Anyways, to keep things a little simpler, while still achieving the goal of having a shared storage provisioner running in the cluster for scalability, I switched gears.

### NFS

I settled on NFS for simplicity's sake, and in some circumstances, it's actually not a bad option to just run one simple NFS server in production. That's how I've had my Raspberry Pi cluster set up for years, and it's easy to maintain, pretty performant, and easy to restore from backups if the server dies.

## Set up an NFS server

Setting up NFS is not too complicated, especially if you just need to be able to set up one shared directory, shared to one private network.

There is an Ansible playbook with its own README guiding you through the process in the [`nfs-server` directory](nfs-server/).

## Reconfigure the Drupal PersistentVolumeClaim for NFS

Assuming you have an NFS server running, and a Kubernetes cluster running which can connect to the NFS server, you will need to modify Drupal's PersistentVolumeClaim to be able to use the NFS storage.

But before you do _that_, you'll need to configure the Kubernetes cluster _itself_ to be able to connect a StorageClass to the NFS server.

### Set up NFS client provisioner in K8s

When you're searching around for how to configure NFS in Kubernetes so your Pods can use NFS-based volumes, you might run into a little confusion; originally, there were two provisioners:

  - `nfs-server-provisioner`
  - `nfs-client-provisioner`

The NFS Server provisioner would run in-cluster NFS instances to allow pods to connect to them. And that's great and handy, but many people would rather manage NFS servers separate from their clusters, and that's where the NFS Client provisioner comes in—it assumes you have an NFS server or storage cluster running elsewhere, and provisions directories inside an NFS share for PVCs in your cluster.

To add to the confusion, though, in 2020 both provisioners were moved and slightly renamed, the server provisioner moving to [nfs-ganesha-server-and-external-provisioner](https://github.com/kubernetes-sigs/nfs-ganesha-server-and-external-provisioner), and the client provisioner to [nfs-subdir-external-provisioner](https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner).

To keep things simple, I'm just going to use a pre-existing Helm Chart to deploy the client provisioner:

```
helm repo add ckotzbauer https://ckotzbauer.github.io/helm-charts
helm install --set nfs.server=192.168.217.5 --set nfs.path=/home/nfs ckotzbauer/nfs-client-provisioner --version 1.0.2 --generate-name
```

For the `nfs.server`, use the Private IPv4 address for your NFS server. For the `nfs.path`, set it to the path of an NFS share on that server.

You can check if the provisioner is running with `kubectl get pods`; make sure an NFS provisioner pod is running.

### Deploy Drupal and MySQL (MariaDB)

Now modify the Drupal PVC definition in the `drupal.yml` manifest to look like the following:

```yaml
---
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: drupal-files-pvc
  namespace: drupal
spec:
  accessModes:
    - ReadWriteMany  # Was ReadWriteOnce before!
  resources:
    requests:
      storage: 1Gi
  storageClassName: nfs-client  # This is new!
```

Since we're now using `nfs-client` for the storage class, we can set `ReadWriteMany` for the access mode, meaning our Drupal deployment should be able to scale beyond just one Pod. Yay!

You can deploy Drupal and MariaDB now:

```
kubectl create namespace drupal
kubectl apply -f mariadb.yml -f drupal.yml
```

And monitor the deployment with `kubectl get deployments -n drupal -w`.

Once Drupal is ready, get the NodePort with `kubectl get service -n drupal` and an IP address of one of the servers with `kubectl get nodes -o wide`. Then access the site and install it.

### Save a File and observe it

If you want to manually verify the NFS share is working, you can log into your NFS server and monitor the NFS folder, then log into Drupal and create an Article, uploading an image to the Body field of that article.

After you save the new Article, you should a new image inside the shared Drupal files directory the NFS client provisioner created in the NFS share. Fancy!

## Scale Drupal up... and down!

One quick way to test how many requests you can serve on a Drupal site is to use the 'ApacheBench' tool, and we can benchmark how fast Drupal can serve its cached home page with the single replica we have:

```
ab -n 500 -c 10 http://45.79.40.239:32119/
```

(Substitute one of your node's IP addresses and the Drupal service NodePort in this command.)

On my test cluster, this process reported the site serving around 80 requests per second. Not bad, but we could get more out of our cluster by scaling up Drupal!

So edit the Drupal deployment, setting `replicas: 5`:

```
kubectl edit deployment -n drupal
```

Then wait for the Deployment to report being up to date with five running instances:

```
watch kubectl get deployment -n drupal drupal
```

Because our persistent NFS storage allows multiple Pods to mount the same volume, the Drupal deployment now successfully scales up to five instances! Hit Ctrl-C to stop watching the deployment, and run ApacheBench again:

```
ab -n 500 -c 10 http://45.79.40.239:32119/
```

As the caches warm up, the requests per second should go up a bit, since the load can be more evenly distributed to multiple Drupal instances.

If you set the deployment back to `replicas: 1`, the requests per second will likely go right back down to the original value.

## Use Horizontal Pod Autoscaling (HPA)

One of the often-touted features of Kubernetes is automatic scaling. What people probably _don't_ tell you is that, as with all things in life, nice things like autoscaling are _not_ free. You usually have to configure the cluster and your applications to make autoscaling actually work.

### Set up `metrics-server`

Many cluster environments do not include an essential component the [Horizontal Pod Autoscaler (HPA)](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/) relies on, namely the `metrics-server` component.

`metrics-server` monitors resource usage in the cluster for pods and nodes and aggregates the data in Kubernetes' API.

To install it manually, run:

```
helm repo add bitnami https://charts.bitnami.com/bitnami
helm install -f values/metrics-server.yml metrics-server bitnami/metrics-server
```

And to make sure it's actually working, run `kubectl top node` and you should see something like the following:

```
$ kubectl top node
NAME                          CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
lke15196-18570-5fd9222ff4ff   112m         5%     956Mi           24%
lke15196-18570-5fd9222ffd28   92m          4%     886Mi           23%
lke15196-18570-5fd92230054d   75m          3%     839Mi           21%
```

> **Note**: See the article [How Can I Deploy the Kubernetes-Metrics Server on LKE?](https://www.linode.com/community/questions/19756/how-can-i-deploy-the-kubernetes-metrics-server-on-lke) for more details on metrics-server on Linode in particular.

### Configuring HPA for Drupal

You can create an autoscaling configuration using `kubectl` just like other resources:

```
kubectl autoscale -n drupal deployment drupal --min=1 --max=8 --cpu-percent=50
```

As with the Drupal deployment itself, it's best long-term to save the HPA configuration in a YAML manifest, so you aren't manually running commands to configure things in your cluster. But for now, it's easy enough to demonstrate how an HPA works using `kubectl`.

Now you can monitor the status of the HPA you just created with:

```
watch kubectl get hpa -n drupal
```

But if you just sit and watch it, it's not likely anything exciting will happen, because Drupal is running pretty much idle.

You can also monitor, in a separate window, the current CPU and Memory usage of the Drupal pods, with:

```
watch kubectl top pods -n drupal -l app=drupal
```

### Testing HPA for Drupal

To quickly create a lot of load on a Drupal site, there's no better way than hammering the Module admin page as an authenticated user.

Log into the Drupal site, and view the session cookie that is set by the site. Copy out the cookie name (starts with `SESS`) and the value (a random string), then paste them into the following `ab` command, which requires Apache Bench to be installed on your computer. Put in the correct IP address and NodePort for your Drupal deployment, and run the command:

```
ab -n 1000 -c 3 -C "session_cookie_name=key" http://45.79.40.239:32119/admin/modules
```

It may take a couple minutes for the Kubernetes' HPA to catch up, because it tries to not be _too_ agressive, but it should eventually kick off eight pods that will help handle the insane amount of load you just put on your cluster.

After the requests are complete, Kubernetes will wait for a preset cooldown period of time before it starts removing Pods, and this is to prevent _thrashing_, which could happen if the scaling were done too quickly.

That's why there's always going to be some lag between traffic and autoscaling. Autoscaling is NOT going to magically solve your scalability problems, and we've also only configured it, at this point, for the application layer—not the database, or any other services Drupal might rely on!

> **NOTE**: The cooldown delay period is only configurable on the cluster level, and is not usually configurable in managed Kubernetes environments. There are alternative pod autoscalers with more configurability, but their setup is outside the scope of this lesson.

## Scaling Databases

The last topic of this episode is scaling databases.

And I have to share my advice that I've learned through the course of building a number of production clusters, some running less than ten applications, others running thousands of applications.

Unless you love dealing with massive complexity and scaling difficulty, I wouldn't recommend trying to configure _traditional_ HA database configurations inside a Kubernetes cluster.

What most people do—and I've done often—is rely on a cloud provider's database. For example:

  - Amazon Aurora with Amazon EKS
  - Google Cloud SQL for Google GKS
  - DigitalOcean Managed MySQL for Kubernetes

The complex database scalability concerns are taken care of by the Cloud provider... but you _do_ end up paying for it!

An alternative is to run single-Pod databases, without high availability, for each application that needs one. Honestly, for most of my own applications, I run a single-server database with good backups, and it's reliable enough for many use cases.

Running things inside Kubernetes in a similar fashion is just as reliable, but could actually be more easily scalable through Kubernetes own tools, depending on the servers you set up.

But running databases inside Kubernetes—even if you use something like [Vitess](https://vitess.io), Bitnami's [MariaDB Cluster Helm Chart](https://engineering.bitnami.com/articles/deploy-a-production-ready-mariadb-cluster-on-kubernetes-with-bitnami-and-helm.html), or Presslabs' [MySQL Cluster Operator](https://www.presslabs.com/docs/mysql-operator/)—is challenging.

The more complex the setup, and the higher the requirements, the more you have to start considering things like:

  - Using dedicated nodes for database Pod scheduling
  - Ensuring database Pods have the correct taints and tolerations to not end up on the same node
  - Configuring specialized storage classes for higher-performance data storage, or using on-instance high-speed storage
  - Configuring robust database backups and snapshotting

Unless your needs are lightweight, you might realize the relatively high cost of cloud-managed databases is well worth it when you switch to cloud-native infrastructure!

> **NOTE**: Another option, if you are able to adapt your applications, is to use a more 'cloud-native' database solution, like [CockroachDB](https://github.com/cockroachdb/cockroach).

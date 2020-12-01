# Episode 3 - Deploying Apps

Date: December 2, 2020

Video URL: https://www.youtube.com/watch?v=nn9J9sWLj_w

Outline:

  - Building a test cluster in the cloud
  - Deploying our Hello Go App into Kubernetes
  - Connecting a private image registry to Kubernetes
  - Exposing, scaling, and updating our App
  - Rolling back a deployment

## Creating a Linode Cluster for cloud-based testing

Some people may not be able to install Minikube on their computers for some reason or another, and if that's the case, you could use some free credit from Linode to build a Kubernetes cluster for development and testing.

To do that, visit this link: [https://linode.com/geerling](https://linode.com/geerling).

Sign up for a new account, then in the Linode Cloud control panel, go to the Kubernetes section. Create a new cluster, add a few nodes to the default node pool, and click Create.

The process takes about two minutes, and at the end, you can download the 'Kubeconfig' file to your computer, so you can use it with `kubectl` to administer the cluster.

On my computer, I copied the file into my `.kube` directory:

    mv ~/Downloads/kube101-kubeconfig.yaml ~/.kube/config-kube101

Then I made sure `kubectl` would use this cluster config:

    export KUBECONFIG=~/.kube/config-kube101

Now if I run any `kubectl` commands, they will work on the new Linode cluster!

```
$ kubectl get nodes
NAME                          STATUS   ROLES    AGE   VERSION
lke14312-17562-5fc5708862ca   Ready    <none>   1h   v1.18.8
lke14312-17562-5fc570886f7d   Ready    <none>   1h   v1.18.8
lke14312-17562-5fc57088781f   Ready    <none>   1h   v1.18.8
```

## Deploying Hello Go into Kubernetes

After creating a local cluster with `minikube start`, or a cloud-based cluster, it's time to deploy 'Hello Go' into the cluster!

You can do that with:

    kubectl create deployment hello-go --image=geerlingguy/kube101-go:1.0.0

Then watch the deployment status with `watch kubectl get deployment hello-go`.

But wait! The rollout seems to be failing, as it is not showing that the Deployment is reaching a 'ready' state...

Let's check on the Pods for this Deployment and see what might be happening:

    kubectl get pod -l app=hello-go

Hmm, an `ErrImagePull` doesn't sound so good. Let's dig in deeper and use the `kubectl describe` command to get the details:

```
$ kubectl describe pod -l app=hello-go
Name:         hello-go-5944979865-qfxfc
Namespace:    default
...
Events:
  Type     Reason     Age                From               Message
  ----     ------     ----               ----               -------
...
  Warning  Failed     2s (x3 over 43s)   kubelet            Failed to pull image "geerlingguy/kube101-go:1.0.0": rpc error: code = Unknown desc = Error response from daemon: pull access denied for geerlingguy/kube101-go, repository does not exist or may require 'docker login': denied: requested access to the resource is denied
```

Ah, so Kubernetes can't pull that image, since it's in a private Docker registry. We'll have to tell Kubernetes how to authenticate to Docker Hub, since that's where the image lives. And we can do that using a special kind of [Kubernetes Secret](https://kubernetes.io/docs/concepts/configuration/secret/) called a Docker Registry secret:

```
kubectl create secret docker-registry regcred \
    --docker-username=geerlingguy \
    --docker-password=[TOKEN GOES HERE] \
    --docker-email=geerlingguy@mac.com
```

For Docker Hub, you would put in your username, password (an Authentication Token, which can be generated in your Account Settings in the 'Security' section), and email address. For other registries, you would also need to add a `--docker-server` URL.

Once you create the secret, you need to modify the `hello-go` deployment to make sure it knows to _use_ that secret.

So edit the deployment with:

    

And add a new `imagePullSecrets:` section under `spec.template.spec` like so:

```yaml
spec:
  ...
  template:
    ...
    spec:
      imagePullSecrets:
      - name: regcred
      containers:
      - image: geerlingguy/kube101-go:1.0.0
```

Then check on the progress of the rollout with `watch kubectl get deployment hello-go`. Ah, much better now!

### Exposing the Hello Go App

The next step to having a usable app is to expose it, to make it visible and usable outside the cluster.

    kubectl expose deployment hello-go --port=80 --target-port=8180 --type=NodePort

The `kubectl expose` command sets up a Service, which allows Kubernetes to route requests to one or more Pods matching a set of conditions.

In this case, we are setting up a Service which will route requests to all Pods running in the `hello-go` deployment, and it will receive requests on port 80 and direct them to the container's port 8180—which is the port we chose to use for our Hello Go application.

Finally, we set the `type` to `NodePort`, and this makes it so the Hello Go App Service will be accessible on a given port on every node in the cluster.

In Minikube's case, the service would be available on the Minikube IP address, which you can get with `minikube ip`.

On a Linode Kubernetes cluster, it would be available on any of the cluster nodes' IP addresses.

You can get the port using:

    kubectl get service hello-go

### Scaling the Hello Go App

Next up, let's scale the App. Since it's stateless, meaning there's no persistent data or external database that it needs to interact with, it should be easy to scale up Hello Go by increasing the number of 'replicas' in the deployment.

Edit the deployment with:

    kubectl edit deployment hello-go

Modify the `replicas` line, and set it to `3`. Save the modifications.

Now check the deployment and verify it is rolling out two additional Pods:

    kubectl get deployment hello-go

And if you want, you can even tail all the logs from all the pods in the deployment with this handy command:

    kubectl logs -f -l app=hello-go --prefix=true

This will let you see which pod and container are serving which request. Head over to a web browser (or two!) and load the IP address and NodePort for the Hello Go service. Refresh the page a few times, and observe the logged requests.

You might note that the requests seem 'sticky'; one browser keeps hitting the same pod, while a different browser hits a different pod every time. Kubernetes' Service layer does have some customizability in this regard (how it distributes requests to different Pods), but if you need more customization, you might want to use an Ingress Controller. We'll talk about those later.

Press Ctrl-C to exit the log viewer.

### Updating the Go App

Now it's time to update the app. Marketing wanted a huge change—change the text from "Hello" to "Hi" since that is more friendly. And the developers just checked in a change and pushed up a new image version—1.1.0.

So to deploy that image version to the Kubernetes cluster, you can modify the image directly:

    kubectl set image deployment/hello-go kube101-go=geerlingguy/kube101-go:1.1.0

And you can watch how the pods are replaced using:

    watch kubectl get pods -l app=hello-go

And confirm the version of the container image used with `kubectl describe`, either inspecting the Deployment or the Pods.

### Rolling back the Deployment

Let's say Marketing just realized that the word "Hi" means "I will destroy you" in an as-yet-unknown alient language, and they want you to quickly revert to the previous version of the App.

The fastest way to do that is to use `kubectl rollout undo`, and we can run this command to undo the latest deployment of the `hello-go` Deployment:

    kubectl rollout undo deployment hello-go

After doing that, check on the version of the container image of the deployment with `kubectl describe deployment hello-go`, and verify it's been reverted to 1.0.0.

Whew! What a day. If you're using a test cluster, be sure to delete it with `minikube delete`, or by deleting the cluster in the Linode Kubernetes control panel.

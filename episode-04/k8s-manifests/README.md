# Drupal Kubernetes Deployment Manifests

This directory contains two deployment manifests, one for MariaDB, and one for Drupal (which builds a Drupal Deployment running Drupal on top of Apache + PHP).

To apply them to a Kubernetes cluster (e.g. with Minikube: `minikube start`), run the commands:

```
# Create a namespace for the Drupal site.
kubectl create namespace drupal

# Create the MySQL (MariaDB) Deployment.
kubectl apply -f mariadb.yml

# Create the Drupal (Apache + PHP) Deployment.
kubectl apply -f drupal.yml
```

You can then observe the status of the deployments:

```
kubectl get deployments -n drupal -w
```

Then press Ctrl+C to stop watching the deployment rollout.

## Working in Namespaces

If you want to do a lot of work in a particular namespace (e.g. `drupal`), you can set the current namespace context using:

```
kubectl config set-context --current --namespace=drupal
```

And you can confirm which namespace context you're in with:

```
kubectl config view | grep namespace:
```

When you're done performing actions (e.g. `kubectl get pods`) only within that namespace, run:

```
kubectl config set-context --current --namespace=""
```

## Accessing the Drupal site

After the Drupal deployment is complete, you can see access it via:

```
# In Minikube, this will open the URL directly.
minikube service -n drupal drupal

# In other clusters, get the service to get the NodePort.
kubectl get service -n drupal drupal
```

That will give you the NodePort on which the service is exposed, but what about the IP address? Well, for many commands in Kubernetes, they give a brief summary by default, but you can get a lot more information adding `-o wide`:

```
kubectl get nodes -o wide
```

Now you can take the IP address of any node, and pair that with the NodePort that maps to TCP port 80 on Drupal's service, and access the site. You should be taken to the Drupal installer UI.

And you can access the Drupal container's Apache logs with:

```
kubectl logs -f -n drupal -l app=drupal
```

Install the Drupal site using the UI, and create an Article (under Content > Add content > Article) with an image inside the Body text.

## Scaling the Drupal deployment

Go ahead and edit the Drupal deployment:

```
kubectl edit deployment -n drupal drupal
```

Set `replicas` to `3`, and save the changes.

Watch the Pods as they are deployed in the scale-up event:

```
kubectl get pods -n drupal -w
```

But oh no! It seems like these pods are stuck on 'Init'. Investigate further, with:

```
kubectl describe pod -n drupal -l app=drupal
```

You might notice a `Multi-Attach error`. It seems that our fancy Drupal deployment—which is similar to every other Drupal, Wordpress, and similar LAMP-based deployment I see in Kubernetes tutorials, has a major problem: _you can't scale it up!_

There goes one of the major advantages we _thought_ we'd get with Kubernetes... or does it? In the next episode we'll dig deeper into scaling Drupal, accessing Drupal with a Domain using Ingress, SSL, and other real-world app concerns.

## Cleaning up

When you're finished testing, one of the best advantages of using Kubernetes namespaces is the easy ability to clean up everything in the namespace.

All you have to do is delete the namespace, and Kubernetes will clean up everything inside:

```
kubectl delete namespace drupal
```

After a minute or two, all traces of Drupal should be gone. Sometimes there may be remnants, however, like persistent volumes that are 'retained' for safekeeping (if you're using Linode, for example). Go ahead and delete those with:

```
kubectl get pv | grep Released | awk '$1 {print$1}' | while read vol; do kubectl delete pv/${vol}; done
```

Confirm the volumes have been deleted in the cloud provider's UI as well; for Linode, at least, deleting the PV via Kubernetes does not trigger a delete of the underlying block storage device!

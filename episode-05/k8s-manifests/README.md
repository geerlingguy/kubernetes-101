# Drupal Kubernetes Deployment Manifests

This directory contains the same deployments from Episode 4, but with some modifications to help scale Drupal.

You can apply these manifests to any Kubernetes cluster (e.g. `minikube start` for a local cluster, or a cloud environment like Linode Kubernetes Engine).

## SEE ALSO

  - NFS: https://github.com/kubernetes-sigs/nfs-ganesha-server-and-external-provisioner
  - Maybe just use EFS ;)

## Setting up an NFS Server

I created an 8GB Linode with a Private IP address. Then I ran the Ansible playbook in the `nfs-server` directory to set it up.

## Configuring a shared filesystem for scalability

TODO: Explain Rook and CephFS.

TODO: Explain how I tried getting it working on Minikube, and Linode (manually and via Helm. It's complicated.

TODO: Explain other options (NFS, EFS, etc.).

TODO: Explain NFS provisioner (server provisioner vs client provisioner, and the fact that everything was just moved around last month!)

```
helm repo add ckotzbauer https://ckotzbauer.github.io/helm-charts
helm install --set nfs.server=192.168.188.190 --set nfs.path=/home/nfs ckotzbauer/nfs-client-provisioner --version 1.0.2 --generate-name
```

For the `nfs.server`, use the Private IPv4 address for your NFS server. For the `nfs.path`, set it to the path of an NFS share on that server.

## Configure Drupal to be able to scale

TODO.

```
# Create a namespace for the Drupal site.
kubectl create namespace drupal

# Create the MySQL (MariaDB) and Drupal (Apache + PHP) Deployments.
kubectl apply -f mariadb.yml -f drupal.yml

# Watch the status of the deployment.
kubectl get deployments -n drupal -w
```

TODO.

```
kubectl get service -n drupal
```

TODO.

## Configure Horizontal Pod Autoscaling

TODO.

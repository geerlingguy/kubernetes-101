# Drupal Kubernetes Deployment Manifests

This directory contains the same deployments from Episode 4, but with some modifications to help scale Drupal.

You can apply these manifests to any Kubernetes cluster (e.g. `minikube start` for a local cluster, or a cloud environment like Linode Kubernetes Engine).

## SEE ALSO

  - NFS: https://github.com/kubernetes-sigs/nfs-ganesha-server-and-external-provisioner
  - Maybe just use EFS ;)

## Configuring a shared filesystem for scalability

TODO: Rook.

First we'll deploy the Rook Ceph cluster operator, which will manage the Ceph cluster, into our Kubernetes cluster:

```
# Download the Rook codebase.
git clone --single-branch --branch release-1.5 https://github.com/rook/rook.git
cd rook/cluster/examples/kubernetes/ceph

# Deploy all the common Rook configuration and operator.
kubectl create -f crds.yaml -f common.yaml -f operator.yaml

# TODO - Might have to `nano operator.yaml` and change `CSI_RBD_GRPC_METRICS_PORT` to `9093` — that is for DigitalOcean...

# Watch for the operator container to be running:
kubectl get pod -n rook-ceph -w
```

Next we'll deploy a Ceph cluster into Kubernetes:

```
# See https://rook.github.io/docs/rook/v1.5/ceph-quickstart.html#cluster-environments

# Create the Ceph cluster (for Linode/cloud environments).
kubectl create -f cluster-on-pvc.yaml
# NOTE: Need to change storageclass from 'gp2' to 'linode-block-storage' in TWO places — see cluster-on-pvc.patch

# Create the Ceph cluster (for test environments like Minikube).
# kubectl create -f cluster-test.yaml

# Watch for all the other pods to be running:
kubectl get pod -n rook-ceph -w
# NOTE: This process can take 5-10 minutes (has to provision block storage, format it, configure it for the CephFS cluster, etc.)
```

Use Rook's toolbox to check on the cluster's health:

```
# Deploy the Interactive toolbox (in this repo):
kubectl create -f rook/toolbox.yml

# Wait for the toolbox to be deployed:
kubectl -n rook-ceph rollout status deploy/rook-ceph-tools

# Once deployed, log into the toolbox with:
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- bash

# Check the status inside the toolbox:
ceph status

(Should see 'HEALTH_OK').
```

Create a Ceph filesystem:

```
# Deploy a Ceph filesystem (in this repo):
kubectl create -f rook/filesystem.yml

# Wait for the pods to be Running:
kubectl -n rook-ceph get pod -l app=rook-ceph-mds
```

Create a StorageClass for Ceph:

```
# Create a CephFS StorageClass (in this repo):
kubectl create -f rook/storageclass.yml
```

## Configure Drupal to use a CephFS PVC

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

CURRENTLY:

  - `kubectl describe pvc -n drupal drupal-files-pvc` never shows it get provisioned
  - `kubectl logs -n rook-ceph -f csi-cephfsplugin-provisioner-7dc78747bf-8gxhh --all-containers --max-log-requests 6` shows that after a timeout period, it just gets stuck in a loop :(
    - See: https://github.com/rook/rook/issues/6183#issuecomment-745060072

TODO: More info here — https://www.digitalocean.com/community/tutorials/how-to-set-up-a-ceph-cluster-within-kubernetes-using-rook

## Configuring Horizontal Pod Autoscaling

TODO.

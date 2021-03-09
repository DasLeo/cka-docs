# Kubernetes Cheat Sheet

## Installation

### Install kubectl on macOS

```shell
curl -LO "https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/darwin/amd64/kubectl" && chmod +x kubectl && sudo mv kubectl /usr/local/bin/
```

## Kubernetes Cluster

### kubeadm YAML config

Examples can be found here:

* <https://github.com/kubernetes/kubeadm/issues/1152>
* <https://github.com/kubernetes/kubernetes/issues/80582>

## kubectl commands

## Maintenance

### Create Snapshots of etcd database

```bash
ETCDPOD=$(kubectl --namespace kube-system get pods -l=component=etcd --template '{{range .items}}{{.metadata.name}}{{"\n"}}{{end}}')
CURDATE=$(date +%m-%d-%y_%H-%M)
kubectl -n kube-system exec -it $ETCDPOD -- sh -c \
"ETCDCTL_API=3 \
ETCDCTL_CACERT=/etc/kubernetes/pki/etcd/ca.crt \
ETCDCTL_CERT=/etc/kubernetes/pki/etcd/server.crt \
ETCDCTL_KEY=/etc/kubernetes/pki/etcd/server.key \
etcdctl snapshot save \
/var/lib/etcd/snapshot-pre-upgrade-$CURDATE"
```

## Authorization

### Checking access to the Cluster

```bash
kubectl auth can-i create deployments
kubectl auth can-i create deployments --as bob
kubectl auth can-i create deployments --as bob --namespace developer
```

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

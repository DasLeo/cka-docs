# Kubernetes Cheat Sheet

## Table of Contents

- [Kubernetes Cheat Sheet](#kubernetes-cheat-sheet)
  - [Table of Contents](#table-of-contents)
  - [Setup Kubernetes Cluster](#setup-kubernetes-cluster)
    - [Setup Internal Network on Hetzner](#setup-internal-network-on-hetzner)
    - [Install latest HWE Kernel](#install-latest-hwe-kernel)
    - [Install and configure CRI-O container runtime](#install-and-configure-cri-o-container-runtime)
      - [Linux Modules and their sysfs configs](#linux-modules-and-their-sysfs-configs)
      - [Ubuntu Packages and repos](#ubuntu-packages-and-repos)
      - [CRI-O Daemon Setup](#cri-o-daemon-setup)
    - [Install Kubernetes binaries](#install-kubernetes-binaries)
    - [Setup hosts file](#setup-hosts-file)
    - [Init the master using kubeadm](#init-the-master-using-kubeadm)
    - [Firewall on Master](#firewall-on-master)
    - [Add Context to local kube config](#add-context-to-local-kube-config)
    - [Join Kubelet as worker node](#join-kubelet-as-worker-node)
      - [Firewall on Worker Node](#firewall-on-worker-node)
      - [Create CA hashed token](#create-ca-hashed-token)
      - [Join using kubeadm](#join-using-kubeadm)
    - [Schedule pods on Control plane node](#schedule-pods-on-control-plane-node)
    - [Install Calido CNI](#install-calido-cni)
      - [Get and adjust Config](#get-and-adjust-config)
      - [Verify node-to-node communication](#verify-node-to-node-communication)
  - [Client Installation](#client-installation)
    - [Install kubectl on macOS](#install-kubectl-on-macos)
    - [Enable Bash completion for Alias 'k'](#enable-bash-completion-for-alias-k)
  - [Kubernetes Cluster](#kubernetes-cluster)
    - [kubeadm YAML config](#kubeadm-yaml-config)
  - [Maintenance](#maintenance)
    - [Upgrade the Cluster](#upgrade-the-cluster)
      - [Determine Packages](#determine-packages)
      - [Upgrade Master Node](#upgrade-master-node)
      - [Upgrade Worker Node](#upgrade-worker-node)
    - [Create Snapshots of etcd database](#create-snapshots-of-etcd-database)
  - [Authorization](#authorization)
    - [Checking access to the Cluster](#checking-access-to-the-cluster)
  - [Working with Pods, Deployments and ReplicaSets](#working-with-pods-deployments-and-replicasets)
    - [Annotate Objects](#annotate-objects)
    - [Scale Deployments](#scale-deployments)
    - [Adjust Deployments](#adjust-deployments)
    - [Record Deployment Changes in an Annotation](#record-deployment-changes-in-an-annotation)
      - [Create Deployment with Change Recording](#create-deployment-with-change-recording)
      - [View recorded Change History](#view-recorded-change-history)
    - [Undo Deployment Rollout](#undo-deployment-rollout)
    - [Delete Deployment & ReplicaSet without touching children](#delete-deployment--replicaset-without-touching-children)
    - [Pause & Resume Deployment](#pause--resume-deployment)
    - [Labels](#labels)
    - [Scheduling and Affinity](#scheduling-and-affinity)
      - [Schedule Pods to specific Nodes using nodeSelector](#schedule-pods-to-specific-nodes-using-nodeselector)
      - [Pod Affinity and Pod AntiAffinity](#pod-affinity-and-pod-antiaffinity)
      - [Pod Affinity Rules and Weights](#pod-affinity-rules-and-weights)
      - [NodeAffinity & NodeAntiAffinity](#nodeaffinity--nodeantiaffinity)
  - [Working with Services, Ingress and CoreDNS](#working-with-services-ingress-and-coredns)
    - [Services](#services)
      - [Expose](#expose)
        - [Add container port to deployment](#add-container-port-to-deployment)
        - [Expose deployment with type cluster ip](#expose-deployment-with-type-cluster-ip)
        - [Expose with type NodePort](#expose-with-type-nodeport)
      - [Check endpoints and services](#check-endpoints-and-services)
    - [CoreDNS](#coredns)
      - [How to dig](#how-to-dig)
      - [How to get all k8s-app pods](#how-to-get-all-k8s-app-pods)
      - [How to map to map nginx.test.io by editing config map of core DNS pod](#how-to-map-to-map-nginxtestio-by-editing-config-map-of-core-dns-pod)
  - [ConfigMaps & Secrets](#configmaps--secrets)
    - [Secrets](#secrets)
  - [Jobs & Cronjobs](#jobs--cronjobs)
    - [Create Job](#create-job)
    - [Create a Cronjob](#create-a-cronjob)
  - [Resource Limits](#resource-limits)
    - [Limit resources in a Deployment](#limit-resources-in-a-deployment)
    - [Limit resources in a Namespace](#limit-resources-in-a-namespace)
  - [Kubernetes API](#kubernetes-api)
    - [Make API calls using curl](#make-api-calls-using-curl)
      - [Create a new pod using curl](#create-a-new-pod-using-curl)
    - [Access Cluster Certificates from inside Pods](#access-cluster-certificates-from-inside-pods)
    - [Access Cluster API Server using Proxy](#access-cluster-api-server-using-proxy)
  - [Troubleshoot](#troubleshoot)
    - [Ubuntu/alpine Pod for debugging](#ubuntualpine-pod-for-debugging)
  - [Command Line Hacks](#command-line-hacks)
    - [Create yaml files on the fly](#create-yaml-files-on-the-fly)
  - [YAML Templates](#yaml-templates)
    - [Deployment](#deployment)
    - [ReplicaSet](#replicaset)
    - [DaemonSet](#daemonset)
    - [ConfigMap](#configmap)
    - [Secret](#secret)
    - [Pod with ConfigMap](#pod-with-configmap)
    - [ClusterRole and ClusterRoleBinding for traefik](#clusterrole-and-clusterrolebinding-for-traefik)
    - [traefik ingress controller + ingress service + service account](#traefik-ingress-controller--ingress-service--service-account)
  - [Useless Chapters](#useless-chapters)

## Setup Kubernetes Cluster

### Setup Internal Network on Hetzner

Disable cloudInit first by adding the file `/etc/cloud/cloud.cfg.d/99-disable-network-config.cfg`:

```/etc/cloud/cloud.cfg.d/99-disable-network-config.cfg
network:
  config: disabled
```

```/etc/network/interfaces.d/90-ens10.cfg
auto ens10
iface ens10 inet static
  address 172.16.0.2
  netmask 255.255.255.0
  broadcast 172.16.0.255
```

Also make sure that `/etc/network/interfaces.d/50-cloud-init.cfg` is present and working.

### Install latest HWE Kernel

```bash
apt-get install --install-recommends linux-generic-hwe-18.04
```

### Install and configure CRI-O container runtime

See also: <https://kubernetes.io/docs/setup/production-environment/container-runtimes/#cri-o>

#### Linux Modules and their sysfs configs

```bash
# Create the .conf file to load the modules at bootup
cat <<EOF | sudo tee /etc/modules-load.d/crio.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# Set up required sysctl params, these persist across reboots.
cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF

sudo sysctl --system
```

#### Ubuntu Packages and repos

Set proper Operating system variable `$OS`

- Ubuntu 20.04    xUbuntu_20.04
- Ubuntu 19.10    xUbuntu_19.10
- Ubuntu 19.04    xUbuntu_19.04
- Ubuntu 18.04    xUbuntu_18.04

Set proper `$VERSION` variable either `VERSION=1.20` or `VERSION=1.20:1.20.0` to pin a specifc version.

```bash
export OS=xUbuntu_18.04
export VERSION=1.20

cat <<EOF | sudo tee /etc/apt/sources.list.d/devel:kubic:libcontainers:stable.list
deb https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable/$OS/ /
EOF
cat <<EOF | sudo tee /etc/apt/sources.list.d/devel:kubic:libcontainers:stable:cri-o:$VERSION.list
deb http://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable:/cri-o:/$VERSION/$OS/ /
EOF

curl -L https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable/$OS/Release.key | sudo apt-key --keyring /etc/apt/trusted.gpg.d/libcontainers.gpg add -
curl -L https://download.opensuse.org/repositories/devel:kubic:libcontainers:stable:cri-o:$VERSION/$OS/Release.key | sudo apt-key --keyring /etc/apt/trusted.gpg.d/libcontainers-cri-o.gpg add -

sudo apt-get update
sudo apt-get install cri-o cri-o-runc
# conntrack don't know if it's required but in CRI-O logs you can see warnings and "things" that can happen
sudo apt install conntrack
# to prevent version mismatch of CRI-O and Kubernetes packages
sudo apt-mark hold cri-o cri-o-runc
```

#### CRI-O Daemon Setup

Next add the `conmon` binary path to `/etc/crio/crio.conf`
Then add image registries to `/etc/containers/registries.conf`
CRI-O uses systemd as cgroup driver by default, but if this ever changes, we need to set `cgroup_manager = "systemd"` and `conmon_cgroup="system.slice"` as well.

```bash
which conmon
```

```/etc/crio/crio.conf
conmon = "/usr/bin/conmon"
```

```/etc/containers/registries.conf
unqualified-search-registries = ["docker.io", "quay.io"]
```

After that we can start the systemd services

```bash
sudo systemctl daemon-reload
sudo systemctl enable crio --now
```

### Install Kubernetes binaries

```bash
sudo apt-get update && sudo apt-get install -y apt-transport-https curl
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
cat <<EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

### Setup hosts file

In Ubuntu /etc/hosts will be generated by the cloudInit module "Update Etc Hosts"
<https://cloudinit.readthedocs.io/en/latest/topics/modules.html#update-etc-hosts>

```/etc/cloud/templates/hosts.debian.tmpl
# ...
# ...
# ...
127.0.0.1 localhost
172.16.0.2 k8smaster
172.16.0.3 k8sworker1
# ...
# ...
# ...
```

```/etc/hosts
# ...
# ...
# ...
127.0.0.1 localhost
172.16.0.2 k8smaster
172.16.0.3 k8sworker1
# ...
# ...
# ...
```

### Init the master using kubeadm

As passing command line arguments is deprecated, we'll create a yaml file for the init process. `cgroupDriver` is the most important parameter when using CRI-O to make sure kubelet and CRI-O are using the same cgroup driver.

```kubeadm-init-config.yml
---
apiVersion: kubeadm.k8s.io/v1beta2
kind: InitConfiguration
nodeRegistration:
  criSocket: "unix:///var/run/crio/crio.sock"
  kubeletExtraArgs:
    cgroup-driver: "systemd"
---
apiVersion: kubeadm.k8s.io/v1beta2
kind: ClusterConfiguration
kubernetesVersion: 1.20.4
controlPlaneEndpoint: "k8smaster:6443"
networking:
  podSubnet: 192.168.0.0/16
---
apiVersion: kubelet.config.k8s.io/v1beta2
kind: KubeletConfiguration
cgroupDriver: systemd
```

### Firewall on Master

Resources:

- <https://stackoverflow.com/questions/60708270/how-can-i-use-flannel-without-disabing-firewalld-kubernetes>
- <https://medium.com/platformer-blog/kubernetes-on-centos-7-with-firewalld-e7b53c1316af>
- <https://github.com/projectcalico/calico/issues/1214>

```bash
firewall-cmd --permanent --zone=public --set-target=DROP
firewall-cmd --permanent --zone=public --add-interface=eth0
firewall-cmd --permanent --zone=internal --add-interface=ens10
firewall-cmd --permanent --zone=internal --set-target=DROP

firewall-cmd --permanent --new-service=kubernetes-api-server
firewall-cmd --permanent --service=kubernetes-api-server --set-description='Kubernetes API server also used by kubectl'
firewall-cmd --permanent --service=kubernetes-api-server --set-short='Kubernetes API server'
firewall-cmd --permanent --service=kubernetes-api-server --add-port=6443/tcp

firewall-cmd --permanent --new-service=etcd-server-client-api
firewall-cmd --permanent --service=etcd-server-client-api --set-description='etcd server client API utilized by kube-apiserver'
firewall-cmd --permanent --service=etcd-server-client-api --set-short='etcd server client API'
firewall-cmd --permanent --service=etcd-server-client-api --add-port=2379-2380/tcp

firewall-cmd --permanent --new-service=kubelet-api
firewall-cmd --permanent --service=kubelet-api --set-description='kubelet API utilized by Control plane'
firewall-cmd --permanent --service=kubelet-api --set-short='kubelet API'
firewall-cmd --permanent --service=kubelet-api --add-port=10250/tcp

firewall-cmd --permanent --new-service=kube-scheduler
firewall-cmd --permanent --service=kube-scheduler --set-description='kube-scheduler'
firewall-cmd --permanent --service=kube-scheduler --set-short='kube-scheduler'
firewall-cmd --permanent --service=kube-scheduler --add-port=10252/tcp

firewall-cmd --permanent --new-service=kube-controller-manager
firewall-cmd --permanent --service=kube-controller-manager --set-description='kube-controller-manager'
firewall-cmd --permanent --service=kube-controller-manager --set-short='kube-controller-manager'
firewall-cmd --permanent --service=kube-controller-manager --add-port=10251/tcp

firewall-cmd --permanent --new-service=nodeport-services-tcp
firewall-cmd --permanent --service=nodeport-services-tcp --set-description='TCP Range for NodePort Services'
firewall-cmd --permanent --service=nodeport-services-tcp --set-short='TCP Range for NodePort Services'
firewall-cmd --permanent --service=nodeport-services-tcp --add-port=30000-32767/tcp

firewall-cmd --permanent --new-service=nodeport-services-udp
firewall-cmd --permanent --service=nodeport-services-udp --set-description='UDP Range for NodePort Services'
firewall-cmd --permanent --service=nodeport-services-udp --set-short='UDP Range for NodePort Services'
firewall-cmd --permanent --service=nodeport-services-udp --add-port=30000-32767/udp

firewall-cmd --permanent --zone=public --add-service=kubernetes-api-server

firewall-cmd --permanent --new-zone=k8s-internal-lan
firewall-cmd --permanent --zone=k8s-internal-lan --add-source=172.16.0.0/24
firewall-cmd --permanent --zone=k8s-internal-lan --add-service=kubernetes-api-server
firewall-cmd --permanent --zone=k8s-internal-lan --add-service=kubelet-api
firewall-cmd --permanent --zone=k8s-internal-lan --add-service=nodeport-services-tcp
firewall-cmd --permanent --zone=k8s-internal-lan --add-service=nodeport-services-udp

# For calico CNI without IPIP or VXLAN as Cross-Subnet
firewall-cmd --permanent --zone=k8s-internal-lan --add-service=bgp

# Maybe required
# firewall-cmd --add-masquerade --permanent --zone=internal
# firewall-cmd --add-masquerade --permanent --zone=public ???

firewall-cmd --reload
systemctl restart kubelet
```

### Add Context to local kube config

Convert all base64 encoded certificates first

```bash
echo 'LS0xxxxxxx==' | base64 -d >> ca.crt
echo 'LS0xxxxxxx==' | base64 -d >> user.crt
echo 'LS0xxxxxxx==' | base64 -d >> user.key

kubectl config set-cluster cluster-cka-lab --embed-certs --server=https://65.21.60.29:6443 --certificate-authority=ca.crt
k config set-credentials admin-cka-lab --embed-certs --client-certificate=./user.crt --client-key=./user.key
k config set-context cka-lab --cluster cluster-cka-lab --user admin-cka-lab
```

### Join Kubelet as worker node

#### Firewall on Worker Node

Resources:

- <https://stackoverflow.com/questions/60708270/how-can-i-use-flannel-without-disabing-firewalld-kubernetes>
- <https://medium.com/platformer-blog/kubernetes-on-centos-7-with-firewalld-e7b53c1316af>
- <https://github.com/projectcalico/calico/issues/1214>

```bash
apt install firewalld -y
systemctl enable firewalld --now

firewall-cmd --permanent --zone=public --set-target=DROP
firewall-cmd --permanent --zone=public --add-icmp-block-inversion
for i in address-unreachable bad-header beyond-scope communication-prohibited destination-unreachable echo-reply echo-request failed-policy fragmentation-needed host-precedence-violation host-prohibited host-redirect host-unknown host-unreachable ip-header-bad neighbour-advertisement neighbour-solicitation network-prohibited network-redirect network-unknown network-unreachable no-route packet-too-big parameter-problem port-unreachable precedence-cutoff protocol-unreachable redirect reject-route required-option-missing router-advertisement router-solicitation source-quench source-route-failed time-exceeded timestamp-reply timestamp-request tos-host-redirect tos-host-unreachable tos-network-redirect tos-network-unreachable ttl-zero-during-reassembly ttl-zero-during-transit unknown-header-type unknown-option; do firewall-cmd --permanent --zone=public --add-icmp-block=$i; done

#firewall-cmd --permanent --zone=public --add-icmp-block=echo-reply
#firewall-cmd --permanent --zone=public --add-icmp-block=echo-request
#firewall-cmd --permanent --zone=public --add-icmp-block=time-exceeded
#firewall-cmd --permanent --zone=public --add-icmp-block=port-unreachable
#firewall-cmd --permanent --zone=public --add-icmp-block=fragmentation-needed
#firewall-cmd --permanent --zone=public --add-icmp-block=packet-too-big


firewall-cmd --permanent --zone=public --add-interface=eth0
firewall-cmd --permanent --zone=internal --add-interface=ens10
firewall-cmd --permanent --zone=internal --set-target=DROP
firewall-cmd --permanent --zone=internal --add-icmp-block-inversion
for i in address-unreachable bad-header beyond-scope communication-prohibited destination-unreachable echo-reply echo-request failed-policy fragmentation-needed host-precedence-violation host-prohibited host-redirect host-unknown host-unreachable ip-header-bad neighbour-advertisement neighbour-solicitation network-prohibited network-redirect network-unknown network-unreachable no-route packet-too-big parameter-problem port-unreachable precedence-cutoff protocol-unreachable redirect reject-route required-option-missing router-advertisement router-solicitation source-quench source-route-failed time-exceeded timestamp-reply timestamp-request tos-host-redirect tos-host-unreachable tos-network-redirect tos-network-unreachable ttl-zero-during-reassembly ttl-zero-during-transit unknown-header-type unknown-option; do firewall-cmd --permanent --zone=internal --add-icmp-block=$i; done

firewall-cmd --permanent --new-service=kubernetes-api-server
firewall-cmd --permanent --service=kubernetes-api-server --set-description='Kubernetes API server also used by kubectl'
firewall-cmd --permanent --service=kubernetes-api-server --set-short='Kubernetes API server'
firewall-cmd --permanent --service=kubernetes-api-server --add-port=6443/tcp

firewall-cmd --permanent --new-service=etcd-server-client-api
firewall-cmd --permanent --service=etcd-server-client-api --set-description='etcd server client API utilized by kube-apiserver'
firewall-cmd --permanent --service=etcd-server-client-api --set-short='etcd server client API'
firewall-cmd --permanent --service=etcd-server-client-api --add-port=2379-2380/tcp

firewall-cmd --permanent --new-service=kubelet-api
firewall-cmd --permanent --service=kubelet-api --set-description='kubelet API utilized by Control plane'
firewall-cmd --permanent --service=kubelet-api --set-short='kubelet API'
firewall-cmd --permanent --service=kubelet-api --add-port=10250/tcp

firewall-cmd --permanent --new-service=kube-scheduler
firewall-cmd --permanent --service=kube-scheduler --set-description='kube-scheduler'
firewall-cmd --permanent --service=kube-scheduler --set-short='kube-scheduler'
firewall-cmd --permanent --service=kube-scheduler --add-port=10252/tcp

firewall-cmd --permanent --new-service=kube-controller-manager
firewall-cmd --permanent --service=kube-controller-manager --set-description='kube-controller-manager'
firewall-cmd --permanent --service=kube-controller-manager --set-short='kube-controller-manager'
firewall-cmd --permanent --service=kube-controller-manager --add-port=10251/tcp

firewall-cmd --permanent --new-service=nodeport-services-tcp
firewall-cmd --permanent --service=nodeport-services-tcp --set-description='TCP Range for NodePort Services'
firewall-cmd --permanent --service=nodeport-services-tcp --set-short='TCP Range for NodePort Services'
firewall-cmd --permanent --service=nodeport-services-tcp --add-port=30000-32767/tcp

firewall-cmd --permanent --new-service=nodeport-services-udp
firewall-cmd --permanent --service=nodeport-services-udp --set-description='UDP Range for NodePort Services'
firewall-cmd --permanent --service=nodeport-services-udp --set-short='UDP Range for NodePort Services'
firewall-cmd --permanent --service=nodeport-services-udp --add-port=30000-32767/udp

firewall-cmd --permanent --new-zone=k8s-internal-lan
firewall-cmd --permanent --zone=k8s-internal-lan --add-icmp-block-inversion
for i in address-unreachable bad-header beyond-scope communication-prohibited destination-unreachable echo-reply echo-request failed-policy fragmentation-needed host-precedence-violation host-prohibited host-redirect host-unknown host-unreachable ip-header-bad neighbour-advertisement neighbour-solicitation network-prohibited network-redirect network-unknown network-unreachable no-route packet-too-big parameter-problem port-unreachable precedence-cutoff protocol-unreachable redirect reject-route required-option-missing router-advertisement router-solicitation source-quench source-route-failed time-exceeded timestamp-reply timestamp-request tos-host-redirect tos-host-unreachable tos-network-redirect tos-network-unreachable ttl-zero-during-reassembly ttl-zero-during-transit unknown-header-type unknown-option; do firewall-cmd --permanent --zone=k8s-internal-lan --add-icmp-block=$i; done
firewall-cmd --permanent --zone=k8s-internal-lan --add-source=172.16.0.0/24
firewall-cmd --permanent --zone=k8s-internal-lan --add-service=kubelet-api
firewall-cmd --permanent --zone=k8s-internal-lan --add-service=nodeport-services-tcp
firewall-cmd --permanent --zone=k8s-internal-lan --add-service=nodeport-services-udp

# For calico CNI without IPIP or VXLAN as Cross-Subnet
firewall-cmd --permanent --zone=k8s-internal-lan --add-service=bgp

# Maybe required
# firewall-cmd --add-masquerade --permanent --zone=internal
# firewall-cmd --add-masquerade --permanent --zone=public ???

firewall-cmd --reload
systemctl restart kubelet
```

#### Create CA hashed token

```bash
openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | openssl dgst -sha256 -hex | sed 's/^.* //'
```

#### Join using kubeadm

As we use CRI-O the Kubelet CGroup Driver must match the one CRI-O uses in `/etc/crio/crio.conf`
See also:

- <https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/kubelet-integration/#providing-instance-specific-configuration-details>
- <https://pkg.go.dev/k8s.io/kubernetes/pkg/kubelet/apis/config#KubeletConfiguration>

To archive that we need to add all our `kubeadm join` parameters to a yaml file:
Get the defaults by executing:

```bash
kubeadm config print init-defaults --component-configs KubeletConfiguration
```

```kubeadm-join-config.yml
apiVersion: kubeadm.k8s.io/v1beta2
kind: JoinConfiguration
nodeRegistration:
  criSocket: "unix:///var/run/crio/crio.sock"
  kubeletExtraArgs:
    cgroup-driver: "systemd"
discovery:
  bootstrapToken:
    token: "xxxxx.xxxxxxxxxxxx"
    apiServerEndpoint: "k8smaster:6443"
    caCertHashes:
      - "sha256:xxxxxxxxxxxxx"
---
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
cgroupDriver: systemd
```

Alternatively you can add the parameter `--cgroup-driver=systemd` to the file `/var/lib/kubelet/kubeadm-flags.env` but this is deprecated.
Since kubeadm 1.19 the value will be written to `/var/lib/kubelet/config.yaml` during a join/init process

```bash
kubeadm join --config kubeadm-join-config.yml
```

### Schedule pods on Control plane node

To remove `node-role.kubernetes.io/master` from all nodes:

```bash
kubectl taint nodes --all node-role.kubernetes.io/master-
```

### Install Calido CNI

#### Get and adjust Config

```bash
wget https://docs.projectcalico.org/manifests/calico.yaml
```

```calico.yaml
            # Enable IPIP
            - name: CALICO_IPV4POOL_IPIP
              value: "Never"


            # The default IPv4 pool to create on startup if none exists. Pod IPs will be
            # chosen from this range. Changing this value after installation will have
            # no effect. This should fall within `--cluster-cidr`.
            - name: CALICO_IPV4POOL_CIDR
              value: "192.168.0.0/16"
```

```bash
kubectl apply -f calico.yaml
```

#### Verify node-to-node communication

```bash
kubectl create deployment shell-demo --image nginx
kubectl get pods --selector app=shell-demo
kubectl exec --stdin --tty shell-demo-xxxxxxx-xxxxx -- /bin/bash

apk add telnet
telnet OtherPodRunningNginx 443
```

## Client Installation

### Install kubectl on macOS

```shell
curl -LO "https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/darwin/amd64/kubectl" && chmod +x kubectl && sudo mv kubectl /usr/local/bin/
```

### Enable Bash completion for Alias 'k'

```bash
echo 'alias ks=kubectl' >>~/.bashrc
echo 'complete -F __start_kubectl k' >>~/.bashrc
```

## Kubernetes Cluster

### kubeadm YAML config

Examples can be found here:

- <https://github.com/kubernetes/kubeadm/issues/1152>
- <https://github.com/kubernetes/kubernetes/issues/80582>

## Maintenance

### Upgrade the Cluster

#### Determine Packages

Update packages and show all available versions for kubeadm.

```bash
apt update
apt-cache madison kubeadm

# Show current version
kubeadm version
```

#### Upgrade Master Node

```bash
apt-mark unhold kubeadm
apt-get install -y kubeadm=1.20.4-00
apt-mark hold kubeadm
# Show installed version
kubeadm version

# Drain master
kubectl drain MASTTERNODE-NAME --ignore-daemonsets

# Plan Master Upgrade
kubeadm upgrade plan

# Do Upgrade
kubeadm upgrade apply v1.20.4

# Determine Kubelet version
kubectl get node

apt-mark unhold kubelet kubectl
apt-get install -y kubelet=1.20.4-00 kubectl=1.20.4-00
apt-mark hold kubelet kubectl

systemctl daemon-reload
systemctl restart kubelet

# Verify Kubelet version
kubectl get node

# Make Node available again
kubectl uncordon MASTTERNODE-NAME
# Check if ready
kubectl get node
```

#### Upgrade Worker Node

```bash
apt-mark unhold kubeadm
apt update
apt-get install -y kubeadm=1.20.4-00
apt-mark hold kubeadm
kubectl drain WORKERNODE-NAME --ignore-daemonsets
kubeadm upgrade node

apt-mark unhold kubelet kubectl
apt-get install -y kubelet=1.20.4-00 kubectl=1.20.4-00
apt-mark hold kubelet kubectl
systemctl daemon-reload
systemctl restart kubelet

# Verify Node Status
kubectl get nodes
kubectl uncordon WORKERNODE-NAME
kubectl get nodes
```

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

## Working with Pods, Deployments and ReplicaSets

### Annotate Objects

You can add annotations to nearly every API object in Kubernetes.

```bash
kubectl annotate pods --all description='Production Pods' -n prod 
kubectl annotate --overwrite pod webpod description="Old Production Pods" -n prod 
kubectl -n prod annotate pod webpod description-
```

### Scale Deployments

```bash
kubectl scale deploy/dev-web --replicas=4
kubectl scale deployment dev-web --replicas=4
```

### Adjust Deployments

```bash
kubectl edit deployment nginx
```

### Record Deployment Changes in an Annotation

#### Create Deployment with Change Recording

To record last changes in the Annotation `kubernetes.io/change-cause`

> :warning: **[Bug: 81289](https://github.com/kubernetes/kubernetes/issues/81289)**: No longer working in `create`. Needs workaround using apply!

```bash
# No longer working command:
kubectl create deploy ghost --image=ghost --record

# Workaround:
kubectl create deploy ghost --image=ghost
kubectl get deployments.apps ghost -o yaml >> ghost.yml

kubectl apply -f ghost.yml --record

# Or during an edit
kubectl edit deploy ghost --record
```

#### View recorded Change History

```bash
kubectl rollout history deployment ghost

REVISION  CHANGE-CAUSE
1         kubectl apply --filename=ghost.yml --record=true
2         kubectl apply --filename=ghost.yml --record=true
```

### Undo Deployment Rollout

```bash
kubectl rollout undo deployment/ghost

# or to a specific revision
kubectl rollout undo deployment/ghost --to-revision=2
```

### Delete Deployment & ReplicaSet without touching children

> :warning: **--cascade=false is deprecated**: Use `--cascade=orphan` now.

```bash
kubectl delete deployment nginx --cascade=orphan
```

### Pause & Resume Deployment

```bash
kubectl rollout pause deployment/ghost
kubectl rollout resume deployment/ghost
```

### Labels

```bash
# get all pods filtered by label run=ghost. This can of course be done with all objects like nodes etc. You can also use delete for creating by label
kubectl get pods -l run=ghost 
# get all pods in the 'default' namespace & show Column with label 'app'
kubectl get pods -L app
# get all pods with all labels
kubectl get pods --show-labels
#Remove system label from node
kubectl label node worker system-

# Add Label on the fly
kubectl label pods ghost-3378155678-eq5i6 foo=bar
```

### Scheduling and Affinity

#### Schedule Pods to specific Nodes using nodeSelector

```yaml
spec:
    containers:
    - image: nginx
    nodeSelector:
        disktype: ssd
```

#### Pod Affinity and Pod AntiAffinity

```yaml
apiVersion: apps/v1
kind: Deployment
spec:
  template:
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - store
            topologyKey: "kubernetes.io/hostname"
```

#### Pod Affinity Rules and Weights

> The scheduler tries to choose, or avoid the node with the greatest combined value.

```yaml
podAntiAffinity:
  preferredDuringSchedulingIgnoredDuringExecution:
  - weight: 100
    podAffinityTerm:
      labelSelector:
        matchExpressions:
        - key: security
          operator: In
          values:
          - S2
    topologyKey: kubernetes.io/hostname
  - weight: 90
    podAffinityTerm:
      labelSelector:
        matchExpressions:
        - key: ssd
          operator: In
          values:
          - Gold
    topologyKey: kubernetes.io/hostname
```

#### NodeAffinity & NodeAntiAffinity

> This will replace `nodeSelector` in the future

- Operators: `In, NotIn, Exists, DoesNotExist`
- Rule types:
  - `requiredDuringSchedulingIgnoredDuringExecution`
  - `preferredDuringSchedulingIgnoredDuringExecution`
  - Planned for future: `requiredDuringSchedulingRequiredDuringExecution`

```yaml
apiVersion: apps/v1
kind: Deployment
spec:
  template:
    spec:
      affinity:
        nodeAffinity: 
          requiredDuringSchedulingIgnoredDuringExecution:  
            nodeSelectorTerms:
            - matchExpressions:
              - key: kubernetes.io/colo-tx-name
                operator: In
                values:
                - tx-aus
                - tx-dal
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 1
            preference:
              matchExpressions:
              - key: disk-speed
                operator: In
                values:
                - fast
                - quick
```

## Working with Services, Ingress and CoreDNS

### Services

#### Expose

##### Add container port to deployment

```yaml
  spec:
    containers:
    - image: nginx
      ports:
      - containerPort: 80
        protocol: TCP
```

##### Expose deployment with type cluster ip

```bash
kubectl expose deployment nginx-one

#Cluster ip can be accessed from inside the cluster with endpoint ip and port 80
```

##### Expose with type NodePort

```bash
  kubectl expose deployment nginx-one --type=NodePort --name=service-lab

  #Node Port can be accessed from outside the cluster with the exposed high port
```

#### Check endpoints and services

```bash
kubectl get svc nginx-one

kubectl get ep nginx-one
```

### CoreDNS

#### How to dig

1. Create an ubuntu/alpine pod
2. Exec into the pod with ```kubectl exec -it alpine sh```
3. Install ```apk add; apk add curl bind```
4. ```dig```
5. ```dig @10.96.0.10 -x 10.96.0.10```
6. curl service-lab.accounting.svc.cluster.local.
7. curl service-lab
8. curl service-lab.accounting

#### How to get all k8s-app pods

```bash
kubectl get pod -l k8s-app --all-namespaces
```

#### How to map to map nginx.test.io by editing config map of core DNS pod

```bash
#Show configmap of coredns
kubectl -n kube-system get configmaps coredns -o yaml
#Edit configmap of coredns 
kubectl -n kube-system edit configmaps coredns
#Add the following to configmap under :53
rewrite name regex (.*)\.test\.io {1}.default.svc.cluster.local
#Delete all coredns pods
k delete pods -l k8s-app=kube-dns --all-namespaces
#Now inside the container this should work
dig nginx.test.io
#Eventually configure answer with rewrite stop
rewrite stop {
  name regex (.*)\.test\.io {1}.default.svc.cluster.local
  answer name (.*)\.default\.svc\.cluster\.local {1}.test.io
}
```

## ConfigMaps & Secrets

### Secrets

- Secrets are limited to 1MB in size as they will be saved to etcd

## Jobs & Cronjobs

### Create Job

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: sleepy
spec:
  completions: 5
  parallelism: 2
  activeDeadlineSeconds: 20
  template:
    spec:
      containers:
      - name: resting
        image: busybox
        command: ["/bin/sleep"]
        args: ["5"]
      restartPolicy: Never
```

### Create a Cronjob

```yaml
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: test
spec:
  schedule: '*/2 * * * *'
  jobTemplate:
    metadata:
      name: test
    spec:
      template:
        spec:
          containers:
          - image: busybox
            name: resting
            command: 
              - "/bin/sleep"
            args: ["5"]
          restartPolicy: Never
```

## Resource Limits

### Limit resources in a Deployment

> :warning: **In case of namespace limits**: Deployment resources overwrite namespace limits!

```yaml
apiVersion: apps/v1
kind: Deployment
spec:
  template:
    spec:
      containers:
      - image: vish/stress
        name: stress
        resources:
          limits:
            memory: 1Gi
          requests:
            memory: 500Mi
        args:
          - -cpus
          - "2"
          - -mem-total
          - "950Mi"
          - -mem-alloc-size
          - "100Mi"
          - -mem-alloc-sleep
          - "1s"
```

### Limit resources in a Namespace

```bash
kubectl create namespace low-usage-limit
```

Create a LimitRange Object for a namespace

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: low-resource-range
  #namespace: low-usage-limit
spec:
  limits:
    - default:
        cpu: 1
        memory: 500Mi
      defaultRequest:
        cpu: 0.5
        memory: 100Mi
      type: Container
```

```bash
kubectl -n low-usage-limit create -f limitrange.yml
```

## Kubernetes API

### Make API calls using curl

This method works only if you have only one Cluster in your `.kube/config`

```bash
export client=$(grep client-cert $HOME/.kube/config |cut -d" " -f 6)
export key=$(grep client-key-data $HOME/.kube/config |cut -d " " -f 6)
export auth=$(grep certificate-authority-data $HOME/.kube/config |cut -d " " -f 6)

echo $client | base64 -d - > ./client.pem
echo $key | base64 -d - > ./client-key.pem
echo $auth | base64 -d - > ./ca.pem

# Determine Servers address
kubectl config view | grep server

# Get Pods
curl --cert ./client.pem \
--key ./client-key.pem \
--cacert ./ca.pem \
https://k8smaster:6443/api/v1/pods
```

#### Create a new pod using curl

```json
{
  "kind": "Pod",
  "apiVersion": "v1",
  "metadata":{
    "name": "curlpod",
    "namespace": "default",
    "labels": {
      "name": "examplepod"
    }
  },
  "spec": {
    "containers": [{
      "name": "nginx",
      "image": "nginx",
      "ports": [{"containerPort": 80}]
    }]
  }
}
```

```bash
curl --cert ./client.pem \
--key ./client-key.pem --cacert ./ca.pem \
https://k8smaster:6443/api/v1/namespaces/default/pods \
-XPOST -H
'
Content-Type: application/json
'
\
-d@curlpod.json
```

### Access Cluster Certificates from inside Pods

You can find all Cluster Certs under `/var/run/secrets/kubernetes.io/serviceaccount/` from inside every Pod.

### Access Cluster API Server using Proxy

```bash
kubectl proxy --api-prefix=/
curl http://127.0.0.1:8001/api/
curl http://127.0.0.1:8001/api/v1/namespaces

# use custom prefix
kubectl proxy --api-prefix=/k8s
curl http://127.0.0.1:8001/k8s/api/
curl http://127.0.0.1:8001/k8s/api/v1/namespaces
```

## Troubleshoot

### Ubuntu/alpine Pod for debugging

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nettool
spec:
  containers:
  - name: alpine
    image: alpine:latest
    command: [ "sleep"  ]
    args: [ "infinity" ]
```

## Command Line Hacks

### Create yaml files on the fly

```bash
kubectl create [deployment|pods|replicaset|secret|configmap] NAME --dry-run=client -o yaml | vim -
kubectl expose deployment NAME --port 80 --type NodePort --output yaml --dry-run=client | vim -
```

## YAML Templates

### Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: nginx
  name: nginx
  namespace: default
spec:
  progressDeadlineSeconds: 600
  replicas: 1
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      app: nginx
  strategy:
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
    type: RollingUpdate
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - image: nginx
        imagePullPolicy: Always
        name: nginx
        resources: {}
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext: {}
      terminationGracePeriodSeconds: 30

```

### ReplicaSet

*Note:* ReplicaSets do not have an update strategy like Replication Controllers. Use a Deployment in case you need rolling updates.

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: rs-one
  namespace: default
spec:
  replicas: 2
  selector:
    matchLabels:
      system: ReplicaOne
  template:
    metadata:
      labels:
        system: ReplicaOne
    spec:
      containers:
      - image: nginx:1.15.1
        imagePullPolicy: Always
        name: nginx
        ports:
        - containerPort: 80
      restartPolicy: Always

```

### DaemonSet

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: ds-one 
  namespace: default
spec:
  selector:
    matchLabels:
      system: DaemonSetOne
  template:
    metadata:
      name: ds-pod
      labels:
        system: DaemonSetOne
    spec:
      containers:
      - image: nginx:1.15.1
        imagePullPolicy: Always
        name: nginx
        ports:
        - containerPort: 80
        # Env vars from Secret
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            name: mysql
              secretKeyRef:
                  name: mysql
                  key: password
        # Mount Secrets as Files
        volumeMounts:
        - name: service-key
          mountPath: /root/key.json
          subPath: key.json
      restartPolicy: Always
      volumes:
      - name: service-key
        secret:
          secretName: my-secret
          items:
          - key: service-account-key
            path: key.json
  updateStrategy:
    rollingUpdate:
      maxUnavailable: 1
    type: OnDelete
```

### ConfigMap

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: fast-car
  namespace: default
data:
  car.make: Opel
  car.model: Speedster
  car.trim: Tiefergelegt
```

### Secret

> Secrets are `base64` encoded

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: lf-secret
data:
  password: TEZUckAxbgo=
```

### Pod with ConfigMap

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: shell-demo
  namespace: default
spec:
  containers:
  - image: nginx
    name: nginx
    env:
    - name: ilike
      valueFrom:
        configMapKeyRef:
          name: colors
          key: favorite
    envFrom:
    - configMapRef:
        name: colors
    volumeMounts:
    - name: car-vol
      mountPath: /etc/car
  volumes:
  - name: car-vol
    configMap:
      name: fast-car
```

### ClusterRole and ClusterRoleBinding for traefik

```yaml
ind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: traefik-ingress-controller
rules:
  - apiGroups:
      - ""
    resources:
      - services
      - endpoints
      - secrets
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - extensions
    resources:
      - ingresses
    verbs:
      - get
      - list
      - watch
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: traefik-ingress-controller
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: traefik-ingress-controller
subjects:
- kind: ServiceAccount
  name: traefik-ingress-controller
  namespace: kube-system
```

### traefik ingress controller + ingress service + service account

```yaml
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: traefik-ingress-controller
  namespace: kube-system
---
kind: DaemonSet
apiVersion: apps/v1
metadata:
  name: traefik-ingress-controller
  namespace: kube-system
  labels:
    k8s-app: traefik-ingress-lb
spec:
  selector:
    matchLabels:
      name: traefik-ingress-lb
  template:
    metadata:
      labels:
        k8s-app: traefik-ingress-lb
        name: traefik-ingress-lb
    spec:
      serviceAccountName: traefik-ingress-controller
      terminationGracePeriodSeconds: 60
      hostNetwork: True
      containers:
      - image: traefik:1.7.13
        name: traefik-ingress-lb
        ports:
        - name: http
          containerPort: 80
          hostPort: 80
        - name: admin
          containerPort: 8080
          hostPort: 8080
        args:
        - --api
        - --kubernetes
        - --logLevel=INFO
---
kind: Service
apiVersion: v1
metadata:
  name: traefik-ingress-service
  namespace: kube-system
spec:
  selector:
    k8s-app: traefik-ingress-lb
  ports:
    - protocol: TCP
      port: 80
      name: web
    - protocol: TCP
      port: 8080
      name: admin
```

## Useless Chapters

Additional Links of Chapters we assume are not neccecary for the exam:

- [Explore API Calls](https://trainingportal.linuxfoundation.org/learn/course/kubernetes-fundamentals-lfs258/apis-and-access/lab-exercises?page=2)

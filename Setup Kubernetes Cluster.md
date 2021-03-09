# Setup Kubernetes Cluster

## Table of Contents

- [Setup Kubernetes Cluster](#setup-kubernetes-cluster)
  - [Table of Contents](#table-of-contents)
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
  - [Maintenance](#maintenance)
    - [Create Snapshots of etcd database](#create-snapshots-of-etcd-database)

## Setup Internal Network on Hetzner

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

## Install latest HWE Kernel

```bash
apt-get install --install-recommends linux-generic-hwe-18.04
```

## Install and configure CRI-O container runtime

See also: <https://kubernetes.io/docs/setup/production-environment/container-runtimes/#cri-o>

### Linux Modules and their sysfs configs

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

### Ubuntu Packages and repos

Set proper Operating system variable `$OS`

* Ubuntu 20.04    xUbuntu_20.04
* Ubuntu 19.10    xUbuntu_19.10
* Ubuntu 19.04    xUbuntu_19.04
* Ubuntu 18.04    xUbuntu_18.04

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

### CRI-O Daemon Setup

Next add the `conmon` binary path and the docker.io registry to `/etc/crio/crio.conf`
CRI-O uses systemd as cgroup driver by default, but if this ever changes, we need to set `cgroup_manager = "systemd"` and `conmon_cgroup="system.slice"` as well.

```bash
which conmon
```

```/etc/crio/crio.conf
conmon = "/usr/bin/conmon"

registries = [
  "docker.io",
  "quay.io",
  "registry.fedoraproject.org",
 ]
```

After that we can start the systemd services

```bash
sudo systemctl daemon-reload
sudo systemctl enable crio --now
```

## Install Kubernetes binaries

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

## Setup hosts file

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

## Init the master using kubeadm

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

## Firewall on Master

Resources:

* <https://stackoverflow.com/questions/60708270/how-can-i-use-flannel-without-disabing-firewalld-kubernetes>
* <https://medium.com/platformer-blog/kubernetes-on-centos-7-with-firewalld-e7b53c1316af>
* <https://github.com/projectcalico/calico/issues/1214>

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

## Add Context to local kube config

Convert all base64 encoded certificates first

```bash
echo 'LS0xxxxxxx==' | base64 -d >> ca.crt
echo 'LS0xxxxxxx==' | base64 -d >> user.crt
echo 'LS0xxxxxxx==' | base64 -d >> user.key

kubectl config set-cluster cluster-cka-lab --embed-certs --server=https://65.21.60.29:6443 --certificate-authority=ca.crt
k config set-credentials admin-cka-lab --embed-certs --client-certificate=./user.crt --client-key=./user.key
k config set-context cka-lab --cluster cluster-cka-lab --user admin-cka-lab
```

## Join Kubelet as worker node

### Firewall on Worker Node

Resources:

* <https://stackoverflow.com/questions/60708270/how-can-i-use-flannel-without-disabing-firewalld-kubernetes>
* <https://medium.com/platformer-blog/kubernetes-on-centos-7-with-firewalld-e7b53c1316af>
* <https://github.com/projectcalico/calico/issues/1214>

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

### Create CA hashed token

```bash
openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | openssl dgst -sha256 -hex | sed 's/^.* //'
```

### Join using kubeadm

As we use CRI-O the Kubelet CGroup Driver must match the one CRI-O uses in `/etc/crio/crio.conf`
See also:

* <https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/kubelet-integration/#providing-instance-specific-configuration-details>
* <https://pkg.go.dev/k8s.io/kubernetes/pkg/kubelet/apis/config#KubeletConfiguration>

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

## Schedule pods on Control plane node

To remove `node-role.kubernetes.io/master` from all nodes:

```bash
kubectl taint nodes --all node-role.kubernetes.io/master-
```

## Install Calido CNI

### Get and adjust Config

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

### Verify node-to-node communication

```bash
kubectl create deployment shell-demo --image nginx
kubectl get pods --selector app=shell-demo
kubectl exec --stdin --tty shell-demo-xxxxxxx-xxxxx -- /bin/bash

apk add telnet
telnet OtherPodRunningNginx 443
```

## Maintenance

### Create Snapshots of etcd database

```bash
kubectl -n kube-system exec -it etcd-k8s-test-master -- sh -c "ETCDCTL_API=3 ETCDCTL_CACERT=/etc/kubernetes/pki/etcd/ca.crt ETCDCTL_CERT=/etc/kubernetes/pki/etcd/server.crt ETCDCTL_KEY=/etc/kubernetes/pki/etcd/server.key etcdctl help"
```

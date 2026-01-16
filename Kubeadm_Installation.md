## HA K8 Cluster
https://cri-o.io/

To set up a highly available Kubernetes cluster with two master nodes and three worker nodes without using a cloud load balancer, you can use a virtual machine to act as a load balancer for the API server. Here are the detailed steps for setting up such a cluster:

## Prerequisites
   3 master nodes
   3 worker nodes
   1 load balancer node
All nodes should be running a Linux distribution like Ubuntu

## Step 1: Prepare the Load Balancer Node
1. Install HAProxy:
```
sudo apt-get update
sudo apt-get install -y haproxy
```
2. Configure HAProxy: Edit the HAProxy configuration file (/etc/haproxy/haproxy.cfg):
```
sudo nano /etc/haproxy/haproxy.cfg
```
Add the following configuration:
```
frontend kubernetes-frontend
    bind *:6443
    option tcplog
    mode tcp
    default_backend kubernetes-backend

backend kubernetes-backend
    mode tcp
    balance roundrobin
    option tcp-check
    server master1 <MASTER1_IP>:6443 check
    server master2 <MASTER2_IP>:6443 check
```
3. Restart HAProxy:
```
sudo systemctl restart haproxy
```

## Step 2: Prepare All Nodes (Masters and Workers)
1. Install Docker, kubeadm, kubelet, and kubectl:
```
sudo apt-get update
sudo apt install docker.io -y
sudo chmod 666 /var/run/docker.sock
```
Define the Kubernetes version and used CRI-O stream
```
KUBERNETES_VERSION=v1.32
CRIO_VERSION=v1.32
```
Add the Kubernetes repository
```
cat <<EOF | tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/$KUBERNETES_VERSION/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/$KUBERNETES_VERSION/rpm/repodata/repomd.xml.key
EOF
```
Add the CRI-O repository
```
cat <<EOF | tee /etc/yum.repos.d/cri-o.repo
[cri-o]
name=CRI-O
baseurl=https://download.opensuse.org/repositories/isv:/cri-o:/stable:/$CRIO_VERSION/rpm/
enabled=1
gpgcheck=1
gpgkey=https://download.opensuse.org/repositories/isv:/cri-o:/stable:/$CRIO_VERSION/rpm/repodata/repomd.xml.key
EOF
```
Install package dependencies from the official repositories

```
dnf install -y container-selinux
```
Install the packages
```
dnf install -y cri-o kubelet kubeadm kubectl
```
Start CRI-O and enable
```
systemctl start crio.service
systemctl enable crio.service
```
Bootstrap a cluster
```
swapoff -a
modprobe br_netfilter
sysctl -w net.ipv4.ip_forward=1
```
```
vi /etc/fstab
```
'# /swap. image    none swap 0  0'

```
systemctl stop ufw
systemctl disable ufw
```
```
kubeadm init
```







Below is a production-grade, step-by-step guide to set up HAProxy + kubeadm HA Kubernetes cluster on Ubuntu 24.04.
These are exact commands you can copy-paste.

doc: https://cri-o.io/

## Prerequisites

3 master nodes
3 worker nodes
1 load balancer node
All nodes should be running a Linux distribution like Ubuntu

## Architecture (Recommended)

Node Type	Hostname	Example IP

HAProxy	haproxy	10.0.0.10

Master 1	master1	10.0.0.11

Master 2	master2	10.0.0.12

Master 3	master3	10.0.0.13

Worker 1	worker1	10.0.0.21

Worker 2	worker1	10.0.0.22

Worker 3	worker1	10.0.0.23

## HAProxy will load-balance port 6443 to all masters

## Install & Configure HAProxy (ONLY ON HAProxy NODE)
```
sudo apt install -y haproxy
```

## Edit config
```
sudo nano /etc/haproxy/haproxy.cfg
```

Replace EVERYTHING with 👇
```
global
    log /dev/log local0
    maxconn 2000
    daemon

defaults
    log global
    mode tcp
    timeout connect 10s
    timeout client  1m
    timeout server  1m

frontend kubernetes
    bind *:6443
    default_backend k8s-masters

backend k8s-masters
    balance roundrobin
    option tcp-check
    server master1 10.0.0.11:6443 check
    server master2 10.0.0.12:6443 check
    server master3 10.0.0.13:6443 check
```
```
sudo systemctl restart haproxy
sudo systemctl enable haproxy
```

✅ HAProxy is now ready

## Common setup (RUN ON ALL NODES)
## Install Docker
```
sudo apt-get update
sudo apt install docker.io -y
sudo chmod 666 /var/run/docker.sock
```

```
sudo apt update
sudo apt install -y apt-transport-https ca-certificates curl gpg
```
Bootstrap a cluster
```
swapoff -a
modprobe br_netfilter
sysctl -w net.ipv4.ip_forward=1
```
```
systemctl stop ufw
systemctl disable ufw
```

## Disable swap
```
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab
```
## Load kernel modules
```
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter
```
## Sysctl settings
```
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sudo sysctl --system
```
## Install containerd (ALL NODES)
```
sudo apt install -y containerd
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
```
## Enable systemd cgroup
```
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' \
/etc/containerd/config.toml
```
```
sudo systemctl restart containerd
sudo systemctl enable containerd
```
## Install kubeadm, kubelet, kubectl (ALL NODES)
```
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key |
sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] \
https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /" |
sudo tee /etc/apt/sources.list.d/kubernetes.list
```
```
sudo apt update
sudo apt install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

## Initialize Kubernetes (ONLY ON MASTER1)
```
sudo kubeadm init \
  --control-plane-endpoint "10.0.0.10:6443" \
  --upload-certs \
  --pod-network-cidr=10.244.0.0/16 \
  --kubernetes-version=v1.29.15
```

⚠️ SAVE the output (join commands are important)

## Configure kubectl ( MASTER1 )
```
mkdir -p $HOME/.kube
sudo cp /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

## Install CNI ( ONLY ON MASTER1 )
```
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
```

## Join other Masters ( MASTER2 & MASTER3 )

Run the control-plane join command you got earlier, example:
```
sudo kubeadm join 10.0.0.10:6443 \
 --token abcdef.0123456789abcdef \
 --discovery-token-ca-cert-hash sha256:XXXX \
 --control-plane \
 --certificate-key YYYY
```
## Join Worker Nodes
```
sudo kubeadm join 10.0.0.10:6443 \
 --token abcdef.0123456789abcdef \
 --discovery-token-ca-cert-hash sha256:XXXX
```
🔍 Verify Cluster
```
kubectl get nodes
kubectl get pods -A
```

Expected:

All masters → Ready

kube-system pods → Running

## Install Ingress-NGINX Controller: ( ONLY ON MASTER1 )


Clone the repository :
```
git clone https://github.com/nginx/kubernetes-ingress.git --branch v5.3.1
```
Change the active directory :
```
cd kubernetes-ingress
``
Create a namespace and a service account:
```
kubectl apply -f deployments/common/ns-and-sa.yaml
```
Create a cluster role and binding for the service account:
```
kubectl apply -f deployments/rbac/rbac.yaml
```
1.(Optional) Create a secret for the default NGINX server’s TLS certificate and key.

```
kubectl apply -f examples/shared-examples/default-server-secret/default-server-secret.yaml
```
2. Create a ConfigMap to customize your NGINX settings:
```
kubectl apply -f deployments/common/nginx-config.yaml
```
3. Create an IngressClass resource. NGINX Ingress Controller won’t start without an IngressClass resource.
```
kubectl apply -f deployments/common/ingress-class.yaml
```

Install CRDs after cloning the repo :
```
kubectl apply -f config/crd/bases/k8s.nginx.org_virtualservers.yaml
kubectl apply -f config/crd/bases/k8s.nginx.org_virtualserverroutes.yaml
kubectl apply -f config/crd/bases/k8s.nginx.org_transportservers.yaml
kubectl apply -f config/crd/bases/k8s.nginx.org_policies.yaml
kubectl apply -f config/crd/bases/k8s.nginx.org_globalconfigurations.yaml
```

Deploy NGINX Ingress Controller:

Using a Deployment:
```
kubectl apply -f deployments/deployment/nginx-ingress.yaml
```
Using a DaemonSet :
```
kubectl apply -f deployments/daemon-set/nginx-ingress.yaml
```
Using a StatefulSet :
```
kubectl apply -f deployments/stateful-set/nginx-ingress.yaml
```
```
kubectl get po -n nginx-ingress

kubectl get po -n nginx-ingress -o wide

kubectl edit deploy nginx-ingress -n nginx-ingress
```
go to ```"sepc:
            hostNetwork : true ``` (you have to add this above automountServiceAccountToken : true)

now it can use host network ip

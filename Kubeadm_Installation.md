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
apt-get update
apt-get install -y software-properties-common curl

curl -fsSL https://pkgs.k8s.io/core:/stable:/$KUBERNETES_VERSION/deb/Release.key |
    gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/$KUBERNETES_VERSION/deb/ /" |
    tee /etc/apt/sources.list.d/kubernetes.list

```
Add the CRI-O repository
```
curl -fsSL https://download.opensuse.org/repositories/isv:/cri-o:/stable:/$CRIO_VERSION/deb/Release.key |
    gpg --dearmor -o /etc/apt/keyrings/cri-o-apt-keyring.gpg

echo "deb [signed-by=/etc/apt/keyrings/cri-o-apt-keyring.gpg] https://download.opensuse.org/repositories/isv:/cri-o:/stable:/$CRIO_VERSION/deb/ /" |
    tee /etc/apt/sources.list.d/cri-o.list
```
Install the packages
```
apt-get update
apt-get install -y cri-o kubelet kubeadm kubectl
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
## Step 3: Initialize the First Master Node
1. Initialize the first master node:
```
sudo kubeadm init --control-plane-endpoint "LOAD_BALANCER_IP:6443" --upload-certs --pod-network-cidr=10.244.0.0/16
```
2. Set up kubeconfig for the first master node:
```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
3.Install Calico network plugin:
```
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
```

## Step 4: Join the Second & third Master Node
1. Get the join command and certificate key from the first master node:
```
kubeadm token create --print-join-command --certificate-key $(kubeadm init phase upload-certs --upload-certs | tail -1)
```
2. Run the join command on the second master node:
```
sudo kubeadm join LOAD_BALANCER_IP:6443 --token <token> --discovery-token-ca-cert-hash sha256:<hash> --control-plane --certificate-key <certificate-key>
```
3. Set up kubeconfig for the second master node:
```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
## Step 5: Join the Worker Nodes
1. Get the join command from the first master node:
```
kubeadm token create --print-join-command
```
2. Run the join command on each worker node:
```
sudo kubeadm join LOAD_BALANCER_IP:6443 --token <token> --discovery-token-ca-cert-hash sha256:<hash>
```
## Step 6: Verify the Cluster
Check the status of all nodes:
```
kubectl get nodes
```
3. Check the status of all pods:
```
kubectl get pods --all-namespaces
```

## Step 7: Install Ingress-NGINX Controller: (Initialize the First Master Node)

1. Clone the repository :
```
git clone https://github.com/nginx/kubernetes-ingress.git --branch v5.3.1
```
2. Change the active directory :
```
cd kubernetes-ingress
``
3. Create a namespace and a service account:
```
kubectl apply -f deployments/common/ns-and-sa.yaml
```
4. Create a cluster role and binding for the service account:
```
kubectl apply -f deployments/rbac/rbac.yaml
```
5.(Optional) Create a secret for the default NGINX server’s TLS certificate and key.

```
kubectl apply -f examples/shared-examples/default-server-secret/default-server-secret.yaml
```
6. Create a ConfigMap to customize your NGINX settings:
```
kubectl apply -f deployments/common/nginx-config.yaml
```
7. Create an IngressClass resource. NGINX Ingress Controller won’t start without an IngressClass resource.
```
kubectl apply -f deployments/common/ingress-class.yaml
```

8. Install CRDs after cloning the repo :
```
kubectl apply -f config/crd/bases/k8s.nginx.org_virtualservers.yaml
kubectl apply -f config/crd/bases/k8s.nginx.org_virtualserverroutes.yaml
kubectl apply -f config/crd/bases/k8s.nginx.org_transportservers.yaml
kubectl apply -f config/crd/bases/k8s.nginx.org_policies.yaml
kubectl apply -f config/crd/bases/k8s.nginx.org_globalconfigurations.yaml
```

9. Deploy NGINX Ingress Controller:

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
```
```
kubectl get po -n nginx-ingress -o wide
```
```
kubectl edit deploy nginx-ingress -n nginx-ingress
```
go to "sepc:
        hostNetwork : true  (you have to add this above automountServiceAccountToken : true)

now it can use host network ip

https://docs.nginx.com/nginx-ingress-controller/install/manifests/



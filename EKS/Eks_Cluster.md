## create an Amazon EKS cluster from an Ubuntu machine (24.04)

1) Launch new Ubuntu VM using AWS Ec2 ( t2.micro )	  
2) Connect to machine and install kubectl using below commands  

## 🔹 Prerequisites

AWS account
IAM user with AdministratorAccess (or EKS + EC2 + VPC permissions)
Ubuntu machine (local / EC2)
Internet access


## Step 1: Install AWS CLI (v2)
```
sudo apt update
sudo apt install unzip curl -y

curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o awscliv2.zip
unzip awscliv2.zip
sudo ./aws/install
```
Verify:
```
aws --version
```
## Step 2: Configure AWS CLI
```
aws configure
```
Enter:

AWS Access Key ID

AWS Secret Access Key

Region (example: ap-south-1)

Output format: json

## Step 3: Install kubectl
```
curl -LO "https://dl.k8s.io/release/$(curl -Ls https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/
```
## Verify:
```
kubectl version --client
```
## Step 4: Install eksctl
```
curl --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz
sudo mv eksctl /usr/local/bin/
eksctl version
```
## Step 5: Create EKS Cluster (Simple Way)

## ✅ Create cluster with managed node group
```
eksctl create cluster \
--name my-eks-cluster \
--region ap-south-1 \
--nodegroup-name my-nodes \
--node-type t3.medium \
--nodes 2 \
--nodes-min 1 \
--nodes-max 3 \
--managed
```
## 🔹 Step 6: Verify Cluster
```
kubectl get nodes
kubectl get pods -A
```
## 🔹 Step 7: Update kubeconfig (if needed)
```
aws eks --region ap-south-1 update-kubeconfig --name my-eks-cluster
```
## 🔹 Step 8: Delete Cluster (Important to avoid billing)
```
eksctl delete cluster --name my-eks-cluster --region ap-south-1
```

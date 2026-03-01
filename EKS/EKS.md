--------------------------------------------------------------------------------------------------------------------------------------------------------------------------
										                            E-Commerce App Deployment
											                                  by
										                                 Velocity9919
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------
## Step 1
Use the same Security Group to all the instances that are going to be launched

## 1.1. Launch 1 VM (Ubuntu, 24.04, t2.large, 20 GB, Name: Jenkins Server)

Open below ports for the Security Group attached to the above VM
_____________________________________________________________________________________________________________________
Port 			| Protocol 	| Service 				| Purpose
_____________________________________________________________________________________________________________________
22 			| TCP 	| SSH 				| Secure shell access to servers for remote login, command execution 

25 			| TCP 	| SMTP 				| Used for sending email between servers (outbound email)

80 			| TCP 	| HTTP 				| Handles unencrypted web traffic; used for serving websites over the internet.

443 		| TCP 	| HTTPS 			| Secure version of HTTP; encrypts communication between browser and server using TLS/SSL.

465 		| TCP 	| SMTPS 			| Secure SMTP; used for sending emails securely with SSL 

6443 		| TCP 	| Kubernetes API server 	| Used for communication with the Kubernetes control plane (kubectl, kubelets, etc.).

3000–10000 	| TCP 	| Service Ports 			| Used for custom application services (e.g., Node.js apps often use 3000)

30000–32767 | TCP 	| K8S NodePort Services | Kubernetes allocates this range for exposing services via NodePort (to access a service outside the cluster).


## 1.2. Launch 2 VMs (Ubuntu, 24.04, t2.medium, 20 GB, One VM Name: Nexus Server, Another VM Name: SonarQube Server)

Attach the same security group which is used for Jenkins Server

## 1.3. Creation of EKS Cluster

## 1.3.1. Creation of IAM user (To create EKS Cluster, its not recommended to create using Root Account)

IAM ----> Users ----> Create User ----> 
Name: eks-kastro-user, 
'Check' Provide access to console, 
'Check' I want to create IAM User, 
Custom Password ----> Next 
(dont attach any policies) ----> Next ----> Create user

## 1.3.2. Attach policies to the user 

Open the user created ----> 'Permissions' tab ----> In 'Add permissions' dropdown select 'Add permissions' ----> 'Check' Attach policies directly 
----> AmazonEC2FullAccess, AmazonEKS_CNI_Policy, AmazonEKSClusterPolicy, AmazonEKSWorkerNodePolicy, AWSCloudFormationFullAccess, IAMFullAccess 
----> Next ----> Add permissions

Attach the below inline policy also for the same user

Open the user created ----> 'Permissions' tab 
----> In 'Add permissions' dropdown select 'Create Inline Policy' 
----> Json ----> Remove any existing policy. 

Paste the below policy
```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "VisualEditor0",
            "Effect": "Allow",
            "Action": "eks:*",
            "Resource": "*"
        }
    ]
}
```
----> Next ----> Policy Name: eks-inline-policy ----> Create Policy

A total of 7 policies we have attached to the IAM User created in 1.3.1.

## 1.3.3. Create Access Keys

Open the user created ----> Security Credentials tab ----> Create access key ----> Use case: CLI ----> Next ----> Create Keys ----> Download the csv file

Now we have created the IAM User with appropriate permissions to create the EKS Cluster

AKIA6G75DMYKLKK
YNvoMvkeA7uIiNmuw6C5YyLREpi2b1DlAo

## 1.4. Creation of EKS Cluster 

Connect to the 'Jenkins Server' VM
```
sudo apt update
```
## 1.4.1. Install AWS CLI (to interact with AWS Account)
```
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
sudo apt install unzip
unzip awscliv2.zip
sudo ./aws/install
aws configure
```
## Configure aws
```
aws configure 
```
----> Paste the Access Key 
----> Paste the Secret Access Key 
----> Region: ap-south-1 
----> Output: click enter (no need to give anything)

## 1.4.2. Install KubeCTL (to interact with K8S)

```
curl -o kubectl https://amazon-eks.s3.us-west-2.amazonaws.com/1.19.6/2021-01-05/bin/linux/amd64/kubectl
chmod +x ./kubectl
sudo mv ./kubectl /usr/local/bin
kubectl version --short --client
```
## 1.4.3. Install EKS CTL (used to create EKS Cluster) 
```
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin
eksctl version
```
## 1.4.4. Create EKS Cluster

Execute the below commands as separate set

## (a) Create EKS Cluster (WITHOUT nodegroup)
```
eksctl create cluster \
  --name my-eks-cluster \
  --region ap-south-1 \
  --zones ap-south-1a,ap-south-1b \
  --version 1.30 \
  --without-nodegroup
```

It will take 5-10 minutes to create the cluster
Goto EKS Console and verify the cluster.

## (b) Associate IAM OIDC Provider (MANDATORY)
```
eksctl utils associate-iam-oidc-provider \
  --region ap-south-1 \
  --cluster my-eks-cluster \
  --approve
```

The above command is crucial when setting up an EKS cluster because it enables IAM roles for service accounts (IRSA)
Amazon EKS uses OpenID Connect (OIDC) to authenticate Kubernetes service accounts with IAM roles.
Associating the IAM OIDC provider allows Kubernetes workloads (Pods) running in the cluster to assume IAM roles securely.
Without this, Pods in EKS clusters would require node-level IAM roles, which grant permissions to all Pods on a node.
Without it, these services will not be able to access AWS resources securely.

## (c) Create Managed Node Group
Before executing the below command, in the 'ssh-public-key' keep the  '<PEM FILE NAME>' (dont give .pem. Just give the pem file name) which was used to create Jenkins Server
```
eksctl create nodegroup \
  --cluster my-eks-cluster \
  --region ap-south-1 \
  --name node2 \
  --node-type t3.small \
  --nodes 3 \
  --nodes-min 2 \
  --nodes-max 4 \
  --node-volume-size 20 \
  --ssh-access \
  --ssh-public-key Jenkins_key \
  --managed \
  --asg-access \
  --external-dns-access \
  --full-ecr-access \
  --appmesh-access \
  --alb-ingress-access
```

It will take 5-10 minutes 

## (d) For internal communication b/w control plane and worker nodes, we need to open one traffic in the security group of EKS Cluster

Open the cluster created 
----> 'Networking' tab 
----> Under 'Additional Security Groups' you will see a link, click on that link 
----> 'Inbound rules' tab 
----> Edit inbound rules 
----> Add rule 
----> All traffic, Anywhere IPv4 
----> Save rules

                                                ---------------------------------------
                                                                  Deploying to EKS Cluster
                                               ---------------------------------------

## 5 Create Service Account, Role & Assign that role, And create a secret for Service Account and generate a Token

Here we will create all the resources inside a namespace called 'webapps' which is a recommended approach in the realtime scenario

## (a) Creating Service Account 
```
vi svc.yml
```
```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: jenkins
  namespace: webapps
```
Lets create a namespace ---> ```kubectl create namespace webapps``` ---> ```kubectl apply -f svc.yml```

## (b) Create Role
```
vi role.yml
```
```
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: app-role
  namespace: webapps
rules:
  - apiGroups:
        - ""
        - apps
        - autoscaling
        - batch
        - extensions
        - policy
        - rbac.authorization.k8s.io
    resources:
      - pods
      - componentstatuses
      - configmaps
      - daemonsets
      - deployments
      - events
      - endpoints
      - horizontalpodautoscalers
      - ingress
      - jobs
      - limitranges
      - namespaces
      - nodes
      - pods
      - persistentvolumes
      - persistentvolumeclaims
      - resourcequotas
      - replicasets
      - replicationcontrollers
      - serviceaccounts
      - services
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
```
```
kubectl apply -f role.yml
```

## (c) Bind the role to service account
```
vi bind.yml
```
```
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: app-rolebinding
  namespace: webapps 
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: app-role 
subjects:
- namespace: webapps 
  kind: ServiceAccount
  name: jenkins 
```
```
kubectl apply -f bind.yml
```
## (d) In order to use the service account for deployment, we need to create token for the service account which will be used as authentication
```
vi secret.yml
```
--- Paste the below content; (reference URL for secret: https://kubernetes.io/docs/reference/access-authn-authz/service-accounts-admin/#:~:text=To%20create%20a%20non%2Dexpiring,with%20that%20generated%20token%20data.)
```
apiVersion: v1
kind: Secret
type: kubernetes.io/service-account-token
metadata:
  name: mysecretname
  annotations:
    kubernetes.io/service-account.name: jenkins #Here my service account name is 'jenkins' based on the above created yml files
```
```
kubectl apply -f secret.yml -n webapps
```
--- Now the secret will get created inside the namespace 
--- To get the token 
--- kubectl describe secret (kubectl describe secret mysecretname -n webapps) 
--- You can see the token. 
--- Copy and paste in notepad

## We will configure this in Jenkins in sometime

eyJhbGciOiJSUzI1NiIsImtpZCI6IkNxMkxsR3VNTVlsSGhNRHd2T1F1X1N1eUxOVnJfODdmZHVHZ0E2WHRmdjAifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJ3ZWJhcHBzIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZWNyZXQubmFtZSI6Im15c2VjcmV0bmFtZSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50Lm5hbWUiOiJqZW5raW5zIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQudWlkIjoiYjAxNmFmZjctOGE1OC00ZDEyLTlhYjEtY2EwYjcxMzQzYzc1Iiwic3ViIjoic3lzdGVtOnNlcnZpY2VhY2NvdW50OndlYmFwcHM6amVua2lucyJ9.worPaZdFJfeswDgIghIngEjeiZ2x2V9yFQc9qLG3yyJRh5ZOnt3WiHeQkP_wfbLd5vt5r-WIsEA3BjFDI8KMaxgOa9lmrGa9jYQjaEyCw-oErDUz1Xh87Iby_OG1bkmij-sTU4GKXLyoavw5dEr5FLY0iAv2aoAzaUchb-MVoTmxWrTQa3RcVGeraOWIWkv-jP1bq-4eFGAeBm_1HG5KakWJ5Y7gnpNGn6RWAowh_9p0R-VMsirmSmeZ19_-FAdBg0zgjaHtcm317P5wUVMNNT3zxoWL9NN7_VAPFAIgW2SLKID45ww6bfEWiaFNaZETizKfZjhRW5rhS8sIevJe9g

## Lets create CD pipeline
Dashboard --- Manage Jenkins --- Security --- Credentials --- Global credentials (unrestricted) 
--- Kind: secret text 
--- Scope: global 
--- Secret: Paste the token generated above 
--- ID: k8-token, Decription: k8-token 
--- Create

## Pipeline syntax 
--- Sample step: withKubeCredentials: Configure Kubernetes CLI (kubectl) 
--- Credentials: k8-token 
--- Kubernetes Server Endpoint: To get this endpoint, go to EKS Cluster 
--- Overview tab --- Details --- Copy the API server endpoint and paste it 
--- Cluster name: <GiveTheNameAsInEKSConsole>, Context name: <leaveBlank>, Namespace: <NameSpaceName i.e webapps>, 
--- Generate pipeline script 
--- Copy and paste in the 'Deploy To Kubernetes' and 'verify Deployment' stages in the below pipeline script

                                            ___________________________________________________
                                                Ecommerce App - Successful (with K8S Stage)
                                            ___________________________________________________

```
pipeline {
    agent any

    tools {
        jdk 'jdk17'
        maven 'maven'
    }

    environment {
        SCANNER_HOME = tool 'sonar-scanner'
        DOCKER_IMAGE = "kastrov/ecommerce:latest"
    }

    stages {
        stage('Git Checkout') {
            steps {
                git 'https://github.com/KastroVKiran/Ecommerce-App-Kastro.git'
            }
        }

        stage('Maven Compile') {
            steps {
                sh "mvn compile"
            }
        }

        stage('Maven Test') {
            steps {
                sh "mvn test -DskipTests=true"
            }
        }

        stage('File System Scan') {
            steps {
                sh "trivy fs --format table -o trivy-fs-report.html ."
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar') {
                    sh '''
                    $SCANNER_HOME/bin/sonar-scanner \
                    -Dsonar.projectName=ECommerce \
                    -Dsonar.projectKey=ECommerce \
                    -Dsonar.java.binaries=target/classes
                    '''
                }
            }
        }

        stage('Maven Build') {
            steps {
                sh "mvn package -DskipTests=true"
            }
        }

        stage('Publish to Nexus') {
            steps {
                withMaven(globalMavenSettingsConfig: 'maven-setting', jdk: 'jdk17', maven: 'maven') {
                    sh "mvn deploy -DskipTests=true"
                }
            }
        }

        stage('Docker Build & Tag') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker-cred') {
                        sh "docker build -t ${DOCKER_IMAGE} ."
                    }
                }
            }
        }

        stage('Docker Image Scan') {
            steps {
                sh "trivy image --format table -o trivy-image-report.html ${DOCKER_IMAGE}"
                archiveArtifacts artifacts: 'trivy-image-report.html', fingerprint: true
            }
        }

        stage('Push Docker Image') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker-cred') {
                        sh "docker push ${DOCKER_IMAGE}"
                    }
                }
            }
        }

        stage('Deploy to Container') {
            steps {
                script {
                    sh "docker stop ecommerce-container || true && docker rm ecommerce-container || true"
                    sh "docker run -d --name ecommerce-container -p 8083:8080 ${DOCKER_IMAGE}"
                }
            }
        }

        // ✅ These two stages were outside of "stages" block before. Moved inside.
        stage('Deploy To Kubernetes') {
            steps {
                withKubeConfig(caCertificate: '', clusterName: 'kastro-eks', contextName: '', credentialsId: 'k8-token', namespace: 'webapps', restrictKubeConfigAccess: false, serverUrl: 'https://A3A38B4561B8159534054B870C447E05.yl4.us-east-1.eks.amazonaws.com') {
                    sh "kubectl apply -f deployment-service.yaml -n webapps"
                }
            }
        }

        stage('Verify the Deployment') {
            steps {
                withKubeConfig(caCertificate: '', clusterName: 'kastro-eks', contextName: '', credentialsId: 'k8-token', namespace: 'webapps', restrictKubeConfigAccess: false, serverUrl: 'https://A3A38B4561B8159534054B870C447E05.yl4.us-east-1.eks.amazonaws.com') {
                    sh "kubectl get pods -n webapps"
                    sh "kubectl get svc -n webapps"
                }
            }
        }
    } // ✅ Correct closing of "stages" block

    post {
        always {
            script {
                def jobName = env.JOB_NAME
                def buildNumber = env.BUILD_NUMBER
                def pipelineStatus = currentBuild.result ?: 'SUCCESS'
                def bannerColor = pipelineStatus.toUpperCase() == 'SUCCESS' ? 'green' : 'red'

                def body = """
                <html>
                <body>
                <div style="border: 4px solid ${bannerColor}; padding: 10px;">
                <h2>${jobName} - Build ${buildNumber}</h2>
                <div style="background-color: ${bannerColor}; padding: 10px;">
                <h3 style="color: white;">Pipeline Status: ${pipelineStatus.toUpperCase()}</h3>
                </div>
                <p>Check the <a href="${BUILD_URL}">console output</a>.</p>
                </div>
                </body>
                </html>
                """

                emailext (
                    subject: "${jobName} - Build ${buildNumber} - ${pipelineStatus.toUpperCase()}",
                    body: body,
                    to: 'kastrokiran@gmail.com',
                    from: 'jenkins@example.com',
                    replyTo: 'jenkins@example.com',
                    mimeType: 'text/html',
                    attachmentsPattern: 'trivy-image-report.html'
                )
            }
        }
    }
}
```

To access the App, we need to get the Load Balancer URL
```
kuetl get all -n webapps
```




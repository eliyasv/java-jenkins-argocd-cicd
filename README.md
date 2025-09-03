# 🚀 CI/CD Setup with Jenkins, SonarQube, Docker, Kubernetes, and Argo CD  

This guide walks through setting up a **CI/CD pipeline** on an AWS EC2 instance with **Jenkins**, **SonarQube**, **Docker**, and deploying to **Kubernetes** with **Argo CD**.  

---

## 📌 Prerequisites  

- AWS account with permissions to create EC2 and Security Groups  
- Ubuntu EC2 instance (`t2.medium` or higher recommended)  
- Security Group inbound rules:  
  - `22` → SSH  
  - `80` → HTTP  
  - `8080` → Jenkins  
  - `9000` → SonarQube  

---

## ⚙️ Step 1: Provision an EC2 Instance  

Launch an EC2 instance (Ubuntu) and allow required inbound traffic.  
> ⚠️ Do **not** allow all traffic (`0.0.0.0/0`) unless testing.  

---

## ⚙️ Step 2: Install Jenkins  

### Install Java  
```bash
sudo apt update
sudo apt install openjdk-17-jre -y

```

### Install Jenkins

```bash

curl -fsSL https://pkg.jenkins.io/debian/jenkins.io-2023.key | sudo tee \
  /usr/share/keyrings/jenkins-keyring.asc > /dev/null

echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] \
  https://pkg.jenkins.io/debian binary/ | sudo tee \
  /etc/apt/sources.list.d/jenkins.list > /dev/null

sudo apt-get update
sudo apt-get install jenkins  

```

### Access jenkins at 👉 http://<public ip address of ec2>:8080 and follow on screen info to unlock and login

## ⚙️ Step 3: Install Required Plugins

Navigate to Manage Jenkins → Plugins → Available Plugins

install:
* Docker Pipeline
* Sonarqube Scanner

🔄 After installing each plugin, restart Jenkins by visiting:
👉 http://<EC2-Public-IP>:8080/restart


## ⚙️ Step 4: Configure Docker

```bash
sudo apt update
sudo apt install docker.io

# grant jenkins user and (if needed) ubuntu user permission to docker deamon

sudo su - 
usermod -aG docker jenkins
usermod -aG docker ubuntu
systemctl restart docker

```

## ⚙️ Step 5: Install & Configure SonarQube

```bash

adduser sonarqube

#switch user by sudo su - sonarqube

wget https://binaries.sonarsource.com/Distribution/sonarqube/sonarqube-10.4.1.88267.zip
unzip *
chown -R sonarqube:sonarqube home/opt/sonarqube-10.4.1.88267
chmod -R 775 home/opt/sonarqube-10.4.1.88267
cd /opt/sonarqube-10.4.1.88267/bin/linux-x86-64
./sonar.sh start

```
### Access Sonarqube at 👉 http://<public ip address of ec2>:9000

Generate a token:
Account → Administrator → Security → Generate Token


## ⚙️ Step 6: Add Credentials in Jenkins

Inside Jenkins → Manage Jenkins → Credentials → Global → Add Credentials:

add the creds for 
* SonarQube token
* Docker Hub credentials
* GitHub credentials


## ⚙️ Step 7: Jenkins Pipeline

### Refer the JenkinsFile for stages and details

Jenkinsfile includes stages for:

* Checkout → Get source code
* Build → Compile and package app
* SonarQube Scan → Static Code Analysis
* Docker Image Build & Push → Publish Docker image to registry
* Update Deployment → Update K8s manifest in GitHub (for Argo CD to deploy)

## ⚙️ Step 8: Setup Kubernetes & Argo CD

### Start Minikube (local test)

```bash

minikube start

```
### Install argo cd in the cluster and configure the service type as nodeport

```bash

kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

```

### Change Argo CD Service to NodePort

```bash

kubectl get svc # will show all services (we are looking for argocd-server)
kubectl edit svc argocd-server # edit the type to NodePort instead of ClusterIp
```
### Access Argo CD

```bash

minikube service argocd-server # this will provide us with url for accesing the argocd server

```

### The password for argocd can be obtained by 

```bash

kubectl get secret # a list will show , we need the one argocd-cluster 
kubectl edit secret argocd-cluster 
# the secret is base 64 encripted decript by
echo "<encripted password>" | base64 -d

```
### Then login and confiqure the server
Login at:
👉 http://<NodePort-URL>

* Create new application
configure with the repo url and path to manifest file to track 

Application Name: app
Project Name: default
Sync Policy  - Automatic

* Source
Repository URL: GitHub repo (e.g., https://github.com/user/repo)
Revision: main
Path: Path to manifests (e.g., spring-boot-app-manifests/)

* Destination
Cluster URL: https://kubernetes.default.svc (default K8s cluster)
Namespace: default (or the namespace where you want to deploy)

### Then the argo cd will automaticaly sync the manifest files each time a change or drift is detected



#### reference https://github.com/iam-veeramalla/Jenkins-Zero-To-Hero/tree/main/java-maven-sonar-argocd-helm-k8s/spring-boot-app

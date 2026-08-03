# 🚀 Frontend CI/CD Deployment on Amazon EKS using Jenkins, Amazon ECR & AWS Load Balancer Controller

---

# 📖 Project Overview

This project demonstrates how to automate the deployment of a frontend application to an **Amazon Elastic Kubernetes Service (EKS)** cluster using **Jenkins** as the CI/CD server.

Instead of pushing Docker images to Docker Hub, the application image is stored securely in **Amazon Elastic Container Registry (ECR)**. External traffic is managed through **AWS Load Balancer Controller**, which automatically provisions an **Application Load Balancer (ALB)** based on Kubernetes Ingress resources.

---

# 🏗️ Architecture

```text
                     Developer
                         │
                  Git Push (GitHub)
                         │
                         ▼
                 GitHub Webhook Trigger
                         │
                         ▼
              Jenkins Server (Amazon EC2)
                         │
      ┌──────────────────┼─────────────────────┐
      │                  │                     │
Checkout Code     Build Frontend        Build Docker Image
      │                  │                     │
      └──────────────────┼─────────────────────┘
                         │
              Authenticate with Amazon ECR
                         │
                         ▼
               Push Docker Image to ECR
                         │
                         ▼
                kubectl apply -f manifests
                         │
                         ▼
                 Amazon EKS Kubernetes
                         │
      ┌──────────────────┼─────────────────┐
      │                  │                 │
 Deployment        ClusterIP Service    Ingress
                         │
                         ▼
      AWS Load Balancer Controller (Helm)
                         │
                         ▼
          Application Load Balancer (ALB)
                         │
                         ▼
                    End Users
```

---

# 📋 Table of Contents

* Project Overview
* Architecture
* Technologies Used
* Prerequisites
* Phase 1 – AWS Infrastructure
* Phase 2 – Jenkins Server Setup
* Phase 3 – Amazon EKS Cluster
* Phase 4 – Amazon ECR
* Phase 5 – Install Helm
* Phase 6 – Install AWS Load Balancer Controller
* Phase 7 – Jenkins Configuration
* Phase 8 – Kubernetes Manifests
* Phase 9 – CI/CD Pipeline
* Phase 10 – Deployment Verification
* Troubleshooting
* Cleanup
* Conclusion

---

# 🛠 Technologies Used

| Technology                   | Purpose                             |
| ---------------------------- | ----------------------------------- |
| GitHub                       | Source Code Management              |
| Jenkins                      | Continuous Integration & Deployment |
| Docker                       | Containerization                    |
| Amazon EC2                   | Jenkins Server                      |
| Amazon ECR                   | Docker Image Registry               |
| Amazon EKS                   | Kubernetes Cluster                  |
| kubectl                      | Kubernetes CLI                      |
| eksctl                       | EKS Cluster Creation                |
| Helm                         | Kubernetes Package Manager          |
| AWS Load Balancer Controller | Automatic ALB Provisioning          |
| Node.js & npm                | Frontend Build                      |

---

# 📌 Prerequisites

Before starting, ensure you have:

* AWS Account
* GitHub Account
* IAM User with required permissions
* Ubuntu EC2 Instance
* Frontend Project
* Docker Installed
* Basic Kubernetes Knowledge

---

# 🚀 Phase 1 – AWS Infrastructure Setup

## Step 1 – Create IAM User

Create an IAM user with programmatic access.

Required permissions:

* AmazonEKSClusterPolicy
* IAMFullAccess
* AmazonEC2ContainerRegistryFullAccess
* AmazoneEKSWorkerNodePolicy

---

## Step 2 – Launch EC2 Instance

Recommended Configuration

| Property       | Value             |
| -------------- | ----------------- |
| AMI            | Ubuntu 22.04      |
| Instance Type  | t3.medium         |
| Storage        | 30 GB             |
| Security Group | 22, 80, 443, 8080 |

Connect

```bash
ssh -i key.pem ubuntu@PUBLIC-IP
```

---

# ⚙️ Phase 2 – Jenkins Server Setup

## Update System

```bash
sudo apt update
sudo apt upgrade -y
```

---

## Install Java

```bash
sudo apt install openjdk-17-jdk -y
```

Verify

```bash
java -version
```

---

## Install Jenkins

```bash
 sudo wget -O /etc/apt/keyrings/jenkins-keyring.asc   https://pkg.jenkins.io/debian-stable/jenkins.io-2026.key
 echo "deb [signed-by=/etc/apt/keyrings/jenkins-keyring.asc]"   https://pkg.jenkins.io/debian-stable binary/ | sudo tee   /etc/apt/sources.list.d/jenkins.list > /dev/null
 sudo apt update
 sudo apt install jenkins -y
```

Enable

```bash
sudo systemctl enable jenkins
sudo systemctl start jenkins
```

Check

```bash
sudo systemctl status jenkins
```

---

## Install Docker

```bash
sudo apt install docker.io -y
```

Enable

```bash
sudo systemctl enable docker
sudo systemctl start docker
```

---

## Allow Jenkins to Use Docker

```bash
sudo usermod -aG docker jenkins
sudo systemctl restart jenkins
```

---

## Install Node.js

```bash
sudo apt install nodejs npm
```

Verify

```bash
node -v
npm -v
```

---

## Install AWS CLI

Verify

```bash
aws --version
```

---

## Configure AWS CLI

```bash
aws configure
```

Enter:

* Access Key
* Secret Key
* Region
* Output Format

---

## Install kubectl

Verify

```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
kubectl version --client
```

---

## Install eksctl

Verify

```bash
eksctl version
```

---

# ☸️ Phase 3 – Create Amazon EKS Cluster

Create Cluster

```bash
eksctl create cluster \
--name demo-cluster \
--region ap-south-1 \
--nodegroup-name workers \
--node-type t3.medium \
--nodes 2
```

Verify

```bash
kubectl get nodes
```

Expected

```text
NAME              STATUS
ip-xxx            Ready
ip-yyy            Ready
```

Update kubeconfig

```bash
aws eks create-access-entry \
  --cluster-name cluster-name \
  --principal-arn arn:aws:iam::<account_id>:role/EC2IamRole \
  --type STANDARD \
  --region ap-south-1
```
---
```bash
aws eks associate-access-policy \
  --cluster-name cluster-name \
  --principal-arn arn:aws:iam::<accou_id>:role/EC2IamRole \
  --policy-arn arn:aws:eks::aws:cluster-access-policy/AmazonEKSClusterAdminPolicy \
  --access-scope type=cluster \
  --region ap-south-1
```
---

# 📦 Phase 4 – Amazon ECR

Create Repository

Example

```text
frontend-app
```

Login

```bash
aws ecr get-login-password \
| docker login \
--username AWS \
--password-stdin ACCOUNT_ID.dkr.ecr.ap-south-1.amazonaws.com
```

Build Image

```bash
docker build -t frontend .
```

Tag Image

```bash
docker tag frontend:latest ACCOUNT_ID.dkr.ecr.ap-south-1.amazonaws.com/frontend:latest
```

Push Image

```bash
docker push ACCOUNT_ID.dkr.ecr.ap-south-1.amazonaws.com/frontend:latest
```

---

# ⛵ Phase 5 – Install Helm

Verify

```bash
helm version
```

Add Repository

```bash
helm repo add eks https://aws.github.io/eks-charts
helm repo update
```

---

# 🌐 Phase 6 – Install AWS Load Balancer Controller

Associate OIDC Provider

```bash
eksctl utils associate-iam-oidc-provider \
--cluster frontend-cluster \
--approve
```

Download IAM Policy

```bash
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/main/docs/install/iam_policy.json
```

Create IAM Policy

```bash
aws iam create-policy \
--policy-name AWSLoadBalancerControllerIAMPolicy \
--policy-document file://iam_policy.json
```

Create IAM Service Account

```bash
eksctl create iamserviceaccount
```

Install Controller

```bash
helm install aws-load-balancer-controller \
eks/aws-load-balancer-controller
```

Verify

```bash
kubectl get deployment -n kube-system
```

Expected

```text
aws-load-balancer-controller
```

---

# ⚙️ Phase 7 – Jenkins Configuration

Install Plugins

* Git
* Pipeline
* Docker Pipeline
* Credentials
* NodeJS
* Kubernetes CLI

Configure Credentials

* GitHub Token

Create Pipeline Job

Pipeline Source

```
GitHub Repository
```

Jenkinsfile Path

```
Jenkinsfile
```

Enable

```
GitHub Webhook Trigger
```

---

# 📂 Phase 8 – Kubernetes Manifests

Repository Structure

```text
frontend-project/

├── src/
├── public/
├── Dockerfile
├── Jenkinsfile
├── package.json
├── deployment.yaml
├── service.yaml
├── ingress.yaml
└── README.md
```

Files

* deployment.yaml
* service.yaml
* ingress.yaml

---

# 🔄 Phase 9 – CI/CD Pipeline Workflow

Pipeline Stages

```text
Checkout Source Code

↓

Login to Amazon ECR

↓

Build Docker Image

↓

Tag Docker Image

↓

Push Docker Image

↓

Configure kubectl

↓

Deploy to EKS

↓

Verify Rollout

↓

Verify Deployment

↓

Ingress Created

↓

AWS Load Balancer Controller

↓

Application Load Balancer

↓

Frontend Live
```

---

# ✅ Phase 10 – Verification

Pods

```bash
kubectl get pods
```

Deployments

```bash
kubectl get deployment
```

Services

```bash
kubectl get svc
```

Ingress

```bash
kubectl get ingress
```

Events

```bash
kubectl get events
```

Logs

```bash
kubectl logs <pod-name>
```

---

# ❗ Troubleshooting

### Docker Permission Denied

```bash
sudo usermod -aG docker jenkins
sudo systemctl restart jenkins
```

---

### kubectl Cannot Connect

```bash
aws eks update-kubeconfig \
--region ap-south-1 \
--name frontend-cluster
```

---

### ImagePullBackOff

Check:

* Image URI
* Image Tag
* ECR Authentication
* IAM Permissions

---

### Ingress Address Pending

Check:

```bash
kubectl get pods -n kube-system
```

Verify:

* AWS Load Balancer Controller
* IAM Role
* OIDC Provider
* Subnet Tags

---

# 🎯 Learning Outcomes

After completing this project, we will understand:

* Jenkins Pipeline
* Docker Image Build Process
* Amazon ECR Authentication
* Kubernetes Deployment
* Amazon EKS Administration
* Helm Package Management
* AWS Load Balancer Controller
* Kubernetes Ingress
* Continuous Integration
* Continuous Deployment

---

# 🎉 Conclusion

This project demonstrates a production-oriented CI/CD workflow for deploying a frontend application on Amazon EKS using Jenkins. Docker images are securely stored in Amazon ECR, while application traffic is managed through an Application Load Balancer automatically created by the AWS Load Balancer Controller installed with Helm. This architecture provides scalability, automation, and aligns with common enterprise DevOps practices.

---

**Author:** *Nilesh Dubey*
**Project:** Frontend CI/CD on Amazon EKS using Jenkins, Amazon ECR & AWS Load Balancer Controller

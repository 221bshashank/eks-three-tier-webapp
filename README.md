# AWS EKS Core Deployment: 3-Tier Web Application Architecture

## Overview

This repository contains the configuration, manifests, and source artifacts used to deploy a highly-available Three-Tier Web Application (ReactJS Frontend, NodeJS Backend, and MongoDB Database) onto **Amazon EKS (Elastic Kubernetes Service)**.

This guide outlines native cluster resource orchestration, automation scripts via `eksctl`, OpenID Connect (OIDC) identity binding, and Application Load Balancer (ALB) ingress controllers optimized for the restrictive constraints of the **KodeKloud AWS Sandbox** network infrastructure.

> 📋 **Configuration Note:** Throughout this guide, you will see placeholders in angle brackets (e.g., `<211125536240>`, `<three-tier-cluster>`, `<shashank.devopsk8s.com>`). Before executing any command, ensure you replace these placeholders with your specific AWS Account ID, Region, Cluster Name, and Domain settings.

---

# 📂 Repository File Structure

```text
.
├── Application-Code
│   ├── backend
│   │   ├── db.js
│   │   ├── Dockerfile
│   │   ├── index.js
│   │   ├── models
│   │   │   └── task.js
│   │   ├── package.json
│   │   ├── package-lock.json
│   │   ├── routes
│   │   │   └── tasks.js
│   │   └── trust-policy.json
│   └── frontend
│       ├── Dockerfile
│       ├── package.json
│       ├── package-lock.json
│       ├── public
│       │   ├── index.html
│       │   └── ...
│       └── src
│           ├── App.js
│           ├── services
│           │   └── taskServices.js
│           └── Tasks.js
├── Kubernetes-Manifests-file
│   ├── Backend
│   │   ├── deployment.yaml
│   │   └── service.yaml
│   ├── Database
│   │   ├── deployment.yaml
│   │   ├── pvc.yaml
│   │   ├── pv.yaml
│   │   ├── secrets.yaml
│   │   └── service.yaml
│   ├── Frontend
│   │   ├── deployment.yaml
│   │   └── service.yaml
│   └── eks
│       ├── cluster.yaml
│       ├── eks-trust-policy.json
│       ├── iam_policy.json
│       └── ingress.yaml
└── README.md
```

---

# 🏗️ Architectural Overview

- **Routing Engine:** AWS Application Load Balancer (ALB) distributed dynamically across public availability subnets managed by the `aws-load-balancer-controller`.

- **Compute Layer:** Managed AWS Elastic Kubernetes Service cluster processing containerized logic inside redundant worker nodes (`t2.medium`).

- **Identity & Security:** IAM Roles for Service Accounts (IRSA) maps granular AWS IAM policies down to specific service account runtimes using OpenID Connect (OIDC), avoiding long-term static AWS root access keys.

---

# 🚀 Step-by-Step Deployment Guide

## Step 1: Initialize the Workstation Environment
> **⚠️ Important:** Before running `aws configure`, create an IAM user with **Programmatic/API Access**, generate an **Access Key ID** and **Secret Access Key**, then use those credentials to configure the AWS CLI.

```bash
# Update and configure AWS CLI v2
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
sudo apt install unzip -y
unzip awscliv2.zip
sudo ./aws/install --update
aws configure

# Bootstrap the local Docker platform engine
sudo apt-get update
sudo apt install docker.io -y
sudo chown $USER /var/run/docker.sock

# Install compatible cluster management binaries (kubectl & eksctl)
curl -O https://s3.us-east-1.amazonaws.com/amazon-eks/1.30.0/2024-05-12/bin/linux/amd64/kubectl
chmod +x ./kubectl
sudo mv ./kubectl /usr/local/bin

curl --silent --location \
"https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" \
| tar xz -C /tmp

sudo mv /tmp/eksctl /usr/local/bin
```

---

## Step 2: Create ECR Repositories, Authenticate, and Push Images

Before building or pushing, create your private registries in Amazon ECR, authenticate your local engine, containerize your application tiers, and push them up.


```bash
# 1. Create Private Repositories in Amazon ECR
aws ecr create-repository --repository-name 3-tier-app-backend --region <us-east-1>
aws ecr create-repository --repository-name 3-tier-app-frontend --region <us-east-1>

# 2. Authenticate your local Docker engine to the remote Amazon ECR Registry
aws ecr get-login-password --region <us-east-1> | docker login \
--username AWS \
--password-stdin <211125536240>.dkr.ecr.<us-east-1>.amazonaws.com

# 3. Build, Tag, and Push the Backend Image
cd Application-Code/backend/

docker build -t 3-tier-app-backend .

docker tag 3-tier-app-backend:latest \
<211125536240>.dkr.ecr.<us-east-1>.amazonaws.com/3-tier-app-backend:latest

docker push \
<211125536240>.dkr.ecr.<us-east-1>.amazonaws.com/3-tier-app-backend:latest

# 4. Build, Tag, and Push the Frontend Image
cd ../frontend/

docker build -t 3-tier-app-frontend .

docker tag 3-tier-app-frontend:latest \
<211125536240>.dkr.ecr.<us-east-1>.amazonaws.com/3-tier-app-frontend:latest

docker push \
<211125536240>.dkr.ecr.<us-east-1>.amazonaws.com/3-tier-app-frontend:latest
```

> **⚠️ Important:** Ensure the image references inside your `Kubernetes-Manifests-file/Backend/deployment.yaml` and `Frontend/deployment.yaml` files are updated to point to these new ECR repository URIs.

---

## Step 3: Create the EKS Cluster IAM Role

Before creating the EKS cluster, create an IAM role that allows the Amazon EKS control plane to manage AWS resources on your behalf.

```bash
# 1. Create the IAM role using the trust policy
aws iam create-role \
  --role-name EKS-Cluster-Role \
  --assume-role-policy-document file://eks-trust-policy.json

# 2. Attach the required Amazon EKS Cluster policy
aws iam attach-role-policy \
  --role-name EKS-Cluster-Role \
  --policy-arn arn:aws:iam::aws:policy/AmazonEKSClusterPolicy
```
---

## Step 4: Provision EKS Cluster via Configuration Profile

```bash
cd ../../Kubernetes-Manifests-file/eks/

# Launch the managed infrastructure stack using the explicit configuration file template
eksctl create cluster -f cluster.yaml

# Re-link your local context kubeconfig to handle the active cluster endpoints
aws eks update-kubeconfig --region <us-east-1> --name <three-tier-cluster>

kubectl get nodes
```

---

## Step 5: Launch Cluster Objects & Application Services

```bash
kubectl create namespace workshop

# Navigate back and apply manifest layers
cd ../

kubectl apply -f Database/ -n workshop
kubectl apply -f Backend/ -n workshop
kubectl apply -f Frontend/ -n workshop
```

---

## Step 6: Configure OIDC and Apply Workspace IAM Policy

```bash
# Bind the global OpenID Connect identity discovery engine
eksctl utils associate-iam-oidc-provider \
  --region=<us-east-1> \
  --cluster=<three-tier-cluster> \
  --approve

# Create the policy using our validated local configuration file (Fixes official legacy bugs)
aws iam create-policy \
  --policy-name AWSLoadBalancerControllerIAMPolicy \
  --policy-document file://iam_policy.json

# Bind service account boundaries to the infrastructure layer
eksctl create iamserviceaccount \
  --cluster=<three-tier-cluster> \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --role-name AmazonEKSLoadBalancerControllerRole \
  --attach-policy-arn=arn:aws:iam::<211125536240>:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve \
  --region=<us-east-1>
```

---

## Step 7: Install AWS Load Balancer Controller and Ingress Route

```bash
sudo snap install helm --classic

helm repo add eks https://aws.github.io/eks-charts
helm repo update eks

helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=<three-tier-cluster> \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller

# Publish Ingress routing rules
kubectl apply -f eks/ingress.yaml -n workshop
```

---

## Step 8: Define Local DNS Domain Mapping

```bash
# Extract dynamic AWS Application Load Balancer endpoint
kubectl get ingress -n workshop

# Resolve live dynamic network targets
nslookup <k8s-workshop-mainlb-7f430e3450-2019631899.us-east-1.elb.amazonaws.com>
```

Open your local hosts routing system file:

```bash
sudo vi /etc/hosts
```

```text
<98.95.161.17> <shashank.devopsk8s.com>
```

---

# 🔍 Validation and Verification Metrics

### Controller Ingestion Logs

Run this to confirm the stack reconciles perfectly:

```bash
kubectl logs -n kube-system deployment/aws-load-balancer-controller --tail=20
```

**Success confirmation matches:**

```text
{"level":"info", ..., "msg":"successfully deployed model"}
```

### Path Endpoints Behavior

Browsing:

```text
http://<shashank.devopsk8s.com>
```

brings up the Frontend layer.

Accessing the base API endpoint:

```text
http://<shashank.devopsk8s.com>/api
```

returns the Express framework default response:

```text
Cannot GET /api
```

This indicates correct routing onto the backend cluster.

To retrieve database objects, query the API routes directly, for example:

```text
http://<shashank.devopsk8s.com>/api/tasks
```

---

# ⚠️ Real Post-Deployment Troubleshooting & Sandbox Learnings

## 1. The Implicit Node Instance IAM Role Bug

### The Problem

Attempting to build out the environment directly via inline CLI command parameters without specifying explicit underlying Cluster IAM Role:

```bash
eksctl create cluster \
  --name three-tier-cluster \
  --region us-east-1 \
  --node-type t2.medium \
  --nodes-min 2 \
  --nodes-max 2
```

### The Error

```text
ERROR: User: arn:aws:iam::<211125536240>:user/kk_labs_user_xxxxxx is not authorized to perform: iam:PassRole on resource: arn:aws:iam::<211125536240>:role/<Generated-Node-Role>
```

This `iam:PassRole` denial prevents Amazon EKS from assigning the generated IAM role to the worker nodes during cluster provisioning. As a result, the nodes fail to join the cluster, eventually causing the deployment to terminate with:

```text
[✖] cannot create nodegroup "ng-1": "Instances failed to join the kubernetes cluster"
```

### The Fix

We provision the cluster using a declarative `cluster.yaml` configuration and pre-create the `EKS-Cluster-Role` using the `eks-trust-policy.json` trust policy. This approach establishes the required IAM resources before the Amazon EKS control plane provisions the cluster, avoiding the sandbox's `iam:PassRole` restriction and allowing the worker nodes to register successfully.

---

## 2. Official IAM Policy vs. Our Workspace Policy Deficit

### The Problem

The standard template downloaded directly from the official AWS LBC documentation endpoint completely lacks the permission actions required to work inside restricted sandbox environments.

Using the official file causes the controller pods to run into an infinite loop of `AccessDenied` errors when trying to register the ALB:

```text
"msg":"reconcile failed","error":"AccessDenied: User ... is not authorized to perform: elasticloadbalancing:DescribeListenerAttributes"
```

### The Fix

The `Kubernetes-Manifests-file/eks/iam_policy.json` file included in this repository contains the working, patched policy configuration.

We explicitly appended the missing parameters needed to satisfy strict Service Control Policies (SCPs):

- `elasticloadbalancing:DescribeListenerAttributes`
  - Added to the primary statement description block.

- `elasticloadbalancing:ModifyListenerAttributes`
  - Added to the final rule block.

Using our local file to overwrite the active AWS console permission layer allows the controller to reconcile flawlessly:

```bash
aws iam create-policy-version \
  --policy-arn arn:aws:iam::<211125536240>:policy/AWSLoadBalancerControllerIAMPolicy \
  --policy-document file://iam_policy.json \
  --set-as-default

kubectl rollout restart deployment/aws-load-balancer-controller -n kube-system
```

---

## 3. Simulating Private Ingress and DNS Targets Locally

### The Problem

The ingress rules process routing for an explicit production host string (`<shashank.devopsk8s.com>`), but purchasing a live domain and configuring AWS Route 53 Hosted Zones is not possible inside temporary cloud sandbox labs.

### The Fix

We bypassed commercial registrars completely using local hosts mapping.

By running `nslookup` on our dynamic ALB name:

```text
<k8s-workshop-mainlb-7f430e3450-2019631899.us-east-1.elb.amazonaws.com>
```

we intercepted the raw public IPs and bound them inside `/etc/hosts`.

This safely fooled our workstation browser into treating the sandbox network as the live destination route over standard HTTP.

---

# 🧹 Infrastructure Cleanup

To safely clear all infrastructure components and minimize running costs:

```bash
eksctl delete cluster \
  --name <three-tier-cluster> \
  --region <us-east-1>
```
🚀 End-to-End CI/CD Pipeline with GitOps on Kubernetes

This project is part of my On-Premises DevOps Platform for E-commerce series.

It demonstrates a production-style CI/CD and GitOps workflow running on a High-Availability Kubernetes cluster, fully on-premises.

🏗 Architecture Overview

<img width="1755" height="938" alt="image" src="https://github.com/user-attachments/assets/756fadfe-bff6-4e16-a3e1-2a50426f2697" />

Code → CI → Container → Registry → GitOps → Kubernetes → Production

GitLab – Source code management

Jenkins – CI automation

Harbor – Private container registry

Argo CD – GitOps continuous delivery

Kubernetes (HA) – Application runtime

HashiCorp Vault – Secrets management via External Secrets Operator

⚙️ Setup Guide
1️⃣ GitLab Setup

Create repository for:

Application source code

GitOps repository

Configure Webhook to Jenkins

Enable Container Registry (optional if not using Harbor directly)

Guide:
👉 /docs/setup-gitlab.md

2️⃣ Jenkins Setup

Install Jenkins on-prem

Install required plugins:

Git

Docker

Pipeline

Configure:

Docker credentials

Harbor credentials

Create Pipeline Job pointing to:

cicd/Jenkinsfile.prod

Guide:
👉 /docs/setup-jenkins.md

3️⃣ Harbor Setup

Deploy Harbor

Create Project

Create Robot Account

Configure image retention policy

Guide:
👉 /docs/setup-harbor.md

4️⃣ Kubernetes Cluster (HA)

3-node control plane (kubeadm)

Calico CNI

Ingress NGINX

Rancher (optional)

Guide:
👉 /docs/setup-kubernetes-ha.md

5️⃣ Argo CD Setup

Install Argo CD

Configure repository access

Apply Application manifest:

kubectl apply -f gitops/argo-app/application.yaml

Guide:
👉 /docs/setup-argo.md

6️⃣ HashiCorp Vault Setup

Deploy Vault

Enable KV secret engine

Store application secrets

Install External Secrets Operator

Configure SecretStore / ClusterSecretStore

Guide:
👉 /docs/setup-vault.md

🔄 CI/CD Flow
CI Pipeline (Jenkins)

Detect changed services

Build Docker image

Tag image using commit SHA

Push image to Harbor

Update image tag in GitOps values.yaml

Commit & push to GitOps repository

Pipeline file:

cicd/Jenkinsfile.prod
GitOps Deployment (Argo CD)

Argo CD detects changes in GitOps repo

Sync Helm chart

Deploy to Kubernetes

Self-heal drifted resources

Application manifest:

gitops/argo-app/application.yaml
🔐 Secret Management Flow

Vault → External Secrets Operator → Kubernetes Secret → Pod

No secrets stored in Git

Secrets injected dynamically at deployment time

Template file:

gitops/charts/app-chart/templates/externalsecret.yaml
📦 Helm Chart Template

Location:

gitops/charts/app-chart/

Contains:

Deployment

Service

Ingress

ExternalSecret

values.yaml

🧪 Environments

Staging

Production

Image tags managed automatically by CI.

🚀 How to Deploy

Push code:

git push origin main

CI will:

Build & push image

Update GitOps repo

Argo CD will:

Auto-sync and deploy

🛡 Security Considerations

Private registry (Harbor)

Vault-managed secrets

No plaintext secrets in Git

Role-based access control (RBAC)

HA control plane

📈 Future Improvements

Add Trivy image scanning

Implement SOPS for Git encryption

Add Prometheus & Grafana monitoring

Implement OPA / Kyverno policies

👤 Author

DucAnh
I am a software engineer on the path to becoming a DevOps engineer.

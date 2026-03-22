# 🚀 End-to-End CI/CD Pipeline with GitOps on Kubernetes

This project is part of my **On-Premises DevOps Platform for E-commerce** series.

It demonstrates a production-style CI/CD and GitOps workflow running on a High-Availability Kubernetes cluster, fully on-premises.

---

## 🏗 Architecture Overview

**Flow:**  
Code → CI → Container → Registry → GitOps → Kubernetes → Production

<img width="1755" height="938" alt="image" src="https://github.com/user-attachments/assets/f8ccc44f-5b64-465a-bbb7-cceb7c6b75a0" />

**Tech Stack:**

- GitLab – Source Code Management  
- Jenkins – CI Automation  
- Harbor – Private Container Registry  
- Kubernetes (3-node HA control plane)  
- Argo CD – GitOps Continuous Delivery  
- HashiCorp Vault – Secrets Management  
- External Secrets Operator – Secret Injection  

---

## ⚙️ Setup Guide

### 1️⃣ GitLab Setup
- Create application repository  
- Create GitOps repository  
- Configure Webhook to Jenkins  

See: `docs/setup-gitlab.md`

---

### 2️⃣ Jenkins Setup
- Install Jenkins  
- Install required plugins (Git, Docker, Pipeline)  
- Configure Harbor credentials  
- Create pipeline pointing to:


cicd/Jenkinsfile.prod


See: `docs/setup-jenkins.md`

---

### 3️⃣ Harbor Setup
- Deploy Harbor  
- Create project  
- Create robot account  
- Configure registry credentials in Jenkins  

See: `docs/setup-harbor.md`

---

### 4️⃣ Kubernetes (HA)
- kubeadm 3-node control plane  
- Calico CNI  
- Ingress NGINX  

See: `docs/setup-kubernetes-ha.md`

---

### 5️⃣ Argo CD Setup


kubectl apply -f gitops/argo-app/application.yaml


See: `docs/setup-argo.md`

---

### 6️⃣ Vault Setup

- Deploy Vault  
- Enable KV secret engine  
- Store application secrets  
- Install External Secrets Operator  
- Configure SecretStore  

See: `docs/setup-vault.md`

---

## 🔄 CI/CD Flow

### CI (Jenkins)

1. Detect changed services  
2. Build Docker image  
3. Tag image with commit SHA  
4. Push image to Harbor  
5. Update image tag in GitOps repo  
6. Commit & push changes  

Pipeline file:


cicd/Jenkinsfile.prod


---

### CD (Argo CD - GitOps)

1. Argo CD detects change in GitOps repo  
2. Sync Helm chart  
3. Deploy to Kubernetes  
4. Self-heal drifted resources  

Application manifest:


gitops/argo-app/application.yaml


---

## 🔐 Secret Management

Vault → External Secrets Operator → Kubernetes Secret → Pod

- No secrets stored in Git  
- Secrets injected dynamically during deployment  

Template file:


[gitops/charts/app-chart/templates/externalsecret.yaml](https://github.com/ducanhduocdochu/helm-microservice-template)


---

## 🚀 How to Deploy

Push code:


git push origin main


CI will:
- Build and push image  
- Update GitOps repo  

Argo CD will:
- Automatically sync and deploy
- Source value helmchart: https://github.com/ducanhduocdochu/value-helm-microservice-gitops

---

## 🛡 Security Considerations

- Private container registry  
- Vault-managed secrets  
- RBAC enabled  
- HA control plane  

---

## 📈 Future Improvements

- Trivy image scanning  
- SOPS integration  
- Monitoring with Prometheus & Grafana  
- Policy enforcement (OPA / Kyverno)  

---

## 👤 Author

DucAnh  
I am a software engineer on the path to becoming a DevOps engineer.

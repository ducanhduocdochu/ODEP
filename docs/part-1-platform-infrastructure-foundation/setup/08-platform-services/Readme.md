# README.md – Setup Vault, Redis, RabbitMQ in Kubernetes with Helm Charts (GitOps)

Repository: https://github.com/ducanhduocdochu/ecommerce-gitops

This document explains how to deploy **Vault**, **Redis**, and **RabbitMQ** to Kubernetes using **Helm charts**.
It supports both:
- **Manual Helm install** (quick verification)
- **GitOps / Argo CD** installation (recommended for production)

---

## 0) Topology / Components

| Component | Purpose | Default Port | Namespace |
|---|---|---:|---|
| Vault | Secrets management | 8200 | `vault` |
| Redis | Cache / session / queue | 6379 | `redis` |
| RabbitMQ | Message broker | 5672 / UI 15672 | `rabbitmq` |

> In production, you should use Ingress + TLS and set up persistence for all services.

---

## 1) Prerequisites

### 1.1 Kubernetes access
You need a working Kubernetes cluster and kubectl access.

```bash
kubectl get nodes -o wide
```

### 1.2 Helm installed
```bash
helm version
```

### 1.3 (Optional but recommended) Argo CD installed
If you want true GitOps deployment, install Argo CD first.

Check:
```bash
kubectl get pods -n argocd
```

---

## 2) Clone repository

```bash
git clone https://github.com/ducanhduocdochu/ecommerce-gitops.git
cd ecommerce-gitops
```

---

## 3) Create namespaces

```bash
kubectl create namespace vault
kubectl create namespace redis
kubectl create namespace rabbitmq
```

Verify:
```bash
kubectl get ns | egrep "vault|redis|rabbitmq"
```

---

# PART A — Helm install (Manual install for quick test)

> Use this part if you want to deploy quickly with Helm and confirm everything works.
> For production GitOps flow, go to PART B.

---

## A1) Install Vault (HashiCorp chart)

### A1.1 Add Helm repo
```bash
helm repo add hashicorp https://helm.releases.hashicorp.com
helm repo update
```

### A1.2 Install Vault with Helm values from repo
> Ensure you have `infrastructure/vault/values.yaml` in this repo.

```bash
helm install vault hashicorp/vault   -n vault   -f infrastructure/vault/values.yaml
```

### A1.3 Verify Vault pods
```bash
kubectl get pods -n vault -o wide
kubectl get svc -n vault
```

### A1.4 Initialize + Unseal Vault
Exec into vault pod:
```bash
kubectl -n vault exec -it vault-0 -- sh
```

Initialize:
```bash
vault operator init
```

Unseal (repeat 3 keys):
```bash
vault operator unseal <UNSEAL_KEY_1>
vault operator unseal <UNSEAL_KEY_2>
vault operator unseal <UNSEAL_KEY_3>
```

Login:
```bash
vault login <ROOT_TOKEN>
```

> IMPORTANT: store your unseal keys + root token securely.

### A1.5 Access Vault UI (port-forward)
```bash
kubectl -n vault port-forward svc/vault 8200:8200
```

Open in browser:
- http://localhost:8200

---

## A2) Install Redis (Bitnami chart)

### A2.1 Add Helm repo
```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
```

### A2.2 Install Redis with Helm values from repo
> Ensure you have `infrastructure/redis/values.yaml`.

```bash
helm install redis bitnami/redis   -n redis   -f infrastructure/redis/values.yaml
```

### A2.3 Verify Redis pods/services
```bash
kubectl get pods -n redis -o wide
kubectl get svc -n redis
```

### A2.4 Get Redis password
```bash
kubectl get secret -n redis redis -o jsonpath="{.data.redis-password}" | base64 -d
echo
```

### A2.5 Test Redis connection
```bash
kubectl run -it --rm redis-client   --image=bitnami/redis:latest   --restart=Never   -n redis -- bash
```

Inside container:
```bash
redis-cli -h redis-master -a <REDIS_PASSWORD>
ping
```

Expected output:
```
PONG
```

---

## A3) Install RabbitMQ (Bitnami chart)

### A3.1 Install RabbitMQ with Helm values from repo
> Ensure you have `infrastructure/rabbitmq/values.yaml`.

```bash
helm install rabbitmq bitnami/rabbitmq   -n rabbitmq   -f infrastructure/rabbitmq/values.yaml
```

### A3.2 Verify RabbitMQ pods/services
```bash
kubectl get pods -n rabbitmq -o wide
kubectl get svc -n rabbitmq
```

### A3.3 Get RabbitMQ username/password
```bash
kubectl get secret -n rabbitmq rabbitmq -o jsonpath="{.data.rabbitmq-username}" | base64 -d
echo
kubectl get secret -n rabbitmq rabbitmq -o jsonpath="{.data.rabbitmq-password}" | base64 -d
echo
```

### A3.4 Access RabbitMQ UI (port-forward)
```bash
kubectl -n rabbitmq port-forward svc/rabbitmq 15672:15672
```

Open in browser:
- http://localhost:15672

Login with username/password above.

---

# PART B — GitOps Deployment (Argo CD) ✅ Recommended

This section describes the recommended GitOps approach:
- Define Helm apps inside Git
- Argo CD syncs them into the cluster automatically

---

## B1) Application manifests layout

Recommended layout:

```
ecommerce-gitops/
├── infrastructure/
│   ├── vault/          # helm chart wrapper or values.yaml
│   ├── redis/
│   └── rabbitmq/
└── argocd-apps/
    ├── vault.yaml
    ├── redis.yaml
    └── rabbitmq.yaml
```

---

## B2) Example ArgoCD Application for Redis

File: `argocd-apps/redis.yaml`
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: redis
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/ducanhduocdochu/ecommerce-gitops
    targetRevision: main
    path: infrastructure/redis
    helm:
      valueFiles:
        - values.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: redis
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

Apply:
```bash
kubectl apply -f argocd-apps/redis.yaml
```

---

## B3) Apply Vault + RabbitMQ ArgoCD apps

```bash
kubectl apply -f argocd-apps/vault.yaml
kubectl apply -f argocd-apps/rabbitmq.yaml
```

Verify:
```bash
kubectl get applications -n argocd
```

---

## B4) Check sync status in ArgoCD UI
ArgoCD UI will show:
- OutOfSync / Synced
- Healthy / Degraded

Useful commands:
```bash
kubectl get pods -n vault
kubectl get pods -n redis
kubectl get pods -n rabbitmq
```

---

# 4) Verification checklist ✅

### Vault
```bash
kubectl get pods -n vault -o wide
kubectl get svc -n vault
kubectl logs -n vault vault-0 --tail=50
```

### Redis
```bash
kubectl get pods -n redis -o wide
kubectl get svc -n redis
```

### RabbitMQ
```bash
kubectl get pods -n rabbitmq -o wide
kubectl get svc -n rabbitmq
```

---

# 5) Best practices / production notes

## 5.1 Persistence
- Vault: persistent storage / HA mode recommended
- Redis: enable persistence if needed (AOF/RDB)
- RabbitMQ: persistence + PVC + replication

## 5.2 Security
- Use Ingress + TLS (cert-manager)
- Do NOT store passwords directly in Git.
  Use:
  - Sealed Secrets
  - External Secrets Operator
  - Vault itself 😄

## 5.3 Monitoring
- Enable metrics and scrape using Prometheus
- Export dashboards to Grafana

---

DONE ✅
Vault, Redis, RabbitMQ deployed using Helm charts in Kubernetes.

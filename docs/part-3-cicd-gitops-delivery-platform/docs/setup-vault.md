# 🔐 HashiCorp Vault + External Secrets Operator Setup

This guide describes how to deploy HashiCorp Vault on Kubernetes and integrate it with External Secrets Operator (ESO) for secure secret injection.

---

## 📌 Prerequisites

- Kubernetes cluster running
- Ingress NGINX installed
- Helm installed
- Namespace for application (e.g. `backend`)

---

# 1️⃣ Install HashiCorp Vault

## Add Helm repository

```bash
helm repo add hashicorp https://helm.releases.hashicorp.com
helm repo update
```
Create namespace
```
kubectl create namespace vault
```
Install Vault (Dev mode for lab)
```
helm install vault hashicorp/vault \
  --namespace vault \
  --set "server.dev.enabled=true" \
  --set "ui.enabled=true"
```
⚠ Dev mode is for learning/lab only. Not production.

Verify Installation
```
kubectl get pods -n vault
```
Check logs:
```
kubectl logs vault-0 -n vault
```
2️⃣ Expose Vault via Ingress

Create Ingress (example):
```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: vault-ingress
  namespace: vault
spec:
  ingressClassName: nginx
  rules:
  - host: vault.ducanh.io.vn
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: vault
            port:
              number: 8200
```
Apply:
```
kubectl apply -f vault-ingress.yaml
```
3️⃣ Install External Secrets Operator
Add repository
```
helm repo add external-secrets https://charts.external-secrets.io
helm repo update
```
Install ESO
```
helm install external-secrets external-secrets/external-secrets \
  -n external-secrets \
  --create-namespace
```
Verify:
```
kubectl get pods -n external-secrets
```
4️⃣ Configure Vault Kubernetes Auth

Exec into Vault pod:
```
kubectl exec -it vault-0 -n vault -- sh
```
Enable Kubernetes auth:
```
vault auth enable kubernetes
```
Configure Kubernetes auth:
```
vault write auth/kubernetes/config \
  token_reviewer_jwt="$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)" \
  kubernetes_host="https://$KUBERNETES_PORT_443_TCP_ADDR:443" \
  kubernetes_ca_cert=@/var/run/secrets/kubernetes.io/serviceaccount/ca.crt
```
5️⃣ Create Vault Policy

Create policy:
```
vault policy write eso-policy - <<EOF
path "secret/data/auth" {
  capabilities = ["read"]
}
EOF
```
Create role:
```
vault write auth/kubernetes/role/eso-role \
  bound_service_account_names=external-secrets \
  bound_service_account_namespaces=external-secrets \
  policies=eso-policy \
  ttl=24h
```
6️⃣ Store Secret in Vault

Example secret:
```
vault kv put secret/auth Jwt__Secret="my-super-secret-key"
```
7️⃣ Create SecretStore (Kubernetes Resource)
```
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: vault-backend
  namespace: backend
spec:
  provider:
    vault:
      server: "http://vault.vault:8200"
      path: "secret"
      version: "v2"
      auth:
        kubernetes:
          mountPath: "kubernetes"
          role: "eso-role"
```
Apply:
```
kubectl apply -f secretstore.yaml
```
8️⃣ Create ExternalSecret
```
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: auth-secret
  namespace: backend
spec:
  refreshInterval: 1m
  secretStoreRef:
    name: vault-backend
    kind: SecretStore
  target:
    name: auth-secret
  data:
  - secretKey: Jwt__Secret
    remoteRef:
      key: auth
      property: Jwt__Secret
```
Apply:
```
kubectl apply -f externalsecret.yaml
```
🔄 Secret Flow

Vault
→ External Secrets Operator
→ Kubernetes Secret
→ Pod

Secrets are dynamically injected at deployment time.
No secrets are stored in Git.

🛡 Security Recommendations

Do NOT use dev mode in production

Enable TLS for Vault

Enable Auto Unseal (KMS)

Use RBAC properly

Restrict network access

✅ Verify Secret Creation
kubectl get secrets -n backend
kubectl describe secret auth-secret -n backend
👤 Author

DucAnh

I am a software engineer on the path to becoming a DevOps engineer.

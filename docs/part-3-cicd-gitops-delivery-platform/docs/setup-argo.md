# 🚀 Argo CD Setup Guide (GitOps Deployment)

This guide describes how to install and configure Argo CD on a Kubernetes cluster and expose it via NGINX Ingress.

---

## 📌 Prerequisites

- Kubernetes cluster running
- Ingress NGINX installed
- Domain name (e.g. `argocd.ducanh.io.vn`)
- DNS record pointing to your cluster node IP

---

# 1️⃣ Create Namespace

```bash
kubectl create namespace argocd
```
2️⃣ Install Argo CD

Apply official manifest:
```
kubectl apply -n argocd \
-f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```
Verify pods:
```
kubectl get pods -n argocd
```
Wait until all pods are Running.

3️⃣ Expose Argo CD via Ingress

Create Ingress resource:
```
kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: argocd-ingress
  namespace: argocd
  annotations:
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
spec:
  ingressClassName: nginx
  rules:
  - host: argocd.ducanh.io.vn
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: argocd-server
            port:
              number: 443
EOF
```
4️⃣ Retrieve Initial Admin Password

Get the admin password:
```
kubectl -n argocd get secret argocd-initial-admin-secret \
-o jsonpath="{.data.password}" | base64 -d
```
Default username:
```
admin
```
5️⃣ Access Argo CD

Open in browser:
```
https://argocd.ducanh.io.vn
```
Login with:
```
Username: admin

Password: (retrieved above)
```
6️⃣ (Optional) Disable HTTPS Redirection

If you encounter redirect issues, patch the service:
```
kubectl patch svc argocd-server -n argocd \
-p '{"spec": {"type": "ClusterIP"}}'
```
7️⃣ Add Git Repository

In Argo CD UI:

Settings → Repositories

Add repository

Provide GitOps repo URL

Add credentials if private

8️⃣ Deploy Application via Manifest

Example:
```
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: ecommerce-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://gitlab.example.com/gitops-repo.git
    targetRevision: main
    path: charts/app
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```
Apply:
```
kubectl apply -f application.yaml
```
✅ Verify Installation
```
kubectl get pods -n argocd
kubectl get ingress -n argocd
```
👤 Author

DucAnh
I am a software engineer on the path to becoming a DevOps engineer.

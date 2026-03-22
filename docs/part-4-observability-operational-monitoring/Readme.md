# Kubernetes Observability Stack Setup  
**Prometheus + Grafana + Loki + Promtail using Helm**

This guide explains how to deploy a complete **observability stack in Kubernetes** using **Helm**, including:

- Prometheus (metrics collection)
- Grafana (visualization)
- Loki (log storage)
- Promtail (log collection)
- ServiceMonitor (scraping metrics from services)

The stack will allow you to monitor **cluster metrics, application metrics, and logs** in a centralized dashboard.

---

# Architecture Overview

<img width="919" height="604" alt="image" src="https://github.com/user-attachments/assets/d44c95cb-23d4-4636-8575-3bb3f5db220c" />

<img width="587" height="897" alt="image" src="https://github.com/user-attachments/assets/f0118ff0-6565-46b1-afef-95186b94ca65" />

---

# Prerequisites

Ensure the following tools are installed:

- Kubernetes cluster (kubeadm / EKS / GKE / AKS)
- kubectl
- Helm v3+

Check versions:

```bash
kubectl version --client
helm version
```
Step 1 — Create Namespace

Create a dedicated namespace for observability.
```
kubectl create namespace observability
```
Step 2 — Add Helm Repositories

Add required Helm repositories.
```
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
```
Step 3 — Install Prometheus Stack

Install kube-prometheus-stack which includes:

- Prometheus

- Grafana

- Alertmanager

- Node Exporter

- kube-state-metrics

Prometheus Operator
```
helm install monitoring prometheus-community/kube-prometheus-stack \
  --namespace observability
```
Check pods:
```
kubectl get pods -n observability
```
Expected components:

- Prometheus

- Grafana

- Alertmanager

- kube-state-metrics

- node-exporter

- prometheus-operator

Step 4 — Access Grafana

Retrieve Grafana admin password:
```
kubectl get secret monitoring-grafana -n observability \
-o jsonpath="{.data.admin-password}" | base64 --decode
```
Port-forward Grafana:
```
kubectl port-forward svc/monitoring-grafana 3000:80 -n observability
```
Open browser:
```
http://localhost:3000
```
Default username:
```
admin
```
Step 5 — Install Loki

Loki is used for log aggregation.

Install Loki using Helm.
```
helm install loki grafana/loki \
  --namespace observability
```
Verify installation:
```
kubectl get pods -n observability
```
Step 6 — Install Promtail

Promtail collects logs from Kubernetes nodes and sends them to Loki.
```
helm install promtail grafana/promtail \
  --namespace observability \
  --set "loki.serviceName=loki"
```
Check daemonset:
```
kubectl get daemonset -n observability
```
Promtail will run on every node.

Step 7 — Connect Loki to Grafana

Open Grafana:
```
http://localhost:3000
```
Navigate to:

Connections → Data Sources

Add Loki datasource.

Configuration:
```
URL: http://loki:3100
```
Save & Test.

Step 8 — Enable ServiceMonitor for Applications

Prometheus uses ServiceMonitor to discover metrics endpoints.

Your application must expose metrics:
```
/metrics
```
Example service:
```
apiVersion: v1
kind: Service
metadata:
  name: auth-service
  namespace: backend
  labels:
    app: auth-service
spec:
  selector:
    app: auth-service
  ports:
  - port: 80
    targetPort: 80
    name: http
```
Step 9 — Create ServiceMonitor

Create a file:
```
servicemonitor-auth.yaml
```
Example configuration:
```
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: auth-service-monitor
  namespace: observability
spec:
  selector:
    matchLabels:
      app: auth-service
  namespaceSelector:
    matchNames:
      - backend
  endpoints:
  - port: http
    path: /metrics
    interval: 15s
```
Apply configuration:
```
kubectl apply -f servicemonitor-auth.yaml
```
Step 10 — Verify Metrics in Prometheus

Port-forward Prometheus:
```
kubectl port-forward svc/monitoring-kube-prometheus-prometheus 9090:9090 -n observability
```
Open:
```
http://localhost:9090
```
Check targets:

Status → Targets

You should see:
```
auth-service-monitor
```
Step 11 — Query Logs in Grafana

Open Grafana → Explore.

Select Loki datasource.

Example query:
```
{namespace="backend"}
```
Filter logs:
```
{app="auth-service"}
```
Step 12 — Common Prometheus Metrics

Useful queries:

CPU usage:
```
sum(rate(container_cpu_usage_seconds_total[5m])) by (pod)
```
Memory usage:
```
container_memory_usage_bytes
```
Pod count:
```
count(kube_pod_info)
```
Step 13 — Verify Observability Stack

Check all resources:
```
kubectl get pods -n observability
```
Check services:
```
kubectl get svc -n observability
```
Check servicemonitors:
```
kubectl get servicemonitors -A
```

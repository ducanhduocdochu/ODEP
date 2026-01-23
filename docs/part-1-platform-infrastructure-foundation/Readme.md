# Part 1 – Platform & Infrastructure Foundation

This part documents the **platform and infrastructure foundation** of the on-premises DevOps platform.

The focus is not on application logic, but on **how the platform is prepared, deployed, and operated**, including networking, Kubernetes runtime, core platform services, observability foundations, and access control.

---

## 🎯 Scope of This Part

This part answers the question:

> **“How is the platform environment built and operated to run large-scale systems reliably?”**

**In scope:**

* Infrastructure preparation
* Kubernetes runtime setup
* Platform services
* Observability foundations
* Data layer (PostgreSQL HA)
* Access and management tooling

---

## 🧭 High-Level Architecture

This part is supported by multiple architecture diagrams, including:

* Overall platform architecture
* Traffic flow & network boundary
* Platform services & data layer separation
* Reliability & observability overlay

---

## 🚀 How to Run / Setup Guide

The setup is documented as **step-by-step building blocks**.
Each step corresponds to a dedicated folder containing configuration, scripts, or manifests.

> ⚠️ This guide assumes a Linux-based on-premises environment and basic familiarity with Kubernetes and networking concepts.

---

### 1️⃣ Prepare Virtual Machines

**Goal:**
Provision and baseline the virtual machines used for the platform.

**Includes:**

* OS installation
* Host networking
* Basic system hardening
* Required packages and dependencies

📁 Folder:
[`setup/01-prepare-vm`](./setup/01-prepare-vm)

---

### 2️⃣ Set Up Edge Load Balancer (NGINX)

**Goal:**
Expose a single controlled entry point from the internet into the platform.

**Includes:**

* NGINX installation
* Basic reverse proxy configuration
* TLS termination (if applicable)

📁 Folder:
[`setup/02-edge-nginx`](./setup/02-edge-nginx)

---

### 3️⃣ Set Up Kubernetes & Ingress NGINX

**Goal:**
Provision the Kubernetes runtime environment.

**Includes:**

* Kubernetes cluster initialization
* Ingress NGINX deployment
* Basic cluster networking setup

📁 Folder:
[`setup/03-kubernetes-ingress`](./setup/03-kubernetes-ingress)

---

### 4️⃣ Set Up Observability Collectors (In-Cluster)

**Goal:**
Enable foundational observability within the Kubernetes cluster.

**Includes:**

* Prometheus (metrics collection)
* Alertmanager (alert processing)
* Promtail (log collection)

📁 Folder:
[`setup/04-observability-collectors`](./setup/04-observability-collectors)

---

### 5️⃣ Set Up PostgreSQL HA (External Data Layer)

**Goal:**
Provide a highly available external data layer, independent from the Kubernetes lifecycle.

**Includes:**

* Primary / standby PostgreSQL
* Replication configuration
* Failover considerations

📁 Folder:
[`setup/05-postgresql-ha`](./setup/05-postgresql-ha)

---

### 6️⃣ Set Up Teleport (Access Plane)

**Goal:**
Centralize and secure access to infrastructure and Kubernetes.

**Includes:**

* Teleport installation
* Node and Kubernetes access
* Role-based access control

📁 Folder:
[`setup/06-teleport`](./setup/06-teleport)

---

### 7️⃣ Set Up Rancher (Cluster Management)

**Goal:**
Provide a management plane for Kubernetes clusters.

**Includes:**

* Rancher deployment
* Cluster registration
* Basic access configuration

📁 Folder:
[`setup/07-rancher`](./setup/07-rancher)

---

### 8️⃣ Set Up Platform Services with Helm

**Goal:**
Deploy shared platform services used by applications.

**Includes:**

* RabbitMQ
* Redis
* Vault
* Helm-based deployments

📁 Folder:
[`setup/08-platform-services`](./setup/08-platform-services)

---

### 9️⃣ Set Up Loki & Grafana (External Observability Backend)

**Goal:**
Provide centralized log storage and visualization.

**Includes:**

* Loki deployment
* Grafana deployment
* Datasource configuration

📁 Folder:
[`setup/09-loki-grafana`](./setup/09-loki-grafana)

---

## 🧠 Design Notes

* Kubernetes is treated as a **runtime**, not a data store.
* Data and observability backends are intentionally **externalized**.
* Internal APIs are not exposed publicly.
* Reliability and observability are considered **platform-wide concerns**.

---

## ⚠️ Disclaimer

The configurations and setups in this part are based on personal experimentation and learning.
They are intended for educational and reference purposes and may require adaptation for different environments.

---

## 📌 Status

🚧 This part is actively evolving as the platform matures.

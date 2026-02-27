# On-Premises DevOps Platform

This repository documents a personal **on-premises DevOps platform** project, built as a hands-on exploration of **platform engineering, infrastructure design, and operational practices** for running scalable systems.

The project is **evolving** and **not positioned as a fully completed end-to-end solution**.
Instead, it focuses on building a **solid, production-oriented foundation**, applying real-world DevOps and platform engineering principles in an on-premises environment.

The content is organized into multiple parts, each representing a **clear responsibility area** within the platform.

---

## 🎯 Project Goals

* Design and document a **production-minded on-premises DevOps platform**
* Practice **platform engineering thinking** rather than application-centric design
* Apply **clear separation of concerns** across infrastructure, delivery, and operations
* Improve understanding of **reliability, observability, and security**
* Build a structured learning path from **software engineering toward DevOps / platform engineering**

---

## 🧩 Repository Structure

```
onprem-devops-platform/
├── docs/
│   ├── part-1-platform-infrastructure-foundation/
│   ├── part-2-core-microservices-architecture/
│   ├── part-3-cicd-gitops-delivery-platform/
│   ├── part-4-observability-operational-monitoring/
│   └── part-5-devsecops-practices-security-integration/
│
└── README.md
```

Each part can be read independently, but together they describe a cohesive **platform-oriented system**.

---

## 📘 Part 1 – Platform & Infrastructure Foundation

**Folder:**
[`docs/part-1-platform-infrastructure-foundation`](docs/part-1-platform-infrastructure-foundation)

**Focus:**
How the platform is deployed, isolated, and operated at the infrastructure level.

<img width="1431" height="1094" alt="Part1-a" src="https://github.com/user-attachments/assets/79b6a243-ff62-4aa9-b056-b867b937e3f3" />
<img width="1567" height="1001" alt="Part1-b" src="https://github.com/user-attachments/assets/f696ada9-6e5a-404e-b528-a0b4f3edf221" />
<img width="1342" height="1072" alt="Part1-c" src="https://github.com/user-attachments/assets/f12c8a20-a3ad-4c3a-87ab-a37b9e02725e" />

**Key topics:**

* On-premises Kubernetes runtime
* Network boundaries (public vs private)
* Edge load balancing and internal API gateway
* Platform services (Vault, Redis, RabbitMQ)
* External PostgreSQL HA
* Foundational observability components

**Key diagrams:**

* Overall platform & infrastructure architecture
* Traffic flow and network boundary
* Platform services & data layer separation
* Reliability & observability overlay

---

## 📗 Part 2 – Core E-commerce Microservices Architecture

**Folder:**
[`docs/part-2-core-microservices-architecture`](docs/part-2-core-microservices-architecture)

**Focus:**
How the business system is designed to scale, evolve, and remain maintainable.

<img width="1482" height="1072" alt="Part2-a" src="https://github.com/user-attachments/assets/6d5353ab-56ae-4512-8e26-194cbe47b31f" />
<img width="1246" height="794" alt="Part2-b" src="https://github.com/user-attachments/assets/15958e29-5624-4ca1-9588-94acfb223127" />
<img width="1234" height="802" alt="Part2-c" src="https://github.com/user-attachments/assets/785f7273-262e-4d41-839a-478a855ece32" />
<img width="1244" height="760" alt="Part2-d" src="https://github.com/user-attachments/assets/243f9922-7d89-4177-ae93-49f8be5e49a6" />

**Key topics:**

* Microservices architecture (Auth, Product, Order, Payment, Inventory)
* REST and gRPC communication
* Asynchronous messaging with RabbitMQ
* Saga pattern
* Redis caching
* Scheduled jobs (e.g. discount processing)
* Database-per-service (logical separation)

> This part intentionally avoids deep infrastructure and CI/CD details.

---

## 📙 Part 3 – CI/CD & GitOps Delivery Platform

**Folder:**
[`docs/part-3-cicd-gitops-delivery-platform`](docs/part-3-cicd-gitops-delivery-platform)

**Focus:**
How code is built, packaged, and delivered to the platform in a controlled and repeatable way.

<img width="1755" height="938" alt="image" src="https://github.com/user-attachments/assets/18d15089-cd6f-4eef-8849-38d4bc935938" />

**Key topics:**

* GitLab for source control
* Jenkins for CI pipelines
* Harbor as container registry
* Helm for application packaging
* Argo CD for GitOps-based deployment
* Environment promotion strategies
* Vault for secret management

> Application logic and runtime internals are intentionally abstracted.

---

## 📕 Part 4 – Observability & Operational Monitoring

**Folder:**
[`docs/part-4-observability-operational-monitoring`](docs/part-4-observability-operational-monitoring)

**Focus:**
How the platform is observed and operated during runtime.

**Key topics:**

* Metrics collection with Prometheus
* Logging with Promtail and Loki
* Visualization with Grafana
* Alerting with Alertmanager
* Alerting strategies and basic SLO concepts
* Correlation between metrics and logs

> Observability is treated as a platform-wide capability rather than a set of isolated tools.

---

## 📓 Part 5 – DevSecOps Practices & Security Integration

**Folder:**
[`docs/part-5-devsecops-practices-security-integration`](docs/part-5-devsecops-practices-security-integration)

**Focus:**
How security is integrated into the platform and delivery process.

**Key topics:**

* DevSecOps principles and mindset
* Shift-left security practices
* Secrets management with Vault
* Runtime security considerations
* Security as part of platform operations

> This part focuses on practices and integration rather than claiming a fully mature DevSecOps implementation.

---

## 🧠 Design Philosophy

* Platform-first, not application-first
* Clear separation of responsibilities
* Internal APIs are not exposed publicly
* Reliability, observability, and security are platform-wide concerns
* Tooling choices support architectural intent, not the other way around

---

## ⚠️ Disclaimer

This project reflects personal learning, experimentation, and interpretation of real-world system design and DevOps practices.

Readers are encouraged to:

* Selectively evaluate the content
* Cross-check information from multiple sources
* Adapt ideas to their own technical and organizational context

Feedback, discussions, and suggestions are always welcome.

---

## 👤 About Me

I am a software engineer on the path to becoming a **DevOps / platform engineer**, focusing on building a strong foundation in infrastructure, automation, reliability, and operations.

---

## 📌 Project Status

🚧 Actively evolving
📈 New refinements and additional content will be added over time

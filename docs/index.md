# 📚 GCP Professional Cloud Architect — Study Notes

> Structured notes for the **Google Professional Cloud Architect** certification exam.  
> Each module is formatted as a **Service Blueprint**, **Architecture Decision Record**, **Compare & Choose**, or **Scenario Breakdown** — matched to the dominant content type.

---

## 🗂️ Module Index

### 🧱 Foundations

| # | Module | Note Type | Key Topics |
|---|--------|-----------|------------|
| 01 | [Cloud Fundamentals & Core Infrastructure](01-cloud-fundamentals-core-infrastructure.md) | 🏗️ Service Blueprint | IaaS/PaaS, IAM, VPC, Compute Engine, Storage, GKE, Cloud Run |
| 02 | [Essential Core Services](02-essential-core-services.md) | 🏗️ Service Blueprint | Core GCP services overview, resource management |

### ⚙️ Compute & Scaling

| # | Module | Note Type | Key Topics |
|---|--------|-----------|------------|
| 03 | [Elastic Scaling & Automation](03-elastic-scaling-automation.md) | 🧠 ADR | Autoscaling, MIGs, load balancers, preemptible VMs |
| 09 | [Kubernetes Engine (GKE)](09-kubernetes-engine-gke.md) | 🏗️ Service Blueprint | GKE Autopilot vs Standard, workloads, node pools |
| 10 | [Terraform on GCP](10-terraform-gcp.md) | 🧠 ADR | IaC patterns, state management, modules |

### 🌐 Networking

| # | Module | Note Type | Key Topics |
|---|--------|-----------|------------|
| 04 | [Network Fundamentals](04-network-fundamentals.md) | 🏗️ Service Blueprint | VPC, subnets, firewall rules, DNS, routing |
| 05 | [Network Architecture](05-network-architecture.md) | 🧠 ADR | Shared VPC, VPC peering, load balancer tiers |
| 06 | [Hybrid & Multicloud Connectivity](06-hybrid-multicloud-connectivity.md) | ⚖️ Compare & Choose | Interconnect vs VPN vs Peering |

### 🔒 Security & Operations

| # | Module | Note Type | Key Topics |
|---|--------|-----------|------------|
| 07 | [Managing Security on GCP](07-managing-security-gcp.md) | 🧠 ADR | IAM, VPC-SC, CMEK, Security Command Center |
| 11 | [Logging & Monitoring](11-logging-monitoring-gcp.md) | 🏗️ Service Blueprint | Cloud Logging, Cloud Monitoring, alerting, SLOs |
| 13 | [Reliable Infrastructure](13-reliable-infrastructure.md) | 🎯 Scenario Breakdown | HA patterns, DR strategies, SLAs, chaos engineering |

---

## 🧭 Note Type Legend

| Icon | Type | When Used |
|------|------|-----------|
| 🏗️ | **Service Blueprint** | How a GCP service works end-to-end |
| 🧠 | **Architecture Decision Record (ADR)** | When/why to choose an architecture pattern |
| ⚖️ | **Compare & Choose** | GCP service A vs B vs C tradeoffs |
| 🎯 | **Scenario Breakdown** | Exam-style case studies and use cases |

---

## 🎯 Quick Exam Reference

| If the question says... | Think... |
|-------------------------|----------|
| Global SQL + high availability | Cloud Spanner |
| Serverless containers, HTTP | Cloud Run |
| Event-driven functions | Cloud Run Functions |
| Managed Kubernetes | GKE Autopilot |
| On-prem to GCP, dedicated | Dedicated Interconnect |
| On-prem to GCP, shared | Partner Interconnect |
| Cheap encrypted backups | Cloud Storage — Archive |
| IoT/time-series at scale | Bigtable |
| Real-time mobile sync | Firestore |
| Restrict a service account's access | Predefined IAM role, least privilege |
| Infrastructure as Code on GCP | Terraform + remote state in GCS |

---

*Last updated: May 2026 · Missing modules: 08, 12 (in progress)*

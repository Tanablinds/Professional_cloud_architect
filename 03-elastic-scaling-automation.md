---
title: GCP Elastic Cloud Infrastructure — Scaling & Automation
tags: [gcp, networking, vpn, interconnect, load-balancing, autoscaling, terraform, dataflow, dataproc, pca]
note_type: Compare & Choose
source: Elastic Cloud Infrastructure — Scaling & Automation (Instructor-led)
exam_domain: Networking, Compute, Infrastructure Automation, Data Processing
---

# ⚖️ GCP Elastic Cloud Infrastructure: Scaling & Automation

> **Note Type:** Compare & Choose  
> **Why this type?** Every module in this PDF is structured around decision frameworks — comparing connectivity options, load balancer types, autoscaling policies, IaC tools, and data processing services. The material is built for choosing the right option given requirements.

---

## 📦 What This Module Covers

| # | Module | Decision Theme |
|---|--------|---------------|
| 1 | Interconnecting Networks | Which connectivity option fits your hybrid/multi-cloud need |
| 2 | Load Balancing & Autoscaling | Which load balancer type and autoscaling policy to use |
| 3 | Infrastructure Automation | Terraform IaC patterns and Cloud Marketplace |
| 4 | Managed Services | BigQuery, Dataflow, Dataprep, Dataproc — when to use each |

---

## Module 1 — Interconnecting Networks

### The 5 Ways to Connect to Google Cloud

```
On-premises / Other Cloud → Google Cloud
  ├── Cloud VPN            (encrypted, over internet)
  ├── Dedicated Interconnect (physical direct connection)
  ├── Partner Interconnect   (via service provider)
  ├── Cross-Cloud Interconnect (to another cloud provider)
  ├── Direct Peering         (Google public IPs, no SLA)
  └── Carrier Peering        (via ISP, Google public IPs, no SLA)
```

---

### ⚖️ Interconnect Options Compared

| Option | Bandwidth | Access Type | Requires | SLA |
|---|---|---|---|---|
| **Cloud VPN** | 1.5–3 Gbps/tunnel | Internal RFC1918 IPs via VPC | VPN device on-prem | ✅ Yes |
| **Dedicated Interconnect** | 10 or 100 Gbps/link (up to 8 links) | Internal RFC1918 IPs via VPC | Colocation at Google-supported facility | ✅ Yes |
| **Partner Interconnect** | 50 Mbps – 50 Gbps/connection | Internal RFC1918 IPs via VPC | Service provider relationship | ✅ Yes |
| **Cross-Cloud Interconnect** | 10 or 100 Gbps/connection | Internal RFC1918 IPs via VPC | Ports at both Google Cloud + remote cloud provider | ✅ Yes |
| **Direct Peering** | 10 Gbps/link | Google **public** IPs only | Connection at Google Cloud PoP | ❌ No SLA |
| **Carrier Peering** | Varies by ISP | Google **public** IPs only | Service provider | ❌ No SLA |

> **Key split:** Interconnect options = private RFC1918 VPC access + SLA.  
> Peering options = Google public IPs only, no SLA.

---

### ⚖️ Peering Options Compared

| Option | Connection | Capacity | Access | SLA |
|---|---|---|---|---|
| **Direct Peering** | Direct BGP exchange at Google PoP | 10 Gbps/link | All Google services (public IPs) | ❌ No |
| **Carrier Peering** | Through ISP to Google's network | Varies | Google Workspace, APIs (public IPs) | ❌ No |

> 💡 Google recommends **Cloud Interconnect over peering** for production workloads. Use peering only to access Workspace/YouTube/APIs without needing VPC private IP access.

---

### Decision Trees: Choosing a Connection

#### Connect on-premises to Google Cloud VPC

```
Need dedicated enterprise-grade connection?
├── YES, can colocate at Google facility  →  Dedicated Interconnect
├── YES, no colocation available          →  Partner Interconnect
└── NO (lower cost / dev/test / encrypt)  →  Cloud VPN
```

#### Connect to Google Workspace / public APIs only

```
Meet Google's Direct Peering requirements?
├── YES  →  Direct Peering
└── NO   →  Carrier Peering
```

#### Connect Google Cloud to another cloud provider

```
→ Cross-Cloud Interconnect
  (dedicated physical connection between VPCs across clouds)
  Options: Google-managed routing (Cloud VPN) OR self-managed connectivity
```

---

### HA VPN vs Classic VPN

| | **Classic VPN** | **HA VPN** |
|---|---|---|
| Tunnels | 1 tunnel, 1 external IP | 2 tunnels, 2 external IPs |
| SLA | 99.9% | **99.99%** |
| Routing | Static or dynamic | Dynamic (BGP required) |
| Best for | Dev/test, simple setups | Production HA hybrid connectivity |

**Cloud Router** enables **dynamic routing (BGP)** for HA VPN — automatically propagates route changes between your on-prem and GCP networks without manual static route updates.

---

### Shared VPC vs VPC Network Peering

| Consideration | **Shared VPC** | **VPC Network Peering** |
|---|---|---|
| Across organisations | ❌ No | ✅ Yes |
| Within same project | ❌ No | ✅ Yes |
| Cross-project, same org | ✅ Yes | ✅ Yes |
| Network administration | Centralised (one host VPC) | Decentralised (each VPC manages itself) |
| Firewall rules | Managed centrally in host VPC | Each VPC maintains its own firewall |
| Use case | Enterprise with central networking team | SaaS provider ↔ customer, separate admin domains |

**Shared VPC:** Host project shares its VPC with service projects. One network team controls security policy centrally.

**VPC Peering:** Two VPCs exchange routes privately without traversing the public internet. No SLA overhead, no VPN latency. Each side controls its own firewall.

---

## Module 2 — Load Balancing & Autoscaling

### Managed Instance Groups (MIGs)

A **MIG** is a collection of identical VM instances managed as one entity via an **instance template**.

#### What MIGs give you

| Feature | Detail |
|---|---|
| **Auto-healing** | Detects unhealthy instances via health check → auto-recreates |
| **Autoscaling** | Add/remove instances based on load signal |
| **Rolling updates** | Update all instances with a new template, zero downtime |
| **Load balancing integration** | Register MIG as a backend for any Cloud LB |
| **Zone distribution** | Regional MIGs span multiple zones → survive zonal failure |

> ✅ **Always prefer regional MIGs** over zonal for production — protects against zonal outage.

#### MIG Creation Steps

```
1. Create instance template (machine type, disk, network, startup script)
2. Create MIG → select template
3. Choose: stateless (web, batch) OR stateful (DB, legacy app)
4. Choose: single-zone OR regional (multi-zone)
5. Configure autoscaling policy
6. Attach health check
```

---

### Autoscaling Policies

| Policy | Signal | Use Case |
|---|---|---|
| **CPU utilisation** | Average CPU % across MIG | General compute workloads |
| **Load balancing capacity** | LB utilisation target | HTTP/HTTPS serving workloads |
| **Monitoring metric** | Any Cloud Monitoring metric (built-in or custom) | Custom app metrics (e.g., active users, queue depth) |
| **Queue-based (Pub/Sub)** | Number of undelivered messages | Batch processing, async pipelines |
| **Schedule-based** | Time/date/recurrence | Predictable traffic patterns (e.g., business hours) |

> 💡 Use **custom metrics** when CPU/LB capacity don't correlate well with actual load (e.g., a game server where active session count matters more than CPU).

---

### ⚖️ Load Balancer Decision Framework

#### Step 1 — What traffic type?

```
HTTP or HTTPS?   →  Application Load Balancer  (Layer 7)
TCP/UDP/other?   →  Network Load Balancer      (Layer 4)
```

#### Step 2 — External or Internal?

```
Internet-facing?  →  External
Within VPC only?  →  Internal
```

#### Step 3 — Global or Regional?

```
Backends in multiple regions / global anycast IP?  →  Global (Premium Tier)
Single region only?                                →  Regional (Standard Tier)
```

#### Step 4 — Proxy or Passthrough? (Network LB only)

```
Need TLS offload / TCP proxy?           →  Proxy Network LB
Need to preserve client IP / UDP / ESP? →  Passthrough Network LB
```

---

### ⚖️ Full Load Balancer Comparison Table

| Load Balancer | Traffic | Scope | Scheme | Use Case |
|---|---|---|---|---|
| **Global External Application LB** | HTTP/HTTPS | Global | EXTERNAL_MANAGED | Global web apps, multi-region failover, anycast IP |
| **Regional External Application LB** | HTTP/HTTPS | Regional | EXTERNAL_MANAGED | Single-region HTTPS apps (Standard tier) |
| **Classic Application LB** | HTTP/HTTPS | Global/Regional | EXTERNAL | Legacy apps (migration path to managed) |
| **Internal Application LB** | HTTP/HTTPS | Regional | INTERNAL_MANAGED | Internal microservices, service mesh |
| **Global External Proxy Network LB** | TCP + SSL offload | Global | EXTERNAL | TCP/SSL apps across regions |
| **Regional External Proxy Network LB** | TCP | Regional | EXTERNAL_MANAGED | Single-region TCP apps |
| **Internal Proxy Network LB** | TCP | Regional | INTERNAL_MANAGED | Internal TCP services |
| **External Passthrough Network LB** | TCP/UDP/ESP/ICMP | Regional | EXTERNAL | Preserve client IP, raw protocol support |
| **Internal Passthrough Network LB** | TCP/UDP | Regional | INTERNAL | Internal services needing client IP preservation |

> 💡 **3-tier web services tip:** Combine an **External Application LB** (internet-facing) + **Internal Application LB** (between app and data tier). This keeps backend traffic private.

---

### Key Load Balancing Concepts

#### Network Endpoint Groups (NEGs)

Defines a group of **backend endpoints** more granularly than a full VM.

| NEG Type | Endpoints | Use Case |
|---|---|---|
| **Zonal NEG** | VMs or containers (GKE pods) by IP:port | Fine-grained container load balancing |
| **Serverless NEG** | Cloud Run, App Engine, Cloud Functions | Route LB traffic to serverless backends |
| **Internet NEG** | External (non-GCP) IP:port | Hybrid: LB in GCP, backend on-prem |
| **Private Service Connect NEG** | Published services via PSC | Service-to-service with private endpoints |

#### Cloud CDN

- Caches content at Google's global edge PoPs → reduces latency for repeat requests
- Sits in front of **External Application Load Balancer**
- **Cache modes:**

| Mode | Behaviour |
|---|---|
| **USE_ORIGIN_HEADERS** | Respects `Cache-Control` headers from backend |
| **CACHE_ALL_STATIC** | Automatically caches static content regardless of headers |
| **FORCE_CACHE_ALL** | Caches all responses (overrides `no-store`) |

---

### Backend Services

A **backend service** defines how the LB distributes traffic to backends.

Key settings:
- **Balancing mode:** `UTILIZATION` (CPU %), `RATE` (req/sec), `CONNECTION` (active connections)
- **Health checks:** Required; defines which backends are healthy
- **Session affinity:** Stick requests from same client to same backend (cookie, client IP, header)
- **Timeout:** Backend response timeout

---

## Module 3 — Infrastructure Automation

### Why Infrastructure as Code (IaC)?

| Benefit | Detail |
|---|---|
| **Repeatability** | Same config → identical environments every time |
| **Idempotency** | Apply same config multiple times, same result |
| **Dev/Test parity** | Replicate prod in dev with one command |
| **Disaster recovery** | Config files = runbook for re-deploying everything |
| **CI/CD integration** | Infra changes go through pull requests and pipelines |
| **Dependency management** | IaC tools figure out creation order |

---

### Terraform on GCP

Terraform uses **HashiCorp Configuration Language (HCL)** — declarative, human-readable.

#### Core Concepts

| Concept | Description |
|---|---|
| **Provider** | Plugin that talks to a cloud API (e.g., `google` provider) |
| **Resource** | A GCP object (VM, VPC, firewall, bucket…) |
| **Variable** | Parameterises your config |
| **Output** | Exposes values after apply (e.g., IP address) |
| **Module** | Reusable group of resources |
| **State file** | `terraform.tfstate` — Terraform's record of what it created |

#### HCL Syntax

```hcl
resource "google_compute_network" "default" {
  name                    = var.network_name
  auto_create_subnetworks = false
}

resource "google_compute_firewall" "allow_http" {
  name    = "allow-http"
  network = google_compute_network.default.name

  allow {
    protocol = "tcp"
    ports    = ["80"]
  }
}
```

#### Terraform Lifecycle Commands

```bash
terraform init    # Download provider plugins, initialise working directory
terraform plan    # Preview changes — what will be created/modified/destroyed
terraform apply   # Execute the plan — provision actual infrastructure
terraform destroy # Tear down all resources defined in config
```

> 💡 Always run `terraform plan` before `apply` — treat it like `git diff` for infrastructure.

#### IaC Tools Supported by GCP

| Tool | Style | Best For |
|---|---|---|
| **Terraform** | Declarative (HCL) | Multi-cloud, most popular, rich ecosystem |
| **Ansible** | Procedural (YAML) | Configuration management + provisioning |
| **Chef / Puppet** | Declarative/procedural | Legacy enterprise config management |
| **Packer** | Declarative | Building machine images (VM, containers) |

---

### Google Cloud Marketplace

- Deploy **production-grade third-party solutions** with one click (LAMP, WordPress, MongoDB, etc.)
- Billed on a **single Google Cloud bill** — no separate vendor invoices
- Solutions can be **managed via Terraform** (infrastructure definitions available)
- Receive **security update notifications** automatically

---

## Module 4 — Managed Services (Data Processing)

### Overview

Managed services exist on a spectrum between **PaaS and SaaS** — you focus on your data pipeline, Google manages the infrastructure.

| Service | What it does | Open source roots |
|---|---|---|
| **BigQuery** | Serverless data warehouse — SQL analytics at petabyte scale | N/A (proprietary) |
| **Dataflow** | Serverless stream + batch data pipeline execution | Apache Beam |
| **Dataprep** | Visual data exploration, cleaning, and preparation | Trifacta Wrangler |
| **Dataproc** | Managed Spark and Hadoop cluster execution | Apache Spark / Hadoop |

---

### BigQuery

- **Serverless, fully managed** data warehouse
- Petabyte-scale; sub-second query performance via columnar storage + Dremel
- Standard SQL interface — no DBA required
- Integrated billing, IAM, audit logging
- Supports **BigQuery ML** (train ML models with SQL), **BigQuery BI Engine** (in-memory analysis), **Omni** (query data in S3/Azure)

```sql
-- Example: Query across 1TB table with no infrastructure config
SELECT department, SUM(revenue) AS total
FROM `project.dataset.sales`
WHERE EXTRACT(YEAR FROM date) = 2024
GROUP BY department
ORDER BY total DESC;
```

---

### Dataflow

- **Serverless, autoscaling** pipeline execution engine
- Built on **Apache Beam SDK** (Java, Python, SQL)
- Handles both **stream** (real-time) and **batch** (historical) with the same pipeline code
- Scales automatically to millions of queries per second
- Integrates with Pub/Sub (input), BigQuery/Bigtable (output), Cloud Monitoring (observability)

#### Dataflow Architecture Pattern

```
Source         →   Transform (Beam SDK)   →   Sink
Pub/Sub            Filter, enrich,            BigQuery
Cloud Storage      aggregate, join            Bigtable
Apache Kafka       window operations          Cloud Storage
Datastore                                     Vertex AI
```

---

### Dataprep (by Trifacta)

- Visual, no-code interface for **data cleaning and preparation**
- Serverless, scales automatically
- Auto-suggests transformations based on data patterns
- Auto-detects schema, data types, joins, anomalies
- Outputs to BigQuery or Cloud Storage via Dataflow pipeline
- Partner service (Trifacta) — no separate license or install

---

### Dataproc

- Managed **Apache Spark and Hadoop** clusters
- Fast cluster start/scale/shutdown: **< 90 seconds**
- Per-second billing + preemptible VM support → cheapest Spark/Hadoop in the cloud
- Built-in integration: BigQuery, Cloud Storage, Bigtable, Cloud Logging, Cloud Monitoring
- No code changes needed if you're already running Spark/Hadoop/Pig/Hive

---

### ⚖️ Dataflow vs Dataproc

| Question | If YES → | If NO → |
|---|---|---|
| Dependencies on Spark/Hadoop ecosystem tools? | **Dataproc** | Continue |
| Prefer manual cluster control (DevOps model)? | **Dataproc** | **Dataflow** |
| Want fully serverless, no cluster management? | **Dataflow** | Continue |

```
Have Hadoop/Spark dependencies?
├── YES  →  Dataproc
└── NO
      └── Manual cluster control preferred?
            ├── YES  →  Dataproc
            └── NO   →  Dataflow
```

| | **Dataflow** | **Dataproc** |
|---|---|---|
| Cluster management | ❌ None (serverless) | ✅ You manage clusters |
| Programming model | Apache Beam (unified batch + stream) | Spark / Hadoop / Pig / Hive |
| Best for | New pipelines, unified stream+batch | Migrating existing Spark/Hadoop jobs |
| Scaling | Fully automatic | Manual or autoscaling |
| Cost model | Per vCPU/hr while pipeline runs | Per-second billing, supports preemptibles |

---

### ⚖️ Full Data Service Summary

| Need | Service |
|---|---|
| Run SQL analytics on petabyte data | **BigQuery** |
| Build a streaming or batch pipeline (new) | **Dataflow** (Apache Beam) |
| Migrate existing Spark/Hadoop workloads | **Dataproc** |
| Clean and prepare raw data visually | **Dataprep** |
| Real-time ingestion pipeline → BigQuery | Pub/Sub → **Dataflow** → BigQuery |
| Ad-hoc SQL on semi-structured data | **BigQuery** |
| ML model training on structured data with SQL | **BigQuery ML** |

---

## 🎯 PCA Exam Scenarios to Know

| Scenario | Answer |
|---|---|
| Low-cost, encrypted hybrid connection for dev/test | Cloud VPN |
| Enterprise connection; 10 Gbps+; colocation available | Dedicated Interconnect |
| Enterprise connection; no colocation; need SLA | Partner Interconnect |
| Connect GCP VPC to AWS VPC with dedicated bandwidth | Cross-Cloud Interconnect |
| Two VPCs in different orgs need private RFC1918 comms | VPC Network Peering |
| Central network team managing multiple project networks | Shared VPC |
| Global HTTP/S app with multi-region failover | Global External Application LB (Premium Tier) |
| Internal microservice-to-microservice HTTP traffic | Internal Application LB |
| Preserve client IP address in load balancing | Passthrough Network Load Balancer |
| VM fleet needs to scale based on Pub/Sub queue depth | MIG + autoscaling with queue-based (Pub/Sub) policy |
| Deploy identical infra across dev/staging/prod consistently | Terraform (IaC) |
| Existing Spark ETL job; migrate to GCP with minimal changes | Dataproc |
| New pipeline ingesting Pub/Sub events → BigQuery | Dataflow (Apache Beam) |
| Non-technical analyst needs to clean raw CSV data | Dataprep |

---

## 🔗 Key CLI & Terraform Reference

```bash
# HA VPN
gcloud compute vpn-gateways create ha-vpn-gw --network=my-vpc --region=us-central1
gcloud compute vpn-tunnels create tunnel-1 --peer-gcp-gateway=PEER_GW --vpn-gateway=ha-vpn-gw

# MIG
gcloud compute instance-templates create my-template --machine-type=e2-medium
gcloud compute instance-groups managed create my-mig \
  --template=my-template --size=3 --zone=us-central1-a

# Autoscaling
gcloud compute instance-groups managed set-autoscaling my-mig \
  --max-num-replicas=10 --target-cpu-utilization=0.75

# Terraform
terraform init && terraform plan && terraform apply

# Dataproc
gcloud dataproc clusters create my-cluster --region=us-central1 --num-workers=2
gcloud dataproc jobs submit spark --cluster=my-cluster --jar=my-job.jar
```

```hcl
# Terraform: Auto-mode VPC + Firewall
resource "google_compute_network" "mynetwork" {
  name                    = "mynetwork"
  auto_create_subnetworks = true
}

resource "google_compute_firewall" "allow_http" {
  name    = "mynetwork-allow-http"
  network = google_compute_network.mynetwork.self_link

  allow {
    protocol = "tcp"
    ports    = ["80"]
  }

  source_ranges = ["0.0.0.0/0"]
}
```

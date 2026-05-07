---
title: GCP Cloud Fundamentals — Core Infrastructure
tags: [gcp, cloud-fundamentals, pca, iam, networking, storage, containers, serverless]
note_type: Service Blueprint
source: Google Cloud Fundamentals — Core Infrastructure (Instructor-led)
exam_domain: Foundation — All PCA Domains
---

# 🏗️ GCP Cloud Fundamentals: Core Infrastructure

> **Note Type:** Service Blueprint  
> **Why this type?** This module is a foundational walkthrough of how GCP core infrastructure services are structured, how they interact, and when to use each. It is primarily descriptive and architectural, not comparative or scenario-driven.

---

## 📦 What This Module Covers

| # | Module | Key Services |
|---|--------|-------------|
| 1 | Introducing Google Cloud | Cloud computing model, IaaS/PaaS, network, security, pricing |
| 2 | Resources and Access in the Cloud | Resource hierarchy, IAM, service accounts, Cloud Identity |
| 3 | Virtual Machines and Networks | VPC, Compute Engine, load balancing, VPN, Interconnect |
| 4 | Storage in the Cloud | Cloud Storage, Cloud SQL, Spanner, Firestore, Bigtable |
| 5 | Containers in the Cloud | Docker, Kubernetes, GKE |
| 6 | Applications in the Cloud | Cloud Run, Cloud Run Functions |
| 7 | Prompt Engineering & LLMs | Prompts, LLM types, Vertex AI |

---

## Module 1 — Introducing Google Cloud

### What is Cloud Computing?

Cloud computing is defined by the US NIST with five key traits:
- **On-demand self-service** — Provision resources without human interaction
- **Broad network access** — Access over the network from any device
- **Resource pooling** — Provider serves multiple customers from shared resources
- **Rapid elasticity** — Scale up/down automatically
- **Measured service** — Pay only for what you consume

### Three Waves of Cloud

```
Wave 1 — Colocation        : Customer-owned hardware in provider's data center
Wave 2 — Virtualised       : Customer-configured VMs in provider's infrastructure
Wave 3 — Container-native  : Fully elastic, autoscaled, serverless (Google Cloud)
```

### Service Model Spectrum

```
IaaS ──────────────────────────── SaaS
(you manage most)         (provider manages all)

IaaS    → VMs, storage, networking (Compute Engine)
PaaS    → Platform + runtime (App Engine, Cloud Run)
SaaS    → End-user software (Google Workspace)
```

**IaaS:** Delivers on-demand infrastructure resources — compute, storage, network. You manage the OS and up.  
**PaaS:** Delivers and manages all the hardware and infrastructure; developer focuses on code only.

### Google Cloud Network

- **Largest network of its kind** — designed for high throughput and low latency
- Traffic stays on Google's private backbone as long as possible, only touching the public internet at the edge
- **Geographic hierarchy:** Multi-regions → Regions → Zones

```
Multi-Region (e.g., US)
  └── Region (e.g., us-central1)
        └── Zone (e.g., us-central1-a)   ← where resources are deployed
```

> 💡 **PCA tip:** Deploy across multiple zones for high availability. Use multiple regions for disaster recovery and global latency.

### Google Cloud Security Model (Layers)

| Layer | What it secures |
|---|---|
| Hardware infrastructure | Custom chips (Titan), secure boot, hardware-level encryption |
| Service deployment | Service identity, inter-service encryption |
| User identity | Google 2-step verification, hardware security keys |
| Storage services | Encryption at rest (default, no config needed) |
| Internet communication | HTTPS/TLS enforced by default |
| Operational security | Red team exercises, intrusion detection, bug bounty |

### Open Source & Interoperability

Google supports **open-source interoperability** to prevent vendor lock-in:
- Kubernetes (open-sourced by Google)
- TensorFlow (works on GCP and outside it)
- Apache Beam, Kubeflow

### Pricing & Billing

- **Compute Engine:** billed **per second** (not per hour), minimum 1 minute
- **Sustained use discounts:** automatic discounts when VMs run for a significant portion of the month
- **Committed use discounts:** 1- or 3-year commitments for deeper savings
- **Billing tools:** Budgets, alerts, cost breakdowns, quotas (allocated at project level)

---

## Module 2 — Resources and Access in the Cloud

### Resource Hierarchy

```
Organization Node (topmost — e.g., your company)
  └── Folder (e.g., department or team)
        └── Project (basis for using all GCP services)
              └── Resources (VMs, buckets, topics, datasets…)
```

**Policy inheritance flows downward** — a policy applied at a higher level cascades to all resources below it.

#### Project Attributes

| Attribute | Type | Notes |
|---|---|---|
| Project ID | Globally unique, immutable | Chosen at creation, cannot change |
| Project Name | Not unique, mutable | Human-readable label |
| Project Number | Globally unique, immutable | Assigned by Google |

> Use **Resource Manager** (API or console) to programmatically list, create, update, or delete projects.

---

### Identity and Access Management (IAM)

**Core concept:** Define **who** can do **what** on **which resource**.

```
Principal  →  Role  →  Permissions on Resource
```

- **Principal (who):** Google Account, Google Group, Service Account, Cloud Identity domain
- **Role (can do what):** A collection of permissions bundled together
- **Resource (which):** Anything in the hierarchy — project, bucket, VM, topic, etc.

#### IAM Policy Inheritance

Policies flow **downward** through the hierarchy. A role granted at the Organization level applies to all folders, projects, and resources below.

**Deny policies** override allow policies — IAM checks deny rules *before* allow rules.

---

### IAM Role Types

| Role Type | Scope | Best Practice |
|---|---|---|
| **Basic** | Entire project — affects all resources | Avoid in production; too broad |
| **Predefined** | Specific service and action | ✅ Recommended for most use cases |
| **Custom** | You define exact permissions | Use when predefined doesn't match |

#### Basic Roles

| Role | Capabilities |
|---|---|
| Viewer | Read-only — examine resources |
| Editor | Read + modify resources |
| Owner | Editor + manage roles/permissions + set billing |
| Billing Admin | Manage billing only — no resource access |

> 💡 **PCA tip:** Always prefer **predefined roles** over basic roles. Basic roles are overly permissive in multi-team environments.

---

### Service Accounts

- A **service account** is an identity for a non-human actor (VM, app, pipeline)
- Identified by an email address (e.g., `my-sa@project.iam.gserviceaccount.com`)
- Managed by IAM — can be granted roles just like users
- **Use case:** A VM needs to write to Cloud Storage → create a service account, grant it `storage.objectCreator` role, attach it to the VM

---

### Cloud Identity

- Manages **user and group policies** across the organisation
- Enables **SSO** with existing identity providers (Active Directory, LDAP)
- If a user leaves, deprovisioning in Cloud Identity removes GCP access automatically

---

### Interacting with GCP

| Interface | Tool | Notes |
|---|---|---|
| Web UI | **Google Cloud Console** | Best for exploration, dashboards |
| CLI | **Cloud SDK** (`gcloud`, `gsutil`, `bq`) | Scripting, automation |
| Browser CLI | **Cloud Shell** | In-browser terminal, always up-to-date |
| APIs | **REST / client libraries** | Programmatic control from your app |
| Mobile | **Google Cloud App** | Start/stop instances, view alerts |

---

## Module 3 — Virtual Machines and Networks

### Virtual Private Cloud (VPC)

- A **VPC is your private, software-defined network** within Google Cloud
- VPCs are **global** — a single VPC spans all regions
- **Subnets are regional** — a subnet exists in one region but can span multiple zones

```
VPC (global)
  └── Subnet (regional, e.g., us-central1)
        ├── VM in us-central1-a
        └── VM in us-central1-b
```

> Resources in the same VPC can communicate privately, even across zones, **without a router**.

#### Firewall Rules

- VPCs include an **implicit deny-all ingress** firewall rule
- Rules are defined based on: IP ranges, tags, service accounts
- No hardware firewall to provision — it's all software-defined

#### VPC Peering and Shared VPC

- **VPC Peering:** Two separate VPCs (even in different projects) can communicate privately
- **Shared VPC:** One host project's VPC is shared with service projects — centralised network management

---

### Compute Engine

**Fully customisable VMs** — choose CPU, memory, GPU, storage independently.

| Feature | Detail |
|---|---|
| Machine types | General, compute-optimised, memory-optimised |
| Persistent disks | Standard (HDD) or SSD — survive VM deletion |
| Local SSD | Highest performance, ephemeral (lost on stop) |
| Preemptible / Spot VMs | Up to 91% cheaper — can be reclaimed by Google |
| Sustained use discount | Automatic; kicks in at ~25% monthly usage |
| Committed use discount | 1- or 3-year commitment — up to 57% discount |

#### Auto-scaling

- **Managed Instance Groups (MIGs):** Group of identical VMs managed together
- Scales in/out based on CPU utilisation, HTTP load, or custom metrics
- Supports rolling updates and health checks

---

### Load Balancing

**Cloud Load Balancing** — fully managed, no pre-warming needed, reacts to traffic spikes automatically.

#### Load Balancer Decision Tree

```
What traffic type?
├── HTTP / HTTPS  →  Application Load Balancer (Layer 7)
│     ├── External Global   → Global Application LB (anycast IP)
│     ├── External Regional → Regional Application LB
│     └── Internal          → Internal Application LB
└── TCP / UDP / other IP  →  Network Load Balancer (Layer 4)
      ├── Proxy (External/Internal)
      └── Passthrough (External/Internal)
```

| Load Balancer | Layer | Use Case |
|---|---|---|
| Global Application LB | L7 | Global HTTPS web apps, multi-region failover |
| Regional Application LB | L7 | Single-region HTTPS apps |
| Internal Application LB | L7 | Microservices internal traffic |
| Network LB (Proxy) | L4 | TCP/SSL non-HTTP traffic |
| Network LB (Passthrough) | L4 | Preserve client IP, UDP traffic |

---

### Connecting to Other Networks

| Option | What it does | Use when |
|---|---|---|
| **Cloud VPN** | Encrypted tunnel over public internet | Dev/test, low-cost hybrid |
| **Direct Peering** | Route traffic through Google PoP | Access Google Workspace + GCP |
| **Carrier Peering** | ISP connects you to Google's network | No direct Google PoP nearby |
| **Dedicated Interconnect** | Physical direct connection to Google | High-bandwidth, low-latency prod |
| **Partner Interconnect** | Via a service provider | Can't reach a Dedicated facility |
| **Cross-Cloud Interconnect** | Connect to another cloud provider | Multi-cloud architecture |

---

## Module 4 — Storage in the Cloud

### Cloud Storage (Object Storage)

- Fully managed, infinitely scalable **object storage**
- Objects are **immutable** — you replace, never modify
- Organised into **globally named buckets**
- Encrypted at rest and in transit by default

#### Storage Classes

| Class | Access Frequency | Min Duration | Use Case |
|---|---|---|---|
| **Standard** | Frequent | None | Hot data, websites, streaming |
| **Nearline** | ~Once/month | 30 days | Backups, infrequently accessed data |
| **Coldline** | ~Once/quarter | 90 days | Disaster recovery archives |
| **Archive** | ~Once/year | 365 days | Long-term regulatory archiving |

> 💡 **Autoclass:** Automatically moves objects to colder/warmer storage based on access pattern — simplifies cost optimisation.

#### Object Lifecycle Management

Define rules to automatically delete or transition objects:
```
If age > 365 days → delete
If age > 30 days AND storage class = Standard → move to Nearline
```

#### Bringing Data to Cloud Storage

| Method | Use When |
|---|---|
| `gcloud storage` / Console drag-drop | Small to medium uploads |
| **Storage Transfer Service** | Large online transfers (TBs/PBs) from S3, HTTP, or another bucket |
| **Transfer Appliance** | Offline — ship a physical device to Google |

---

### Cloud SQL

- Fully managed **relational database** (MySQL, PostgreSQL, SQL Server)
- Handles: patching, backups, replication, failover automatically
- Supports automatic replication (read replicas, cross-region)
- Includes a managed **network firewall**
- Max capacity: **up to 64 TB**

> Best for: web apps, CMS, e-commerce, user credential storage, existing SQL workloads

---

### Cloud Spanner

- Fully managed **globally distributed relational database**
- Combines **SQL semantics** with **horizontal scalability**
- Strong consistency across regions with no downtime
- Capacity: **Petabyte scale**

> Best for: global financial systems, inventory management, ad tech — anywhere you need ACID + horizontal scale

---

### Firestore

- Fully managed **serverless NoSQL document database**
- Supports **real-time synchronisation** (online + offline)
- Optimised for **mobile and web** clients
- Scales automatically
- Max entity size: **1 MB**

> Best for: mobile apps, real-time collaboration, offline-first applications

---

### Bigtable

- Fully managed **wide-column NoSQL database**
- Designed for massive scale: **1 TB+ of structured/semi-structured data**
- Low-latency reads/writes for analytical and operational workloads
- Used internally by Google (Search, Maps, Gmail)
- Max cell size: **10 MB**, max row: **100 MB**

> Best for: IoT time-series, financial data, ML feature stores, high-frequency analytics

---

### ⚖️ Storage Option Comparison

| Service | Type | Best For | Capacity |
|---|---|---|---|
| **Cloud Storage** | Object | Immutable blobs > 10 MB (images, video, backups) | Petabytes (5 TB/object max) |
| **Cloud SQL** | Relational SQL | Web frameworks, OLTP, existing SQL apps | Up to 64 TB |
| **Cloud Spanner** | Relational SQL + horizontal scale | Global OLTP, financial systems | Petabytes |
| **Firestore** | NoSQL document | Mobile/web apps, real-time sync, offline support | Terabytes (1 MB/entity) |
| **Bigtable** | NoSQL wide-column | Time-series, analytics, ML feature stores, high R/W | Petabytes (10 MB/cell) |
| **BigQuery** | Analytical data warehouse | SQL analytics, BI dashboards, large-scale queries | Petabytes |

> 💡 **Decision shortcut:**
> - Need SQL + small scale? → **Cloud SQL**
> - Need SQL + global scale? → **Spanner**
> - Need NoSQL + mobile sync? → **Firestore**
> - Need NoSQL + high throughput analytics? → **Bigtable**
> - Need files/blobs? → **Cloud Storage**
> - Need analytics on massive datasets? → **BigQuery**

---

## Module 5 — Containers in the Cloud

### Why Containers?

- Package code + dependencies together → runs identically anywhere
- Lighter than VMs — no full OS per instance
- Scale horizontally: duplicate containers, not whole machines

```
Physical Server
  └── Hypervisor (VM layer)
        ├── VM → OS → App           ← heavy
        └── VM → OS → App

vs.

Physical Server
  └── OS
        └── Container Runtime
              ├── Container → App   ← lightweight
              └── Container → App
```

---

### Kubernetes

Kubernetes (K8s) is an **open-source container orchestration system** for automating deployment, scaling, and management of containerised applications.

#### Core Concepts

| Concept | What it is |
|---|---|
| **Pod** | Smallest deployable unit — one or more containers deployed together |
| **Deployment** | Declares desired state — manages replica count, rolling updates |
| **Service** | Stable IP/DNS endpoint that routes to a set of pods |
| **Node** | A VM that runs pods |
| **Cluster** | A set of nodes managed by a control plane |

#### Control Plane Components

```
Control Plane
  ├── kube-apiserver     ← entry point for all API requests
  ├── etcd               ← cluster state database
  ├── kube-scheduler     ← assigns pods to nodes
  └── kube-controller-manager ← ensures desired state is maintained
```

---

### Google Kubernetes Engine (GKE)

GKE = **Managed Kubernetes on GCP**. Google manages the control plane. You interact with it via the same Kubernetes API.

#### GKE Modes

| Mode | Who manages nodes | Recommended |
|---|---|---|
| **Autopilot** | Google manages node config, scaling, upgrades, security | ✅ Yes — production default |
| **Standard** | You manage node config and optimisation | Only if you need custom node control |

#### GKE Benefits

- Google Cloud load balancing integrated
- Node pools for workload isolation
- Automatic node scaling, upgrades, and repair
- Built-in logging and monitoring via Google Cloud Observability

```bash
# Create a GKE cluster
gcloud container clusters create my-cluster

# Rolling update
kubectl set image deployment/my-app container=image:v2
```

---

## Module 6 — Applications in the Cloud

### Cloud Run

- **Fully managed serverless platform** for stateless containers
- No infrastructure management — Google handles scaling to zero and back
- Charges **only when code is executing** (per request/per 100ms)
- Supports any language, any binary — if it runs in a container, it runs on Cloud Run

#### Cloud Run Workflow

```
1. Write your code
2. Build & package into a container → push to Artifact Registry
3. Deploy to Cloud Run
   ↳ Cloud Run starts container on demand per request
```

> Also supports **source-based deployment** (Buildpacks) — skip the Dockerfile, just `gcloud run deploy`.

**Use when:** HTTP-triggered workloads, APIs, microservices, scheduled jobs — you want serverless without managing K8s.

---

### Cloud Run Functions (formerly Cloud Functions)

- **Event-driven serverless functions** — write a function, trigger it on an event
- No server, no container build required
- Scale from zero automatically

#### Supported Event Triggers

- HTTP requests
- Cloud Storage events (file uploaded, deleted)
- Pub/Sub messages
- Firestore document changes
- Cloud Scheduler (cron)

```
Event → Cloud Run Function → Action
e.g.: File uploaded to GCS → resize image → save thumbnail
```

| | Cloud Run | Cloud Run Functions |
|---|---|---|
| Unit of deployment | Container | Function (code snippet) |
| Trigger | HTTP | HTTP + events |
| Startup | Fast (container) | Very fast (function) |
| Use case | Full services, APIs | Lightweight event handlers |

---

## Module 7 — Prompt Engineering & LLMs (Bonus)

> This module was included in the compiled PDF as an introduction to Vertex AI and generative AI concepts.

### Large Language Models

- **Pre-trained on large text corpora** — learn language patterns, facts, reasoning
- Types: Text-to-text, text-to-image, multimodal
- **Hallucinations:** Model generates plausible-sounding but incorrect facts — validate outputs

### Prompt Engineering Basics

A **prompt** is the text input you give to a model to get a response.

#### Types of Prompts

| Type | Description | Example |
|---|---|---|
| **Zero-shot** | No examples given | "Summarise this article" |
| **One-shot** | One example given | "Here's an example: … Now do this: …" |
| **Few-shot** | Multiple examples | 3–5 examples before the task |
| **Chain-of-thought** | Ask model to reason step-by-step | "Think step by step before answering" |

#### Elements of a Good Prompt

```
[Persona]   + [Task]   + [Context]  + [Format]   + [Constraints]
"You are a  + Summarise + this legal + as bullet  + in plain English,
GCP expert"   document    contract     points       max 5 bullets"
```

#### Best Practices

- Write **detailed and explicit** instructions — don't assume context
- Define **boundaries** — what NOT to include
- Adopt a **persona** to shape tone/depth
- Keep sentences **concise** — avoid ambiguity
- Iterate — prompt engineering is experimental

---

## 🎯 PCA Exam Scenarios to Know

| Scenario | Answer |
|---|---|
| Need HA VM deployment with automatic failover | Deploy across multiple zones in a MIG with health checks |
| Connect on-premises to GCP with dedicated bandwidth | Dedicated Interconnect or Partner Interconnect |
| Store user session data for a mobile app with offline support | Firestore |
| Store IoT sensor time-series data at petabyte scale | Bigtable |
| Run a stateless REST API without managing servers | Cloud Run |
| Need SQL with global consistency and horizontal scale | Cloud Spanner |
| Cheapest option for data archived once a year | Cloud Storage — Archive class |
| Secure a VM's access to Cloud Storage (no user credentials) | Service Account with appropriate IAM role |
| Restrict a junior developer to read-only on one GCS bucket | Predefined role: `storage.objectViewer` on that bucket only |
| Auto-scale a containerised app on GCP | GKE Autopilot or Cloud Run |

---

## 🔗 Key CLI Commands Reference

```bash
# IAM
gcloud projects add-iam-policy-binding PROJECT_ID \
  --member="user:alice@example.com" \
  --role="roles/storage.objectAdmin"

# Compute
gcloud compute instances create my-vm --zone=us-central1-a
gcloud compute instances list

# GKE
gcloud container clusters create my-cluster --zone=us-central1-a
kubectl get pods
kubectl set image deployment/my-app container=image:v2

# Cloud Storage
gcloud storage buckets create gs://my-bucket
gcloud storage cp file.txt gs://my-bucket/

# Cloud Run
gcloud run deploy my-service --image=gcr.io/project/image --region=us-central1
```

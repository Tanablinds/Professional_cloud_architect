---
title: GCP Essential Cloud Infrastructure — Core Services
tags: [gcp, iam, storage, databases, observability, resource-management, pca]
note_type: Service Blueprint
source: Essential Cloud Infrastructure — Core Services (Instructor-led)
exam_domain: Security, Storage, Reliability
---

# 🏗️ GCP Essential Cloud Infrastructure: Core Services

> **Note Type:** Service Blueprint  
> **Why this type?** This module is a deep architectural dive into four foundational GCP service areas: IAM internals, Storage and Database services, Resource Management, and Cloud Observability. It focuses on *how* each service is structured and how components interconnect.

---

## 📦 What This Module Covers

| # | Module | Key Concepts |
|---|--------|-------------|
| 1 | Identity and Access Management (IAM) | Org hierarchy, role types, service accounts, IAP, org policies, IAM conditions |
| 2 | Storage and Database Services | Cloud Storage deep-dive, Filestore, Cloud SQL, Spanner, AlloyDB, Firestore, Bigtable, Memorystore |
| 3 | Resource Management | Resource hierarchy, labels vs tags, quotas, billing/budgets |
| 4 | Resource Monitoring | Cloud Observability, metrics, dashboards, alerting, logging, Error Reporting, Tracing |

---

## Module 1 — Identity and Access Management (IAM)

### IAM Object Model

```
IAM Policy = WHO  +  CAN DO WHAT  +  ON WHICH RESOURCE
              ↓            ↓                  ↓
           Principal     Role             Resource
```

- **Principal (who):** Google Account, Google Group, Service Account, Cloud Identity domain, `allUsers`, `allAuthenticatedUsers`
- **Role (can do what):** A named collection of permissions
- **Resource (which):** Any GCP resource — project, bucket, VM, Pub/Sub topic, dataset

---

### IAM Resource Hierarchy

```
Organization Node  ← root; policies set here flow everywhere
  └── Folders       ← departments, teams, environments
        └── Projects ← trust boundary; billing unit
              └── Resources ← VMs, buckets, topics, datasets…
```

**Inheritance rule:** Policies are **additive downward** — a role granted at a higher level is inherited by all resources below.  
**Deny policies override allow policies** — IAM evaluates deny rules first, always.

---

### IAM Role Types

#### 1. Basic Roles — Avoid in production

Applied across **all resources in a project**. Concentric (Owner ⊇ Editor ⊇ Viewer).

| Role | Capabilities |
|---|---|
| **Viewer** | Read-only |
| **Editor** | Read + modify + deploy |
| **Owner** | Editor + manage IAM + set billing |
| **Billing Admin** | Billing management only — no resource access |

> ⚠️ Basic roles are too broad for multi-team production environments. Use predefined roles instead.

#### 2. Predefined Roles — Recommended default

Service-specific, granular, defined by Google. Applied at project, folder, or org level.

```
compute.instanceAdmin  →  [compute.instances.create, .delete, .start, .stop, ...]
storage.objectViewer   →  [storage.objects.get, storage.objects.list]
```

**Example Compute Engine predefined roles:**

| Role | What it grants |
|---|---|
| `Compute Admin` | Full control of all Compute Engine resources |
| `Network Admin` | Create/modify/delete network resources (excl. firewall, SSL certs) |
| `Storage Admin` | Create/modify/delete disks, images, snapshots |

#### 3. Custom Roles — Use when predefined doesn't fit

- You define the exact set of permissions
- Can be created at project or organization level
- Requires ongoing maintenance when GCP adds new permissions

> 💡 **PCA principle:** Grant the **minimum permissions** needed. Prefer predefined. Use custom only when predefined is too broad *and* you need tighter control.

---

### IAM Conditions

- Add **attribute-based conditions** to role bindings
- Restrict access by: date/time, resource type, resource name, IP address
- Example: grant `storage.objectViewer` only between 9am–5pm on weekdays

---

### Organisation Policies

An **Org Policy** is a **configuration of restrictions** applied at the org, folder, or project level.

- Restricts *what* can be done (not *who*) — enforces compliance guardrails
- Defined using **constraints** (e.g., `iam.disableServiceAccountKeyCreation`)
- Inherited by all child resources
- Only an **Org Policy Admin** can grant exceptions

```
Example constraints:
- Disable service account key creation
- Restrict which regions resources can be deployed in
- Prevent public Cloud Storage buckets
- Restrict allowed external IPs for VMs
```

> **IAM vs Org Policy:**  
> IAM controls *who* can do what on a resource.  
> Org Policy controls *what* can be done, regardless of who.

---

### Service Accounts

A **service account** is an identity for a **non-human actor** (application, VM, pipeline).

#### Types of Service Accounts

| Type | Email format | Notes |
|---|---|---|
| **User-created (custom)** | `name@project.iam.gserviceaccount.com` | Full control, explicit creation |
| **Default Compute Engine** | `{proj-num}-compute@developer.gserviceaccount.com` | Auto-created; auto-granted Editor — disable or scope down |
| **Google APIs SA** | `{proj-num}@cloudservices.gserviceaccount.com` | Runs internal Google processes; auto Editor |

#### Service Account Keys

| Key Type | Managed by | Best practice |
|---|---|---|
| **Google-managed** | GCP rotates automatically | ✅ Always prefer this |
| **User-managed** | You create, store, rotate | ⚠️ High risk if leaked — establish rotation policies |

> 💡 Avoid user-managed keys wherever possible. Use Workload Identity Federation or attached service accounts instead.

#### Access Scopes (for default SA on VMs)

- Scopes are legacy tokens that further restrict what a service account can do on a VM
- For **user-created service accounts**, rely entirely on **IAM roles** — scopes are ignored
- For the **default service account**, scopes act as a secondary guard

---

### IAM Best Practices

| Practice | Why |
|---|---|
| Grant roles to **Google Groups**, not individuals | Update group membership instead of IAM policies |
| Use **principle of least privilege** | Limit blast radius of compromised credentials |
| Audit with **Cloud Audit Logs** (`setIamPolicy`) | Track who changed what access |
| Use **predefined roles** over basic | Prevent accidental over-permission |
| Be careful with `serviceAccountUser` role | Grants access to *all resources* the SA can access |
| Name service accounts clearly + use naming conventions | Easier governance at scale |
| Establish **key rotation policies** and audit with `serviceAccount.keys.list()` | Reduce leaked key exposure |
| Use projects to group resources with the **same trust boundary** | Clean separation of concerns |

---

### Identity-Aware Proxy (IAP)

- Enforces **identity-based access control** at the application layer (not network layer)
- Acts as a central authorisation layer for HTTPS apps on GCP
- Users authenticate via Google Identity; IAP checks IAM policy before proxying the request
- Replaces VPN for internal app access — zero-trust model

```
User → IAP → [identity check + IAM policy check] → App (e.g., App Engine, GCE, GKE)
```

---

### SSO and Directory Sync

- **Google Cloud Directory Sync (GCDS):** One-way sync from Active Directory / LDAP → Cloud Identity
- **SSO:** Users authenticate with existing corporate IdP (SAML 2.0) — no separate Google password needed
- Deprovisioning in Cloud Identity = removes GCP access automatically

---

## Module 2 — Storage and Database Services

### ⚖️ Storage Decision Tree

```
Is your data structured?
├── NO
│   ├── Need shared file system?   → Filestore
│   └── No                        → Cloud Storage
└── YES
    ├── Workload is analytics?
    │   ├── Need low latency + high R/W?  → Bigtable
    │   └── No (batch SQL analytics)      → BigQuery
    └── Not analytics
        ├── Not relational
        │   ├── Need app caching?          → Memorystore
        │   └── No                         → Firestore
        └── Relational
            ├── Need HTAP?                 → AlloyDB
            ├── Need global scalability?   → Spanner
            └── Standard OLTP             → Cloud SQL
```

---

### Cloud Storage (Object Storage)

#### Core Architecture

- Objects are **immutable** — versions replace objects, not modify them
- Files are stored in **globally unique buckets**
- Encrypted at rest and in transit by default (no configuration needed)
- Bucket-level settings apply to all objects within

#### Storage Classes

| Class | Min Duration | Use Case | Access Frequency |
|---|---|---|---|
| **Standard** | None | Hot data, streaming, websites | Frequent |
| **Nearline** | 30 days | Monthly backups, data lake cold tier | ~1×/month |
| **Coldline** | 90 days | DR archives | ~1×/quarter |
| **Archive** | 365 days | Regulatory long-term archiving | ~1×/year |

**Choosing by location type:**
- **Single region** → lowest latency within that region
- **Dual-region** → redundancy + geo-compliance within two regions
- **Multi-region** → highest availability, global access (e.g., `us`, `eu`, `asia`)

#### Autoclass

- Automatically moves objects between storage classes based on access pattern
- Cold → warm if accessed; warm → cold if not accessed
- Removes the need to manually manage lifecycle rules for access-pattern-driven tiering

#### Access Control

| Method | Best for |
|---|---|
| **IAM (uniform bucket-level)** | ✅ Recommended; applies to all objects in bucket |
| **ACLs (object-level)** | Legacy; per-object control; avoid mixing with IAM |
| **Signed URLs** | Time-limited access for unauthenticated users |
| **Signed Policy Documents** | Control what can be uploaded (POST operations) |

**Signed URLs:**
- A cryptographically signed URL tied to a service account
- Time-limited (you set expiry), supports GET/PUT/DELETE
- Any user with the URL can invoke the operation — no Google account needed
- Use when you can't require users to have GCP credentials (e.g., external partners)

```bash
gcloud storage signurl -d 10m path/to/key.p12 gs://bucket/object
```

#### Object Versioning

- When enabled: deletions and overwrites create **non-current versions**, not permanent deletes
- Charges apply for stored versions (treated as separate objects)
- Use **Soft Delete** (bucket-level) as a simpler alternative — retains deleted objects for a configurable period

#### Object Lifecycle Management

Define rules triggered by conditions:

```json
{
  "condition": { "age": 365 },
  "action": { "type": "Delete" }
},
{
  "condition": { "age": 30, "matchesStorageClass": ["STANDARD"] },
  "action": { "type": "SetStorageClass", "storageClass": "NEARLINE" }
}
```

**Object Retention Lock:** Prevent objects from being deleted or modified for a set retention period (WORM compliance).

#### Data Import Options

| Method | Use Case |
|---|---|
| `gcloud storage` / Console | Small-medium uploads |
| **Storage Transfer Service** | Online bulk import from S3, HTTP, another GCS bucket |
| **Transfer Appliance** | Offline — ship physical hardware to Google |

---

### Filestore

- Fully managed **NFS file system** (shared file storage)
- Supports NFS v3 and v4.1
- Lift-and-shift for apps that need a shared POSIX file system
- Use cases: media rendering, EDA, genome sequencing, content management

---

### Cloud SQL

- Fully managed **relational DB**: MySQL, PostgreSQL, SQL Server
- Handles: patching, backups, replication, failover automatically
- Includes: managed network firewall, point-in-time recovery

#### Key Capabilities

| Capability | Detail |
|---|---|
| **Replication** | Read replicas (same/cross-region), HA failover replica |
| **Backup** | Automated daily backups + on-demand |
| **Encryption** | At rest and in transit by default |
| **Capacity** | Up to 64 TB |
| **Connectivity** | Public IP, Private IP (VPC), Cloud SQL Auth Proxy |

#### Connecting to Cloud SQL

```
Option 1: Cloud SQL Auth Proxy     ← Recommended; encrypts, manages certs
Option 2: Private IP (VPC peering) ← Low-latency internal access
Option 3: Public IP + SSL          ← For external clients
```

---

### Cloud Spanner

- Fully managed, **globally distributed, horizontally scalable relational DB**
- ACID transactions + strong consistency across regions
- Combines **SQL + horizontal scale** (no other managed DB does this)

#### Architecture

```
Global Spanner Instance
  └── Replication across zones (Paxos consensus)
        └── Splits data across nodes automatically
              └── No manual sharding needed
```

| Characteristic | Detail |
|---|---|
| **Consistency** | External consistency (strongest available) |
| **Scale** | Petabytes; nodes added without downtime |
| **Availability** | 99.999% SLA for multi-region |
| **SQL** | Full ANSI SQL with joins, indexes |

> Best for: global financial systems, inventory, ad-tech — anywhere needing ACID at massive scale.

---

### AlloyDB

- Fully managed **PostgreSQL-compatible** DB optimised for **HTAP** (hybrid transactional + analytical)
- Google-built database engine on cloud-native multi-node architecture
- **4× faster than standard PostgreSQL** for OLTP; **100× faster** for analytical queries
- Built-in **Vertex AI integration** for ML model calls from SQL
- 99.99% SLA including maintenance

> Choose AlloyDB over Cloud SQL when: you need HTAP, Vertex AI integration, or extreme PostgreSQL performance at scale. Choose Cloud SQL for standard PostgreSQL/MySQL/SQL Server workloads.

---

### Firestore

- Fully managed **serverless NoSQL document database**
- Real-time sync + offline support (mobile/web SDKs)
- ACID transactions, multi-region replication, strong consistency

#### Modes

| Mode | Use Case |
|---|---|
| **Native mode** | New mobile/web apps with real-time sync |
| **Datastore mode** | Migrating existing Datastore server-side apps |

> Best for: mobile apps, real-time collaboration, IoT state management, user profiles.

---

### Bigtable

- Fully managed **wide-column NoSQL** database
- Petabyte-scale; sub-10ms latency at high throughput
- Used by Google Search, Maps, Gmail

#### Internal Architecture

```
Processing (Compute)       separated from       Storage (Colossus)
     ↓                                                ↓
Tablet servers handle requests             Data stored in SSTables
     ↓                                                ↓
Learns access patterns → auto-rebalances tablets across nodes
```

**Throughput scales linearly** — add nodes, get proportional throughput increase.  
**Rebalancing:** Bigtable moves tablets (not data) when nodes are added, so no downtime.

| Characteristic | Detail |
|---|---|
| **Data model** | Rows × columns × timestamp (time-series native) |
| **Max cell size** | 10 MB |
| **Max row size** | 100 MB |
| **Scale** | Petabytes; nodes added without restart |

> Best for: IoT telemetry, financial tick data, ad-tech event logs, ML feature stores.

---

### Memorystore

- Fully managed **in-memory data store** (Redis and Memcached)
- Sub-millisecond latency for read-heavy, session, or caching workloads
- Fully compatible with Redis/Memcached APIs — no code changes needed
- Use for: **application caching, session management, leaderboards, real-time counters**

---

### ⚖️ Storage & Database Comparison

| Service | Type | Best For | Max Scale |
|---|---|---|---|
| **Cloud Storage** | Object | Blobs, files, backups, static assets | Petabytes |
| **Filestore** | File (NFS) | Shared POSIX file system | 100+ TB |
| **Cloud SQL** | Relational | Standard OLTP, web apps, SQL workloads | 64 TB |
| **AlloyDB** | Relational (HTAP) | High-perf PostgreSQL + analytics + AI | Petabytes |
| **Spanner** | Relational + global | Global OLTP, financial systems | Petabytes |
| **Firestore** | NoSQL document | Mobile/web real-time, offline-first | Terabytes |
| **Bigtable** | NoSQL wide-column | High-throughput analytics, time-series | Petabytes |
| **BigQuery** | Analytical warehouse | SQL analytics, BI, large-scale queries | Petabytes |
| **Memorystore** | In-memory cache | App caching, sessions, leaderboards | 300 GB |

---

## Module 3 — Resource Management

### Resource Hierarchy & Billing

```
Organisation → Folders → Projects → Resources

Billing accumulates UPWARD:
  Resource usage → Project billing account → Organisation billing
```

- Each **resource belongs to exactly one project**
- Each **project is tied to one billing account**
- **Organisation node** aggregates all billing accounts

### Labels

Key-value pairs attached to resources (VMs, disks, snapshots, images, buckets, etc.)

```
team:engineering       environment:prod
component:frontend     state:readyfordeletion
owner:alice            cost-center:marketing
```

**Uses:**
- Filter resources in console or API
- Cost allocation and billing export analysis
- Bulk operations in scripts
- Inventory and auditing

> Up to **64 labels** per resource. Labels propagate through billing exports.

### Labels vs Network Tags

| | **Labels** | **Network Tags** |
|---|---|---|
| Applied to | VMs, disks, snapshots, images, buckets, etc. | VM instances only |
| Format | Key-value pairs | Plain strings |
| Primary use | Organisation, cost tracking, filtering | Networking (firewall rules, routing) |
| Billing propagation | ✅ Yes | ❌ No |

### Quotas

- Applied at **project level** — limit resource consumption
- Two types: **rate quotas** (requests/sec) and **allocation quotas** (total count)
- Protects against runaway usage and ensures fair use across customers
- Can be increased via quota request in Cloud Console

### Budgets & Billing Alerts

```
Budget → Alert thresholds (e.g., 50%, 90%, 100%) → Email notifications
                                                 → Pub/Sub → Cloud Run Function (programmatic action)
```

- Budgets do **not** cap spending — they alert only
- Use Pub/Sub trigger to automate cost control actions (e.g., disable billing, spin down resources)
- Visualise spend with **Looker Studio** (formerly Data Studio) connected to **BigQuery billing export**

---

## Module 4 — Resource Monitoring (Cloud Observability)

### Google Cloud Observability Overview

Unified, integrated observability stack — monitoring, logging, error reporting, tracing, profiling in one platform.

```
Cloud Observability
  ├── Monitoring     → metrics, dashboards, alerting, uptime checks
  ├── Logging        → log ingestion, routing, export, analysis
  ├── Error Reporting → aggregated error tracking per service
  ├── Trace          → distributed request tracing, latency analysis
  └── Profiler       → CPU/memory profiling of running applications
```

Works across: **Google Cloud, AWS**, and on-premises (via agents).

---

### Cloud Monitoring

#### Metrics Scope

- **Root entity** that holds all monitoring config for one or more projects
- A metrics scope can monitor **1–375 projects**
- Contains: dashboards, alerting policies, uptime checks, notification channels

```
Metrics Scope (scoping project)
  └── Monitors:
        ├── GCP Project A
        ├── GCP Project B
        └── AWS Account (via connector)
```

#### Data Collection

- **Ops Agent** (recommended): Installed on Compute Engine VMs; collects host metrics, process metrics, and third-party app metrics that the hypervisor can't see (e.g., memory usage inside VM)
- **Cloud Monitoring API**: Push custom metrics from any application
- Supports most major OS: CentOS, Ubuntu, Windows

#### Dashboards & Alerting

| Feature | Detail |
|---|---|
| **Dashboards** | Custom charts visualising any metric; built-in templates available |
| **Alerting policies** | Trigger on threshold, rate of change, absence of metric |
| **Uptime checks** | HTTP/HTTPS/TCP checks from multiple regions; alert if service goes down |
| **Notification channels** | Email, PagerDuty, Slack, Pub/Sub, webhooks |

#### Custom Metrics

Use when built-in metrics don't capture what matters for your app (e.g., active game sessions, queue depth).

```python
from google.cloud import monitoring_v3

client = monitoring_v3.MetricServiceClient()
# Define and write custom time-series metric
```

#### Autoscaling with Custom Metrics

```
Custom metric (e.g., active users)
  → Cloud Monitoring
    → Autoscaler compares avg metric vs utilization target
      → Scale out if above / Scale in if below
```

---

### Cloud Logging

- Fully managed, real-time **log ingestion and analysis**
- Ingests from: GCP services (automatic), VMs (via Ops Agent), AWS, on-prem
- Default retention: 30 days (_Default log bucket); configurable up to 3650 days

#### Log Types

| Type | What it captures |
|---|---|
| **Admin Activity** | Who changed what (always on, non-billable) |
| **Data Access** | Who read/wrote data (off by default; billable) |
| **System Event** | GCP infrastructure operations |
| **Access Transparency** | Google support access to your data |

#### Log Routing and Export

```
Log sink → [filter] → Destination
                        ├── Cloud Storage (long-term archiving)
                        ├── BigQuery (analysis + Looker Studio dashboards)
                        ├── Pub/Sub (real-time streaming to SIEM, Splunk)
                        └── Another log bucket
```

**Reference architecture (Splunk integration):**

```
GCP Logs → Log Router (sink) → Pub/Sub → Dataflow → Splunk
```

---

### Error Reporting

- Automatically **aggregates and groups errors** from running cloud services
- Counts occurrences, shows first/last seen, links to stack trace
- Integrates with: App Engine, Cloud Run, Cloud Functions, Compute Engine (via logging agent)
- Sends alerts when new errors appear

---

### Cloud Trace

- **Distributed latency tracing** across services
- Collects and displays time-to-complete data for requests
- Identifies bottlenecks and slow dependencies in microservice architectures
- Auto-instrumented for App Engine; add SDK for Cloud Run/GKE/custom

```
Request →  Service A (50ms) → Service B (300ms) → DB (200ms)
                              ↑ bottleneck flagged by Trace
```

---

### Cloud Profiler

- Continuous **CPU and memory profiling** of production applications
- Low overhead (<1% CPU) — safe to run in production
- Helps identify which functions consume the most resources

---

## 🎯 PCA Exam Scenarios to Know

| Scenario | Answer |
|---|---|
| Grant a contractor read access to one GCS bucket without a Google account | Signed URL with time-limited expiry |
| Prevent all teams from creating service account keys across the org | Org Policy: `iam.disableServiceAccountKeyCreation` |
| Application needs to write to BigQuery — no user involved | Custom service account + `bigquery.dataEditor` predefined role |
| Need global relational DB with ACID + horizontal scale | Cloud Spanner |
| Need PostgreSQL with ML integration and HTAP | AlloyDB |
| Store IoT time-series data at petabyte scale, sub-10ms reads | Bigtable |
| Mobile app needs real-time sync + offline support | Firestore |
| Organise resources for cost allocation across teams | Labels (`team:X`, `cost-center:Y`) |
| Restrict firewall rules to specific VMs | Network tags (not labels) |
| Application logs are going to Splunk for SIEM analysis | Log Router sink → Pub/Sub → Dataflow → Splunk |
| Alert ops team when VM CPU exceeds 80% for 5 minutes | Cloud Monitoring alerting policy with notification channel |
| Debug a slow microservice call chain | Cloud Trace |
| VM memory metrics not appearing in Cloud Monitoring | Install Ops Agent on the VM |
| Grant IAM access to a whole team without editing IAM per person | Create a Google Group, grant role to the group |

---

## 🔗 Key CLI Reference

```bash
# IAM
gcloud projects add-iam-policy-binding PROJECT \
  --member="serviceAccount:sa@project.iam.gserviceaccount.com" \
  --role="roles/storage.objectAdmin"

# Service Account
gcloud iam service-accounts create my-sa --display-name="My Service Account"
gcloud iam service-accounts keys list --iam-account=my-sa@project.iam.gserviceaccount.com

# Cloud Storage
gcloud storage signurl -d 10m key.p12 gs://bucket/object
gcloud storage buckets update gs://bucket --lifecycle-file=lifecycle.json

# Labels
gcloud compute instances add-labels my-vm --labels=env=prod,team=backend

# Monitoring
gcloud monitoring dashboards create --config-from-file=dashboard.json
gcloud logging sinks create my-sink bigquery.googleapis.com/projects/PROJECT/datasets/DS \
  --log-filter='resource.type="gce_instance"'
```

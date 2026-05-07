---
title: Reliable Cloud Infrastructure — Design and Process
tags: [GCP, PCA, Architecture, Microservices, Storage, Networking, Reliability, Security, DevOps, SLO]
modules: "Defining Services | Microservices | DevOps Automation | Storage | Networking | Deploying | Reliability | Security | Maintenance"
note_type: "🏗️ Service Blueprint"
---

# 🏗️ Reliable Cloud Infrastructure — Design and Process

> **GCP Professional Cloud Architect Notes**
> Source: *Reliable Cloud Infrastructure: Design and Process — 9 Modules (313 pages)*

---

## Module 1: Defining Services

### Requirements Framework

**Qualitative requirements** — define systems from the user's point of view using: **Who / What / Why / When / How**

#### Roles → Personas → User Stories

```
Role (goal of a user at a point in time)
 └── Persona (imaginary representation of a role — gives human context)
       └── User Story (one thing the user wants the system to do)
```

- **Roles** describe objectives, not people or job titles. Everyone is a user — "user" is not a good role.
- **Personas** tell a story; they help architects think about edge cases and UX details.
- **User story template:** `As a [role], I want to [action], so that I can [benefit].`

**INVEST criteria for good user stories:**

| Letter | Criterion | Why It Matters |
|---|---|---|
| **I** | Independent | Prevents prioritization conflicts |
| **N** | Negotiable | Promotes collaboration, not a contract |
| **V** | Valuable | Focus on outcomes, not deliverables |
| **E** | Estimatable | If not, story is too large or missing details |
| **S** | Small | Keeps scope tight; enables fast feedback |
| **T** | Testable | Developers can verify it's done |

---

### KPIs and SMART Criteria

**KPIs measure whether you're on track toward a goal** — they are not the goal itself.

| Type | Examples |
|---|---|
| Business KPIs | ROI, EBIT, employee turnover, customer churn |
| Technical KPIs | Page views, user registrations, clickthroughs, checkouts |

**SMART criteria for KPIs and SLOs:**

| Letter | Meaning | Anti-example → Better |
|---|---|---|
| **S** | Specific | "User friendly" → "Section 508 accessible" |
| **M** | Measurable | "Fast" → "HTTP GET < 400ms aggregated per minute" |
| **A** | Achievable | "100% availability" → "99.9% availability" |
| **R** | Relevant | Does it matter to the user? |
| **T** | Time-bound | "99% available" — per year? per month? |

---

### SLI → SLO → SLA Hierarchy

```
SLI (what you measure — a KPI)
 └── SLO (target: e.g. 99.9% availability — stricter than SLA)
       └── SLA (customer contract — lower threshold, penalties if breached)
```

| Term | Description | Example |
|---|---|---|
| **SLI** | Quantifiable, time-bound measurement | `% successful HTTP requests per minute` |
| **SLO** | SLI + target; must be SMART | `99% of HTTP GETs complete in < 100ms` |
| **SLA** | Business contract with penalties | `Error rate < 0.3%`; compensation if breached |

**SLO tips:**
- Goal is to keep users happy, **not** to maximize the SLO number
- Higher SLO → higher compute redundancy + operations cost
- Don't significantly outperform SLOs — users come to expect it
- SLO threshold must always be **stricter** than the SLA
- Not every service needs an SLA, but **every service needs an SLO**

**Latency SLO example (multiple percentiles):**
```
90% of HTTP GET calls complete in < 50ms
99% of HTTP GET calls complete in < 100ms
99.9% of HTTP GET calls complete in < 500ms
```

**Common pitfall:** Inconsistent SLI measurement (e.g., SLO based on average latency but SLA on p99) can make the SRE team think the SLO is met while the SLA is being violated. Always align metrics, measurement windows, and scope across SLI/SLO/SLA.

---

## Module 2: Microservice Design and Architecture

### Monolith vs. Microservices

| | Monolithic | Microservices |
|---|---|---|
| **Codebase** | Single codebase, single database | Multiple codebases, each manages own data |
| **Deployment** | Deploy everything together | Deploy services independently |
| **Scaling** | Scale entire app | Scale services independently |
| **Failure** | One failure can affect everything | Isolated failure domains |
| **Complexity** | Low initially | Higher — distributed system challenges |

**Pros of microservices:** Easier to develop/maintain, reduced deployment risk, independent scaling, faster innovation, mix languages/frameworks, fine-grained cost accounting.

**Cons:** Inter-service communication complexity, increased latency at boundaries, securing inter-service traffic, multiple deployments, must maintain backward compatibility.

---

### Decomposing Applications

Three strategies for identifying microservice boundaries:

| Strategy | Example |
|---|---|
| **By feature** | Reviews service, Orders service, Products service |
| **By architectural layer** | Web UI, Data access services, Android/iOS |
| **By shared functionality** | Auth service, Reporting service |

---

### Stateful vs. Stateless Services

| | Stateless | Stateful |
|---|---|---|
| **Scaling** | Easy — add instances | Hard — need sticky sessions |
| **Upgrades** | Easy | Harder |
| **Backup** | Not needed | Required |

**Best practice:** Store state in backend storage services shared by stateless frontends:
- Persistent state: Firestore, Cloud SQL
- Caching: Memorystore for Redis

**Architecture pattern:**
```
Internet → Cloud DNS → Application LB (global)
           → Stateless Frontend Servers
           → Internal LB
           → Stateless Backend Servers
           → Stateful Data Layer (isolated)
```

**Avoid:** Storing shared state in-memory on servers (requires sticky sessions / session affinity; blocks autoscaling).

---

### The 12-Factor App

| # | Factor | Key Principle |
|---|---|---|
| 1 | **Codebase** | One repo per service, tracked in version control (Git) |
| 2 | **Dependencies** | Explicitly declare + isolate (Maven, Pip, NPM); package in container |
| 3 | **Config** | Store secrets/endpoints in environment variables, not source code |
| 4 | **Backing services** | Databases, caches, queues accessed via URLs; easy to swap |
| 5 | **Build, release, run** | Strictly separate: Build → Release (build + config) → Run |
| 6 | **Processes** | Stateless processes; state stored in backing service |
| 7 | **Port binding** | App is self-contained and exports a port + protocol |
| 8 | **Concurrency** | Scale out via the process model (add instances) |
| 9 | **Disposability** | Fast startup, graceful shutdown; fault-tolerant |
| 10 | **Dev/prod parity** | Keep dev, staging, prod as similar as possible (Docker + IaC) |
| 11 | **Logs** | Treat logs as event streams; write to stdout; aggregate centrally |
| 12 | **Admin processes** | One-off admin tasks should be automated, decoupled from app |

---

### REST Architecture

**REST** (Representational State Transfer) — protocol-independent (HTTP most common, gRPC also used).

| HTTP Verb | Purpose | Idempotent? |
|---|---|---|
| `GET` | Retrieve data | ✅ Yes |
| `POST` | Create new resource (server generates ID) | ❌ No |
| `PUT` | Create or update resource (client specifies ID) | ✅ Yes |
| `DELETE` | Remove resource | ✅ Yes |

**HTTP Response codes:**
- `2xx` = Success (200 OK, 201 Created)
- `4xx` = Client error (400 Bad Request, 403 Forbidden, 404 Not Found)
- `5xx` = Server error

**URI design rules:**
- Plural nouns for collections: `GET /pets`
- Singular nouns for items: `GET /pet/1`
- Don't use verbs: ❌ `GET /getPets` → ✅ `GET /pets`
- Include version: `GET /v1/pets`
- URIs are case-insensitive

---

### API Management Tools

| Tool | Best For |
|---|---|
| **Cloud Endpoints** | GCP backends; lightweight API gateway; OpenAPI + gRPC |
| **Apigee** | Enterprise API management; on-prem/cloud/hybrid; monetization; analytics |
| **API Gateway** | Secure, consistent REST API across multiple backends |

**OpenAPI** — industry standard for REST API specification (based on Swagger 2.0); language-independent; auto-generates client libraries and documentation.

**gRPC** — binary protocol by Google; fast; HTTP/2; supports streaming; ideal for internal service-to-service communication; supported by Application LB + Cloud Endpoints + GKE (via Envoy proxy).

---

## Module 3: DevOps Automation

### CI/CD Pipeline Components

```
Code commit → Unit tests → Build Docker image → Push to Artifact Registry → Deploy
```

| GCP Service | Role |
|---|---|
| **Cloud Source Repositories** | Managed private Git repos; IAM-integrated; connects to GitHub/Bitbucket |
| **Cloud Build** | Managed build service; steps defined in `cloudbuild.yaml` or Dockerfile; no infra to manage |
| **Build Triggers** | Watches repo; auto-starts build on branch/tag push; supports regex matching |
| **Artifact Registry** | Universal package manager; Docker, Maven, npm, PyPI; IAM-integrated |
| **Binary Authorization** | Enforce only trusted/attested images are deployed to GKE; uses Kritis spec |

**Cloud Build key points:**
- Builders = Docker containers with language tools installed
- Steps can run unit tests, linting, integration tests, image scanning
- `gcloud builds submit --tag gcr.io/PROJECT/image`

**Binary Authorization workflow:**
1. Enable Binary Authorization on GKE cluster
2. Add policy requiring signed images
3. Cloud Build uses an "attestor" to verify image source
4. Artifact Registry vulnerability scanner scans containers
5. Kritis Signer creates attestation
6. Unsigned images are blocked at deploy time

---

### IaC Mindset: Cloud vs. On-Premises

| On-Premises | Cloud |
|---|---|
| Buy machines | Rent machines |
| Keep machines running for years | Turn off ASAP |
| Fewer big machines (scale up) | Many small machines (scale out) |
| Capital expenditure | Monthly operating expense |

**In the cloud: treat all infrastructure as disposable.** Never fix broken machines — delete and recreate. Use IaC to automate everything.

**IaC tools:** Terraform (primary focus), Ansible, Chef, Puppet, Packer.

**Terraform benefits:** Repeatable, declarative, parallel deployment, template-driven, multi-cloud.

---

## Module 4: Choosing Storage and Data Solutions

### Storage Portfolio Overview

| Service | Type | Schema | Scales | Best For |
|---|---|---|---|---|
| **Cloud SQL** | Relational | Fixed | Vertically (64 TB) | MySQL, PostgreSQL, SQL Server; web frameworks, CMS, eCommerce |
| **Spanner** | Relational | Fixed | Horizontally (infinite) | Global scale, strong consistency, ACID, financial |
| **AlloyDB** | Relational | Fixed | Regional/multi-regional | RDBMS + scale, HTAP (transactional + analytical) |
| **Firestore** | NoSQL document | Schemaless | Auto, infinite | Hierarchical data, mobile/web, user profiles, game state |
| **Bigtable** | NoSQL wide-column | Schemaless | Horizontally | Heavy read/write, IoT, AdTech, financial streams |
| **Cloud Storage** | Object | Schemaless | Auto, infinite | Binary data, images, media, backups |
| **Persistent Disk** | Block | Schemaless | Fixed size | VM block storage, high-performance disk |
| **BigQuery** | Data warehouse | Fixed | Auto, infinite | SQL analytics, BI dashboards, reporting |
| **Memorystore** | In-memory | Schemaless | Vertically | Caching, session state, game state (Redis) |
| **Filestore** | File (NAS) | Fixed | Configurable | Shared file system, ML, Generative AI workloads |

---

### Key Storage Selection Criteria

#### Availability SLAs

| Service + Config | Availability SLA |
|---|---|
| Cloud Storage multi-region | ≥99.95% |
| Cloud Storage regional | 99.9% |
| Cloud Storage coldline | 99.0% |
| Spanner multi-region | 99.999% |
| Spanner single-region | 99.99% |
| Firestore multi-region | 99.999% |
| Firestore single-region | 99.99% |

#### Consistency Model

| Strongly Consistent | Eventually Consistent (default) |
|---|---|
| Cloud Storage, Cloud SQL, Spanner, Firestore | Bigtable (with replication), Memorystore replicas |
| Bigtable (no replication) | — |

#### Scaling Model

| Horizontal (add nodes) | Vertical (bigger machines) | Automatic / Unlimited |
|---|---|---|
| Bigtable, Spanner, Firestore | Cloud SQL, Memorystore | Cloud Storage, BigQuery |

#### Durability (Shared Responsibility)

| Service | Google Provides | Your Responsibility |
|---|---|---|
| Cloud Storage | 11 9's durability | Enable versioning; lifecycle policy |
| Persistent Disk | Snapshots | Schedule snapshot jobs |
| Cloud SQL | Automated backups, PITR | Create backups; failover replica |
| Spanner | Auto replication | Run export jobs |
| Firestore | Auto replication | Run export jobs |

---

### Storage Decision Chart

```
Start → Is data structured?
  NO  → Need shared file system? → YES → Filestore
                                   NO  → Cloud Storage (object)
  YES → Workload involves analytics?
    YES → Need extensive updates/low latency? → YES → Bigtable
                                                NO  → BigQuery (data warehouse)
    NO  → Data relational?
      YES → Need HTAP? → YES → AlloyDB
                         NO  → Need global scale? → YES → Spanner
                                                    NO  → Cloud SQL
      NO  → Need application caching? → YES → Memorystore
                                         NO  → Firestore (document)
```

---

### Data Transfer Options

| Method | Best For | Key Details |
|---|---|---|
| **Cloud Storage Transfer Service** | Online transfers, scheduled uploads | From S3, HTTP/HTTPS, or GCS→GCS; one-time or recurring |
| **Storage Transfer Service (on-prem)** | Large-scale on-prem uploads | Docker agent; 300 Mbps minimum; scales to 100s TB |
| **Transfer Appliance** | Massive datasets too large to upload | Physical rackable device; up to 1 PB; customer holds encryption key |
| **BigQuery Data Transfer Service** | SaaS → BigQuery | Google Ads, YouTube, Teradata, Amazon Redshift, S3 |

---

## Module 5: Network Architecture

### VPC Design Principles

- **VPCs are global** — subnets are regional
- Resources across regions communicate via internal IPs (no added interconnect)
- Custom subnets: specify region + CIDR range; non-overlapping; expandable without downtime
- One VM can have up to **8 network interfaces**, each in a different VPC
- **Shared VPC:** Host project owns VPC; service projects use it. Centralizes network admin; developers focus on workloads.

---

### Load Balancer Selection

| Balancer Type | Layer | Traffic | Scope | Use Case |
|---|---|---|---|---|
| **Global External Application LB** | L7 | HTTP(S) | Global | Public-facing; multi-region; SSL termination; content routing |
| **Regional External Application LB** | L7 | HTTP(S) | Regional | Regional public traffic |
| **Regional Internal Application LB** | L7 | HTTP(S) | Regional | Internal HTTP microservices |
| **External Proxy Network LB** | L4 | TCP/SSL | Global/Regional | TLS offload; multi-region backends |
| **Internal Proxy Network LB** | L4 | TCP | Regional | Internal TCP services |
| **External Passthrough Network LB** | L4 | TCP/UDP | Regional | Preserve client IP; low latency |
| **Internal Passthrough Network LB** | L4 | TCP/UDP | Regional | Internal high-throughput services |

**Quick rule:** HTTP(S) public → Application LB; TLS offload/TCP proxy → Proxy Network LB; Preserve client IP → Passthrough LB.

**Cloud CDN:** Enable on Global External Application LB; caches static content at edge; reduces latency + egress costs; protects backend from DDoS (cache hits absorb attacks).

---

### Network Connectivity Options

| Option | When to Use | SLA | Bandwidth |
|---|---|---|---|
| **VPC Peering** | Connect two GCP VPCs (same or diff org); private RFC 1918 | — | Up to VPC limits |
| **Classic VPN** | Low-volume encrypted on-prem connection; static or dynamic routes | 99.9% | Up to ~3 Gbps; MTU ≤ 1460 |
| **HA VPN** | High-availability encrypted connection; requires BGP/Cloud Router | 99.99% | 2+ tunnels; 2 external IPs |
| **Dedicated Interconnect** | High-speed direct physical connection; colocation required | 99.99% | 10–200 Gbps |
| **Partner Interconnect** | No colocation; through service provider | 99.99% | 50 Mbps–10 Gbps |

**Cloud Router:** Enables dynamic route exchange via BGP; required for HA VPN and dynamic routing with Cloud Interconnect. Routes update without changing tunnel configuration.

**HA VPN topologies:**
1. HA VPN → peer VPN devices (2 tunnels for 99.99%)
2. HA VPN → AWS (requires Transit Gateway or VGW; 4 external IPs)
3. Two HA VPN gateways connected (GCP-to-GCP cross-VPC with 99.99%)

---

## Module 6: Deploying Applications to Google Cloud

### Compute Platform Selection

```
Start
↓
Need OS-level control, custom images, or lift-and-shift? → Compute Engine
↓
Need container orchestration with persistent workloads?  → GKE
↓
Need stateless containers, event-driven, scale to zero?  → Cloud Run
↓
Need event-triggered, single-purpose functions?          → Cloud Run Functions
↓
Need managed platform, focus only on code?               → App Engine
```

| Platform | Key Characteristics |
|---|---|
| **Compute Engine** | Full OS control; MIGs for HA + autoscaling; health checks + autohealing |
| **GKE** | Container orchestration; Autopilot (managed) or Standard; multi-zone clusters |
| **Cloud Run** | Stateless containers; HTTP/Pub/Sub triggers; scale to zero; no infra |
| **Cloud Run Functions** | Event-driven; single-purpose; loosely coupled microservices |
| **App Engine** | Managed PaaS; automatic rolling updates + traffic splitting built-in |

---

## Module 7: Designing Reliable Systems

### Key Performance Metrics

| Metric | Definition | How to Achieve |
|---|---|---|
| **Availability** | % time system is running + processing requests | Fault tolerance, health checks, multi-zone |
| **Durability** | Odds of NOT losing data due to hardware/software failure | Replication, regular backups, restore drills |
| **Scalability** | Ability to handle growing load + data | Monitor usage; auto-scaling; capacity planning |

---

### Failure Modes and Mitigations

#### 1. Single Points of Failure → N+2 Redundancy
- Deploy **N+2** units: one for maintenance/upgrade + one for unexpected failure
- Deploy across **multiple zones** (different failure domains)
- Make units interchangeable stateless clones

#### 2. Correlated Failures → Distribute Across Failure Domains
- Failure domain = group of related items that fail together
- Causes: shared hardware (rack switch), shared region/zone, same software bug, shared config system
- Mitigations:
  - Divide business logic by failure domain
  - Deploy to multiple zones and/or regions
  - Split responsibilities across independent processes
  - Loosely coupled services (failure in one ≠ failure in collaborating service)

#### 3. Cascading Failures → Health Checks + Autoscaling
- Cause: one server fails → remaining servers overloaded → they fail too
- Mitigations:
  - Health checks (Compute Engine) or readiness/liveness probes (GKE)
  - Ensure new instances start fast and don't depend on other backends to start
  - Adequate spare capacity (N+2 rule)

#### 4. Query of Death → Monitoring + Developer Feedback
- Business logic error manifests as resource overconsumption → service overload
- Mitigation: Monitor latency, resource utilization, and error rates; route alerts to developers

#### 5. Positive Feedback Cycle (Retry Storm) → Exponential Backoff + Circuit Breaker

**Truncated Exponential Backoff:**
```
Request fails → wait 1s + random_ms → retry
Request fails → wait 2s + random_ms → retry
Request fails → wait 4s + random_ms → retry
... up to max_backoff and max_retries
```

**Circuit Breaker:**
- Proxy monitors service health
- If unhealthy → stop forwarding requests (protect service from retry storm)
- When healthy → resume forwarding gradually
- GKE: use **Istio** service mesh (automatic circuit breakers)

#### 6. Accidental Deletion → Lazy Deletion

```
User deletes → [Soft delete / Trash] (≤ 30 days, user can restore)
            → [Soft delete backend] (≤ 60 days, support can restore)
            → [Hard delete / Purge] (only backup recovery remains)
```

---

### High Availability by Service

| Service | HA Strategy |
|---|---|
| **Compute Engine** | Regional managed instance group (distributes across zones) + auto-healing health check |
| **GKE** | Regional cluster (replicates control plane + node pools across zones) |
| **Cloud SQL** | Failover replica in another zone (same region); doubles cost; auto-failover |
| **Spanner** | Multi-region deployment (99.999% SLA; < 6 min/year downtime) |
| **Firestore** | Multi-region deployment (99.999% SLA) |
| **Cloud Storage** | Multi-region bucket (99.95% SLA vs 99.9% for single-region) |

**Cost vs. Availability trade-off table:**

| Deployment | Availability | Cost |
|---|---|---|
| Single zone | Lowest | Lowest |
| Multiple zones in a region | Medium | Medium |
| Multiple regions | Highest | Highest |

Always compare: **cost of deployment** vs. **cost of being down**.

---

### Disaster Recovery Planning

**Two key metrics:**

| Metric | Definition |
|---|---|
| **RPO** (Recovery Point Objective) | Maximum acceptable data loss (e.g., 0 = no data loss, 24h = daily backup is ok) |
| **RTO** (Recovery Time Objective) | Maximum acceptable downtime (e.g., 1 minute = near-instant failover required) |

**DR Strategies:**

| Strategy | Mechanism | Cost | RTO |
|---|---|---|---|
| **Cold standby** | Snapshots + machine images in multi-region storage; spin up when needed | Low | Minutes to hours |
| **Hot standby** | Instance groups in multiple regions; global load balancer active; multi-region databases | High | Near-zero |

**DR Planning process:**
1. **Brainstorm** scenarios (hardware failure, accidental deletion, region outage, software bug)
2. **Define** RPO + RTO per scenario
3. **Prioritize** by impact and likelihood
4. **Document** backup strategy + recovery procedure
5. **Test with drills** — at least annually; ideally part of daily operations

---

## Module 8: Security

### Security Best Practices

| Principle | Implementation |
|---|---|
| **Principle of least privilege** | Grant minimal permissions via IAM roles; applies to users AND service accounts |
| **Separation of duties** | No single person can change/delete/steal data undetected; use multiple projects + folders |
| **Defense in depth** | Security at every layer: hardware → boot → OS → IPC → storage → app → deployment → operations |
| **Regular audits** | Review admin logs, data access logs, VPC flow logs, firewall logs, system logs |
| **Shared responsibility** | Google secures infrastructure layers; you secure your application + config + IAM |

---

### Securing People

**IAM Best Practices:**
- Grant roles to **Groups**, not individuals — simpler management, consistent permissions
- Use **predefined roles** over custom roles (Google designed them correctly)
- Grant roles at the **smallest scope needed** (project > folder > org)
- **Limit Owner/Editor** roles — prefer more specific roles
- Consider **policy inheritance** when assigning roles (org → folder → project → resource)

**Key authentication services:**

| Service | Purpose |
|---|---|
| **IAM** | Authenticate + authorize GCP users and resources |
| **Identity-Aware Proxy (IAP)** | App-level access control without VPN; works with App Engine, GKE, Compute Engine behind ALB |
| **Identity Platform** | Customer-facing auth (CIAM); supports SAML, OpenID, OAuth, email/password, social, phone |
| **Cloud Identity** | Manage users and groups; GCDS for on-prem sync |

---

### Securing Machine Access (Service Accounts)

- Service account = identity for a VM, container, or application (not a human)
- Create service account → assign IAM roles → attach to VM/GKE node pool
- Two key types of service account keys:

| Type | Managed By | Storage | Rotation |
|---|---|---|---|
| **Google-managed** | Google | Cloud; never exposed | Automatic (≤ 2 weeks) |
| **User-managed** | You | JSON file you download | Manual; up to 10 per SA |

> ⚠️ User-managed keys are **extremely powerful credentials** — security risk if mismanaged. Apply `constraints/iam.disableServiceAccountKeyCreation` org policy where possible.

```bash
# Use service account key in CLI
gcloud auth activate-service-account --key-file=[PATH TO KEY FILE]
```

---

### Network Security

| Control | Purpose |
|---|---|
| **Remove external IPs** | Prevent direct internet access to VMs; use bastion host or IAP for admin |
| **Private Google Access** | VMs with only internal IPs can still reach GCP services (Cloud Storage, etc.) |
| **Firewall rules** | Default: deny all ingress; explicitly allow required traffic by port/protocol/source |
| **Cloud Endpoints** | API management; JWT validation; rate limiting; integrates with Identity Platform |
| **TLS only** | Use HTTPS frontends on all load balancers; never HTTP |
| **Cloud CDN** | L3/L4 DDoS protection via cache hits absorbing attack traffic |
| **Cloud Armor** | L7 WAF; allow/deny by IP, geo, headers, cookies; SQL injection/XSS protection |

**Cloud Armor WAF rule examples:**
```
inIpRange(origin.ip, '9.9.9.0/24')
request.headers['cookie'].contains('80=BLAH')
origin.region_code == 'AU'
evaluatePreconfiguredExpr('xss-canary')
```

---

### Encryption

| Method | Key Management | Use Case |
|---|---|---|
| **Google-managed (default)** | Google manages; AES-256; auto-rotation | Default for all GCP data at rest |
| **CMEK (Customer-Managed)** | Keys in Cloud KMS; you set rotation frequency | Compliance requirements |
| **CSEK (Customer-Supplied)** | Keys on-prem; supplied per API call; only in RAM at Google | Maximum key control; Compute Engine + Cloud Storage |

**Encryption stack:**
```
Data → encrypted with DEK (AES-256)
DEK  → encrypted with KEK (stored in Cloud KMS)
KEK  → periodically auto-rotated
```

**Data Loss Prevention (DLP) API:**
- Scans Cloud Storage, BigQuery, Firestore + images
- Detects: emails, credit cards, SSNs, phone numbers, passport numbers, etc.
- Actions: identify, mask, tokenize, delete

---

## Module 9: Maintenance and Monitoring

### Deployment Strategies

| Strategy | How It Works | Best When | GCP Implementation |
|---|---|---|---|
| **Rolling update** | Update instances one at a time; old + new versions run simultaneously | API is backward compatible | MIG: change instance template; GKE: change Docker image; App Engine: automatic |
| **Blue/Green** | Create full new environment (green); test; switch all traffic at once | Can't have 2 versions running simultaneously | Compute Engine: DNS migration; GKE: service label change; App Engine: Traffic Splitting |
| **Canary** | Route small % of traffic to new version; monitor; increase gradually | Reduce risk before full rollout | Compute Engine: add new MIG as additional backend; GKE: new pod with same labels; App Engine: Traffic Splitting |

**Version management rules:**
- Include version in URI (`/v1/`, `/v2/`)
- Change version when making breaking changes
- Deploy new versions with **zero downtime**
- Never remove items from API responses (only add)

---

### Cost Optimization

**Compute cost levers:**
- Start with small VMs + autoscaling (test first, then scale)
- **Committed use discounts** for predictable workloads
- **Spot VMs** for fault-tolerant workloads: 60-91% discount (preemptible)
- Use **GCP rightsizing recommendations** to eliminate waste
- **GKE usage metering**: compares requested vs consumed resources per namespace

**Storage cost levers:**
- Don't over-allocate disk; match I/O pattern to disk type (Standard PD is 4x cheaper than SSD PD)
- Choose storage service wisely: 1 GB in Firestore ≈ free; 1 GB in Bigtable ≈ $500/month
- Use **CDN + caching** (Memorystore) to reduce backend compute + storage needs
- Use **Pub/Sub** instead of a datastore to decouple services

**Network cost:** Egress within same zone = free; between zones = small cost; between regions/intercontinental = highest. Keep machines close to data.

**Cost management tools:**

| Tool | Purpose |
|---|---|
| **Pricing Calculator** | Estimate costs before deploying |
| **Cloud Billing Reports** | Monitor current spend; breakdown by project/product/region |
| **BigQuery export** | Advanced cost analysis; identify large expenses; SQL queries |
| **Looker Studio** | Visual billing dashboards (daily + monthly views) |
| **Budget alerts** | Email + Pub/Sub → Cloud Run function for automated cost management |

**Capacity planning** is a continuous cycle: Forecast → Allocate → Approve → Deploy → Monitor → repeat.

---

## 🎯 Scenario Breakdowns

### Scenario: Designing for High Availability (Multi-tier App)

```
Architecture (US users, two regions):
  Global External Application LB
    ├── us-central1 (primary)
    │   ├── Managed Instance Group (web tier, zone a + b)
    │   ├── Cloud SQL (primary) + failover replica (zone b)
    │   └── Firestore (multi-region, 99.999%)
    └── us-east1 (backup)
        └── MIG (standby, cold or hot depending on RTO)

Cloud Storage (multi-region) for backups
```

### Scenario: Disaster Recovery Decision Matrix

| Service | RPO | RTO | Strategy | Location |
|---|---|---|---|---|
| Orders DB (Cloud SQL) | 0 (no data loss) | 2 min | Failover replica + binary logging | Multi-zone |
| Inventory (Firestore) | 1 hr | 1 hr | Daily automated backups | Multi-region Cloud Storage |
| Analytics (BigQuery) | 0 min | 24 hr | Re-import data (no specific backup) | — |

### Scenario: Choosing Load Balancers for a Travel Portal

| Service | Internet-facing? | Protocol | Load Balancer |
|---|---|---|---|
| Search | ✅ Yes | HTTP | Global External Application LB |
| Web UI | ✅ Yes | HTTP | Global External Application LB |
| Inventory | ❌ Internal | TCP | Regional Internal Passthrough LB |
| Orders | ❌ Internal | TCP | Regional Internal Passthrough LB |
| Analytics | ✅ Yes | HTTP | Regional External Application LB |

### Scenario: Security Architecture

```
Internet → Cloud Armor (WAF, deny lists) → Global External Application LB
         → Custom VPC (firewall: allow HTTPS 0.0.0.0/0, allow SSH from known IPs)
            ├── us-central1 subnet (primary)
            ├── us-east1 subnet (backup)
            └── europe-west2 subnet (EU users)
         → Private Google Access enabled (VMs reach GCP services via internal IP)
         → Cloud VPN → On-prem LAN (internal-only services)
         → GCP Managed Services (Firestore, Cloud SQL, BigQuery) via Private Access
```

---

## 🔑 Key Exam Tips

- **SLO must always be stricter than SLA** — provides buffer for proactive action
- **Not every service needs an SLA, but every service needs an SLO**
- **12-Factor factor 3 (Config):** Store secrets in env vars, never in source code
- **12-Factor factor 11 (Logs):** Write to stdout; aggregate to central sink
- **N+2 rule for single points of failure** — one for maintenance, one for unexpected failure
- **Correlated failures** = failure domain; mitigate by spreading across zones/regions
- **Cascading failure mitigation** = health checks + autohealing + capacity headroom
- **Circuit breaker** (GKE: Istio) protects overloaded services from retry storms
- **Lazy deletion** = trash → soft delete → hard delete; protects against accidental deletion
- **RPO** = data loss tolerance; **RTO** = downtime tolerance
- **Cold standby** = snapshots; restore later. **Hot standby** = multi-region active deployment
- **Spanner + Firestore multi-region** = 99.999% ("five nines"); < 6 min downtime/year
- **Rolling update** = 2 versions run simultaneously (OK for backward-compatible APIs)
- **Blue/green** = full environment swap; no simultaneous versions; easy rollback
- **Canary** = small % traffic to new version; validate before full rollout
- **Shared VPC** = centralized network control; developers focus on workloads, not networking
- **VPC Peering** = GCP-to-GCP; subnets cannot overlap
- **HA VPN** = 99.99%; requires BGP/Cloud Router; 2 external IPs; dynamic routing
- **Dedicated Interconnect** = direct physical 10–200 Gbps; requires colocation facility
- **Partner Interconnect** = via service provider; 50 Mbps–10 Gbps; no colocation needed
- **CSEK** = you supply keys per API call; only in RAM at Google; Compute Engine + Cloud Storage
- **CMEK** = keys in Cloud KMS; you control rotation; broader service support
- **DLP API** = scan + redact PII (credit cards, SSNs, emails) from Cloud Storage/BQ/Firestore
- **Binary Authorization** = enforce only attested images deploy to GKE; Kritis spec
- **Cloud Armor** = L7 WAF; geo-blocking; SQL injection/XSS protection; allow/deny lists

---

*Notes generated from: Reliable Cloud Infrastructure: Design and Process — Defining Services, Microservice Design & Architecture, DevOps Automation, Choosing Storage, Network Architecture, Deploying Applications, Designing Reliable Systems, Security, Maintenance & Monitoring*

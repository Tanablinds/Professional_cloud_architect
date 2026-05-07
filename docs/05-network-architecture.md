---
title: GCP Networking in Google Cloud — Network Architecture & Topologies
tags: [gcp, network-architecture, topology, hub-and-spoke, mesh, mirrored, gated, ncc, pca]
note_type: Architecture Decision Record
source: Networking in Google Cloud — Introduction to Network Architecture + Network Topologies
exam_domain: Networking — Architecture Design, Topology Selection
---

# 🧠 GCP Networking: Network Architecture & Topologies

> **Note Type:** Architecture Decision Record (ADR)
> **Why this type?** This module is explicitly structured as an architectural decision framework — it teaches *how to think* about network design, what constraints to evaluate, and which topology pattern to select for a given scenario.

---

## 📦 What This Module Covers

| # | Module | Focus |
|---|--------|-------|
| 1 | Introduction to Network Architecture | Design pillars, components, design process |
| 2 | Network Topologies | Hub-and-spoke, Mesh, Mirrored, Gated — when and how to use each |

---

## Module 1 — Introduction to Network Architecture

### What Is Network Architecture?

Network architecture is the **design of a virtual network** in Google Cloud — how VMs, containers, and resources connect and communicate. You are responsible for configuring VPCs, firewalls, routers, VPNs, and on-premises access points. Google manages the underlying physical infrastructure.

> A poorly designed network leads to outages, security breaches, and slow performance. Not all designs suit all situations.

---

### The Four Design Pillars

| Pillar | What it means |
|---|---|
| **Scalability** | Can the network handle growing users, apps, and traffic without redesign? |
| **Security & Compliance** | Are resources protected? Does design comply with regulations (data segregation, logging, access control)? |
| **Performance** | Does the design minimise latency and maximise throughput for your workloads and geography? |
| **Cost Efficiency** | Is the design right-sized? Using committed use discounts, managed services, and appropriate tiers? |

---

### Network Architecture Components

| Component | Architectural Role |
|---|---|
| **VPC Network** | Isolated virtual network — groups resources by function, security boundary, or project |
| **Cloud VPN** | Encrypted tunnel over public internet (1.5–3 Gbps/tunnel) for on-premises connectivity |
| **Cloud Interconnect** | Direct physical connection (10–100 Gbps) for high-bandwidth, low-latency hybrid connectivity. Can layer Cloud VPN over Interconnect for encryption |
| **Cloud Router** | Dynamic BGP route exchange between VPCs and on-premises; control plane for Cloud NAT; works with Interconnect, VPN, and Router Appliance |
| **Load Balancers** | Distribute traffic across backends — enable HA, autoscaling, and global reach |
| **Firewall Rules** | Granular per-VM access control; group into policies for easier management |
| **Cloud CDN** | Cache content at Google's global edge PoPs; accelerates web apps (requires Premium Tier) |
| **Network Service Tiers** | Premium = Google private backbone; Standard = public internet (lower cost) |
| **Shared VPC** | Centralised network mgmt — host project shares subnets with service projects (same org) |
| **VPC Network Peering** | Private RFC1918 connectivity across VPCs — same or different org; decentralised admin |
| **Centralized Appliances** | IDS/IPS, WAF, custom load balancers as VMs — enhance protection beyond native GCP firewall |

---

### Network Design Process

```
Step 1: Identify stakeholders & requirements
  └── App owners, security architects, ops managers, finance
  └── Gather: security/compliance needs, perf targets, budget, scale projections

Step 2: Understand existing environment
  └── Inventory existing VPCs, subnets, IAM, on-prem systems

Step 3: Define technical constraints
  └── Compliance (GDPR, PCI-DSS, HIPAA) → data segregation, logging
  └── Cost → right-sizing, committed discounts, managed services
  └── Operations → roles, automation, monitoring, incident response

Step 4: Build High-Level + Low-Level Designs
  └── HLD: topology, connectivity strategy, region/zone selection
  └── LLD: IP address planning (no CIDR overlaps), security configs

Step 5: Build BoM + Calculate Cost
  └── List all services, licenses, resources
  └── Calculate CapEx + OpEx → present to stakeholders
```

---

### Inefficient vs Optimised Network Design

#### Inefficient (before)

```
Client → [Web Server] → [Business Logic] → [Database]
          All 3 VMs in one subnet, one region
```

Problems: Single point of failure, no security segmentation, bottleneck at scale.

#### Optimised (after)

```
Client
  → Load Balancer
    ├── us-east1:    [Web Server /20] → [Business Logic /20] → [Database /20]
    └── us-central1: [Web Server /20] → [Business Logic /20] → [Database /20]
```

Improvements:
- Load balancer distributes requests — eliminates bottleneck
- Each tier in its own subnet — different firewall rules per tier
- Database locked to only accept traffic from the business logic subnet
- Two regions — survive regional failure

> Note: What is optimal for one VPC environment may not be optimal for another.

---

## Module 2 — Network Topologies

### ⚖️ Topology Comparison

| Topology | Structure | Best For | Admin Model |
|---|---|---|---|
| **Hub-and-Spoke** | Central hub VPC; spokes connect through hub | Centralised enterprise network management | Centralised |
| **Mesh** | Multiple direct interconnections between nodes | High-availability microservices; mission-critical failover | Decentralised |
| **Mirrored** | Identical replica of network in another region/env | DR, dev/test parity, global distribution | Independent per env |
| **Gated** | API gateway/firewall controls traffic in/out | Strict security, regulated industries, hybrid/multi-cloud | Centralised at gate |

---

### Hub-and-Spoke Topology

```
On-premises
    │
    ▼
Shared Resources VPC (hub)
    ├──→ Education-VPC (spoke)
    ├──→ Healthcare-VPC (spoke)
    ├──→ Banking-VPC (spoke)
    └──→ Automotive-VPC (spoke)
```

The hub acts as a **transit point and central control plane**. All inter-spoke traffic flows through the hub.

#### Benefits

| Benefit | Detail |
|---|---|
| Centralised control | Configure, monitor, and troubleshoot the entire network from the hub |
| Simplified administration | Hub auto-routes between spokes based on predefined rules |
| Scalability | Add spokes without impacting existing network config |
| Improved security | Enforce centralised security policies at the hub |

#### Implementation Options

| Method | Trade-offs |
|---|---|
| **VPC Network Peering** | Low latency; but no transitive routing — spokes cannot reach each other directly |
| **Cloud VPN** | Flexible cross-region/on-premises; higher latency, more config |
| **Network Connectivity Center (NCC)** | ✅ Recommended — managed hub-and-spoke; centralised config, auto provisioning, integrated security; additional cost |

#### Network Connectivity Center (NCC) Spoke Types

| Type | What it connects |
|---|---|
| **VPC Spoke** | A VPC network |
| **Hybrid Spoke** | On-premises via HA VPN tunnels, Cloud Interconnect VLAN attachments, or Router Appliance VMs |

**NCC constraints:**
- No overlapping IP spaces between hub, spokes, and on-premises networks
- IPv6 not supported for hybrid spokes
- Privately-used public IPs (PUPIs) are supported

```bash
# Create NCC hub
gcloud network-connectivity hubs create my-hub

# Attach VPC spoke
gcloud network-connectivity spokes linked-vpc-networks create spoke-a \
  --hub=my-hub \
  --vpc-network=projects/PROJECT/global/networks/spoke-vpc \
  --region=us-central1
```

---

### Mesh Topology

```
Full Mesh:     A ── B     Partial Mesh:  A ── B
               │╲╱│                     │
               │╱╲│                     C ── D
               C ── D                   (strategic connections only)
```

#### Two Types

| Type | Use Case |
|---|---|
| **Full Mesh** | GKE Enterprise — all microservices need direct low-latency paths to each other |
| **Partial Mesh** | Regional failover — connect VPCs across regions; configure routing to active region with automatic failover |

#### Benefits

- **High availability** — multiple paths mean no single point of failure
- **Performance** — multiple pathways allow better load balancing
- **Security** — no central hub to compromise; distributed paths

#### Mesh vs Hub-and-Spoke

| Need | Choose |
|---|---|
| All services talk directly to all others | Full Mesh |
| Failover between regions automatically | Partial Mesh |
| Centralised control, simple admin, enterprise | Hub-and-Spoke |

---

### Mirrored Topology

```
Production (us-east1-a)            Test/DR (us-east2-a)
  my-network                         my-network-test
    └── my-server                      └── my-server
        [identical config]                 [isolated, identical]
```

Creates an **exact replica** of your network in a different region or environment.

| Use Case | Detail |
|---|---|
| **Disaster Recovery** | Standby environment ready to take over with minimal downtime |
| **Testing & Development** | Identical but isolated — changes here have zero impact on production |
| **Global Workload Distribution** | Mirror near users → reduced latency |

---

### Gating Topologies

Fine-grained control over what traffic enters or exits your cloud environment through a structured gateway layer.

```
Application VPC (workloads)
  ↕ API Requests
API Gateway or Proxy  ←→  Transit VPC Firewall  ←→  Cloud VPN/Interconnect  ←→  External
```

#### Three Types

| Type | Controls | Best For |
|---|---|---|
| **Gated Egress** | Outbound traffic from cloud to external | Prevent data exfiltration; control which external services cloud workloads reach |
| **Gated Ingress** | Inbound traffic from external to cloud | Strict control over which external sources access internal services |
| **Gated Ingress + Egress** | Both directions | Highly regulated hybrid/multi-cloud (finance, healthcare, government) |

> 💡 **PCA tip:** Financial services needing to isolate sensitive data and limit communication between segments = **Gated ingress and egress**. Multiple offices needing central management = **Hub-and-spoke**.

---

### Network Topology Visualisation Tool

Part of **Network Intelligence Center** — a live visual graph of your deployed network.

| Feature | Detail |
|---|---|
| **Infrastructure view** | VPC networks, hybrid connectivity, Google-managed service connections |
| **Real-time data** | Traffic paths, throughput, connection state |
| **History** | 6 weeks of data in hourly snapshots |
| **Hierarchical display** | VMs → instance groups → zones; GKE pods → workloads → namespaces → clusters |

#### Troubleshooting Workflow with Network Topology

```
Symptom: App latency spikes reported
  1. Open Network Topology → view full graph
  2. Filter by specific load balancer (e.g., shopping-site-lb)
  3. Select traffic metrics: Americas → LB
     → view QPS + HTTP latency percentiles (p50, p95, p99)
  4. Extend time series to 6 weeks → find when spike started
  5. Correlate with backend health → remove slow backend instance
  → Latency resolved
```

---

## Best Practices: Hybrid Cloud Network Architecture

| Practice | Why it matters |
|---|---|
| Match connectivity to required SLA | VPN = 99.9%; Dedicated Interconnect = 99.99% |
| Centralise hybrid connectivity in hub VPC + peer spokes | Share static and dynamic routes across projects cleanly |
| Expose apps via API gateway (Apigee) or Cloud LB | Adds security, rate limiting, quotas, analytics |
| Use Cloud LB application capacity optimisation | Handles capacity challenges in globally distributed apps |
| Use two authoritative DNS systems | One for GCP private resources; one for on-premises — prevents DNS conflicts |

---

## 🎯 PCA Exam Scenarios to Know

| Scenario | Answer |
|---|---|
| Central networking team managing many project networks | Hub-and-spoke (NCC or Shared VPC as hub) |
| GKE microservices needing direct low-latency paths to all others | Full Mesh topology |
| Create isolated identical environment for testing (no prod impact) | Mirrored topology |
| Financial services: isolate sensitive data + limit inter-segment traffic | Gated ingress and egress topology |
| Many dispersed regional offices → central management + isolation | Hub-and-spoke |
| Latency issue — need to trace to specific LB backend over 6 weeks | Network Topology tool → filter by LB → extend time series |
| Hub-and-spoke connecting on-premises over Interconnect via NCC | Hybrid Spoke (Cloud Interconnect VLAN attachment) |
| Hub-and-spoke — spokes need to communicate with each other | Route inter-spoke traffic through hub (NCC handles this automatically) |

---

## 🔗 Architecture Decision Quick Reference

```
Requirement                              →  Architecture Choice
─────────────────────────────────────────────────────────────────
Centralised control, enterprise network  →  Hub-and-Spoke (NCC)
High HA, all services talk to all        →  Full Mesh
DR / dev-test env parity                 →  Mirrored
Regulated industry, strict traffic gating→  Gated (ingress/egress)
Cross-org private connectivity           →  VPC Network Peering
Same-org centralised network admin       →  Shared VPC
High-bandwidth on-prem (≥10 Gbps, SLA)  →  Dedicated Interconnect
Low-cost / dev on-prem hybrid            →  Cloud VPN (HA VPN)
Dynamic BGP route exchange (hybrid)      →  Cloud Router
Performance-critical, global users       →  Premium Network Tier
Cost-sensitive, single-region only       →  Standard Network Tier
```

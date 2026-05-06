---
title: GCP Networking in Google Cloud — VPC Fundamentals
tags: [gcp, vpc, networking, network-tiers, shared-vpc, vpc-peering, flow-logs, network-intelligence-center, pca]
note_type: Service Blueprint
source: Networking in Google Cloud — VPC Networking Fundamentals (Instructor-led)
exam_domain: Networking — VPC Design, Connectivity, Observability
---

# 🏗️ GCP Networking in Google Cloud: VPC Fundamentals

> **Note Type:** Service Blueprint  
> **Why this type?** This module is a deep structural walkthrough of GCP's VPC networking model — how VPCs, subnets, NICs, and sharing mechanisms are architected — and how the network monitoring toolchain (Network Intelligence Center, VPC Flow Logs, Packet Mirroring) is built and used.

---

## 📦 What This Module Covers

| # | Module | Key Concepts |
|---|--------|-------------|
| 1 | VPC Networking Fundamentals | VPC internals, multiple NICs, Network Service Tiers (Premium vs Standard) |
| 2 | Sharing VPC Networks | VPC Network Peering, Shared VPC, VM migration between networks |
| 3 | Network Monitoring and Logging | Network Intelligence Center, VPC Flow Logs, Flow Analyzer, Packet Mirroring |

---

## Module 1 — VPC Networking Fundamentals

### What is a VPC Network?

A **Virtual Private Cloud (VPC)** is a virtual version of a physical network, providing:

- Connectivity for **Compute Engine VMs**, GKE clusters, App Engine flexible instances
- Built-in **internal passthrough Network Load Balancers** and proxy systems for internal Application LBs
- Distribution of traffic from **external load balancers** to backends
- Connectivity to on-premises networks via **Cloud VPN** and **Cloud Interconnect**

```
VPC Network (global)
  ├── Subnet (regional) — us-central1
  │     ├── VM instances in zone -a
  │     └── VM instances in zone -b
  └── Subnet (regional) — europe-west1
        └── VM instances in zone -c
```

---

### VPC Network Modes

| Mode | Subnets | Best For |
|---|---|---|
| **Auto mode** | One per region, automatically created with predefined CIDR ranges | Learning / quick start |
| **Custom mode** | You define subnets, regions, and CIDR ranges | ✅ Production — Google recommended |

> 💡 New projects get a **default auto mode VPC** — delete it or convert to custom mode before production use.

---

### Default Routing Behaviour

- Every VPC has **routes for all subnets** — instances can reach each other across subnets without a router
- Every VPC has a **default route** (0.0.0.0/0) directing traffic outside the network to the internet gateway
- **Custom routes** can override default routes (e.g., route specific CIDRs through a VPN tunnel or appliance)
- Routes alone are not enough — **firewall rules must also allow the traffic**

> ⚠️ The **default VPC** comes with pre-configured firewall rules allowing all instances to communicate with each other. **Manually created VPCs do not** — you must create firewall rules explicitly.

---

### Multiple Network Interface Controllers (NICs)

#### Why Multiple NICs?

VPC networks are **isolated by default** — instances in different VPCs cannot communicate via internal IPs without explicit mechanisms. Multiple NICs allow a **single VM to bridge multiple isolated VPC networks**.

Use cases:
- **Virtual network appliance** — security appliance, IDS/IPS, WAF, load balancer sitting between networks
- **Traffic separation** — separate data plane traffic from management/control plane traffic
- **Multi-tenant environments** — isolate traffic streams while sharing compute

#### How NICs Work

```
VM Appliance
  ├── nic0 → subnet-1 in VPC-A  (data plane)
  ├── nic1 → subnet-1 in VPC-B  (management plane)
  ├── nic2 → subnet-2 in VPC-C  (partner network)
  └── nic3 → subnet-1 in VPC-D  (on-prem via VPN)
```

#### NIC Rules

| Rule | Detail |
|---|---|
| Each NIC → different VPC | No two NICs can be on the same VPC |
| Each NIC has an internal IP | External IP is optional per NIC |
| NICs are immutable | **Cannot add or remove NICs after instance creation** — plan upfront |
| Uses internal IP across networks | Communication between VPCs via NIC uses internal IP (not external) |

> 💡 **PCA tip:** When a question involves a virtual appliance filtering traffic between VPCs (firewall, IDS/IPS, WAF), the answer is **multiple NICs** — not VPC peering or Shared VPC.

#### Example Architecture: Security Appliance

```
Internet
    ↓
VM Appliance (nic0: untrusted internet-facing, nic1: VPC-A web tier, nic2: VPC-B DB tier)
    ├── Firewall rules applied per NIC independently
    ├── Inspects and filters control plane vs data plane traffic
    └── Routes between VPC-A and VPC-B internally
```

---

### Network Service Tiers

#### The Problem

Traffic from your GCP workloads to end users must travel over a network. By default, that network is the **public internet** — subject to congestion, variable latency, and ISP routing decisions. Google offers two network tiers to control this trade-off.

---

#### ⚖️ Premium Tier vs Standard Tier

| Feature | **Premium Tier** | **Standard Tier** |
|---|---|---|
| Routing | **Cold potato** — enters Google network at nearest PoP to user; stays on Google's private backbone until destination | **Hot potato** — exits Google network near source; travels over public internet |
| Performance | Highest — minimum hops, minimum congestion | Comparable to other cloud providers |
| Availability SLA | **99.99%** | **99.9%** |
| IP addresses | Regional and **global** external IPv4 + IPv6 | Regional external IPv4 only (no BYOIP) |
| Load balancing | Global + regional external LBs, Cloud CDN, GKE, Cloud NAT, Cloud VPN | Regional only: Cloud NAT, Regional External Application LB, External Passthrough Network LB |
| Unique to Google | ✅ Yes — Google's private backbone | ❌ No — ISP networks |
| Price | Higher | Lower |
| When to choose | Performance-critical, global apps, multi-region users | Cost-sensitive, single-region, standard workloads |

#### Network Service Tiers Decision Tree

```
What matters most for this workload?
├── Performance / low latency  →  Premium Tier
└── Cost optimisation          →  Standard Tier (but check limits first)

If Standard Tier — do you need any of the following?
  ├── Global load balancing?    → ❌ Not available → use Premium
  ├── Cloud CDN?                → ❌ Not available → use Premium
  ├── Multi-region backends?    → ❌ Not available → use Premium
  └── None of the above?        → ✅ Standard Tier is fine
```

#### How Premium Tier Works (Cold Potato Routing)

```
End User (Tokyo)
  → Enters Google Network at Tokyo PoP (closest PoP to user)
  → Travels entirely on Google's private fiber backbone
  → Exits near destination GCP region (e.g., us-central1)
  → Single hop to end user's ISP at destination

Result: consistent, low-latency, high-throughput path
```

#### How Standard Tier Works (Hot Potato Routing)

```
GCP Region (us-central1)
  → Exits Google network near the source region
  → Travels over public internet / ISP networks
  → Arrives at end user via public routing

Result: lower cost, variable latency, depends on ISP quality
```

#### Pricing Model

- **Premium Tier:** Priced on both **source AND destination** (because traffic travels far over Google's network)
- **Standard Tier:** Priced on **source only** (traffic exits Google network quickly)
- Geographic pricing tiers: NA ↔ NA cheapest; intercontinental traffic more expensive; Oceania and China most expensive

---

## Module 2 — Sharing VPC Networks

### VPC Network Peering

**VPC Network Peering** allows **private RFC1918 IP connectivity** between two VPC networks — regardless of whether they belong to the same project, same organisation, or different organisations.

#### How Peering is Established

```
Org A (Producer)           Org B (Consumer)
  VPC-Producer               VPC-Consumer
      ↓                           ↓
  Admin peers                Admin peers
  Producer → Consumer        Consumer → Producer
      ↓                           ↓
  Both sides configured?  →  Session Active → Routes Exchanged
```

> Both sides **must independently configure** the peering. If only one side peers, the session stays inactive.

#### VPC Peering Properties

| Property | Detail |
|---|---|
| **Transitive peering** | ❌ Not supported — if A↔B and B↔C, A cannot reach C via B |
| **Subnet CIDR overlap** | ❌ Not allowed — peered VPC subnets must not overlap |
| **Administrative separation** | ✅ Each VPC keeps its own firewall rules, routes, VPN independently |
| **Deletion** | Either side can delete the peering at any time |
| **Works with** | Compute Engine, GKE, App Engine flexible |
| **Route exchange** | Dynamic — custom routes can optionally be exported/imported |

#### Advanced: Peering with a Shared VPC Network

```
Host Project P1
  Shared VPC network-SVPC
    └── Subnet 1, Subnet 2, Subnet 3
          ├── Service Project P3 (VM3 in Subnet 1)
          └── Service Project P4 (VM4 in Subnet 2)

Project P2
  network-A (peered with network-SVPC)
    └── VM2

Result: VM1 (P3), VM2 (P2), VM4 (P4) all communicate via private IPs
```

You can also peer two Shared VPC networks together.

---

### Shared VPC

**Shared VPC** allows a **host project** to share its VPC subnets with **service projects** within the same organisation. Service project VMs run in the host VPC's subnets — centrally managed networking.

#### Roles in Shared VPC

| Role | Responsibility |
|---|---|
| **Organisation Admin** | Nominates Shared VPC admins |
| **Shared VPC Admin** | Sets up host project, attaches service projects |
| **Service Project Admin** | Creates VMs in the shared subnets |

#### Architecture

```
Organisation
  └── Host Project (owns shared VPC network)
        ├── Shared VPC network (subnets: subnet-A, subnet-B)
        └── Service Project 1 → VMs run in subnet-A
        └── Service Project 2 → VMs run in subnet-B
        └── Service Project 3 → VMs run in subnet-A or subnet-B
```

---

### ⚖️ Shared VPC vs VPC Network Peering

| Consideration | **Shared VPC** | **VPC Network Peering** |
|---|---|---|
| Across organisations | ❌ No | ✅ Yes |
| Within same project | ❌ No | ✅ Yes |
| Cross-project, same org | ✅ Yes | ✅ Yes |
| Network administration | **Centralised** — one host VPC, one admin team | **Decentralised** — each VPC manages its own firewall/routing |
| Firewall management | Central host project controls firewall | Each VPC maintains its own rules |
| IP address ownership | VMs get IPs from host VPC subnets | VMs keep IPs in their own VPC |
| Use case | Enterprise with central networking team; compliance-driven | SaaS producer↔consumer; separate orgs; decentralised teams |

> 💡 **Choosing guide:**
> - Need **central network control + compliance**? → Shared VPC
> - Need to connect **different orgs or same project**? → VPC Peering
> - Need **transitive routing** (A → B → C)? → Neither; use VPN or a hub-and-spoke NVA

---

### VM Migration Between VPC Networks

You can move a VM from one VPC network to another **within the same zone and project** (intra-project migration).

**Pre-migration requirements:**
- VM must be in the same zone/project as the target network
- No conflicting subnet CIDR ranges
- Target network must have appropriate firewall rules

**Supported migrations:**
- Auto mode network → Custom mode network
- Custom mode → Custom mode (same project, same zone)

> ⚠️ Cross-project VM migration is **not supported**. Use snapshots + re-deployment for that.

---

## Module 3 — Network Monitoring and Logging

### Network Intelligence Center

**Network Intelligence Center** is GCP's centralised network observability platform. It shifts network operations from **reactive to proactive** — 75% of network outages are caused by misconfiguration, often discovered only in production.

```
Network Intelligence Center
  ├── Topology        → Visualise VPC network graph
  ├── Connectivity Tests → Self-diagnose reachability issues
  ├── Performance Dashboard → Real-time latency and packet loss metrics
  ├── Firewall Insights → Analyse firewall rule usage and optimise rules
  └── Network Analyzer → Auto-detect misconfigurations and suggest fixes
```

---

### Connectivity Tests

**Self-service reachability testing** — diagnose whether a packet can travel from source to destination given current network configuration.

- Tests paths: **VM → VM**, **VM → external IP**, **GCP → on-premises**
- Checks: firewall rules, routes, VPC peering, Shared VPC, NAT
- **Saves and reruns tests** — detect if config changes break expected connectivity
- Used internally by Google Cloud Support for customer issue resolution

```
Create test:
  Source: VM-A (10.10.0.2, us-central1-a)
  Destination: VM-B (10.50.0.2, us-east1-b)
  Protocol: TCP, Port 443

Result: REACHABLE or UNREACHABLE + reason (e.g., "Firewall rule blocks egress on port 443")
```

---

### Performance Dashboard

- Shows **real-time latency and packet loss** between zones where you have VMs
- Helps distinguish: **is this an application problem or a network problem?**
- Shows both **customer project metrics** and **Google Cloud general metrics**
- Query example (via Gemini): *"What's the average latency between my VMs in us-east4 and us-central1?"* → Performance Dashboard

---

### Firewall Insights

Helps you **understand, verify, and optimise firewall rules** using real traffic data.

**Data sources:**
- **Cloud Monitoring metrics** — derived from Firewall Rules Logging
- **Recommender insights** — ML-based suggestions for rule optimisation

**What you can do with Firewall Insights:**

| Task | How |
|---|---|
| Analyse rule usage | See which rules are being hit and which are unused |
| Track connection behaviour | Verify rules allow/block correct traffic over time |
| Diagnose dropped connections | Find rules accidentally blocking legitimate traffic |
| Identify potential threats | Detect anomalous spikes in firewall rule hit counts |

> ⚠️ Must enable **Firewall Rules Logging** per firewall rule to populate Firewall Insights metrics.

---

### Network Analyzer

- **Continuously monitors** VPC network configurations automatically
- Detects **misconfigurations** (e.g., missing firewall rules, route conflicts) and **suboptimal configurations**
- Runs on near-real-time config updates — triggers analysis when config changes
- Provides: root cause information + suggested resolution

**Example insight:**
```
Type: Error
Issue: GKE node cannot reach control plane
Root cause: Ingress firewall rule blocking node → control plane connection
            (default rule was deleted/shadowed)
Fix: Create new firewall rule OR increase priority of correct rule
```

---

### VPC Flow Logs

**VPC Flow Logs** record a sample of **network flows** sent from and received by VM instances within a VPC subnet.

#### Key Properties

| Property | Detail |
|---|---|
| **Granularity** | Per VM instance; captures both directions of a flow |
| **Latency** | Near-real-time — log updates every **5 seconds** |
| **Performance impact** | None — no delay or penalty to live traffic |
| **Enablement** | Per-subnet (applies to all VMs in that subnet) |
| **Sample rate** | Configurable (default ~50%) — tradeoff between coverage and log volume/cost |

#### What Each Log Record Contains (5-tuple + more)

```
connection.src_ip      Source IP address
connection.src_port    Source port
connection.dest_ip     Destination IP address
connection.dest_port   Destination port
protocol               Protocol number
bytes_sent             Bytes in this flow
start_time / end_time  First/last observed packet timestamps
instance details       VM name, zone, project
VPC details            Network, subnetwork
geo details            Country/region of source/destination
```

#### Use Cases

| Use Case | How Flow Logs Help |
|---|---|
| **Network monitoring** | Understand traffic volumes and patterns |
| **Security forensics** | Investigate suspicious connections post-incident |
| **Real-time security analysis** | Stream logs to SIEM for anomaly detection |
| **Cost optimisation** | Identify unnecessary cross-region or cross-zone traffic |
| **Compliance** | Audit trail of all network communication |

#### Exporting and Analysing Flow Logs

```
VPC Subnet (Flow Logs enabled)
  → Cloud Logging (automatic ingestion)
    └── Log Sink → BigQuery
          └── Looker Studio (visualise traffic patterns)
          └── Flow Analyzer (query without SQL)
```

---

### Flow Analyzer

A **purpose-built UI tool** for analysing VPC Flow Logs without writing SQL.

- Built on **Log Analytics**, powered by **BigQuery**
- Provides **5-tuple granularity** (src IP, dst IP, src port, dst port, protocol)
- In-depth inbound/outbound traffic analysis per VM
- Use for: monitoring, troubleshooting, optimisation, compliance, cost analysis

> Flow Analyzer vs raw BigQuery: Use Flow Analyzer for exploratory network traffic analysis. Use BigQuery directly for custom aggregations and joining with other datasets.

---

### Packet Mirroring

**Packet Mirroring** clones traffic from specific VM instances and forwards it to a **security/analysis appliance** for deep inspection.

| Feature | Detail |
|---|---|
| What it captures | **All** ingress and egress traffic — full payload + headers (not sampled) |
| Where mirroring happens | On the **VM instance** (not the network switch) |
| Bandwidth impact | ✅ Consumes additional bandwidth on the host — plan capacity |
| vs VPC Flow Logs | Flow Logs = sampled metadata; Packet Mirroring = full payload, all packets |
| Use case | Security analysis, intrusion detection, full application performance analysis |

```
VM Instance (mirrored)
  ├── Live traffic → destination (unaffected)
  └── Cloned traffic → IDS/IPS appliance or network analysis tool
```

---

### ⚖️ Network Observability Tools Compared

| Tool | What it answers | Data type | Use for |
|---|---|---|---|
| **Cloud Monitoring dashboards** | Is my VM's CPU/network healthy? | Metrics | Capacity planning, alerting |
| **Uptime checks** | Is my service reachable from the internet? | Availability | SLA monitoring |
| **Connectivity Tests** | Can packet A reach host B given current config? | Config simulation | Pre-prod validation, troubleshooting |
| **Performance Dashboard** | What's the latency/packet loss between my zones? | Real-time metrics | Network quality troubleshooting |
| **Firewall Insights** | Are my firewall rules working as intended? | Metrics + ML insights | Rule audit, optimisation |
| **Network Analyzer** | Is my VPC config correct and optimal? | Config analysis | Proactive misconfiguration detection |
| **VPC Flow Logs** | What traffic is flowing between my VMs? | Sampled flow metadata | Security forensics, cost analysis |
| **Flow Analyzer** | Which VMs are talking to what, by port/protocol? | 5-tuple flow data | Traffic analysis without SQL |
| **Packet Mirroring** | What exactly is in the packets my VMs send/receive? | Full packet payload | Deep security inspection, IDS/IPS |

---

## 🎯 PCA Exam Scenarios to Know

| Scenario | Answer |
|---|---|
| VM needs to act as a firewall/IDS between two VPC networks | Multiple NICs — one per VPC |
| Cannot add NICs to an existing VM — what to do? | Create a new VM with the required NICs from the start |
| Low latency global app; users worldwide | Premium Tier (cold potato routing; stays on Google backbone) |
| Cost-optimised single-region internal app | Standard Tier |
| Need global load balancing or Cloud CDN | Requires Premium Tier |
| Two teams in different orgs need private RFC1918 connectivity | VPC Network Peering |
| Central networking team needs to control firewall rules for multiple projects | Shared VPC (centralised admin model) |
| VM in Project A needs to access VM in the same project's other VPC | VPC Network Peering (Shared VPC doesn't work within same project) |
| Detect if a firewall rule change broke connectivity before deploying | Connectivity Tests (Network Intelligence Center) |
| Investigate which IP addresses connected to a compromised VM | VPC Flow Logs → export to BigQuery |
| Deep packet inspection for IDS without affecting live traffic | Packet Mirroring |
| Find root cause of GKE node not reaching control plane | Network Analyzer (auto-detects and suggests fix) |
| Measure latency between zones in real-time | Network Intelligence Center Performance Dashboard |
| Analyse VPC traffic patterns without writing SQL | Flow Analyzer |

---

## 🔗 Key CLI Reference

```bash
# Create custom VPC
gcloud compute networks create my-vpc --subnet-mode=custom

# Create subnet
gcloud compute networks subnets create my-subnet \
  --network=my-vpc \
  --range=10.0.0.0/24 \
  --region=us-central1

# Create VM with multiple NICs
gcloud compute instances create my-appliance \
  --network-interface=network=vpc-a,subnet=subnet-a \
  --network-interface=network=vpc-b,subnet=subnet-b \
  --network-interface=network=vpc-c,subnet=subnet-c \
  --zone=us-central1-a

# VPC Network Peering
gcloud compute networks peerings create peer-a-to-b \
  --network=vpc-a \
  --peer-network=vpc-b \
  --peer-project=project-b

# Enable VPC Flow Logs on a subnet
gcloud compute networks subnets update my-subnet \
  --region=us-central1 \
  --enable-flow-logs \
  --logging-aggregation-interval=interval-5-sec \
  --logging-flow-sampling=0.5

# Connectivity Test
gcloud network-management connectivity-tests create my-test \
  --source-instance=VM-A \
  --destination-instance=VM-B \
  --protocol=TCP \
  --destination-port=443
```

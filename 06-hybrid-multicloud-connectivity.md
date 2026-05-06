---
title: GCP Hybrid & Multicloud Connectivity Options
tags: [GCP, PCA, Networking, Hybrid, VPN, Interconnect]
module: "Networking in Google Cloud – Module 13 + Cloud VPN"
note_type: "⚖️ Compare & Choose"
---

# ⚖️ GCP Hybrid & Multicloud Connectivity Options

> **GCP Professional Cloud Architect Notes**
> Source: *Networking in Google Cloud – Module 13: Connectivity Options + Cloud VPN*

---

## 🏗️ Service Blueprint

### What Are We Connecting?

All hybrid connectivity options share one goal: provide **internal IP address access** between on-premises (or other cloud) resources and a **GCP VPC network**.

### The Four Options at a Glance

| Connection | Provides | Capacity | Requirements |
|---|---|---|---|
| **Cloud VPN** | Encrypted IPsec tunnel over public internet | 1.5–3 Gbps/tunnel | Remote VPN gateway |
| **Dedicated Interconnect** | Direct physical fiber to VPC | 10 or 100 Gbps/link | Colocation facility presence |
| **Partner Interconnect** | Dedicated bandwidth via service provider | 50 Mbps – 50 Gbps | Supported service provider |
| **Cross-Cloud Interconnect** | Dedicated physical link to another cloud provider | 10 or 100 Gbps | Primary + redundant ports on both clouds |

> ⚠️ **Encryption note:** Dedicated Interconnect and Partner Interconnect do **not** encrypt data in transit by default. Use MACsec or TLS at the application layer.

---

### Cloud Router & BGP — The Glue

All Interconnect options rely on **Cloud Router** + **BGP** for dynamic route exchange:

- A VLAN attachment is linked to a Cloud Router
- Cloud Router opens a BGP session with the on-prem peer router
- Routes are exchanged bidirectionally and installed as **custom dynamic routes** in the VPC

---

## ⚖️ Compare & Choose

### Decision Tree

```
Need to connect on-prem to GCP?
│
├─ Do you need encryption in transit? ──► Cloud VPN (IPsec)
│
├─ Can you physically co-locate at a Google facility?
│   └─ YES ──► Dedicated Interconnect (10/100 Gbps per link)
│
├─ No colocation access, but have a service provider?
│   └─ YES ──► Partner Interconnect (50 Mbps–50 Gbps)
│
├─ Connecting GCP ↔ AWS / Azure / OCI / Alibaba Cloud?
│   └─ YES ──► Cross-Cloud Interconnect
│
└─ Low bandwidth or quick setup needed? ──► Cloud VPN or Partner Interconnect
```

---

### Cloud VPN vs Cloud Interconnect

| Factor | Cloud VPN | Cloud Interconnect |
|---|---|---|
| **Bandwidth** | Up to ~3 Gbps/tunnel | 10–200 Gbps |
| **Encryption** | Built-in IPsec ✅ | Not included (use MACsec/TLS) |
| **Setup complexity** | Low — software only | Higher — physical provisioning |
| **Cost** | Lower | Higher (port fees + egress) |
| **Latency** | Internet-dependent | Low, dedicated path |
| **Use when** | Lower bandwidth, encrypted transit needed | High throughput, mission-critical workloads |

---

### Dedicated vs Partner Interconnect

| Factor | Dedicated Interconnect | Partner Interconnect |
|---|---|---|
| **Physical connection** | You own fiber port at Google colocation | Via service provider network |
| **Bandwidth** | 10 or 100 Gbps per link (up to 8×10G or 2×100G) | 50 Mbps – 50 Gbps |
| **Setup time** | Longer (LOA-CFA process) | Faster (provider infra exists) |
| **Routing** | BGP via Cloud Router | L2: you configure BGP; L3: provider handles BGP |
| **When to use** | High throughput, colocation available | No colocation access, or sub-50 Gbps needs |

**If you need < 50 Mbps** → use **Cloud VPN** instead (Partner Interconnect minimum is 50 Mbps).

---

### Cross-Cloud Interconnect Supported Providers

- Amazon Web Services (AWS)
- Microsoft Azure
- Oracle Cloud Infrastructure (OCI)
- Alibaba Cloud

> Google supports the connection up to where it reaches the remote cloud's network. Google does **not** guarantee the remote cloud's uptime.

---

## 🧠 Architecture Decision Record (ADR)

### ADR-01: Choosing HA VPN over Classic VPN

**Status:** Recommended

**Context:** Two Cloud VPN variants exist — HA VPN and Classic VPN.

**Decision:** Prefer **HA VPN** for all new deployments.

**Rationale:**
- HA VPN provides **99.99% SLA** vs 99.9% for Classic VPN
- Requires **dynamic BGP routing** (no static routes)
- Google auto-assigns 2 external IPs from separate pools
- Supports active/active or active/passive tunnel configurations

**Classic VPN:** Only for legacy static routing use cases.

---

### ADR-02: High Availability for Dedicated Interconnect (99.99%)

To achieve 99.99% availability:
1. Create **at least 4 Interconnect connections**
2. Distribute across **2 metropolitan areas**
3. Within each metro, place connections in **2 different edge availability domains (EADs)**
4. Deploy **minimum 2 Cloud Routers** across at least 2 GCP regions

> This ensures scheduled maintenance never affects more than one connection at a time.

---

### ADR-03: MACsec for Cloud Interconnect Security

MACsec encrypts traffic at the link layer:

| Connection Type | Encrypted Segment |
|---|---|
| Dedicated Interconnect | Google peering edge ↔ on-prem router |
| Partner Interconnect | Google peering edge ↔ service provider edge |
| Cross-Cloud Interconnect | Google peering edge ↔ remote cloud router |

> MACsec does **not** encrypt traffic within Google's internal network. Combine with IPsec or TLS for end-to-end security.

---

### ADR-04: BGP Path Selection — Setting Base Priority

Cloud Router includes a **route priority (MED attribute)** in BGP advertisements:

- Range: `0` (highest priority) to `65535` (lowest)
- Default: `100`
- Configurable per VPN tunnel or VLAN attachment
- In **global dynamic routing mode**: MED = base priority + region-to-region cost (201–9999, Google-managed, not configurable)

---

## 🎯 Scenario Breakdown

### Scenario 1: Mission-Critical Migration (Dedicated Interconnect)

**Persona:** Sam, Network Engineer at Cymbal Corp
**Problem:** Migrating mission-critical workloads to GCP. Needs low latency, high throughput, no public internet exposure.
**Why not VPN?** Internet variability + VPN overhead degrades performance.
**Solution:** **Dedicated Interconnect** — fiber ports, dedicated bandwidth, optional MACsec/IPsec encryption.

---

### Scenario 2: Remote Data Center Without Colocation (Partner Interconnect)

**Persona:** Ken, Network Engineer at Cymbal Corp
**Problem:** Data center location has no access to a Google colocation facility.
**Solution:** **Partner Interconnect** — connects through a service provider's existing infrastructure, high bandwidth, low latency, no direct colocation needed.

---

### Scenario 3: Google Cloud ↔ Microsoft Azure (Cross-Cloud Interconnect)

**Persona:** Rob, Network Engineer at Cymbal Corp
**Problem:** Cymbal runs workloads across both GCP and Azure. Need consistent, secure, high-bandwidth connectivity between clouds.
**Solution:** **Cross-Cloud Interconnect** — dedicated physical connection between GCP VPC and Azure virtual network. Eliminates public internet latency and bandwidth caps.

---

### Scenario 4: Improving Reliability of On-Prem VPN (HA VPN)

**Persona:** Jack, Cloud Infrastructure Engineer at Cymbal Corp
**Problem:** Classic VPN causing occasional connectivity drops. Needs better reliability.
**Solution:** **HA VPN** — redundant gateways with automatic failover, 99.99% SLA, dynamic BGP routing.

---

## 📋 Setup Checklists

### Dedicated Interconnect Setup (5 Steps)
1. **Order** connection via Google Cloud Console → receive LOA-CFA
2. **Send LOA-CFA** to colocation facility vendor
3. **Test connection** (light levels → IP connectivity) via automated Google emails
4. **Create VLAN attachment** → Cloud Router initiates BGP session
5. **Configure on-premises router** using VLAN ID, interface IP, and peering IP from attachment

### Partner Interconnect Setup (5 Steps)
1. **Order** from a supported service provider
2. **Create VLAN attachment** → generates a pairing key
3. **Submit pairing key** + capacity details to service provider
4. **Activate** the VLAN attachment once provider confirms
5. **Configure on-premises router** with VLAN ID and BGP peering info

### Layer 2 vs Layer 3 (Partner Interconnect)
- **L2:** You configure BGP between your on-prem router and Cloud Router
- **L3:** Service provider establishes BGP on your behalf — no local BGP config needed

---

## 🧩 HA VPN Topology Reference

| Topology | Description |
|---|---|
| HA VPN → Two peer VPN devices | Each device on separate EAD; provides 99.99% |
| HA VPN → AWS Virtual Private Gateway | 4 tunnels (2 AWS connections × 2 IPs each); use transit gateway for ECMP |
| HA VPN ↔ HA VPN (GCP-to-GCP) | Two HA gateways, each with 2 interfaces cross-connected |
| HA VPN → Compute Engine VMs | VMs act as VPN peer devices (IPsec network appliances) |
| HA VPN over Cloud Interconnect | Two tiers: Interconnect tier (VLAN + Cloud Router) + VPN tier (HA VPN + Cloud Router) |

---

## 🔑 Key Exam Tips

- **Dedicated Interconnect** = physical presence at Google colocation required
- **Partner Interconnect** = no colocation needed, uses service provider
- **Cross-Cloud Interconnect** = GCP ↔ another cloud (AWS, Azure, OCI, Alibaba)
- **Cloud VPN** = cheapest, encrypted, up to ~3 Gbps, internet-based
- For **99.99% SLA** on Interconnect: 4 connections across 2 metros, 2 EADs each
- For **99.99% SLA** on HA VPN: 2 tunnels, each on a separate interface
- **BGP MED** controls path selection; lower value = higher priority (default: 100)
- **MACsec** encrypts at the link layer but NOT within Google's internal network
- Use **Cloud VPN** if bandwidth < 50 Mbps (below Partner Interconnect minimum)

---

*Notes generated from: Networking in Google Cloud — Module 13: Connectivity Options + Cloud VPN*

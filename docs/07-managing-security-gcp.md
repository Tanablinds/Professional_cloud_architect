---
title: Managing Security in Google Cloud
tags: [GCP, PCA, Security, IAM, VPC, Firewall, CloudIdentity, OrgPolicy]
modules: "Foundations of GCP Security | Securing Access | IAM | VPC Security"
note_type: "🏗️ Service Blueprint"
---

# 🏗️ Managing Security in Google Cloud

> **GCP Professional Cloud Architect Notes**
> Source: *Managing Security in Google Cloud — 4 Modules (205 pages)*

---

## Module 1: Foundations of Google Cloud Security

### Google's Infrastructure Security Layers (Bottom-Up)

Security is built in progressive layers — true **defense in depth**. No single control is relied upon alone.

```
┌─────────────────────────────────────┐
│  Operations (Operational Security)  │
├─────────────────────────────────────┤
│  Internet Communication (GFE, DoS)  │
├─────────────────────────────────────┤
│  Data Storage (Encryption at rest)  │
├─────────────────────────────────────┤
│  Service Deployment (RPC crypto)    │
├─────────────────────────────────────┤
│  Low-Level Infrastructure (Titan)   │
└─────────────────────────────────────┘
```

| Layer | Key Controls |
|---|---|
| **Low-level infra** | Custom data centers, Titan security chip, secure boot stack, hardware provenance |
| **Service deployment** | Cryptographic service identity, inter-service RPC encryption, 2-party code review, bug bounty |
| **Data storage** | Encryption at rest (default), CMEK/CSEK/EKM options, hardware tracking, secure disposal |
| **Internet communication** | Google Front End (GFE), multi-tier DoS protection, IPsec VPN, Cloud Interconnect |
| **Operations** | Secure software dev practices, intrusion detection, insider risk reduction |

---

### Shared Security Responsibility Model

Responsibility shifts based on service type:

| Layer | On-Premises | IaaS | PaaS | SaaS |
|---|---|---|---|---|
| Content | Customer | Customer | Customer | Customer |
| Access policies | Customer | Customer | Customer | Customer |
| OS / Data | Customer | Customer | Shared | Google |
| Network security | Customer | Customer | Google | Google |
| Hardware / Infra | Customer | Google | Google | Google |

**Customer always owns:**
- Data access control (IAM, API Gateways, Anthos Service Mesh)
- Compute access (service accounts, VM login)
- Network access (firewall rules, VPC Service Controls, VPN)

---

### Threats Mitigated Automatically by Google

| Threat | GCP Mitigation |
|---|---|
| **DoS / DDoS** | Built into Application Load Balancers; multi-tier Google Front End protection |
| **Application attacks (SQLi, XSS)** | Cloud Armor + WAF (OWASP Top 10, ModSecurity CRS) |
| **Physical access** | Layered data center security; <1% of Googlers access data centers |
| **Data at rest exposure** | All data chunked + encrypted by default; CMEK/CSEK/EKM available |
| **Data in transit** | All external traffic auto-encrypted; internal traffic authenticated |
| **Data disposal** | Deleted data purged from all systems within 180 days |

---

### Access Transparency

- **Cloud Audit Logs** — track actions by your own admins
- **Access Transparency** — near-real-time logs of Google support/engineering accessing your data
- **Access Approval API** — Google support can only access your project data after you give explicit approval
- Google does **not** scan customer data for ads or sell it to third parties
- You own your data; data is not processed beyond contractual obligations

---

### Encryption Options Summary

| Option | Who Manages Keys | Use Case |
|---|---|---|
| **Google-managed (default)** | Google | Default for all; no config needed |
| **CMEK** (Customer Managed) | Customer via Cloud KMS | Regulatory/compliance requirements |
| **CSEK** (Customer Supplied) | Customer provides key per request | Maximum customer key control |
| **External Key Manager (EKM)** | Customer, stored outside Google | Keys never leave customer infra |

---

## Module 2: Securing Access to Google Cloud

### Identity Options on GCP

| Service | Purpose |
|---|---|
| **Cloud Identity** | Managed identity service for GCP users; free + premium editions |
| **Google Workspace** | Full collaboration suite + identity (Workspace admins = GCP admins) |
| **Google Cloud Directory Sync (GCDS)** | One-way sync from existing LDAP/AD to Cloud Identity |
| **Managed Microsoft AD** | Fully managed Active Directory on GCP |
| **Identity Platform** | CIAM for apps (multi-tenant SaaS, mobile/web apps) |

---

### Cloud Identity

- Manages users and groups for GCP access
- Administered via **Google Admin Console**
- Free tier: core identity features
- Premium tier: MDM, app management, advanced security
- Replaces need for Google Workspace if you only need GCP access

---

### Google Cloud Directory Sync (GCDS)

- Runs **on your server**, not in Google's infrastructure
- **One-way sync only**: existing directory → Cloud Identity (never reverse)
- Syncs: users, groups, group members, aliases, shared contacts
- Enables **auto-provisioning and deprovisioning** based on your source directory
- Use when you have an existing corporate directory (LDAP/AD) and don't want to manage users twice

---

### Single Sign-On (SSO) with Cloud Identity

SSO setup requires **3 links + 1 certificate**:
1. Sign-in page URL (your IdP)
2. Sign-out page URL (your IdP)
3. Change password URL (your IdP)
4. Verification certificate (public key from your IdP)

---

### IAM Group Best Practices

- Manage access via **groups**, not individual users
- Groups defined in Google Workspace / Cloud Identity Admin console
- Assign IAM roles to groups → individual users inherit via group membership
- Minimizes IAM changes in GCP when team members change

---

## Module 3: Identity and Access Management (IAM)

### IAM = Who Can Do What on Which Resource

```
Principal (who)  →  Role (can do what)  →  Resource (on which resource)
```

---

### Resource Hierarchy

```
Organization (root)
  └── Folders (optional; group projects by dept/team/env)
        └── Projects (required; all resources belong to a project)
              └── Resources (VMs, buckets, topics, etc.)
```

- **Policies inherit downward**: a less restrictive parent policy overrides a more restrictive child policy
- Folders enable organizational structure: Department A → Prod + Dev projects
- Projects: track billing, enable APIs, assign permissions

---

### Three Types of IAM Roles

| Type | Granularity | Managed By | When to Use |
|---|---|---|---|
| **Basic** (Owner/Editor/Viewer) | Coarse — all services in project | Google | Dev/test only; avoid in production |
| **Predefined** | Service-level, job-function mapped | Google (auto-updated) | Default choice; maps to real job roles |
| **Custom** | Exact permissions you specify | You (not auto-maintained) | When predefined doesn't fit exactly |

**Basic Roles (concentric — each includes the one below):**
- **Owner** = invite/remove members, delete projects + all Editor permissions
- **Editor** = deploy apps, modify code, configure services + all Viewer permissions
- **Viewer** = read-only access to all resources
- **Billing Admin** = manage billing (separate from resource access)

---

### Service Accounts

Used for **server-to-server authentication** — not human users.

| Type | Who Creates | Who Manages |
|---|---|---|
| **Google-managed** | Google (auto-created for GCP services) | Google; not visible in your console |
| **User-managed** | You | You (up to 100 per project) |

**Service Account Keys:**

| Type | Key Storage | Risk |
|---|---|---|
| **Google-managed keys** | Google stores public + private; private never directly accessible | Lower risk |
| **User-managed keys** | You hold private key; Google stores only public key | Higher risk — losing the key is unrecoverable |

> ⚠️ Prefer **Workload Identity Federation** over user-managed service account keys for external workloads.

```bash
# List all keys for a service account
gcloud iam service-accounts keys list --iam-account SERVICE_ACCOUNT_EMAIL
```

---

### Workload Identity Federation

**Problem:** Service account keys are powerful credentials — security risk if leaked.

**Solution:** Grant external identities (AWS, Azure, on-prem) temporary GCP tokens without long-lived keys.

**How it works:**
1. Create a **Workload Identity Pool** in your GCP project
2. External app authenticates with its own IdP → receives IdP credential
3. App exchanges credential with GCP **Security Token Service** → gets short-lived access token
4. Token impersonates a service account → accesses GCP resources

> No service account keys are created or stored. Tokens expire automatically.

---

### IAM Policy Hierarchy

- **Allow policies** = grant access; inherit downward
- **Deny policies** = block access regardless of allow policies; **deny is evaluated first**
- A less restrictive parent policy **always overrides** a more restrictive child policy
- IAM Conditions: grant access only when attribute conditions are met (e.g., resource name prefix, time-based access)

---

### Organization Policies vs IAM Policies

| | Org Policy | IAM Policy |
|---|---|---|
| **Focus** | WHAT — what can be done with a resource | WHO — who can act on a resource |
| **Controls** | Service configuration & resource behavior | Principal permissions |
| **Example** | Disable external IP on all VMs | Allow Alice to create VMs |
| **Constraint types** | **List** (allow/deny from a list) or **Boolean** (on/off) | Role bindings with optional conditions |

**Org Policy Constraint Examples:**

| Service | Constraint |
|---|---|
| Compute | `compute.disableNestedVirtualization` |
| Compute | `compute.vmExternalIpAccess` |
| Compute | `compute.disableSerialPortAccess` |
| IAM | `iam.disableServiceAccountCreation` |
| IAM | `iam.disableServiceAccountKeyCreation` |

---

### Policy Intelligence Suite

| Tool | Purpose |
|---|---|
| **Policy Troubleshooter** | Why does/doesn't user X have access to resource Y? Needs `roles/iam.securityReviewer` for full results |
| **Policy Analyzer** | Which principals have what access to which resources? |
| **Recommender** | ML-based; identifies unused permissions (90-day window); suggests revoke/replace/add |
| **Policy Simulator** | Test IAM policy changes against 90 days of real access logs before applying |

**Recommender logic:**
- Evaluates only project-level role grants in place for ≥ 90 days
- Suggests revoking roles unused in the past 90 days
- Three recommendation types: **Revoke**, **Replace**, **Add permissions to existing role**
- Recommendations are **not applied automatically** — you must approve them

---

### IAM Best Practices

1. **Principle of Least Privilege** — always grant the minimum permissions needed
2. **Use groups** — assign roles to groups, not individual users
3. **Prefer predefined roles** over basic roles; basic roles are coarse-grained
4. **Custom roles** require manual maintenance — use when predefined is not available
5. **Use Policy Intelligence** — Troubleshooter, Analyzer, Recommender, Simulator
6. **Parent policy overrides child** — be careful when setting org-level policies
7. Use **Workload Identity Federation** instead of service account keys for external workloads

---

## Module 4: Configuring VPC for Isolation and Security

### VPC Firewall Rules

- **Stateful** — return traffic for allowed connections is automatically permitted
- Defined at **VPC network level**, enforced at **per-instance level**
- VPC = distributed firewall (rules apply even between instances in the same network)

**Firewall Rule Parameters:**

| Parameter | Values |
|---|---|
| Direction | Ingress or Egress |
| Source/Destination | CIDR ranges, service accounts, tags |
| Protocol/Port | TCP, UDP, ICMP, or specific ports |
| Action | Allow or Deny |
| Priority | 0 (highest) to 65535 (lowest) |

**Implied Rules (always present, lowest priority):**
- Allow all egress (IPv4 + IPv6 if enabled)
- Deny all ingress (IPv4 + IPv6 if enabled)

**Default VPC extra rules:**
- `default-allow-internal` — all protocols between instances in the VPC
- `default-allow-ssh` — port 22
- `default-allow-rdp` — port 3389
- `default-allow-icmp` — ICMP

> ⚠️ In production, **delete the default VPC** and create custom-mode VPCs with explicit rules.

---

### Firewall Naming Convention (Best Practice)

```
{direction}-{allow/deny}-{service}-{to/from}-{location}

Examples:
  ingress-allow-ssh-from-onprem
  egress-allow-all-to-gcevms
  ingress-deny-rdp-from-internet
```

---

### Firewall Policy Hierarchy

```
Organization → Hierarchical Firewall Policies (apply to many VPCs across projects)
  └── Folder → Hierarchical Firewall Policies
        └── Project → VPC
              ├── Global Network Firewall Policies (all regions in a VPC)
              ├── Regional Network Firewall Policies (single region)
              └── VPC Firewall Rules (legacy; per-VPC)
                    └── Built-in Implied Rules (lowest priority)
```

**Firewall Insights** (part of Network Intelligence Center): analyzes firewall rule usage, identifies misconfigs, and surfaces overly permissive rules via Cloud Monitoring metrics + Recommender.

---

### SSL/TLS Policies for Load Balancers

Applied to Application Load Balancers and Proxy Network Load Balancers.

| Profile | Description |
|---|---|
| **COMPATIBLE** | Broadest client support; includes older SSL/TLS |
| **MODERN** | Wide SSL/TLS features for modern clients |
| **RESTRICTED** | Reduced feature set for strict compliance |
| **CUSTOM** | You select exact SSL features; you manage updates |

> If no SSL policy is set, the **COMPATIBLE** profile is applied by default.

---

### VPC Connectivity Options

| Option | Use Case |
|---|---|
| **VPC Peering** | Connect two non-overlapping VPCs (same or different projects/orgs); private RFC1918 space; not transitive |
| **Shared VPC** | Centralized network admin; host project shares VPC with service projects; requires Shared VPC Admin role |
| **Cloud VPN** | IPsec encrypted tunnel over internet to on-prem; up to 3 Gbps/tunnel |
| **Cloud Interconnect** | Dedicated (10/100 Gbps) or Partner (50 Mbps–50 Gbps) private connection to on-prem |

**VPC Peering advantages:** no data egress charges for peered traffic; uses private IPs; each side manages their own firewall.

**Shared VPC:** separates network administration from workload management; service project VMs use host project subnets.

---

### VPC Service Controls

Enforce a **security perimeter** around GCP APIs to prevent data exfiltration.

**Key capabilities:**
- Extend VPC perimeter to restrict GCP service APIs (Cloud Storage, BigQuery, etc.)
- Block data from leaving the perimeter even if attacker has valid credentials
- Supports hybrid environments (on-prem access via VPN/Interconnect + access levels)
- Configure with: Console, `gcloud`, Access Context Manager API

**Perimeter types:**
- **Regular perimeter** — enforce restrictions on services within the perimeter
- **Bridge perimeter** — allow communication between resources in different perimeters

**Ingress/Egress rules:** control which identities/services can communicate across the perimeter boundary.

**VPC Accessible Services:** when enabled, access from network endpoints inside the perimeter is limited to specified services.

---

### Private Google API Access

Allows VMs without external IPs to reach Google APIs privately.

Options:
- **Private Google Access** — VMs in a subnet can reach Google APIs via internal IPs
- **Private Service Connect** — create private endpoints for Google services in your VPC

Services accessible via Private Google Access include: BigQuery, Cloud Storage, Dataproc, Datastore, Pub/Sub, and many others.

---

### Access Context Manager

Extends VPC Service Controls with **attribute-based access policies**.

- Define **access levels** based on: IP ranges, device policy (OS version, screen lock), user identity
- Uses **Endpoint Verification** to gather device info
- Reduces the size of your trusted perimeter — allows context-aware access
- Integrates with BeyondCorp Enterprise

---

### VPC Flow Logs

- Captures network flow samples for **VM-to-VM, VM-to-internet, and VM-to-GCP-services** traffic
- Enabled per subnet
- Logs stored in **Cloud Logging** → can export to BigQuery, Pub/Sub, Cloud Storage
- Use cases: network forensics, security monitoring, bandwidth analysis, compliance

> Flow logs capture **sampled flows**, not every packet. Default sampling rate is 0.5 (50%).

---

### Cloud IDS (Intrusion Detection System)

- **Network-based threat detection** using Google + Palo Alto Networks threat intelligence
- Uses **Packet Mirroring** — IDS endpoint mirrors traffic from monitored VMs
- Detects: malware, spyware, C2 attacks, port scans, network exploits
- Two components: **IDS Endpoint** (receives mirrored traffic) + **IDS Packet Mirroring Policy** (what traffic to mirror)
- Alerts viewable in Security Command Center and Cloud Logging

---

## 🎯 Scenario Breakdown

### Scenario: Enforcing Least Privilege at Scale

**Problem:** Developers have Editor role on production projects — too broad.
**Solution:**
1. Replace `roles/editor` with specific predefined roles (e.g., `roles/storage.objectViewer`, `roles/bigquery.dataViewer`)
2. Run **Recommender** to identify unused permissions over 90 days
3. Use **Policy Simulator** to validate impact before applying changes
4. Use **Policy Analyzer** to audit "who can access what" across the org
5. Enforce `iam.disableServiceAccountKeyCreation` Org Policy + move to Workload Identity Federation

---

### Scenario: Data Exfiltration Prevention

**Problem:** A compromised credential could exfiltrate data from Cloud Storage to an external bucket.
**Solution:**
1. Create a **VPC Service Perimeter** around Cloud Storage + BigQuery
2. Block `storage.objects.copy` to destinations outside the perimeter
3. Use **Access Context Manager** to require corporate IP range or managed device for access
4. Enable **VPC Flow Logs** to detect anomalous traffic patterns
5. Deploy **Cloud IDS** for network-level threat detection

---

### Scenario: Connecting On-Prem to GCP Securely

**Problem:** On-prem workloads need to call GCP APIs without exposing data to the internet.
**Solution:**
1. Establish **Cloud Interconnect** (Dedicated or Partner) for private connectivity
2. Enable **Private Google Access** on relevant subnets so VMs without external IPs can reach APIs
3. Configure **VPC Service Controls** with hybrid access rules to allow on-prem IP ranges
4. Use **Access Transparency** + **Access Approval** for compliance auditability

---

## 🔑 Key Exam Tips

- **IAM deny policies are evaluated BEFORE allow policies** — deny always wins
- **Parent policy overrides child** in IAM — a less restrictive parent grants more than a restrictive child
- **Org Policy** = controls WHAT resources can do; **IAM Policy** = controls WHO can act on resources
- **Workload Identity Federation** = no service account keys needed for external workloads
- **VPC Service Controls** = perimeter-based control to prevent data exfiltration
- **Recommender** evaluates role grants ≥ 90 days old; unused for 90 days → suggests revocation
- **Policy Simulator** uses 90 days of real access logs to simulate policy changes
- **Shared VPC** requires Shared VPC Admin role at org or folder level
- **VPC Peering** is not transitive — A↔B + B↔C does NOT mean A↔C
- **Firewall rules** are stateful; they apply between instances in the same network too
- **Cloud IDS** uses Packet Mirroring — traffic is mirrored, not inline; detection only, not prevention
- **Access Context Manager** = attribute-based access levels (device state, IP, identity) used with VPC Service Controls
- **Titan chip** = Google's hardware security chip for trusted boot in data centers
- **MACsec** encrypts at the link layer on Interconnect; does NOT encrypt within Google's network

---

*Notes generated from: Managing Security in Google Cloud — 4 Modules: Foundations of GCP Security, Securing Access to GCP, IAM, Configuring VPC for Security*

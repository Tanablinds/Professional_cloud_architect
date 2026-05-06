---
title: Logging and Monitoring in Google Cloud
tags: [GCP, PCA, Observability, CloudMonitoring, CloudLogging, SLO, AlertPolicy, AuditLogs]
modules: "Intro to Observability | Monitoring Critical Systems | Alerting Policies | Advanced Logging | Cloud Audit Logs"
note_type: "🏗️ Service Blueprint"
---

# 🏗️ Logging and Monitoring in Google Cloud

> **GCP Professional Cloud Architect Notes**
> Source: *Logging and Monitoring in Google Cloud — 5 Modules (227 pages)*

---

## Module 1: Introduction to Google Cloud Observability

### Why Observability?

In the cloud you can't physically touch the servers — observability tools are your window into system health.

**Four user needs for observability:**

| Need | What Users Want |
|---|---|
| **Visibility into system health** | "Are my systems functioning? Do they have sufficient resources?" |
| **Error reporting & alerting** | Proactive, automatic detection — avoid connecting dots manually |
| **Efficient troubleshooting** | Correlated signals, search across logs + metrics in one place |
| **Performance improvement** | Optimization guidance backed by data |

---

### Four Golden Signals

> From Google's SRE book: the minimal set of metrics that measure a system's performance and reliability.

| Signal | Definition | Why It Matters | Sample Metrics |
|---|---|---|---|
| **Latency** | How long it takes a system to return a result | Directly affects UX; changes indicate emerging issues | Page load time, query duration, time to first byte |
| **Traffic** | How many requests reach your system | Current demand indicator; drives capacity planning | HTTP requests/sec, active sessions, concurrent connections |
| **Saturation** | How close to capacity a service is | Tied to degrading performance as limits are reached | % CPU/memory/disk, thread pool utilization |
| **Errors** | Events that measure system failures | May indicate config issues, SLO violations, capacity limits | # failed requests, HTTP 4xx/5xx, exceptions |

---

### Observability Signal Flow

```
Capture Signals         Visualize & Analyze       Manage Incidents
────────────────         ──────────────────         ────────────────
Metrics                  Dashboards                 Alerts
Logs                     Metrics Explorer           Error Reporting
Traces                   Log Explorer               SLO / Service Monitoring
                         Log Analytics              Health Checks / Uptime
```

---

### GCP Observability Products

| Product | Purpose |
|---|---|
| **Cloud Monitoring** | Metrics collection, dashboards, uptime checks, alerting |
| **Cloud Logging** | Collect, store, search, analyze, export log data |
| **Error Reporting** | Auto-detect, count, and group application errors |
| **Cloud Trace** | Distributed tracing; latency analysis across services |
| **Cloud Profiler** | CPU + memory profiling via statistical sampling (flame graph) |

> **Metrics scope only affects Cloud Monitoring.** All other tools (Logging, Error Reporting, APM) are strictly project-based.

---

### Cloud Monitoring Features

- **1,500+ free metrics** across 100+ GCP services — no config needed
- Open-source support: **Prometheus** + **OpenTelemetry** for custom metric collection
- **GKE**: Google Managed Prometheus (GMP) for Prometheus-native monitoring
- **Compute Engine**: **Ops Agent** for OS-level metrics + 30+ third-party app plugins

---

### Cloud Logging Overview

| Aspect | Details |
|---|---|
| **Collect** | Cloud events, config changes, customer service logs; all hierarchy levels |
| **Analyze** | Logs Explorer (real-time); Log Analytics (SQL-powered via BigQuery) |
| **Export** | Cloud Storage, Pub/Sub, BigQuery via log sinks |
| **Retain** | Data access + service logs: 30 days default; Admin logs: 400 days |

---

### Error Reporting

- **Real-time processing** — errors displayed within seconds
- Automatically groups related errors by type
- Shows: time chart, occurrence count, affected users, stack trace
- **Instant notification** when a new error type is first detected

---

### Cloud Trace

- Latency data from distributed applications
- Near-real-time latency reports
- Automatically identifies performance regressions and changes

---

### Cloud Profiler

- Statistical sampling — extremely low overhead, runs on production
- Flame graph UI showing call hierarchy + CPU/heap consumption
- Works on: Compute Engine, App Engine, GKE, other clouds + on-prem
- Languages: Java, Go, Python, Node.js

---

## Module 2: Monitoring Critical Systems

### Cloud Monitoring Architecture

```
Collection                 Storage                    Analysis & Visualization
──────────────             ────────────               ────────────────────────
GCP services (auto)        Cloud Monitoring           Dashboards
Ops Agent (Compute)        Storage & API              Metrics Explorer
Prometheus (GKE)                                      Alert Policies
OpenTelemetry (custom)                                Uptime Checks
BindPlane (on-prem/AWS)                               Notifications
```

### Monitoring Architecture Patterns

| Pattern | Solution |
|---|---|
| **Platform monitoring** | Automatic ingestion; system metrics collected by default; no config needed |
| **GKE workloads** | Google Managed Prometheus (GMP); free GKE control plane metrics |
| **Compute Engine** | Ops Agent; supports 30+ plugins; OpenTelemetry-based |
| **Hybrid / multi-cloud** | BindPlane by Blue Medora; imports from AWS, Azure, on-prem |

---

### Monitoring Multiple Projects — Metrics Scope

**Default behavior:** Each project is its own metrics scope.

**Problem:** If your app spans multiple projects, you can't see across them by default.

**Solution: Create a dedicated monitoring project** whose metrics scope includes all resource projects.

```
"big-app" project (metrics scope host)
├── svc1  ← added to scope
├── svc2  ← added to scope
├── svc3  ← added to scope
└── svc4  ← added to scope
```

- A single metrics scope → one set of dashboards + alerting policies across all projects
- Role `roles/monitoring.viewer` on the monitoring project grants access to all scoped data
- **Recommendation:** Use a dedicated project for monitoring config; keep resource projects separate

> ⚠️ Metrics scope only controls Cloud Monitoring. Logging, Error Reporting, APM are per-project.

---

### Cloud Monitoring Data Model

Each **time series** has four fields:

| Field | Content |
|---|---|
| `metric` | metric type + metric labels (what is being measured) |
| `resource` | resource type + resource labels (where it's measured) |
| `metricKind` / `valueType` | DELTA/GAUGE/CUMULATIVE; INT64/DOUBLE/BOOL |
| `points` | Array of timestamped values |

---

### Dashboards & Metrics Explorer

| Tool | When to Use |
|---|---|
| **Auto-created dashboards** | GCP detects resources (GKE, Compute Engine) and auto-creates dashboards |
| **Dashboard Builder** | Build custom dashboards with specific charts |
| **Metrics Explorer** | Ad-hoc exploration; save to dashboard; share via URL |

**Metrics Explorer workflow:**
1. Select **metric** (resource type + metric type)
2. Apply **filters** to reduce noise
3. Set **grouping** to combine multiple time series
4. Set **alignment** — break data into regular time buckets (default: 1 minute)
5. Configure **display** (line chart, stats mode, X-ray mode, compare to past)

**Chart analysis modes:**
- **Standard** — unique color per time series
- **Stats** — shows mean, moving average
- **X-Ray** — translucent gray; overlaps show density bands (useful for many time series)

---

### PromQL for Cloud Monitoring

```promql
# Basic structure
metric_name{label="value"}

# GCP metric naming: dots → underscores, / → :
# compute.googleapis.com/instance/cpu/utilization →
compute_googleapis_com:instance_cpu_utilization{monitored_resource="gce_instance"}
```

**Why use PromQL:**
- Open-source standard; large community + documentation
- Designed for time-series data; rich functions (`rate()`, `increase()`, `avg()`)
- Compatible with Grafana + Prometheus Alertmanager ecosystem

---

### Uptime Checks

- Test availability of **public** services from multiple global locations
- Configure: protocol, resource type, hostname, optional custom headers + auth
- Auto-creates alert policy for failing checks
- Tests connectivity from multiple world regions

---

## Module 3: Alerting Policies

### SLI → SLO → SLA

```
SLI (What you measure)
  └── SLO (Your internal target)
        └── SLA (Your customer commitment)
```

| Term | Definition | Example |
|---|---|---|
| **SLI** (Service Level Indicator) | A quantifiable measure of service behavior | Error rate, latency at p95 |
| **SLO** (Service Level Objective) | SLI + target; typically just under 100% | 99.9% of requests return in < 200ms |
| **SLA** (Service Level Agreement) | Promise to customers + consequences for breach | < 0.3% error rate for billing system |

**SLO must be S.M.A.R.T.:** Specific, Measurable, Achievable, Relevant, Time-bound.

> Alert thresholds should be **substantially higher than SLA minimums** — gives you time to fix before SLA is breached.

---

### Error Budgets

```
Error budget = 100% − SLO target

Example: SLO = 99.9% availability
→ Error budget = 0.1% = ~43 min/month of allowed downtime
```

- Alert when the error budget is **trending toward exhaustion** before the window ends
- Don't alert when the budget is already spent — too late

---

### Alert Evaluation Metrics

| Metric | Definition | Trade-off |
|---|---|---|
| **Precision** | Relevant alerts ÷ (relevant + irrelevant) | Too few false positives but may miss events |
| **Recall** | Relevant alerts ÷ (relevant + missed) | Catching everything but may false-positive |
| **Detection time** | How long to notice an alert condition | Short = fast detection; Long = false positive risk |
| **Reset time** | How long alerts fire after issue resolved | Long = confusion; Short = noise |

---

### Alert Window Strategies

| Window Type | Speed | Precision | Error Budget Consumed |
|---|---|---|---|
| **Short windows** (e.g., 10 min) | Fast detection | Poor (false positives) | Very little |
| **Longer windows** (e.g., 36 hr) | Slower detection | Better | More |
| **Short window + successive count** | Fast spot + anomaly filter | Better | Moderate |
| **Multiple conditions** | Configurable | Best | Depends |

**Recommendation:** Use **multiple conditions** in an alerting policy. Short window detects fast; long window confirms. Use Pub/Sub + Cloud Run for complex logic before paging humans.

---

### Alerting Policy Types

| Type | Source | Example |
|---|---|---|
| **Metric-based** | Cloud Monitoring time-series data | Alert when VM CPU > 80% for 5 min |
| **Log-based** | Specific log message occurrences | Alert when a service account key is accessed by a human |

---

### Alert Trigger Conditions (Metric-based)

| Condition Type | Trigger |
|---|---|
| **Metric threshold** | Values above/below threshold for a duration window |
| **Metric absence** | No measurements received for a duration window |
| **Forecast** | Predicted to violate threshold within a forecast window |

---

### Notification Channels

| Channel | Best For |
|---|---|
| **Email** | Low-priority; informative but can become spam |
| **SMS** | Fast; choose recipient carefully |
| **Slack** | Popular for support teams |
| **PagerDuty** | On-call management + incident response |
| **Webhook / Pub/Sub** | Automation; external systems; custom logic |
| **Google Cloud app** | Mobile push notifications |

**Best practice:** High priority → Slack + SMS + PagerDuty. Low priority → Email or ticket system. Use **severity labels** in policies to automate routing.

---

### Incident States in Alerting Console

| State | Meaning |
|---|---|
| **Firing** | Conditions are currently met; new/unhandled alert |
| **Acknowledged** | Someone is investigating; signals team it's being handled |
| **Closed** | Conditions no longer met |

**Snooze:** Temporarily suppress alerts during outages to reduce notification noise.

---

### Service Monitoring (SLO Tracking)

- Automatically identifies GKE and App Engine services as candidates
- Creates **request-based** or **windows-based** SLOs

| SLO Type | Calculation | Risk |
|---|---|---|
| **Request-based** | Good requests ÷ total requests | Transparent; direct customer impact view |
| **Windows-based** | Good windows ÷ total windows | Can hide burst failures (short spike in a window doesn't fail the window) |

**Burn rate alerting:**
```
Burn rate threshold = 1 → uses 100% of error budget in compliance period
If burn rate > threshold in lookback window → Alert!
```

---

## Module 4: Advanced Logging & Analysis

### Log Types

| Category | Description |
|---|---|
| **Platform logs** | Written by GCP services (e.g., VPC Flow Logs) |
| **Component logs** | From Google-provided software on user systems (e.g., GKE components) |
| **Security logs** | Cloud Audit Logs + Access Transparency logs |
| **User-written logs** | App logs written via Cloud Logging API, client libraries, Ops Agent |
| **Multi/Hybrid cloud logs** | From AWS, Azure, on-prem via BindPlane |

### Auto-collected Logs

| Platform | How Collected |
|---|---|
| **GKE** | Logs to stdout/stderr → collected automatically |
| **Cloud Run / Functions** | stdout/stderr → auto-collected |
| **Compute Engine** | Install **Ops Agent** → collects host metrics + third-party app logs |

---

### Log Routing Architecture

```
All log entries (any source)
       ↓
Cloud Logging API
       ↓
Log Router (evaluates sinks + exclusions)
       ↓
    ┌──────────────────────────────┐
    │  _Required sink              │ → _Required bucket (400d, free, non-configurable)
    │  _Default sink               │ → _Default bucket (30d, standard pricing)
    │  User-defined sinks          │ → Cloud Storage / BigQuery / Pub/Sub / Log bucket
    └──────────────────────────────┘
```

### Log Buckets

| Bucket | Holds | Retention | Cost | Modifiable? |
|---|---|---|---|---|
| `_Required` | Admin Activity, System Event, Access Transparency | 400 days | Free | Cannot delete/modify |
| `_Default` | All other ingested logs | 30 days (configurable) | Standard rates | Cannot delete; can disable |
| **User-defined** | Whatever you route | Configurable | Standard rates | Fully configurable |

---

### Log Sink Destinations

| Destination | Best For |
|---|---|
| **Cloud Logging bucket** | Pre-separate entries; different retention per log type |
| **BigQuery dataset** | SQL-powered analysis of large/complex log datasets |
| **Cloud Storage** | Long-term archival; cost-efficient; lifecycle management |
| **Pub/Sub topic** | Real-time streaming to Dataflow, third-party SIEMs (Splunk), Cloud Run |

**Aggregation levels for sinks:**
- **Project** — logs from one project
- **Folder** — logs from folder + all child projects/subfolders
- **Organization** — logs from entire org + all children (recommended for security analytics)

---

### Log Exclusions

- Write a query in Logs Explorer to select entries to exclude
- Attach query as an **exclusion filter** on a sink
- ⚠️ **Excluded entries are permanently lost** — they are never ingested

---

### Log Export Pipelines

| Destination | Pipeline |
|---|---|
| **Real-time analysis** | Logs → Pub/Sub → Dataflow → BigQuery |
| **Long-term storage** | Logs → Cloud Storage (with lifecycle rules) |
| **Third-party SIEM** | Logs → Pub/Sub → Splunk (via Dataflow or Splunk Add-on) |
| **Security analytics** | Aggregated org sink → Log Analytics / BigQuery / Chronicle |

---

### Logs Explorer Query Tips

**Basic syntax:**
```
[FIELD_NAME] [OP] [VALUE]

resource.type="gce_instance"              # equals
resource.labels.instance_id!="123"        # not equals
severity>=ERROR                           # numeric ordering
textPayload:"GET /check"                  # has (contains)
jsonPayload.error:*                       # presence check
logName =~ ".*apache.*"                   # regex match
```

**Boolean operators** (use ALL CAPS):
```
textPayload:("foo" AND "bar")
textPayload:("foo" AND NOT "bar")
textPayload:("foo" OR "bar")
```

**Performance tips:**
- Filter on **indexed fields first**: `httpRequest.status`, `logName`, `resource.type`, `severity`, `timestamp`
- Restrict by `logName` and time range early
- Use `SEARCH(textPayload, "hello world")` for full-text search (case-insensitive)
- Be specific: `jsonPayload.message="/score called"` is much faster than `textPayload:"/score called"`

---

### Logs-Based Metrics

Transform log entries into **time series data** usable in Cloud Monitoring.

| Metric Type | Description |
|---|---|
| **Counter** | Count matching log entries (e.g., count of 500 errors) |
| **Distribution** | Statistical distribution of extracted values (e.g., latency histogram) |
| **Boolean** | Whether a log entry matches a filter |

**Create flow:**
1. Find relevant logs in Logs Explorer
2. Filter to target entries
3. `Actions → Create Metric`
4. Choose Counter or Distribution
5. Optionally add **labels** for group-by/filter in Monitoring (max 10; cannot delete once created)

**Use cases:** Count errors crossing a threshold; detect latency spikes; visualize trends.

---

### Log Analytics

- Brings BigQuery SQL power **inside Cloud Logging console**
- Create an **analytics-enabled log bucket** — no separate BigQuery export needed
- Can also query the data directly from BigQuery (read-only linked dataset)
- **Permanent** — cannot downgrade an analytics-enabled bucket

```
Log Analytics-enabled bucket → SQL query in Log Analytics UI
                             → BigQuery read-only linked view
```

**Use cases:**
- DevOps: Top requests grouped by response type and severity (reduce MTTR)
- Security: Find all audit logs for a user over the past month
- Network Ops: Identify GKE network issues using VPC + firewall logs

---

## Module 5: Cloud Audit Logs

### The Four Audit Log Types

> "Who did what, where, and when?"

| Log Type | Records | Always On? | Retention | Charge? | View Role Required |
|---|---|---|---|---|---|
| **Admin Activity** | Config/metadata changes by users (create VM, change IAM) | ✅ Yes | 400 days | Free | `Logging/Logs Viewer` |
| **System Event** | Google-initiated config changes (not user-driven) | ✅ Yes | 400 days | Free | `Logging/Logs Viewer` |
| **Data Access** | Reads/writes to metadata or user data | ❌ Off by default (except BigQuery) | 30 days | Charged | `Logging/Private Logs Viewer` |
| **Policy Denied** | Access denied by security policy | ✅ Yes | Standard | Charged (can exclude) | `Logging/Logs Viewer` |

> Access **Transparency logs** = Google staff accessing your data (vs. Cloud Audit Logs = your principals acting on your resources).

---

### Data Access Log Sub-types

| Sub-type | Records |
|---|---|
| **Admin-read** | Reading metadata or configuration |
| **Data-read** | Reading user-provided data |
| **Data-write** | Writing user-provided data |

Data Access logs can be **enabled per service** at org/folder/project level:
```bash
# Step 1: Export current IAM policy
gcloud projects get-iam-policy [project-id] > policy.yaml

# Step 2: Add auditConfigs to policy.yaml
auditConfigs:
- service: run.googleapis.com
  auditLogConfigs:
  - logType: ADMIN_READ
  - logType: DATA_READ
  - logType: DATA_WRITE

# Step 3: Apply updated policy
gcloud projects set-iam-policy [project-id] policy.yaml
```

---

### Audit Log Entry Format

Key fields in every audit log entry:

| Field | What It Identifies |
|---|---|
| `logName` | Audit log type (e.g., `cloudaudit.googleapis.com%2Fdata_access`) |
| `protoPayload` | Distinguishes it as an audit log; contains audit details |
| `protoPayload.authenticationInfo.principalEmail` | Who performed the action |
| `protoPayload.methodName` | What operation was performed |
| `protoPayload.resourceName` | Which resource was affected |
| `protoPayload.serviceName` | Which GCP service |
| `resource.type` | Resource type (e.g., `bigquery_resource`) |
| `timestamp` | When the action occurred |

---

### IAM Roles for Logging & Monitoring

| Role | Permissions |
|---|---|
| `logging.viewer` | View Admin Activity, System Event, Policy Denied logs |
| `logging.privateLogViewer` | View Data Access logs (also includes `logging.viewer`) |
| `logging.configWriter` | Create/update/delete log-based metrics, sinks |
| `monitoring.viewer` | Read time series data in Cloud Monitoring |
| `roles/logging.admin` | Full logging administration |

---

## 🏗️ Architecture Best Practices

### Monitoring Architecture

1. **Create a dedicated monitoring project** for metrics scope when monitoring multiple projects
2. Use **Ops Agent** for Compute Engine (not legacy Stackdriver agent)
3. Use **Google Managed Prometheus** for GKE workloads
4. Use **BindPlane** for hybrid/multi-cloud monitoring

### Alerting Strategy

1. Alert based on **error budget burn rate**, not raw thresholds
2. Use **multiple conditions** (short + long window) for better precision + recall
3. Route alerts by **severity**: critical → SMS + PagerDuty; low → email/ticket
4. Include **documentation/playbooks** in alert policies
5. Only page humans for **critical alerts**

### Logging Architecture

1. Route logs at the **organization level** for centralized security analytics
2. Always enable **versioning** on Cloud Storage buckets holding exported logs
3. Use **Log Analytics-enabled buckets** instead of separate BigQuery exports
4. Use **exclusion filters** carefully — excluded logs are permanently lost
5. Set Data Access log exemptions for high-volume service accounts to control costs

### Audit Log Best Practices

1. **Plan first** — test in a test project before enabling at org level
2. Enable Data Access logs at **organization level** for maximum coverage
3. Use **IaC (Terraform)** to deploy and version-control audit log configuration
4. Enable **CMEK** on log storage buckets for advanced encryption requirements
5. Use **log views** to restrict access to sensitive audit data within a bucket
6. Apply **principle of least privilege** — Data Access logs contain PII

---

## 🎯 Scenario Breakdown

### Scenario: Operational Monitoring Role Matrix

| Role | IAM Grant | Access |
|---|---|---|
| CTO | `resourcemanager.organizationAdmin` | Assign permissions to teams |
| Security team | `logging.viewer` + `logging.privateLogViewer` at org level | All admin + data access logs |
| Dev team | `logging.viewer` + `logging.privateLogViewer` at folder level | Logs for their dev folder only |
| External auditors | `logging.viewer` at org + `bigquery.dataViewer` on exported dataset | Pre-built dashboards + BQ access |

### Scenario: SLO Alerting Design

**Setup:** 99.9% availability SLO over 30 days.

```
Error budget = 0.1% = ~43 min of downtime per 30 days

Alert 1: Short window (10 min) with 3 successive failures
  → Fast detection; filters single-window anomalies

Alert 2: Long window (1 hr) with burn rate > 10x
  → Confirms sustained issue; pages on-call

Notification routing:
  Alert 1 → Pub/Sub → Cloud Run → evaluate → Slack (if confirmed)
  Alert 2 → PagerDuty + SMS (immediate human response)
```

---

## 🔑 Key Exam Tips

- **Four Golden Signals** = Latency, Traffic, Saturation, Errors — memorize all four
- **Metrics scope only affects Cloud Monitoring** — Logging/APM are per-project
- **Recommended pattern**: Dedicated monitoring project whose scope includes all resource projects
- **`_Required` bucket**: Admin Activity + System Event + Access Transparency; 400 days; free; cannot modify
- **`_Default` bucket**: Everything else; 30 days; standard pricing; cannot delete but can disable
- **Log sinks** route copies of log entries to: Cloud Storage, BigQuery, Pub/Sub, or another Log bucket
- **Excluded entries are gone forever** — no recovery possible
- **Data Access audit logs** are off by default (except BigQuery); need explicit enablement
- **Admin Activity logs** = always on, free, 400 days; view with `logging.viewer`
- **Data Access logs** = off by default; charged; view requires `logging.privateLogViewer`
- **Access Transparency** = Google STAFF accessing your data (different from Audit Logs = your principals)
- **Error budget** = 100% − SLO; alert when trending to exhaust budget before window ends
- **Request-based SLOs** = good:total requests; **Windows-based** = good:total windows (can hide burst failures)
- **Log Analytics** = SQL on logs within Cloud Logging console (BigQuery-powered); upgrade is permanent
- **Logs-based metrics**: Counter (count matches), Distribution (statistical buckets), Boolean (filter match)
- **Labels on log-based metrics**: Max 10 user-defined; **cannot be deleted once created**
- **PromQL on GCP**: Dots → underscores, slashes → colons in metric names
- **Ops Agent** = recommended agent for Compute Engine (supports 30+ third-party plugins, OpenTelemetry)
- **Google Managed Prometheus** = recommended for GKE workloads
- **BindPlane** = for hybrid/multi-cloud monitoring (AWS, Azure, on-prem) → imports as custom metrics

---

*Notes generated from: Logging and Monitoring in Google Cloud — Introduction to Observability, Monitoring Critical Systems, Alerting Policies, Advanced Logging & Analysis, Cloud Audit Logs*

---
title: Getting Started with Google Kubernetes Engine
tags: [GCP, PCA, GKE, Kubernetes, Containers, Docker, kubectl]
modules: "Intro to GCP | Containers & Kubernetes | K8s Architecture | K8s Operations"
note_type: "⚖️ Compare & Choose"
---

# ⚖️ Getting Started with Google Kubernetes Engine

> **GCP Professional Cloud Architect Notes**
> Source: *Getting Started with Google Kubernetes Engine — 4 Modules (207 pages)*

---

## Module 1: GCP Compute Options — Compare & Choose

### The Five GCP Compute Services

| Service | Type | Abstraction Level | Best For |
|---|---|---|---|
| **Compute Engine** | IaaS | Full VM control | Lift-and-shift, OS customization, legacy apps |
| **Google Kubernetes Engine (GKE)** | CaaS | Container orchestration | Microservices, containerized apps at scale |
| **App Engine** | PaaS | Code + libraries | Web apps, REST APIs, mobile backends |
| **Cloud Run** | Serverless containers | Stateless HTTP containers | Event-driven, auto-scaling to zero |
| **Cloud Run Functions** | FaaS | Single-purpose functions | Event triggers, lightweight integrations |

---

### Compute Engine (IaaS)

- **Up to** 416 vCPUs, 12+ TB memory per VM
- Block storage: Persistent Disks (up to 257 TB, snapshots) or Local SSDs (high IOPS)
- Managed Instance Groups (MIGs) for autoscaling
- **Use when:** Other options don't fit; need OS-level control; lift-and-shift; mix of OS types

---

### GKE (CaaS)

- Managed Kubernetes service; runs containerized applications
- Container = code packaged with all dependencies
- **Use when:** Microservices architecture; need container orchestration at scale

---

### App Engine (PaaS)

- Binds code to libraries; no infrastructure management
- Supported: Java, Node.js, Python, PHP, C#, .NET, Ruby, Go + containers
- Integrated with Cloud Monitoring, Cloud Logging, Profiler, Error Reporting
- Supports version control and traffic splitting
- **Use when:** Focus on code, not infra; websites, mobile backends, REST APIs

---

### Cloud Run (Serverless)

- Stateless containers triggered by HTTP/Pub/Sub; built on **Knative**
- Scales to zero; charges per 100ms
- Portability: runs on GCP, GKE, or anywhere Knative runs
- **Use when:** Stateless workloads, unpredictable traffic, event-driven processing

---

### Cloud Run Functions (FaaS)

- Lightweight, event-based single-purpose functions
- Triggers: Cloud Storage upload, Pub/Sub message, HTTP
- Languages: Node.js, Python, Go, Java, .NET, Ruby, PHP
- **Use when:** Microservice integration, IoT backends, third-party API integration, AI/ML pipelines

---

## Module 2: Containers & Kubernetes

### Evolution: Physical → VM → Container

```
Physical Machine          Virtual Machine           Container
─────────────────         ──────────────────        ──────────────────
App + OS + Hardware       App + OS + Hypervisor     App + Runtime (no OS copy)
• Wasted resources        • Better utilization      • Lightweight (no OS boot)
• Slow to deploy          • Faster than physical    • Starts in seconds
• Hard to scale           • Still bundles full OS   • Shares host OS kernel
```

**Why VMs still have problems:**
- Each app still carries its own full OS kernel copy
- One VM per app at scale = wasteful (hundreds/thousands of VMs)
- Dependency conflicts between apps sharing a VM
- Moving VMs across hypervisor products is complex

**Containers solve this by virtualizing only the user space** (code + dependencies), not the entire OS:
- No OS boot time
- No duplicate kernel
- Lightweight — use a fraction of VM memory
- Portable — "works on my machine" becomes "works everywhere"

---

### VM vs Container — Key Difference

| | VM | Container |
|---|---|---|
| **Virtualizes** | Entire machine (hardware to OS) | Software layers above OS kernel |
| **Size** | GBs (full OS) | MBs (just app + dependencies) |
| **Startup** | Minutes (OS boot) | Seconds (process start) |
| **Isolation** | Full kernel isolation | User-space isolation (shared kernel) |
| **Portability** | Limited (hypervisor-dependent) | High (runs anywhere with container runtime) |
| **Use case** | Full OS control, legacy apps | Microservices, cloud-native apps |

---

### Container Images

- **Image** = application + all dependencies (read-only layers)
- **Container** = running instance of an image
- Each running container adds a thin **writable, ephemeral layer** on top of the image
- Multiple containers can share the same base image — only the diff is stored

**Dockerfile layer order (best practice: least-changed → most-changed):**
```dockerfile
FROM ubuntu:18.04        # Base layer (rarely changes)
COPY ./app               # Add source files
RUN make/app             # Build step
CMD python/app/app.py    # Entrypoint (changes most often)
```

**Multi-stage build** (production best practice):
- Stage 1: Build container (with build tools)
- Stage 2: Runtime container (only what's needed to run)
- This reduces attack surface and image size

**Key rules:**
- Container layer is **ephemeral** — data written to it is lost when container stops
- Permanent data must be stored externally (Cloud Storage, Cloud SQL, Persistent Disk, etc.)
- Each layer is read-only; only the top container layer is writable

---

### Container Registries

| Option | Description |
|---|---|
| **Artifact Registry** (recommended) | GCP-native; private repos; IAM-integrated; supports Docker, Maven, npm, PyPI |
| **Docker Hub** | Public registry; many open-source base images |
| **pkg.dev** | Google-maintained public open-source images |
| **Cloud Build** | Managed build service; pulls from Cloud Source Repos, GitHub, Bitbucket |

**Cloud Build steps:** fetch deps → compile → run integration tests → build image → push to Artifact Registry → deploy to GKE/Cloud Run/App Engine

---

### Kubernetes Overview

> Kubernetes is an **open-source platform** for managing containerized workloads and services across a cluster of nodes.

**Core capabilities:**

| Feature | Description |
|---|---|
| **Declarative config** | Describe desired state in YAML; K8s maintains it continuously |
| **Imperative config** | Issue commands for quick fixes; not recommended for production |
| **Autoscaling** | Scale containers in/out based on resource utilization |
| **Self-healing** | Replace failed containers; reschedule on healthy nodes |
| **Resource controls** | Set requests + limits per workload |
| **Stateless + stateful** | Supports web servers AND stateful apps + batch jobs |
| **Extensibility** | Rich plugin/addon ecosystem |

**Declarative vs Imperative:**

| | Declarative | Imperative |
|---|---|---|
| **How** | Write YAML describing desired state | Run `kubectl` commands to change state |
| **When** | Standard; version-controlled; production | Quick temporary fixes; learning |
| **Risk** | Lower (self-documenting) | Higher (manual, harder to audit) |

---

### Microservices and Containers

| Architecture | Description | Pros | Cons |
|---|---|---|---|
| **Monolithic** | Single codebase, single database | Simple initially | Hard to scale parts independently |
| **Microservices** | Multiple independent services, each with own data | Independent deploy, scale, failure isolation | More complexity, distributed system challenges |

Containers enable microservices by letting each service be **deployed, scaled, and updated independently**.

---

## Module 3: Kubernetes Architecture

### Cluster Structure

```
Cluster
├── Control Plane (GKE-managed in GKE)
│   ├── kube-apiserver       → Only entry point; all commands go through here
│   ├── etcd                 → Cluster database (state, config, all objects)
│   ├── kube-scheduler       → Assigns Pods to nodes based on constraints
│   ├── kube-controller-manager → Watches state; runs control loops
│   └── kube-cloud-manager   → Interfaces with GCP APIs (LBs, storage, etc.)
│
└── Nodes (Worker VMs)
    ├── kubelet              → Agent on each node; executes Pod specs
    ├── kube-proxy           → Network rules; routes traffic to Pods
    └── containerd           → Container runtime (Docker's runtime component)
```

---

### Control Plane Components

| Component | Role | Key Facts |
|---|---|---|
| **kube-apiserver** | Single point of truth; accepts all commands | Only component you interact with directly |
| **etcd** | Key-value store; cluster state | Never interact with directly |
| **kube-scheduler** | Assigns Pods to nodes | Considers hardware, affinity, anti-affinity, resource needs |
| **kube-controller-manager** | Runs control loops (watch loops) | Reconciles desired vs actual state |
| **kube-cloud-manager** | Cloud-provider integration | Provisions GCP load balancers, persistent disks, etc. |

**How kube-scheduler selects a node:**
- Knows state of all nodes
- Evaluates: hardware constraints, software policies, affinity/anti-affinity rules
- Writes chosen node name into the Pod object; doesn't launch Pods directly

---

### GKE vs DIY Kubernetes

| Dimension | DIY Kubernetes | GKE |
|---|---|---|
| **Control plane** | Manual management, patching, scaling, HA | Fully Google-managed; auto-patched, SLA-backed |
| **Node management** | You manage all worker VMs | Autopilot: Google manages; Standard: you manage |
| **Multi-cluster** | Custom federation tooling needed | Fleets for centralized management (GCP, AWS, Azure, on-prem) |
| **Networking** | Manual ingress controller setup | Cloud Load Balancing + Cloud Service Mesh built-in |
| **Security** | Manual hardening + custom policy enforcement | Built-in security features + Policy Controller |
| **Observability** | Manual 3rd-party logging/monitoring setup | Native Cloud Logging + Cloud Monitoring integration |
| **Cost model** | Pay for all provisioned infra | Autopilot: pay per Pod; Standard: pay per node |

---

### GKE Autopilot vs Standard Mode

| | Autopilot (Recommended) | Standard |
|---|---|---|
| **Node management** | Google manages nodes, config, autoscaling | You manage nodes, sizing, upgrades |
| **Cluster optimization** | Google optimizes based on workloads | You control machine types and sizing |
| **Security posture** | Nodes locked down; SSH/privilege escalation removed | You configure node security |
| **Cost model** | Pay per Pod (not node) | Pay for all provisioned nodes |
| **Config control** | More restrictive | Full control |
| **SSH access** | ❌ Not available | ✅ Available |
| **Node affinity** | Limited | Full support |
| **Deployment speed** | Faster (no cluster config) | Slower (requires manual config) |
| **Use when** | Production, standard workloads, want managed ops | Specific node config needed; SSH; legacy workloads |

---

### Kubernetes Objects

**Every K8s resource is an "object" with two elements:**
- **Spec**: Desired state (you define)
- **Status**: Current state (K8s reports)

K8s continuously reconciles Status → Spec via **watch loops** (controllers).

---

### Key Object Types

#### Pod — Smallest Deployable Unit

```
Pod
├── Shared IP address (unique per Pod)
├── Shared network namespace (containers talk via localhost)
├── Shared storage volumes
└── One or more containers (tightly coupled)
```

- Pods are **ephemeral** — created and destroyed as needed
- Pod IP addresses are **temporary** — change on restart/reschedule
- One container per Pod is typical; multi-container Pods for tightly coupled sidecars

#### Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:latest
```

- Describes desired state of Pods (how many replicas, which image)
- Creates and manages a **ReplicaSet** to maintain replica count
- Deployment controller continuously monitors and reconciles

**3 ways to create a Deployment:**

| Method | Command | Style |
|---|---|---|
| YAML + apply | `kubectl apply -f deployment.yaml` | Declarative (recommended) |
| CLI | `kubectl create deployment NAME --image=IMG --replicas=3` | Imperative |
| Console | GKE Workloads → Create Deployment | GUI |

> 💡 Always save YAML files in version control (Cloud Source Repositories, GitHub)

#### Service — Stable Networking Layer

**Why Services exist:** Pod IPs are ephemeral. Services provide a stable virtual IP and DNS name.

```
Service
└── Virtual IP (static, durable)
    └── Endpoints (dynamic list of matching Pod IPs)
        └── Pods with matching labels
```

**Three Service types:**

| Type | Accessible From | Use Case |
|---|---|---|
| **ClusterIP** | Inside the cluster only | Internal service-to-service communication |
| **NodePort** | Outside cluster via node IP + port | Dev/testing; low-level external access |
| **LoadBalancer** | Internet (via GCP Passthrough Network LB) | Production external-facing services |

```bash
# Expose a Deployment as a LoadBalancer
kubectl expose deployment my-app \
  --name=my-service \
  --type=LoadBalancer \
  --port=80 \
  --target-port=8080
```

```yaml
# LoadBalancer Service YAML
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  type: LoadBalancer
  ports:
  - port: 80
    targetPort: 8080
  selector:
    app: my-app
```

#### Labels — Organizing & Selecting Objects

- Key-value pairs attached to objects (during or after creation)
- Used to select and filter objects with `kubectl` and in YAML selectors
- Example: `app: nginx`, `env: prod`, `tier: frontend`

---

## Module 4: Kubernetes Operations (kubectl)

### kubectl Overview

```
kubectl → kube-apiserver (HTTPS) → K8s cluster state
```

- Transforms CLI commands into Kubernetes API calls
- Handles authentication + authorization (via credentials in `$HOME/.kube/config`)
- **Cannot** create new clusters or change cluster shape (use `gcloud` for that)

### kubectl Setup

```bash
# Step 1: Get credentials for a GKE cluster (done once per cluster)
gcloud container clusters get-credentials CLUSTER_NAME --region REGION

# This writes to $HOME/.kube/config
# Config contains: list of clusters + credentials per cluster

# View current configuration
kubectl config view
```

### kubectl Command Syntax

```
kubectl [command] [TYPE] [NAME] [flags]

Examples:
kubectl get pods                          # List all Pods
kubectl get pod my-pod                    # Get specific Pod
kubectl get pod my-pod -o=yaml            # Output as YAML
kubectl get pods -o=wide                  # Extended info (node, IP)
kubectl apply -f deployment.yaml          # Apply declarative config
kubectl create deployment NAME --image=IMG # Imperative create
```

---

### Introspection — Debugging Commands

| Command | Purpose | Key Output |
|---|---|---|
| `kubectl get pods` | Quick status check | Pod phase: Pending / Running / Succeeded / Failed / Unknown / CrashLoopBackOff |
| `kubectl describe pod POD_NAME` | Detailed Pod + container info | Labels, node, IP, container state, restart counts, events |
| `kubectl logs POD_NAME` | View stdout/stderr | App logs; error messages |
| `kubectl logs POD_NAME -c CONTAINER` | Logs for specific container in multi-container Pod | — |
| `kubectl exec POD_NAME -- COMMAND` | Run single command in container | File listing, env vars, process list |
| `kubectl exec -it POD_NAME -- /bin/bash` | Interactive shell in container | Full terminal access |

---

### Pod Status Reference

| Status | Meaning |
|---|---|
| **Pending** | K8s accepted Pod; container image not yet pulled/created |
| **Running** | All containers created; at least one running |
| **Succeeded** | All containers completed successfully; won't restart |
| **Failed** | Container terminated with failure; won't restart |
| **Unknown** | Can't determine state (usually network issue) |
| **CrashLoopBackOff** | Container keeps crashing; K8s backing off restart attempts |
| **ImagePullBackOff** | Container image failed to download |

---

### Debugging Workflow

```
1. kubectl get pods              → Check Pod status/phase
2. kubectl describe pod NAME     → Identify events, container states
3. kubectl logs NAME             → Read app logs for errors
4. kubectl exec -it NAME -- sh   → Interactive debug (last resort)
5. Fix root cause in container image → Rebuild + redeploy
```

> ⚠️ Don't install software directly into a running container — changes are ephemeral. Fix the Dockerfile and redeploy.

---

## 🎯 Scenario Breakdown

### Scenario: Choosing the Right Compute Service

| Requirement | Recommended Service |
|---|---|
| Lift-and-shift Windows Server app | **Compute Engine** |
| Microservices with mixed languages + shared cluster | **GKE** |
| Python web app, minimal infra management | **App Engine** |
| Event-triggered image processing (stateless) | **Cloud Run** |
| Respond to Cloud Storage upload events | **Cloud Run Functions** |
| Container workloads needing SSH/custom node config | **GKE Standard** |
| Container workloads, production, want managed ops | **GKE Autopilot** |

---

### Scenario: Debugging a CrashLoopBackOff

```bash
# 1. Identify the failing Pod
kubectl get pods
# NAME          READY   STATUS             RESTARTS
# my-app-xyz    0/1     CrashLoopBackOff   5

# 2. Get detailed events
kubectl describe pod my-app-xyz
# Look for: "Back-off restarting failed container"

# 3. Check logs from last crash
kubectl logs my-app-xyz --previous

# 4. If logs are insufficient, exec into a new copy
kubectl exec -it my-app-xyz -- /bin/bash

# 5. Fix in Dockerfile/source → rebuild → redeploy
kubectl apply -f deployment.yaml
```

---

### Scenario: Zero-Downtime Deployment in GKE

```yaml
# deployment.yaml with 3 replicas
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
  template:
    spec:
      containers:
      - name: my-app
        image: gcr.io/my-project/my-app:v2  # Updated image
```

```bash
# Apply update — K8s rolls out new ReplicaSet gradually
kubectl apply -f deployment.yaml

# Monitor rollout
kubectl rollout status deployment/my-app

# Rollback if needed
kubectl rollout undo deployment/my-app
```

---

## 🔑 Key Exam Tips

- **GKE manages the control plane** — you never touch kube-apiserver, etcd directly
- **Autopilot = pay per Pod** (not node); **Standard = pay per provisioned node** regardless of use
- **Use Autopilot** unless you need SSH, node-level customization, or specific machine types
- **Pods are ephemeral** — Pod IPs change; use **Services** for stable endpoints
- **Services provide stable virtual IPs** — ClusterIP (internal), NodePort (external via node), LoadBalancer (GCP LB)
- **Declarative > Imperative** — always prefer YAML + `kubectl apply`; store in version control
- **Container writable layer is ephemeral** — data is lost when container stops; use external storage
- **etcd** = cluster state store; you never interact with it directly
- **kube-scheduler** writes the node name into the Pod object; doesn't launch Pods
- **kube-cloud-manager** is GKE's integration with GCP APIs (LBs, disks, etc.)
- **kubectl** cannot create or resize clusters — use `gcloud` or GCP Console for that
- `CrashLoopBackOff` = container keeps crashing; check `kubectl logs --previous`
- `ImagePullBackOff` = container image failed to download (check image name/tag/registry access)
- **Multi-stage Docker builds** = build stage + runtime stage; reduces attack surface + image size
- **Artifact Registry** is the recommended container registry for GCP (replaces Container Registry)
- **Cloud Build** integrates with Cloud Source Repos, GitHub, Bitbucket for CI/CD pipelines
- **Labels** are key-value tags for selecting and organizing K8s objects
- **ReplicaSet** is managed by a Deployment controller — don't create them directly

---

*Notes generated from: Getting Started with Google Kubernetes Engine — Introduction to GCP, Introduction to Containers & Kubernetes, Kubernetes Architecture, Kubernetes Operations*

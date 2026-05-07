---
title: Getting Started with Terraform for Google Cloud
tags: [GCP, PCA, Terraform, IaC, HCL, Modules, State]
modules: "Intro to IaC | Terms & Concepts | Writing Infrastructure Code | Modules | Terraform State"
note_type: "🏗️ Service Blueprint"
---

# 🏗️ Getting Started with Terraform for Google Cloud

> **GCP Professional Cloud Architect Notes**
> Source: *Getting Started with Terraform for Google Cloud — 5 Modules (233 pages)*

---

## Module 1: Infrastructure as Code (IaC)

### What Is IaC?

Instead of clicking through a UI or SSH-ing into servers to run commands manually, **IaC lets you write code in files to define, provision, and manage infrastructure**.

> Declare the desired end state → the tool figures out how to get there.

### Problems IaC Solves

| Problem | Description |
|---|---|
| **Inability to scale rapidly** | Manual infra can't keep pace with business demand |
| **Operational bottlenecks** | Ops teams blocked on managing infra consistently at scale |
| **Disconnected feedback loops** | Dev and Ops can't audit or collaborate on infra changes |
| **High manual errors** | Increased scale = more human error |

### IaC Benefits

| Benefit | What It Means |
|---|---|
| **Declarative** | Specify desired end state, not step-by-step commands |
| **Code-managed** | Commit, version, branch, review infra just like app code |
| **Auditable** | Compare desired vs. current state; comments explain "why" |
| **Portable** | Build reusable modules; share templates across teams |

### Declarative vs. Imperative

| | Declarative | Imperative |
|---|---|---|
| **Approach** | "I should have 5 servers" | "Give me 5 servers" |
| **Tool decides** | What changes are needed | Nothing — you specify exactly |
| **Idempotent** | ✅ Runs again, only creates what's missing | ❌ Runs again, creates 5 more servers |
| **Example** | Terraform HCL / YAML | Shell scripts, `gcloud` commands |

### IaC vs. Configuration Management

| | IaC (e.g., Terraform) | Config Management (e.g., Ansible, Chef) |
|---|---|---|
| **Manages** | Cloud resource provisioning | VM/OS-level configuration |
| **Example** | Launch a GKE cluster | Deploy containers into the cluster |
| **Manipulates** | Cloud provider APIs | Package installs, services, configs inside VMs |

---

## Module 2: Terraform Terms & Concepts

### What Is Terraform?

Terraform is an IaC tool by **HashiCorp** that provisions GCP resources using **declarative configuration files** written in **HCL (HashiCorp Configuration Language)**.

- Multi-cloud + multi-API (GCP, AWS, Azure, GitHub, Kubernetes, etc.)
- Focuses on **provisioning** infrastructure, not configuring it
- Community: thousands of reusable modules in the Terraform Registry

### Terraform Editions

| Edition | Deployment | Concurrent | Cost | Interface |
|---|---|---|---|---|
| **Community (CE)** | Local / VM | ❌ | Free | CLI only |
| **Cloud** | SaaS (managed) | ✅ | License fee | GUI + CLI |
| **Enterprise** | Self-hosted (private) | ✅ | Infra + license | GUI + CLI |

### Terraform Workflow — 5 Phases

```
1. Scope    → Confirm which resources are needed
2. Author   → Write .tf files (HCL)
3. Init     → terraform init  — download provider plugins
4. Plan     → terraform plan  — preview changes
5. Apply    → terraform apply — create real infrastructure
```

Optional phase between Plan and Apply:
```
Validate  → gcloud beta terraform vet  — check against org policies
```

---

### HCL Syntax Reference

```hcl
<BLOCK_TYPE> "<LABEL>" "<LABEL>" {
  # Block body
  <IDENTIFIER> = <EXPRESSION>  # Argument
}
```

| Element | Description | Example |
|---|---|---|
| **Block** | Lines of code belonging to a type | `resource`, `variable`, `output` |
| **Arguments** | Assign values within a block | `name = "mynetwork"` |
| **Identifiers** | Names of arguments, blocks, constructs | `name`, `location` |
| **Expressions** | Values assigned to identifiers | `"US"`, `var.region`, `google_compute_network.net.id` |
| **Comments** | `#` single-line, `//` single-line, `/* */` multi-line | |

> HCL is **order-independent** — block order doesn't matter; Terraform determines execution order from dependencies.

---

### Core File Structure

```
project/
├── main.tf          ← Resources (required)
├── providers.tf     ← Provider + version declarations
├── variables.tf     ← Input variable declarations
├── outputs.tf       ← Output value declarations
├── terraform.tfvars ← Variable values (not in version control)
└── terraform.tfstate← State file (local; prefer remote)
```

---

### Resources

```hcl
resource "google_storage_bucket" "example-bucket" {
  name     = "<unique-bucket-name>"  # Required
  location = "US"
}
```

| Rule | Detail |
|---|---|
| Resource type is fixed | `google_storage_bucket` is a provider keyword; cannot be user-defined |
| Resource name must be unique | Per resource type, per module |
| Arguments inside block | All arguments must be inside `{}` |
| Required args must be defined | Missing required args → `terraform plan` fails |

**Reference a resource attribute from another resource:**
```hcl
# Format: resource_type.resource_name.attribute
network = google_compute_network.vpc_network.id
```

---

### Providers

```hcl
# providers.tf
terraform {
  required_providers {
    google = {
      source  = "hashicorp/google"
      version = "4.23.0"  # Pin version to prevent breaking changes
    }
  }
}

provider "google" {
  project = "<project_id>"
  region  = "us-central1"
}
```

- Provider configs belong in the **root module**
- `terraform init` downloads the provider plugin
- If no version pinned, Terraform downloads the latest (risky)

---

### Variables

```hcl
# variables.tf
variable "bucket_region" {
  type        = string
  description = "Specify the bucket region."
  default     = "US"
  sensitive   = false  # Set true for passwords, secrets
}
```

**Variable types:** `string`, `number`, `bool`, complex types (`list`, `map`, `object`)

**Reference a variable:**
```hcl
location = var.bucket_region
```

**Ways to assign values (precedence: last listed wins):**

| Method | Example | When to Use |
|---|---|---|
| `terraform.tfvars` file (recommended) | `bucket_region = "EU"` | Standard; version-controlled |
| `-var` CLI flag | `terraform apply -var="bucket_region=EU"` | Quick one-offs; automation |
| `-var-file` flag | `terraform apply -var-file my-vars.tfvars` | Multiple environments |
| Environment variable | `TF_VAR_bucket_region=EU terraform apply` | Scripts and pipelines |
| CLI prompt | Terraform asks interactively if unset | Fallback |

**Validation:**
```hcl
validation {
  condition     = contains(["STANDARD", "MULTI_REGIONAL", "REGIONAL"], var.storageclass)
  error_message = "Allowed classes: STANDARD, MULTI_REGIONAL, REGIONAL."
}
```

---

### Output Values

```hcl
# outputs.tf
output "bucket_URL" {
  description = "URL of the bucket"
  value       = google_storage_bucket.mybucket.self_link
  sensitive   = false
}
```

```bash
terraform output              # List all outputs
terraform output bucket_URL   # Query a specific output
```

**Uses:** Print computed resource attributes after `apply`; pass attributes between modules.

---

### State

```hcl
# terraform.tfstate (auto-generated JSON)
{
  "version": 4,
  "terraform_version": "1.2.3",
  "resources": [...]
}
```

- Stores the **binding between your config and real GCP resources**
- Used to determine what to create/update/delete on next run
- Default: stored locally as `terraform.tfstate`
- **Never modify state manually** — use `terraform state` commands

---

### Meta-Arguments

| Meta-Argument | Purpose | Example |
|---|---|---|
| `count` | Create N identical resources | `count = 3`; reference with `count.index` |
| `for_each` | Create resources from a set/map | `for_each = toset(["us-central1-a", "europe-west1-b"])` |
| `depends_on` | Explicit dependency | `depends_on = [google_compute_instance.server]` |
| `lifecycle` | Control create/destroy behavior | `prevent_destroy = true`; `create_before_destroy = true` |
| `provider` | Non-default provider | Use a different project or region |

---

### Resource Dependencies

| Type | Description | How Defined |
|---|---|---|
| **Implicit** | Terraform detects from attribute references | `network = google_compute_network.net.name` → auto-detected |
| **Explicit** | Application-level dependency not visible to Terraform | `depends_on = [google_compute_instance.server]` |

Terraform creates a **dependency graph** and executes independent resources in parallel when safe.

---

## Module 3: Terraform Commands

### Core Commands

| Command | Phase | What It Does |
|---|---|---|
| `terraform init` | Initialize | Downloads provider plugins; initializes `.terraform` directory |
| `terraform plan` | Plan | Shows execution plan (+ add, ~ change, - destroy); doesn't apply |
| `terraform apply` | Apply | Executes the plan; creates/updates/destroys resources |
| `terraform destroy` | Teardown | Destroys all resources in the configuration |
| `terraform fmt` | Format | Auto-formats `.tf` files to canonical style |
| `terraform output` | Inspect | Lists all output values |
| `terraform validate` | Syntax check | Validates config syntax (NOT policy compliance) |

### Plan Output Symbols

| Symbol | Meaning |
|---|---|
| `+` | Resource will be **created** |
| `~` | Resource will be **updated in-place** |
| `-` | Resource will be **destroyed** |
| `-/+` | Resource will be **destroyed then recreated** |
| `(known after apply)` | Value computed at apply time |

### Code Conventions (terraform fmt enforces these)

```hcl
# BEFORE fmt (bad)
resource "google_compute_instance" "my-instance" {
count = 2
name = "test"
machine_type="e2-micro"
boot_disk {    # nested block not at bottom
  ...
}
}

# AFTER fmt (good)
resource "google_compute_instance" "my-instance" {
  count        = 2           # meta-args first
  name         = "test"      # 2-space indent; aligned = signs
  machine_type = "e2-micro"

  boot_disk {                # nested blocks at bottom; blank line before
    ...
  }
}
```

---

### Terraform Validator (`gcloud beta terraform vet`)

- Runs **between Plan and Apply** in CI/CD pipelines
- Enforces **organization policies** (e.g., "only deploy to these regions")
- Uses constraint library created by security/governance teams
- Returns fail/pass; blocks deployments that violate constraints
- **Different from `terraform validate`** — that checks syntax only; `vet` checks org policy compliance

```
Author → Init → Plan → [gcloud beta terraform vet] → Apply
```

---

## Module 4: Writing Infrastructure Code

### Variables Best Practices

1. **Parameterize only when necessary** — don't expose variables that won't change
2. **Use `.tfvars` files** — don't use `-var` CLI flags (they're ephemeral, can't be version-controlled)
3. **Give descriptive names** — include units: `ram_size_gb`, not `ram_size`
4. **Use positive booleans** — `enable_external_access`, not `no_external_access`
5. **Always add descriptions** — auto-included in module documentation
6. **Validate with `validation` blocks** — prevent invalid inputs early

### Output Values Best Practices

1. **Output computed/useful info** — not variables you already know (e.g., network ID, URLs, IPs)
2. **Descriptive names + descriptions** — appear in published module docs
3. **Keep all outputs in `outputs.tf`** — one file, consistent location
4. **Mark sensitive outputs** with `sensitive = true` — masks values in plan/apply output

### Terraform Registry & Cloud Foundation Toolkit

| Tool | Description |
|---|---|
| **Terraform Registry** | Public registry of providers + modules; `registry.terraform.io` |
| **Cloud Foundation Toolkit (CFT)** | Google's reusable modules implementing GCP best practices |
| **Cloud Foundation Fabric (CFF)** | End-to-end blueprints for fast prototyping; cloneable as unit |
| **Infrastructure Manager** | Managed GCP service that deploys Terraform configs to GCP; doesn't deploy apps |

---

## Module 5: Terraform Modules

### Why Modules?

Without modules, repeating resource blocks across environments:
- Becomes unmanageable as code grows
- Causes copy-paste errors
- Makes updates require changes in multiple places

**Solution: DRY (Don't Repeat Yourself) — create modules.**

### What Is a Module?

> Any `.tf` file or directory of `.tf` files = a module.

```
project/
├── main.tf              ← Root module (where you run terraform commands)
├── network/             ← Child module
│   ├── main.tf
│   ├── variables.tf
│   └── outputs.tf
└── server/              ← Child module
    ├── main.tf
    ├── variables.tf
    └── outputs.tf
```

- **Root module** = working directory where `terraform apply` runs
- **Child module** = any module called from the root or another module
- `terraform init` must be re-run whenever modules are added/changed

---

### Module Use Cases

| Use Case | When to Use |
|---|---|
| **Modularize code** | Your config is getting unreadably long |
| **Eliminate repetition** | Same resource group deployed multiple times |
| **Standardize resources** | Enforce org conventions (e.g., mandatory labels, fixed MTU) |

### Benefits

- **Readable** — hide complexity behind a clean module call
- **Reusable** — write once, use in dev/staging/production
- **Abstract** — separate concerns into logical units
- **Consistent** — guarantee every instance is configured the same way

---

### Calling a Module

```hcl
# main.tf (root module)
module "web_server" {
  source       = "./server"      # Required: path to module
  network_name = "my-network"    # Pass input variables
}

module "server_network" {
  source = "./network"
}
```

### Module Source Types

| Source Type | Syntax | Notes |
|---|---|---|
| **Local path** | `source = "./server"` | No download needed; `./ ` or `../` prefix required |
| **Terraform Registry** | `source = "terraform-google-modules/vm/google//modules/compute_instance"` | Format: `NAMESPACE/NAME/PROVIDER` |
| **GitHub** | `source = "github.com/terraform-google-modules/terraform-google-vm//modules/compute_instance"` | Full GitHub URL |
| **GCS bucket** | `source = "gcs::https://storage.googleapis.com/..."` | — |

```hcl
# Always pin module versions from Terraform Registry
module "web_server" {
  source  = "terraform-google-modules/vm/google//modules/compute_instance"
  version = "0.0.5"  # Prevents auto-upgrade to potentially breaking versions
}
```

---

### Parameterizing Modules with Variables

Step 1 — Replace hardcoded values in module with `var.name`:
```hcl
# /network/main.tf
resource "google_compute_network" "vpc_network" {
  name = var.network_name   # Was hardcoded "mynetwork"
}
```

Step 2 — Declare variable in module's `variables.tf`:
```hcl
# /network/variables.tf
variable "network_name" {
  type        = string
  description = "Name of the network"
}
```

Step 3 — Pass value when calling the module:
```hcl
# root main.tf
module "dev_network" {
  source       = "./network"
  network_name = "my-network-dev"
}
module "prod_network" {
  source       = "./network"
  network_name = "my-network-prod"
}
```

> ⚠️ You **cannot** pass variable values to modules at runtime via CLI prompts. Values must be passed in the module block.

---

### Passing Attributes Between Modules (Output Values)

```hcl
# /network/outputs.tf
output "network_name" {
  value = google_compute_network.my_network.name
}
```

```hcl
# /server/variables.tf
variable "network_name" {
  type = string
}
```

```hcl
# root main.tf — reference output from network module
module "server_VM1" {
  source       = "./server"
  network_name = module.my_network_1.network_name  # module.<NAME>.<OUTPUT>
}
module "my_network_1" {
  source = "./network"
}
```

### Module Best Practices

1. **Don't over-modularize** — some duplication is fine if it makes code clearer
2. **Parameterize only what users need to change** — standardize the rest
3. **Use local modules** to organize and encapsulate code
4. **Use Terraform Registry + CFT** for complex GCP architecture
5. **Publish and share modules** with your team (internal or public registry)
6. Use `count` / `for_each` for loops instead of custom scripting

---

## Module 6: Terraform State

### What Is State?

Terraform state is a **metadata repository** that maps your configuration to real infrastructure objects.

- Stored in `terraform.tfstate` by default
- Auto-generated on first `terraform apply`
- Used to determine what to **create / update / destroy** on future runs
- Contains all resource attributes (including sensitive data in plaintext)

### State Decision Flow

```
Read config + Read state
→ Resource in state?
    → No:  Create()
    → Yes: Has changes?
        → Yes + Destroy plan: Delete()
        → Yes + Update plan:  Update()
        → No changes:         No-op
→ Output plan
```

### Local vs. Remote State

| | Local State | Remote State (GCS) |
|---|---|---|
| **Default?** | ✅ | Requires configuration |
| **Shared access** | ❌ One developer only | ✅ Team-shared |
| **Locking** | ❌ No locking | ✅ GCS native locking |
| **Confidentiality** | ❌ Plaintext; anyone with access reads secrets | ✅ IAM + encryption |
| **Auto-updates** | Manual | ✅ Auto on plan/apply |
| **When to use** | Solo developer | Team environments (always) |

### Configuring Remote State in Cloud Storage

```hcl
# Step 1: Create the bucket (main.tf)
resource "google_storage_bucket" "tfstate" {
  name          = "bucket-tfstate"
  force_destroy = false
  location      = "US"
  storage_class = "STANDARD"
  versioning {
    enabled = true  # Critical: enables rollback
  }
}
```

```hcl
# Step 2: Configure backend (backend.tf)
terraform {
  backend "gcs" {
    bucket = "bucket-tfstate"
    prefix = "terraform/state"
  }
}
```

```bash
# Apply backend config
terraform init   # Migrates local state to remote
```

### State Best Practices

| Practice | Guidance |
|---|---|
| **Use remote state for teams** | Always use GCS backend; enables locking and versioning |
| **Enable bucket versioning** | Allows rollback to previous state if corrupted |
| **Encrypt state** | Use `GOOGLE_ENCRYPTION_KEY` env var for CSEK on GCS bucket |
| **Don't store secrets** | State stores values in plaintext; use Secret Manager instead |
| **Don't modify state manually** | Use `terraform state` CLI commands instead |
| **Restrict bucket access** | Only build systems and privileged admins should access state bucket |
| **Add to `.gitignore`** | Prevent dev state files from being accidentally committed |

---

## 🎯 Scenario Breakdown

### Scenario: Multi-Environment Deployment (Dev / Staging / Prod)

**Problem:** Same server infra needed in 3 environments; manually copying code causes drift.

**Solution:**
```
modules/
  server/
    main.tf       ← Define compute instance, static IP, disk, service account
    variables.tf  ← Parameterize: name, machine_type, num_instances
    outputs.tf    ← Expose: instance_id, self_link

environments/
  development/main.tf  → module "dev-server" { source = "../../modules/server"; num_vm = 2 }
  staging/main.tf      → module "stag-server" { source = "../../modules/server"; num_vm = 4 }
  production/main.tf   → module "prod-server" { source = "../../modules/server"; num_vm = 6 }
```
All environments call the same module → one change propagates everywhere.

---

### Scenario: Enforcing Org Policies in CI/CD Pipeline

**Problem:** Dev team deploying resources in unauthorized regions.

**Solution:**
1. Security team creates constraint: `gcp.resourceLocations` → allow only `europe-west1`
2. Add `gcloud beta terraform vet` step between `terraform plan` and `terraform apply` in Cloud Build
3. Pipeline fails with policy violation message before any deployment occurs

---

### Scenario: Team Working on Shared Infrastructure

**Problem:** Two engineers running `terraform apply` simultaneously corrupt the state file.

**Solution:**
1. Create GCS bucket for remote state with versioning enabled
2. Configure `backend "gcs"` in `backend.tf`
3. GCS native locking prevents concurrent apply
4. IAM restricts state bucket access to pipeline SA + admins only

---

## 🔑 Key Exam Tips

- **Terraform manages provisioning** (GCP resources); configuration management (inside VMs) is Ansible/Chef/etc.
- **Declarative = desired state**; Terraform reconciles current → desired automatically
- **`terraform init`** must be run first; downloads providers AND initializes modules
- **`terraform plan`** never modifies infrastructure — preview only
- **`terraform fmt`** = auto-format; **`terraform validate`** = syntax check; **`gcloud beta terraform vet`** = policy compliance
- **State file** = source of truth for what Terraform manages; stores all attributes including secrets
- **Never edit `terraform.tfstate` manually** — use `terraform state` commands
- **Remote state on GCS** = locking + versioning + IAM + encryption → required for teams
- **Modules** = primary code reuse mechanism; root module is where you run `terraform` commands
- **`source` in module block is mandatory** — local (`./path`) or remote (Registry, GitHub, GCS)
- **Variables** parameterize configs; use `.tfvars` file (not CLI flags) for values
- **Outputs** expose computed resource attributes; use `outputs.tf` file; mark sensitive ones
- **Implicit dependency** = Terraform detects from attribute references automatically
- **Explicit dependency** = use `depends_on` for app-level dependencies Terraform can't see
- **CFT (Cloud Foundation Toolkit)** = Google's pre-built Terraform modules implementing GCP best practices
- **Infrastructure Manager** = GCP managed service to deploy Terraform configs; doesn't deploy apps
- **Provider version pinning** = prevents breaking changes on `terraform init`

---

*Notes generated from: Getting Started with Terraform for Google Cloud — Introduction to IaC, Terms & Concepts, Writing Infrastructure Code, Organizing with Modules, Introduction to Terraform State*

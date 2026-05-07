# GCP Professional Cloud Architect — Study Notes

Structured study notes for the **Google Professional Cloud Architect (PCA)** certification exam, organized as a [MkDocs](https://www.mkdocs.org/) documentation site with the Material theme.

## Modules

| # | Module | Key Topics |
|---|--------|------------|
| 01 | Cloud Fundamentals & Core Infrastructure | IaaS/PaaS, IAM, VPC, Compute Engine, Storage, GKE, Cloud Run |
| 02 | Essential Core Services | Core GCP services, resource management |
| 03 | Elastic Scaling & Automation | Autoscaling, MIGs, load balancers, preemptible VMs |
| 04 | Network Fundamentals | VPC, subnets, firewall rules, DNS, routing |
| 05 | Network Architecture | Shared VPC, VPC peering, load balancer tiers |
| 06 | Hybrid & Multicloud Connectivity | Cloud VPN, Interconnect, network topologies |
| 07 | Managing Security on GCP | IAM, org policies, VPC Service Controls, encryption |
| 09 | Kubernetes Engine (GKE) | Autopilot vs Standard, workloads, node pools |
| 10 | Terraform on GCP | IaC patterns, state management, modules |
| 11 | Logging & Monitoring | Cloud Monitoring, Cloud Logging, alerting |
| 13 | Reliable Infrastructure | SLOs, DR strategies, deployment patterns |

## Local Preview

```bash
pip install mkdocs mkdocs-material
mkdocs serve
```

Open `http://127.0.0.1:8000` in your browser.

## Deploy to GitHub Pages

```bash
mkdocs gh-deploy
```

Site will be live at: https://Tanablinds.github.io/Professional_cloud_architect/

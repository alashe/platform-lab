# Platform Lab Architecture (Homelab + AWS)

## Purpose

This lab is a career-grade platform/CloudOps project designed to demonstrate:

- Linux administration and Bash workflow
- Docker-based service operations (no Kubernetes)
- Ansible configuration management and compliance
- Terraform infrastructure provisioning (AWS EC2-focused)
- Monitoring/observability (metrics, dashboards, alerts; logs optional)
- Backup and recovery architecture with tested restores

The platform runs a small set of internal services locally on Proxmox and mirrors a target workload on AWS to demonstrate operational parity between homelab and cloud.

---

## Goals and Non-Goals

### Goals
- Operate real services with uptime expectations and measurable signals.
- Use a consistent automation flow across environments:
  - **Terraform → Ansible → Docker → Monitoring → Operations**
- Ensure the monitoring layer is independent of the service host.
- Implement backups with documented restore procedures and periodic restore drills.
- Produce documentation suitable for interview narratives (architecture, runbooks, incident notes).

### Non-Goals (for this phase)
- Kubernetes
- Complex HA/cluster orchestration
- Multi-region AWS design
- Full zero-trust networking or hybrid VPN mesh (can be Phase 2)

---

## System Overview

### Local Platform (Proxmox)
Two Proxmox hosts provide compute for dedicated VMs:

- **Utility VM (Debian + Docker)**
  - Pi-hole (network DNS filtering)
  - Monitoring target service (nginx or small API)

- **Monitoring VM (Debian)**
  - Prometheus (metrics)
  - Grafana (dashboards)
  - Alertmanager (alerts)
  - Optional: Loki + promtail/agent (logs)

- **Proxmox Backup Server VM (PBS)**
  - Backs up Utility VM and Monitoring VM
  - Syncs/copies backups to NAS and cold tier

**Separation requirement:** Monitoring is independent of the Utility VM so service failures do not remove visibility/alerting.

**Mobility requirement:** Monitoring VM must be restorable/migratable between Proxmox hosts (PBS-backed restore is sufficient for this phase).

---

### AWS Platform (Terraform-managed)
AWS resources are provisioned using Terraform:

- VPC + subnet(s) (minimal architecture acceptable for a lab)
- Security Group(s)
- EC2 instance running Docker
- Optional: CloudWatch for logs/metrics

The AWS EC2 instance runs the same monitoring target service as the homelab (same container image and deployment pattern) to demonstrate workload parity.

---

## Functional Services

### 1) Network Service: Pi-hole (Docker)
Role: primary DNS filtering for the home network.

Operational expectations:
- DNS availability and stable performance
- Visibility into query volume, block rate, upstream latency
- Config managed via Ansible (container runtime + host baseline)

### 2) Monitoring Target Service (Docker)
Role: simple service used to generate metrics/logs and practice operations.

Runs in:
- Homelab Utility VM (Docker)
- AWS EC2 instance (Docker)

Operational expectations:
- Health endpoint (`/healthz`)
- Basic metrics (request rate, latency, errors) and logs
- Alerting on availability and error rate

### 3) Configuration Compliance (Ansible)
Ansible enforces baseline configuration across hosts/VMs:

- SSH configuration and hardening
- Packages and updates policy (defined scope)
- Docker runtime installation and configuration
- Firewall rules
- User accounts and permissions
- Service deployment (compose + systemd integration where appropriate)

Compliance is validated by re-running playbooks to correct drift.

---

## Automation Flow

1. **Terraform**
   - Provisions AWS infrastructure (VPC/SG/EC2, optional CloudWatch)
2. **Ansible**
   - Applies baseline configuration and deploys services
3. **Docker**
   - Runs Pi-hole and the monitoring target service
4. **Monitoring**
   - Collects metrics (and logs if enabled), produces dashboards and alerts
5. **Operations**
   - Runbooks + game days validate troubleshooting and recovery procedures

---

## Backup and Recovery

Backup tiers:

1. **VM backups**
   - Utility VM + Monitoring VM backed up to PBS
2. **PBS → NAS**
   - PBS datastore synced/copied to NAS
3. **NAS → Cold tier**
   - Periodic copy/replication to cold storage (offline or secondary NAS)

Operational requirement: backups are not considered complete until a restore drill is performed and documented.

---

## Observability Signals (Minimum)

### Monitoring target service
- Availability (up/down)
- Request rate (RPS)
- Latency (p95)
- Error rate (5xx)
- Resource usage (CPU/memory/disk)
- Container restarts

### Pi-hole
- Query rate
- Blocked vs allowed
- Upstream response latency
- Container/VM resource usage
- DNS availability

---

## Diagrams

### Platform Diagram (Homelab + AWS)

```mermaid
flowchart LR
  subgraph Repo[Git Repo: IaC + Config + Docs]
    TF[Terraform: AWS provisioning]
    ANS[Ansible: baseline + deploy]
    DC[Docker Compose: service defs]
    DOC[Docs: diagram + runbooks + game days]
  end

  subgraph Proxmox[Homelab: Proxmox Cluster (7550 + 7560)]
    UVM[Utility VM (Debian)\nDocker: Pi-hole + Target Service]
    MVM[Monitoring VM (Debian)\nPrometheus + Grafana + Alertmanager\n(+ Loki optional)]
    PBS[PBS VM\nProxmox Backup Server]
  end

  subgraph Storage[Backup Tiers]
    NAS[NAS Tier]
    COLD[Cold Tier (DS207 / offline)]
  end

  subgraph AWS[AWS (Terraform-managed)]
    VPC[VPC + Subnet]
    SG[Security Group]
    EC2[EC2 Instance\nDocker: Target Service]
    CW[CloudWatch (optional)\nlogs/metrics]
  end

  TF --> AWS
  ANS --> UVM
  ANS --> MVM
  ANS --> EC2
  DC --> UVM
  DC --> EC2

  UVM --> MVM
  EC2 --> MVM

  UVM --> PBS
  MVM --> PBS
  PBS --> NAS --> COLD

  SG --> EC2
  VPC --> EC2
  EC2 --> CW

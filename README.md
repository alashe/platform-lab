# platform-lab

A career-grade homelab platform demonstrating CloudOps and platform engineering skills.  
Runs real services, enforces configuration compliance, provisions cloud infrastructure, and implements observability — end to end.

> **Status:** Private working repo. Current build state is Milestone 1 in progress with Milestone 2 underway — `pbs01` is provisioned, and backup-job validation is the main next step. Higher-level sections below describe the intended platform shape unless a milestone note says otherwise.

---

## What This Is

A self-managed platform that mirrors a production engineering workflow:

- Containerized services running locally and in AWS
- Ansible enforcing baseline configuration across all hosts
- Terraform provisioning AWS and Proxmox infrastructure from code
- Prometheus + Grafana + Loki providing unified observability
- Uptime Kuma providing endpoint availability monitoring and a status page
- Pi-hole DNS with automatic failover across two hosts
- A tiered backup architecture with documented recovery procedures
- Proxmox HA with live migration for the Monitoring VM — zero-downtime failover on node failure, backed by NFS shared storage on the NAS

No Kubernetes. Intentionally scoped to what a small ops team actually runs.

---

## Architecture

```
Proxmox Cluster (pve01 + pve02)
├── Utility VM      (pve01)   Docker · Pi-hole (primary DNS) · nginx target
├── Monitoring VM   (pve02)   Prometheus · Grafana · Alertmanager · Loki · Uptime Kuma
│                             HA: live migration to pve01 on node failure (NFS-backed disk)
├── PBS VM          (pve01)   Proxmox Backup Server → NAS → cold storage
└── Automation VM   (pve02)   Ansible execution · Terraform workspace

NAS (nas01 · TrueNAS Scale)   ZFS RAIDZ1 · PBS datastore · shared VM storage pool
HP Elitedesk 800 mini (bare metal)
├── Corosync QDevice          Quorum tie-breaker for 2-node cluster
└── Docker                    Pi-hole (secondary DNS) · nginx target — mirrors Utility VM

Fedora Workstation            Local Ansible + Terraform development

AWS (Terraform-managed)
├── VPC + Security Group
├── EC2 instance              Debian · Docker · nginx (mirrors homelab)
└── IAM instance profile      least-privilege
```

→ Full architecture: [`docs/architecture/platform-lab.md`](docs/architecture/platform-lab.md)

---

## Repository Layout

```
platform-lab/
├── terraform/          Infrastructure as code (Proxmox + AWS)
├── ansible/            Configuration management and service deployment
├── scripts/            Operational utilities
├── docs/
│   ├── architecture/   Design decisions and diagrams
│   ├── operations/     Runbooks and incident scenarios
│   └── setup/          Prerequisites for local and AWS environments
└── .github/workflows/  CI — Terraform plan, Ansible syntax check (Planned)
```

---

## Quick Start

### Prerequisites

- [Local setup](docs/setup/local-prereqs.md) — Ansible, Terraform, Proxmox access
- [AWS setup](docs/setup/aws-prereqs.md) — credentials, S3 backend bucket, key pair

### Provision AWS infrastructure

```bash
cd terraform/environments/aws-dev
terraform init
terraform plan
terraform apply
```

### Run baseline configuration

```bash
cd ansible
ansible-playbook -i inventories/aws playbooks/baseline.yml
ansible-playbook -i inventories/aws playbooks/docker.yml
ansible-playbook -i inventories/aws playbooks/monitoring-target.yml
```

### Validate the stack

```bash
make validate
```

---

## Automation

Planned delivery point: Milestone 5.

| Trigger | Action |
|---|---|
| PR to `main` | `terraform plan` + Ansible syntax check |
| Approval on PR | `terraform apply` via CI |

Workflow files: [`.github/workflows/`](.github/workflows/)

---

## Documentation

| Doc | Description |
|---|---|
| [Architecture overview](docs/architecture/platform-lab.md) | Full platform design |
| [Architecture decisions](docs/architecture/decisions.md) | Why things are built this way |
| [Platform story](docs/platform-story.md) | Resume narrative and interview framing |
| [Monitoring runbook](docs/operations/monitoring-runbook.md) | Alerting, dashboards, triage |
| [Backup & restore](docs/operations/backup-restore.md) | Backup chain, RTO/RPO, restore drills |
| [Incident scenarios](docs/operations/incident-scenarios.md) | Practice exercises |
| [Terraform README](terraform/README.md) | Module and environment reference |
| [Ansible README](ansible/README.md) | Playbook and role reference |

---

## Skills Demonstrated

`Linux` `Bash` `Docker` `Ansible` `Terraform` `AWS EC2` `Prometheus` `Grafana` `Loki` `Uptime Kuma` `Proxmox` `Proxmox HA` `ZFS` `NFS`

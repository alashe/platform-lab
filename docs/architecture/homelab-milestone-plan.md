# Homelab Platform Engineering Milestone Plan

This document is the authoritative milestone plan for the `platform-lab` project.

The platform demonstrates:

- Infrastructure provisioning with Terraform
- Configuration management with Ansible
- Containerized workloads with Docker
- Observability and monitoring
- Backup and recovery architecture
- Infrastructure delivery through CI/CD

## Repository Structure

```text
platform-lab/
├── README.md
├── Makefile
│
├── docs/
│   ├── architecture/
│   │   ├── platform-diagram.md
│   │   ├── decisions.md
│   │   └── homelab-milestone-plan.md
│   │
│   ├── operations/
│   │   ├── backup-restore.md
│   │   ├── monitoring-runbook.md
│   │   ├── pihole-rollout.md
│   │   ├── pihole-cutover.md
│   │   ├── proxmox-baseline.md
│   │   ├── utility-node.md
│   │   └── incident-scenarios.md
│   │
│   ├── setup/
│   │   ├── local-prereqs.md
│   │   └── aws-prereqs.md
│   │
│   └── platform-story.md
│
├── terraform/
│   ├── backend/
│   │   └── backend-notes.md
│   │
│   ├── modules/
│   │   ├── proxmox_vm/
│   │   └── ec2_monitoring_target/
│   │
│   ├── environments/
│   │   ├── homelab/
│   │   └── aws-dev/
│   │
│   └── README.md
│
├── ansible/
│   ├── inventories/
│   │   ├── homelab/
│   │   ├── aws/
│   │   └── mixed/
│   │
│   ├── playbooks/
│   │   ├── baseline.yml
│   │   ├── docker.yml
│   │   ├── monitoring-target.yml
│   │   └── utility-node.yml
│   │
│   ├── roles/
│   │   ├── baseline/
│   │   ├── docker/
│   │   ├── monitoring_target/
│   │   └── utility_node/
│   │
│   ├── group_vars/
│   ├── host_vars/
│   ├── ansible.cfg
│   └── README.md
│
├── scripts/
│   ├── restore-check.sh
│   ├── validate.sh
│   └── tf-plan.sh
│
├── .github/
│   └── workflows/
│       ├── terraform.yml
│       └── ansible-check.yml
│
└── .gitignore
```

## Architectural Invariants

These rules do not change unless explicitly revised:

- Terraform provisions infrastructure where needed:
  - AWS EC2
  - Proxmox VMs
- Ansible configures all host types:
  - bare-metal utility node
  - Proxmox VMs
  - AWS EC2 instances
- The same Ansible roles should be reusable across these targets whenever possible.
- CI/CD remains a single infrastructure delivery pipeline for Terraform-managed environments.

## Makefile Operational Commands

Operational commands are standardized through the repository `Makefile`.

Example targets:

```text
make tf-plan ENV=homelab
make tf-apply ENV=homelab
make tf-plan ENV=aws-dev
make tf-apply ENV=aws-dev
make ansible-baseline INVENTORY=ansible/inventories/homelab
make restore-test
```

# Milestones

## Milestone 0 — Lab Blueprint and Standards

### Objective

Establish architecture documentation, repository structure, and infrastructure conventions.

### Key Work

- Initialize GitHub repository
- Implement the repository layout
- Define hostnames, roles, and IP plan
- Create architecture diagrams
- Establish operational Makefile commands

### Deliverables

- `docs/architecture/platform-diagram.md`
- `docs/architecture/decisions.md`
- `docs/architecture/homelab-milestone-plan.md`
- `docs/setup/local-prereqs.md`
- root `Makefile`
- repository structure implemented with:
  - `terraform/modules/proxmox_vm/`
  - `terraform/modules/ec2_monitoring_target/`
  - `terraform/environments/homelab/`
  - `terraform/environments/aws-dev/`
  - `ansible/inventories/homelab/`
  - `ansible/inventories/aws/`
  - `ansible/inventories/mixed/`

### Done

- Repository structure committed
- Architecture diagram completed
- Node roles and IPs documented
- Makefile operational commands functional

---

## Milestone 1 — Proxmox Baseline (pve1 + pve2)

### Objective

Create a stable virtualization foundation for the homelab platform.

### Key Work

- Patch and update both Proxmox hosts
- Configure SSH access and firewall posture
- Create Debian 12 VM template
- Document Proxmox host operations

### Deliverables

- `docs/operations/proxmox-baseline.md`
- Debian 12 VM template
- consistent administrative access to both hosts

### Done

- Both Proxmox hosts patched and rebooted successfully
- SSH key access configured
- VM template successfully cloned
- Proxmox baseline documented

---

## Milestone 2 — Career-Grade Backups (PBS → NAS → Cold Tier)

### Objective

Implement a multi-tier backup architecture and verify recovery capability.

### Key Work

- Deploy Proxmox Backup Server VM
- Configure scheduled VM backups
- Replicate PBS datastore to NAS
- Implement cold-tier storage copy
- Create restore validation process

### Deliverables

- `docs/operations/backup-restore.md`
- `scripts/restore-check.sh`
- `make restore-test`
- PBS VM
- NAS replication workflow
- cold-tier backup copy process

### Done

- Scheduled VM backups running
- PBS datastore replicated to NAS
- Cold storage copy created
- Restore test verified with `make restore-test`

---

## Milestone 3 — Utility Node Foundation (Debian + Docker)

### Objective

Deploy the utility host responsible for running platform services.

### Key Work

- Install Debian 12 on the utility node
- Install Docker runtime
- Configure baseline host security
- Prepare Ansible inventory for the utility host

### Deliverables

- `docs/operations/utility-node.md`
- `ansible/inventories/homelab/`
- utility host entry in inventory

### Done

- Docker installed
- Utility node boots reliably
- Docker services persist through reboot
- Host configuration documented
- Ansible inventory contains the utility node
- Homelab Vault file exists and is encrypted

---

## Milestone 4 — Terraform Foundations

### Objective

Use Terraform as the provisioning layer for Proxmox VM environments

### Key Work

- Set up Terraform for homelab VM provisioning
- Define provider and version constraints
- Create Terraform structure for local VM builds
- Provision the next homelab VM(s) through Terraform instead of manual creation
- Document local workflow for `terraform fmt`, `terraform validate`, `terraform plan`, `terraform apply`, and `terraform destroy`

### Deliverables

Terraform structure:

    terraform/
    ├── modules/
    ├── environments/
    │   └── homelab/
    └── README.md

Documentation:

- `docs/setup/terraform-prereqs.md`
- `docs/operations/homelab-vm-provisioning.md`

### Done

- Terraform runs cleanly from the repo
- provider and homelab environment scaffolding are in place
- at least one homelab VM is provisioned through Terraform
- basic apply / destroy workflow is documented
  
---

## Milestone 5 — Network Service: Pi-hole Limited Rollout

### Objective

Deploy Pi-hole DNS filtering in a controlled rollout.

### Key Work

- Deploy Pi-hole container
- Configure persistent storage
- Test DNS filtering with limited devices
- Document rollback procedure

### Deliverables

- `docs/operations/pihole-rollout.md`
- `ansible/roles/utility_node/`
- deployment path from Ansible to the utility node

### Done

- Pi-hole container stable
- Limited client rollout successful
- DNS behavior verified
- Rollback procedure documented

---

## Milestone 6 — Monitoring Layer (Proxmox VM)

### Objective

Deploy monitoring infrastructure and collect operational metrics.

### Key Work

- Provision monitoring VM on Proxmox
- Deploy monitoring stack
- Configure dashboards and alerts
- Include monitoring VM in backup scope
- Preserve restore or migration path to the other Proxmox host

### Deliverables

- `docs/operations/monitoring-runbook.md`
- `ansible/roles/monitoring_target/`
- monitoring VM deployment target on Proxmox
- monitoring VM included in PBS backup schedule

### Done

- Grafana dashboards operational
- Alerts configured and validated
- Monitoring VM included in backup schedule
- Monitoring VM restorable or recoverable on the alternate Proxmox node

---

## Milestone 7 — Pi-hole Whole-Home Cutover

### Objective

Promote Pi-hole to the primary DNS resolver for the home network.

### Key Work

- Configure router DHCP
- Validate DNS resolution across devices
- Document rollback process

### Deliverables

- `docs/operations/pihole-cutover.md`

### Done

- Router distributes Pi-hole DNS
- Network devices appear in logs
- DNS behavior verified
- Rollback procedure tested

---

## Milestone 8 — Ansible Automation with Vault

### Objective

Implement configuration management and cross-target service deployment using Ansible. 

### Key Work

- Build Ansible roles and playbooks
- Encrypt secrets with Ansible Vault
- Reuse the same roles across bare metal, Proxmox VMs, and AWS EC2 where appropriate
- Support deployment of the utility stack to bare metal or a Proxmox utility mirror VM
- Support deployment of the monitoring target workload to Proxmox VM or AWS EC2

### Deliverables

- `ansible/inventories/homelab/`
- `ansible/inventories/aws/`
- `ansible/inventories/mixed/`
- `ansible/playbooks/baseline.yml`
- `ansible/playbooks/docker.yml`
- `ansible/playbooks/monitoring-target.yml`
- `ansible/playbooks/utility-node.yml`
- Vault secrets under `ansible/group_vars/` or inventory-specific group vars
- `make ansible-baseline`

### Done

- Ansible deploys baseline configuration
- Playbooks are idempotent
- Secrets encrypted with Vault
- Same Ansible roles can target bare metal, Proxmox VM, and AWS EC2 where intended
- Utility stack deployable to bare metal or Proxmox VM by inventory/host selection

---

## Milestone 9 — AWS Infrastructure Mirror (Terraform)

### Objective

Use Terraform as the provisioning layer for AWS environment, and deploy workloads using Ansible.

### Key Work

- Create Terraform modules for Proxmox VM and AWS EC2 provisioning
- Define Terraform environment configuration for `homelab` and `aws-dev`
- Document Terraform backend strategy
- Provision Proxmox VMs through Terraform where appropriate
- Provision AWS EC2 infrastructure through Terraform
- Run Ansible configuration after provisioning

### Deliverables

- `terraform/modules/proxmox_vm/`
- `terraform/modules/ec2_monitoring_target/`
- `terraform/environments/homelab/`
- `terraform/environments/aws-dev/`
- `terraform/backend/backend-notes.md`
- `scripts/tf-plan.sh`
- `make tf-plan ENV=homelab`
- `make tf-apply ENV=homelab`
- `make tf-plan ENV=aws-dev`
- `make tf-apply ENV=aws-dev`
- `docs/setup/aws-prereqs.md`

### Done

- Terraform provisions Proxmox VM infrastructure in `homelab`
- Terraform provisions AWS infrastructure in `aws-dev`
- Ansible configures provisioned hosts after infrastructure creation
- Terraform backend strategy documented
- Infrastructure deployable through Makefile commands for both environments

---

## Milestone 10 — CI/CD Infrastructure Delivery Pipeline

### Objective

Deliver Terraform-managed infrastructure changes through a controlled GitHub Actions pipeline.

### Key Work

- Implement GitHub Actions workflows
- Validate Terraform configuration
- Run Terraform plan for repo environments
- Require approval before apply
- Keep a single infrastructure delivery pipeline for Terraform-managed environments
- Optionally run Ansible deployment after infrastructure provisioning

### Deliverables

- `.github/workflows/terraform.yml`
- `.github/workflows/ansible-check.yml`
- pipeline stages:
  - `terraform fmt -check`
  - `terraform validate`
  - `terraform plan`
  - manual approval
  - `terraform apply`
- pipeline logic that supports:
  - `terraform/environments/homelab/`
  - `terraform/environments/aws-dev/`

### Done

- Terraform validation runs automatically
- Terraform plan generated in CI for the intended environment
- Infrastructure changes require approval
- Terraform apply executed through CI pipeline
- Single CI/CD pipeline supports both `homelab` and `aws-dev` Terraform environments

---

## Milestone 11 — Reliability Drills and Platform Documentation

### Objective

Demonstrate operational readiness and produce an interview-ready platform project.

### Key Work

- Conduct operational drills
- Validate backup and restore scenarios
- Document troubleshooting procedures
- Finalize architecture documentation

### Deliverables

- `docs/operations/backup-restore.md`
- `docs/operations/monitoring-runbook.md`
- `docs/operations/incident-scenarios.md`
- `docs/platform-story.md`

### Done

- VM restore from PBS verified
- Monitoring VM recovery validated
- Pi-hole restore validated
- Utility node failure recovery documented
- Repository provides reproducible infrastructure and operational documentation

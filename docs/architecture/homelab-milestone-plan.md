# Homelab Platform Engineering Milestone Plan

This document defines the sequential implementation plan for the
Platform Lab used to demonstrate platform engineering skills.

## Milestones

### Milestone 0 --- Lab Blueprint and Standards

**Objective** Establish architecture documentation, repository
structure, and infrastructure conventions.

**Key Work** - Initialize GitHub repository - Implement repository
layout - Define hostnames, roles, and IP plan - Create architecture
diagrams - Create Makefile operational commands

**Deliverables** - docs/architecture/platform-diagram.md -
docs/architecture/decisions.md - docs/setup/local-prereqs.md

**Done** - Repo structure committed - Architecture diagram created -
Hostnames and IP plan documented

------------------------------------------------------------------------

### Milestone 1 --- Proxmox Baseline

**Objective** Create a stable virtualization platform.

**Key Work** - Patch Proxmox hosts - Configure SSH access - Create
Debian VM template

**Deliverables** - docs/operations/proxmox-baseline.md

**Done** - VM template clones successfully - Hosts patched and
documented

------------------------------------------------------------------------

### Milestone 2 --- Career‑Grade Backups

**Objective** Implement PBS → NAS → Cold storage backup architecture.

**Key Work** - Deploy Proxmox Backup Server - Configure scheduled VM
backups - Replicate backups to NAS - Implement cold tier copies

**Deliverables** - docs/operations/backup-restore.md -
scripts/restore-check.sh

**Done** - Successful restore test via make restore-test

------------------------------------------------------------------------

### Milestone 3 --- Utility Node Foundation

**Objective** Deploy Debian utility node running Docker.

**Key Work** - Install Debian - Install Docker runtime - Prepare Ansible
inventory

**Deliverables** - docs/operations/utility-node.md

**Done** - Docker services persist through reboot

------------------------------------------------------------------------

### Milestone 4 --- Pi‑hole Limited Rollout

**Objective** Deploy DNS filtering safely.

**Key Work** - Deploy Pi‑hole container - Configure volumes - Test with
limited devices

**Deliverables** - docs/operations/pihole-rollout.md

**Done** - Stable DNS filtering for test devices

------------------------------------------------------------------------

### Milestone 5 --- Monitoring Layer

**Objective** Deploy Prometheus/Grafana monitoring VM.

**Key Work** - Create monitoring VM - Configure dashboards and alerts

**Deliverables** - docs/operations/monitoring-runbook.md

**Done** - Alerts validated - VM included in PBS backups

------------------------------------------------------------------------

### Milestone 6 --- Pi‑hole Whole‑Home Cutover

**Objective** Promote Pi‑hole to network DNS.

**Key Work** - Update router DHCP - Validate DNS resolution

**Deliverables** - docs/operations/pihole-cutover.md

**Done** - Home network successfully using Pi‑hole

------------------------------------------------------------------------

### Milestone 7 --- Ansible Automation

**Objective** Implement configuration management.

**Key Work** - Build Ansible roles and playbooks - Implement Vault
secrets

**Deliverables** - ansible/playbooks/\* -
ansible/group_vars/all/vault.yml

**Done** - Idempotent playbooks deploy services

------------------------------------------------------------------------

### Milestone 8 --- AWS Infrastructure Mirror

**Objective** Provision AWS infrastructure with Terraform.

**Key Work** - Create Terraform modules - Deploy EC2 monitoring target -
Run Ansible deployment

**Deliverables** - terraform/modules/ec2_monitoring_target/ -
terraform/backend/backend-notes.md

**Done** - Infrastructure deployable with make tf-plan / make tf-apply

------------------------------------------------------------------------

### Milestone 9 --- CI/CD Infrastructure Delivery

**Objective** Deliver infrastructure changes through GitHub Actions.

**Key Work** - Implement Terraform workflow - Require approval before
apply

**Deliverables** - .github/workflows/terraform.yml -
.github/workflows/ansible-check.yml

**Done** - Terraform plan and apply executed via CI pipeline

------------------------------------------------------------------------

### Milestone 10 --- Reliability Drills

**Objective** Demonstrate operational readiness.

**Key Work** - Restore VM from PBS - Simulate outages - Document
incident scenarios

**Deliverables** - docs/operations/incident-scenarios.md -
docs/platform-story.md

**Done** - Platform reproducible and documented

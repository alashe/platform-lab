# Homelab Platform Engineering Milestone Plan

This document is the authoritative milestone plan for the platform‑lab
project.

The platform demonstrates:

-   Infrastructure provisioning with Terraform
-   Configuration management with Ansible
-   Containerized workloads with Docker
-   Observability and monitoring
-   Backup and recovery architecture
-   Infrastructure delivery through CI/CD

## Architectural Invariants

These rules do not change unless explicitly revised:

-   Terraform provisions infrastructure where needed
    -   AWS EC2
    -   Optional Proxmox VMs
-   Ansible configures all host types
    -   Bare‑metal utility node
    -   Proxmox VMs
    -   AWS EC2 instances
-   The same Ansible roles should be reusable across these targets
    whenever possible.

------------------------------------------------------------------------

# Milestones

## Milestone 0 --- Lab Blueprint and Standards

Objective: Establish architecture documentation, repository structure,
and infrastructure conventions.

Key Work - Initialize GitHub repository - Define hostnames, roles, and
IP plan - Create architecture diagrams - Establish operational Makefile
commands

Deliverables - docs/architecture/platform-diagram.md -
docs/architecture/decisions.md - docs/setup/local-prereqs.md

Done - Repository structure committed - Architecture diagram completed -
Node roles and IPs documented

------------------------------------------------------------------------

## Milestone 1 --- Proxmox Baseline

Objective: Create a stable virtualization platform.

Key Work - Patch Proxmox hosts - Configure SSH access - Create Debian VM
template

Deliverables - docs/operations/proxmox-baseline.md

Done - VM template clones successfully - Hosts patched and documented

------------------------------------------------------------------------

## Milestone 2 --- Career‑Grade Backups

Objective: Implement PBS → NAS → Cold‑tier backup architecture.

Key Work - Deploy Proxmox Backup Server - Configure scheduled VM
backups - Replicate backups to NAS - Implement cold‑tier storage copy

Deliverables - docs/operations/backup-restore.md -
scripts/restore-check.sh

Done - Successful restore test executed

------------------------------------------------------------------------

## Milestone 3 --- Utility Node Foundation

Objective: Deploy Debian utility node running Docker.

Key Work - Install Debian - Install Docker runtime - Prepare Ansible
inventory

Deliverables - docs/operations/utility-node.md

Done - Docker services persist after reboot

------------------------------------------------------------------------

## Milestone 4 --- Pi‑hole Limited Rollout

Objective: Deploy DNS filtering in a controlled rollout.

Key Work - Deploy Pi‑hole container - Configure volumes - Test DNS
filtering with limited devices

Deliverables - docs/operations/pihole-rollout.md

Done - DNS stable for test clients

------------------------------------------------------------------------

## Milestone 5 --- Monitoring Layer

Objective: Deploy Prometheus/Grafana monitoring VM.

Key Work - Create monitoring VM - Deploy monitoring stack - Configure
dashboards and alerts

Deliverables - docs/operations/monitoring-runbook.md

Done - Alerts validated - Monitoring VM included in backups

------------------------------------------------------------------------

## Milestone 6 --- Pi‑hole Whole‑Home Cutover

Objective: Promote Pi‑hole to primary DNS for the home network.

Key Work - Update router DHCP settings - Validate DNS resolution across
devices

Deliverables - docs/operations/pihole-cutover.md

Done - Home network operating on Pi‑hole DNS

------------------------------------------------------------------------

## Milestone 7 --- Ansible Automation with Vault

Objective: Implement configuration management using Ansible.

Key Work - Build Ansible roles and playbooks - Implement Ansible Vault
secrets - Deploy services through playbooks

Deliverables - ansible/playbooks/baseline.yml -
ansible/playbooks/docker.yml - ansible/playbooks/monitoring-target.yml

Done - Idempotent playbooks deploy baseline configuration

------------------------------------------------------------------------

## Milestone 8 --- AWS Infrastructure Mirror

Objective: Provision AWS infrastructure using Terraform.

Key Work - Create Terraform modules - Deploy EC2 monitoring target - Run
Ansible configuration on EC2

Deliverables - terraform/modules/ec2_monitoring_target/ -
terraform/backend/backend-notes.md

Done - Infrastructure deployable with Terraform and configured via
Ansible

------------------------------------------------------------------------

## Milestone 9 --- CI/CD Infrastructure Delivery Pipeline

Objective: Deliver infrastructure changes through GitHub Actions.

Key Work - Implement Terraform workflow - Validate Terraform
configuration - Require approval before apply

Deliverables - .github/workflows/terraform.yml -
.github/workflows/ansible-check.yml

Done - Terraform plan and apply executed via CI pipeline

------------------------------------------------------------------------

## Milestone 10 --- Reliability Drills

Objective: Demonstrate operational readiness.

Key Work - Restore VM from PBS - Simulate service outages - Document
incident scenarios

Deliverables - docs/operations/incident-scenarios.md -
docs/platform-story.md

Done - Platform reproducible and documented

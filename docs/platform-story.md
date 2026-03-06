# Platform Story

This project demonstrates the design and operation of a small internal service platform built across a homelab and AWS.

## Core Technologies

- Proxmox virtualization
- Docker containers
- Ansible configuration management
- Terraform infrastructure provisioning
- Prometheus and Grafana monitoring
- Proxmox Backup Server backup architecture

## Platform Goals

- Demonstrate infrastructure automation
- Maintain parity between homelab and AWS environments
- Provide observable services with dashboards and alerts
- Validate recovery procedures through restore testing

## Deployment Model

The deployment model is intentionally consistent across targets.

### Terraform

Terraform provisions infrastructure where infrastructure creation is required:

- Proxmox VMs
- AWS EC2 infrastructure

Terraform does not manage bare-metal installation in this project.

### Ansible

Ansible configures all target host types:

- bare-metal utility node
- Proxmox VMs
- AWS EC2 instances

The same Ansible roles and playbooks should be reused across these targets whenever practical.

### Docker

Docker runs the service workloads after hosts are provisioned and configured.

### Monitoring

Monitoring collects metrics, dashboards, and alerts for the platform, including local and cloud-hosted targets.

### Backups

Backup architecture protects virtualized infrastructure using:

- Proxmox Backup Server
- NAS replication
- cold-tier storage

## Operational Model

1. Terraform provisions infrastructure for Proxmox VM and AWS environments.
2. Ansible configures the resulting hosts and deploys services.
3. Docker runs the workloads.
4. Monitoring captures health, metrics, and alerts.
5. Backup and restore procedures validate recoverability.

## Interview Narrative

This platform demonstrates a realistic infrastructure workflow:

- provision infrastructure with Terraform
- configure hosts with Ansible
- run workloads with Docker
- observe the platform through dashboards and alerts
- recover through documented backup and restore procedures

It also demonstrates a cross-target operations model in which the same configuration approach applies to bare metal, Proxmox VM, and AWS EC2, while Terraform remains the infrastructure provisioning layer for Proxmox VM and AWS.

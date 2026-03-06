# Architecture Decisions

This file records architecture decisions for the platform-lab project.

## ADR-001: Virtualization Platform

**Decision**  
Use Proxmox as the homelab virtualization platform.

**Reason**  
Proxmox provides VM lifecycle management and integrates with Proxmox Backup Server.

---

## ADR-002: Configuration Management

**Decision**  
Use Ansible for host configuration.

**Reason**  
Ansible is agentless and can configure bare-metal hosts, Proxmox VMs, and AWS EC2 instances using a consistent operational model.

---

## ADR-003: Infrastructure Provisioning

**Decision**  
Use Terraform for infrastructure provisioning.

**Scope**  
Terraform provisions:

- AWS EC2 infrastructure
- Proxmox VMs

Terraform does not provision bare-metal hosts in this project.

**Reason**  
This keeps infrastructure creation consistent across VM and cloud targets while allowing Ansible to remain the common configuration layer.

---

## ADR-004: Cross-Target Configuration Model

**Decision**  
Use Ansible as the cross-target configuration and deployment layer across:

- bare metal
- Proxmox VM
- AWS EC2

**Reason**  
This preserves a single operational pattern for host configuration and service deployment, even when infrastructure is created through different mechanisms.

---

## ADR-005: Monitoring Stack

**Decision**  
Use Prometheus and Grafana for monitoring and observability.

**Reason**  
They provide a widely used open-source observability stack with strong support for dashboards and alerting.

---

## ADR-006: Backup Architecture

**Decision**  
Use Proxmox Backup Server with NAS replication and cold-tier storage.

**Reason**  
This provides reliable VM backups and supports tested restore workflows.

---

## ADR-007: CI/CD Scope

**Decision**  
Use a single infrastructure delivery pipeline for Terraform-managed environments.

**Scope**  
The CI/CD pipeline supports Terraform validation, planning, approval, and apply for:

- `terraform/environments/homelab/`
- `terraform/environments/aws-dev/`

**Reason**  
This keeps infrastructure delivery centralized and avoids splitting Terraform delivery into separate pipelines for Proxmox VM and AWS without a clear need.

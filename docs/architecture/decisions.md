# Architecture Decisions

## ADR‑001: Virtualization Platform

Decision Use Proxmox as the homelab virtualization platform.

Reason Provides VM lifecycle management and integrates with Proxmox
Backup Server.

------------------------------------------------------------------------

## ADR‑002: Configuration Management

Decision Use Ansible for host configuration.

Reason Agentless automation that can configure bare‑metal, VM, and cloud
hosts.

------------------------------------------------------------------------

## ADR‑003: Infrastructure Provisioning

Decision Use Terraform for infrastructure provisioning.

Scope - AWS EC2 instances - Optional Proxmox VM provisioning

Bare‑metal hosts are installed manually and configured via Ansible.

------------------------------------------------------------------------

## ADR‑004: Monitoring Stack

Decision Use Prometheus and Grafana.

Reason Widely used open‑source observability stack.

------------------------------------------------------------------------

## ADR‑005: Backup Architecture

Decision Use Proxmox Backup Server with NAS replication and cold
storage.

Reason Provides reliable VM backups and supports tested restore
workflows.

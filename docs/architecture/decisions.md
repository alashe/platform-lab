# Architecture Decisions

This file records architecture decisions for the platform lab.

## ADR‑001: Virtualization Platform

**Decision** Use Proxmox as the homelab virtualization platform.

**Rationale** - Open source - Supports VM snapshots and backups -
Integrates with Proxmox Backup Server

------------------------------------------------------------------------

## ADR‑002: Configuration Management

**Decision** Use Ansible for host configuration.

**Rationale** - Agentless automation - Widely used in infrastructure
operations

------------------------------------------------------------------------

## ADR‑003: Infrastructure Provisioning

**Decision** Use Terraform to provision AWS infrastructure.

**Rationale** - Industry‑standard Infrastructure‑as‑Code - Enables
repeatable deployments

------------------------------------------------------------------------

## ADR‑004: Monitoring Stack

**Decision** Use Prometheus and Grafana.

**Rationale** - Common observability stack - Strong ecosystem for
dashboards and alerting

------------------------------------------------------------------------

## ADR‑005: Backup Architecture

**Decision** Use Proxmox Backup Server with NAS replication and cold
storage.

**Rationale** - Provides reliable VM backups - Supports tested restore
workflows

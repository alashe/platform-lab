# Platform Story

This project demonstrates the design and operation of a small internal
service platform built across a homelab and AWS.

## Core Technologies

-   Proxmox virtualization
-   Docker containers
-   Ansible configuration management
-   Terraform infrastructure provisioning
-   Prometheus and Grafana monitoring
-   Proxmox Backup Server

## Platform Goals

-   Demonstrate infrastructure automation
-   Maintain parity between homelab and AWS environments
-   Provide observable services with dashboards and alerts
-   Validate recovery procedures through restore testing

## Deployment Model

Terraform provisions infrastructure (AWS EC2 and optional Proxmox VMs).

Ansible configures hosts (bare metal, VM, and EC2).

Docker runs service workloads.

Monitoring collects metrics and alerts.

Backup architecture protects infrastructure using PBS → NAS → cold
storage.

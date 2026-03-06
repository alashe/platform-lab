# Platform Lab Story

This project demonstrates the design and operation of a small internal
service platform built across a homelab and AWS.

## Core Technologies

-   Proxmox virtualization
-   Docker containers
-   Ansible configuration management
-   Terraform infrastructure provisioning
-   Prometheus and Grafana monitoring
-   Proxmox Backup Server backup architecture

## Platform Goals

-   Demonstrate infrastructure automation
-   Maintain parity between homelab and AWS environments
-   Provide observable services with dashboards and alerts
-   Validate recovery procedures through restore tests

## Operational Model

1.  Terraform provisions infrastructure.
2.  Ansible configures hosts and deploys services.
3.  Docker runs workloads.
4.  Monitoring captures metrics and logs.
5.  Backup architecture protects infrastructure.

The platform demonstrates a realistic DevOps workflow suitable for
interviews and portfolio review.

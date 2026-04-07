# Platform Architecture

## Overview

This platform runs a small service workload across two environments — a local Proxmox cluster and AWS — with unified configuration management, observability, and a tiered backup strategy.

The design principle is **operational realism over complexity**: every layer exists because it solves a real operational problem, not to demonstrate a tool.

---

## Goals and Non-Goals

### Goals

- Operate real services with uptime expectations and measurable signals
- Use a consistent automation flow across environments: **Terraform → Ansible → Docker → Monitoring → Operations**
- Keep the monitoring layer independent of the service host
- Implement backups with documented restore procedures and periodic restore drills
- Produce documentation suitable for interview narratives — architecture, runbooks, incident notes

### Non-Goals (this phase)

- Kubernetes
- Complex HA or cluster orchestration
- Multi-region AWS design
- Full zero-trust networking or hybrid VPN mesh

---

## Environment Map

### Local — Proxmox Cluster

**Physical nodes:**

| Node | Hardware | Role |
|---|---|---|
| pve01 | Dell Precision 7550 | Primary compute |
| pve02 | Dell Precision 7560 | Secondary compute |
| qdev01 | HP Elitedesk 800 mini | Corosync QDevice · bare-metal utility node |

**Network infrastructure:**

| Device | Role |
|---|---|
| Router | Gateway and DHCP server |
| Unmanaged 2.5GbE switch (GigaPlus 10-port) | Connects router to lab nodes; enables 2.5GbE connectivity to the NAS; plug-and-play, no VLAN or LACP support |

**Storage:**

| Device | OS / Filesystem | Role |
|---|---|---|
| NAS — Terramaster F4-424 Pro | TrueNAS Scale · ZFS | Primary storage; PBS datastore via NFS share; personal data (documents, photos, music) |

**NAS storage:**

| Pool | Drives | Capacity | Status |
|---|---|---|---|
| Boot pool | 2x 1TB SSD (P310) | Mirror — ~1TB usable | Installed 2026-03-29 |
| Data pool | 3x 8TB HDD | RAIDZ1 — ~14.55 TiB usable | Created 2026-04-02 |

> TrueNAS OS boots from a mirrored pool across two 1TB P310 SSDs. The data pool is independent. See ADR-016 and ADR-020.

**Developer workstation:**

| Machine | OS | Role |
|---|---|---|
| Fedora workstation | Fedora Linux | Local Ansible + Terraform development — not cluster-managed |

> The Fedora workstation is used for writing and testing code locally before pushing. It is not in the Ansible inventory and is not managed by the platform.

**VMs on cluster:**

| VM | Node | Purpose | Services |
|---|---|---|---|
| `util01` | pve01 | Primary service runtime | Docker · Pi-hole · nginx target |
| `mon01` | pve02 | Observability — independent failure domain | Prometheus · Grafana · Alertmanager · Loki · Uptime Kuma |
| `pbs01` | pve01 | Backup server | Proxmox Backup Server |
| `auto01` | pve02 | Ansible + Terraform execution | Ansible controller · Terraform workspace |
| `llm01` | TBD at M12 | LLM inference service | Ollama — AI capstone (M12) |

> Node assignments are recommended defaults, adjusted based on resource availability at build time. **Exception: `pbs01` on `pve01` is confirmed** — its datastore is hosted on the NAS via NFS, so migrating `pbs01` to `pve02` requires no datastore changes.

**Non-platform VMs (retained — not Ansible/Terraform managed):**

| VM | Node | Notes |
|---|---|---|
| `win01` (VM ID 111) | pve01 | Retained from pre-platform setup · personal use |

> This VM coexists on the cluster but is outside the platform's automation scope. RAM allocation must be accounted for when sizing platform VMs, particularly `llm01` at M12.

**HP Elitedesk 800 mini — bare-metal roles:**

| Role | Details |
|---|---|
| Corosync QDevice | Provides quorum tie-breaker vote for the 2-node cluster |
| Bare-metal utility node | Docker · Pi-hole (secondary DNS) · nginx target — mirrors Utility VM |

> The EliteDesk is managed by the same Ansible roles as the Utility VM, demonstrating role portability across bare metal and VMs. Pi-hole on the EliteDesk is activated as secondary DNS during the whole-home cutover (Milestone 8), after the Utility VM is established as primary.

**Separation requirement:** `mon01` runs on `pve02`, independent of `util01` on `pve01`, so service failures do not remove visibility or alerting.

**Mobility requirement:** `mon01` must be restorable or migratable between Proxmox hosts. PBS-backed restore is sufficient for this phase.

---

### AWS — Terraform-Provisioned

```
VPC (10.0.0.0/16)
└── Public subnet
    ├── Security Group     SSH · HTTP · Node Exporter port
    ├── EC2 (t3.micro)     Debian AMI · Docker runtime
    │   ├── nginx          monitoring target (mirrors homelab)
    │   └── node_exporter  scraped by homelab Prometheus
    └── IAM instance profile   least-privilege · CloudWatch write
```

Terraform state is stored in S3 with native locking (`use_lockfile = true`, Terraform ≥ 1.10).
Ansible runs the identical baseline playbook against EC2 as it does locally.

---

## Functional Services

### 1 — Network Service: Pi-hole

Role: primary DNS filtering for the home network.

**Deployment model:**

| Instance | Host | Role | Activated |
|---|---|---|---|
| Primary | `util01` (pve01) | Active DNS for whole network | Milestone 8 cutover |
| Secondary | HP Elitedesk 800 mini | Failover DNS | Milestone 8 cutover |

Router DHCP distributes both IPs: Utility VM as primary, EliteDesk as secondary. Devices fail over automatically if the Utility VM is unreachable.

Operational expectations:
- DNS availability and stable performance
- Config managed via Ansible — same role applied to both hosts
- Visibility into query volume, block rate, and upstream latency

### 2 — Monitoring Target Service

Role: a simple containerized service used to generate metrics and logs and practice operations.

Runs identically across environments:

| Attribute | Homelab (Utility VM) | Homelab (EliteDesk) | AWS EC2 |
|---|---|---|---|
| Runtime | Docker Compose | Docker Compose | Docker Compose |
| Config source | Ansible role | Same Ansible role | Same Ansible role |
| Metrics | node_exporter | node_exporter | node_exporter → homelab Prometheus |
| Logs | Promtail → Loki | Promtail → Loki | CloudWatch agent |
| Restart policy | `unless-stopped` | `unless-stopped` | `unless-stopped` |

Operational expectations:
- Health endpoint at `/healthz`
- Basic metrics: request rate, latency, error rate
- Alerting on availability and error rate

### 3 — Configuration Compliance

Ansible enforces baseline configuration across all managed hosts:

| Area | What is enforced |
|---|---|
| SSH | `PasswordAuthentication no` · `PermitRootLogin no` · idle timeout |
| Packages | curl · vim · ufw · fail2ban · unattended-upgrades |
| Docker | CE runtime · Compose plugin · user group membership |
| Firewall | ufw default deny · allow only required ports |
| Users | `ops` user with sudo · authorized keys deployed |

Managed hosts: `util01`, `mon01`, `pbs01`, `auto01`, `qdev01` (bare metal), AWS EC2.  
The Fedora workstation is not managed by Ansible.

Compliance is validated by re-running playbooks in check mode to detect drift.

---

## Observability Stack

Hosted on `mon01` (pve02). Scrapes all managed hosts and EC2.

| Component | Role |
|---|---|
| Prometheus | Metrics collection and alerting rules |
| Grafana | Dashboards and visualization |
| Alertmanager | Alert routing and notification |
| Loki | Log aggregation |
| Uptime Kuma | Endpoint availability monitoring and status page |
| Promtail | Log shipping agent (all local hosts) |

```
Targets                     Collection               Storage / UI
─────────────────────────────────────────────────────────────────
All hosts (node_exporter) → Prometheus           → Grafana dashboards
Containers (cAdvisor)     → Prometheus           → Alertmanager → alerts
Pi-hole (FTL exporter)    → Prometheus
Container stdout          → Promtail             → Loki → Grafana (Explore)
EC2 syslog                → CloudWatch
HTTP endpoints            → Uptime Kuma          → Status page · alerts
```

**Uptime Kuma monitors:**
- nginx target — `util01` (`/healthz`)
- nginx target — EliteDesk (`/healthz`)
- nginx target — AWS EC2 (`/healthz`)
- Pi-hole DNS endpoint — `util01`
- Pi-hole DNS endpoint — EliteDesk
- Grafana availability
- External DNS — upstream resolution check

### Observability Signals — Monitoring Target Service

| Signal | Source |
|---|---|
| Availability (up/down) | Uptime Kuma |
| Request rate (RPS) | Prometheus |
| Latency (p95) | Prometheus |
| Error rate (5xx) | Prometheus + Alert |
| CPU / memory / disk usage | Prometheus |
| `/healthz` endpoint | Uptime Kuma probe |
| Container restart count | Prometheus + Alert |

### Observability Signals — Pi-hole

| Signal | Source |
|---|---|
| DNS availability | Uptime Kuma |
| Query rate | Prometheus |
| Blocked vs. allowed ratio | Prometheus |
| Upstream response latency | Prometheus |
| Container / VM resource usage | Prometheus |

### Dashboards

- Node health (all environments)
- Container resource usage
- Pi-hole DNS analytics
- Cross-environment service status
- Uptime Kuma status page

---

## Automation Flow

```
1. Terraform         provision infrastructure via Automation VM or CI pipeline
        ↓
2. Ansible           baseline.yml  → SSH config · packages · firewall · users
                     docker.yml    → Docker runtime + Compose
                     monitoring-target.yml → deploy nginx container
        ↓
3. Prometheus        auto-discovers new node_exporter endpoint
   Uptime Kuma       endpoint check configured for new target
        ↓
4. Grafana           new host appears in dashboards within one scrape interval
```

From `terraform apply` to a monitored, compliant host: under 15 minutes.

---

## Backup Architecture

```
VM snapshots (Proxmox schedule)
        ↓
Proxmox Backup Server (`pbs01` · pve01)
        ↓
NAS  (warm storage · local)
  RAIDZ1 HDD pool — 3x 8TB (~16TB usable)
        ↓
Cold tier  (offsite / S3 Glacier)
```

**Operational requirement:** Backups are not considered complete until a restore drill is performed and documented.

**Recovery targets:**

| Metric | Target |
|---|---|
| RTO | < 30 minutes (full VM restore from PBS) |
| RPO | 24 hours (daily snapshots) |

**Configuration recovery:**

| Asset | Backup method |
|---|---|
| Ansible playbooks / roles | Git repository |
| Terraform modules + state | Git + S3 backend |
| Grafana dashboards | JSON exports in repo |
| Uptime Kuma config | Exported JSON in repo |

---

## Related Docs

- [Architecture decisions](decisions.md)
- [Naming convention](naming-convention.md)
- [Access control](access-control.md)
- [Hardening baseline](hardening-baseline.md)
- [Milestone plan](homelab-milestone-plan.md)
- [Terraform reference](../../terraform/README.md)
- [Ansible reference](../../ansible/README.md)
- [Monitoring runbook](../operations/monitoring-runbook.md)
- [Backup & restore](../operations/backup-restore.md)

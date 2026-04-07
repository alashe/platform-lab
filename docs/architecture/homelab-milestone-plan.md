# Homelab Milestone Plan

Authoritative milestone plan for the `platform-lab` project.  
Status reflects reality only — aspirational items are marked 🔲, not ✅.

> **Last updated:** 2026-04-06
> **Current phase:** Milestone 1 — Proxmox Baseline (in progress) · non-platform VMs restored to pve01

---

## Status Key

| Symbol | Meaning |
|---|---|
| ✅ | Complete and verified |
| 🔄 | In progress |
| 🔲 | Planned — not started |
| ⏸ | Deferred — conscious decision |

---

## Architectural Invariants

These rules do not change unless explicitly revised and documented in [`decisions.md`](decisions.md).

- **Terraform** provisions infrastructure: AWS EC2 and Proxmox VMs — with one bootstrap exception: the Automation VM is provisioned manually because it must exist before Terraform can run. All other Proxmox VMs go through Terraform.
- **Ansible** configures all host types: bare-metal (EliteDesk), Proxmox VMs, AWS EC2 instances
- Ansible roles are written to be reusable across host types wherever possible
- **Ansible execution** runs from the Automation VM — the Fedora workstation is for development only
- CI/CD is a single infrastructure delivery pipeline for Terraform-managed environments
- Operational commands are standardized through the root `Makefile`

---

## Makefile Reference

```bash
make tf-plan ENV=homelab
make tf-apply ENV=homelab
make tf-plan ENV=aws-dev
make tf-apply ENV=aws-dev
make ansible-baseline INVENTORY=ansible/inventories/homelab
make restore-test
make validate
```

---

## Milestone Sequence

| # | Milestone | Depends On |
|---|---|---|
| 0 | Lab Blueprint and Standards | — |
| 1 | Proxmox Baseline | 0 |
| 2 | Backup Architecture | 1 |
| 3 | Utility Node Foundation | 1 |
| 4 | Terraform Foundations (Homelab) | 3 |
| **5** | **CI/CD Infrastructure Delivery Pipeline** | **4** |
| 6 | Network Service: Pi-hole Limited Rollout | 3 |
| 7 | Monitoring Layer | 3 |
| 8 | Pi-hole Whole-Home Cutover + EliteDesk Secondary DNS | 6, 7 |
| 9 | Ansible Automation with Vault | 5, 7 |
| 10 | AWS Infrastructure Mirror (Terraform + Ansible) | 5, 9 |
| 11 | Reliability Drills and Platform Documentation | 10 |
| 12 | AI Capstone: LLM Inference as a Managed Platform Service | 5, 7, 11 |

> **Why CI/CD is Milestone 5:** The pipeline is built as soon as Terraform is proven locally, so every subsequent change is delivered through it.
>
> **Why the Ansible syntax check is included at Milestone 5:** `ansible-playbook --syntax-check` requires no live hosts. Adding the workflow early creates a feedback loop for every playbook written during Milestone 9.
>
> **Pi-hole sequencing:** Pi-hole is deployed to the Utility VM at Milestone 6 (limited rollout). The EliteDesk Pi-hole instance is deployed by the same Ansible role but is not activated as a DNS server until Milestone 8 (whole-home cutover), when the router is configured with both IPs.

---

## VM Reference

Recommended node assignments. Adjust based on resource availability at build time.

| VM | Node | Purpose |
|---|---|---|
| Utility VM | pve01 | Primary service runtime — Docker · Pi-hole · nginx target |
| Monitoring VM | pve02 | Observability — must be on separate node from Utility VM |
| PBS VM | pve01 | Proxmox Backup Server |
| Automation VM | pve02 | Ansible execution · Terraform workspace |

**HP Elitedesk 800 mini (bare metal — not a Proxmox node):**

| Role | Details |
|---|---|
| Corosync QDevice | Quorum tie-breaker for the 2-node cluster |
| Bare-metal utility node | Docker · Pi-hole (secondary DNS) · nginx target — Ansible-managed |

---

## Milestone 0 — Lab Blueprint and Standards

**Objective:** Establish architecture documentation, repository structure, and infrastructure conventions.

| Item | Status | Notes |
|---|---|---|
| GitHub repository initialized | ✅ | Private repo |
| Repository layout implemented | 🔄 | Structure defined; files in progress |
| Hostnames, roles, and IP plan documented | ✅ | `docs/architecture/naming-convention.md` |
| Router DHCP pool shrunk to `192.168.0.100`–`192.168.0.253` | ✅ | Frees `.2`–`.99` for static assignments; do before assigning static IPs |
| IP plan written into `docs/architecture/naming-convention.md` | ✅ | Full address table per range grouping |
| Architecture diagram created | ✅ | `docs/architecture/platform-lab.md` |
| Architecture decisions log started | ✅ | `docs/architecture/decisions.md` |
| Access control standard documented | ✅ | `docs/architecture/access-control.md` |
| Hardening baseline documented | ✅ | `docs/architecture/hardening-baseline.md` |
| Milestone plan written | ✅ | This document |
| `docs/setup/local-prereqs.md` written | ✅ | |
| Root `Makefile` with operational commands | ✅ | |
| Terraform module scaffold in place | ✅ | `modules/proxmox_vm/`, `modules/ec2_monitoring_target/` |
| Terraform environment scaffold in place | ✅ | `environments/homelab/`, `environments/aws-dev/` |
| Ansible inventory scaffold in place | ✅ | `homelab/`, `aws/`, `mixed/` |
| Public GitHub repo created | ✅ | `alashe/platform-lab` |
| `.github/workflows/sync-public.yml` written | ✅ | Workflow active — triggers on push to main |
| `.github/sync-manifest.txt` initialized | ✅ | Milestone 0 docs + showcase page listed; code files added as built |

---

## Pre-req: TrueNAS Scale Install ✅ Complete 2026-04-06

**Objective:** Install TrueNAS Scale on the Terramaster F4-424 Pro, create the RAIDZ1 data pool with 3x 8TB HDDs, and establish the dataset hierarchy. Must complete before Milestone 2 — the PBS datastore NFS share depends on this pool existing.

> **NFS share verification** (one remaining item) is intentionally deferred to Milestone 2 when PBS is configured.

> **Why a named pre-req and not a milestone:** The TrueNAS install is not a Proxmox VM operation and does not depend on the cluster being built. It can proceed in parallel with or before Milestone 1. Naming it explicitly prevents it from being silently assumed.

> **Hardware milestone (2026-03-28):** 3x 8TB HDDs purchased. Pool configuration: RAIDZ1 (~16TB usable, 1-drive fault tolerance). See ADR-016.

| Item | Status | Notes |
|---|---|---|
| 3x 8TB HDDs purchased | ✅ | 2026-03-28 — RAIDZ1 pool (see ADR-016) |
| TrueNAS Scale ISO downloaded and boot media prepared | ✅ | Loaded on Ventoy USB |
| TrueNAS Scale installed to mirrored boot pool (2x 1TB P310) | ✅ | Mirrored boot pool — see ADR-020 |
| Network config complete — static IP, accessible on LAN | ✅ | `192.168.0.81` — `nas01.lab` |
| Dataset hierarchy designed and documented in `decisions.md` | ✅ | ADR-021 — `backups/pbs`, `apps/<name>` (per-app at deploy time), `personal/` |
| RAIDZ1 pool created with 3x 8TB HDDs | ✅ | ~14.55 TiB usable; see ADR-016 |
| Datasets created per hierarchy design | ✅ | `apps`, `backups`, `backups/pbs`, `personal` — managed via Terraform |
| NFS share for PBS datastore configured | ✅ | `tank/backups/pbs` — restricted to pbs01 (192.168.0.63) via Ansible |
| NFS share accessible from Proxmox network | 🔲 | Verify when PBS is configured in M2 |
| Snapshot schedule configured | ✅ | Daily snapshots: `tank/backups/pbs` (7-day retention), `tank/personal` (4-week retention) — via Ansible |
| Scrub schedule configured | ✅ | Monthly scrub of `tank` pool (weekly check, 30-day threshold) — via Ansible |
| Terraform `deevus/truenas` provider configured | ✅ | Manages datasets and snapshot schedules — see ADR-017 |
| All post-pool dataset config committed to Terraform | ✅ | Dataset hierarchy managed via `terraform/modules/truenas` |
| NFS shares and user accounts managed via Ansible | ✅ | `arensb.truenas` collection; `ansible/roles/truenas` |
| Ansible documented for day-2 config tasks | ✅ | `docs/operations/truenas-day2.md` |
| Install process and dataset hierarchy decisions documented | ✅ | `docs/operations/truenas-setup.md`; dataset hierarchy in ADR-021 |

---

## Milestone 1 — Proxmox Baseline (pve01 + pve02)

**Objective:** Create a stable virtualization foundation for the homelab platform.

| Item | Status | Notes |
|---|---|---|
| win-11-pro VM disk backed up before reinstall | ✅ | Backed up to `nas01-backups` NFS share via Proxmox backup — 2026-04-05 |
| ubuntu-server VM — back up before reinstall | ✅ | Backed up to `nas01-backups` NFS share via Proxmox backup — 2026-04-05 · VM deleted 2026-04-07, backup removed from NAS |
| `ala-pve01` renamed to `pve01` in Proxmox | ✅ | Fresh install with correct hostname — 2026-04-05 |
| `pve01` static IP set to `192.168.0.51` | ✅ | 2026-04-05 |
| Both Proxmox hosts patched and rebooted | 🔲 | |
| SSH key access configured on both hosts | 🔄 | pve01 complete 2026-04-06 · pve02 pending this weekend |
| Firewall posture defined | ✅ | LAN-trust — Proxmox built-in firewall disabled; documented in hardening-baseline.md |
| Debian 12 VM template created | 🔲 | |
| Template successfully cloned to test VM | 🔲 | |
| Corosync QDevice configured on HP EliteDesk | 🔲 | `corosync-qnetd` installed; cluster quorum verified |
| `docs/operations/proxmox-baseline.md` written | 🔄 | In progress — review and finalize at end of M1 |

---

## Milestone 2 — Backup Architecture (PBS → NAS → Cold Tier)

**Objective:** Implement a multi-tier backup architecture and verify recovery capability.

> **Depends on:** TrueNAS pre-req complete — NFS share for PBS datastore must exist before PBS can be configured.

| Item | Status | Notes |
|---|---|---|
| PBS VM provisioned on pve01 | 🔲 | |
| Scheduled VM backup jobs configured | 🔲 | |
| First backup completed and verified | 🔲 | |
| PBS datastore pointed at NAS NFS share | 🔲 | `tank/pbs` — RAIDZ1 HDD pool (see ADR-016) |
| Cold-tier copy process implemented | 🔲 | |
| `scripts/restore-check.sh` written | 🔲 | |
| `make restore-test` functional | 🔲 | |
| Restore test verified — RTO measured | 🔲 | |
| `docs/operations/backup-restore.md` reflects actual state | 🔄 | Doc written; lab not yet built |

---

## Milestone 3 — Bootstrap: Automation VM + EliteDesk

**Objective:** Manually provision the two hosts that cannot go through Terraform. The Automation VM must exist before Terraform can run. The EliteDesk is bare metal and outside Terraform's scope entirely. Nothing else is provisioned here — the Utility VM waits until Terraform is working in M4.

| Item | Status | Notes |
|---|---|---|
| Automation VM provisioned manually on pve02 | 🔲 | Bootstrap exception — Terraform cannot provision its own execution host |
| Debian 12 installed and accessible via SSH | 🔲 | |
| Baseline host security configured on Automation VM | 🔲 | |
| Ansible installed and functional on Automation VM | 🔲 | |
| Automation VM added to Ansible homelab inventory | 🔲 | |
| Ansible Vault file created and encrypted | 🔲 | |
| Debian 12 installed on HP EliteDesk | 🔲 | Bare metal — manual install |
| Baseline host security configured on EliteDesk | 🔲 | |
| EliteDesk added to Ansible homelab inventory | 🔲 | |
| `docs/operations/utility-node.md` scaffolded | 🔲 | Full content added after M4 when Utility VM exists |

---

## Milestone 4 — Terraform Foundations + Utility VM

**Objective:** Install Terraform on the Automation VM, write the Proxmox VM module, and use it to provision the Utility VM. This is the first real `terraform apply` — and the point where the environment becomes consistent with the architectural invariant that Terraform provisions all Proxmox VMs (except the Automation VM bootstrap). Complete the Utility VM configuration once it exists.

| Item | Status | Notes |
|---|---|---|
| Terraform installed on Automation VM | 🔲 | |
| Terraform provider and version constraints defined | 🔲 | |
| `terraform/environments/homelab/` scaffolded | 🔲 | |
| `terraform/modules/proxmox_vm/` written | 🔲 | |
| Utility VM provisioned via `terraform apply` | 🔲 | First Terraform-managed VM in the environment |
| Utility VM added to Ansible homelab inventory | 🔲 | |
| Baseline host security configured on Utility VM | 🔲 | Via Ansible |
| Docker CE + Compose installed on Utility VM | 🔲 | Via Ansible |
| Docker CE + Compose installed on EliteDesk | 🔲 | Via Ansible — same role as Utility VM |
| Docker services persist through reboot on both | 🔲 | |
| `terraform fmt`, `validate`, `plan`, `apply`, `destroy` workflow documented | 🔲 | |
| `docs/setup/terraform-prereqs.md` written | 🔲 | |
| `docs/operations/utility-node.md` completed | 🔲 | |
| SSH key rotation — replace `fedora_ed25519` with `auto01` key | 🔲 | Terraform provider config (`environments/homelab/main.tf`) and Ansible inventory (`hosts.ini`) both use workstation key during pre-req/M1 development; must be updated when `auto01` becomes the execution host |

---

## Milestone 5 — CI/CD Infrastructure Delivery Pipeline

**Objective:** Deliver Terraform-managed infrastructure changes through a controlled GitHub Actions pipeline. Establish the Ansible syntax check workflow as a feedback loop for all subsequent playbook development.

| Item | Status | Notes |
|---|---|---|
| `.github/workflows/terraform.yml` implemented | 🔲 | Supports `homelab` env; `aws-dev` added at M10 |
| `.github/workflows/ansible-check.yml` implemented | 🔲 | Runs `--syntax-check` on all playbooks |
| `terraform fmt -check` runs in CI | 🔲 | |
| `terraform validate` runs in CI | 🔲 | |
| `terraform plan` output generated on PR | 🔲 | |
| Manual approval gate before apply | 🔲 | |
| `terraform apply` executes through CI after approval | 🔲 | |
| Ansible syntax check runs on PR | 🔲 | No live hosts required |
| Placeholder playbook scaffolded so CI has a target | 🔲 | Replaced by real playbooks at Milestone 9 |
| Pipeline documented in `terraform/README.md` | 🔲 | |
| AWS cost estimate documented before first apply | 🔲 | See ADR-011 · baseline estimate for EC2, S3 |
| AWS Budget alert configured | 🔲 | Alert at $10/mo — before pipeline can trigger `terraform apply` |
| `PUBLIC_REPO_PAT` secret configured in private repo | 🔲 | PAT with `repo` scope on the public repo |
| `PUBLIC_REPO` variable configured in private repo | 🔲 | Set to `<username>/<public-repo-name>` |
| Sync workflow tested — Milestone 0 docs appear in public repo | 🔲 | First live run of `sync-public.yml` |

---

## Milestone 6 — Network Service: Pi-hole Limited Rollout

**Objective:** Deploy Pi-hole on the Utility VM in a controlled rollout. Deploy to EliteDesk via same Ansible role, but do not activate as DNS server until Milestone 8.

| Item | Status | Notes |
|---|---|---|
| Pi-hole container running on Utility VM | 🔲 | |
| Pi-hole container running on EliteDesk | 🔲 | Same Ansible role — not yet active as DNS |
| Persistent storage configured on both | 🔲 | |
| Limited client rollout tested against Utility VM | 🔲 | |
| DNS filtering behavior verified | 🔲 | |
| Rollback procedure documented | 🔲 | |
| `docs/operations/pihole-rollout.md` written | 🔲 | |
| Ansible `utility_node` role deploys Pi-hole to both hosts | 🔲 | |

---

## Milestone 7 — Monitoring Layer (Proxmox VM)

**Objective:** Deploy monitoring infrastructure on pve02. Monitor all current hosts including EliteDesk.

| Item | Status | Notes |
|---|---|---|
| Monitoring VM provisioned on pve02 | 🔲 | Separate node from Utility VM (pve01) |
| Prometheus running, scraping all hosts | 🔲 | Utility VM · EliteDesk · Automation VM |
| node_exporter on all managed hosts | 🔲 | |
| cAdvisor on Utility VM and EliteDesk | 🔲 | |
| Pi-hole FTL exporter on Utility VM and EliteDesk | 🔲 | |
| Grafana running, connected to Prometheus | 🔲 | |
| Loki + Promtail running | 🔲 | |
| Alertmanager configured | 🔲 | |
| Alert rules defined and tested | 🔲 | |
| Uptime Kuma running on Monitoring VM | 🔲 | |
| Uptime Kuma monitors configured | 🔲 | nginx + Pi-hole on Utility VM and EliteDesk · Grafana · external DNS |
| Uptime Kuma EC2 monitor defined but inactive | 🔲 | Activated at Milestone 10 |
| Uptime Kuma config exported as JSON to repo | 🔲 | |
| Grafana dashboards exported as JSON to repo | 🔲 | |
| Monitoring VM included in PBS backup schedule | 🔲 | |
| Monitoring VM recovery verified on alternate host | 🔲 | |
| `docs/sre/slo-definitions.md` authored — SLIs, SLO targets, error budgets defined | 🔲 | Pi-hole DNS · nginx · Prometheus · PBS — see ADR-014, ADR-015 |
| Prometheus recording rules for SLI calculations deployed | 🔲 | `prometheus/rules/slo-recording.yml` |
| Alertmanager burn rate rules active for all three SLOs | 🔲 | Fast burn (1h) and slow burn (6h) windows |
| Grafana error budget panels created for each SLO | 🔲 | One gauge panel per SLI — % budget remaining |
| `docs/operations/monitoring-runbook.md` reflects actual state | 🔄 | Doc written; lab not yet built |
| Jaeger all-in-one container deployed on Monitoring VM | ⏸ | Deferred — tracing adds most value once multi-service calls exist (M10+); revisit after AWS mirror |

---

## Milestone 8 — Pi-hole Whole-Home Cutover + EliteDesk Secondary DNS

**Objective:** Promote Pi-hole to the primary DNS resolver for the whole network. Activate EliteDesk Pi-hole as secondary DNS for automatic failover.

| Item | Status | Notes |
|---|---|---|
| Router DHCP configured — Utility VM as primary DNS | 🔲 | |
| Router DHCP configured — EliteDesk as secondary DNS | 🔲 | |
| DNS resolution verified across multiple device types | 🔲 | |
| Network devices appearing in Pi-hole logs | 🔲 | |
| Failover tested — Utility VM DNS unavailable, EliteDesk takes over | 🔲 | |
| Rollback procedure tested | 🔲 | |
| `docs/operations/pihole-cutover.md` written | 🔲 | |

---

## Milestone 9 — Ansible Automation with Vault

**Objective:** Implement configuration management and cross-target service deployment using Ansible. CI syntax check from Milestone 5 validates every playbook on push.

| Item | Status | Notes |
|---|---|---|
| `ansible/playbooks/baseline.yml` written and tested | 🔲 | |
| `ansible/playbooks/docker.yml` written and tested | 🔲 | |
| `ansible/playbooks/monitoring-target.yml` written and tested | 🔲 | |
| `ansible/playbooks/utility-node.yml` written and tested | 🔲 | |
| All playbooks idempotent (second run = no changes) | 🔲 | |
| Playbooks run correctly on Utility VM, EliteDesk, and Automation VM | 🔲 | |
| Secrets encrypted with Ansible Vault | 🔲 | |
| Vault vars under `group_vars/` or inventory-specific vars | 🔲 | |
| Same roles confirmed reusable across bare metal, Proxmox VM, EC2 | 🔲 | |
| `make ansible-baseline` functional | 🔲 | |
| `ansible/README.md` reflects actual playbook behavior | 🔄 | Doc written; not yet verified against live runs |

---

## Milestone 10 — AWS Infrastructure Mirror (Terraform + Ansible)

**Objective:** Provision AWS infrastructure with Terraform and deploy workloads with Ansible. Extend the existing CI/CD pipeline to support `aws-dev`.

| Item | Status | Notes |
|---|---|---|
| S3 backend bucket created | 🔲 | |
| `terraform/backend/backend-notes.md` written | 🔲 | |
| Cost estimate reviewed and documented | 🔲 | EC2 · S3 · data transfer — see ADR-011 |
| AWS Budget alert active before `terraform apply` | 🔲 | Confirmed from M5 setup |
| `terraform/modules/ec2_monitoring_target/` written | 🔲 | |
| `terraform/environments/aws-dev/` configured | 🔲 | |
| `terraform apply` produces running EC2 instance | 🔲 | |
| `aws-dev` environment added to CI/CD pipeline | 🔲 | Extends M5 pipeline |
| Ansible `baseline.yml` runs clean on EC2 | 🔲 | |
| Ansible `docker.yml` runs clean on EC2 | 🔲 | |
| nginx monitoring target running on EC2 | 🔲 | |
| node_exporter running on EC2 | 🔲 | |
| EC2 scraped by homelab Prometheus | 🔲 | |
| Uptime Kuma EC2 monitor activated | 🔲 | Monitor defined at M7; enabled here |
| Cross-environment Grafana dashboard built | 🔲 | |
| EC2 logs shipping via CloudWatch | 🔲 | |
| `make tf-plan ENV=aws-dev` functional | 🔲 | |
| `make tf-apply ENV=aws-dev` functional | 🔲 | |
| `docs/setup/aws-prereqs.md` written | 🔲 | |
| `terraform/README.md` reflects actual module behavior | 🔄 | Doc written; not yet verified against live runs |

---

## Milestone 11 — Reliability Drills and Platform Documentation

**Objective:** Demonstrate operational readiness and produce an interview-ready platform project.

| Item | Status | Notes |
|---|---|---|
| Scenario 1 — service down drill completed | 🔲 | Time-to-alert recorded (Uptime Kuma + Alertmanager); error budget burn rate reviewed post-drill |
| Scenario 2 — Terraform destroy + rebuild completed | 🔲 | Rebuild time recorded |
| Scenario 3 — config drift detection completed | 🔲 | |
| Scenario 4 — Monitoring VM migration completed | 🔲 | No data gap verified |
| Scenario 5 — VM restore drill completed | 🔲 | Actual RTO recorded |
| Scenario 6 — disk space alert drill completed | 🔲 | |
| Scenario 7 — Ansible compliance audit completed | 🔲 | |
| Pi-hole failover tested — Utility VM down, EliteDesk serves DNS | 🔲 | |
| RTO measured and recorded in `backup-restore.md` | 🔲 | |
| All docs updated to reflect what's actually built | 🔲 | |
| No aspirational claims in `README.md` | 🔲 | |
| Sync manifest reviewed — all public-ready files included | 🔲 | Add any final code files; confirm exclusions are correct |
| Public repo README polished for portfolio presentation | 🔲 | Standalone context for someone landing on the public repo |
| Public repo walkable as a portfolio artifact | 🔲 | No broken links, no placeholder content, no internal references |
| Repo walkable live in an interview | 🔲 | |
| Jaeger trace captured from a reliability drill scenario | 🔲 | Optional — instrument one service (e.g. EC2 nginx target); capture a trace during a drill and export to repo as portfolio artifact |

---

## Milestone 12 — AI Capstone: LLM Inference as a Managed Platform Service

**Objective:** Provision and operate an LLM inference service (Ollama) as a managed platform workload — demonstrating that any service class can be onboarded, monitored, and integrated into operational workflows using the existing stack.

> **Full design doc:** `docs/planning/ai-capstone-llm-inference.md`

| Item | Status | Notes |
|---|---|---|
| Host placement decision made and documented in ADR | 🔲 | Placement must fit existing topology — no exceptions |
| Ansible role: install and configure Ollama | 🔲 | systemd service management · resource constraints · model pull on first run |
| Ollama role idempotent and re-runnable | 🔲 | |
| Prometheus scrape config addition | 🔲 | Confirm which metrics Ollama exposes natively |
| Grafana dashboard: request rate · latency · uptime · token throughput | 🔲 | |
| Uptime Kuma HTTP check on Ollama API endpoint | 🔲 | |
| Promtail config: ship Ollama systemd logs to Loki | 🔲 | Same pattern as other services |
| Alertmanager rule: inference endpoint unavailable or error rate exceeded | 🔲 | Fits existing routing conventions — no new routing trees |
| Webhook integration: Alertmanager → script → Ollama API → triage output | 🔲 | Phase 4 — highest variance; scope creep risk; flag and defer polish |
| Triage script fails gracefully if Ollama is unavailable | 🔲 | No dependency loop |
| Triage script committed to repo with runbook entry | 🔲 | |
| `docs/capstone/ai-ops-integration.md` written | 🔲 | Framed as AIOps-adjacent infrastructure work, not ML engineering |
| ADR: local inference vs. external API | 🔲 | Rationale: cost · data locality · operational control · learning value |
| 2027 Kubernetes evolution path documented in portfolio doc | 🔲 | Ollama on k3s + ArgoCD — note intended direction without over-engineering now |

---

## Deferred / Out of Scope

| Item | Reason |
|---|---|
| Jaeger (full deployment) | Deferred to M10+ — tracing value is low until multi-service request paths exist; all-in-one container planned for Monitoring VM once AWS mirror is live |
| Kubernetes | Complexity not justified for service count — see ADR-001 |
| Ansible AWX / Tower | Overkill for single-operator homelab |
| HashiCorp Vault | `ansible-vault` sufficient for this scope |
| Multi-region AWS | Not needed to demonstrate the core skills |
| Internal TLS / local CA | Browser warnings on Proxmox, TrueNAS, PBS, and Grafana are acceptable during build; required before app01 services (Nextcloud etc.) are in use — see ADR-019 |

---

## Notes

- Mark items ✅ only after verifying, not just after completing setup
- Update **Last updated** date when changing this doc
- If a planned item is dropped, move it to Deferred with a reason — don't delete it
- Docs marked 🔄 are written ahead of the build — update them as each milestone completes

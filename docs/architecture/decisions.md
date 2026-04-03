# Architecture Decisions

Lightweight decision log. Each entry documents a choice, the reasoning, and what was rejected.

---

## ADR-001 — No Kubernetes

**Decision:** Docker Compose only. No Kubernetes.

**Reasoning:**  
The service count does not justify the operational overhead of a container orchestrator. Running two or three containers on a host is a solved problem with Compose. Adding Kubernetes would shift time toward cluster maintenance and away from the platform skills this lab is designed to demonstrate.

**Rejected:** k3s, k0s, microk8s

**Revisit when:** The platform needs multi-host scheduling, rolling deployments without downtime, or more than ~10 containers across services.

---

## ADR-002 — Monitoring VM is separate from Utility VM

**Decision:** Prometheus, Grafana, Alertmanager, Loki, and Uptime Kuma run on a dedicated VM on pve2.

**Reasoning:**  
Observability should not share a failure domain with the services it monitors. If the Utility VM (pve1) goes down, the Monitoring VM (pve2) continues running and alerting. This separation also allows the Monitoring VM to migrate between Proxmox hosts independently.

**Rejected:** Running the monitoring stack alongside Pi-hole and the monitoring target on the Utility VM.

---

## ADR-003 — Ansible manages all hosts from a dedicated Automation VM

**Decision:** Ansible execution runs from the Automation VM on pve2. The Fedora workstation is used for local development only and does not run applies against live infrastructure.

**Reasoning:**  
A dedicated automation host separates code development from execution against live infrastructure. The Automation VM runs 24/7 and can be triggered by CI, scheduled jobs, or manual runs without requiring a developer workstation to be available. It is itself managed by Ansible, demonstrating the pattern of treating automation infrastructure as code. Placing it on pve2 means it survives a pve1 failure.

**Rejected:** Running all Ansible execution from the developer workstation (no always-on host, no CI integration path).

**Note:** The Fedora workstation is used for writing playbooks, testing syntax locally, and pushing to Git. It is not in the Ansible inventory.

---

## ADR-004 — Terraform S3 backend with native S3 locking

**Decision:** Remote state in S3, state locking via native S3 locking (`use_lockfile = true`). No DynamoDB table.

**Reasoning:**
Local state files get lost with disk failures or laptop wipes. Remote state in S3 means the AWS infrastructure definition survives any local machine loss. Native S3 locking (introduced in Terraform 1.10, stable in 1.14+) stores a `.tflock` file in S3 alongside the state, preventing concurrent applies from corrupting state — without the overhead of provisioning and paying for a separate DynamoDB table. This is the current industry standard for S3 backends on Terraform ≥ 1.10.

**Rejected:** Local `.tfstate` file; DynamoDB locking (superseded by native S3 locking as of Terraform 1.10 — an unnecessary dependency for new deployments on current Terraform versions).

**Setup:** [`terraform/backend/backend-notes.md`](../../terraform/backend/backend-notes.md)

---

## ADR-005 — EC2 instance uses same Debian base as local VMs

**Decision:** EC2 AMI is Debian, matching the base OS on Proxmox VMs and the EliteDesk.

**Reasoning:**  
Using the same OS family means Ansible roles don't need conditional branching for package managers or init systems. A single `baseline.yml` runs correctly everywhere. It also makes troubleshooting consistent — behavior on EC2 should match behavior locally.

**Rejected:** Amazon Linux 2 (diverges from local OS, requires separate role branches).

---

## ADR-006 — Pi-hole runs in Docker, not dedicated VMs

**Decision:** Pi-hole runs as a Docker container — on the Utility VM (primary) and on the HP EliteDesk (secondary). Not in dedicated VMs.

**Reasoning:**  
Pi-hole does not warrant dedicated VM overhead on either host. Running it in Docker means it is managed identically on both: same Compose file, same Ansible role, same restart policy. The two-instance model provides DNS failover without additional infrastructure complexity.

**DNS failover model:** Router DHCP distributes both IPs. Utility VM is primary; EliteDesk is secondary. Devices fail over automatically. The EliteDesk instance is not activated as a DNS server until the whole-home cutover (Milestone 8).

**Trade-off:** Both instances share their host's failure domain. Acceptable — DNS failure is recoverable quickly via router fallback to a public resolver.

---

## ADR-007 — HP Elitedesk 800 mini as QDevice and bare-metal utility node

**Decision:** The HP EliteDesk serves two roles: Corosync QDevice for the Proxmox cluster, and bare-metal utility node running the same Docker services as the Utility VM.

**Reasoning:**  
A 2-node Proxmox cluster cannot achieve quorum majority on its own when one node fails. A QDevice provides a lightweight third vote without requiring a full third Proxmox node. The EliteDesk runs `corosync-qnetd` — a single package — which does not interfere with any other workloads on the machine.

The EliteDesk also serves as the bare-metal mirror of the Utility VM: it runs Pi-hole and the nginx monitoring target via Docker, managed by the same Ansible roles. This demonstrates role portability across bare metal and VMs, which is an explicit architectural invariant of this project.

The EliteDesk does not run Proxmox and is not a cluster node. It is a non-VM host in the Ansible inventory.

**Rejected:**
- Dedicated third Proxmox node for quorum (unnecessary hardware cost)
- Single-vote override (`expected_votes=1`) — bypasses the safety mechanism; less realistic for learning purposes
- EliteDesk as control plane for Ansible/Terraform execution — replaced by dedicated Automation VM (see ADR-003)

---

## ADR-008 — Grafana dashboards and Uptime Kuma config stored as JSON in Git

**Decision:** All Grafana dashboard definitions and Uptime Kuma configuration are exported as JSON and committed to the repository.

**Reasoning:**  
Config built only in the UI is lost when the Monitoring VM is rebuilt. Storing JSON in Git means both are reproducible and versioned. Grafana mounts dashboard JSON automatically via a provisioning directory on startup. Uptime Kuma config is imported post-restore.

**Rejected:** Manual recreation of dashboards and monitors after restores.

---

## ADR-009 — Automation VM for Ansible and Terraform execution

**Decision:** A dedicated Automation VM on pve2 handles all Ansible and Terraform execution against live infrastructure.

**Reasoning:**  
Separating execution from development provides a stable, always-on automation host that can be triggered by CI or scheduled jobs. It also means the EliteDesk does not need to serve as a control plane, keeping its roles clean (QDevice + bare-metal utility node). The Automation VM is placed on pve2 so it remains available if pve1 fails. It is managed by Ansible itself.

**Bootstrap:** The Automation VM is the one exception to the rule that Terraform provisions all Proxmox VMs. It must be provisioned manually — Terraform cannot provision the machine it runs from. All other Proxmox VMs go through Terraform.

**Workflow:** The Fedora workstation is used for development and pushes to Git. CI or the Automation VM handles execution against live environments.

**Rejected:** EliteDesk as the sole execution host (too many responsibilities, not always available for scheduled runs).

---

## ADR-010 — Uptime Kuma added alongside Prometheus for endpoint monitoring

**Decision:** Uptime Kuma runs on the Monitoring VM as a dedicated availability and endpoint monitoring tool, complementing Prometheus rather than replacing it.

**Reasoning:**  
Prometheus is optimized for time-series metrics and requires exporters on each target. Endpoint availability monitoring — HTTP health checks, DNS probes, status pages — is operationally simpler with a purpose-built tool. Uptime Kuma provides a clean status page UI and availability history without requiring PromQL or exporter configuration for simple up/down checks.

**Scope:** Uptime Kuma monitors the nginx target (Utility VM, EliteDesk, and EC2), Pi-hole DNS endpoint (Utility VM and EliteDesk), Grafana availability, and external sites via DNS resolution checks against upstream resolvers.

**Config backup:** Uptime Kuma configuration is exported as JSON and stored in the repository, consistent with ADR-008.

**Rejected:** Prometheus Blackbox Exporter as the sole availability monitoring solution — more complex to configure and provides no status page without additional tooling.

---

## ADR-011 — AWS cost management: estimate before provisioning, budget alert before apply

**Decision:** A cost estimate is documented before the first `terraform apply` against AWS. An AWS Budget alert is configured before the CI/CD pipeline is permitted to run `terraform apply`.

**Reasoning:**  
The CI/CD pipeline introduced at Milestone 5 includes a `terraform apply` stage gated by manual approval. Once AWS resources are introduced at Milestone 10, every approved apply has a cost implication. Establishing a budget alert before that point ensures unexpected charges trigger a notification rather than a surprise on the billing statement. For a lab environment this is a small absolute cost, but the practice itself — estimate first, alert second, provision third — is a real CloudOps habit worth building.

**Cost baseline (estimated at time of build):**

| Resource | Estimated monthly cost |
|---|---|
| EC2 t3.micro (on-demand, us-east-1) | ~$8–10 |
| S3 state bucket (minimal data) | < $0.01 |
| Data transfer (light scraping traffic) | < $1 |
| **Total estimate** | **~$9–11/mo** |

> Verify current pricing at [aws.amazon.com/pricing](https://aws.amazon.com/pricing) before provisioning — on-demand rates change.

**Budget alert:** AWS Budgets alert configured at $10/month, notifying before the estimated ceiling is reached. Threshold reviewed if architecture changes.

**Teardown habit:** Run `terraform destroy` on the `aws-dev` environment when not actively using it for development or testing. The environment is fully reproducible from code — there is no reason to leave it running between sessions.

**Rejected:** No cost controls (acceptable risk for a lab, but not a good habit to build).

---

## ADR-012 — NAS as PBS datastore host via NFS

**Decision:** PBS writes its datastore to an NFS share hosted on the NAS rather than to local Proxmox storage.

**Reasoning:**
Storing the PBS datastore on the NAS means VM backups receive ZFS checksumming and snapshot protection automatically. It also makes the NAS the single aggregation point for the 3-2-1 backup chain — both personal data and VM backups flow through one device to offsite cold storage. This approach is host-independent: the PBS VM on pve1 can be migrated or restored without losing access to its datastore.

**Pool:** The NFS share is hosted on the 3x 8TB RAIDZ1 pool — see ADR-016. ZFS checksumming and snapshot protection apply to all PBS datastore writes.

**Rejected:** Dedicated disk on a Proxmox node (no ZFS protection, outside the 3-2-1 chain, host-dependent).

---

## ADR-014 — Availability as primary SLI class

**Decision:** SLIs for this platform are defined using service availability (ratio of successful observations to total observations), not latency or throughput.

**Reasoning:**
The services in this platform are internal utilities — Pi-hole DNS, nginx monitoring target, Prometheus. For these services, the operationally meaningful failure mode is up/down, not slow. A DNS query that takes 500ms instead of 5ms is not a platform event; Pi-hole not responding is. Availability SLIs are also more tractable: they use `up` metrics and HTTP probe results that are already being collected by the existing stack without additional instrumentation.

Latency SLIs would require histogram metrics and percentile queries — meaningful for request-serving services, but not the primary concern for the utilities this lab runs.

**Rejected:** Latency-based SLIs (e.g., p99 DNS response time). Would require additional instrumentation and introduces complexity without commensurate signal for these service types.

**Revisit when:** A service is added where latency directly affects user experience (e.g., an API server, a proxied application).

---

## ADR-015 — 30-day rolling window for SLO measurement

**Decision:** All SLOs are measured over a 30-day rolling window, not a calendar month.

**Reasoning:**
A rolling window gives a consistent, always-current view of reliability. Calendar months create a discontinuity at month boundaries: an outage on the last day of February is gone from the SLO calculation on March 1, regardless of its severity. A 30-day rolling window means the error budget reflects the last 30 days at all times, making burn rate alerts and dashboard panels meaningful on any day.

Rolling windows are also the approach used by Google SRE and Prometheus-native SLO tooling — aligning with industry practice makes this implementation more directly comparable to production SRE work.

**Trade-off:** Rolling windows are slightly harder to explain verbally ("99.5% over the last 30 days, measured continuously") than calendar months ("99.5% in March"). This is acceptable for a portfolio project where the implementation matters more than the reporting boundary.

**Rejected:** Calendar month window — boundary artifacts make burn rate signals misleading in the final days of the month.

---

## ADR-016 — TrueNAS data pool: 3x 8TB RAIDZ1

**Decision:** Build the NAS data pool with 3x 8TB HDDs in a RAIDZ1 configuration. No interim SSD data pool. Drives purchased 2026-03-28.

**Hardware:** Terramaster F4-424 Pro
- Boot drive: 1x 1TB SSD (TrueNAS OS + boot environments only)
- Data pool: 3x 8TB HDD — RAIDZ1 (~16TB usable; 1-drive fault tolerance)
- Bay 4: empty — available for future expansion

**Why 3x 8TB instead of the previously planned 4x ~6TB:**
- 3x 8TB delivers ~16TB usable at comparable or lower cost per usable TB vs. 4x 6TB (~12TB usable)
- Fewer drives to manage; simpler RAIDZ1 vdev
- 8TB drives are readily available at normalized pricing — no reason to wait for 6TB to drop further
- RAIDZ1 with 3 drives is sufficient for this backup workload; a 4th drive can be added as a second vdev later if capacity demands it

**Pool characteristics:**
- RAIDZ1: survives 1-drive failure; data intact during resilver of a failed drive
- ~16TB usable across PBS datastore + personal data + app persistent volumes
- ZFS checksumming provides data integrity on all writes

**Dataset hierarchy:** Must be designed before initial pool creation — ZFS datasets are significantly harder to reorganize after data is present. Specific record sizes matter (e.g., PBS datastore benefits from `recordsize=16K`). Design documented as part of the TrueNAS install pre-req.

**Rejected:**
- Interim single-drive SSD data pool (originally planned as Phase 1) — bypassed entirely now that HDDs are in hand
- 4x 6TB RAIDZ1 (fewer usable TB, more drives for marginal redundancy gain at this scale)
- Leaving PBS datastore on Proxmox local storage (outside the 3-2-1 chain)

---

## ADR-017 — TrueNAS configuration managed by Terraform and Ansible

**Decision:** After initial pool creation (manual), TrueNAS configuration above the pool layer is managed as code where provider support exists. Terraform owns dataset and snapshot definitions; Ansible handles NFS share configuration and day-2 tasks.

**Provider:** `deevus/truenas` (Terraform Registry: `registry.terraform.io/providers/deevus/truenas`). Uses SSH + `midclt` rather than the REST API. The previously cited `dariusbakunas/truenas` provider was archived October 2025 — TrueNAS deprecated the REST APIs it relied on.

**Scope split:**

| Layer | Tool | Covers |
|---|---|---|
| OS install, network config, pool creation | Manual | One-time; not reproducible via IaC — documented in install pre-req |
| Datasets, snapshot schedules | Terraform | `deevus/truenas` provider |
| NFS/SMB shares, user accounts | Ansible | TrueNAS REST API or `midclt` — `deevus/truenas` does not manage shares |
| Ongoing config drift, state verification, day-2 tasks | Ansible | Same |

**Reasoning:** Everything above pool creation is reproducible from code. This is a strong operational story: IaC applied to bare-metal storage infrastructure, not just cloud resources. The Terraform-owns-definitions / Ansible-handles-operations split mirrors the rest of the platform. The share gap is a provider limitation, not a design choice — Ansible covers it cleanly.

**Rejected:** Manual TrueNAS UI configuration only (not reproducible, no version history, no drift detection); `dariusbakunas/truenas` (archived, depends on deprecated REST APIs).

---

## ADR-013 — DS212 decommissioned from backup chain (superseded 2026-03-28)

**Decision:** The Synology DS212 is removed from the backup chain. The backup architecture simplifies to: PBS → nas01 (NFS datastore) → Backblaze B2 (offsite cold).

**Reasoning:**
The DS212 (~2012 hardware, ~2TB usable) and DS207 (~2008 hardware, ~1TB drives) are both beyond expected drive lifespan and holding personal media data with no offsite copy. The right resolution is to migrate that data to `tank/personal` on nas01 and back it up to Backblaze B2 — not to repurpose aging hardware as a backup target. The capacity mismatch (2TB vs. 16TB usable on nas01) means the DS212 could only ever hold a subset of data anyway, limiting its value as a recovery option. Backblaze B2 provides the offsite tier that the DS212 was intended to complement; with B2 in place, the DS212 adds minimal protection at the cost of operational complexity.

**Rejected (original ADR-013 rationale):** Scoping DS212 to PBS datastore only — superseded by hardware age and the cleaner NAS → B2 path.

---

## ADR-018 — Dedicated apps VM separate from util01

**Decision:** Self-hosted personal applications run on a dedicated `app01` VM, not on `util01`.

**Reasoning:**
`util01` is a platform infrastructure host — it runs Pi-hole (DNS for the whole network) and the nginx monitoring target. Mixing personal apps onto the same VM creates an unacceptable failure domain: a misbehaving Nextcloud or Paperless container could exhaust resources and take down DNS. Keeping `app01` separate means personal app failures are isolated from platform services, and the operational story is clean — `util01` is infrastructure, `app01` is personal productivity.

`util01` is also mirrored on `qdev01` (bare metal) for DNS failover. That mirror would be unnecessarily complicated if it had to replicate a full personal app stack.

**Apps on app01:** Linkding · Wallabag · Navidrome · Paperless · Audiobookshelf · Nextcloud · Netbox

**Rejected:** Running all containers on `util01` (conflates platform and personal workloads; complicates the bare-metal mirror; resource contention risk).

---

## ADR-019 — Internal TLS deferred until before app01

**Decision:** Do not implement a local CA or internal TLS during Milestones 0–10. Treat browser warnings on Proxmox, TrueNAS, PBS, and Grafana as acceptable during the build phase. Implement a local CA before `app01` services are brought up.

**Reasoning:**
Internal admin UIs (Proxmox, TrueNAS, PBS, Grafana) are single-operator, LAN-only, and accessed infrequently. The operational overhead of a local CA is not justified during the build phase when these are the only HTTPS surfaces. Browser warnings are a minor inconvenience, not a security risk in this context.

`app01` services change the calculus: Nextcloud requires HTTPS to function correctly, and services like Paperless and Wallabag behave better with it. A local CA becomes necessary before `app01` is usable.

**Implementation (when scheduled):**
- Run `step-ca` (Smallstep) as a lightweight CA — suitable for a single VM or container
- Install the local CA root cert in the OS/browser trust store on all client devices
- Issue certs for Proxmox, TrueNAS, PBS, Grafana, and each `app01` service
- Once root is trusted on client devices, all internally issued certs are trusted automatically

**Rejected:** Purchasing public TLS certs for internal hostnames (unnecessary cost and complexity for `.lab` domain); leaving all services on self-signed certs permanently (unacceptable for Nextcloud and other app01 services).

---

## ADR-021 — TrueNAS dataset hierarchy

**Decision:** The `tank` pool is organized into three top-level branches: `backups/`, `apps/`, and `personal/`. Datasets under `apps/` are created at app deployment time — not pre-provisioned.

**Hierarchy:**

```
tank/
├── backups/
│   └── pbs/          recordsize=16K  — PBS datastore
├── apps/             ← one dataset per app, created at deploy time
└── personal/
    ├── music/
    ├── photos/
    └── movies/
```

**ZFS properties by branch:**

| Dataset | recordsize | compression | snapshots |
|---|---|---|---|
| `tank/backups/pbs` | 16K | lz4 | No — PBS manages its own retention |
| `tank/apps/<name>` | 128K (default) | lz4 | Yes — daily, keep 14 |
| `tank/personal/*` | 1M | lz4 | Yes — daily, keep 30 |

**Key decisions:**

- **One dataset per app** — snapshot and quota granularity is per-app; no shared volume between containers
- **No pre-provisioning under `apps/`** — a dataset is created when the app is deployed, not before; Terraform adds a new dataset resource at deploy time
- **No quotas at initial setup** — add quotas on specific apps (e.g. Nextcloud) once real usage data is available
- **NFS shares follow dataset boundaries** — each `apps/<name>` dataset gets its own NFS share, mounted on `app01` as a bind mount source

**Reasoning:**
Per-dataset structure gives snapshot and quota granularity at the app level without locking in a fixed app list upfront. ZFS pool free space is shared dynamically — datasets grow as needed. Pre-creating empty datasets for apps that may never be deployed adds no value and creates cleanup overhead.

**Rejected:** Single `tank/apps` share with subdirectories per app — loses per-app snapshot and quota capability; pre-creating all app datasets regardless of deployment status — unnecessary, harder to keep clean.

---

## ADR-020 — TrueNAS boot pool mirrored across two P310 SSDs

**Decision:** Use both 1TB P310 SSDs as a ZFS mirror for the TrueNAS boot pool.

**Reasoning:**
A second P310 was originally purchased as temporary data storage before the 3x 8TB HDDs were committed to. With the data pool now planned as RAIDZ1 on HDDs, the second P310 has no role in the data pool. Mirroring the boot pool is the correct use — it costs nothing extra and protects against OS drive failure without requiring a reinstall.

**Rejected:** Using the second P310 as a separate data volume or cache device — unnecessary given the RAIDZ1 data pool covers all storage requirements.

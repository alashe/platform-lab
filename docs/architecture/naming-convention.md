# Naming Convention

Standards for all identifiers in the platform: hostnames, VM IDs, storage pools, datasets, services, and cloud resource names/tags.

This document is written at Milestone 0. Apply these names at first configuration — retroactive renaming of established resources (e.g., Proxmox node names post-cluster-init) requires cluster teardown and is out of scope unless explicitly planned.

---

## Guiding Principles

- **Lowercase, hyphen-separated** — consistent with DNS and Linux naming rules
- **Role-based, not hardware-based** — names reflect function, not make/model
- **Zero-padded sequences** — `01`, `02` scales cleanly without renaming
- **Short** — hostnames under 12 characters; tooling, logs, and Ansible inventories stay readable
- **No environment prefix for local** — local hosts live under `.lab`; cloud resources carry an env prefix

---

## Hostnames

### Pattern

```
{role}{seq}           local hosts
{env}-{role}{seq}     cloud hosts
```

### Physical — Proxmox Nodes

| Hostname | Hardware | Role |
|---|---|---|
| `pve01` | Dell Precision 7550 | Proxmox host — primary compute |
| `pve02` | Dell Precision 7560 | Proxmox host — secondary compute |

> Proxmox node names are set at cluster init and cannot be changed without tearing down and rebuilding the cluster. Set to `pve01` / `pve02` at initial install — do not use `pve1` / `pve2`.

### Physical — Bare-Metal (Non-Proxmox)

| Hostname | Hardware | Role |
|---|---|---|
| `qdev01` | HP EliteDesk 800 mini | Corosync QDevice · bare-metal utility node |

### Virtual Machines

| Hostname | VM ID | Assigned Node | Role |
|---|---|---|---|
| `util01` | 101 | pve01 | Docker runtime — Pi-hole primary · nginx target |
| `mon01` | 102 | pve02 | Monitoring — Prometheus · Grafana · Alertmanager · Loki · Uptime Kuma |
| `pbs01` | 103 | pve01 | Proxmox Backup Server |
| `auto01` | 104 | pve02 | Ansible execution · Terraform workspace |
| `app01` | 105 | pve01 | Self-hosted personal apps — Docker Compose services |
| `llm01` | 106 | TBD at M12 | LLM inference service — Ollama · AI capstone (M12) |

> Node assignments are defaults. Adjust at build time based on resource availability. The exception is `pbs01` on `pve01` — confirmed per architecture doc.

### Non-Platform VMs (retained — not Ansible/Terraform managed)

| Hostname | VM ID | Role |
|---|---|---|
| `win01` | 111 | Retained Windows 11 Pro VM from pre-platform setup |

> This VM coexists on the cluster but is outside the platform's automation scope. It is assigned an ID and IP from the non-platform range to avoid conflicts with managed infrastructure.

### Storage Devices

| Hostname | Device | Role |
|---|---|---|
| `nas01` | Terramaster F4-424 Pro | Primary NAS — TrueNAS Scale · PBS datastore via NFS · personal data |

### Cloud — AWS

| Identifier | Resource | Role |
|---|---|---|
| `aws-web01` | EC2 t3.micro | Monitoring target — nginx · node_exporter |

---

## IP Address Plan

Subnet: `<LAB_IP>/24` · Gateway: `<LAB_IP>`

### Range Assignments

| Range | Group | Notes |
|---|---|---|
| `.1` | Gateway | Router (TP-Link AX1500) |
| `.2–.9` | Network infrastructure | Switch, AP, other network gear |
| `.10–.49` | Reserved | Do not assign |
| `.51–.59` | Proxmox nodes + bare metal | Static — set before cluster init |
| `.61–.69` | VMs | Static — assigned at provision time |
| `.81–.89` | Storage devices | Static |
| `.100–.253` | DHCP pool | Dynamically assigned by router |

### Static Assignments

| Host | IP | Group |
|---|---|---|
| `pve01` | `<LAB_IP>` | Proxmox nodes |
| `pve02` | `<LAB_IP>` | Proxmox nodes |
| `qdev01` | `<LAB_IP>` | Proxmox nodes / bare metal |
| `util01` | `<LAB_IP>` | VMs |
| `mon01` | `<LAB_IP>` | VMs |
| `pbs01` | `<LAB_IP>` | VMs |
| `auto01` | `<LAB_IP>` | VMs |
| `app01` | `<LAB_IP>` | VMs |
| `llm01` | `<LAB_IP>` | VMs |
| `win01` | `<LAB_IP>` | VMs (non-platform) |
| `nas01` | `<LAB_IP>` | Storage |

> Last octet matches the host sequence number within its group (e.g., `pve01=.51`, `pve02=.52`). This makes addresses immediately readable without a lookup.

> Set static IPs on the router as DHCP reservations by MAC address, **and** configure the IP statically on the host itself. Both are required — DHCP reservation alone does not guarantee the host uses the correct address before first contact with the router.

---

## DNS — Local Domain

Local domain: `.lab`

DNS A records are managed as Pi-hole local DNS entries. Activated at Milestone 8.

| FQDN | Host |
|---|---|
| `pve01.lab` | pve01 |
| `pve02.lab` | pve02 |
| `qdev01.lab` | qdev01 |
| `util01.lab` | util01 |
| `mon01.lab` | mon01 |
| `pbs01.lab` | pbs01 |
| `auto01.lab` | auto01 |
| `app01.lab` | app01 |
| `llm01.lab` | llm01 |
| `nas01.lab` | nas01 |

---

## Proxmox VM IDs

| VM ID | Hostname | Role |
|---|---|---|
| 101 | `util01` | Utility VM |
| 102 | `mon01` | Monitoring VM |
| 103 | `pbs01` | PBS VM |
| 104 | `auto01` | Automation VM |
| 105 | `app01` | Apps VM |
| 106 | `llm01` | LLM Inference VM (M12) |
| 111 | `win01` | Retained Windows 11 Pro VM (non-platform) |

Reserved ranges:

| Range | Purpose |
|---|---|
| 100 | Reserved — do not assign |
| 101–109 | Core platform VMs |
| 110–119 | Non-platform / retained VMs |
| 120–199 | Future VMs |
| 9000+ | VM templates |

---

## Storage — ZFS Pools and Datasets (nas01)

### Pool Names

| Pool | Drives | Capacity | Status |
|---|---|---|---|
| `tank` | 3× 8TB HDD RAIDZ1 | ~16TB usable | Drives purchased — pool pending creation |

### Dataset Hierarchy

**`tank` pool:**

```
tank/
├── pbs         PBS datastore (NFS export to pbs01)
├── personal    Documents · photos · music
└── apps        Persistent volumes for app01 services
```

### ZFS Dataset Properties

| Dataset | `recordsize` | Reason |
|---|---|---|
| `*/pbs` | `16K` | PBS chunk size — improves write performance |
| `*/personal` | `128K` (default) | General-purpose large files |
| `*/apps` | Set at creation | Depends on workload |

> Set `recordsize` before writing data to a dataset. Changing it afterward does not rewrite existing blocks.

### NFS Export (PBS datastore)

| NAS export path | Mount point on pbs01 | Proxmox storage ID |
|---|---|---|
| `/mnt/tank/pbs` | `/mnt/pbs-nfs` | `pbs-nfs` |

---

## Services and Containers

Two VMs host services. They have distinct purposes and are kept separate:

| VM | Purpose | Examples |
|---|---|---|
| `util01` | Platform infrastructure services | Pi-hole · nginx monitoring target |
| `app01` | Self-hosted personal applications | Linkding · Wallabag · Navidrome · Paperless · Audiobookshelf · Nextcloud · Netbox |

`util01` is also mirrored on `qdev01` (bare metal) for DNS failover. `app01` has no bare-metal mirror.

See ADR-001 for the conditions under which the Docker Compose model is revisited.

### Container and Compose Service Names

| Pattern | Example | Used for |
|---|---|---|
| `{service}` | `linkding`, `navidrome`, `nextcloud` | Docker Compose service name and container name |

- Lowercase, no hyphens — matches how the service is commonly referred to
- The Compose project name matches the service name (set via `COMPOSE_PROJECT_NAME` or `--project-name`)
- One `docker-compose.yml` per service, stored under `ansible/roles/{service}/files/`

### Service DNS Names

Services get a named DNS entry in Pi-hole separate from the host they run on. This decouples the service address from the host — if a service moves to a different VM, only the DNS record changes.

| Pattern | Example | Resolves to |
|---|---|---|
| `{service}.lab` | `navidrome.lab` | IP of the VM hosting the service |

Service DNS entries:

| FQDN | Service | Host |
|---|---|---|
| `pihole.lab` | Pi-hole web UI | `util01` |
| `linkding.lab` | Bookmark manager | `app01` |
| `wallabag.lab` | Read-later | `app01` |
| `navidrome.lab` | Music streaming | `app01` |
| `paperless.lab` | Document management | `app01` |
| `audiobookshelf.lab` | Audiobook / podcast server | `app01` |
| `nextcloud.lab` | File sync and personal cloud | `app01` |
| `netbox.lab` | Network and infrastructure documentation | `app01` |

> DNS entries are managed as Pi-hole local DNS records. Add a row when a new service is deployed.

### When a New VM Is Needed

If a future service requires its own VM (isolation, resource, or failure domain requirement), assign it from the `110–199` VM ID range and use the `svc{seq}` hostname pattern:

| Hostname | VM ID | Example use |
|---|---|---|
| `svc01` | 110 | First dedicated single-service VM |
| `svc02` | 111 | Second, and so on |

This is not expected for current scope — document the reason in `decisions.md` when it happens.

---

## AWS Resource Tagging

All Terraform-provisioned AWS objects carry these tags:

| Tag | Value |
|---|---|
| `Name` | Resource name (e.g., `aws-web01`) |
| `Env` | `aws-dev` |
| `Project` | `platform-lab` |
| `ManagedBy` | `terraform` |

---

## Related Docs

- [Architecture overview](platform-lab.md)
- [Access control](access-control.md)
- [Hardening baseline](hardening-baseline.md)
- [Architecture decisions](decisions.md)
- [Milestone plan](homelab-milestone-plan.md)

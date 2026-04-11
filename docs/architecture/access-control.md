# Access Control

User account standards for all hosts in the platform. Defines which accounts exist on each host, how they are provisioned, and how access is granted and revoked.

SSH hardening settings (ciphers, timeouts, daemon config) are in [hardening-baseline.md](hardening-baseline.md). This document covers identity — who gets in and how.

---

## Accounts

### `ops` — Automation and Operations Account

The standard human-and-machine account on all Ansible-managed Linux hosts.

| Property | Value |
|---|---|
| Username | `ops` |
| Shell | `/bin/bash` |
| Sudo | `ALL=(ALL) NOPASSWD:ALL` |
| Auth | SSH public key only — password login disabled |
| Created by | Ansible `baseline` role |

This is the account Ansible connects as for all configuration management and the account used for day-to-day SSH operations. It is the only account with persistent passwordless sudo on managed hosts.

Applied to: `util01`, `mon01`, `pbs01`, `auto01`, `qdev01`, `aws-web01`.

---

### `pveadmin` — Proxmox Web UI Admin

Proxmox PAM user for web UI and API access on the hypervisors. Not a Linux shell account.

| Property | Value |
|---|---|
| Username | `pveadmin` |
| Type | Proxmox PAM user |
| Proxmox role | `Administrator` |
| Auth | Strong password · TOTP recommended |
| Created by | Manual — Proxmox web UI or `pveum` |

Applied to: `pve01`, `pve02`.

> After `pveadmin` is confirmed working, disable `root` login to the Proxmox web UI under Datacenter → Permissions → Users. Root should only be used via the physical console.

---

### `root` — Break-Glass Only

| Property | Value |
|---|---|
| Hosts | All Linux hosts |
| SSH login | Disabled — `PermitRootLogin no` enforced by Ansible baseline |
| Console | Physical console access only |
| Stored | Strong, unique password in password manager — never in repo |

Root is not used for routine operations. Its password is set at OS install and stored offline.

---

### `pbs01` — Proxmox Backup Server

PBS manages its own authentication realm (`@pbs`), separate from the Linux host accounts.

| Account | Created by | Purpose |
|---|---|---|
| `root@pam` | PBS install | Web UI and appliance administration — break-glass only |
| `pve@pbs` | Manual — `proxmox-backup-manager` (see `pbs01-setup.md` step 4) | Proxmox VE integration account — authenticates with password, not token |

`pve@pbs` should have `DatastoreAudit` and `DatastoreBackup` on `/datastore/tank-pbs`. `DatastoreAudit` lets Proxmox VE enumerate and inspect the datastore; `DatastoreBackup` allows backup writes. PBS storage is defined at Datacenter level in Proxmox and shared across all nodes. Store the password in the password manager.

---

### `nas01` — TrueNAS Scale

TrueNAS Scale manages users through its own system, not Linux PAM. Ansible manages the `ops` account on `nas01` via the `arensb.truenas` collection.

| Account | Created by | Purpose |
|---|---|---|
| `truenas_admin` | TrueNAS setup wizard | Web UI and appliance administration |
| `ops` | Ansible `truenas` role (bootstrap: one-time manual creation) | Ansible automation account |

Use a strong, unique password for `truenas_admin`. Store in a password manager, not in the repo.

---

## Account Matrix

| Host | `ops` | `pveadmin` | Appliance admin | Notes |
|---|---|---|---|---|
| `pve01` | — | Yes | — | Proxmox PAM only; `ops` not needed on hypervisor |
| `pve02` | — | Yes | — | Same |
| `qdev01` | Yes | — | — | Ansible-managed bare metal |
| `util01` | Yes | — | — | Ansible-managed VM |
| `mon01` | Yes | — | — | Ansible-managed VM |
| `pbs01` | Yes | — | `root@pam` | Ansible-managed VM; PBS web UI auth separate — see PBS accounts below |
| `auto01` | Yes | — | — | Ansible-managed VM; all Ansible plays run as `ops` |
| `nas01` | Yes (TrueNAS) | — | `truenas_admin` | `ops` managed via `arensb.truenas`; bootstrap: manual creation in TrueNAS UI |
| `aws-web01` | Yes | — | — | Ansible-managed EC2; default cloud user superseded by `ops` |

---

## SSH Key Management

All SSH authentication uses key pairs. Password authentication is disabled on all managed hosts by the Ansible baseline.

| Item | Standard |
|---|---|
| Key type | `ed25519` (preferred) or `rsa` 4096-bit minimum |
| Key location (control) | `~/.ssh/` on the Fedora workstation and `auto01` |
| Authorized keys deployment | Ansible `baseline` role — `authorized_keys` in `group_vars` or `host_vars` |
| Per-host keys | Not required — one ops key pair is sufficient for this platform scale |
| Key rotation | Rotate by updating the authorized key in Ansible vars and re-running `baseline.yml` |

> The private key for `ops` is never committed to the repository. Store it on the control machine only.

---

## Credential Storage

| Credential type | Storage |
|---|---|
| SSH private keys | Control machine filesystem only — never in repo |
| Service passwords (grafana, etc.) | Ansible Vault encrypted vars |
| Root passwords | Password manager (offline) |
| Proxmox `pveadmin` password | Password manager |
| NAS admin passwords | Password manager |
| AWS credentials | Environment variables or `~/.aws/credentials` — never in repo |
| PBS `pve@pbs` password | Password manager |

---

## Granting and Revoking Access

**Granting access:**
1. Add the public key to the appropriate `authorized_keys` entry in Ansible `group_vars` or `host_vars`
2. Run `baseline.yml` — key is deployed to all targeted hosts

**Revoking access:**
1. Remove the public key from Ansible vars
2. Run `baseline.yml` — key is removed from `~/.ssh/authorized_keys` on all targeted hosts
3. Verify with `ssh-keygen -F <host>` and a manual SSH attempt

> Ansible manages the `authorized_keys` file declaratively. Keys not present in Ansible vars will be removed on the next baseline run if `exclusive: true` is set on the `authorized_key` task.

---

## Related Docs

- [Hardening baseline](hardening-baseline.md)
- [Naming convention](naming-convention.md)
- [Architecture overview](platform-lab.md)
- [Architecture decisions](decisions.md)

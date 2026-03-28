# Hardening Baseline

Minimum security configuration enforced on all Ansible-managed Linux hosts: `util01`, `mon01`, `pbs01`, `auto01`, `qdev01`, `aws-web01`.

This baseline is implemented by the Ansible `baseline` role. Re-running `baseline.yml` in check mode detects drift.

User accounts and SSH key management are in [access-control.md](access-control.md). This document covers daemon config, firewall, and package requirements.

---

## SSH Daemon Configuration

Applied via Ansible to `/etc/ssh/sshd_config` on all managed hosts.

| Setting | Value | Reason |
|---|---|---|
| `PasswordAuthentication` | `no` | Key-only auth — eliminates brute-force surface |
| `PermitRootLogin` | `no` | Root access via console only |
| `PubkeyAuthentication` | `yes` | Required — only auth method permitted |
| `ClientAliveInterval` | `300` | Disconnect idle sessions after 5 minutes |
| `ClientAliveCountMax` | `2` | Two missed keepalives before disconnect |
| `AllowUsers` | `ops` | Restrict SSH to the `ops` account only |
| `Protocol` | `2` | SSHv1 disabled |

> Changes to `sshd_config` require `sshd` reload. The Ansible task notifies a handler to reload automatically.

---

## Required Packages

Installed on all managed hosts by the `baseline` role.

| Package | Purpose |
|---|---|
| `curl` | HTTP utility — health checks, script fetches |
| `vim` | Editor — consistent across all hosts |
| `ufw` | Firewall — default deny, allowlist per host |
| `fail2ban` | Brute-force protection — SSH jail enabled by default |
| `unattended-upgrades` | Automatic security patch application |
| `ca-certificates` | TLS root CA bundle — required for HTTPS package sources |
| `gnupg` | GPG — required for verifying package signing keys |

---

## Firewall — ufw

Default policy applied to all managed hosts. Per-host port allowances are defined in `host_vars`.

| Rule | Value |
|---|---|
| Default incoming | `deny` |
| Default outgoing | `allow` |
| SSH (22) | `allow` — all managed hosts |

Additional ports opened per host:

| Host | Port | Service |
|---|---|---|
| `util01` | 80, 443 | nginx |
| `util01` | 53 | Pi-hole DNS |
| `mon01` | 3000 | Grafana |
| `mon01` | 9090 | Prometheus |
| `mon01` | 3100 | Loki |
| `pbs01` | 8007 | PBS web UI |
| All hosts | 9100 | node_exporter (Prometheus scrape) |
| `qdev01` | 5405 | Corosync QDevice (`corosync-qnetd`) |
| `aws-web01` | 80 | nginx |
| `aws-web01` | 9100 | node_exporter |

> AWS security groups enforce the same allowlist at the network layer. `ufw` on EC2 is a second layer, not a replacement.

---

## fail2ban

Default configuration applied by the `baseline` role.

| Setting | Value |
|---|---|
| Jail | `sshd` — enabled on all managed hosts |
| `bantime` | `1h` |
| `findtime` | `10m` |
| `maxretry` | `5` |
| Action | `ufw` ban (drops packets; no ICMP rejection) |

---

## Unattended Upgrades

| Setting | Value |
|---|---|
| Origins | `Debian-Security` (security patches only) |
| Auto-reboot | `false` — reboots are manual and scheduled |
| Reboot time | N/A |
| Notification email | Not configured — patch status visible via Prometheus node metrics |

> Security patches are applied automatically. Kernel upgrades requiring a reboot are queued and applied during planned maintenance.

---

## Docker Hosts

Applies to hosts running Docker: `util01`, `mon01`, `auto01`, `qdev01`, `aws-web01`.

| Item | Standard |
|---|---|
| Runtime | Docker CE (not Docker Desktop) |
| Install source | Official Docker apt repository |
| Compose | Docker Compose plugin (`docker compose`) — not standalone `docker-compose` |
| User group | `ops` added to `docker` group — avoids `sudo` for Docker commands |
| Daemon socket | Not exposed over TCP — Unix socket only |

> The `docker` group grants root-equivalent access to the host. Only `ops` is in this group.

---

## Compliance Verification

Drift is detected by re-running `baseline.yml` in check mode:

```bash
ansible-playbook ansible/playbooks/baseline.yml --check
```

A clean run with no changes confirms the host matches the baseline. Any reported changes indicate drift.

Run after:
- OS updates that may modify `sshd_config` or package defaults
- Any manual changes made directly on a host
- Suspected unauthorized access

---

## Related Docs

- [Access control](access-control.md)
- [Naming convention](naming-convention.md)
- [Architecture overview](platform-lab.md)
- [Ansible reference](../../ansible/README.md)

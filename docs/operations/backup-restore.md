# Backup & Restore

Backup architecture, schedules, verification, and restore procedures.

---

## Backup Chain

```
Proxmox VMs (scheduled snapshots)
        ↓
Proxmox Backup Server  (PBS VM)
        ↓
NAS  (warm storage · local network)
        ↓
DS212 (warm local copy · separate physical device)  +  Backblaze B2 (offsite cold)
```

> **Note:** The PBS → NAS link is not a sync job. The PBS datastore is configured to write directly to an NFS share exposed by the NAS. Data written to PBS lands on NAS-hosted ZFS storage in real time. The DS212 receives a nightly rsync of the PBS datastore only (not all NAS data), providing a warm local VM backup copy on a separate physical device.

---

## Recovery Targets

| Metric | Target | Last Verified |
|---|---|---|
| RTO — full VM restore from PBS | < 30 minutes | — |
| RPO — maximum data loss | 24 hours | — |

---

## Backup Schedule

| Job | Source | Destination | Schedule | Retention |
|---|---|---|---|---|
| VM snapshot — Utility VM | Proxmox | PBS | Daily 02:00 | 7 daily / 4 weekly |
| VM snapshot — Monitoring VM | Proxmox | PBS | Daily 02:15 | 7 daily / 4 weekly |
| VM snapshot — PBS VM | Proxmox | PBS | Weekly Sun 03:00 | 4 weekly |
| NAS → DS212 (PBS datastore rsync) | NAS | DS212 | Nightly | mirrors PBS retention |
| NAS → Backblaze B2 (cold) | NAS | Backblaze B2 | Weekly | 90 days |
| Terraform state | S3 backend | Versioned automatically | On every apply | S3 versioning |
| Ansible / Terraform code | Git | Remote repository | On every push | Full history |

---

## What's Covered

| Asset | Method | Location |
|---|---|---|
| Utility VM | PBS snapshot | PBS datastore → NAS (NFS, always-on) → DS212 (warm local rsync) + Backblaze B2 (offsite cold) |
| Monitoring VM | PBS snapshot | PBS datastore → NAS (NFS, always-on) → DS212 (warm local rsync) + Backblaze B2 (offsite cold) |
| Grafana dashboards | JSON in Git | Repository |
| Prometheus rules | Config in Git | Repository |
| Ansible playbooks / roles | Git | Repository |
| Terraform modules + state | Git + S3 versioned | Repository + S3 |
| Pi-hole config/lists | Ansible role re-run | Git |

---

## Restore Procedures

### Restore a VM from PBS

1. Log into Proxmox web UI
2. Navigate to **PBS** storage → **Backups**
3. Select the VM backup to restore
4. Click **Restore** — choose target node and storage
5. Start the restored VM
6. Verify services are running:

```bash
ssh <vm>
docker ps
systemctl status docker
```

7. If restoring the Utility VM, verify Pi-hole is responding:

```bash
# From another host on the network
dig @<utility-vm-ip> google.com
```

8. Log actual restore time in the table above.

---

### Restore Monitoring VM

After restoring from PBS snapshot, dashboards and config should be intact.  
If restoring from scratch (e.g., new VM), re-provision with Ansible:

```bash
# Provision fresh Monitoring VM, then:
ansible-playbook -i inventories/homelab playbooks/monitoring.yml
```

Grafana dashboards are provisioned automatically from JSON files in the role.

---

### Rebuild AWS EC2 from scratch

If the EC2 instance is lost entirely:

```bash
# 1. Reprovision infrastructure
cd terraform/environments/aws-dev
terraform apply

# 2. Update inventory with new public IP
terraform output public_ip
# Edit ansible/inventories/aws/hosts

# 3. Rerun configuration
cd ansible
ansible-playbook -i inventories/aws playbooks/baseline.yml
ansible-playbook -i inventories/aws playbooks/docker.yml
ansible-playbook -i inventories/aws playbooks/monitoring-target.yml

# 4. Verify service is running and visible in Prometheus
curl http://<new-ip>:80
# Check Prometheus UI → Targets
```

Expected time: under 15 minutes.

---

### Recover Terraform state (if S3 bucket is lost)

1. Recreate the S3 bucket and DynamoDB table (see `terraform/backend/backend-notes.md`)
2. Run `terraform import` for each existing resource to rebuild state
3. Run `terraform plan` — should show no changes if import was correct

This is a rare scenario. Document steps if it ever occurs.

---

## Backup Verification

### Monthly restore drill

Pick one VM. Restore it to a test VMID on the same Proxmox host.  
Start it, verify services, log the actual RTO. Delete the test VM.

```bash
# From Proxmox CLI
qmrestore /path/to/backup.vma <new-vmid> --storage local-lvm
qm start <new-vmid>
```

### Backup job health

Alertmanager fires a `BackupJobFailed` alert if a PBS job exits non-zero.  
Check PBS job logs in the PBS UI: **Datastore → Jobs → last run**.

### Verify NAS datastore and DS212 rsync

The PBS datastore lives on a NAS NFS share — no sync job to check. Verify the NFS mount is active and PBS is writing successfully via the PBS UI (Datastore → Show Content).

```bash
# Confirm DS212 rsync completed and backup file dates match expected schedule
ls -lh /mnt/nas/backups/
```

> **DS212 scope:** The DS212 receives a nightly rsync of the PBS datastore only — not all NAS data. Personal data (documents, photos, music) is covered by NAS → Backblaze B2 directly. The DS212 provides a warm local VM backup copy on a separate physical device, recoverable without the NAS or Backblaze B2.

---

## Notes

- PBS deduplication means incremental snapshots are storage-efficient — don't skip daily backups to save space
- Pi-hole blocklists are pulled fresh on container start; no backup needed for list data
- `ansible-vault` encrypted secrets are committed to Git — vault password must be stored securely and separately from the repo

# Backup & Restore

> **Status: Written ahead of build (Milestone 2).** `pbs01` has been provisioned and the PBS datastore path is in place. The remaining live-build work is backup-job configuration plus end-to-end validation. Verify schedules, restore paths, and RTO/RPO targets against live infrastructure when Milestone 2 is complete and remove this notice.
>
> **Review note:** Re-review this document after Milestone 2 completion. Planned commands or playbooks should either exist in-repo by then or be replaced with the implemented procedure.

Backup architecture, schedules, verification, and restore procedures.

---

## Backup Chain

```
Proxmox VMs (scheduled backups)
        ↓
Proxmox Backup Server  (PBS VM)
        ↓
NAS  (warm storage · local network · RAIDZ1)
        ↓
Backblaze B2  (offsite cold)
```

> **Note:** The PBS → NAS link is not a sync job. The PBS datastore is configured to write directly to an NFS share exposed by the NAS. Data written to PBS lands on NAS-hosted ZFS storage in real time.

---

## Recovery Targets

| Metric | Target | Last Verified |
|---|---|---|
| RTO — full VM restore from PBS | < 30 minutes | Populate after first successful restore drill (Milestone 2) |
| RPO — maximum data loss | 24 hours | Populate after daily backup schedule is confirmed running (Milestone 2) |

---

## Backup Schedule

| Job | Source | Destination | Schedule | Retention |
|---|---|---|---|---|
| VM backup — util01 | Proxmox | PBS | Daily 02:00 | 7 daily / 4 weekly |
| VM backup — mon01 | Proxmox | PBS | Daily 02:15 | 7 daily / 4 weekly |
| VM backup — pbs01 | Proxmox | Separate non-PBS storage | Weekly Sun 03:00 | 4 weekly |
| VM backup — auto01 | Proxmox | PBS | Weekly Sun 03:15 | 4 weekly |
| NAS → Backblaze B2 (cold) | NAS | Backblaze B2 | Weekly | 90 days |
| Terraform state | S3 backend | Versioned automatically | On every apply | S3 versioning |
| Ansible / Terraform code | Git | Remote repository | On every push | Full history |

---

## What's Covered

| Asset | Method | Location |
|---|---|---|
| `util01` | PBS backup | PBS datastore → NAS (NFS, always-on) + Backblaze B2 (offsite cold) |
| `mon01` | PBS backup | PBS datastore → NAS (NFS, always-on) + Backblaze B2 (offsite cold); disk lives on `nfs-shared` — restore target must be `nfs-shared` or `local-lvm` |
| `pbs01` | Proxmox backup to separate storage | Separate non-PBS Proxmox backup target; do not rely on the same PBS instance as the only recovery path for `pbs01` |
| `auto01` | PBS backup | PBS datastore → NAS (NFS, always-on) |
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

7. If restoring `util01`, verify Pi-hole is responding:

```bash
# From another host on the network
dig @<utility-vm-ip> google.com
```

8. Log actual restore time in the table above.

---

### Restore mon01

After restoring from PBS backup, dashboards and config should be intact.  
If restoring from scratch (e.g., new VM), re-provision with Ansible:

```bash
# Provision fresh Monitoring VM, then:
ansible-playbook -i inventories/homelab playbooks/monitoring.yml
```

`playbooks/monitoring.yml` is planned, not currently present in-repo. Re-validate this command after the monitoring automation lands.

Grafana dashboards are provisioned automatically from JSON files in the role.

### Restore pbs01

Restore `pbs01` from its separate non-PBS Proxmox backup target, not from `pbs-tank`.

If the backup target is a standard Proxmox VM backup archive, restore it with the normal VM restore flow for that storage backend. After restore:

1. Confirm the PBS VM boots cleanly
2. Confirm the NAS-mounted datastore path is reachable
3. Confirm the PBS UI loads
4. Confirm the `pbs-tank` datastore content is visible

---

### Rebuild AWS EC2 from scratch

If the EC2 instance is lost entirely:

```bash
# 1. Reprovision infrastructure
cd terraform/environments/aws-dev
terraform apply

# 2. Update inventory with new public IP
terraform output public_ip
# Edit ansible/inventories/aws/hosts.ini

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

1. Recreate the S3 bucket and native S3 locking configuration (see `terraform/backend/backend-notes.md`)
2. Run `terraform import` for each existing resource to rebuild state
3. Run `terraform plan` — should show no changes if import was correct

This is a rare scenario. Document steps if it ever occurs.

---

## Backup Verification

### Monthly restore drill

Pick one VM. Restore it to a test VMID on the same Proxmox host.  
Start it, verify services, log the actual RTO. Delete the test VM.

For PBS-backed VMs, use the Proxmox web UI restore flow:

1. Proxmox UI → PBS storage → Backups
2. Select the backup
3. Click **Restore**
4. Choose the test VMID, target node, and target storage
5. Start the restored VM and validate services

### Backup job health

Alertmanager fires a `BackupJobFailed` alert if a PBS job exits non-zero.  
Check PBS job logs in the PBS UI: **Datastore → Jobs → last run**.

### Verify NAS datastore

The PBS datastore lives on a NAS NFS share — no sync job to check. Verify the NFS mount is active and PBS is writing successfully via the PBS UI (Datastore → Show Content).

---

## Notes

- PBS deduplication means incremental backups are storage-efficient — don't skip daily backups to save space
- Pi-hole blocklists are pulled fresh on container start; no backup needed for list data
- `ansible-vault` encrypted secrets are committed to Git — vault password must be stored securely and separately from the repo

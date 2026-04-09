# Incident Scenarios

> **Status: Written ahead of build (Milestone 11).** These scenarios assume a fully running platform with monitoring active. Complete Milestones 1–10 before running drills. Remove this notice when Milestone 11 begins.

Practice exercises for operational troubleshooting. Each scenario maps to a real interview story.

Run these deliberately. Take notes. Time yourself. Write a brief post-mortem afterward.

---

## How to Use This

For each exercise:
1. Set up the failure condition
2. Respond as if it's real — use monitoring, not prior knowledge
3. Resolve and verify
4. Write 3–5 sentences: what failed, how you detected it, what you did, how long it took
5. Note any follow-up actions (alerting improvements, runbook updates)

---

## Scenario 1 — Service Down

**Story:** The monitoring target service stops responding. An alert fires.

**Setup:**
```bash
ssh util01
docker stop <nginx_container>
```

**What to practice:**
- Detect the alert in Alertmanager before checking manually
- Confirm via Prometheus: `up{job="nginx_target"}` returns 0
- Check container logs: `docker logs <container> --tail 50`
- Restore: `docker start <container>` or `docker compose up -d`
- Verify Prometheus target recovers (state: UP)
- Note time-to-alert vs time-to-restore

**Interview angle:** *"Tell me about a time a service went down. How did you detect it?"*

---

## Scenario 2 — Terraform Destroy and Rebuild

**Story:** The AWS environment is gone. Rebuild from code.

**Setup:** `terraform destroy` (aws-dev environment)

**What to practice:**
```bash
cd terraform/environments/aws-dev
terraform apply

# Note new public IP, update inventory
terraform output public_ip

# Reprovision
cd ansible
ansible-playbook -i inventories/aws playbooks/baseline.yml
ansible-playbook -i inventories/aws playbooks/docker.yml
ansible-playbook -i inventories/aws playbooks/monitoring-target.yml
```

**Target:** Full rebuild under 15 minutes.

**Verify:**
- EC2 instance running
- Ansible baseline idempotent (no changes on second run)
- Prometheus scraping new instance
- nginx responding on port 80

**Interview angle:** *"How would you rebuild a lost environment from scratch?"*

---

## Scenario 3 — Configuration Drift

**Story:** Someone manually changed SSH config on a host. Ansible needs to detect and correct it.

**Setup:**
```bash
ssh util01
sudo sed -i 's/PermitRootLogin no/PermitRootLogin yes/' /etc/ssh/sshd_config
sudo systemctl restart sshd
```

**What to practice:**
```bash
# Detect drift (check mode — no changes)
ansible-playbook -i inventories/homelab playbooks/baseline.yml --check --diff

# Correct it
ansible-playbook -i inventories/homelab playbooks/baseline.yml

# Verify
ansible util01 -i inventories/homelab -m command \
  -a "sshd -T | grep permitrootlogin"
```

**Interview angle:** *"How do you handle configuration drift? How do you enforce compliance?"*

---

## Scenario 4 — HA Failover: mon01 Auto-Migration

**Story:** `pve02` fails unexpectedly. Proxmox HA automatically migrates `mon01` to `pve01` — no operator action required. Verify the automatic recovery and confirm there is no monitoring gap.

**Setup:**
```bash
# Simulate node failure: shut down pve02 hard (power button or Proxmox UI → Stop)
# This is a real disruption — confirm no critical work is running on pve02 first
```

**What to practice:**
- Watch Datacenter → HA → Resources — `mon01` should show `started` on `pve01` within ~30 seconds
- Watch Grafana — verify no data gap during migration
- Confirm Prometheus continues scraping all targets post-migration
- Confirm `mon01` returns to `pve02` when `pve02` recovers

**Verify:**
```bash
ssh pve01
pvesh get /cluster/ha/status/current   # mon01 should show: status=started, node=pve01
```

**Interview angle:** *"How do you design for availability? What happens when a host fails?"*

---

## Scenario 5 — Backup Restore Drill

**Story:** `util01` is lost. Restore it.

**Setup:** Note the current VMID. The "failure" is simulated by restoring to a new VMID.

**What to practice:**
```bash
# In PBS UI or Proxmox CLI:
# Restore backup to new VMID on same (or different) host
qmrestore <backup_path> <new_vmid> --storage local-lvm
qm start <new_vmid>
```

**Verify:**
- Pi-hole responding: `dig @<restored-vm-ip> google.com`
- nginx monitoring target running: `curl http://<restored-vm-ip>:80`
- Docker running: `docker ps`

**Record actual RTO.** Compare to target (< 30 min). If over, identify why.

**Interview angle:** *"What's your disaster recovery process? Have you actually tested it?"*

---

## Scenario 6 — Disk Space Alert

**Story:** A disk space warning fires on `util01`.

**Setup:**
```bash
ssh util01
# Create a large file to trigger the threshold
dd if=/dev/zero of=/tmp/bigfile bs=1M count=5000
```

**What to practice:**
- Identify the alert in Alertmanager
- Confirm via Prometheus: `node_filesystem_avail_bytes`
- Find the space: `df -h`, `du -sh /var/lib/docker/*`
- Resolve: `rm /tmp/bigfile`, `docker system prune -f`
- Verify alert clears within next scrape interval

**Interview angle:** *"Walk me through how you'd respond to a disk space alert."*

---

## Scenario 7 — Ansible Compliance Audit

**Story:** Run a read-only compliance check across all hosts before a change window.

**What to practice:**
```bash
# Audit — no changes made
ansible-playbook -i inventories/mixed playbooks/baseline.yml --check --diff

# Review output:
# - "changed" lines indicate drift
# - "ok" lines indicate compliance
# - Save output to a file for review
ansible-playbook -i inventories/mixed playbooks/baseline.yml --check --diff \
  2>&1 | tee audit-$(date +%Y%m%d).txt
```

**Verify:** All hosts return "ok" with no diff output (assuming no drift).

**Interview angle:** *"How do you verify your systems are in the expected state before making changes?"*

---

## Post-Mortem Template

After each exercise, write this up. Keep it brief.

```
Date:
Scenario:
Detection method:  (alert / manual / monitoring)
Time to detect:
Time to resolve:
Root cause:
Resolution:
Follow-up actions:  (alert rule improvement, runbook update, etc.)
```

## Post-Mortem Storage

Save each completed post-mortem as a file in `docs/operations/post-mortems/`.

Naming convention: `YYYY-MM-DD-<scenario-slug>.md`

Example: `2026-11-02-service-down-nginx.md`

Commit each post-mortem to Git after writing it. The commit history serves as the audit trail — date, scenario, and any follow-up actions are all preserved. Review accumulated post-mortems before Milestone 11 (Reliability Drills) to identify patterns and drive alerting or runbook improvements.

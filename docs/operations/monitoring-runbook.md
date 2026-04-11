# Monitoring Runbook

> **Status: Written ahead of build (Milestone 7).** Procedures described here assume the full observability stack is deployed and running. Verify against live infrastructure when Milestone 7 is complete and remove this notice.

Reference for the observability stack: what's running, how to use it, and how to respond to alerts.

---

## Stack

| Component | Role | Host |
|---|---|---|
| Prometheus | Metrics collection and alerting rules | `mon01` |
| Grafana | Dashboards and visualization | `mon01` |
| Alertmanager | Alert routing and silencing | `mon01` |
| Loki | Log aggregation | `mon01` |
| Uptime Kuma | Endpoint availability monitoring and status page | `mon01` |
| Promtail | Log shipping agent | All local hosts |
| node_exporter | Host metrics | All hosts (local + EC2) |
| cAdvisor | Container metrics | `util01`, `qdev01` |
| Pi-hole FTL exporter | DNS metrics | `util01`, `qdev01` |
| CloudWatch agent | EC2 logs | EC2 instance |

---

## Access

| Service | URL |
|---|---|
| Grafana | `http://mon01:3000` |
| Prometheus | `http://mon01:9090` |
| Alertmanager | `http://mon01:9093` |
| Uptime Kuma | `http://mon01:3001` |

---

## Uptime Kuma

Uptime Kuma provides endpoint availability monitoring and a status page. It is the first place to check for service up/down status before investigating metrics.

**Monitored targets:**

| Monitor | Type | Endpoint |
|---|---|---|
| nginx target — util01 | HTTP | `http://util01:80/healthz` |
| nginx target — qdev01 | HTTP | `http://qdev01:80/healthz` |
| nginx target — AWS EC2 | HTTP | `http://<ec2-ip>:80/healthz` |
| Pi-hole DNS — util01 | DNS | UDP port 53 on util01 |
| Pi-hole DNS — qdev01 | DNS | UDP port 53 on qdev01 |
| Grafana | HTTP | `http://mon01:3000` |
| External DNS check | DNS | Upstream resolver — confirms Pi-hole is forwarding correctly |

**Configuration backup:** Uptime Kuma config is exported as JSON and stored in the repository. Re-import after a restore.

---

## Dashboards

| Dashboard | What it shows |
|---|---|
| Node Health — All Environments | CPU · memory · disk · uptime for all hosts |
| Container Overview | Per-container CPU · memory · restart count |
| Pi-hole DNS Analytics | Query volume · block rate · top domains |
| Cross-Environment Status | Side-by-side homelab vs EC2 service health |

Dashboards are provisioned from JSON in `ansible/roles/monitoring/files/dashboards/`. This lands with the monitoring automation at Milestone 7.  
Changes to dashboards should be exported and committed back to the repo.

---

## Alerts

### Active alert rules

| Alert | Condition | Severity |
|---|---|---|
| `ServiceDown` | Target unreachable for > 2m | critical |
| `HostHighCPU` | CPU > 85% for > 5m | warning |
| `HostHighMemory` | Memory > 90% for > 5m | warning |
| `HostDiskFull` | Disk > 85% used | warning |
| `ContainerRestarting` | Restart count increased in 10m | warning |
| `BackupJobFailed` | PBS backup job exit != 0 | critical |
| `NodeExporterDown` | Exporter target missing | warning |

Uptime Kuma sends its own notifications directly (email or webhook) for endpoint availability events, separate from Alertmanager.

### Alert routing

Alerts route to a notification channel (email or webhook) configured in `alertmanager.yml`.  
Critical alerts fire immediately. Warnings group with a 5-minute wait.

---

## Triage Procedures

### Service unreachable — Uptime Kuma alert

```bash
# 1. Check Uptime Kuma dashboard first — confirms which endpoint is down
# http://mon01:3001

# 2. Check container status on the affected host
ssh util01
docker ps -a

# 3. Check container logs
docker logs <container_name> --tail 50

# 4. Check if port is responding
curl -I http://localhost:80/healthz

# 5. Restart if needed
docker compose -f /opt/services/<service>/docker-compose.yml restart

# 6. Verify Uptime Kuma shows recovery and Prometheus target returns UP
```

### Service unreachable — Prometheus alert (no Uptime Kuma alert)

If Prometheus fires but Uptime Kuma does not, the HTTP endpoint is likely up but the metrics exporter is down. Check node_exporter or cAdvisor specifically rather than the service container.

### High memory warning

```bash
# Check per-process memory on host
ssh <host>
ps aux --sort=-%mem | head -20

# Check container memory usage
docker stats --no-stream

# Check for memory leak pattern in Grafana
# Container Overview dashboard → select container → zoom to alert window
```

### Disk space warning

```bash
# Find large directories
ssh <host>
df -h
du -sh /var/lib/docker/*    # check for large images/volumes
docker system df            # summarize Docker disk usage

# Clean up unused Docker objects
docker system prune -f      # safe: removes stopped containers, unused images
```

### Backup job failed alert

```bash
# Check PBS job history
# PBS UI → Datastore → Jobs → last run status

# Check PBS VM is running
# Proxmox UI → PBS VM → Summary

# Manual backup trigger (if safe to do)
# PBS UI → Datastore → Run now

# See also: docs/operations/backup-restore.md
```

---

## Prometheus Queries — Reference

```promql
# Is a host reachable?
up{job="node_exporter", instance="<host>:9100"}

# CPU usage (1m avg)
100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[1m])) * 100)

# Memory available
node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes * 100

# Disk usage
(node_filesystem_size_bytes - node_filesystem_free_bytes) / node_filesystem_size_bytes * 100

# Container restart count (delta over 10m)
increase(container_restart_count_total[10m])

# Pi-hole blocked queries (%)
pihole_queries_blocked_today / pihole_queries_today * 100
```

---

## Silencing an Alert

Use the Alertmanager UI to create a silence during planned maintenance:

1. Navigate to `http://mon01:9093`
2. Click **New Silence**
3. Add a matcher: `alertname=<AlertName>` (optionally scope to an instance)
4. Set duration and add a comment (reason + your name)
5. Submit

For Uptime Kuma monitors, pause the monitor during maintenance:  
Uptime Kuma UI → select monitor → **Pause**.

Remove silence / unpause when maintenance is complete.

---

## Monitoring Stack Is Down

`mon01` runs on `pve02` with its disk on the `nfs-shared` NFS pool. Proxmox HA (`migrate` policy) live-migrates `mon01` to `pve01` automatically if `pve02` fails — no operator action required, no monitoring gap.

If you are seeing a monitoring outage, work through these cases:

### Case 1 — pve02 node failure (expected: auto-recovery)

Proxmox HA detects the failure and migrates `mon01` to `pve01`. Check HA status:

- Datacenter → HA → Resources — `mon01` should show `started` on `pve01` within ~30 seconds
- Grafana at `http://mon01:3000` should be accessible once migration completes

If HA recovery stalls (status stuck at `recovery` for > 2 minutes):

```bash
# From workstation or auto01 — check cluster and HA status
ssh pve01
pvesh get /cluster/ha/status/current
```

If `mon01` failed to start on `pve01`, start it manually:

```bash
qm start <vmid>   # get vmid from: pvesh get /nodes/pve01/qemu
```

### Case 2 — mon01 Docker stack issue (pve02 healthy, mon01 running, services down)

The VM is up but containers have crashed. Check and recover:

```bash
ssh mon01
docker compose -f /opt/services/monitoring/docker-compose.yml ps
docker compose -f /opt/services/monitoring/docker-compose.yml up -d
docker compose -f /opt/services/monitoring/docker-compose.yml ps   # verify all running
```

Check logs if a container won't start:

```bash
docker logs <container_name> --tail 50
```

### Case 3 — NAS (nas01) unreachable

`mon01`'s disk is on the `nfs-shared` NFS pool hosted on `nas01`. If `nas01` goes offline, `mon01`'s disk becomes unavailable and the VM will freeze or fail.

```bash
# From pve02 — check NFS mount
df -h | grep nfs
ping 192.168.0.81   # nas01
```

Restore NAS connectivity first. Once `nas01` is reachable, `mon01` should recover automatically. If the VM is in an error state:

```bash
# Force stop and restart (from pve02 or pve01)
qm stop <vmid> --skiplock
qm start <vmid>
```

> **Note:** This is the accepted trade-off for NFS-backed VM storage (ADR-023). A NAS failure affects monitoring but not `util01`, `pbs01`, or `auto01`.

---

## Adding a New Scrape Target

1. Add the host to Prometheus scrape config in the `monitoring` role (`roles/monitoring/templates/prometheus.yml.j2`) at Milestone 7
2. Re-run the monitoring playbook once it is implemented at Milestone 7. Planned path: `playbooks/monitoring.yml`
3. Verify the target appears in Prometheus UI → Status → Targets (state: UP)
4. Add corresponding HTTP monitor in Uptime Kuma for the service endpoint
5. Check it appears in the Node Health dashboard within one scrape interval (default: 15s)

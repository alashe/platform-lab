# Monitoring Runbook

Reference for the observability stack: what's running, how to use it, and how to respond to alerts.

---

## Stack

| Component | Role | Host |
|---|---|---|
| Prometheus | Metrics collection and alerting rules | Monitoring VM |
| Grafana | Dashboards and visualization | Monitoring VM |
| Alertmanager | Alert routing and silencing | Monitoring VM |
| Loki | Log aggregation | Monitoring VM |
| Uptime Kuma | Endpoint availability monitoring and status page | Monitoring VM |
| Promtail | Log shipping agent | All local hosts |
| node_exporter | Host metrics | All hosts (local + EC2) |
| cAdvisor | Container metrics | Utility VM |
| Pi-hole FTL exporter | DNS metrics | Utility VM |
| CloudWatch agent | EC2 logs | EC2 instance |

---

## Access

| Service | URL |
|---|---|
| Grafana | `http://monitoring-vm:3000` |
| Prometheus | `http://monitoring-vm:9090` |
| Alertmanager | `http://monitoring-vm:9093` |
| Uptime Kuma | `http://monitoring-vm:3001` |

---

## Uptime Kuma

Uptime Kuma provides endpoint availability monitoring and a status page. It is the first place to check for service up/down status before investigating metrics.

**Monitored targets:**

| Monitor | Type | Endpoint |
|---|---|---|
| nginx target — homelab | HTTP | `http://utility-vm:80/healthz` |
| nginx target — AWS EC2 | HTTP | `http://<ec2-ip>:80/healthz` |
| Pi-hole DNS | DNS | UDP port 53 on utility-vm |
| Grafana | HTTP | `http://monitoring-vm:3000` |
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

Dashboards are provisioned from JSON in `ansible/roles/monitoring/files/dashboards/`.  
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
# http://monitoring-vm:3001

# 2. Check container status on the affected host
ssh utility-vm
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

1. Navigate to `http://monitoring-vm:9093`
2. Click **New Silence**
3. Add a matcher: `alertname=<AlertName>` (optionally scope to an instance)
4. Set duration and add a comment (reason + your name)
5. Submit

For Uptime Kuma monitors, pause the monitor during maintenance:  
Uptime Kuma UI → select monitor → **Pause**.

Remove silence / unpause when maintenance is complete.

---

## Adding a New Scrape Target

1. Add the host to Prometheus scrape config in the `monitoring` role (`roles/monitoring/templates/prometheus.yml.j2`)
2. Re-run the monitoring playbook: `ansible-playbook -i inventories/homelab playbooks/monitoring.yml`
3. Verify the target appears in Prometheus UI → Status → Targets (state: UP)
4. Add corresponding HTTP monitor in Uptime Kuma for the service endpoint
5. Check it appears in the Node Health dashboard within one scrape interval (default: 15s)

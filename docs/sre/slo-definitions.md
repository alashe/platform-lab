# SLO Definitions

Service Level Objectives for the `platform-lab` platform.

> **Scope:** Homelab portfolio implementation. These SLOs demonstrate understanding of the SLO model — they are not production SRE commitments. Targets are calibrated for a single-operator homelab with planned maintenance windows.
>
> **Window:** 30-day rolling window throughout.
> **Authored:** Milestone 7 (Monitoring Layer)
> **Reviewed:** Milestone 11 (Reliability Drills)

---

## SLI-1 — Pi-hole DNS Availability

### What is measured

The fraction of Prometheus scrape cycles in which the Pi-hole FTL exporter on the Utility VM responds successfully. A failed scrape indicates Pi-hole is unavailable or the exporter has crashed.

Pi-hole is the most operationally significant service in the platform — it is the primary DNS resolver for the entire home network. Its failure degrades every device on the network.

### SLI definition

```
SLI = (scrape cycles where up{job="pihole_exporter", instance="utility-vm:9617"} == 1)
    / (total scrape cycles in window)
```

**Prometheus query (30-day availability):**

```promql
avg_over_time(up{job="pihole_exporter", instance=~"utility-vm.*"}[30d])
```

**Source:** Prometheus scrapes the Pi-hole FTL exporter (`pihole_exporter`) on port 9617. Scrape interval: 30s (configured in `prometheus.yml`).

### SLO target

**99.5% availability over a 30-day rolling window.**

### Error budget

| Metric | Value |
|---|---|
| Window | 30 days = 43,200 minutes |
| Error budget | 0.5% × 43,200 = **216 minutes (~3.6 hours)** |
| Budget rate | ~7.2 minutes per day |

> **Why 99.5% and not 99.9%:** A single Proxmox host reboot or VM live migration takes 5–10 minutes. 99.9% over 30 days allows only 43 minutes total. Planned maintenance alone would exhaust that budget. 99.5% is honest.

### Alertmanager rule — burn rate

File: `prometheus/rules/slo-pihole.yml`

```yaml
groups:
  - name: slo_pihole_availability
    rules:
      # Fast burn: 1h window, 14.4× burn rate → budget exhausted in ~2 days
      - alert: PiholeSLOFastBurn
        expr: |
          (1 - avg_over_time(up{job="pihole_exporter"}[1h]))
          / (1 - 0.995) > 14.4
        for: 5m
        labels:
          severity: critical
          slo: pihole_availability
        annotations:
          summary: "Pi-hole error budget burning fast"
          description: >
            1h error rate exceeds 14.4× the SLO budget.
            At this rate, the 30-day error budget (216 min) would be
            exhausted in less than 2 days.

      # Slow burn: 6h window, 3× burn rate → budget exhausted in ~10 days
      - alert: PiholeSLOSlowBurn
        expr: |
          (1 - avg_over_time(up{job="pihole_exporter"}[6h]))
          / (1 - 0.995) > 3
        for: 15m
        labels:
          severity: warning
          slo: pihole_availability
        annotations:
          summary: "Pi-hole error budget draining"
          description: >
            6h error rate exceeds 3× the SLO budget.
            Budget is depleting faster than the sustained rate allows.
```

### Grafana panel spec

**Panel type:** Gauge
**Title:** `Pi-hole — Error Budget Remaining (30d)`

**Query:**

```promql
(avg_over_time(up{job="pihole_exporter"}[30d]) - 0.995)
/ (1 - 0.995) * 100
```

**Interpretation:** 100% = full budget available. 0% = SLO violated. Negative = in breach.
**Thresholds:** Green > 50%, Yellow 10–50%, Red < 10%.

---

## SLI-2 — nginx Monitoring Target Availability

### What is measured

The fraction of Uptime Kuma HTTP probes that return HTTP 200 from the nginx monitoring target on the Utility VM. The nginx target is the simplest meaningful HTTP endpoint in the platform — it exists specifically to be monitored.

This SLI spans the homelab environment. When the EC2 instance is active (Milestone 10), the Uptime Kuma EC2 monitor provides a parallel signal that is tracked separately.

### SLI definition

```
SLI = (probes where nginx returns HTTP 200 on Utility VM)
    / (total probes in window)
```

**Source:** Uptime Kuma probes nginx on the Utility VM every 60 seconds. Uptime Kuma exposes a Prometheus-compatible `/metrics` endpoint scraped by Prometheus.

**Prometheus query (availability, approximated from Uptime Kuma metrics):**

```promql
avg_over_time(monitor_status{monitor_name="nginx-utility-vm"}[30d])
```

> `monitor_status` is 1 when up, 0 when down. This is a direct SLI approximation.

### SLO target

**99.0% availability over a 30-day rolling window.**

### Error budget

| Metric | Value |
|---|---|
| Window | 30 days = 43,200 minutes |
| Error budget | 1.0% × 43,200 = **432 minutes (~7.2 hours)** |
| Budget rate | ~14.4 minutes per day |

> **Why 99.0% and not 99.5%:** nginx runs as a single Docker container on the Utility VM with no load balancer. A container restart takes ~30 seconds; a VM reboot takes longer. The 99.0% target is calibrated for a single-instance service with no redundancy layer.

### Alertmanager rule — burn rate

File: `prometheus/rules/slo-nginx.yml`

```yaml
groups:
  - name: slo_nginx_availability
    rules:
      - alert: NginxSLOFastBurn
        expr: |
          (1 - avg_over_time(monitor_status{monitor_name="nginx-utility-vm"}[1h]))
          / (1 - 0.990) > 14.4
        for: 5m
        labels:
          severity: critical
          slo: nginx_availability
        annotations:
          summary: "nginx error budget burning fast"
          description: >
            1h error rate is 14.4× the SLO budget allowance.
            30-day error budget (432 min) would be exhausted in under 2 days.

      - alert: NginxSLOSlowBurn
        expr: |
          (1 - avg_over_time(monitor_status{monitor_name="nginx-utility-vm"}[6h]))
          / (1 - 0.990) > 3
        for: 15m
        labels:
          severity: warning
          slo: nginx_availability
        annotations:
          summary: "nginx error budget draining"
          description: >
            6h error rate exceeds 3× the SLO budget. Budget depleting steadily.
```

### Grafana panel spec

**Panel type:** Stat
**Title:** `nginx — Error Budget Remaining (30d)`

**Query:**

```promql
(avg_over_time(monitor_status{monitor_name="nginx-utility-vm"}[30d]) - 0.990)
/ (1 - 0.990) * 100
```

**Unit:** Percent (0–100)
**Thresholds:** Green > 50%, Yellow 10–50%, Red < 10%.

---

## SLI-3 — Prometheus Scrape Reliability

### What is measured

The fraction of 30-second intervals in which Prometheus successfully completes its scrape round — measured as Prometheus's own self-monitoring `up` gauge. If Prometheus is not up, all SLI data collection for SLI-1 and SLI-2 is silently failing.

This SLI is the canary for the observability layer itself.

### SLI definition

```
SLI = (intervals where up{job="prometheus"} == 1)
    / (total intervals in window)
```

**Prometheus query (30-day availability):**

```promql
avg_over_time(up{job="prometheus", instance="localhost:9090"}[30d])
```

**Source:** Prometheus scrapes itself via the `prometheus` job configured in `prometheus.yml`. No exporter required — this is built-in self-monitoring.

### SLO target

**99.5% availability over a 30-day rolling window.**

### Error budget

| Metric | Value |
|---|---|
| Window | 30 days = 43,200 minutes |
| Error budget | 0.5% × 43,200 = **216 minutes (~3.6 hours)** |

### Alertmanager rule

File: `prometheus/rules/slo-prometheus.yml`

```yaml
groups:
  - name: slo_prometheus_reliability
    rules:
      - alert: PrometheusSLOBurn
        expr: |
          (1 - avg_over_time(up{job="prometheus"}[1h]))
          / (1 - 0.995) > 14.4
        for: 5m
        labels:
          severity: critical
          slo: prometheus_reliability
        annotations:
          summary: "Prometheus scrape reliability degraded"
          description: >
            Prometheus self-scrape is failing at a rate that exhausts the
            30-day error budget in under 2 days. Check Monitoring VM health.
```

> **Note:** If Prometheus itself is down, this alert cannot fire — which is why Uptime Kuma monitors Grafana availability independently (from the Uptime Kuma plan at M7). Uptime Kuma serves as the out-of-band check when the metrics pipeline fails.

### Grafana panel spec

**Panel type:** Gauge
**Title:** `Prometheus — Error Budget Remaining (30d)`

**Query:**

```promql
(avg_over_time(up{job="prometheus", instance="localhost:9090"}[30d]) - 0.995)
/ (1 - 0.995) * 100
```

---

## SLI-4 — PBS Availability

### What is measured

The fraction of Prometheus scrape cycles in which the Proxmox Backup Server responds successfully. PBS is the first link in the backup chain — if it is unavailable, no VM snapshots are being taken.

### SLI definition

```
SLI = (scrape cycles where up{job="pbs01"} == 1)
    / (total scrape cycles in window)
```

**Prometheus query (30-day availability):**

```promql
avg_over_time(up{job="pbs01"}[30d])
```

**Source:** Prometheus scrapes the PBS node exporter on pbs01. Scrape interval: 30s.

### SLO target

**99.0% availability over a 30-day rolling window.**

### Error budget

| Metric | Value |
|---|---|
| Window | 30 days = 43,200 minutes |
| Error budget | 1.0% × 43,200 = **432 minutes (~7.2 hours)** |
| Budget rate | ~14.4 minutes per day |

> **Why 99.0%:** PBS runs as a single VM with no redundancy layer. A PBS VM reboot or Proxmox host maintenance takes 5–15 minutes. 99.5% (216 min budget) is too tight for routine operations.

### Alertmanager rule — burn rate

File: `prometheus/rules/slo-pbs.yml`

```yaml
groups:
  - name: slo_pbs_availability
    rules:
      # Fast burn: budget exhausted in ~2 hours at this rate
      - alert: PBSSLOFastBurn
        expr: |
          (1 - avg_over_time(up{job="pbs01"}[1h]))
          / (1 - 0.990) > 14.4
          and
          (1 - avg_over_time(up{job="pbs01"}[5m]))
          / (1 - 0.990) > 14.4
        for: 2m
        labels:
          severity: critical
          slo: pbs_availability
        annotations:
          summary: "PBS burning error budget at {{ $value | humanize }}× — exhausted in ~2h"
          description: >
            1h error rate exceeds 14.4× the SLO budget allowance.
            30-day error budget (432 min) would be exhausted in under 2 hours.

      # Slow burn: budget exhausted in ~10 days at this rate
      - alert: PBSSLOSlowBurn
        expr: |
          (1 - avg_over_time(up{job="pbs01"}[6h]))
          / (1 - 0.990) > 3
          and
          (1 - avg_over_time(up{job="pbs01"}[1h]))
          / (1 - 0.990) > 3
        for: 15m
        labels:
          severity: warning
          slo: pbs_availability
        annotations:
          summary: "PBS error budget draining at {{ $value | humanize }}×"
          description: >
            6h error rate exceeds 3× the SLO budget. Budget depleting steadily —
            exhausted in ~10 days at this rate.
```

### Grafana panel spec

**Panel type:** Gauge
**Title:** `PBS — Error Budget Remaining (30d)`

**Query:**

```promql
(avg_over_time(up{job="pbs01"}[30d]) - 0.990)
/ (1 - 0.990) * 100
```

**Unit:** Percent (0–100)
**Thresholds:** Green > 50%, Yellow 10–50%, Red < 10%.

**Additional panels:**

```promql
# Burn rate over time (time series) — reference line at 1.0 = on pace
(1 - avg_over_time(up{job="pbs01"}[1h])) / (1 - 0.990)

# Budget consumed in minutes (stat)
(1 - avg_over_time(up{job="pbs01"}[30d])) * 30 * 24 * 60
```

---

## Prometheus Recording Rules

Recording rules pre-compute the SLI ratio to reduce query overhead on dashboards. Place in `prometheus/rules/slo-recording.yml`.

```yaml
groups:
  - name: slo_availability_recording
    interval: 1m
    rules:
      # Pi-hole: 5-minute rolling availability window
      - record: job:pihole_up:avg5m
        expr: avg_over_time(up{job="pihole_exporter"}[5m])

      # nginx: 5-minute rolling availability from Uptime Kuma
      - record: monitor:nginx_up:avg5m
        expr: avg_over_time(monitor_status{monitor_name="nginx-utility-vm"}[5m])

      # Prometheus self: 5-minute rolling availability
      - record: job:prometheus_up:avg5m
        expr: avg_over_time(up{job="prometheus", instance="localhost:9090"}[5m])

      # PBS: 5-minute rolling availability
      - record: job:pbs_up:avg5m
        expr: avg_over_time(up{job="pbs01"}[5m])
```

These feed the burn rate alerts and can be referenced in Grafana instead of re-running the full query.

---

## Error Budget Summary

| Service | SLO | 30-day Budget | Honest rationale |
|---|---|---|---|
| Pi-hole DNS | 99.5% | 216 min | Primary DNS; high availability expectation, but VM reboots happen |
| nginx target | 99.0% | 432 min | Single container, no redundancy layer |
| Prometheus | 99.5% | 216 min | Core to all observability; failures are silent and severe |
| PBS | 99.0% | 432 min | Single VM, no redundancy; routine maintenance consumes budget |

---

## References

- ADR-014 — Availability as primary SLI class
- ADR-015 — 30-day rolling window
- Milestone 2 — Backup Architecture (PBS deployed — SLI-4 monitoring begins at M7)
- Milestone 7 — Monitoring Layer (deployment of Prometheus, Grafana, Alertmanager; all four SLOs active)
- Milestone 11 — Reliability Drills (error budget reviewed post-drill for each SLO)

# Platform Diagrams

---

## Platform Architecture

```mermaid
flowchart TD
  subgraph Dev["Development"]
    FED["Fedora Workstation\nAnsible + Terraform dev\nnot cluster-managed"]
    GIT["Git Repository\nplatform-lab"]
  end

  subgraph Proxmox["Homelab — Proxmox Cluster (pve01 · pve02)"]
    subgraph pve01["pve01 — Dell Precision 7550"]
      UVM["Utility VM\nDocker: Pi-hole · nginx target"]
      PBS["PBS VM\nProxmox Backup Server"]
    end
    subgraph pve02["pve02 — Dell Precision 7560"]
      MVM["Monitoring VM\nPrometheus · Grafana · Alertmanager\nLoki · Uptime Kuma"]
      AVM["Automation VM\nAnsible execution\nTerraform workspace"]
    end
  end

  subgraph EliteDesk["HP Elitedesk 800 mini (bare metal)"]
    QD["Corosync QDevice\nquorum tie-breaker"]
    BM["Docker: Pi-hole (secondary DNS)\nnginx target\nAnsible-managed"]
  end

  subgraph Storage["Backup Tiers"]
    NAS["NAS\nwarm storage"]
    COLD["Cold Tier\noffsite / Backblaze B2"]
  end

  subgraph AWS["AWS — Terraform-managed"]
    VPC["VPC + Security Group"]
    EC2["EC2 Instance (t3.micro)\nDocker: nginx target · node_exporter"]
    CW["CloudWatch\nlogs"]
  end

  FED -->|"push"| GIT
  GIT -->|"pull / trigger"| AVM
  AVM -->|"ansible-playbook"| UVM
  AVM -->|"ansible-playbook"| MVM
  AVM -->|"ansible-playbook"| BM
  AVM -->|"ansible-playbook"| EC2
  AVM -->|"terraform apply"| VPC

  MVM -->|"scrapes metrics"| UVM
  MVM -->|"scrapes metrics"| BM
  MVM -->|"scrapes metrics"| EC2
  MVM -->|"endpoint checks"| UVM
  MVM -->|"endpoint checks"| BM
  MVM -->|"endpoint checks"| EC2

  EC2 --> CW
  VPC --> EC2

  UVM --> PBS
  MVM --> PBS
  AVM --> PBS
  PBS --> NAS --> COLD

  QD -.->|"quorum vote"| pve01
  QD -.->|"quorum vote"| pve02
```

---

## Automation Flow

```mermaid
flowchart LR
  DEV["Fedora Workstation\ndevelop + push"]
  GIT["Git / CI"]
  AVM["Automation VM\nexecute"]
  TF["Terraform\nprovision infrastructure"]
  ANS["Ansible\nconfigure hosts + deploy"]
  DOCK["Docker\nrun workloads"]
  MON["Monitoring\nPrometheus · Uptime Kuma\nGrafana · Loki"]
  OPS["Operations\nrunbooks · drills"]

  DEV --> GIT --> AVM
  AVM --> TF --> ANS --> DOCK --> MON --> OPS
```

---

## Pi-hole DNS Failover

```mermaid
flowchart LR
  subgraph Network["Home Network — Router DHCP"]
    R["TP-Link AX1500\nDHCP: primary · secondary DNS"]
  end

  subgraph Primary["Primary DNS"]
    UVM["Utility VM\nPi-hole\npve01"]
  end

  subgraph Secondary["Secondary DNS (failover)"]
    ED["HP EliteDesk\nPi-hole\nbare metal"]
  end

  R -->|"primary"| UVM
  R -->|"secondary / failover"| ED
```

> Both instances are deployed via the same Ansible role. The EliteDesk instance is activated as secondary DNS during the whole-home cutover (Milestone 8).

---

## Observability Layer

```mermaid
flowchart LR
  subgraph Targets["Monitored Targets"]
    UVM["Utility VM\nnginx · Pi-hole"]
    ED["EliteDesk\nnginx · Pi-hole"]
    EC2["AWS EC2\nnginx target"]
    EXT["External\nDNS upstream"]
  end

  subgraph Stack["Monitoring VM (pve02)"]
    PROM["Prometheus\nmetrics + rules"]
    UK["Uptime Kuma\nendpoint checks"]
    LOKI["Loki\nlogs"]
    AM["Alertmanager\nalert routing"]
    GRAF["Grafana\ndashboards"]
  end

  UVM -->|"node_exporter · cAdvisor · FTL"| PROM
  ED -->|"node_exporter · FTL"| PROM
  EC2 -->|"node_exporter"| PROM
  UVM -->|"Promtail"| LOKI
  ED -->|"Promtail"| LOKI
  EC2 -->|"CloudWatch"| LOKI

  UK -->|"HTTP · DNS probes"| UVM
  UK -->|"HTTP · DNS probes"| ED
  UK -->|"HTTP probes"| EC2
  UK -->|"DNS check"| EXT

  PROM --> AM
  PROM --> GRAF
  UK --> GRAF
  LOKI --> GRAF
```

---

## Backup and Recovery

```mermaid
flowchart TD
  VMs["VMs: Utility · Monitoring · Automation"]
  PBS["Proxmox Backup Server\npve01"]
  NAS["NAS\nwarm storage"]
  COLD["Cold Tier\noffsite / Backblaze B2"]

  VMs -->|"scheduled backup"| PBS
  PBS -->|"datastore writes"| NAS
  NAS -->|"periodic copy"| COLD

  PBS -->|"restore"| VMs
  COLD -->|"restore drill"| VMs
```

---

## CI/CD Pipeline

```mermaid
flowchart LR
  PR["Pull Request"]
  FMT["terraform fmt -check"]
  VAL["terraform validate"]
  PLAN["terraform plan"]
  APPROVE["Manual Approval"]
  APPLY["terraform apply"]
  SYNTAX["ansible --syntax-check"]

  PR --> FMT --> VAL --> PLAN --> APPROVE --> APPLY
  PR --> SYNTAX
```

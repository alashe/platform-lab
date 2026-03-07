
flowchart LR
  subgraph Repo["Git Repo: IaC + Config + Docs"]
    TF["Terraform: AWS provisioning"]
    ANS["Ansible: baseline + deploy"]
    DC["Docker Compose: service defs"]
    DOC["Docs: diagram + runbooks + game days"]
  end

  subgraph Proxmox["Homelab: Proxmox Cluster (7550 + 7560)"]
    UVM["Utility VM (Debian)<br/>Docker: Pi-hole + Target Service"]
    MVM["Monitoring VM (Debian)<br/>Prometheus + Grafana + Alertmanager<br/>(Loki optional)"]
    PBS["PBS VM<br/>Proxmox Backup Server"]
  end

  subgraph Storage["Backup Tiers"]
    NAS["NAS Tier"]
    COLD["Cold Tier (DS207 / offline)"]
  end

  subgraph AWS["AWS (Terraform-managed)"]
    VPC["VPC + Subnet"]
    SG["Security Group"]
    EC2["EC2 Instance<br/>Docker: Target Service"]
    CW["CloudWatch (optional)<br/>logs/metrics"]
  end

  TF --> AWS
  ANS --> UVM
  ANS --> MVM
  ANS --> EC2
  DC --> UVM
  DC --> EC2

  UVM --> MVM
  EC2 --> MVM

  UVM --> PBS
  MVM --> PBS
  PBS --> NAS --> COLD

  SG --> EC2
  VPC --> EC2
  EC2 --> CW

flowchart TD
  VM["VMs: Utility + Monitoring"] --> PBS["PBS datastore"]
  PBS --> NAS["NAS replication/copy"]
  NAS --> COLD["Cold storage tier"]
  COLD -->|"restore test"| VM
  PBS -->|"restore"| VM

  

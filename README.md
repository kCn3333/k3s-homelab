# k3s-homelab

> Production-grade Kubernetes homelab running on bare-metal HP T630 thin clients — built to simulate real-world DevOps workflows.

---

## Overview

This repository is the **single source of truth** for a self-hosted Kubernetes cluster managed entirely through GitOps principles. Every change to the cluster passes through Git — no manual `kubectl apply`, no configuration drift.

The cluster is intentionally designed to mirror production environments: HA control plane, TLS everywhere, automated certificate management, distributed storage, CNI with eBPF, full observability stack with metrics and logs, and a complete GitOps pipeline with continuous image delivery.

---

## Infrastructure

| Component | Technology |
|-----------|-----------|
| Hardware | 3× HP T630 Thin Client |
| OS | Ubuntu 24.04 LTS |
| Kubernetes | k3s v1.34 (embedded etcd, HA) |
| CNI | Cilium v1.19.1 (eBPF) |
| Storage | Longhorn v1.11 (distributed block storage) |
| Object Storage | Garage v2.2.0 (self-hosted S3, Debian host) |
| Ingress | Traefik v3 |
| Load Balancer | HAProxy (bare-metal) |
| Certificate Management | cert-manager + Let's Encrypt (DNS-01) |
| Secrets Management | Sealed Secrets v0.36 |
| GitOps | Flux v2 |
| Metrics | kube-prometheus-stack v82 (Prometheus + Grafana + AlertManager) |
| Logs | Loki v3.6 + Promtail (stored in Garage S3) |
| Alerting | AlertManager → ntfy (self-hosted, Cloudflare Tunnel) |
| DNS | Cloudflare (public) + PiHole (local) |
| Firewall | UFW (managed via Ansible) |

---

## Architecture

```
                        ┌─────────────────────────────────┐
                        │         GitHub Repository        │
                        │    (single source of truth)      │
                        └────────────────┬────────────────┘
                                         │ Flux watches (pull)
                        ┌────────────────▼────────────────┐
Local Network           │                                  │
                        │         HAProxy :80/:443         │
 Client ──────────────► │         192.168.0.45             │
                        │         (bare-metal LB)          │
                        └────────────────┬────────────────┘
                                         │
                        ┌────────────────▼────────────────┐
                        │         Traefik Ingress          │
                        │    HTTP → HTTPS redirect         │
                        │    TLS termination               │
                        └────────────────┬────────────────┘
                                         │
              ┌──────────────────────────┼──────────────────────────┐
              │                          │                          │
    ┌─────────▼────────┐     ┌──────────▼────────┐     ┌──────────▼────────┐
    │  master          │     │  worker1           │     │  worker2          │
    │  192.168.55.10   │     │  192.168.55.11     │     │  192.168.55.12    │
    │  control-plane   │◄────►  control-plane    │◄────►  control-plane   │
    │  etcd            │     │  etcd              │     │  etcd             │
    │  node-exporter   │     │  node-exporter     │     │  node-exporter    │
    │  promtail        │     │  promtail          │     │  promtail         │
    └──────────────────┘     └───────────────────┘     └───────────────────┘
              │                          │                          │
              └──────────────────────────┼──────────────────────────┘
                                         │ metrics + logs
                        ┌────────────────▼────────────────┐
                        │      Debian Host 192.168.0.46    │
                        │      Garage S3 (25GB)            │
                        │      • longhorn-backup bucket    │
                        │      • loki-logs bucket          │
                        └─────────────────────────────────┘
```

---

## GitOps Flow

```
Developer pushes code
        │
        ▼
GitHub Actions builds Docker image
        │
        ▼
Image pushed to registry with semver tag
        │
        ▼
Flux ImagePolicy detects new tag
        │
        ▼
Flux commits updated tag to this repo
        │
        ▼
Flux kustomize-controller detects change
        │
        ▼
Rolling update deployed to cluster
```

---

## Repository Structure

```
k3s-homelab/
├── apps/
│   └── base/                      # Application manifests
│       ├── kustomization.yaml     # App registry
│       ├── cilium/                # CNI — managed by Flux
│       ├── longhorn/              # Distributed storage + S3 backup
│       ├── loki/                  # Log aggregation (Garage S3 backend)
│       ├── monitoring/            # Prometheus + Grafana + AlertManager + ntfy
│       ├── promtail/              # Log collection DaemonSet
│       ├── sealed-secrets/        # Secrets management
│       ├── traefik-dashboard/     # Traefik UI with BasicAuth
│       └── nginx/                 # Example app with image automation
└── clusters/
    └── k3s-homelab/
        ├── apps.yaml
        ├── helmchartconfig-traefik.yaml
        ├── image-update-automation.yaml
        └── flux-system/
```

---

## Key Features

**High Availability**
- 3-node cluster — all nodes run control-plane + etcd
- Embedded etcd with automatic leader election
- HAProxy with round-robin load balancing across all API servers

**CNI — Cilium (eBPF)**
- Replaces both flannel and kube-proxy entirely
- eBPF-based networking — higher performance, lower overhead
- NetworkPolicy support out of the box
- Hubble observability (ready to enable)

**Distributed Storage — Longhorn**
- Block storage replicated across all 3 nodes
- ReadWriteMany (RWX) via built-in NFS share manager
- Daily automated backups to Garage S3 (retain: 2)
- BackupTarget configured via CRD, credentials as SealedSecret

**Object Storage — Garage S3**
- Self-hosted S3-compatible storage on Debian host
- 25GB capacity, single-node deployment via Docker Compose
- Two buckets: `longhorn-backup` and `loki-logs`

**Monitoring — kube-prometheus-stack**
- Prometheus with 7-day retention on Longhorn PVC
- Grafana at `grafana.cluster.kcn333.com` with TLS
- node-exporter DaemonSet — CPU, RAM, disk, network per node
- Prometheus runs with `hostNetwork: true` for direct kubelet access on k3s

**Log Aggregation — Loki + Promtail**
- Loki in SingleBinary mode with Garage S3 backend
- 7-day log retention with automatic compaction
- Promtail DaemonSet collecting logs from all 3 nodes
- Grafana Loki datasource — unified metrics + logs view

**Alerting — AlertManager + ntfy**
- AlertManager routes alerts to self-hosted ntfy instance
- Custom Python webhook adapter — translates JSON to ntfy HTTP API
- `User-Agent: Mozilla/5.0` required for Cloudflare Tunnel
- Priority mapping: critical→urgent, warning→high, info→default
- All credentials stored as SealedSecrets

**TLS Everywhere**
- Wildcard certificate `*.cluster.kcn333.com` via cert-manager
- Let's Encrypt DNS-01 challenge through Cloudflare API
- Automatic certificate rotation — 30 days before expiry

**GitOps with Flux v2**
- Cluster state reconciled every 60 seconds
- `prune: true` — resources removed from Git are removed from cluster
- Automated image tag updates committed back to repo by Flux bot

**Secrets Management — Sealed Secrets**
- All secrets encrypted in Git: cloudflare-token, traefik-auth, grafana-admin, ntfy-credentials, S3-keys

**Infrastructure as Code**
- UFW rules managed via Ansible playbooks
- NTP synchronization playbook (workers → master chrony)
- Graceful node shutdown playbook

---

## Backup Strategy

| What | How | Frequency | Retention | Destination |
|------|-----|-----------|-----------|-------------|
| etcd snapshots | k3s automatic | Daily 12:00 UTC | 5 latest | On cluster |
| etcd snapshots | rsync | Daily 13:00 UTC | 30 days | Debian host |
| Longhorn volumes | RecurringJob | Daily 11:00 UTC | 2 latest | Garage S3 |

---

## Ansible

- **UFW playbook** — firewall rules on all k3s nodes
- **NTP playbook** — timesyncd config pointing to master (chrony)
- **Longhorn prepare** — open-iscsi, nfs-common
- **Graceful shutdown** — safely drains and powers off all nodes

---

## Roadmap

- [ ] **Custom AlertManager rules** — high CPU/RAM thresholds
- [ ] CI/CD pipeline — GitHub Actions + own application
- [ ] Own microservices application (Spring Boot)
- [ ] Progressive delivery — staging / production
- [ ] HashiCorp Vault
- [ ] External-dns
- [ ] NetworkPolicy — pod-level isolation (Cilium ready)
- [ ] Hubble UI — Cilium network observability
- [ ] RBAC
- [x] Sealed Secrets
- [x] Traefik dashboard with BasicAuth
- [x] Monitoring — Prometheus + Grafana
- [x] Log aggregation — Loki + Promtail
- [x] Alerting — AlertManager + ntfy
- [x] S3 backup — Garage + Longhorn RecurringJob

---

## Local DNS

All `*.cluster.kcn333.com` subdomains resolve to `192.168.0.45` (HAProxy) via PiHole.

---

## Notes

Actively developed as a learning environment for production DevOps practices. Each component was chosen to reflect real-world tooling used in professional Kubernetes deployments.

Commit history follows [Conventional Commits](https://www.conventionalcommits.org/) specification.
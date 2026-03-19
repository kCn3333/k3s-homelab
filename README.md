# k3s-homelab

> Production-grade Kubernetes homelab running on bare-metal HP T630 thin clients — built to simulate real-world DevOps workflows.

---

## Overview

This repository is the **single source of truth** for a self-hosted Kubernetes cluster managed entirely through GitOps principles. Every change to the cluster passes through Git — no manual `kubectl apply`, no configuration drift.

The cluster is intentionally designed to mirror production environments: HA control plane, TLS everywhere, automated certificate management, distributed storage, CNI with eBPF, full observability stack with metrics and logs, complete GitOps pipeline with continuous image delivery, and a PostgreSQL database managed by CloudNativePG.

---

## Infrastructure

| Component | Technology |
| :--- | :--- |
| Hardware | 3× HP T630 Thin Client |
| OS | Ubuntu 24.04 LTS |
| Kubernetes | k3s v1.34 (embedded etcd, HA) |
| CNI | Cilium v1.19.1 (eBPF, VXLAN) |
| Storage | Longhorn v1.11 (distributed block storage) |
| Object Storage | Garage v2.2.0 (self-hosted S3, Debian host) |
| Database | CloudNativePG (PostgreSQL 17) |
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
Developer pushes git tag (v*)
        │
        ▼
GitHub Actions: test → build → push
        │
        ▼
DockerHub: semver tags (1.x.x, 1.x, sha-XXXX, latest)
        │
        ▼
Flux ImagePolicy detects new semver tag
        │
        ▼
Flux commits updated tag to this repo
        │
        ▼
Flux HelmRelease upgrade triggered
        │
        ▼
Rolling update deployed to cluster
```

---

## Repository Structure

```
k3s-homelab/
├── apps/
│   └── base/                          # Application manifests
│       ├── kustomization.yaml
│       ├── clients-api/               # Spring Boot API + PostgreSQL
│       │   ├── namespace.yaml
│       │   ├── db-cluster.yaml        # CloudNativePG cluster
│       │   ├── db-secret-sealed.yaml  # DB credentials (SealedSecret)
│       │   ├── imagerepository.yaml   # Scans kcn333/clients-api every 1m
│       │   ├── imagepolicy.yaml       # semver >=1.0.0
│       │   ├── gitrepository.yaml     # Source: clients-api GitHub repo
│       │   ├── helmrelease.yaml       # Deploys helm/clients-api chart
│       │   └── kustomization.yaml
│       └── nginx/                     # Example app with image automation
│           ├── imagepolicy.yaml
│           ├── imagerepository.yaml
│           ├── kustomization.yaml
│           ├── namespace.yaml
│           └── nginx-deploy.yaml
├── clusters/
│   └── k3s-homelab/
│       ├── apps.yaml                  # Flux Kustomization → ./apps/base
│       ├── infrastructure.yaml        # Flux Kustomization → ./infrastructure
│       ├── image-update-automation.yaml
│       └── flux-system/
│           ├── gotk-components.yaml
│           ├── gotk-sync.yaml
│           └── kustomization.yaml
└── infrastructure/
    ├── config/                        # Cluster config (non-operator resources)
    │   ├── kustomization.yaml
    │   ├── longhorn-config/           # BackupTarget, RecurringJob (daily S3 backup)
    │   └── traefik-dashboard/         # IngressRoute, Middleware, BasicAuth, TLS
    └── operators/                     # Helm-managed operators
        ├── kustomization.yaml
        ├── cilium/                    # CNI — eBPF, VXLAN mode
        ├── cloudnative-pg/            # PostgreSQL operator
        ├── loki/                      # Log aggregation (Garage S3 backend)
        ├── longhorn/                  # Distributed block storage
        ├── metrics-server/            # Resource metrics for kubectl top / HPA
        ├── monitoring/                # Prometheus + Grafana + AlertManager + ntfy
        ├── promtail/                  # Log collection DaemonSet
        └── sealed-secrets/            # Secrets encryption
```

---

## Key Features

**High Availability**
- 3-node cluster — all nodes run control-plane + etcd
- Embedded etcd with automatic leader election
- HAProxy with round-robin load balancing across all API servers

**CNI — Cilium (eBPF)**
- eBPF-based networking — higher performance, lower overhead
- NetworkPolicy support out of the box
- VXLAN tunnel mode with kube-proxy for service routing

**Application — clients-api (Spring Boot)**
- Deployed via custom Helm chart from application repository
- HPA: min 2 / max 6 replicas, CPU target 70%
- Pod Disruption Budget: minAvailable 1 (safe during node drain)
- NetworkPolicy: DB accessible only from clients-api; API only from kube-system + monitoring
- Readiness/Liveness probes with JVM warmup delay
- Full observability: ServiceMonitor + custom PrometheusRules + Grafana dashboard + Loki logs

**Database — CloudNativePG**
- PostgreSQL 17 managed by CloudNativePG operator
- Declarative cluster configuration as CRD
- Automatic failover and read replicas

**Distributed Storage — Longhorn**
- Block storage replicated across all 3 nodes
- ReadWriteMany (RWX) via built-in NFS share manager
- Daily automated backups to Garage S3 (retain: 2)

**Object Storage — Garage S3**
- Self-hosted S3-compatible storage on Debian host
- 25GB capacity, single-node deployment via Docker Compose
- Two buckets: `longhorn-backup` and `loki-logs`

**Monitoring — kube-prometheus-stack**
- Prometheus with 7-day retention on Longhorn PVC
- Grafana at `grafana.cluster.kcn333.com` with TLS
- node-exporter DaemonSet — CPU, RAM, disk, network per node
- Custom PrometheusRules for infrastructure and application-level alerts

**Log Aggregation — Loki + Promtail**
- Loki in SingleBinary mode with Garage S3 backend
- 7-day log retention with automatic compaction
- Promtail DaemonSet collecting logs from all 3 nodes

**Alerting — AlertManager + ntfy**
- AlertManager routes alerts to self-hosted ntfy instance
- Custom Python webhook adapter
- Priority mapping: critical→urgent, warning→high, info→default
- Noise suppression: InfoInhibitor routed to null receiver

**TLS Everywhere**
- Wildcard certificate `*.cluster.kcn333.com` via cert-manager
- Let's Encrypt DNS-01 challenge through Cloudflare API
- Automatic certificate rotation — 30 days before expiry

**GitOps with Flux v2**
- Cluster state reconciled every 60 seconds
- `prune: true` — resources removed from Git are removed from cluster
- Automated image tag updates committed back to repo by Flux bot
- Application deployed via Helm chart with `reconcileStrategy: Revision`

**Secrets Management — Sealed Secrets**
- All secrets encrypted in Git: cloudflare-token, traefik-auth, grafana-admin, ntfy-credentials, S3-keys, DB-credentials

**Infrastructure as Code**
- UFW rules managed via Ansible playbooks
- NTP synchronization playbook (workers → master chrony)
- Graceful node shutdown playbook

---

## Backup Strategy

| What | How | Frequency | Retention | Destination |
| :--- | :--- | :--- | :--- | :--- |
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

## Local DNS

All `*.cluster.kcn333.com` subdomains resolve to `192.168.0.45` (HAProxy) via PiHole.

---

## Roadmap

- [ ] **Hubble UI** — requires Cilium native routing migration (planned)
- [ ] Helm chart OCI registry — publish clients-api chart to ghcr.io
- [ ] Progressive delivery — staging / production
- [ ] HashiCorp Vault
- [ ] External-dns
- [ ] RBAC
- [x] Custom Helm chart for clients-api (mono-repo, Flux HelmRelease)
- [x] PrometheusRule for clients-api — HighErrorRate, HighLatency, PodRestarting
- [x] AlertManager noise suppression — null receiver for InfoInhibitor
- [x] Pod Disruption Budget
- [x] NetworkPolicy — pod-level isolation (DB + API)
- [x] HPA — Horizontal Pod Autoscaler (min 2 / max 6)
- [x] Grafana dashboard for application metrics (HTTP, JVM, HikariCP)
- [x] CloudNativePG — PostgreSQL operator
- [x] Custom AlertManager rules (CPU, Memory, Disk, CrashLoop, Longhorn)
- [x] Sealed Secrets
- [x] Traefik dashboard with BasicAuth
- [x] Monitoring — Prometheus + Grafana
- [x] Log aggregation — Loki + Promtail
- [x] Alerting — AlertManager + ntfy
- [x] S3 backup — Garage + Longhorn RecurringJob
- [x] metrics-server — kubectl top + HPA ready

---

## Notes

Actively developed as a learning environment for production DevOps practices. Each component was chosen to reflect real-world tooling used in professional Kubernetes deployments.

Commit history follows [Conventional Commits](https://www.conventionalcommits.org/) specification.
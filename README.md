# k3s-homelab

> Production-grade Kubernetes homelab running on bare-metal HP T630 thin clients — built to simulate real-world DevOps workflows.

---

## Overview

This repository is the **single source of truth** for a self-hosted Kubernetes cluster managed entirely through GitOps principles. Every change to the cluster passes through Git — no manual `kubectl apply`, no configuration drift.

The cluster is intentionally designed to mirror production environments: HA control plane, TLS everywhere, automated certificate management, distributed storage, CNI with eBPF, full observability stack, and a complete GitOps pipeline with continuous image delivery.

---

## Infrastructure

| Component | Technology |
|-----------|-----------|
| Hardware | 3× HP T630 Thin Client |
| OS | Ubuntu 24.04 LTS |
| Kubernetes | k3s v1.34 (embedded etcd, HA) |
| CNI | Cilium v1.19.1 (eBPF) |
| Storage | Longhorn v1.11 (distributed block storage) |
| Ingress | Traefik v3 |
| Load Balancer | HAProxy (bare-metal) |
| Certificate Management | cert-manager + Let's Encrypt (DNS-01) |
| Secrets Management | Sealed Secrets v0.36 |
| GitOps | Flux v2 |
| Monitoring | kube-prometheus-stack v82 (Prometheus + Grafana + AlertManager) |
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
    └──────────────────┘     └───────────────────┘     └───────────────────┘
                    Cilium eBPF CNI — pod network & policy
                    Longhorn — distributed storage with replicas
                    Prometheus — scrapes all nodes via hostNetwork
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
│       ├── longhorn/              # Distributed storage
│       ├── monitoring/            # Prometheus + Grafana stack
│       ├── sealed-secrets/        # Secrets management
│       ├── traefik-dashboard/     # Traefik UI with BasicAuth
│       └── <app-name>/
│           ├── namespace.yaml
│           ├── deployment.yaml    # includes Flux image policy marker
│           ├── service.yaml
│           ├── ingress.yaml
│           ├── imagerepository.yaml
│           ├── imagepolicy.yaml
│           └── kustomization.yaml
└── clusters/
    └── k3s-homelab/
        ├── apps.yaml              # Flux Kustomization → ./apps/base
        ├── helmchartconfig-traefik.yaml
        ├── image-update-automation.yaml
        └── flux-system/           # Flux self-management (auto-generated)
            ├── gotk-components.yaml
            ├── gotk-sync.yaml
            └── kustomization.yaml
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
- Managed by Flux as HelmRelease

**Distributed Storage — Longhorn**
- Block storage replicated across all 3 nodes
- ReadWriteMany (RWX) via built-in NFS share manager
- Dynamic provisioning — no manual PV creation
- Default StorageClass for the cluster
- Managed by Flux as HelmRelease

**Monitoring — kube-prometheus-stack**
- Prometheus with 7-day retention on Longhorn PVC
- Grafana at `grafana.cluster.kcn333.com` with TLS and SealedSecret credentials
- node-exporter DaemonSet — CPU, RAM, disk, network per node
- kube-state-metrics — Kubernetes object metrics
- AlertManager — ready for alert configuration
- Prometheus runs with `hostNetwork: true` for direct kubelet access on k3s

**TLS Everywhere**
- Wildcard certificate `*.cluster.kcn333.com` via cert-manager
- Let's Encrypt DNS-01 challenge through Cloudflare API (no public exposure needed)
- Automatic certificate rotation — 30 days before expiry
- HTTP → HTTPS redirect enforced at Traefik level

**GitOps with Flux v2**
- Cluster state reconciled every 60 seconds
- `prune: true` — resources removed from Git are removed from cluster
- Automated image tag updates committed back to repo by Flux bot
- Conventional Commits enforced for clean history

**Secrets Management — Sealed Secrets**
- Secrets encrypted with cluster public key — safe to store in Git
- Only the cluster can decrypt — asymmetric encryption
- `cloudflare-token`, `traefik-dashboard-auth`, `grafana-admin-secret` stored as SealedSecrets

**Security**
- UFW firewall on all nodes — only HAProxy and intra-cluster traffic allowed
- Direct node access blocked — all traffic routes through HAProxy
- HAProxy bound to dedicated IP alias, isolated from other services
- etcd snapshots automated daily + offsite backup via rsync to separate host
- Traefik dashboard protected with BasicAuth (htpasswd + SealedSecret)
- Grafana protected with credentials stored as SealedSecret

**Infrastructure as Code**
- UFW rules managed via Ansible playbooks
- Graceful node shutdown via Ansible playbook
- All cluster configuration in Git — no manual state
- Cluster bootstrapped from k3s install flags — reproducible

---

## Backup Strategy

| What | How | Frequency | Retention |
|------|-----|-----------|-----------|
| etcd snapshots | k3s automatic | Daily 12:00 UTC | 5 latest (on cluster) |
| etcd snapshots | rsync to Debian host | Daily 13:00 UTC | 30 days |

Offsite backup script: pulls latest snapshot from master via SSH, stores in dated directory, cleans up files older than 30 days.

---

## Ansible

Infrastructure configuration is managed via Ansible with Semaphore UI:

- **UFW playbook** — configures firewall rules on all k3s nodes
- **Longhorn prepare playbook** — installs open-iscsi, nfs-common on all nodes
- **Graceful shutdown playbook** — safely drains and powers off all nodes
- **Maintenance playbooks** — rolling updates, node management

Inventory covers all three k3s nodes. Playbooks are idempotent — safe to run repeatedly.

---

## Roadmap

- [ ] **AlertManager** — configure alerts (node down, high CPU/RAM)
- [ ] CI/CD pipeline — GitHub Actions building and pushing application images
- [ ] Own microservices application (Spring Boot) deployed via this GitOps workflow
- [ ] Helm charts for applications
- [ ] Progressive delivery — staging / production branch strategy
- [ ] HashiCorp Vault — secrets management
- [ ] External Secrets Operator
- [x] Sealed Secrets for secrets in Git
- [ ] external-dns — automatic DNS records from Ingress resources
- [x] Traefik dashboard with BasicAuth
- [ ] NetworkPolicy — pod-level network isolation (Cilium ready)
- [ ] Hubble — Cilium network observability UI
- [ ] RBAC — fine-grained access control

---

## Local DNS

All `*.cluster.kcn333.com` subdomains resolve to `192.168.0.45` (HAProxy) via PiHole. Public DNS on Cloudflare is used only for Let's Encrypt DNS-01 challenge — the cluster is not publicly accessible.

---

## Notes

This project is actively developed as a learning environment for production DevOps practices. Each component was chosen to reflect real-world tooling used in professional Kubernetes deployments.

Commit history follows [Conventional Commits](https://www.conventionalcommits.org/) specification.
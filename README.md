# k3s-homelab

> Production-grade Kubernetes homelab running on bare-metal HP T630 thin clients — built to simulate real-world DevOps workflows.

---

## Overview

This repository is the **single source of truth** for a self-hosted Kubernetes cluster managed entirely through GitOps principles. Every change to the cluster passes through Git — no manual `kubectl apply`, no configuration drift.

The cluster is intentionally designed to mirror production environments: HA control plane, TLS everywhere, automated certificate management, network security policies, and a full GitOps pipeline with continuous image delivery.

---

## Infrastructure

| Component | Technology |
|-----------|-----------|
| Hardware | 3× HP T630 Thin Client |
| OS | Ubuntu 24.04 LTS |
| Kubernetes | k3s v1.34 (embedded etcd, HA) |
| Ingress | Traefik v3 |
| Load Balancer | HAProxy (bare-metal) |
| Certificate Management | cert-manager + Let's Encrypt (DNS-01) |
| GitOps | Flux v2 |
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
    └──────────────────┘     └───────────────────┘     └───────────────────┘
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

**Security**
- UFW firewall on all nodes — only HAProxy and intra-cluster traffic allowed
- Direct node access blocked — all traffic routes through HAProxy
- HAProxy bound to dedicated IP alias, isolated from other services
- etcd snapshots automated daily + offsite backup via rsync to separate host

**Infrastructure as Code**
- UFW rules managed via Ansible playbooks
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
- **Maintenance playbooks** — rolling updates, node management

Inventory covers all three k3s nodes. Playbooks are idempotent — safe to run repeatedly.

---

## Roadmap

- [ ] CI/CD pipeline — GitHub Actions building and pushing application images
- [ ] Own microservices application (Spring Boot) deployed via this GitOps workflow
- [ ] Helm charts for applications
- [ ] Progressive delivery — staging / production branch strategy
- [ ] HashiCorp Vault — secrets management
- [ ] External Secrets Operator
- [ ] Sealed Secrets for secrets in Git
- [ ] external-dns — automatic DNS records from Ingress resources
- [ ] Traefik dashboard with BasicAuth
- [ ] NetworkPolicy — pod-level network isolation
- [ ] RBAC — fine-grained access control

---

## Local DNS

All `*.cluster.kcn333.com` subdomains resolve to `192.168.0.45` (HAProxy) via PiHole. Public DNS on Cloudflare is used only for Let's Encrypt DNS-01 challenge — the cluster is not publicly accessible.

---

## Notes

This project is actively developed as a learning environment for production DevOps practices. Each component was chosen to reflect real-world tooling used in professional Kubernetes deployments.

Commit history follows [Conventional Commits](https://www.conventionalcommits.org/) specification.
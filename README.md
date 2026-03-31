![20260330_151759](https://github.com/user-attachments/assets/53b38115-c0c7-49b5-97b7-fded9abe4953)

# vi3t-lab — Raspberry Pi k3s Homelab

A production-style Kubernetes homelab built on four Raspberry Pi 4B nodes,
designed as a hands-on learning environment for cloud engineering and
infrastructure certifications (AWS, CKA, CompTIA Network+/Security+, IaC).

Every service in this cluster is deployed via GitOps — no manual kubectl apply.
Infrastructure is version-controlled, declarative, and self-healing.

---

## Hardware

| Node | Role | IP |
|---|---|---|
| pi-node-01 | control plane | 192.168.30.10 |
| pi-node-02 | worker | 192.168.30.11 |
| pi-node-03 | worker | 192.168.30.12 |
| pi-node-04 | worker | 192.168.30.13 |
| ASRock Master | Proxmox hypervisor | 192.168.99.67 |

- **OS:** Raspberry Pi OS Lite (64-bit)
- **Kubernetes:** k3s (lightweight Kubernetes for ARM64)
- **Network:** Ubiquiti UniFi, segmented VLANs (Trusted / Homelab / IoT / MGMT)

---

## Core Stack

| Tool | Purpose |
|---|---|
| **k3s** | Lightweight Kubernetes distribution for ARM64 |
| **ArgoCD** | GitOps continuous delivery — all deployments tracked in this repo |
| **MetalLB** | Bare-metal load balancer (Layer 2 / ARP mode) |
| **Traefik** | Ingress controller with automatic TLS termination |
| **Cert-Manager** | Automated TLS certificates via Let's Encrypt + Cloudflare DNS-01 |
| **Sealed Secrets** | Encrypted Kubernetes secrets safe to commit to Git |
| **Pi-hole** | Local DNS resolution + network-wide ad blocking |

---

## Deployed Services

| Service | URL | Notes |
|---|---|---|
| ArgoCD | `argocd.vi3t-lab.com` | GitOps dashboard — manages all apps in this repo |
| Grafana | `grafana.vi3t-lab.com` | Cluster metrics via kube-prometheus-stack |
| Prometheus | `prometheus.vi3t-lab.com` | Metrics scraping and alerting |
| Uptime Kuma | `uptime.vi3t-lab.com` | Service uptime monitoring |
| Homepage | `dashboard.vi3t-lab.com` | Self-hosted dashboard with live cluster stats |
| Pi-hole | `192.168.30.191/admin` | Local DNS + ad blocking |
| WireGuard | `vpn.vi3t-lab.com` | VPN tunnel into the homelab |
| Minecraft | `mc.vi3t-lab.com` | Java Edition server with persistent world storage |

---

## GitOps Workflow

All changes to this cluster flow through Git:

1. YAML is written and pushed to this repo
2. ArgoCD detects the change and syncs the cluster to match
3. ArgoCD's `selfHeal` and `prune` flags keep live state in sync with repo state

Secrets are encrypted with **kubeseal** before being committed.
Plain-text secrets never exist in this repo.

---

## Networking

- **IP range:** `192.168.30.190 – 192.168.30.230` (MetalLB pool)
- **Ingress:** All HTTP/HTTPS traffic enters through Traefik at `192.168.30.190`
- **TLS:** Cert-Manager issues certificates via Let's Encrypt (Cloudflare DNS-01 challenge)
- **Local DNS:** Pi-hole resolves `*.vi3t-lab.com` to MetalLB IPs internally
- **External DNS:** Cloudflare manages public-facing records
- **VPN:** WireGuard tunnels remote clients into the Trusted VLAN with Pi-hole as DNS

---

## Project Chapters

- ✅ **Chapter 1 — Infrastructure Groundwork**
  ArgoCD, MetalLB, Sealed Secrets, Cert-Manager

- ✅ **Chapter 2 — Core Services**
  Pi-hole (local DNS + ad blocking), WireGuard VPN, Cloudflare DDNS

- ✅ **Chapter 3 — Monitoring**
  kube-prometheus-stack (Grafana + Prometheus), Uptime Kuma

- ✅ **Chapter 4 — Applications**
  Minecraft Java server, Glances (per-node stats), Homepage dashboard

- ⏭️ **Chapter 5 — (in progress)**

---

## Repo Structure
```
k3s-cluster/
├── core/               # Cluster infrastructure (ArgoCD, MetalLB, Cert-Manager, Sealed Secrets)
├── apps/               # Deployed applications (Pi-hole, WireGuard, Minecraft, Homepage, etc.)
└── CLUSTER-STATE.md    # Live reference: IPs, services, known issues, firewall rules
```

---

## About

This homelab is my primary learning environment as I transition from 6 years of
help desk experience into cloud engineering. Every component here was built
intentionally — not just to get something running, but to understand why it works.



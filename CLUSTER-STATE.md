# k3s Homelab Cluster State
> Keep this file updated at the end of every chapter. Paste raw URL into new Claude chats to resume.

---

## Ground Rules (Always Follow)
- One step at a time — wait for terminal output before proceeding
- All deployments via ArgoCD + GitHub (GitOps) — no manual `kubectl apply` for new services
- No secrets in plain text ever — always use Sealed Secrets + kubeseal
- Explain the why behind each step
- Do not assume firewall rules — always ask
- Always use `--server-side --force-conflicts` when applying large manifests

---

## Network Layout

| Device | IP | VLAN | Notes |
|---|---|---|---|
| Ubiquiti Cloud Gateway Ultra | Gateway | — | Main router |
| TP-Link TL-SG2210P V3 | 192.168.99.2 | VLAN 99 | Managed switch |
| ASRock_Master | 192.168.99.67 | VLAN 99 | Proxmox 8.4.16, acts as trunk |
| pi-master-0 | 192.168.30.10 | VLAN 30 | k3s master node |
| pi-worker-1 | 192.168.30.11 | VLAN 30 | k3s worker node |
| pi-worker-2 | 192.168.30.12 | VLAN 30 | k3s worker node |
| pi-worker-3 | 192.168.30.13 | VLAN 30 | k3s worker node |

### VLANs
- **VLAN 10** — Trusted (personal PC, phone)
- **VLAN 20** — IoT (streaming devices, gaming consoles)
- **VLAN 30** — Homelab (k3s cluster + Proxmox containers)
- **VLAN 99** — Management (ASRock_Master, managed switch)

### Proxmox Containers (VLAN 30, on ASRock_Master)
| Container | ID | IP |
|---|---|---|
| nginx | 100 | 192.168.30.66 |
| jellyfin | 101 | 192.168.30.65 |
| nextcloud | 102 | 192.168.30.64 |
| ArkServer | 103 | 192.168.30.245 |

### Backburner (not in scope yet)
- 3x HP EliteDesk 705 G4 — reserved for future project

---

## Domain & DNS
- **Domain:** vi3t-lab.com
- **DNS Provider:** Cloudflare
- **Subdomain pattern:** `*.vi3t-lab.com` for all internal services
- **DNS resolution:** Pi-hole handles local DNS for `*.vi3t-lab.com` → `192.168.30.190` (Traefik)
- **VLAN 10 DNS:** Set to `192.168.30.191` (Pi-hole) manually in Ubiquiti + Windows PC NIC
- **IPv6:** Disabled on VLAN 10 devices — Pi-hole is IPv4 only

### Pi-hole Local DNS Records
| Domain | IP |
|---|---|
| argocd.vi3t-lab.com | 192.168.30.190 |
| pihole.vi3t-lab.com | 192.168.30.190 |

---

## k3s Cluster

### Nodes
- 4x Raspberry Pi 4B running Raspberry Pi OS (Debian Trixie)
- k3s installed, `git` installed on pi-master-0
- Repo cloned at `~/k3s-cluster` on pi-master-0
- Git credentials stored via `credential.helper store`

### k3s Config (`/etc/rancher/k3s/config.yaml` on pi-master-0)
```yaml
disable:
  - servicelb
```
> Klipper (built-in ServiceLB) is disabled — MetalLB handles all LoadBalancer IPs

### Built-in Components (do not touch)
- Traefik v3.6.7 — ingress controller, IP: `192.168.30.190`
- CoreDNS — internal cluster DNS
- metrics-server — resource metrics
- local-path-provisioner — default storage provisioner

---

## MetalLB
- **IP Pool:** `192.168.30.190 - 192.168.30.230`
- **Mode:** Layer 2 (ARP)
- **Traefik:** holds `192.168.30.190` (first IP in pool)
- **Pi-hole DNS:** holds `192.168.30.191`
- **Known issue:** `bgppeers.metallb.io` CRD shows OutOfSync due to large caBundle — fixed via `ignoreDifferences` in `argocd-app.yaml`. Harmless — L2 mode doesn't use BGP.

---

## Installed Core Services

| Service | Namespace | Version | Sync | Health |
|---|---|---|---|---|
| ArgoCD | argocd | stable | Synced | Healthy |
| MetalLB | metallb-system | v0.14.9 | Synced | Healthy |
| Sealed Secrets | sealed-secrets | v0.27.3 | Synced | Healthy |
| Cert-Manager | cert-manager | v1.17.1 | Synced | Healthy |
| Pi-hole | pihole | 2024.07.0 | Synced | Healthy |

### ArgoCD
- Installed via `kubectl apply -k` (bootstrapped manually, now self-managing)
- Running in insecure mode (`server.insecure: true` in `argocd-cmd-params-cm`) — Traefik handles HTTPS
- Ingress: `argocd.vi3t-lab.com` ✅ accessible via browser
- TLS cert issued: `argocd-tls` in argocd namespace
- ArgoCD manages itself via `core/argocd/argocd-app.yaml`
- **Important:** Any changes to `argocd-cmd-params-cm` require manual `kubectl apply --server-side --force-conflicts` due to field ownership conflict with upstream install.yaml
- **Important:** Any changes to `argocd-app.yaml` files require manual re-apply with `--server-side --force-conflicts`

### Sealed Secrets
- Controller: `sealed-secrets-controller` in `sealed-secrets` namespace
- `kubeseal` v0.27.3 installed on pi-master-0 at `/usr/local/bin/kubeseal`
- **Sealing command pattern:**
```bash
kubectl create secret generic <secret-name> \
  --namespace <namespace> \
  --from-literal=<key>=<value> \
  --dry-run=client -o yaml | \
  kubeseal --controller-name=sealed-secrets-controller \
  --controller-namespace=sealed-secrets \
  --format yaml
```

### Cert-Manager
- ClusterIssuer: `letsencrypt-cloudflare` — **READY: True**
- Uses DNS-01 challenge via Cloudflare API
- Cloudflare API token stored as SealedSecret: `cloudflare-api-token` in `cert-manager` namespace
- **Certificate pattern:** Add annotations to any Ingress:
```yaml
cert-manager.io/cluster-issuer: letsencrypt-cloudflare
traefik.ingress.kubernetes.io/router.entrypoints: websecure
traefik.ingress.kubernetes.io/router.tls: "true"
```

### Pi-hole
- Namespace: `pihole`
- Image: `pihole/pihole:2024.07.0`
- DNS service: `pihole-dns` — LoadBalancer at `192.168.30.191` (ports 53 UDP+TCP)
- Web service: `pihole-web` — ClusterIP (port 80, internal only)
- Ingress: `pihole.vi3t-lab.com` — TLS cert issued: `pihole-tls`
- Admin password: stored as SealedSecret `pihole-secret` in `pihole` namespace
- Storage: 1Gi PVC via `local-path-provisioner`
- Bootstrapped via: `kubectl apply -f .../apps/pihole/argocd-app.yaml --server-side --force-conflicts`

---

## GitHub Repo
- **URL:** https://github.com/tranv1987-hash/k3s-cluster
- **Deployment method:** ArgoCD watches repo, auto-syncs on push

### Repo Structure
```
k3s-cluster/
├── core/
│   ├── argocd/
│   │   ├── kustomization.yaml
│   │   ├── argocd-cmd-params.yaml
│   │   ├── argocd-ingress.yaml
│   │   └── argocd-app.yaml
│   ├── metallb/
│   │   ├── kustomization.yaml
│   │   ├── metallb-native.yaml
│   │   ├── ipaddresspool.yaml
│   │   ├── l2advertisement.yaml
│   │   └── argocd-app.yaml
│   ├── sealed-secrets/
│   │   ├── kustomization.yaml
│   │   └── argocd-app.yaml
│   └── cert-manager/
│       ├── kustomization.yaml
│       ├── clusterissuer.yaml
│       ├── cloudflare-api-token-sealed.yaml
│       └── argocd-app.yaml
├── apps/
│   └── pihole/
│       ├── kustomization.yaml
│       ├── namespace.yaml
│       ├── pvc.yaml
│       ├── deployment.yaml
│       ├── service-dns.yaml
│       ├── service-web.yaml
│       ├── ingress.yaml
│       ├── admin-secret-sealed.yaml
│       └── argocd-app.yaml
├── _trash/
└── README.md
```

---

## Bootstrapping Pattern
> Any new ArgoCD Application must be bootstrapped once manually, then ArgoCD manages it forever after:
```bash
kubectl apply -f https://raw.githubusercontent.com/tranv1987-hash/k3s-cluster/main/<path>/argocd-app.yaml --server-side --force-conflicts
```

---

## Chapter Roadmap
- ✅ **Chapter 1** — Infrastructure Groundwork (ArgoCD, MetalLB, Sealed Secrets, Cert-Manager)
- ✅ **Chapter 2** — Core Services: Pi-hole (local DNS, VLAN 10 pointing to Pi-hole)
- ⏭️ **Chapter 2b** — Core Services: OpenVPN
- ⏭️ **Chapter 3** — Monitoring: Grafana, Prometheus, Uptime Kuma
- ⏭️ **Chapter 4** — Applications: Minecraft (Cloudflare Tunnel), Custom Dashboard

---

## Cert Roadmap (Owner)
- ✅ CompTIA A+ Core 1
- ⏳ CompTIA A+ Core 2
- ⏳ CompTIA Network+
- ⏳ CompTIA Security+
- ⏳ AWS Cloud Practitioner
- ⏳ AWS Solutions Architect – Associate
- ⏳ AWS Certified DevOps Engineer – Professional
- ⏳ Terraform + Ansible (IaC)
- ⏳ Linux Essentials (Red Hat on backburner)

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
- **DNS resolution strategy:** Pi-hole handles local DNS for `*.vi3t-lab.com` → MetalLB IPs

---

## k3s Cluster

### Nodes
- 4x Raspberry Pi 4B running Raspberry Pi OS (Debian Trixie)
- k3s installed, `git` installed on pi-master-0

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
- **Pi-hole:** holds `192.168.30.191`
- **WireGuard:** holds `192.168.30.192`
- **Minecraft:** holds `192.168.30.193`
- **Homepage:** holds `192.168.30.194`
- **Known issue:** `bgppeers.metallb.io` CRD shows OutOfSync due to large caBundle — fixed via `ignoreDifferences` in `argocd-app.yaml`. Harmless — L2 mode doesn't use BGP.

---

## Installed Core Services

| Service | Namespace | Version | Sync | Health |
|---|---|---|---|---|
| ArgoCD | argocd | stable | Synced | Healthy |
| MetalLB | metallb-system | v0.14.9 | Synced | Healthy |
| Sealed Secrets | sealed-secrets | v0.27.3 | Synced | Healthy |
| Cert-Manager | cert-manager | v1.17.1 | Synced | Healthy |
| Pi-hole | pihole | latest | Synced | Healthy |
| Cloudflare DDNS | cloudflare-ddns | v1.15.1 | Synced | Healthy |
| WireGuard (wg-easy) | wireguard | v14 | Synced | Healthy |
| kube-prometheus-stack | monitoring | 70.3.0 (Helm) | Synced | Healthy |
| Uptime Kuma | uptime-kuma | 1.x | Synced | Healthy |

### ArgoCD
- Installed via `kubectl apply -k` (bootstrapped manually, now self-managing)
- Running in insecure mode internally (`server.insecure: true`) — Traefik handles HTTPS externally
- Ingress: `argocd.vi3t-lab.com` (TLS cert issued, `argocd-tls` secret in argocd namespace)
- ArgoCD manages itself via `core/argocd/argocd-app.yaml`
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
> Always wrap values containing `$` in single quotes to prevent shell interpolation

### Cert-Manager
- ClusterIssuer: `letsencrypt-cloudflare` — **READY: True**
- Uses DNS-01 challenge via Cloudflare API
- Cloudflare API token stored as SealedSecret: `cloudflare-api-token` in `cert-manager` namespace
- **Certificate pattern:** Add annotation `cert-manager.io/cluster-issuer: letsencrypt-cloudflare` to any Ingress resource — cert is issued automatically

### Pi-hole
- Local DNS for `*.vi3t-lab.com` → MetalLB IPs
- IP: `192.168.30.191`
- Web UI: `https://pihole.vi3t-lab.com/admin` (note: must use `/admin` path)
- **Current local DNS records:**
  - `192.168.30.190 argocd.vi3t-lab.com`
  - `192.168.30.190 pihole.vi3t-lab.com`
  - `192.168.30.190 grafana.vi3t-lab.com`
  - `192.168.30.190 uptime.vi3t-lab.com`
  - `192.168.30.190 dashboard.vi3t-lab.com`
  - `192.168.30.193 mc.vi3t-lab.com`
- **To add a new DNS record:**
```bash
kubectl exec -n pihole -it $(kubectl get pod -n pihole -o name | head -1) -- bash -c 'echo "192.168.30.190 <subdomain>.vi3t-lab.com" >> /etc/pihole/custom.list'
kubectl exec -n pihole -it $(kubectl get pod -n pihole -o name | head -1) -- pihole restartdns
```

### WireGuard (wg-easy)
- VPN tunnel + web UI
- IP: `192.168.30.192`
- Web UI: `http://192.168.30.192:51821` (accessible from Trusted VLAN only — no Ingress by design)
- VPN tunnel: UDP 51820 (port forwarded from WAN)
- Clients connect via `vpn.vi3t-lab.com:51820`
- DNS for VPN clients: Pi-hole at `192.168.30.191`
- PASSWORD_HASH env var requires bcrypt hash — generate with:
```bash
python3 -c "import bcrypt; print(bcrypt.hashpw(b'YOUR_PASSWORD', bcrypt.gensalt()).decode())"
```

### Cloudflare DDNS
- Image: favonia/cloudflare-ddns:latest
- Keeps `vpn.vi3t-lab.com` A record pointed at WAN IP
- Checks every 5 minutes
- API token: `cloudflare-ddns-token` SealedSecret in `cloudflare-ddns` namespace
- Token permissions required: Zone → Zone → Read, Zone → DNS → Edit

### kube-prometheus-stack (Grafana + Prometheus)
- Namespace: `monitoring`
- Chart version: `70.3.0` (prometheus-community/kube-prometheus-stack)
- Grafana ingress: `grafana.vi3t-lab.com` (TLS via cert-manager, secret: `grafana-tls`)
- Grafana admin credentials: SealedSecret `grafana-admin-secret` in `monitoring` namespace
- **Correct Helm values field for existing secret:** `admin.existingSecret` (not `adminSecret`)
- Prometheus retention: 10 days, 5Gi PVC via local-path-provisioner
- Alertmanager: disabled
- Two ArgoCD Applications:
  - `monitoring-prereqs` — Kustomize (namespace + SealedSecret)
  - `monitoring` — Helm multi-source (chart from prometheus-community, values from GitHub)
- **Helm values file:** `apps/monitoring/helm-values.yaml`

### Uptime Kuma
- Namespace: `uptime-kuma`
- Ingress: `uptime.vi3t-lab.com` (TLS via cert-manager, secret: `uptime-kuma-tls`)
- Storage: 1Gi PVC (SQLite database — data survives pod restarts)
- **Monitored services:**
  - `https://argocd.vi3t-lab.com`
  - `https://grafana.vi3t-lab.com`
  - `https://pihole.vi3t-lab.com/admin`
  - `https://uptime.vi3t-lab.com`

### Minecraft
- Namespace: `minecraft`
- Image: `itzg/minecraft-server:latest`
- Type: Vanilla Java Edition, latest version
- IP: `192.168.30.193`
- Port: TCP 25565 (game), TCP 25575 (RCON)
- External access: `mc.vi3t-lab.com` via Cloudflare DNS A record → WAN IP → port forward
- World data: 5Gi PVC (survives pod restarts)
- RCON password: SealedSecret `minecraft-rcon` in `minecraft` namespace
- Query protocol enabled (`ENABLE_QUERY: true`) for Homepage widget
- **Port forward:** TCP 25565 → 192.168.30.193:25565
- **Cloudflare DNS:** A record `mc` → WAN IP (DNS only, NOT proxied — raw TCP)

### Glances
- Namespace: `glances`
- Type: DaemonSet — one pod per node automatically
- Image: `nicolargo/glances:latest-full`
- Purpose: Exposes per-node CPU, RAM, disk stats via API on port 61208
- Used by: Homepage dashboard node widgets
- Node IPs:
  - pi-master-0: `192.168.30.10:61208`
  - pi-worker-1: `192.168.30.11:61208`
  - pi-worker-2: `192.168.30.12:61208`
  - pi-worker-3: `192.168.30.13:61208`
- Runs with `hostNetwork: true` and `privileged: true` to read real hardware metrics

### Homepage Dashboard
- Namespace: `homepage`
- Image: `ghcr.io/gethomepage/homepage:latest`
- Version: v1.11.0
- IP: `192.168.30.194`
- Ingress: `dashboard.vi3t-lab.com` (TLS via cert-manager, secret: `homepage-tls`)
- Config: stored in `homepage-config` ConfigMap, copied into pod via initContainer
- Pi-hole API token: SealedSecret `pihole-api-token` in `homepage` namespace
- **Important — subPath caching issue:** Homepage uses an initContainer to copy ConfigMap files into an emptyDir. After updating the ConfigMap, always verify it updated before deleting the pod:
```bash
kubectl get configmap homepage-config -n homepage -o jsonpath='{.data.services\.yaml}' | grep <something-new>
```
- **HOMEPAGE_ALLOWED_HOSTS** must be set as env var in deployment (not in settings.yaml)
- **Widgets configured:**
  - openmeteo weather (Overland, MO)
  - datetime
  - Per-node CPU/RAM/disk via Glances
  - Pi-hole live query stats
  - Minecraft live status/players/version
- **Service groups:** Monitoring, DNS, VPN, Infrastructure, Network, Online, Games

---

## Installed Apps

| Service | Namespace | IP | Port(s) | Notes |
|---|---|---|---|---|
| Pi-hole | pihole | 192.168.30.191 | 53 UDP/TCP, 80 TCP | Local DNS + ad blocking |
| WireGuard | wireguard | 192.168.30.192 | 51820 UDP, 51821 TCP | VPN tunnel + web UI |
| Cloudflare DDNS | cloudflare-ddns | — | — | Keeps vpn.vi3t-lab.com pointed at WAN IP |
| Grafana | monitoring | 192.168.30.190 | 443 | grafana.vi3t-lab.com — via Traefik |
| Uptime Kuma | uptime-kuma | 192.168.30.190 | 443 | uptime.vi3t-lab.com — via Traefik |
| Minecraft | minecraft | 192.168.30.193 | 25565 TCP, 25575 TCP | mc.vi3t-lab.com — direct MetalLB |
| Glances | glances | per-node | 61208 TCP | DaemonSet — one pod per node |
| Homepage | homepage | 192.168.30.194 | 443 | dashboard.vi3t-lab.com — via Traefik |

---

## Firewall Rules (Ubiquiti)

| Rule | Action | Source | Destination | Notes |
|---|---|---|---|---|
| Block IoT to Trusted | Drop | 192.168.20.0/24 | 192.168.10.0/24 | IoT isolation |
| Block IoT to Homelab | Drop | 192.168.20.0/24 | 192.168.30.0/24 | IoT isolation |
| Block IoT to MGMT | Drop | 192.168.20.0/24 | 192.168.99.0/24 | IoT isolation |
| Allow Homelab to Trusted Established | Accept | 192.168.30.0/24 | 192.168.10.0/24 | Allows SSH return traffic + DNS responses |
| Block Homelab to Trusted New | Drop | 192.168.30.0/24 | 192.168.10.0/24 | Blocks unsolicited cluster → PC connections |

---

## Port Forwards (Ubiquiti)

| Name | Protocol | External Port | Internal IP | Internal Port |
|---|---|---|---|---|
| ArkServer | UDP | 7777 | 192.168.30.245 | 7777 |
| ArkServerAdmin | TCP | 27020 | 192.168.30.245 | 27020 |
| ArkFindMe | UDP | 27015 | 192.168.30.245 | 27015 |
| WireGuard VPN | UDP | 51820 | 192.168.30.192 | 51820 |
| Minecraft | TCP | 25565 | 192.168.30.193 | 25565 |

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
│   ├── cloudflare-ddns/
│   │   ├── namespace.yaml
│   │   ├── secret-sealed.yaml
│   │   ├── deployment.yaml
│   │   ├── kustomization.yaml
│   │   └── argocd-app.yaml
│   ├── wireguard/
│   │   ├── namespace.yaml
│   │   ├── secret-sealed.yaml
│   │   ├── deployment.yaml
│   │   ├── service.yaml
│   │   ├── kustomization.yaml
│   │   └── argocd-app.yaml
│   ├── monitoring/
│   │   ├── namespace.yaml
│   │   ├── grafana-admin-sealed.yaml
│   │   ├── helm-values.yaml
│   │   ├── kustomization.yaml
│   │   ├── argocd-app-prereqs.yaml
│   │   └── argocd-app.yaml
│   ├── uptime-kuma/
│   │   ├── namespace.yaml
│   │   ├── pvc.yaml
│   │   ├── deployment.yaml
│   │   ├── service.yaml
│   │   ├── ingress.yaml
│   │   ├── kustomization.yaml
│   │   └── argocd-app.yaml
│   ├── minecraft/
│   │   ├── namespace.yaml
│   │   ├── pvc.yaml
│   │   ├── secret-sealed.yaml
│   │   ├── deployment.yaml
│   │   ├── service.yaml
│   │   ├── kustomization.yaml
│   │   └── argocd-app.yaml
│   ├── glances/
│   │   ├── namespace.yaml
│   │   ├── daemonset.yaml
│   │   ├── service.yaml
│   │   ├── kustomization.yaml
│   │   └── argocd-app.yaml
│   └── homepage/
│       ├── namespace.yaml
│       ├── configmap.yaml
│       ├── pihole-secret-sealed.yaml
│       ├── deployment.yaml
│       ├── service.yaml
│       ├── ingress.yaml
│       ├── kustomization.yaml
│       └── argocd-app.yaml
├── _trash/
│   └── openvpn/
└── README.md
```

---

## Bootstrapping Pattern
> Any new ArgoCD Application must be bootstrapped once manually, then ArgoCD manages it forever after:
```bash
kubectl apply -f https://raw.githubusercontent.com/tranv1987-hash/k3s-cluster/refs/heads/main/<path>/argocd-app.yaml
```

---

## Chapter Roadmap
- ✅ **Chapter 1** — Infrastructure Groundwork (ArgoCD, MetalLB, Sealed Secrets, Cert-Manager)
- ✅ **Chapter 2** — Core Services (Pi-hole + local DNS, WireGuard VPN)
- ✅ **Chapter 3** — Monitoring (kube-prometheus-stack: Grafana + Prometheus, Uptime Kuma)
- ✅ **Chapter 4** — Applications (Minecraft, Homepage Dashboard with Glances)
- ⏭️ **Chapter 5** — Proxmox Integration (expose Proxmox services via k3s ingress)

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

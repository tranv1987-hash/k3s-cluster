## Domain & DNS
- **Domain:** vi3t-lab.com
- **DNS Provider:** Cloudflare
- **Subdomain pattern:** `*.vi3t-lab.com` for all internal services
- **DNS resolution strategy:** Pi-hole handles local DNS for `*.vi3t-lab.com` → MetalLB IPs

## MetalLB
- **IP Pool:** `192.168.30.190 - 192.168.30.230`
- **Mode:** Layer 2 (ARP)
- **Traefik:** holds `192.168.30.190` (first IP in pool)
- **Pi-hole:** holds `192.168.30.191`
- **WireGuard:** holds `192.168.30.192`
- **Known issue:** `bgppeers.metallb.io` CRD shows OutOfSync due to large caBundle — fixed via `ignoreDifferences` in `argocd-app.yaml`. Harmless — L2 mode doesn't use BGP.

| Service | Namespace | Version | Sync | Health |
|---|---|---|---|---|
| ArgoCD | argocd | stable | Synced | Healthy |
| MetalLB | metallb-system | v0.14.9 | Synced | Healthy |
| Sealed Secrets | sealed-secrets | v0.27.3 | Synced | Healthy |
| Cert-Manager | cert-manager | v1.17.1 | Synced | Healthy |
| Pi-hole | pihole | latest | Synced | Healthy |
| Cloudflare DDNS | cloudflare-ddns | v1.15.1 | Synced | Healthy |
| WireGuard (wg-easy) | wireguard | v14 | Synced | Healthy |

> Always wrap values containing `$` in single quotes to prevent shell interpolation

### Pi-hole
- Local DNS for `*.vi3t-lab.com` → MetalLB IPs
- IP: `192.168.30.191`
- Web UI: `http://192.168.30.191/admin`

### WireGuard (wg-easy)
- VPN tunnel + web UI
- IP: `192.168.30.192`
- Web UI: `http://192.168.30.192:51821` (accessible from Trusted VLAN only)
- VPN tunnel: UDP 51820 (port forwarded from WAN)
- Clients connect via `vpn.vi3t-lab.com:51820`
- DNS for VPN clients: Pi-hole at `192.168.30.191`
- PASSWORD_HASH env var requires bcrypt hash — generate with:
  python3 -c "import bcrypt; print(bcrypt.hashpw(b'YOUR_PASSWORD', bcrypt.gensalt()).decode())"

### Cloudflare DDNS
- Image: favonia/cloudflare-ddns:latest
- Keeps vpn.vi3t-lab.com A record pointed at WAN IP
- Checks every 5 minutes
- API token: cloudflare-ddns-token SealedSecret in cloudflare-ddns namespace
- Token permissions required: Zone → Zone → Read, Zone → DNS → Edit

## Installed Apps

| Service | Namespace | IP | Port(s) | Notes |
|---|---|---|---|---|
| Pi-hole | pihole | 192.168.30.191 | 53 UDP/TCP, 80 TCP | Local DNS + ad blocking |
| WireGuard | wireguard | 192.168.30.192 | 51820 UDP, 51821 TCP | VPN tunnel + web UI |
| Cloudflare DDNS | cloudflare-ddns | — | — | Keeps vpn.vi3t-lab.com pointed at WAN IP |

## Firewall Rules (Ubiquiti)

| Rule | Action | Source | Destination | Notes |
|---|---|---|---|---|
| Block IoT to Trusted | Drop | 192.168.20.0/24 | 192.168.10.0/24 | IoT isolation |
| Block IoT to Homelab | Drop | 192.168.20.0/24 | 192.168.30.0/24 | IoT isolation |
| Block IoT to MGMT | Drop | 192.168.20.0/24 | 192.168.99.0/24 | IoT isolation |
| Allow Homelab to Trusted Established | Accept | 192.168.30.0/24 | 192.168.10.0/24 | Allows SSH return traffic + DNS responses |
| Block Homelab to Trusted New | Drop | 192.168.30.0/24 | 192.168.10.0/24 | Blocks unsolicited cluster → PC connections |

## Port Forwards (Ubiquiti)

| Name | Protocol | External Port | Internal IP | Internal Port |
|---|---|---|---|---|
| ArkServer | UDP | 7777 | 192.168.30.245 | 7777 |
| ArkServerAdmin | TCP | 27020 | 192.168.30.245 | 27020 |
| ArkFindMe | UDP | 27015 | 192.168.30.245 | 27015 |
| WireGuard VPN | UDP | 51820 | 192.168.30.192 | 51820 |

k3s-cluster/
├── core/
│   ├── argocd/
│   │   ├── kustomization.yaml
│   │   ├── argocd-cmd-params.yaml
│   │   ├── argocd-ingress.yaml
│   │   └── argocd-app.yaml
│   ├── metallb/
│   │   ├── kustomization.yaml
│   │   ├── metallb-native.yaml        # vendored v0.14.9 manifest
│   │   ├── ipaddresspool.yaml         # pool: 192.168.30.190-230
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
│   └── wireguard/
│       ├── namespace.yaml
│       ├── secret-sealed.yaml
│       ├── deployment.yaml
│       ├── service.yaml
│       ├── kustomization.yaml
│       └── argocd-app.yaml
├── _trash/
│   └── openvpn/                       # abandoned — no ARM64 support
└── README.md

## Chapter Roadmap
- ✅ **Chapter 1** — Infrastructure Groundwork (ArgoCD, MetalLB, Sealed Secrets, Cert-Manager)
- ✅ **Chapter 2** — Core Services (Pi-hole + local DNS, WireGuard VPN)
- ⏭️ **Chapter 3** — Monitoring: Grafana, Prometheus, Uptime Kuma
- ⏭️ **Chapter 4** — Applications: Minecraft (Cloudflare Tunnel), Custom Dashboard

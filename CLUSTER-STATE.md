# k3s Homelab Cluster State
> Keep this file updated at the end of every chapter. Paste raw URL into new Claude chats to resume.
> **Last updated:** 2026-11-20 — Network migration to UniFi switch, Pi-hole + ArgoCD fixes (see Change Log)

---

## Ground Rules (Always Follow)
- One step at a time — wait for terminal output before proceeding
- All deployments via ArgoCD + GitHub (GitOps) — no manual `kubectl apply` for new services
- No secrets in plain text ever — always use Sealed Secrets + kubeseal
- Explain the why behind each step
- Do not assume firewall rules — always ask
- Always use `--server-side --force-conflicts` when applying large manifests
- `kubectl` only works on **pi-master-0** (workers have no API server / kubeconfig)

---

## Internet & Gateway
- **ISP:** Spectrum, 500 Mbps plan (measured ~438 Mbps down / ~22.7 Mbps up)
- **Gateway:** UniFi Cloud Gateway Ultra (192.168.1.1)
  - Port 4 → USW-Lite-16-PoE (port 15) — trunk, the only LAN uplink
  - Port 5 → WAN (Spectrum modem)
  - Ports 1–3 → free (everything moved to the switch)
- **Why it matters:** cable upload is small (~22 Mbps). One heavy uploader can fill it and make the whole house feel "down." All cross-VLAN + internet traffic shares the single 1 Gbps uplink (gateway port 4 ↔ USW port 15).

---

## Network Layout

| Device | IP | VLAN | Notes |
|---|---|---|---|
| UniFi Cloud Gateway Ultra | 192.168.1.1 | 1 (Default) | Router, firewall, DHCP |
| UniFi USW-Lite-16-PoE | 192.168.1.69 | 1 (Default) | Main switch (replaced TP-Link) |
| UniFi U7 Lite | 192.168.1.107 | 1 (Default) | Wi-Fi AP, USW port 7 (PoE) |
| ASRock_Master | 192.168.99.67 | 99 (MGMT) | Proxmox 8.4.16 host — containers on VLAN 30 (tagged) |
| pi-master-0 | 192.168.30.10 | 30 | k3s master node |
| pi-worker-1 | 192.168.30.11 | 30 | k3s worker node |
| pi-worker-2 | 192.168.30.12 | 30 | k3s worker node |
| pi-worker-3 | 192.168.30.13 | 30 | k3s worker node |
| pve-node1 (EliteDesk) | 192.168.30.101 | 30 | Proxmox — AI Platform Lab |
| pve-node2 (EliteDesk) | 192.168.30.102 | 30 | Proxmox — AI Platform Lab |
| pve-node3 (EliteDesk) | 192.168.30.103 | 30 | Proxmox — AI Platform Lab |
| Advantech touchscreen (debian) | 192.168.30.158 | 30 | Proxmox management station (browser to ASRock); often asleep — no link is normal |
| MagicMirror Pi ("Mr. Pi") | 192.168.1.68 | 1 (Default) | Own power supply, USW port 9 |
| NVIDIA Shield TV | 192.168.20.142 | 20 (IoT) | Wired, USW port 10 |
| Friend's PC (NucBox M5 Ultra) | 192.168.40.99 | 40 (Isolated) | Internet-only, managed remotely by friend |
| ~~TP-Link TL-SG2210P V3~~ | ~~192.168.99.2~~ | — | Retired 2026-11 until further notice — kept for a future STP lab |

### VLANs
- **VLAN 1** — Default (network gear: gateway, switch, AP; MagicMirror)
- **VLAN 10** — Trusted (personal PC, phone) — Wi-Fi: **TrustMeBro**
- **VLAN 20** — IoT (streaming devices, gaming consoles) — Wi-Fi: **VIoT**
- **VLAN 30** — Homelab (k3s cluster, EliteDesks, Proxmox containers, Advantech)
- **VLAN 40** — Isolated (friend's PC) — internet only, "Isolate Network" on, DNS 1.1.1.1/1.0.0.1
- **VLAN 99** — Management (ASRock_Master host)

### Switch — USW-Lite-16-PoE (port map)
> **Why this switch:** replaced the TP-Link so the whole network is managed in one UniFi dashboard, powers the Pis + AP over PoE, shows per-port watts/traffic, and freed gateway ports.
> **Rule:** VLAN settings belong to the **port**, not the device — when you move a cable, update that port's settings.
> Ports 1–8 are PoE+ (45W total budget). Ports 9–16 have no PoE.

| Port | Device | Mode | Native VLAN | Tagged | PoE |
|---|---|---|---|---|---|
| 1 | EliteDesk (pve-node1 or 3) | Edge | Homelab | Block All | Off |
| 2 | pi-master-0 | Edge | Homelab | Block All | PoE+ |
| 3 | EliteDesk (pve-node2) | Edge | Homelab | Block All | Off |
| 4 | pi-worker-1 | Edge | Homelab | Block All | PoE+ |
| 5 | EliteDesk (pve-node1 or 3) | Edge | Homelab | Block All | Off |
| 6 | pi-worker-2 | Edge | Homelab | Block All | PoE+ |
| 7 | U7 Lite | Infrastructure | Default | Custom: Trusted + IoT | PoE+ |
| 8 | pi-worker-3 | Edge | Homelab | Block All | PoE+ |
| 9 | MagicMirror | Edge | Default | Block All | — |
| 10 | NVIDIA Shield | Edge | IoT (20) | Block All | — |
| 11 | ASRock_Master | Edge | MGMT (99) | Custom: Homelab (30) | — |
| 12 | Advantech | Edge | Homelab (30) | Block All | — |
| 13 | Desktop PC | Edge | Trusted (10) | Block All | — |
| 14 | Friend's PC | Edge | Isolated (40) | Block All | — |
| 15 | Uplink → gateway port 4 | Infrastructure | Default | Allow All | — |
| 16 | Spare (was TP-Link bridge) | — | — | — | Disabled |

**Why some ports differ:**
- **Pis on the bottom row (2/4/6/8):** Pi tray sits below the switch — cables drop straight down.
- **U7 (port 7):** AP's own IP is on Default (untagged); Wi-Fi networks ride tagged. Only Trusted + IoT tagged (least privilege) — **add any new Wi-Fi VLAN here** or its clients get no IP. Infrastructure mode keeps loop protection on (APs can mesh).
- **ASRock (port 11):** Proxmox host is untagged on VLAN 99; its containers send VLAN 30 **tagged** (`tag=30` in each LXC config). Port needs both.
- **Edge mode:** end device, not a switch → port comes up fast.
- **Unused ports disabled:** a live unused port is an open door.

### PoE Power (measured 2026-11-14)
- Each Pi 4B + PoE HAT: **~3.6W idle**, **~6W boot peak** (~20 sec), blips to ~5.6W when k3s starts pods
- U7 Lite: **13W max** (spec)
- Worst case (4 Pis booting + U7 maxed): ~37W of 45W budget
- **Log per-port power live** (SSH into the switch):
```bash
ssh <device-ssh-user>@192.168.1.69
swctrl poe show                      # all PoE ports, watts/volts/mA
while true; do echo "$(date +%T) $(swctrl poe show | grep '^ *2 ')"; sleep 1; done | tee /tmp/poe-boot.log   # one port, every second
```

### Wi-Fi
| SSID | Network | VLAN |
|---|---|---|
| TrustMeBro | Trusted | 10 |
| VIoT | IoT | 20 |

### Proxmox Containers (VLAN 30, on ASRock_Master)
| Container | ID | IP |
|---|---|---|
| npm (Nginx Proxy Manager) | 100 | 192.168.30.66 |
| jellyfin | 101 | 192.168.30.65 |
| nextcloud | 102 | 192.168.30.64 |
| palworld | 103 | 192.168.30.245 |
| audiobookshelf | 104 | 192.168.30.63 |
| navidrome | 105 | 192.168.30.62 |
| kavita | 106 | 192.168.30.61 |
| arrstack | 107 | 192.168.30.60 |
- All containers use `bridge=vmbr0,tag=30`. Find which container owns a MAC: `grep -ril "<mac>" /etc/pve/`
- VietNas runs inside the ASRock (Jellyfin ↔ NAS traffic never leaves the box)

### EliteDesks (3x HP EliteDesk 705 G4)
- pve-node1/2/3 at `192.168.30.101–103`, each running Proxmox VE
- Host k3s VMs for the **AI Platform Lab** (separate repo: `github.com/tranv1987-hash/ai-platform-lab`) — not part of this Pi cluster

---

## Domain & DNS
- **Domain:** vi3t-lab.com
- **DNS Provider:** Cloudflare
- **Subdomain pattern:** `*.vi3t-lab.com` for all internal services
- **DNS resolution strategy:** Pi-hole handles local DNS for `*.vi3t-lab.com` → MetalLB IPs
- **If Pi-hole is down,** `*.vi3t-lab.com` names won't resolve (ArgoCD/Pi-hole web UIs "won't load"). Emergency workaround: add `192.168.30.190 argocd.vi3t-lab.com` etc. to the desktop hosts file (`C:\Windows\System32\drivers\etc\hosts`).

---

## k3s Cluster

### Nodes
- 4x Raspberry Pi 4B running Raspberry Pi OS (Debian Trixie), powered by PoE from the USW
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

### Safe Shutdown (before moving Pi cables / cutting PoE)
> Unplugging a PoE cable = pulling the power cord. Shut down cleanly first — workers first, master last.
```bash
ssh tranv@192.168.30.11 'sudo shutdown -h now'
ssh tranv@192.168.30.12 'sudo shutdown -h now'
ssh tranv@192.168.30.13 'sudo shutdown -h now'
ssh tranv@192.168.30.10 'sudo shutdown -h now'
```
After power returns: `kubectl get nodes -o wide` (all Ready, same IPs) and `kubectl get pods -A | grep -v Running`

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
- **Normal, not a bug:** UniFi may list a Pi with a service IP (e.g. pi-worker-2 shown at `.193`). In L2 mode one node answers ARP for each service IP, so UniFi sees that node's MAC on it.
- ⚠️ **Service IPs are not pinned** — they could change if a service is deleted/recreated. Firewall rules and DHCP DNS depend on them (especially Pi-hole `.191`). TODO: pin Pi-hole.

---

## Installed Core Services

| Service | Namespace | Version | Sync | Health |
|---|---|---|---|---|
| ArgoCD | argocd | stable | Synced | Healthy |
| MetalLB | metallb-system | v0.14.9 | Synced | Healthy |
| Sealed Secrets | sealed-secrets | v0.27.3 | Synced | Healthy |
| Cert-Manager | cert-manager | v1.17.1 | Synced | Healthy |
| Pi-hole | pihole | 2024.07.0 (v5) | Synced | Healthy |
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
- **Health check (no web UI needed):** `kubectl get applications -n argocd`
  - **SYNC STATUS = Unknown on every app** → `argocd-repo-server` is down (it's the part that downloads the repo). Commits will NOT be applied.
  - Fix: `kubectl get pods -n argocd` → if repo-server is `Unknown`, delete it so the Deployment recreates it:
```bash
kubectl delete pod <argocd-repo-server-pod> -n argocd
# if it hangs: add --force --grace-period=0
```
- Force a re-check after a commit: `kubectl annotate application <app> -n argocd argocd.argoproj.io/refresh=hard --overwrite`

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
- DNS server for **Trusted, IoT** Wi-Fi and **VPN** clients
- IP: `192.168.30.191`
- Image: `pihole/pihole:2024.07.0` (Pi-hole **v5** — settings via env vars)
- Storage: `/etc/pihole` on PVC `pihole-pvc`
- Web UI: `https://pihole.vi3t-lab.com/admin` (note: must use `/admin` path)
- **`DNSMASQ_LISTENING: "all"`** set in `apps/pihole/deployment.yaml`
  - **Why:** default ("local only") refuses queries from other VLANs → phones on TrustMeBro/VIoT got `Query refused` → "limited connectivity." Firewall rules still control who can reach port 53.
- **Test from desktop:** `nslookup google.com 192.168.30.191` (want an address, not "Query refused" / timeout)
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

## Firewall Rules (UniFi — zone-based)
> UniFi converted the old rules into zones. Each rule exists in several copies (Hotspot / Vpn / Internal / External). **Only the `Internal → Internal` copy matters for VLAN-to-VLAN traffic** — the rest are harmless clutter (cleanup TODO).

| Rule | Action | Source | Destination | Notes |
|---|---|---|---|---|
| Block IoT to Trusted | Drop | IoT (192.168.20.0/24) | Trusted (192.168.10.0/24) | IoT isolation |
| Allow IoT to Jellyfin | Accept | IoT | 192.168.30.65 TCP 8096 | Streaming boxes → Jellyfin |
| Allow IoT DNS to Pihole | Accept | IoT | **192.168.30.191** TCP/UDP 53 | Fixed 2026-11-17 — was `.190` (Traefik) |
| Block IoT to Homelab | Drop | IoT | Homelab (192.168.30.0/24) | IoT isolation |
| Block IoT to MGMT | Drop | IoT | MGMT (192.168.99.0/24) | IoT isolation |
| Allow Homelab to Trusted Established | Accept | Homelab | Trusted | Allows SSH return traffic + DNS responses |
| Block Homelab to Trusted New | Drop | Homelab | Trusted | Blocks unsolicited cluster → PC connections |
| Allow Isolated DHCP | Accept | Isolated | Gateway UDP 67 | Friend's PC can get an IP (must be above the block) |
| Block Isolated to Gateway | Drop | Isolated | Gateway | Friend's PC can't touch the gateway |

- **Why the IoT DNS fix:** the allow rule pointed at Traefik (.190), so every IoT DNS question hit "Block IoT to Homelab." VIoT had Wi-Fi + internet but no name lookups.
- Rule ordering matters: Allow rules must sit **above** the matching Block rules.

### QoS (Policy Engine → QoS)
| Rule | Objective | Source | Interface | Limit |
|---|---|---|---|---|
| Limit friend-pc | Limit | Network: Isolated (40) | WAN1 | 200 Mbps down / 10 Mbps up, burst off |
- **Why:** friend's PC (multiple Android emulators + remote access) could fill the ~22 Mbps upload lane and stall everyone. Cap = safety ceiling, not a squeeze.
- Smart Queues **not used** — Ubiquiti advises against them above ~300 Mbps (would slow the 500 Mbps download).

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
│   ├── pihole/
│   │   ├── namespace.yaml
│   │   ├── admin-secret-sealed.yaml
│   │   ├── pvc.yaml
│   │   ├── deployment.yaml
│   │   ├── service-dns.yaml
│   │   ├── service-web.yaml
│   │   ├── ingress.yaml
│   │   ├── kustomization.yaml
│   │   └── argocd-app.yaml
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

## UniFi Gotchas
- **Adoption drops right after "Adopt" (Click to Resolve / Connection Interrupted):** a half-finished adoption. Fix: Remove device in UniFi → hold reset pinhole ~10s (LED white) → adopt again.
- **Device can't find controller** (`info` shows `Unable to resolve (http://unifi:8080/inform)`): SSH in (`ubnt`/`ubnt` after reset) and run `set-inform http://192.168.1.1:8080/inform`.
- **Port VLAN follows the port, not the device** — after moving any cable, check the new port's Native VLAN.
- **Client list port/IP can be stale** — trust a physical cable trace or the switch MAC table over UniFi's "connected to" view.
- **Loops:** never give a switch two paths to the network. RSTP will usually block one, but don't rely on it.
- **Restarting the gateway** clears stuck DHCP/WAN state but hides the cause — check WAN events and note the time first.

---

## Open Issues / TODO
- [ ] `local-path-provisioner` in `local-path-storage` namespace is CrashLoopBackOff (35k+ restarts) — looks like a duplicate of the healthy one in `kube-system`
- [ ] Pin Pi-hole's MetalLB IP to `.191` in its service YAML (firewall + DHCP depend on it)
- [ ] Remove VLAN 40 (Isolated) from mDNS reflection — gateway leaks the Roku's AirPlay/Spotify announcements into the friend's VLAN
- [ ] Clean up duplicate zone firewall rules (Hotspot/Vpn/External copies)
- [ ] Internet drops: keep `ping -t 1.1.1.1` running on wired desktop; on next drop check UniFi WAN events + friend-pc live upload (short time range)
- [ ] Re-check USW port 13/14 STP anomaly score (flapping seen during swap only)
- [ ] Confirm which EliteDesk (pve-node1 vs pve-node3) is on USW port 1 vs 5
- [ ] Ark port forwards (UDP 7777, TCP 27020, UDP 27015) still point at `.245`, which is now **palworld** (CT 103) — update or remove
- [ ] STP lab with retired TP-Link (two cables USW ↔ TP-Link — never through the gateway)

---

## Change Log

### 2026-11-14 → 2026-11-20 — Network migration + DNS/ArgoCD fixes
| Change | Why |
|---|---|
| Replaced TP-Link TL-SG2210P with UniFi USW-Lite-16-PoE | One dashboard for gateway/switch/AP, PoE for Pis + U7, per-port watts/traffic, free gateway ports |
| Adopted USW via Trusted network; reset + re-adopt after a broken first adoption | First adoption half-finished after a firmware reboot (scrambled key) |
| Moved Pis to ports 2/4/6/8 with graceful shutdown | Cleaner cabling to the Pi tray; avoid SD card corruption from cutting PoE |
| EliteDesks on 1/3/5, U7 on 7 (Trusted + IoT tagged only) | Least privilege on the AP port; retired the small unmanaged PoE switch |
| ASRock port = native MGMT + tagged Homelab | Proxmox host lives on VLAN 99, containers tag VLAN 30 |
| Firewall "Allow IoT DNS to Pihole" `.190` → `.191` | Rule pointed at Traefik — VIoT had no DNS |
| QoS cap on Isolated network (10 up / 200 down) | Protect the small ~22 Mbps upload from the friend's emulator/remote-access traffic |
| Pi-hole `DNSMASQ_LISTENING: "all"` (GitOps) | Pi-hole was refusing queries from Trusted/IoT → Wi-Fi "limited connectivity" |
| Deleted stuck `argocd-repo-server` pod (status Unknown) | Repo-server was dead since a node reboot → every app "Unknown", no commits were being applied |
| Added patch panel + 0.5 ft Cat6 28AWG pure-copper jumpers | Front-facing switch ports; no punch-down; neater runs |

---

## Chapter Roadmap
- ✅ **Chapter 1** — Infrastructure Groundwork (ArgoCD, MetalLB, Sealed Secrets, Cert-Manager)
- ✅ **Chapter 2** — Core Services (Pi-hole + local DNS, WireGuard VPN)
- ✅ **Chapter 3** — Monitoring (kube-prometheus-stack: Grafana + Prometheus, Uptime Kuma)
- ✅ **Chapter 4** — Applications (Minecraft, Homepage Dashboard with Glances)
- ⏭️ **Chapter 5** — Bonus chapter

---

## Cert Roadmap (Owner)
- ✅ CompTIA A+ Core 1
- ⏳ CompTIA A+ Core 2 (will not persue)
- ⏳ CCNA
- ⏳ CompTIA Security+
- ⏳ AWS Cloud Practitioner
- ⏳ AWS Solutions Architect – Associate
- ⏳ AWS Certified DevOps Engineer – Professional
- ⏳ Terraform + Ansible (IaC)
- ⏳ Linux Essentials (Red Hat on backburner)

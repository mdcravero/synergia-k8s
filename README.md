# Synergia K8s — Home Lab

Kubernetes home lab running on Talos Linux, managed with Flux CD (GitOps). All configuration is declarative and version-controlled; secrets are encrypted at rest with SOPS + age.

## Hardware

| Node | Role | SoC | Arch |
|---|---|---|---|
| synergia-01 | Control plane | Raspberry Pi 4 (8GB) | ARM64 |
| synergia-02 | Control plane | Raspberry Pi 4 (8GB) | ARM64 |
| synergia-03 | Control plane | Raspberry Pi 4 (8GB) | ARM64 |
| synergia-05 | Worker | Radxa X4 (8GB) | AMD64 |

**NFS storage server:** `10.42.20.10`
**Zigbee coordinator:** SMLIGHT SLZB-06U (CC2652P via TCP at `10.42.30.30:6638`)

## Core Stack

| Component | Purpose |
|---|---|
| [Talos Linux](https://www.talos.dev/) | Immutable, API-managed OS |
| [Flux CD](https://fluxcd.io/) | GitOps controller |
| [SOPS + age](https://github.com/getsops/sops) | Secret encryption |
| [MetalLB](https://metallb.universe.tf/) | Bare-metal load balancer (L2, pool `10.42.20.40–10.42.20.90`) |
| [Traefik](https://traefik.io/) | Ingress controller + reverse proxy |
| [cert-manager](https://cert-manager.io/) | Automatic TLS via Cloudflare DNS-01 (wildcard `*.synergia.net.ar`) |
| [NFS Provisioner](https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner) | Dynamic persistent storage |
| [Authelia](https://www.authelia.com/) | SSO / forward auth for internal services |
| [Renovate](https://docs.renovatebot.com/) | Automated dependency updates (self-hosted) |
| [Goldilocks](https://goldilocks.docs.fairwinds.com/) | VPA-based resource recommendations |

## Applications

### Monitoring (`monitoring` namespace)

| App | Purpose | URL |
|---|---|---|
| Prometheus | Metrics collection + AlertManager (Telegram alerts) | `prometheus.synergia.net.ar` |
| Grafana | Dashboards (Node Exporter, K8s, PostgreSQL, Flux, Alertmanager) | `grafana.synergia.net.ar` |
| Loki + Promtail | Log aggregation (30-day retention) | — |
| Goldilocks | Resource usage recommendations | `goldilocks.synergia.net.ar` |

### Tools (`tools` namespace)

| App | Purpose | URL / IP |
|---|---|---|
| PostgreSQL | Database cluster via Zalando operator (multi-arch ARM64+AMD64) | `10.42.20.70` |
| Valkey | Redis-compatible in-memory cache | ClusterIP |
| Home Assistant | Home automation | `ha.synergia.net.ar` |
| n8n | Workflow automation | `n8n.synergia.net.ar` |
| Mosquitto | MQTT broker for Zigbee integration | ClusterIP `1883` |
| Zigbee2MQTT | Zigbee coordinator bridge (SLZB-06U via TCP) | `zigbee2mqtt.synergia.net.ar` |
| Homarr | Homelab dashboard | `homarr.synergia.net.ar` |
| Homebox | Home inventory management | `homebox.synergia.net.ar` |
| CouchDB | Document database | ClusterIP |
| Transmission Movies | BitTorrent client | `10.42.20.69` |
| Transmission Shows | BitTorrent client | `10.42.20.68` |

### Media (`media` namespace)

| App | Purpose | URL / IP |
|---|---|---|
| Jellyfin | Media server | `jellyfin.synergia.net.ar` — `10.42.20.60` |
| Jellyseerr | Media request management | `jellyseerr.synergia.net.ar` — `10.42.20.61` |
| Sonarr | TV series automation | `sonarr.synergia.net.ar` — `10.42.20.64` |
| Radarr | Movie automation | `radarr.synergia.net.ar` — `10.42.20.63` |
| Prowlarr | Indexer aggregator | `prowlarr.synergia.net.ar` — `10.42.20.62` |
| Bazarr | Subtitle management | `bazarr.synergia.net.ar` — `10.42.20.67` |
| Maintainerr | Media library cleanup | `maintainerr.synergia.net.ar` |

## Zigbee Stack

```
SMLIGHT SLZB-06U (CC2652P)
  TCP 10.42.30.30:6638
          │
          ▼
    Zigbee2MQTT ──── MQTT ──── Mosquitto ──── Home Assistant
    (bridge)                   (broker)       (MQTT integration)
```

- Coordinator connects over LAN — no USB device passthrough required
- Zigbee2MQTT publishes device discovery to Home Assistant automatically via MQTT
- Devices paired in Zigbee2MQTT appear as entities in Home Assistant immediately

## Backup Strategy

| Target | Method | Schedule | Destination |
|---|---|---|---|
| etcd snapshots | `talosctl etcd snapshot` via CronJob | Daily 3 AM UTC | Mega.nz via rclone |
| Home Assistant | Tar + rclone via CronJob | Daily | Mega.nz via rclone |
| PostgreSQL | Zalando logical backup (multi-arch image) | Daily | S3-compatible |

## Secrets Management

All secrets are encrypted with **SOPS + age** and stored in `bootstrap/kubernetes/apps/secrets/`. The age recipient key is stored securely offline.

```bash
# Encrypt a new secret in-place
sops -e -i bootstrap/kubernetes/apps/secrets/secret-myapp.yaml

# Edit an existing encrypted secret
sops bootstrap/kubernetes/apps/secrets/secret-myapp.yaml
```

## Project Structure

```
bootstrap/kubernetes/apps/
├── flux-system/          # Flux CD sync configuration
├── namespaces/           # All namespace definitions (centralized)
├── system/
│   ├── cert-manager/     # TLS certificate management
│   └── metrics-server/   # Resource metrics for kubectl top / HPA
├── network/
│   ├── metallb/          # L2 load balancer
│   └── traefik/          # Ingress controller + per-namespace Authelia middlewares
├── storage/
│   └── nfs-provisioner/  # Dynamic NFS storage (storageClass: nfs-client)
├── security/
│   └── authelia/         # Forward auth / SSO
├── monitoring/
│   ├── prometheus/       # Prometheus + AlertManager
│   ├── grafana/          # Dashboards
│   ├── loki-stack/       # Loki + Promtail
│   └── goldilocks/       # VPA recommendations
├── tools/                # See Tools section above
├── media/                # See Media section above
└── secrets/              # SOPS-encrypted secrets
```

## Flux Deployment Order

```
Namespaces
    └── cert-manager + metrics-server
            └── MetalLB
                    └── Traefik + middlewares
                            └── NFS provisioner
                                    └── Secrets (SOPS)
                                            └── Monitoring
                                                └── Security (Authelia)
                                                    └── Tools (PostgreSQL → Valkey → apps)
                                                        └── Media
```

## Useful Commands

```bash
# Flux status
flux get kustomizations

# Force reconciliation
flux reconcile kustomization apps --with-source
flux reconcile helmrelease n8n -n tools --force

# Node and pod resource usage
kubectl top nodes
kubectl top pods -A --sort-by=memory

# Run Renovate manually
kubectl create job --from=cronjob/renovate renovate-manual -n tools
kubectl logs -n tools -l job-name=renovate-manual -f

# Zigbee2MQTT / Mosquitto logs
kubectl logs -n tools deployment/zigbee2mqtt -f
kubectl logs -n tools deployment/mosquitto -f

# Trigger etcd backup manually
kubectl create job --from=cronjob/etcd-backup etcd-backup-manual -n tools

# Certificate status
kubectl get certificates -A
kubectl get certificaterequests -A
```

## Renovate Update Policy

| Package group | Schedule | Patch auto-merge |
|---|---|---|
| n8n, Home Assistant, Traefik, Authelia | Any time (as soon as detected) | n8n / HA: yes — Traefik / Authelia: no |
| Media stack | Mondays before 9 AM ART | No (single grouped PR) |
| Infrastructure (cert-manager, MetalLB) | Mondays before 9 AM ART | No |
| Everything else | Mondays before 9 AM ART | Yes |
| Major versions (all packages) | Mondays — requires Dependency Dashboard approval | No |

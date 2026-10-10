# Synergia K8s — Home Lab

Kubernetes home lab running on Talos Linux and managed with Flux CD (GitOps).
All configuration is declarative and version-controlled; secrets are encrypted
at rest with SOPS + age.

_Last updated: 2026-10-10._

## At a Glance

| Layer | Choice |
|---|---|
| OS | [Talos Linux](https://www.talos.dev/) v1.11.6 |
| Kubernetes | v1.34.1 (3 control-plane nodes + 1 worker) |
| GitOps | [Flux CD](https://fluxcd.io/) v2.9 |
| CNI | [Cilium](https://cilium.io/) 1.20.2 (VXLAN overlay, kube-proxy kept) |
| Load balancing | [MetalLB](https://metallb.io/) (L2) |
| Ingress / TLS | [Traefik](https://traefik.io/) v3 + [cert-manager](https://cert-manager.io/) (Cloudflare DNS-01) |
| Auth | [Authelia](https://www.authelia.com/) (forward auth, 2FA, OIDC) |
| Storage | NFS (default) + [Longhorn](https://longhorn.io/) 1.13 (local block storage on the worker) |
| Database | PostgreSQL 17 via the [Zalando operator](https://github.com/zalando/postgres-operator) |
| Observability | Prometheus + Alertmanager (Telegram), Grafana, Loki |
| Updates | Self-hosted [Renovate](https://docs.renovatebot.com/) |

## Hardware

| Node | Role | Board | Arch | RAM | Disks |
|---|---|---|---|---|---|
| synergia-01 | Control plane | Raspberry Pi 4 | ARM64 | 8 GB | 64 GB USB SSD |
| synergia-02 | Control plane | Raspberry Pi 4 | ARM64 | 8 GB | 64 GB USB SSD |
| synergia-03 | Control plane | Raspberry Pi 4 | ARM64 | 8 GB | 64 GB USB SSD |
| synergia-05 | Worker | Radxa X4 (Intel N100) | AMD64 | 8 GB | NVMe (system) + 62 GB eMMC (Longhorn) |

Outside the cluster:

- **NFS server.** It backs the default StorageClass and stores the Longhorn
  backups.
- **Zigbee coordinator.** A network-attached SMLIGHT SLZB-06U (CC2652P).

## Talos

The machine configs live in a separate repository. The settings that matter
here:

- **Image Factory schematics.** All nodes carry `iscsi-tools` and
  `util-linux-tools`, which Longhorn needs. The worker also has `i915` and
  `intel-ucode` for the iGPU that Jellyfin uses.
- **API endpoint.** The control-plane endpoint is a Talos shared VIP. KubePrism
  serves the API locally on every node, and Cilium uses it.
- **Longhorn disk.** The worker mounts its eMMC as a Talos user volume at
  `/var/mnt/longhorn`.
- **Image garbage collection.** kubelet deletes images that have been unused
  for 7 days (`imageMaximumGCAge: 168h`). It also trims unused images once
  `/var` passes 70%, down to 60%. On the Raspberry Pis, `/var` shares its USB
  disk with etcd.

## Networking

- **CNI.** Cilium runs on every node: cluster-pool IPAM with one `/24` per
  node, and a VXLAN tunnel. kube-proxy still handles services alongside it.
  - Installed with Helm from
    `bootstrap/kubernetes/apps/network/cilium/app/helm-values-migration.yaml`;
    Flux does not manage it.
  - The flannel DaemonSet that Talos deployed at bootstrap is still present,
    but no pod traffic goes through it.
  - See [MIGRATION-FROM-FLANNEL.md](bootstrap/kubernetes/apps/network/cilium/MIGRATION-FROM-FLANNEL.md).
- **LoadBalancer services.** MetalLB announces them in L2 mode from a pool on
  the server LAN. Traefik holds the main ingress IP.
- **External access.** Public hostnames are proxied by Cloudflare to Traefik.
  cert-manager issues a wildcard certificate through a Cloudflare DNS-01
  challenge.

## Storage

| StorageClass | Backend | Used by |
|---|---|---|
| `nfs-client` (default) | NFS subdir provisioner | Most apps, the media library, the PostgreSQL data volume |
| `longhorn` | Longhorn 1.13 on the worker's eMMC: 1 replica, `Retain` | OpenClaw and Jellyfin's config |

Longhorn exists for workloads whose SQLite databases misbehave on NFS. All of
its volumes are backed up every day to the NFS server. See
[storage/longhorn/README.md](bootstrap/kubernetes/apps/storage/longhorn/README.md).

## Ingress and Access Control

Traefik terminates TLS for every hostname:

- **Administrative UIs** go through Authelia forward auth (default policy:
  deny), with two-factor for the most sensitive ones. Each namespace has its
  own Traefik middleware.
- **Other apps** handle their own authentication.
- **OIDC.** Authelia also acts as an OIDC provider; Homebox signs in with it.
  Its storage is PostgreSQL.

See [security/authelia/README.md](bootstrap/kubernetes/apps/security/authelia/README.md).

## Applications

### Agents (`agents` namespace)

| App | Purpose |
|---|---|
| OpenClaw | Personal AI agent on Telegram, integrated with Home Assistant. It runs on the worker, with its state on Longhorn. |

### Monitoring (`monitoring` namespace)

| App | Purpose |
|---|---|
| Prometheus | Metrics, alert rules, Alertmanager → Telegram |
| Grafana | Dashboards |
| Loki + Promtail | Log aggregation (30-day retention) |
| Goldilocks + VPA recommender | Resource request recommendations |

### Tools (`tools` namespace)

| App | Purpose |
|---|---|
| PostgreSQL | Shared database (see below) |
| Valkey | Redis-compatible cache (primary + replicas) |
| Home Assistant | Home automation; the recorder stores history in PostgreSQL |
| n8n | Workflow automation |
| Mosquitto | MQTT broker |
| Zigbee2MQTT | Zigbee bridge |
| Homarr | Homelab dashboard |
| Homebox | Home inventory (OIDC login via Authelia) |
| CouchDB | Document database |

### Media (`media` namespace)

| App | Purpose |
|---|---|
| Jellyfin | Media server, with hardware transcoding on the worker's Intel iGPU (VA-API); config on Longhorn |
| Seerr | Media requests |
| Prowlarr | Indexer manager |
| Radarr / Sonarr / Lidarr | Movies / TV series / music |
| Bazarr | Subtitles |
| SABnzbd | Usenet downloader |
| Transmission (movies / shows / music) | BitTorrent clients |
| Maintainerr | Library cleanup |

## PostgreSQL

A single-instance cluster, `postgres-cluster`, runs under the Zalando
operator:

- **Image and placement.** PostgreSQL 17 on a custom Spilo image (UID 1000,
  arm64). It is pinned to the Raspberry Pi nodes, and its data volume is on
  NFS.
- **Endpoint.** Patroni keeps its leader and configuration in ConfigMaps, and
  the master Service selects the pod labelled `spilo-role=master`. Kubernetes
  therefore keeps the endpoint on the pod's real IP.
- **Restarts.** On a Raspberry Pi a pod restart means about 3–6 minutes
  without a database, because Spilo decompresses itself on every start.
- **Cordoning.** The operator moves the master off any node that becomes
  unschedulable. Cordoning its node therefore restarts PostgreSQL.

See [tools/postgresql/README.md](bootstrap/kubernetes/apps/tools/postgresql/README.md)
and [PATRONI-DCS-CONFIGMAPS.md](bootstrap/kubernetes/apps/tools/postgresql/PATRONI-DCS-CONFIGMAPS.md).

## Zigbee Stack

```
SMLIGHT SLZB-06U (CC2652P)
  (TCP, IoT network)
          │
          ▼
    Zigbee2MQTT ──── MQTT ──── Mosquitto ──── Home Assistant
    (bridge)                   (broker)       (MQTT integration)
```

- The coordinator is reached over the network, so no USB passthrough is
  needed and Zigbee2MQTT can run on any node.
- Zigbee2MQTT publishes device discovery to Home Assistant over MQTT.

## Monitoring and Alerting

Prometheus scrapes:
- the nodes (node-exporter) and kube-state-metrics;
- Traefik, the Flux controllers and the PostgreSQL exporter.

Alertmanager sends to Telegram:
- It groups by alert name, namespace and instance, so each node or pod gets
  its own message.
- Critical alerts repeat every 4 h, warnings every 24 h.

| Group | Alerts |
|---|---|
| nodes | `NodeFilesystemAlmostFull` (>80%), `NodeFilesystemCritical` (>90%), `NodeRebooted`, `NodeNotReady` |
| pods | `PodCrashLooping` (3+ restarts in 15 min) |
| postgresql | `PostgresqlNotResponding`, `PostgresqlExporterUnreachable`, `PostgresqlHighConnectionCount`, `PostgresqlMasterEndpointStale`, `HomeAssistantRecorderDisconnected` |
| backups | `BackupCronJobFailed`, `BackupCronJobNotScheduled` |
| flux | `FluxKustomizationFailed`, `FluxHelmReleaseFailed` |
| renovate | `RenovateJobFailed`, `RenovateNoSuccessfulRun` |
| certificates | `TalosConfigCertExpiringSoon` |

Grafana dashboards: Node Exporter Full, Kubernetes cluster, PostgreSQL and
Flux cluster stats. Its datasources are Prometheus and Loki.

## Backups

| What | How | When (UTC) | Where |
|---|---|---|---|
| etcd | `talosctl etcd snapshot` CronJob (`backup-system`) | Daily 03:00 | Off-site cloud storage (rclone) |
| Home Assistant | HA backups copied by a CronJob | Daily 09:00 | Off-site cloud storage (rclone) |
| PostgreSQL | Zalando logical backup (`pg_dumpall`) | Daily 02:00 | S3-compatible storage, 1-week retention |
| Longhorn volumes | Longhorn `RecurringJob` `backup-daily` | Daily 06:00 | NFS server, 7 kept |

Another daily CronJob checks the expiry of the talosconfig client certificate
that the etcd backup uses; see
[TALOSCONFIG-CERT-RENEWAL.md](bootstrap/kubernetes/apps/tools/backup/TALOSCONFIG-CERT-RENEWAL.md).
Each backup CronJob keeps its last 3 successful and 3 failed Jobs.

## Secrets Management

All secrets are encrypted with **SOPS + age** and stored in
`bootstrap/kubernetes/apps/secrets/`. Flux decrypts them in the
`apps-secrets-config` Kustomization. The age key is stored offline.

```bash
# Encrypt a new secret in place
sops -e -i bootstrap/kubernetes/apps/secrets/secret-myapp.yaml

# Edit an existing encrypted secret
sops bootstrap/kubernetes/apps/secrets/secret-myapp.yaml
```

Before committing a change to a secret, compare its key names with the
previous version. A key dropped by mistake leaves the pod in
`CreateContainerConfigError`. See [secrets/README.md](bootstrap/kubernetes/apps/secrets/README.md).

## Config Reload

Kubernetes does not restart pods when a mounted ConfigMap or Secret changes.
[Reloader](https://github.com/stakater/Reloader) triggers a rolling restart
when a referenced resource is updated. Workloads opt in through a pod template
annotation:

```yaml
spec:
  template:
    metadata:
      annotations:
        configmap.reloader.stakater.com/reload: "openclaw-config"
        secret.reloader.stakater.com/reload: "secret-openclaw"
```

`reloader.stakater.com/auto: "true"` watches every ConfigMap and Secret the
workload references.

## Dependency Updates (Renovate)

Renovate runs in-cluster as a daily CronJob, using
[renovate.json](renovate.json). A GitHub Actions workflow posts every new
Renovate PR to Telegram.

| Rule | Packages | Schedule | Auto-merge |
|---|---|---|---|
| Default | Everything not listed below | Mondays before 9 AM (ART) | Patch and digest |
| Frequent releases | n8n, Home Assistant, Traefik | Any time | Patch (n8n, HA) |
| Media stack | Jellyfin, *arr apps, Seerr, Maintainerr, Transmission | Mondays, one grouped PR | No |
| OpenClaw | `ghcr.io/openclaw/openclaw` | Any time | No |
| Zigbee2MQTT | `ghcr.io/koenkk/zigbee2mqtt` | Any time | No |
| Security | Authelia | Any time | No |
| Infrastructure | Traefik, cert-manager, MetalLB | Default | No |
| Restarts PostgreSQL | postgres-exporter, postgres-operator, Spilo | Default | No; label `restarts-postgres`, title suffix `[restarts Postgres]` |
| Longhorn | `longhorn` | Default | No; label `merge-alone`, title suffix `[merge alone]` |
| Major versions | All | Needs Dependency Dashboard approval | No |

Merge these PRs one at a time, and merge the flagged ones on their own. A
PostgreSQL restart takes Authelia (SSO) down with it, and while Longhorn
upgrades, its admission webhook rejects every PVC change in the cluster.

## Repository Layout

```
bootstrap/kubernetes/apps/
├── flux-system/        # Flux sync configuration (gotk-sync)
├── namespaces/         # All namespace definitions
├── secrets/            # SOPS-encrypted secrets
├── system/             # cert-manager, metrics-server, Reloader
├── network/
│   ├── metallb/        # L2 load balancer
│   ├── traefik/        # Ingress controller + Authelia middlewares
│   ├── cilium/         # CNI values and the migration runbook
│   ├── core-dns/       # CoreDNS network policies
│   └── external-services/
├── storage/
│   ├── nfs-provisioner/
│   └── longhorn/       # Helm release + recurring backup job
├── security/authelia/
├── monitoring/         # prometheus, grafana, loki-stack, goldilocks
├── tools/              # postgresql, valkey, homeassistant, n8n, mosquitto,
│                       # zigbee2mqtt, homarr, homebox, couchdb, renovate, backup
├── agents/openclaw/
├── media/              # jellyfin, jellyseerr, *arr apps, sabnzbd, transmission_*
└── declarations/       # Entry point of the `apps` Kustomization (media)
```

## Flux Reconciliation Order

```
flux-system
apps-namespaces
├── apps-system                  (cert-manager, metrics-server, Reloader)
├── apps-secrets-config          (SOPS)
└── apps-metallb-install
    └── apps-network             (MetalLB config, Traefik, CoreDNS policies)
        ├── apps-traefik-middlewares
        └── apps-storage         (NFS provisioner, Longhorn)
            └── apps-longhorn-config

apps-monitoring   ← apps-secrets-config + apps-traefik-middlewares
apps-authelia     ← apps-secrets-config + apps-traefik-middlewares
apps-tools        ← apps-secrets-config + apps-traefik-middlewares
apps-agents       ← apps-tools + apps-secrets-config + apps-traefik-middlewares
apps (media)      ← apps-authelia + apps-storage + apps-traefik-middlewares
```

## Runbooks and Docs

| Document | Topic |
|---|---|
| [network/cilium/MIGRATION-FROM-FLANNEL.md](bootstrap/kubernetes/apps/network/cilium/MIGRATION-FROM-FLANNEL.md) | How the cluster moved from flannel to Cilium, node by node |
| [tools/postgresql/PATRONI-DCS-CONFIGMAPS.md](bootstrap/kubernetes/apps/tools/postgresql/PATRONI-DCS-CONFIGMAPS.md) | Why Patroni runs in ConfigMaps mode (the stale-endpoint incident) |
| [tools/postgresql/README.md](bootstrap/kubernetes/apps/tools/postgresql/README.md) | Operating and restoring PostgreSQL |
| [storage/longhorn/README.md](bootstrap/kubernetes/apps/storage/longhorn/README.md) | Managing Longhorn, backups and restores |
| [security/authelia/README.md](bootstrap/kubernetes/apps/security/authelia/README.md) | Authelia setup |
| [tools/backup/README.md](bootstrap/kubernetes/apps/tools/backup/README.md) | etcd backups |
| [tools/backup/TALOSCONFIG-CERT-RENEWAL.md](bootstrap/kubernetes/apps/tools/backup/TALOSCONFIG-CERT-RENEWAL.md) | Renewing the talosconfig certificate |

## Useful Commands

```bash
# Flux
flux get kustomizations
flux reconcile kustomization apps --with-source
flux reconcile helmrelease n8n -n tools --force

# Resource usage
kubectl top nodes
kubectl top pods -A --sort-by=memory

# Cilium health (per node)
kubectl -n kube-system exec ds/cilium -c cilium-agent -- cilium-dbg status

# PostgreSQL / Patroni
kubectl exec -n tools postgres-cluster-0 -c postgres -- patronictl list
kubectl get endpointslices -n tools -l kubernetes.io/service-name=postgres-cluster

# Longhorn volumes and backups
kubectl get volumes.longhorn.io -n longhorn-system
kubectl get backups.longhorn.io -n longhorn-system

# Manual runs of the CronJobs
kubectl create job --from=cronjob/renovate renovate-manual -n tools
kubectl create job --from=cronjob/etcd-backup etcd-backup-manual -n backup-system
kubectl create job --from=cronjob/logical-backup-postgres-cluster pg-backup-manual -n tools

# Certificates
kubectl get certificates -A
```

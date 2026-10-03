# Migrating synergia-k8s from flannel to Cilium

Rolling, node-by-node migration. Nothing here is wired into Flux: the manifests in
`app/` are applied by hand, and a node only moves when it is explicitly labelled.

## Why

Two reasons, both already costing us:

1. **The flannel FDB bug.** A node reboot can leave its peers without the VXLAN FDB
   entry, black-holing pod traffic. The `bridge fdb add` fix does not persist.
2. **Longhorn needs node reboots.** Adding the `iscsi-tools` and `util-linux-tools`
   system extensions means rebooting every node, which is precisely what triggers
   the bug above. Migrating the CNI first removes the trigger before we multiply it.

Longhorn itself is blocked behind this: OpenClaw cannot upgrade past 2026.6.34
because its startup migration needs `renameat2(RENAME_NOREPLACE)`, which NFS does
not implement.

## Order, and why synergia-05 goes first

| Node | Arch | Role | Disk |
|---|---|---|---|
| synergia-01/02/03 | arm64 | control-plane | 1× 64 GB USB (system) |
| synergia-05 | amd64 | worker | NVMe 62 GB (system) + **eMMC 62 GB free** |

`synergia-05` is the only worker, the only amd64, and the node where Longhorn will
live. If a step goes wrong there, quorum and API availability are untouched. The
three control-plane nodes only move once the path is proven.

**etcd quorum does not depend on the CNI.** Every control-plane component runs on
`hostNetwork` against the LAN addresses (`10.42.20.x`), as does etcd under Talos.
The CNI only carries pod-to-pod traffic. There is no "wait for two nodes on Cilium
to regain quorum" scenario — quorum is never at stake here.

## Pre-flight facts

Measured on this cluster, not assumed:

- Talos `v1.11.6`, Kubernetes `v1.34.1`, kernel `6.12.62-talos`
- CNI: flannel, 4/4 pods healthy. kube-proxy: 4/4, **stays in place**
- Pod CIDR `10.244.0.0/16`, services `10.96.0.0/12`, LAN `10.42.20.0/24`
  → `10.245.0.0/16` is free and is what Cilium will use
- KubePrism enabled at `127.0.0.1:7445`
- Installed extensions: `synergia-05` has `i915` + `intel-ucode`; the RPi4 nodes
  have none
- Pod Security is `baseline` on ordinary namespaces — Longhorn will later need a
  `privileged` namespace

## Phase 1 — Talos extensions on synergia-05

Schematics generated against the v1.11.6 factory, preserving what each node
already has:

| Node | Extensions | Schematic ID |
|---|---|---|
| synergia-05 | i915, intel-ucode, iscsi-tools, util-linux-tools | `249d9135de54962744e917cfe654117000cba369f9152fbab9d055a00aa3664f` |
| synergia-01/02/03 | iscsi-tools, util-linux-tools | `613e1592b2da41ae5e265e8789429f22e121aab91cb4deb6bc3c0b6262961245` |

These are needed for Longhorn, not for Cilium. They are done here because the node
has to reboot anyway, and one reboot is better than two.

Add the kubelet mount for Longhorn's data path — Talos has a read-only root, so
without this Longhorn cannot write:

```yaml
machine:
  kubelet:
    extraMounts:
      - destination: /var/lib/longhorn
        type: bind
        source: /var/lib/longhorn
        options: [bind, rshared, rw]
```

Then upgrade the node:

```bash
export TALOSCONFIG=~/Dev/HomeLab/talos-config/talosconfig
talosctl -e 10.42.20.12 -n 10.42.20.12 upgrade \
  --image factory.talos.dev/installer/249d9135de54962744e917cfe654117000cba369f9152fbab9d055a00aa3664f:v1.11.6
```

Confirm afterwards:

```bash
talosctl -e 10.42.20.12 -n 10.42.20.12 get extensions
```

Expect `iscsi-tools` and `util-linux-tools` alongside the two existing entries.

**This reboot happens while the cluster is still fully on flannel, so watch for
the FDB bug on the other three nodes.** Verify pod-to-pod traffic across nodes
before continuing.

### Phase 1 execution notes (synergia-05, 2026-09-28)

Done and verified. What the run taught us, for the control-plane nodes later:

- **Use a `talosctl` that matches the node.** The workstation had v1.13.5
  against v1.11.6 nodes. The run used the v1.11.6 client from the official
  release, sha256-checked. Worth doing again for Phase 5.
- **`apply-config --dry-run` first.** Its diff is computed against the live
  node, so it catches drift between the repo file and reality, not just your own
  edit. Here it showed exactly the two intended changes.
- **The reboot is not the long part.** kexec brought the node back in ~20 s. The
  service outage was ~7 min, and almost all of it was the RPi4 nodes pulling
  arm64 images they had never needed. Expect the same for each drain.
- **Wait for rescheduled pods before rebooting.** While the drained node is still
  up, an uncordon undoes everything. After the reboot, it does not.
- **The FDB bug did not trigger.** VTEP MAC unchanged, all three peers kept
  `da:af:d0:1a:c7:f0 dst 10.42.20.12 self permanent`. Check it the same way:
  `kubectl -n kube-system exec <flannel-pod> -c kube-flannel -- bridge fdb show dev flannel.1`
  — the pods are labelled `k8s-app=flannel`, not `app=flannel`.
- **The best functional check is OpenClaw → Home Assistant.** HA lands on a
  different node after the drain, so a working MCP probe proves inbound
  cross-node traffic, which is the direction the FDB bug breaks.
- **Drained pods do not come back.** Traefik, Authelia, Home Assistant and the
  Flux controllers stayed on the RPi4 nodes after the uncordon. Harmless, but the
  load distribution is now different from before.

## Phase 2 — Install Cilium in migration mode

Installing the chart changes nothing on its own: no node carries the migration
label yet, so flannel keeps serving every pod.

```bash
helm repo add cilium https://helm.cilium.io/
helm repo update

helm install cilium cilium/cilium --version 1.20.2 \
  --namespace kube-system \
  --values bootstrap/kubernetes/apps/network/cilium/app/helm-values-migration.yaml

kubectl apply -f bootstrap/kubernetes/apps/network/cilium/app/cilium-node-config.yaml
```

Cilium agents should come up on all four nodes and stay idle:

```bash
kubectl -n kube-system get pods -l k8s-app=cilium -o wide
```

## MetalLB during the migration

MetalLB is largely unaffected, for two structural reasons:

- Its speakers run on `hostNetwork`, so they announce ARP on the node's real NIC
  and never traverse the CNI.
- The pool `10.42.20.40-10.42.20.90` lives on the LAN, nowhere near either pod
  CIDR (`10.244.0.0/16` flannel, `10.245.0.0/16` Cilium). No overlap to resolve.

Two things still need attention.

**Cilium's own LoadBalancer IPAM is on by default.** `enableLBIPAM=true` ships in
the chart. It does nothing without a `CiliumLoadBalancerIPPool`, but it is one
`kubectl apply` away from putting two controllers in charge of the same service
IPs. Both it and `l2announcements` are pinned off in `helm-values-migration.yaml`
— keep them off for as long as MetalLB owns these addresses.

**Draining a node fails its VIPs over.** Every `kubectl drain` in Phase 3 and
Phase 5 moves the LoadBalancer IPs that node was announcing to another speaker.
The gap is an ARP reconvergence, seconds rather than minutes, but it is visible:
14 services depend on it, including Traefik at `10.42.20.40`, which fronts every
ingress in the cluster.

Do the drains outside hours when anyone is watching Jellyfin, and confirm
recovery before moving to the next node:

```bash
kubectl get svc -A --field-selector spec.type=LoadBalancer
ping -c3 10.42.20.40
```

If a VIP does not come back, the usual cause is the speaker on the new node not
having re-announced. `kubectl -n metallb-system rollout restart ds/metallb-speaker`
forces it.

## What migrating synergia-05 does to the rest of the cluster

Starting with the worker protects quorum and the API. It does not leave the other
nodes untouched. Measured on this cluster:

**Phase 2 already touches every node.** The Cilium DaemonSet puts an agent on all
four nodes, and each one brings up its own `cilium_vxlan` device and routes toward
the new pod range. That is how flannel pods reach Cilium pods, so it is required,
not a side effect. No node's CNI config changes until it is labelled.

**The two overlays cannot collide on the wire.** Flannel on Talos uses UDP 4789;
Cilium is pinned to 8472. No host firewall is configured (no `NetworkRuleConfig`
in the machine config), so nothing blocks 8472 between nodes.

**The drain pushes load onto the control-plane nodes.** None of the nodes are
tainted, so everything movable reschedules onto the RPi4s. All of it has an arm64
image (checked against each registry's manifest index), and it fits:

| | CPU requests | Memory requests |
|---|---|---|
| synergia-01 | 33% | 41% |
| synergia-02 | 15% | 28% |
| synergia-03 | 35% | 44% |

What moves: Traefik, Authelia, Home Assistant, the Flux controllers, cert-manager,
Bazarr, Maintainerr, Homebox, Goldilocks, Reloader.

**Three workloads go down for the whole window** because they are pinned with
`nodeSelector: type: wk-amd` and have nowhere else to run: OpenClaw, Jellyfin
(needs the Radxa's iGPU) and Grafana. They return when the node does.

**Everything web-facing blips.** Traefik and Authelia both restart on a different
node, and the MetalLB VIP for Traefik (`10.42.20.40`) fails over. Every ingress
and every SSO-protected app is briefly unreachable, then recovers on its own.

**The reboot is when the FDB bug can bite.** At this point the other three nodes
are still entirely on flannel, which is exactly the condition that bug needs.
Check pod-to-pod traffic across nodes as soon as synergia-05 is back.

## Phase 3 — Migrate synergia-05

```bash
kubectl cordon synergia-05
kubectl drain synergia-05 --ignore-daemonsets --delete-emptydir-data

kubectl label node synergia-05 io.cilium.migration/cilium-default=true
kubectl -n kube-system delete pod -l k8s-app=cilium --field-selector spec.nodeName=synergia-05

talosctl -e 10.42.20.12 -n 10.42.20.12 reboot
```

Once it is back:

```bash
kubectl uncordon synergia-05
```

## Phase 4 — Validate before touching anything else

```bash
# The node must hold a CIDR from the new range
kubectl get ciliumnode synergia-05 -o jsonpath='{.spec.ipam.podCIDRs}'   # 10.245.x.0/24

# Pods rescheduled onto it must carry 10.245.x addresses
kubectl get pods -A -o wide --field-selector spec.nodeName=synergia-05 | head

# Cross-CNI traffic: a pod on synergia-05 must reach one on a flannel node
kubectl -n kube-system exec ds/cilium -- cilium-health status
```

The real check is the workload: OpenClaw runs on this node and talks to the Home
Assistant MCP server in `tools`. If that still answers, pod-to-pod across the two
overlays is working.

Leave the cluster in this hybrid state for a few days. It is a supported
configuration, not a window to rush through.

### Phases 2–4 execution notes (synergia-05, 2026-09-28)

synergia-05 has served pod networking from Cilium since 2026-09-28. Notes for
the control-plane nodes:

- **Compare values with the guide verbatim before installing.** A summary of
  the guide missed that `cni.configMap` is not part of it; the values had it
  pointed at `cilium-config`, which would have broken the CNI config. Caught
  and fixed before `helm install` (e129781).
- **Phase 2 is genuinely inert.** After install every node still had only
  `10-flannel.conflist` and all 61 pods stayed on `10.244`, while
  `cilium-dbg status` already showed 4/4 nodes reachable over the new overlay.
  The chart also deploys a `cilium-envoy` DaemonSet (Envoy runs external).
- **The switch happens at the agent restart, not the reboot.** As soon as the
  labelled node's agent restarted it wrote `05-cilium.conflist` and renamed
  flannel's to `10-flannel.conflist.cilium_bak`. The reboot did not revert it:
  flannel's init did not bring its conflist back.
- **Much cheaper than Phase 1:** 2 min 24 s, since only the three pinned
  workloads were still on the node and no images had to be pulled.
- **Validation that proved the hybrid, in both directions:**
  - Pods on the migrated node carry `10.245.x`; hostNetwork pods keep the node IP.
  - Cilium → flannel: OpenClaw's MCP probe to Home Assistant on another node,
    which also exercises cluster DNS (CoreDNS stays on flannel).
  - flannel → Cilium: Traefik on a flannel node reaching backends on the
    migrated node. **A 302 through Traefik proves nothing on its own** — it may
    be Authelia answering. Check the `Location` header, or hit an unauthenticated
    health path (`/health` on Jellyfin, `/api/health` on Grafana).
  - Prometheus, on a flannel node, still scraping the node-exporter that moved
    to `10.245`.
- **The FDB bug did not trigger on this reboot either**, but the three peers are
  still on flannel, so check it again on every control-plane reboot.

## Phase 5 — Control-plane nodes, one at a time

Same cycle per node, one at a time, waiting for full recovery in between. Start
with a node that is not the etcd leader (`talosctl etcd status`). The Talos
upgrade doubles as the migration reboot, so each node reboots once:

```bash
# Point install.image in controlplane-N.yaml (synergia-k8s-talos) at the
# schematic, then check the diff against the live node and apply (no reboot)
talosctl -e <ip> -n <ip> apply-config --dry-run --file controlplane-N.yaml
talosctl -e <ip> -n <ip> apply-config --file controlplane-N.yaml

kubectl cordon <node> && kubectl drain <node> --ignore-daemonsets --delete-emptydir-data
# wait until every evicted pod is Running and Ready elsewhere

kubectl label node <node> io.cilium.migration/cilium-default=true
kubectl -n kube-system delete pod -l k8s-app=cilium --field-selector spec.nodeName=<node>
# wait for the new agent: /etc/cni/net.d must hold 05-cilium.conflist

talosctl -e <ip> -n <ip> upgrade \
  --image factory.talos.dev/installer/613e1592b2da41ae5e265e8789429f22e121aab91cb4deb6bc3c0b6262961245:v1.11.6
kubectl uncordon <node>
```

Check `talosctl -n <ip> etcd members` and `kubectl get nodes` before moving on.
Quorum survives one node down; it does not survive two.

The FDB risk shrinks with each node migrated and disappears with the last one.

### Phase 5 execution notes (synergia-02, 2026-10-03)

synergia-02 has been on Cilium since 2026-10-03, with the Longhorn extensions.
About 12 minutes end to end, with no service outage:

- **Check the schematic before upgrading an RPi4.** These nodes were flashed
  from the generic `metal-arm64` image and upgraded with the generic installer
  (see the talos-config README), so the schematic must be extensions-only, with
  no `rpi_generic` overlay. Re-posting the YAML to `factory.talos.dev/schematics`
  returned the same ID, `613e1592…`, which confirms it. The factory manifest
  lists arm64.
- **Drain: 17 s, plus 3 min until everything was Ready elsewhere.** No PDB
  blocked it, because nothing PDB-protected ran on the node. Traefik landed on
  synergia-05 and has run on Cilium since.
- **No MetalLB failover.** After the 2026-10-02 power cut, which took down the
  three RPi4s and the switch for 46 min, every VIP was announced from
  synergia-05. Check `kubectl get servicel2statuses -A` before each drain.
- **Upgrade plus reboot: 4.5 min** from cordoned to Ready. `get extensions`
  then lists `iscsi-tools`, `util-linux-tools` and the schematic.
- **Unlike synergia-05, flannel's init rewrote `10-flannel.conflist` on boot.**
  The Cilium agent renamed it back to `.cilium_bak` within seconds
  (`cni-exclusive`), and `05-cilium.conflist` sorts first anyway. Expect this
  on every reboot until Phase 6 removes flannel.
- **Peer health probes lag.** synergia-01 and synergia-03 showed 3/4 reachable
  for one probe interval (~2 min) after the node came back, then 4/4.
- **Validation:**
  - A busybox pod pinned to the node got `10.245.1.x`. From there, CoreDNS,
    Prometheus on flannel and Jellyfin on Cilium all answered.
  - Prometheus on flannel scraped the node's node-exporter at `10.245.1.23`.
  - The FDB on all three peers kept synergia-02's VTEP MAC.

## Phase 6 — Finish (only once all four nodes are on Cilium)

1. Set `cluster.network.cni.name: none` in the Talos machine config so flannel is
   not reinstalled.
2. Remove the flannel DaemonSet and its RBAC.
3. Move the Helm values out of migration mode: drop `policyEnforcementMode: never`
   and `operator.unmanagedPodWatcher.restart: false`, and convert the install into
   a Flux `HelmRelease` under `app/` wired into `kustomization.yaml`.
4. Only then consider `kubeProxyReplacement`, as its own change.

## Rollback

Per node, before Phase 6:

```bash
kubectl label node <node> io.cilium.migration/cilium-default-
kubectl cordon <node> && kubectl drain <node> --ignore-daemonsets --delete-emptydir-data
talosctl -e <ip> -n <ip> reboot
kubectl uncordon <node>
```

The node returns to flannel. This works because both overlays remain installed for
the whole migration — which is why Phase 6 is the point of no easy return, and why
it waits until everything else is proven.

To abandon the migration entirely: remove the label from every node, then
`helm uninstall cilium -n kube-system`.

## Out of scope here

- **kube-proxy replacement.** Stacking it on the CNI swap makes any breakage
  ambiguous.
- **Longhorn itself.** The extensions and the kubelet mount are prepared in
  Phase 1; deploying Longhorn onto the free eMMC is separate work.

# Longhorn

Block storage for workloads that need full POSIX filesystem semantics, which the
cluster's NFS (`nfs-client`) does not provide.

## Why it exists

NFS does not implement `renameat2(RENAME_NOREPLACE)`. OpenClaw 2026.9.x needs it
for a startup state migration, so it cannot run on NFS-backed state. Measured on
2026-09-28 from the same pod, same call:

| Volume | Rename to new target | Rename onto existing target |
|---|---|---|
| Longhorn (ext4) | OK | refused with `EEXIST` (correct) |
| NFS | `EINVAL` | `EINVAL` |

## Current scope: synergia-05 only

Only synergia-05 has the Talos prerequisites (`iscsi-tools`,
`util-linux-tools`) and a dedicated disk: the 62 GB eMMC, provisioned by a Talos
`UserVolumeConfig` and mounted at `/var/mnt/longhorn` (see the
`synergia-k8s-talos` repo). The RPi4 control-plane nodes get the extensions in
Phase 5 of the Cilium migration (`../../network/cilium/MIGRATION-FROM-FLANNEL.md`).

Until then, every Longhorn component is pinned to synergia-05, which means:

- **A pod can only mount a Longhorn volume if it runs on synergia-05.** Pin it
  with `nodeSelector: type: wk-amd`, or it will fail to attach.
- **One replica, no redundancy.** Backups are the only other copy.
- **Name the class explicitly:** `storageClassName: longhorn`. It is
  deliberately not the default StorageClass (there is none in this cluster).

## Caveat: the webhook and synergia-05 downtime

Longhorn registers `validator.longhorn.io` with `failurePolicy: Fail`, and it
intercepts **every PVC `UPDATE` in the cluster**, not only Longhorn ones: no
namespace or object selector, core group `persistentvolumeclaims`.

While synergia-05 is down or rebooting, the webhook has no backend, so **any PVC
update anywhere is rejected** — including `nfs-client` PVCs, and including the
PV controller binding a newly created PVC. Pods whose PVCs are already bound
keep running. Everything recovers on its own once the node is back, since the
PV controller and Flux retry.

Patching the webhook does not work: longhorn-manager reconciles it back. The
exposure goes away once longhorn-manager runs on more than one node, since it is
a DaemonSet and the webhook service then has several backends.

## Reclaim policy is Retain

Deleting a PVC does not delete its data: the PV goes to `Released` and the
Longhorn volume stays. That is intentional, since there is only one replica. To
really delete one:

```bash
PV=$(kubectl get pvc <name> -n <ns> -o jsonpath='{.spec.volumeName}')
kubectl delete pvc <name> -n <ns>
kubectl delete pv "$PV"
kubectl delete volumes.longhorn.io -n longhorn-system "$PV"
```

## Backups

The backup target is `nfs://10.42.20.10:/srv/nfs4/k8s/longhorn-backups`
(available). Backups are plain files, so NFS works as a target even though it
cannot host Longhorn's data.

**A backup target alone takes no backups.** Nothing is scheduled yet; add a
`RecurringJob` before any volume holds data that matters.

## Draining synergia-05

`nodeDrainPolicy` is `allow-if-replica-is-stopped`. The default would block any
drain of a node holding a volume's last replica, which here is every drain of
synergia-05. The drain now proceeds once the consumer pod is gone and the
volume has detached.

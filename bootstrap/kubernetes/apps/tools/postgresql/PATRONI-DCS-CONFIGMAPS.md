# Moving Patroni's DCS from Endpoints to ConfigMaps

**Status:** planned 2026-10-03, not executed yet.

**Goal:** the `postgres-cluster` Service should always point at the real
Postgres pod. Kubernetes should maintain that endpoint, not Patroni.

## Why: the 2026-10-02 incident

After the power cut, which took down all three control-plane nodes, the
Postgres pod restarted *in place*. With the control plane down, nothing
evicted it. The pod got a new IP, but the Service kept pointing at the old one:

| When (UTC) | Pod UID | Real pod IP | Endpoint published by Patroni |
|---|---|---|---|
| before the cut | `3dfc2226…` | `10.244.1.213` | `10.244.1.213` |
| 15:45–17:00 | `3dfc2226…` (same pod, restarted in place) | **`10.244.1.243`** | **`10.244.1.213`** |
| from 17:05 | `38afff13…` (pod deleted and recreated) | `10.244.1.251` | `10.244.1.251` |

Authelia, n8n and Homebox timed out against the dead IP and crash-looped, and
SSO was down for about 75 minutes. Deleting the pod fixed it, because a new
pod object starts with a fresh `POD_IP`.

The mechanism: in Endpoints mode the master Service has no selector. Patroni
writes the endpoint itself, using `kubernetes.pod_ip`. Spilo's
`configure_spilo.py` fills that from the `POD_IP` environment variable on every
container start, and after an in-place restart that variable can still hold
the pre-reboot address. Patroni rewrites the endpoint every cycle, so patching
it by hand does not stick.

In ConfigMaps mode the operator gives the master Service a selector
(`spilo-role: master` plus the cluster labels). Kubernetes then builds the
endpoint from the real IP of whichever pod carries that label, and Patroni only
manages the label. A stale IP can no longer be advertised.

The v1 Endpoints API is also deprecated since Kubernetes 1.33, which is
another reason not to depend on it.

## Verified, not assumed

Read from the source of the versions running here: operator `v1.15.1`, Spilo
`4.0-p3` (the custom `mdcravero/spilo-17:4.0-p3-uid1000-arm64` image) and
Patroni `4.0.4`.

- **The Service gains a selector.** `generateService` in
  `pkg/cluster/k8sres.go` adds it to the master Service when
  `kubernetes_use_configmaps` is set.
- **The Service is updated in place.** `compareServices` detects the selector
  difference, and `updateService` (`pkg/cluster/resources.go`) does an
  `Update` rather than a delete. The LoadBalancer IP `10.42.20.70` is also
  pinned by the `metallb.universe.tf/loadBalancerIPs` annotation.
- **The manual env var has to go.** The operator adds
  `KUBERNETES_USE_CONFIGMAPS=true`, but the cluster's own `spec.env` is
  appended after it and overrides it. So `PATRONI_KUBERNETES_USE_ENDPOINTS=true`
  in `cluster-zalando.yaml` must be removed, or Patroni would keep writing
  endpoints for a Service that now has a selector.
- **Spilo defaults to Endpoints.** Without that env var it still uses
  Endpoints (`use_endpoints: True` unless `use_configmaps` is set). Removing
  the variable on its own changes nothing.
- **Patroni adopts the existing data.** With empty ConfigMaps and a data
  directory that has a valid system ID, `ha.py` takes the initialize key with
  the data's sysid and becomes leader. It does not bootstrap a new cluster.
- **The dynamic config is rebuilt unchanged.** Patroni rewrites it from
  `bootstrap.dcs`. On 2026-10-03 that matched the live `patronictl show-config`
  exactly: 33 keys, no differences.
- **The operator restarts itself.** Its Deployment carries a `checksum/config`
  annotation, so a values change rolls it.
- **Changing loop_wait restarts Postgres.** The operator patches `loop_wait`
  and `retry_timeout` through the Patroni API, but they are also part of
  `SPILO_CONFIGURATION`, so the StatefulSet changes and the pod is recreated.

## Also fix: `loop_wait`

The manifest sets `ttl: 30`, `loop_wait: 30`, `retry_timeout: 14`. That breaks
Patroni's rule `loop_wait + 2*retry_timeout <= ttl`, so Patroni lowers
`loop_wait` to **2 s** and logs `Violated the rule ... Adjusting loop_wait from
30 to 2` on every start. In Kubernetes mode, every leader cycle writes to the
API, so that is an etcd write every 2 seconds on the RPi4 USB disks. Use
Patroni's defaults instead: `ttl: 30`, `loop_wait: 10`, `retry_timeout: 10`.

## Plan

Expect two short database outages, about a minute each, one per step. During
them Authelia is down, and with it SSO. Don't run this while a Cilium node
migration is in progress.

### Step 0: before

1. Take a fresh logical backup, and check that the job completes and its log
   shows the upload to S3:
   ```bash
   kubectl create job -n tools --from=cronjob/logical-backup-postgres-cluster logical-backup-pre-dcs
   kubectl wait -n tools --for=condition=complete job/logical-backup-pre-dcs --timeout=15m
   ```
2. Save the current state outside the repo (it holds no secrets) to compare
   against later:
   ```bash
   kubectl exec -n tools postgres-cluster-0 -c postgres -- patronictl list
   kubectl exec -n tools postgres-cluster-0 -c postgres -- patronictl show-config > patroni-config-before.yaml
   kubectl get svc,endpoints -n tools postgres-cluster -o yaml > postgres-svc-before.yaml
   ```
3. Confirm `PostgresqlMasterEndpointStale` is inactive.

### Step 1: cluster manifest (one restart, still in Endpoints mode)

In `cluster/cluster-zalando.yaml`:
- Remove the `PATRONI_KUBERNETES_USE_ENDPOINTS` entry from `env`. This changes
  no behaviour, because Spilo defaults to Endpoints.
- In `patroni:`, set `loop_wait: 10` and `retry_timeout: 10`, and keep
  `ttl: 30`.

Commit and push. The operator recreates the pod, which comes back with a fresh
`POD_IP`. Verify:
- `patronictl list` shows `Leader running`.
- `patronictl show-config` shows `loop_wait: 10`, and the "Violated the rule"
  warning is gone from the pod log.
- The endpoint IP equals the pod IP.

Let it run for a day before Step 2.

#### Step 1 execution notes (2026-10-06)

Done (`c50ac19f`). Before it, logical backup `1791306438.sql.gz` was taken
and uploaded.
- **Database down for 5.5 minutes, not ~1.** Postgres was unavailable from
  17:11:05 to 17:16:30 UTC:
  - The pod was rescheduled onto synergia-01, freshly uncordoned and the
    emptiest node. That node had never run Postgres, so it pulled the 585 MB
    Spilo image: 2 min 16 s.
  - Spilo's `launch.sh` decompresses `/a.tar.xz` with xz on **every**
    container start, about 2.5 minutes on an RPi4 with no log output. Do not
    mistake that silence for a hang.
  - Patroni then took about 20 seconds to promote itself (timeline 59 → 60).
- **Alerts during the window.** `PostgresqlNotResponding` fired, because the
  exporter started about 3 minutes before Postgres. It resolved at 17:19:48.
  `PostgresqlMasterEndpointStale` stayed quiet: the endpoint was empty for less
  than its 10-minute `for`.
- **Result.** `show-config` changed exactly `loop_wait` 30 → 10 and
  `retry_timeout` 14 → 10. The "Violated the rule" warning is gone, and the
  endpoint equals the pod IP (`10.245.2.224`).
- **Apps.**
  - Home Assistant's recorder reconnected by itself (3 connections), and so
    did n8n and Homebox.
  - Authelia logged no errors and reconnects on the next login; its pool
    holds no idle connections.
- **Postgres now runs on synergia-01, which is on Cilium**, not on
  synergia-03.

**For Step 2:** budget about 3 minutes of downtime, since the image is now on
synergia-01. That assumes the pod is recreated there. A pod move to another
node adds the image pull.

### Step 2: operator in ConfigMaps mode (one restart)

In `operator/configmap-zalando.yaml`, under `configGeneral`, set
`kubernetes_use_configmaps: true`. Commit and push, then watch the operator log
and the pod:
```bash
kubectl logs -n tools deploy/zalando-operator-postgres-operator -f
kubectl get pod -n tools postgres-cluster-0 -w
```
What happens:
1. The operator restarts and syncs the cluster.
2. The Service gets the selector. The running pod already has
   `spilo-role=master`, so the endpoint stays correct at that moment.
3. The StatefulSet gains `KUBERNETES_USE_CONFIGMAPS=true`, and the pod is
   recreated.
4. Patroni finds empty ConfigMaps and an existing data directory. It adopts
   the data, becomes leader and labels the pod master. The timeline goes up
   by one, which is normal.

#### Step 2 execution notes (2026-10-08)

Done (`a4d7f84b`). Before it, logical backup `1791501840.sql.gz` was taken.
- **Order of events.** The operator updated the Service in place first:
  selector added, LoadBalancer IP `10.42.20.70` kept. Then it recreated the
  pod. The pod stayed on synergia-01, where the image was already cached.
- **Database down for about 3 min 20 s** (23:27:25 to 23:30:45 UTC), almost
  all of it Spilo's xz decompression. Patroni found empty ConfigMaps, adopted
  the existing data and promoted itself (timeline 60 → 61).
- **Verification passed:**
  - `KUBERNETES_USE_CONFIGMAPS=true`, and the ConfigMaps
    `postgres-cluster-config` and `postgres-cluster-leader` exist.
  - `show-config` is identical to the pre-change state.
  - The EndpointSlice is now managed by `endpointslice-controller.k8s.io` and
    points at the pod IP.
- **The key test happened as part of the step.** The pod was recreated with a
  new IP (`10.245.2.224` → `10.245.2.212`), and the endpoint followed it with
  no Patroni involvement, so no extra pod deletion was needed.
- **Apps.** Authelia, Home Assistant's recorder and n8n reconnected by
  themselves.
- **Alerts.** `PostgresqlNotResponding` fired during startup, as in step 1.
  `PostgresqlMasterEndpointStale` went `pending` while the endpoint was empty,
  and its 10-minute `for` kept it from firing. Both cleared by 23:33:41.
  `kube_endpointslice_endpoints` reads the controller-managed slice fine, so
  that alert still works in this mode.

Step 4 (cleanup) is due from about 2026-10-15.

### Step 3: verify

- The Service selector includes `spilo-role: master`, and `EXTERNAL-IP` is
  still `10.42.20.70`:
  ```bash
  kubectl get svc -n tools postgres-cluster -o jsonpath='{.spec.selector}'
  ```
- `kubectl get configmap -n tools | grep postgres-cluster` lists Patroni's
  ConfigMaps (`-leader`, `-config`).
- `patronictl list` shows `Leader running`.
- `patronictl show-config` matches `patroni-config-before.yaml`, apart from the
  intended `loop_wait`/`retry_timeout`.
- Authelia login, n8n, Homebox and Home Assistant's recorder all work.
- `PostgresqlMasterEndpointStale` and `PodCrashLooping` stay quiet.
- **The test that matters:** `kubectl delete pod -n tools postgres-cluster-0`.
  The endpoint should follow the new pod IP with nothing writing it but
  Kubernetes.

### Step 4: clean up, a week later

The Endpoints-mode leftovers can go once the rollback window has passed:
- `endpoints/postgres-cluster-config`, which holds Patroni's old config,
  history and initialize key.
- The Patroni annotations on `endpoints/postgres-cluster`. The object itself
  is now managed by Kubernetes, so do not delete it.

Keep both until then, because they are what a rollback falls back on.

## Rollback

- **Undo Step 2:** revert the commit (`kubernetes_use_configmaps` back to
  false).
  1. The operator removes the selector in place and recreates the pod in
     Endpoints mode.
  2. Patroni finds the old DCS keys on the Endpoints: same sysid, and a stale
     leader key that expires after `ttl`.
  3. It takes over and writes the endpoint from a fresh `POD_IP`.

  The data directory is never touched.
- **Undo Step 1:** revert its commit.
- **Last resort:** restore the Step 0 logical backup (see `README.md`).

## Relation to the Cilium migration

`postgres-cluster-0` runs on synergia-03, the last node due for Cilium. Doing
this migration first removes the stale-IP risk from that node's reboot. Until
then, follow the Phase 5 note in
`network/cilium/MIGRATION-FROM-FLANNEL.md`: delete the pod explicitly before
draining or rebooting its node.

# Cluster status

_Last updated: 2026-10-23_

## Summary

Two-node Talos cluster `super-hellthalos`: one control plane that also runs workloads (`talos-i`) and one worker (`talos-ii`, an old laptop). Both nodes report `Ready`. Everything in Kubernetes is deployed by **Flux v2.9.6** from the `main` branch of this repo (bootstrapped 2026-10-22); `main` is protected, so every change goes through a PR. Each node has a dedicated 120 GiB data partition, with local-path-provisioner on top of it as the default StorageClass. Installed infrastructure: local-path-provisioner, Sealed Secrets, cert-manager, the CloudNativePG operator, and the CNPG Barman Cloud plugin. The Postgres `Cluster` `pg-main` is healthy with two instances, one per node (one primary and one streaming replica). It archives WAL continuously and takes a daily base backup to Cloudflare R2, and it holds the app database `wg_app`. Kubernetes Secrets are committed only in sealed form. The Talos config (patches, per-node patches, Image Factory schematics) is in this repo under `talos/`; its secrets are not committed but are backed up (see "Where things live"). Control-plane metrics endpoints are exposed for Prometheus, and kube-prometheus-stack (Prometheus, Grafana, Alertmanager, exporters) is prepared as a Flux `HelmRelease` on branch `feat/prometheus`, not yet merged or deployed (2026-10-23).

The cluster was rebuilt on 2026-10-09: EPHEMERAL was wiped on both nodes to resize it, and etcd was re-bootstrapped (see "Disk layout" and "Lessons learned").

## Nodes

| | `talos-i` | `talos-ii` |
|---|---|---|
| Role | control plane (workloads allowed, no taint) | worker (ROLES shows `<none>`, see below) |
| IP | `192.168.0.192/24` | `192.168.0.220/24` |
| Gateway/DNS | `192.168.0.1` | `192.168.0.1` |
| DHCP reservation in router | yes | yes |
| Hardware | Intel i5 (6 logical CPUs), 16 GB RAM | Acer Aspire E5-571G laptop, 4 logical CPUs, 8 GB RAM |
| Install disk | `/dev/nvme0n1`: WDC PC SN730 256 GB, serial `19422B800421` | `/dev/sda`: Samsung SSD 850 250 GB (SATA) |
| Previous OS | Windows (wiped by the Talos install) | wiped by the Talos install |

The worker has no `node-role.kubernetes.io/*` label because the NodeRestriction admission plugin stops a kubelet from giving itself role labels. It can be added by an admin with `kubectl label node talos-ii node-role.kubernetes.io/worker=`. This is cosmetic and has no effect on scheduling.

## Versions

| Component | Version |
|---|---|
| Talos | v1.14.2 (both nodes) |
| Kubernetes | v1.37.1 (both nodes) |
| containerd | 2.3.6 |
| kubectl (WSL) | v1.37.1 |
| Flux | v2.9.6 (source-controller v1.9.6, kustomize-controller v1.9.6, helm-controller v1.6.5, notification-controller v1.9.4) |
| Image Factory schematic (`talos-i`) | `2d61dd07b20062062ea671b4d01873506103b67c0f7a4c3fb6cf4ee85585dcb8`, source `talos/schematics/default.yaml` |
| Image Factory schematic (`talos-ii`) | `4b3cd373a192c8469e859b7a0cfbed3ecc3577c4a2d346a37b0aeff9cd17cdb0`, source `talos/schematics/talos-ii.yaml` |

Installer image: `factory.talos.dev/metal-installer/<schematic>:v1.14.2`. Both schematic files were checked against the running nodes (`talosctl get extensions`, where the `schematic` entry shows the running ID) and by POSTing them to the factory, which returns the same IDs.

| Node | Extensions |
|---|---|
| `talos-i` | `intel-ucode` (20260812) |
| `talos-ii` | `i915` (20260916-v1.14.2), `intel-ucode` (20260812) |

`talos-ii` has `i915` (Intel iGPU driver) on top of the default. It only matters for GPU video transcoding or the laptop's own display. Keeping it or moving `talos-ii` to the default schematic is still undecided (see "Open items").

## Cluster config

- Cluster name `super-hellthalos`, Kubernetes endpoint `https://192.168.0.192:6443`
- Talos defaults are used for everything except the patches below: Flannel CNI, kube-proxy, CoreDNS.
- All Talos files live in `talos/`; run the commands below from that directory.
- Base configs (`controlplane.yaml`, `worker.yaml`) were generated with:
  ```bash
  talosctl gen config super-hellthalos https://192.168.0.192:6443 \
    --with-secrets secrets.yaml \
    --config-patch @patch-all.yaml \
    --config-patch-control-plane @patch-controlplane.yaml \
    --output-dir _out
  ```
- `patch-all.yaml`:
  - `UnattendedInstallConfig` with the Factory installer image
  - `VolumeConfig` `EPHEMERAL` with `provisioning.maxSize: 80GiB`. This merges into the default `VolumeConfig` that `gen config` emits (which only sets `mount.secure: true`), because Talos matches documents by kind + name.
  - `UserVolumeConfig` `local-path-provisioner`: `volumeType: partition`, disk selector `system_disk`, `maxSize: 120GiB`, filesystem `xfs`. The `system_disk` selector matches the install disk on both nodes, even though one is NVMe and the other SATA.
- `patch-controlplane.yaml`:
  - `KubeNodeConfig` deleting the `node-role.kubernetes.io/control-plane` taint and the `node.kubernetes.io/exclude-from-external-load-balancers` label
  - Metrics endpoints for Prometheus (added 2026-10-23, see "Control-plane metrics endpoints")
- Per-node patches in `nodes/` set what differs between machines. They are applied on top of the base config to produce `_out/talos-i.yaml` and `_out/talos-ii.yaml`:
  - `nodes/talos-i.yaml`: `HostnameConfig` (`talos-i`), disk selector `disk.serial == "19422B800421"`
  - `nodes/talos-ii.yaml`: `HostnameConfig` (`talos-ii`), disk selector `disk.transport == "sata"`, installer image override (see "Versions")
- Regenerating and applying after a patch change:
  ```bash
  talosctl gen config ... --output-types controlplane,worker --output-dir _out --force   # same flags as above; leaves talosconfig alone
  talosctl machineconfig patch _out/controlplane.yaml --patch @nodes/talos-i.yaml  -o _out/talos-i.yaml
  talosctl machineconfig patch _out/worker.yaml       --patch @nodes/talos-ii.yaml -o _out/talos-ii.yaml
  talosctl -n <ip> apply-config -f _out/<node>.yaml --dry-run   # review the diff, then run again without --dry-run
  ```
  All three steps are needed. `gen config` alone only refreshes `controlplane.yaml` and `worker.yaml`; the per-node files stay stale, and a dry-run against them reports `No changes.` (happened on 2026-10-23). `worker.yaml` cannot be applied directly: it has no disk selector, so Talos rejects it with `provisioning.diskSelector.match is required`. Never apply `talos-i.yaml` to `talos-ii`: it is a complete control-plane config with talos-i's hostname, and its dry-run shows a large, seemingly unrelated diff.
- In `--patch @file`, `--config-patch @file` the `@` means "read from this file". Without it, the argument itself is parsed as the patch (useful for inline one-off patches). Same convention as `curl --data-binary @file`.

### Control-plane metrics endpoints (2026-10-23)

By default Talos binds controller-manager, scheduler, and kube-proxy metrics to `127.0.0.1`, and etcd serves no plain metrics endpoint at all (its metrics on `:2379` need an etcd client certificate, which would also grant full read/write on the cluster state). Prometheus scrapes from a pod IP, so all of these would show as down. Appended to `patch-controlplane.yaml`:

```yaml
cluster:                       # legacy v1alpha1 document; etcd still lives here in 1.14
  etcd:
    extraArgs:
      listen-metrics-urls: http://0.0.0.0:2381
---
apiVersion: v1alpha1
kind: KubeControllerManagerConfig
extraArgs:
  bind-address: 0.0.0.0
---
apiVersion: v1alpha1
kind: KubeSchedulerConfig
extraArgs:
  bind-address: 0.0.0.0        # not a KubeSchedulerConfiguration field, so it has to be a flag
---
apiVersion: v1alpha1
kind: KubeProxyConfig
config:
  metricsBindAddress: 0.0.0.0:10249   # kube-proxy reads a config file; most flags have no effect
```

| Endpoint | Port | Protocol / auth |
|---|---|---|
| kube-controller-manager | `talos-i:10257` | HTTPS, delegated authn/authz: anonymous requests get 403, Prometheus uses its ServiceAccount token |
| kube-scheduler | `talos-i:10259` | same as controller-manager |
| etcd | `talos-i:2381` | plain HTTP, unauthenticated, only `/metrics` and `/health` |
| kube-proxy | `<node>:10249` (both nodes) | plain HTTP, unauthenticated |

- Everything goes in the control-plane patch, including kube-proxy: it runs on both nodes, but it is a DaemonSet that the control plane deploys from its machine config. `talos-ii`'s machine config is unchanged.
- `0.0.0.0` (in practice dual-stack `:::`) exposes the ports on every interface, including the public SLAAC IPv6 addresses. Accepted because the unauthenticated endpoints are read-only metrics, and the router is expected to block inbound IPv6 (to be confirmed with the ingress work). The alternative was binding to the node IP via per-node patches.
- The static pods carry the labels kube-prometheus-stack selects on by default (`component=kube-controller-manager`, `component=kube-scheduler`, `k8s-app=kube-proxy`).
- Rollout behavior (see "Lessons learned (Talos 1.14)"): controller-manager and scheduler picked up the change immediately; etcd needed a restart (talos-i was rebooted); kube-proxy needed `talosctl -n 192.168.0.192 upgrade-k8s --to 1.37.1`.
- Verified 2026-10-23 from a pod: `:2381/metrics` returns `etcd_server_has_leader 1`, `:10249/metrics` returns 200 on both nodes, `:10257` and `:10259` return 403 without a token.

## Disk layout

Both nodes use the same layout on their install disk:

| Partition | Size | Purpose |
|---|---|---|
| EFI, META, STATE | ~1 GiB total | Boot, Talos metadata, machine config. STATE survives an EPHEMERAL wipe. |
| EPHEMERAL (`/var`) | 80 GiB | Container images, logs, kubelet, etcd (on `talos-i`) |
| `u-local-path-provisioner` | 120 GiB | Data volume for local-path-provisioner PVs. On `talos-ii` it is `/dev/sda5`. |
| unallocated | ~37 GiB (`talos-i`), ~31 GiB (`talos-ii`) | Left free on purpose for a future user volume, or to grow the local-path volume. Talos can only grow a volume into free space that directly follows it, so it can be used for one of these, not both. |

The user volume is mounted at `/var/mnt/local-path-provisioner` on both nodes (confirmed with `talosctl get mountstatus`; the volume is named `u-local-path-provisioner`, but the mount path has no `u-` prefix).

## Kubernetes workloads

Manifests live in this repo as Kustomize directories, and Flux applies them (see "Flux"). Upstream manifests are referenced by URL with a pinned tag, not copied into the repo, and only local patches are committed. The component sections below still mention the original manual `kubectl apply` commands from before Flux; they are history, not the current workflow.

Repo layout (one directory per component, top-level directories ordered by dependency):
```
talos/                   machine config layer, applied with talosctl (not by Flux)
  patch-all.yaml, patch-controlplane.yaml
  nodes/                 per-node patches
  schematics/            Image Factory schematics
  _out/, secrets.yaml    gitignored
clusters/super-hellthalos/
  flux-system/           Flux components and sync, generated by `flux bootstrap` (do not edit)
  infrastructure.yaml    one Flux Kustomization per infrastructure component
  databases.yaml         Flux Kustomization for pg-main
infrastructure/          cluster-wide foundations (storage, operators, CRDs)
  local-path-provisioner/
  sealed-secrets/        sealed-secrets-keys.yaml is gitignored
  cert-manager/
  cnpg-operator/
  cnpg-barman-plugin/
  kube-prometheus-stack/ Namespace, OCIRepository, HelmRelease, values.yaml, SealedSecret
databases/               things that use the foundations
  pg-main/               Cluster, Database, ObjectStore, ScheduledBackup, SealedSecrets
docs/                    STATUS.md (this file), TODO.md
```

### Flux (bootstrapped 2026-10-22)

- Flux v2.9.6, bootstrapped from `ssh://git@github.com/superhellth/k8s-homelab`, branch `main`, path `clusters/super-hellthalos`. All four controllers run in `flux-system`; helm-controller gets its first `HelmRelease` with kube-prometheus-stack.
- `main` is protected: changes go through PRs, and Flux only reconciles what is merged. Force a sync after a merge with `flux reconcile kustomization <name> --with-source`.
- The `flux-system` Kustomization reconciles `clusters/super-hellthalos/`. Flux generates the kustomization for that directory itself, so every YAML file placed there is picked up. Each file holds one Flux `Kustomization` per component, pointing at `./infrastructure/<component>` or `./databases/pg-main`, with `prune: true`, `wait: true`, `interval: 1h`, `retryInterval: 2m`, `timeout: 5m` (`kube-prometheus-stack`: `timeout: 15m`, see its section).
- Ordering uses `dependsOn`: `cnpg-barman-plugin` depends on `cert-manager` and `cnpg-operator`; `pg-main` depends on `local-path-provisioner`, `cnpg-operator`, `cnpg-barman-plugin`, and `sealed-secrets`; `kube-prometheus-stack` depends on `sealed-secrets` and `local-path-provisioner`. With `wait: true`, a Kustomization only counts as Ready once its objects are healthy, so dependents wait for running operators, not just applied manifests.
- **Prune protection:** `prune: true` deletes objects that disappear from Git, and deleting or renaming a Flux Kustomization prunes everything it applied. Deleting a CRD deletes every custom resource of that type, so Kustomizations that ship CRDs patch them with `kustomize.toolkit.fluxcd.io/prune: disabled`. The `pg-main` Kustomization does the same for the `Cluster` and the `Namespace` (pg-main was handed over to Flux on 2026-10-23). The key must be exactly `kustomize.toolkit.fluxcd.io/prune`; `kustomize.toolkit.io/...` is silently ignored.
- kustomize-controller always uses server-side apply, so the CNPG CRDs (too big for client-side apply) need no special handling under Flux.
- Render exactly what Flux will apply, including the Kustomization's patches: `flux build kustomization <name> --path ./infrastructure/<dir> --kustomization-file ./clusters/super-hellthalos/infrastructure.yaml --dry-run`.

### local-path-provisioner (installed 2026-10-10)

- Upstream `rancher/local-path-provisioner` v0.0.37 (`deploy/` directory), namespace `local-path-storage`
- `infrastructure/local-path-provisioner/` with three patch files:
  - `patch-namespace.yaml`: label `pod-security.kubernetes.io/enforce: privileged`. The helper pods use `hostPath`, which Talos's default `baseline` enforcement would reject.
  - `patch-config.yaml`: `config.json` points `DEFAULT_PATH_FOR_NON_LISTED_NODES` at `/var/mnt/local-path-provisioner`. Upstream's `/opt/...` is read-only on Talos.
  - `patch-storageclass.yaml`: makes `local-path` the default StorageClass
- StorageClass `local-path`: `WaitForFirstConsumer`, `reclaimPolicy: Delete` (upstream default, kept on purpose; data protection will come from backups). Sizes are not enforced; all PVs share the 120 GiB partition. No volume expansion or snapshots: it is not a CSI driver.
- Applying it prints a PodSecurity `restricted` warning for the provisioner Deployment. This is expected: the namespace only overrides `enforce`, so `warn` stays at the Talos default `restricted`.
- End-to-end PVC test passed (2026-10-10): the PVC stays `Pending` until a pod is scheduled, a helper pod creates the directory, the PV gets `nodeAffinity` to that node, data survives a pod restart, and deleting the PVC removes the PV and the directory. Pin test pods with `nodeSelector`, not `nodeName`: `nodeName` skips the scheduler, so the `volume.kubernetes.io/selected-node` annotation is never set and the PVC stays `Pending`.

### cnpg-operator (installed and tested 2026-10-10)

- Upstream release manifest `cnpg-1.30.1.yaml` (from the `release-1.30` branch), referenced by URL in `infrastructure/cnpg-operator/kustomization.yaml`. No local patches.
- Applied with `kubectl apply --server-side -k infrastructure/cnpg-operator/`, then `kubectl -n cnpg-system rollout status deploy/cnpg-controller-manager`.
- Installs 11 CRDs in `postgresql.cnpg.io/v1` (`Cluster`, `Backup`, `ScheduledBackup`, `Pooler`, `Database`, …), namespace `cnpg-system`, one `cnpg-controller-manager` replica (requests 100m / 100Mi, limit 200Mi), cluster-wide RBAC, and mutating + validating webhooks.
- The operator pod already meets PSA `restricted`, so `cnpg-system` needs no namespace label.
- The webhooks use `failurePolicy: Fail`: while the operator is not Ready, creating or changing any CNPG object is rejected.
- The operator creates its own CA and webhook certificate (`cnpg-ca-secret`, `cnpg-webhook-cert`). Default images: `ghcr.io/cloudnative-pg/postgresql:18.6-system-trixie` and `pgbouncer:1.25.2`.

### sealed-secrets (installed 2026-10-20)

- Upstream `bitnami/sealed-secrets` v0.40.0 `controller.yaml`, referenced by URL in `infrastructure/sealed-secrets/kustomization.yaml`. No local patches.
- Deployment `sealed-secrets-controller` in `kube-system`. It has no resource requests or limits yet (see `docs/TODO.md`).
- The sealing key pair is the `kubernetes.io/tls` Secret `sealed-secrets-keygb2ln` in `kube-system`. It is exported to `infrastructure/sealed-secrets/sealed-secrets-keys.yaml` (gitignored, backed up). Without this key, no committed SealedSecret can be decrypted after a cluster rebuild.
- Workflow: generate the Secret locally and seal it in one pipeline, so the plaintext never reaches the cluster or Git:
  ```bash
  kubectl create secret generic <name> -n <ns> --type <type> --from-literal=... --dry-run=client -o yaml \
    | kubeseal --format yaml > <dir>/sealedsecret-<name>.yaml
  ```
  The ciphertext is bound to the name and namespace (default `strict` scope), so renaming means resealing. The controller only writes Secrets that it owns (ownerReference to the SealedSecret).
- Cluster rebuild: `kubectl apply` the key backup **before** Flux deploys the controller. Otherwise the controller generates a new key, and the committed SealedSecrets cannot be decrypted.

### cert-manager (installed 2026-10-21)

- Upstream `cert-manager.yaml` v1.21.2, referenced by URL in `infrastructure/cert-manager/kustomization.yaml`. No local patches.
- Deployments `cert-manager`, `cert-manager-cainjector`, and `cert-manager-webhook` in namespace `cert-manager`.
- Installed because the Barman Cloud plugin needs it: the plugin's gRPC connection to the CNPG operator uses mTLS certificates that cert-manager issues.
- No resource requests or limits yet (see `docs/TODO.md`).

### cnpg-barman-plugin (installed 2026-10-21)

- Upstream `plugin-barman-cloud` v0.15.1 `manifest.yaml`, referenced by URL in `infrastructure/cnpg-barman-plugin/kustomization.yaml`. Deployment `barman-cloud` in `cnpg-system`, plus the `ObjectStore` CRD (`barmancloud.cnpg.io/v1`).
- One local JSON patch lowers the log level from `debug` to `error`. It starts with an `op: test` on `args[6]`, so the render fails loudly if a future upstream release moves the argument, instead of silently replacing the wrong one.
- This is CNPG's CNPG-I plugin architecture: the operator talks to the plugin over gRPC, and the plugin injects a sidecar into each instance pod that ships WAL and base backups to object storage. It replaces the in-tree `barmanObjectStore` field, which is deprecated.
- No resource requests or limits yet (see `docs/TODO.md`).

### kube-prometheus-stack (prepared 2026-10-23, not yet deployed)

- Helm chart `kube-prometheus-stack` **82.16.0** (Prometheus Operator v0.89.0), installed by helm-controller from `infrastructure/kube-prometheus-stack/`. Everything runs in namespace `monitoring`. Flux Kustomization `kube-prometheus-stack` in `infrastructure.yaml`, `dependsOn` `sealed-secrets` (Grafana secret) and `local-path-provisioner` (PVCs), `timeout: 15m`.
- How the pieces fit:
  - `namespace.yaml`: `monitoring` with `pod-security.kubernetes.io/enforce: privileged`, because node-exporter needs `hostNetwork`, `hostPID`, and `hostPath`. PSA works per namespace, so every pod in `monitoring` is *allowed* to request privileged settings (each still only gets what its spec asks for). Owned in Git because helm-controller's `createNamespace` cannot set labels. No prune protection: metrics history is treated as rebuildable.
  - `ocirepository.yaml`: source-controller pulls the chart from `oci://ghcr.io/prometheus-community/charts/kube-prometheus-stack`, `ref.tag: 82.16.0` (exact pin, upgrades are PRs), `interval: 12h`. `layerSelector` picks `application/vnd.cncf.helm.chart.content.v1.tar+gzip` with `operation: copy`. This is required here: in this artifact the **provenance layer comes first**, and without a selector Flux takes the first layer.
  - `helmrelease.yaml`: `chartRef` to the OCIRepository. `install.crds: Create`, `upgrade.crds: CreateReplace` (plain Helm never upgrades CRDs from `crds/`), `remediation.retries: 3` (uninstall and retry a failed install, roll back a failed upgrade), `timeout: 15m` (first install pulls ~8 images). Drift detection is off: helm-controller does not revert a manual `kubectl edit` (kustomize-controller would).
  - `values.yaml`: plain Helm values, turned into the ConfigMap `kube-prometheus-stack-values-<hash>` by a `configMapGenerator` and read via `valuesFrom`. `kustomizeconfig.yaml` (`nameReference` for `HelmRelease` `spec/valuesFrom/name`) makes kustomize rewrite the reference to the hashed name, so a values change changes the HelmRelease spec and triggers an upgrade immediately; old ConfigMaps are pruned. The generator needs `namespace: monitoring`: without it the ConfigMap has no namespace, `nameReference` does not rewrite across namespaces, and Flux would apply it to `default`.
  - `sealedsecret-grafana-admin.yaml`: Secret `grafana-admin` (keys `admin-user`, `admin-password`), referenced by `grafana.admin.existingSecret`. Read the password with `kubectl -n monitoring get secret grafana-admin -o jsonpath='{.data.admin-password}' | base64 -d`.
- Values (see `values.yaml`):
  - `kubeEtcd.endpoints: [192.168.0.192]`: on Talos, etcd is a system service, not a pod, so the chart's default Service selector `component=etcd` matches nothing. With `endpoints` set, the chart drops the selector and writes the Endpoints itself. Port `2381`/HTTP is the chart default. Controller-manager and scheduler need no values: for Kubernetes ≥ 1.22 the chart switches to `10257`/`10259` over HTTPS with `insecureSkipVerify`.
  - Prometheus: `retention: 15d`, `retentionSize: 8GB`, 10Gi `local-path` PVC via `storageSpec.volumeClaimTemplate`, `nodeSelector` `kubernetes.io/hostname: talos-i`. local-path does not enforce the 10Gi, so `retentionSize` is the real disk limit (kept ~20% below the PVC size).
  - Grafana: SQLite on a 1Gi `local-path` PVC (`persistence`). The chart default is an `emptyDir`, which loses UI-made dashboards and users on every restart. The default dashboards and the Prometheus data source are provisioned by the chart's sidecar and rebuilt on every start.
  - Resources: Prometheus 200m / 1Gi request, 2Gi limit; Grafana 50m / 192Mi, 512Mi; operator 50m / 64Mi, 256Mi; kube-state-metrics 10m / 64Mi, 256Mi; node-exporter 10m / 32Mi, 128Mi; Alertmanager 10m / 32Mi, 128Mi. No CPU limits on purpose (they throttle via CFS quota even on an idle node); memory limits protect the control plane on `talos-i`. The numbers are estimates; to be checked against real usage. Sidecars (config-reloader, Grafana dashboard sidecar) are left at defaults.
  - Alertmanager stays enabled but has no receiver configured yet.
- Values paths: `prometheus.prometheusSpec.*` and `alertmanager.alertmanagerSpec.*` end up in the `Prometheus` / `Alertmanager` custom resources, which the Prometheus Operator turns into StatefulSets (two layers to check when debugging). Grafana, kube-state-metrics, and node-exporter values go straight into the Deployment/DaemonSet. Subchart values sit under the subchart's exact name (`kube-state-metrics`, `prometheus-node-exporter`); the camelCase `kubeStateMetrics`/`nodeExporter` keys only toggle them.
- Checks after deploying: `flux get sources oci -n monitoring`, `flux get helmreleases -n monitoring`, `kubectl -n monitoring get pods,pvc -o wide`, `kubectl -n monitoring get prometheus -o yaml` (did the values arrive?), and in the Prometheus UI *Status → Targets*: etcd, controller-manager, scheduler, kube-proxy all UP. `kubectl top` does not work yet (no metrics-server); use `container_memory_working_set_bytes` in Prometheus for real memory usage.

### pg-main (healthy since 2026-10-20)

- `Cluster` `pg-main` in namespace `databases`, from `databases/pg-main/`. Applied manually until 2026-10-23, since then reconciled by the Flux Kustomization `pg-main` (`clusters/super-hellthalos/databases.yaml`), with prune disabled for the `Cluster` and the `Namespace`.
- `instances: 2`, `podAntiAffinityType: required` (one instance per node), image `postgresql:18.6-system-trixie`
- Guaranteed QoS: 500m CPU / 1Gi memory (requests = limits), `shared_buffers: 256MB`
- Storage: `local-path`, 50Gi per instance. local-path does not enforce the size; the PVs share the 120 GiB partition with every other PV on the node. Reclaim policy `Delete` (from the StorageClass), and CNPG makes the `Cluster` the owner of its PVCs, so **deleting the `Cluster` deletes the data**.
- Bootstrap: `initdb` with database `app`, owner `app`
- Verified 2026-10-20: `Cluster in healthy state`, primary `pg-main-1` on `talos-i`, replica `pg-main-2` on `talos-ii`. Each PV is pinned to the node of its pod. `pg_stat_replication` on the primary shows `pg-main-2` as `streaming` / `async`, and the replica reports `pg_is_in_recovery() = t`.
- 2026-10-21 14:36 UTC: the primary role moved to `pg-main-2` (`talos-ii`), and `pg-main-1` (`talos-i`) was recreated as a replica. The cause was not recorded. The primary was later back on `pg-main-1`.
- 2026-10-23 ~16:57 UTC: rebooting `talos-i` (to restart etcd) left `pg-main-1`, then primary, in phase `Completed`. Once the API server and the CNPG operator were back (~3 minutes), the operator promoted `pg-main-2` (`talos-ii`) and recreated `pg-main-1` as a replica; the cluster was healthy again within seconds after that. While `talos-i` was down, nothing could be promoted, because failover goes through the API server. Check the current primary with `kubectl get pods -n databases -L cnpg.io/instanceRole`, and connect through `pg-main-rw`, never through a pod name.
- App database `wg_app` (added 2026-10-21):
  - Role `wg_app` (login) under `spec.managed.roles` in the `Cluster`. Its password comes from the `kubernetes.io/basic-auth` Secret `pg-main-wg-app` (keys `username` and `password`, with a random 48-character hex password), which is created from `sealedsecret-wg-app.yaml`.
  - `Database` resource `pg-main-wg-app` creates database `wg_app` owned by `wg_app`.
  - Tables stay in the `public` schema. The database is the isolation boundary, and since PG 15 `public` is owned by `pg_database_owner` (here `wg_app`) with only `USAGE` for `PUBLIC`.
  - Verified 2026-10-21: the SealedSecret is unsealed into a Secret owned by it, `managedRolesStatus` shows `wg_app` as `reconciled`, and the `Database` shows `APPLIED: true`. Retrieve the password with `kubectl get secret pg-main-wg-app -n databases -o jsonpath='{.data.password}' | base64 -d`.
- Backups (set up 2026-10-21/22) through the Barman Cloud plugin:
  - `ObjectStore` `r2-store`: Cloudflare R2 bucket `k8s-pg-backups` (S3 API), credentials from the Secret `r2-backup-creds` (created from `sealedsecret-r2-backup.yaml`). WAL compressed with `zstd` (`maxParallel: 2`), base backups with `gzip`. `retentionPolicy: 30d`.
  - The `Cluster` references it under `spec.plugins` with `isWALArchiver: true`, so every WAL segment is archived continuously. Together with base backups this allows point-in-time recovery, not just restoring the last dump.
  - `ScheduledBackup` `pg-main-daily`: `schedule: "0 0 3 * * *"` (six fields, the first is seconds; 03:00 UTC daily), `method: plugin`, `immediate: true` (one backup right after creation), `backupOwnerReference: self`.
  - Verified 2026-10-23: two `Backup` objects in phase `completed` (2026-10-22 18:50 UTC, the immediate one, and 2026-10-23 03:00 UTC), and the `Cluster` conditions show `ContinuousArchiving=True` and `LastBackupSucceeded=True`.
  - **A restore has not been tested yet** (see "Open items").
- History: the first attempt (2026-10-11) deadlocked and was deleted and recreated on 2026-10-20. See "Lessons learned (Kubernetes / tooling)".

### Decisions

- **CNPG runs in this cluster, not a separate one.** A second cluster would double the control-plane overhead and split the two nodes into two single-node clusters. Isolation happens inside the cluster instead: own namespace, Guaranteed QoS (requests = limits) for Postgres pods, optionally a PriorityClass. Flannel does not enforce NetworkPolicies, so network isolation needs a CNI change (e.g. Cilium).
- **Known limits of this setup:** etcd and Postgres share the NVMe on `talos-i` (watch for `slow fdatasync` warnings from etcd). CNPG failover runs through the API server, so if `talos-i` (the only control plane) dies, a replica on `talos-ii` is **not** promoted automatically.
- **Backups: CNPG Barman Cloud plugin to Cloudflare R2** (decided 2026-10-21). Volume snapshots are not an option, because local-path doesn't support them. The plugin gives continuous WAL archiving plus base backups, so point-in-time recovery, and the data leaves the house. The alternatives were `pg_dump` (only point-in-time snapshots, no WAL) and Velero file-level backups with Kopia (not consistent for a running database without extra hooks).
- **GitOps: Flux** (bootstrapped 2026-10-22), not Argo CD. The reasons were not recorded at the time. Relevant differences for this setup: Flux has no UI or API server of its own (less RAM), and everything is a CRD reconciled by a controller; Argo CD offers a web UI for visualizing sync state.
- **Control-plane metrics on `0.0.0.0`** (2026-10-23), see "Control-plane metrics endpoints".
- **Monitoring: kube-prometheus-stack via Flux `HelmRelease` from an `OCIRepository`** (2026-10-23). OCI instead of a `HelmRepository`, which downloads the repo's whole multi-MB `index.yaml` on every interval; OCI fetches one tag, like a container image.
- **One Prometheus replica, pinned to `talos-i`** (2026-10-23). Prometheus replicas don't replicate: each one scrapes everything independently, so an HA pair doubles memory and disk and needs a dedup layer (Thanos Query) to query cleanly. With a single control plane, losing `talos-i` takes the API down anyway. `talos-i` has the RAM and the SSD, and the `nodeSelector` makes the permanent `WaitForFirstConsumer` binding a deliberate choice.
- **Prometheus TSDB on local-path, not elsewhere** (2026-10-23). Prometheus only writes its own TSDB format; it cannot store data in Postgres. Durable metrics would mean shipping blocks to object storage (Thanos sidecar, Mimir), possibly a later learning project.
- **Grafana: SQLite on a local-path PVC, not `pg-main`** (2026-10-23). Storing Grafana's state in `pg-main` would tie the monitoring UI to the system it monitors: when Postgres is in trouble, Grafana can't log you in. Everything in Grafana's database is either rebuildable or should become dashboards-as-code anyway.

## Where things live

The Talos files moved from `~/talos` into `talos/` in this repo (2026-10-10). The rule for what gets committed: sources without secrets are committed, generated files are not, and secrets are committed only once they are encrypted.

- Committed: `patch-all.yaml`, `patch-controlplane.yaml`, `nodes/*.yaml`, `schematics/*.yaml`
- Committed in encrypted form: SealedSecrets (`databases/pg-main/sealedsecret-wg-app.yaml`, `databases/pg-main/sealedsecret-r2-backup.yaml`, `infrastructure/kube-prometheus-stack/sealedsecret-grafana-admin.yaml`)
- Not committed: `.claude/` is gitignored, so `.claude/CLAUDE.md` (instructions for Claude Code) exists only in the working tree.
- Gitignored and plaintext in the working tree. **The secret files are backed up outside the working tree** (as of 2026-10-21):
  - `talos/secrets.yaml`: cluster CAs and tokens. Losing it means losing admin access. `git clean -fdx` would delete the working-tree copy.
  - `infrastructure/sealed-secrets/sealed-secrets-keys.yaml`: the Sealed Secrets controller's private key. It is needed to decrypt the committed SealedSecrets after a rebuild.
  - `talos/_out/`: generated machine configs (`controlplane.yaml`, `worker.yaml`, `talos-i.yaml`, `talos-ii.yaml`) and `talosconfig`. They all contain secrets and can be regenerated from the sources plus `secrets.yaml`.
- `talosctl` uses `~/.talos/config` (context `super-hellthalos`, endpoint `192.168.0.192`, client cert expires 2027-10-05). `TALOSCONFIG` is not set.
- kubeconfig merged into `~/.kube/config`

## Lessons learned (Talos 1.14)

- 1.14 generates multi-document configs. `.machine.install` is replaced by `UnattendedInstallConfig`, and node labels, taints, and `allowSchedulingOnControlPlanes` are replaced by `KubeNodeConfig`. Old-style fields in patches cause "already set in v1alpha1 config" errors.
- `--config-patch` applies to **both** controlplane and worker configs. A `$patch: delete` on a key missing from the worker config fails with `lookup failed`, so role-specific patches go in `--config-patch-control-plane` / `--config-patch-worker`.
- The generated `UnattendedInstallConfig` sets `wipe: false` explicitly. The install still repartitions the disk.
- Machine-specific settings (hostname, install disk) belong in per-node patches, not in `patch-all.yaml`. A disk selector like `dev_path == "/dev/nvme0n1"` doesn't match a laptop with a SATA disk.
- Worker nodes show ROLES `<none>`. That is normal Kubernetes behavior, not a Talos problem.
- `VolumeConfig` and `UserVolumeConfig` only take effect when a volume is provisioned. Applying a new `maxSize` to an existing EPHEMERAL does nothing. In the meantime, a user volume that does not fit shows `failed` in `talosctl get volumestatus`. To resize EPHEMERAL: apply the config, then run `talosctl reset --system-labels-to-wipe EPHEMERAL --reboot`. On boot, Talos recreates EPHEMERAL with the new size, and the user volume gets the freed space.
- Wiping EPHEMERAL on the only control plane deletes etcd, which means the whole cluster state. It needs `--graceful=false`, because a graceful reset tries to leave etcd, and the last member can't. Afterwards, wait for etcd to sit in "waiting for bootstrap" and run `talosctl bootstrap` once. Kubeconfig and talosconfig keep working because the CA comes from `secrets.yaml`, and Talos re-applies Flannel, CoreDNS, and kube-proxy by itself.
- A kubelet only registers its Node object at startup. After etcd was wiped, `talos-ii` (whose kubelet kept running) did not reappear until `talosctl -n 192.168.0.220 service kubelet restart`. `talosctl health` showed this as `can't find expected node with IPs [...]`.
- A node cordoned by a graceful reset stays cordoned after the reboot (`kubectl uncordon`). The flag is stored on the Node object in etcd, not on the node itself.
- An Image Factory schematic ID is the hash of the schematic YAML. The factory normalises the YAML first (indentation, comments), but **list order counts**: `[intel-ucode, i915]` and `[i915, intel-ucode]` produce different IDs for the same image. There is no inheritance between schematics; each file describes a complete image.
- `POST https://factory.talos.dev/schematics` with the YAML returns its ID (`curl -sX POST --data-binary @file.yaml ...`; `@` means "read from file", and `--data-binary` keeps newlines, which `-d` strips). An empty body returns just `EOF`. `GET /schematics/<id>` returns the stored YAML for any existing ID.
- In `talosctl get extensions`, `schematic` (the running schematic ID) and `modules.dep` (kernel module index rebuilt because an extension, here `i915`, ships kernel modules) are generated entries, not extensions to put in a schematic.
- Changing `installer.image` in the machine config does not change the running OS. It only takes effect on the next install or `talosctl upgrade --image ...`. Upgrading with the wrong schematic ID silently removes extensions.
- Applying a machine config change does not restart every affected component:
  - **Static pods** (kube-apiserver, kube-controller-manager, kube-scheduler): Talos rewrites the manifest and the kubelet recreates the pod immediately.
  - **etcd** is a Talos service, not a pod. Talos updates the etcd spec (`talosctl get etcdspecs`), but the running process keeps its old flags until etcd restarts (check with `talosctl processes`). On a single control plane this means a short API outage.
  - **Bootstrap manifests** (kube-proxy, Flannel, CoreDNS): Talos creates objects that are missing but does not update existing ones. kube-proxy's ConfigMap is named after a hash of its content, so a config change produced a new ConfigMap while the DaemonSet kept mounting the old one. `talosctl upgrade-k8s --to <current version>` syncs the manifests (it walks through every component, but at the same version only the manifest diff matters). It also prunes objects that are no longer in the manifests, using the inventory recorded in the `config.k8s.io/owning-inventory: talos-bootstrap-manifests-inventory` annotation, so the old ConfigMap was deleted automatically.
  - Naming a ConfigMap after its content hash is a deliberate pattern: pods do not restart when a mounted ConfigMap changes, but a new name changes the pod template and triggers a rolling update (Kustomize's `configMapGenerator` does the same).
- `talosctl docs <dir> --config` writes the full machine config reference for the installed `talosctl` version as Markdown. Useful for checking field names offline.
- Talos enables Pod Security Admission cluster-wide with `enforce: baseline`, `warn: restricted`, `audit: restricted`, and exempts `kube-system` (`KubeAdmissionControlConfig` `PodSecurity`). Workloads that need `hostPath` or privileged access need a namespace label like `pod-security.kubernetes.io/enforce: privileged`.

## Lessons learned (Kubernetes / tooling)

- Kustomize patches are matched by apiVersion + kind + name (+ namespace). A typo there (e.g. `storage.k82.io`) fails the render with `no resource matches strategic merge patch`, which is a useful safety net. Kustomize does not look inside string values such as `config.json`, so JSON errors only show up when the program reads the file.
- `kubectl kustomize` re-serialises objects with fields sorted alphabetically, so `kind:` can appear after a long `data:` block. Key order has no meaning in YAML.
- `kubectl get a,b,c -w` fails with `you may only specify a single resource type`. Watch one type at a time, use `kubectl get events -A -w`, or poll with `watch -n1 '...'`.
- CNPG's manifests must be applied with `kubectl apply --server-side`. Its CRDs exceed the 256 KiB annotation limit that client-side apply runs into.
- The CNPG operator pod restarted once on first install (exit code 143 = SIGTERM, about 27 s after start, nothing wrong in the logs). Most likely cause: the startup probe (6 × 5 s) checks the webhook port, and the certificate Secret the operator had just created had not been synced into the pod yet (it is mounted `optional: true`, and the kubelet only syncs Secret volumes periodically). The second start worked. This is harmless on first install.
- `kubectl rollout status deploy/<name>` waits until the Deployment's `.status` shows all replicas updated and Ready, and exits non-zero after `progressDeadlineSeconds` (600 s by default). Use it between applying an operator and applying its custom resources.
- Heredocs: the end marker (`EOF`) must start at column 0, and `<<'EOF'` (quoted) stops the local shell from expanding `$(...)` inside the manifest.
- The first `pg-main` (2026-10-11, same manifest) deadlocked for 8 days:
  1. `pg-main-1` (primary) started on `talos-i`. The join job for `pg-main-2` stayed `Pending`: required anti-affinity ruled out `talos-i`, and `talos-ii` was evidently not schedulable.
  2. Seven minutes later the kubelet on `talos-i` terminated `pg-main-1` (`DisruptionTarget`, `TerminationByKubelet`, most likely a node shutdown or reboot). The pod ended as `Succeeded` and was never restarted, despite `restartPolicy: Always`: a pod in a final phase is never restarted. CNPG creates pods directly (no StatefulSet), and the operator was busy in the "Creating a new replica" phase.
  3. When `talos-ii` became `Ready` again (2026-10-20), the scheduler placed the join pod on `talos-i`: it ignores pods in a final phase for anti-affinity, so the dead primary no longer blocked that node. `WaitForFirstConsumer` then created `pg-main-2`'s PV on `talos-i`, which pinned the replica there for good.
  4. The join pod retried `pg-main-rw:5432` (`connection refused`) while the operator waited for the join. Fix: the cluster held no data, so it was deleted (PVCs and PVs went with it) and re-applied while both nodes were `Ready`.
  - Takeaways: create a cluster with required anti-affinity only while every node is `Ready`. With `WaitForFirstConsumer` and local volumes, the first scheduling decision is permanent. If a CNPG cluster sits in a "Creating …" phase, check the instance pods and job logs instead of waiting.
- Kustomize only renders files listed under `resources:`. A manifest in the directory that is not listed is silently skipped. Check with `kubectl kustomize <dir> | grep '^kind:'`.
- `kubectl apply -k <dir> --dry-run=server` runs validation and admission webhooks without persisting anything. It is a good last check before a real apply. `--dry-run=client` never contacts the server.
- A Secret's `type` is immutable, and the Sealed Secrets controller refuses to overwrite a Secret it does not own. A manually created or broken Secret with the same name must be deleted before the SealedSecret can take over.
- Broken shell line continuations (`\` followed by a space) can silently drop flags like `--dry-run`. The `> file` redirect still truncates the target even if the pipeline fails, so write to a temporary file and `mv` it on success.
- After a reboot of `talos-i` (2026-10-23), operators that started before the API server was reachable (CNPG operator, Flux notification-controller) exited with `dial tcp 10.96.0.1:443: connect: connection refused` and went into `CrashLoopBackOff`. They recovered on their own once the API was up (~3 minutes); Flux Kustomizations briefly showed `Source artifact not found`. No action needed, but don't panic at the first `kubectl get pods`.
- Pods that were running when a node shut down end in a final phase (`Completed`) and are replaced by their ReplicaSet, not restarted. The old pods stay around as clutter until deleted (several in `kube-system`, `cnpg-system`, `flux-system`).
- `kubectl run -i --rm` can lose the first lines of output: kubectl only attaches after the container has started ("If you don't see a command prompt…"). For a short test, run the pod without `-i`, wait for `Succeeded`, and read `kubectl logs`.
- Helm silently ignores unknown values keys, and a `HelmRelease`'s `spec.values` is free-form in the CRD, so `--dry-run=server --validate=strict` accepts typos there. `persistance:` instead of `persistence:` passed every check and would have left Grafana on an `emptyDir`. Compare value paths against the chart's `values.yaml` (kube-prometheus-stack ships no `values.schema.json`), and after deploying check that the setting reached the rendered object.
- Kustomize `nameReference` (including custom ones from `configurations:`) only rewrites a reference when the referenced object is in the same namespace as the referrer. A `configMapGenerator` output has no namespace unless the generator sets one, so a hashed ConfigMap name was not rewritten in a namespaced `HelmRelease`. Always look at the rendered output (`kubectl kustomize`, `flux build kustomization`), not just whether it builds.
- To inspect a Helm chart from an OCI registry without the `helm` CLI: get an anonymous token from `https://ghcr.io/token?scope=repository:<repo>:pull`, fetch `/v2/<repo>/manifests/<tag>` (Accept `application/vnd.oci.image.manifest.v1+json`), and download the chart layer from `/v2/<repo>/blobs/<digest>`.
- The Red Hat YAML extension for VS Code applies SchemaStore schemas by **filename**: `databases/pg-main/cluster.yaml` gets the RKE "Cluster Configuration" schema and reports `Property apiVersion is not allowed`. Fix with a `# yaml-language-server: $schema=<url>` modeline per file (CRD schemas from `datreeio/CRDs-catalog`) or workspace settings.
- `ERROR: cannot execute ... in a read-only transaction` in `psql` means the session is on a hot-standby replica (`SELECT pg_is_in_recovery()` returns `t`). Writes, DDL, and `GRANT`/`REVOKE` must go to the primary (`pg-main-rw`). Schema privileges are per database, so `\c <db>` first.

## Open items / next steps

1. **Prometheus** (on `feat/prometheus`, manifests ready, not yet merged): merge the PR, then run the checks listed under "kube-prometheus-stack". Follow-ups: a `PodMonitor` for `pg-main` (CNPG exposes metrics on port 9187), check real memory usage and adjust requests/limits, an Alertmanager receiver, and how to reach Grafana (port-forward until there is an ingress).
2. **Test a restore** of `pg-main` from R2, e.g. bootstrap a temporary second `Cluster` with `bootstrap.recovery` from `r2-store`. Backups that were never restored are not proven to work.
3. Kubernetes Secrets use Sealed Secrets (decided 2026-10-20). Still open: whether to commit `talos/secrets.yaml` encrypted as well (it is not a Kubernetes object, so Sealed Secrets doesn't fit; SOPS + age would), and a script for the regenerate commands in "Cluster config" (half-run regenerations happened on 2026-10-23).
4. Resource requests and limits for the sealed-secrets controller, cert-manager, and the Barman Cloud plugin (see `docs/TODO.md`).
5. Optional: label `talos-ii` as a worker, and think about laptop specifics (lid-close behavior, battery as a built-in UPS, Wi-Fi vs. Ethernet). `talos-ii` was unreachable from 2026-10-09 ~21:42 until 2026-10-10 (`NotReady`, Talos API `no route to host`); the cause was not recorded. It was apparently not schedulable again from about 2026-10-11 until 2026-10-20 17:54 UTC (see the `pg-main` lesson). Now that `talos-ii` holds a Postgres replica (currently the primary), its reliability matters more.
6. Decide whether `talos-ii` keeps `i915`. If not, delete `schematics/talos-ii.yaml`, remove the installer override from `nodes/talos-ii.yaml`, and run `talosctl upgrade` on `talos-ii` with the default schematic ID.
7. Clean up the `Completed` pods left over from node shutdowns, and the local branches that are already merged into `main` (`feat/local-path-provisioner`, `feat/cnpg-operator`, `feat/talos-config`, `feat/cnpg-database`, `feat/flux`).
8. Add `# yaml-language-server: $schema=...` modelines to the CRD manifests (CNPG, Flux, SealedSecrets) for editor validation, as a separate small PR.
9. Later: ingress/load balancer, CNI choice (keep Flannel or move to Cilium; Flannel does not enforce NetworkPolicies), metrics-server (`kubectl top` currently fails with "Metrics API not available"). The nodes also get public IPv6 addresses (SLAAC) from the router, which matters once services are exposed; confirm the router blocks inbound IPv6.

Done: DHCP reservations for both nodes; secret files backed up (2026-10-21); GitOps controller chosen and bootstrapped (Flux, 2026-10-22); stacked branches merged into `main`; backups for `pg-main` (Barman Cloud to R2, 2026-10-22).

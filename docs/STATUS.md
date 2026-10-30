# Cluster status

_Last updated: 2026-10-30_

## Summary

Two-node Talos cluster `super-hellthalos`: one control plane that also runs workloads (`talos-i`) and one worker (`talos-ii`, a Lenovo ThinkCentre M720q since 2026-10-30, previously an old laptop). Both nodes report `Ready`. Everything in Kubernetes is deployed by **Flux v2.9.6** from the `main` branch of this repo (bootstrapped 2026-10-22); `main` is protected, so every change goes through a PR. Argo CD runs next to it since 2026-10-30, deployed by Flux, and takes over components one at a time (see "Argo CD"). Each node has a dedicated 120 GiB data partition, with local-path-provisioner on top of it as the default StorageClass. Installed infrastructure: Cilium (CNI and kube-proxy replacement, replaced Flannel and kube-proxy on 2026-10-23; since 2026-10-24 also LoadBalancer IPs via LB-IPAM and L2 announcements, and Gateway API), the Gateway API CRDs, local-path-provisioner, Sealed Secrets, cert-manager (since 2026-10-24 also issuing a Let's Encrypt wildcard certificate via DNS-01), the CloudNativePG operator, the CNPG Barman Cloud plugin, kube-prometheus-stack (Prometheus, Grafana, Alertmanager, exporters; deployed 2026-10-23), and since 2026-10-25 a Gateway `main` plus `cloudflared`, which publishes Grafana at `https://grafana.superhellth.com` through a Cloudflare Tunnel behind Cloudflare Access. The Postgres `Cluster` `pg-main` is healthy with two instances, one per node (one primary and one streaming replica). It archives WAL continuously and takes a daily base backup to Cloudflare R2, and it holds the app database `wg_app`. Kubernetes Secrets are committed only in sealed form. The Talos config (patches, per-node patches, Image Factory schematics) is in this repo under `talos/`; its secrets are not committed but are backed up (see "Where things live"). Control-plane metrics endpoints are exposed for Prometheus, and all Prometheus targets report up (checked 2026-10-24).

The cluster was rebuilt on 2026-10-09: EPHEMERAL was wiped on both nodes to resize it, and etcd was re-bootstrapped (see "Disk layout" and "Lessons learned").

## Nodes

| | `talos-i` | `talos-ii` |
|---|---|---|
| Role | control plane (workloads allowed, no taint) | worker (ROLES shows `<none>`, see below) |
| IP | `192.168.0.192/24` | `192.168.0.193/24` |
| Gateway/DNS | `192.168.0.1` | `192.168.0.1` |
| DHCP reservation in router | yes | yes (MAC `f8:75:a4:22:1c:4c`) |
| Hardware | Intel i5 (6 logical CPUs), 16 GB RAM | Lenovo ThinkCentre M720q, Intel i5-9400T (6 cores, 6 threads), 16 GB RAM (2 × 8 GB) |
| Install disk | `/dev/nvme0n1`: WDC PC SN730 256 GB, serial `19422B800421` | `/dev/sda`: Samsung SSD 850 250 GB (SATA), carried over from the previous `talos-ii` with its partitions and data |
| Previous OS | Windows (wiped by the Talos install) | wiped by the Talos install |

`talos-ii` was an Acer Aspire E5-571G laptop (4 logical CPUs, 8 GB RAM) until 2026-10-30. Its SSD moved to the ThinkCentre, and the machine kept the name `talos-ii`, so it took over the existing Node object.

The worker has no `node-role.kubernetes.io/*` label because the NodeRestriction admission plugin stops a kubelet from giving itself role labels. It can be added by an admin with `kubectl label node talos-ii node-role.kubernetes.io/worker=`. This is cosmetic and has no effect on scheduling.

## Versions

| Component | Version |
|---|---|
| Talos | v1.14.2 (both nodes) |
| Kubernetes | v1.37.1 (both nodes) |
| containerd | 2.3.6 |
| kubectl (WSL) | v1.37.1 |
| Flux | v2.9.6 (source-controller v1.9.6, kustomize-controller v1.9.6, helm-controller v1.6.5, notification-controller v1.9.4) |
| Cilium | 1.20.4 (Helm chart, OCI) |
| kube-prometheus-stack | 82.16.0 (Helm chart, OCI) |
| Argo CD | v3.5.6 (Helm chart `argo-cd` 10.11.6, OCI) |
| Image Factory schematic (`talos-i`) | `2d61dd07b20062062ea671b4d01873506103b67c0f7a4c3fb6cf4ee85585dcb8`, source `talos/schematics/default.yaml` |
| Image Factory schematic (`talos-ii`) | configured: `4b3cd373a192c8469e859b7a0cfbed3ecc3577c4a2d346a37b0aeff9cd17cdb0`, source `talos/schematics/talos-ii.yaml`; running (2026-10-30): the default `2d61dd07...`, see "Open items" 6 |

Installer image: `factory.talos.dev/metal-installer/<schematic>:v1.14.2`. Both schematic files were checked against the running nodes (`talosctl get extensions`, where the `schematic` entry shows the running ID) and by POSTing them to the factory, which returns the same IDs. That check no longer holds for `talos-ii` as of 2026-10-30.

| Node | Extensions |
|---|---|
| `talos-i` | `intel-ucode` (20260812) |
| `talos-ii` | `intel-ucode` (20260812) |

`talos-ii` used to have `i915` (Intel iGPU driver) on top of the default, and `nodes/talos-ii.yaml` still points at that schematic, but the running node no longer has it. It only matters for GPU video transcoding. Which schematic `talos-ii` should use is still undecided (see "Open items").

## Cluster config

- Cluster name `super-hellthalos`, Kubernetes endpoint `https://192.168.0.192:6443`
- Talos defaults are used for everything except the patches below. Talos still deploys CoreDNS; Flannel and kube-proxy are disabled (since 2026-10-23), and Cilium, installed by Flux, takes over both jobs.
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
  - `KubeProxyConfig` with `enabled: false` and `KubeFlannelCNIConfig` with `$patch: delete` (2026-10-23, see "cilium")
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
- **kube-proxy is gone since 2026-10-23** (replaced by Cilium). The `KubeProxyConfig` document now only says `enabled: false`, so the `:10249` endpoint and the `metricsBindAddress` setting above are history. kube-prometheus-stack has `kubeProxy.enabled: false`, otherwise it keeps a ServiceMonitor for a target that no longer exists (permanently down).

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
  infrastructure/        Flux Kustomizations for infrastructure/, grouped by concern:
    networking.yaml      gateway-api-crds, cilium, cilium-config, gateway, cloudflared
    security.yaml        sealed-secrets, cert-manager, cert-manager-config
    storage.yaml         local-path-provisioner
    cnpg.yaml            cnpg-operator, cnpg-barman-plugin
    monitoring.yaml      kube-prometheus-stack
    gitops.yaml          argocd, argocd-config
  databases.yaml         Flux Kustomization for pg-main
  apps.yaml              Flux Kustomization for wg-app
argocd/
  apps/                  child Application manifests, synced by the root Application
infrastructure/          cluster-wide foundations (storage, operators, CRDs)
  gateway-api-crds/      upstream Gateway API CRDs (standard channel)
  cilium/                OCIRepository, HelmRelease, values.yaml
  cilium-config/         Cilium CRs: LoadBalancer IP pool, L2 announcement policy
  local-path-provisioner/
  sealed-secrets/        sealed-secrets-keys.yaml is gitignored
  cert-manager/
  cert-manager-config/   ClusterIssuers (Let's Encrypt staging + prod), Cloudflare token SealedSecret
  gateway/               Namespace gateway, wildcard Certificate, Gateway main
  cloudflared/           Namespace, Deployment, config.yaml (ConfigMap generator), credentials SealedSecret
  cnpg-operator/
  cnpg-barman-plugin/
  kube-prometheus-stack/ Namespace, OCIRepository, HelmRelease, values.yaml, SealedSecret
  argocd/                Namespace, OCIRepository, HelmRelease, values.yaml
  argocd-config/         AppProjects, root Application
databases/               things that use the foundations
  pg-main/               Cluster, Database, ObjectStore, ScheduledBackup, SealedSecrets
apps/                    workloads
  wg-app/                Namespace, Deployments, Services, SealedSecret
docs/                    STATUS.md (this file), TODO.md, roadmaps
```

### Flux (bootstrapped 2026-10-22)

- Flux v2.9.6, bootstrapped from `ssh://git@github.com/superhellth/k8s-homelab`, branch `main`, path `clusters/super-hellthalos`. All four controllers run in `flux-system`. helm-controller manages two `HelmRelease`s: kube-prometheus-stack and cilium.
- Flux installs its own `NetworkPolicy` objects in `flux-system` (`allow-egress`, `allow-scraping`, `allow-webhooks`). Under Flannel they were ignored; since Cilium they are enforced.
- `main` is protected: changes go through PRs, and Flux only reconciles what is merged. Force a sync after a merge with `flux reconcile kustomization <name> --with-source`.
- The `flux-system` Kustomization reconciles `clusters/super-hellthalos/`. Flux generates the kustomization for that directory itself, so every YAML file placed there is picked up (including subdirectories; checked with `flux build` on 2026-10-24). The Flux `Kustomization`s are grouped by concern (restructured 2026-10-24, when the single `infrastructure.yaml` got too long): `clusters/super-hellthalos/infrastructure/` holds one file per concern for everything in `infrastructure/` (`networking`, `security`, `storage`, `cnpg`, `monitoring`; see the layout above), and `databases.yaml` holds `pg-main`, mirroring the top-level `infrastructure/` and `databases/` directories. The subdirectory must not get its own `kustomization.yaml`: Flux only generates one (with every file) when none exists, and an explicit one would silently skip unlisted files. Kustomize rejects the same Flux Kustomization defined in two files (`may not add resource with an already registered id`), which happened once during the move. Moving a Kustomization between files changes nothing in the cluster, as long as its name stays the same: Flux tracks objects by kind, name, and namespace, and a rename would mean delete plus create, which prunes. Each Kustomization points at `./infrastructure/<component>` or `./databases/pg-main`, with `prune: true`, `wait: true`, `interval: 1h`, `retryInterval: 2m`, `timeout: 5m` (`kube-prometheus-stack`: `timeout: 15m`, `cilium`: `timeout: 10m`, see their sections).
- **Controller/config split:** a controller and the custom resources it acts on are separate Flux Kustomizations: `infrastructure/<x>/` installs the controller (and its CRDs), `infrastructure/<x>-config/` holds the CRs, wired with `dependsOn` (`cilium` → `cilium-config`, `cert-manager` → `cert-manager-config`; `cnpg-operator` → `pg-main` is the same pattern). Reasons: some CRDs only exist once the controller runs (Cilium creates its own), admission webhooks must be running, not just applied (cert-manager), and the two `Ready` states mean different things (pods running vs. CRs processed), so `flux get kustomizations` shows which layer is broken. The official Flux example uses `controllers/` and `configs/` directories for the same idea; not worth it here while only two components have a config part.
- Ordering uses `dependsOn`: `cilium` depends on `gateway-api-crds`, `cilium-config` on `cilium` (see "cilium"), `cert-manager-config` on `cert-manager` and `sealed-secrets`, `gateway` on `cert-manager-config`; `cnpg-barman-plugin` depends on `cert-manager` and `cnpg-operator`; `pg-main` depends on `local-path-provisioner`, `cnpg-operator`, `cnpg-barman-plugin`, and `sealed-secrets`; `kube-prometheus-stack` depends on `sealed-secrets` and `local-path-provisioner`; `argocd-config` depends on `argocd`. With `wait: true`, a Kustomization only counts as Ready once its objects are healthy, so dependents wait for running operators, not just applied manifests.
- **Prune protection:** `prune: true` deletes objects that disappear from Git, and deleting or renaming a Flux Kustomization prunes everything it applied. Deleting a CRD deletes every custom resource of that type, so Kustomizations that ship CRDs patch them with `kustomize.toolkit.fluxcd.io/prune: disabled`. The `pg-main` Kustomization does the same for the `Cluster` and the `Namespace` (pg-main was handed over to Flux on 2026-10-23). The key must be exactly `kustomize.toolkit.fluxcd.io/prune`; `kustomize.toolkit.io/...` is silently ignored.
  - The rule for what gets protected: *if Flux deleted this object and recreated it from Git a minute later, would anything be lost or broken beyond a short blip?* Yes for objects that **cascade** (CRDs, Namespaces), objects backed by **state not in Git** (CNPG `Cluster`), and objects whose deletion **breaks the machinery that would recreate them** (the Cilium `HelmRelease`). Everything else is pruned normally: protection orphans objects, so removing them from Git no longer removes them from the cluster.
  - The CRD patch targets `kind: CustomResourceDefinition` without a name, so it covers every CRD in the build, including ones a future upstream version adds. The `name: not-used` in the patch body is ignored when a `target` is set.
  - Audit a component: `kubectl kustomize infrastructure/<dir> | grep -E '^kind: (CustomResourceDefinition|Namespace|PersistentVolumeClaim)'`, then ask the question above for any stateful custom resources.
- kustomize-controller always uses server-side apply, so the CNPG CRDs (too big for client-side apply) need no special handling under Flux.
- Render exactly what Flux will apply, including the Kustomization's patches: `flux build kustomization <name> --path ./infrastructure/<dir> --kustomization-file ./clusters/super-hellthalos/<file>.yaml --dry-run`, where `<file>` is the file that defines that Kustomization (`infrastructure/networking`, `infrastructure/security`, `infrastructure/storage`, `infrastructure/cnpg`, `infrastructure/monitoring`, or `databases`). With the wrong file it fails with `failed find kustomization with name ...`.
- Validate changed Flux Kustomizations against the live CRD schema before a PR: `kubectl apply --dry-run=server -f clusters/super-hellthalos/infrastructure/ -f clusters/super-hellthalos/databases.yaml 2>&1 | grep -E 'created|rror'` (changes nothing). Only new Kustomizations should show `created`; existing ones report `configured` (client-side apply wants to add its `last-applied-configuration` annotation), which is noise. Don't use `-R` on `clusters/super-hellthalos/`: it also picks up `flux-system/kustomization.yaml`, which is not a Kubernetes object. `flux build` (even of the root) does not catch schema errors in the Kustomization objects themselves.
- Check for duplicate Flux Kustomizations: `grep -rh -A2 '^kind: Kustomization' clusters/super-hellthalos/ --include='*.yaml' | grep 'name:' | sort | uniq -c` (every count must be 1).

### Argo CD (installed 2026-10-30)

- Helm chart `argo-cd` 10.11.6 (Argo CD v3.5.6) from `oci://ghcr.io/argoproj/argo-helm/argo-cd`, `HelmRelease` in namespace `argocd`, values from `values.yaml` through a hashed `configMapGenerator` ConfigMap. Flux Kustomization `argocd` in `infrastructure/gitops.yaml` (`timeout: 10m`). Dex and notifications are off. Plan: `docs/argocd-migration-roadmap.md`.
- `infrastructure/argocd-config/`, Flux Kustomization `argocd-config` in `infrastructure/gitops.yaml`, `dependsOn` `argocd`: the `Application` and `AppProject` CRDs come from the Helm chart, so these objects cannot be applied together with the install on a fresh cluster. Not prune-protected: the objects are recreatable from Git, and deleting the root does not cascade (no finalizer).
- **App-of-apps:** Flux applies the root `Application` once, which solves the bootstrap problem (something has to create the first `Application`). The root syncs `argocd/apps/` to the `argocd` namespace, with `prune` and `selfHeal` on. `allowEmpty` stays `false`, so auto-sync refuses to prune everything if the directory comes up empty. The root's own destination is `argocd` because its output is `Application` objects; their workloads go wherever each child's `destination` says.
- **AppProjects:** `apps` allows namespaced resources plus `Namespace`, with destinations limited to app namespaces (`wg-app`). `infrastructure` allows cluster-scoped resources, with destinations limited to `argocd` for now; every handover adds its namespace. Both only accept this repo as a source. An unset `clusterResourceWhitelist` denies all cluster-scoped resources, while an unset `namespaceResourceWhitelist` allows all namespaced ones. Argo checks the namespace of every rendered resource against `destinations`, not just the `Application`'s `destination`. The chart's `default` project still allows everything, so projects are guardrails here, not a security boundary.
- Repo access: HTTPS without credentials, because `superhellth/k8s-homelab` is public (checked 2026-10-30). A private repo would need a deploy key as a SealedSecret.
- The chart installs `NetworkPolicy` objects in `argocd`. They restrict ingress only, so the repo-server can reach GitHub.
- No `argocd` CLI installed yet, and the UI has no route. Use `kubectl -n argocd port-forward svc/argocd-server 8080:443`; the initial admin password is in the Secret `argocd-initial-admin-secret`.

**Ownership** (one owner per object; update in every handover PR):

| Component | Owner |
|---|---|
| Flux itself, `cilium`, `cilium-config`, `gateway-api-crds`, `gateway`, `cloudflared` | Flux |
| `sealed-secrets`, `cert-manager`, `cert-manager-config`, `local-path-provisioner` | Flux |
| `cnpg-operator`, `cnpg-barman-plugin`, `pg-main` | Flux |
| `kube-prometheus-stack`, `wg-app` | Flux |
| `argocd`, `argocd-config` (AppProjects, root `Application`) | Flux |
| Child `Application`s in `argocd/apps/` | Argo CD (none yet) |

### cilium (installed 2026-10-23)

- Helm chart `cilium` **1.20.4** from `oci://quay.io/cilium/charts/cilium` (`OCIRepository` with `layerSelector`, same pattern as kube-prometheus-stack), `HelmRelease` in `kube-system`, values from `values.yaml` through a hashed `configMapGenerator` ConfigMap (`kustomizeconfig.yaml`). Flux Kustomization `cilium` in `infrastructure/networking.yaml` (`timeout: 10m`). Its only `dependsOn` is `gateway-api-crds` (since 2026-10-24): the chart's `gatewayAPI.gatewayClass.create: auto` checks at render time whether the `GatewayClass` API exists, and the operator only enables Gateway API if the CRDs are there when it starts. The dependency did not hold on the rollout that introduced it (see "Lessons learned (Kubernetes / tooling)"). The dependency is safe even though everything else needs pod networking: applying CRDs needs only the API server, no pods. It patches the `HelmRelease` with `kustomize.toolkit.fluxcd.io/prune: disabled`: if the Kustomization is ever removed, uninstalling the CNI would cut every pod off the network.
- Values (see `values.yaml`):
  - `ipam.mode: kubernetes`: Cilium uses the per-node pod CIDRs that the controller-manager assigns (`10.244.0.0/24` on `talos-i`, `10.244.1.0/24` on `talos-ii`, Talos default pod network `10.244.0.0/16`), so pod IPs stayed in the same range as under Flannel.
  - `kubeProxyReplacement: true` with `k8sServiceHost: localhost`, `k8sServicePort: 7445`: Services are handled by eBPF programs instead of kube-proxy's iptables rules. Without kube-proxy, the `kubernetes` Service (`10.96.0.1`) only works once Cilium runs, so Cilium itself reaches the API server through **KubePrism**, Talos's node-local API load balancer on `localhost:7445`.
  - `cgroup.autoMount.enabled: false`, `cgroup.hostRoot: /sys/fs/cgroup`, and an explicit capability list without `SYS_MODULE`: Talos mounts cgroup v2 itself and does not allow loading kernel modules.
  - Resources: agent 100m / 256Mi request, 1Gi limit; operator `replicas: 1` (two nodes, and a second replica just waits for leader election), 25m / 64Mi, 256Mi.
  - `l2announcements.enabled: true` (2026-10-24): Cilium implements `type: LoadBalancer` Services on bare metal. LB-IPAM assigns an IP from a `CiliumLoadBalancerIPPool`; for each Service, one node holds a Kubernetes Lease and answers ARP for that IP with its own MAC. If the node dies, another one takes the lease and sends a gratuitous ARP (failover after the lease expires, ~15 s by default). Requires `kubeProxyReplacement: true`, and the announcing interfaces must be among Cilium's devices (auto-detected: `eno1` on both nodes; check with `cilium-dbg status`, line `KubeProxyReplacement`).
  - `k8sClientRateLimit` `qps: 20`, `burst: 40` (default 10/20): the agent renews one Lease per announced Service, from the same client budget as everything else it does.
  - `gatewayAPI.enabled: true` (2026-10-24): Cilium implements Gateway API with the `cilium-envoy` DaemonSet and creates `GatewayClass cilium`. Each `Gateway` gets a `LoadBalancer` Service, which gets its IP from LB-IPAM. Needs `l7Proxy: true` (chart default) and the Gateway API CRDs (see "gateway-api-crds"). Changing values does **not** restart anything: the chart defaults `rollOutCiliumPods`, `operator.rollOutPods`, and `envoy.rollOutPods` are `false`, and agent and operator read the `cilium-config` ConfigMap only at startup. After a values change, `kubectl -n kube-system rollout restart ds/cilium deploy/cilium-operator` (proposed follow-up: set the three `rollOutPods` values to `true`).
  - Everything else is the chart default: VXLAN tunnel between the nodes, iptables masquerading, legacy host routing, Hubble enabled (flows only, no metrics, no UI/relay), `cilium-envoy` DaemonSet as the L7 proxy.
- Cutover (2026-10-23): Cilium was installed first while Flannel was still running (temporarily on `tunnelPort: 8473`, because both use VXLAN port 8472), then Flannel and kube-proxy were disabled in `patch-controlplane.yaml`, and the tunnel port override was removed again.
- Check: `kubectl -n kube-system exec ds/cilium -c cilium-agent -- cilium-dbg status` (`KubeProxyReplacement: True`, controllers healthy, `Cluster health: 2/2 reachable`) and `cilium-health status --verbose` for the probes between nodes.
- State 2026-10-24: both agents healthy, cross-node pod traffic works, no Flannel or kube-proxy objects left. `Cluster health` showed `1/2 reachable` because of a missing health listener on `talos-i` (see "Lessons learned (Kubernetes / tooling)"). Pod traffic was not affected.

### gateway-api-crds (added 2026-10-24)

- Upstream `standard-install.yaml` of Gateway API **v1.6.3**, referenced by URL in `infrastructure/gateway-api-crds/kustomization.yaml`. The version matches what the Cilium 1.20.4 docs pin; bump it together with Cilium and read Cilium's upgrade notes on Gateway API first.
- Contents: 10 CRDs (`gatewayclasses`, `gateways`, `httproutes`, `grpcroutes`, `tlsroutes`, `tcproutes`, `udproutes`, `referencegrants`, `backendtlspolicies`, `listenersets`, all `.gateway.networking.k8s.io`) and the `ValidatingAdmissionPolicy`/`Binding` `safe-upgrades.gateway.networking.k8s.io`, which rejects installing older or experimental-channel CRDs over these.
- Flux Kustomization `gateway-api-crds` in `infrastructure/networking.yaml`, CRDs prune-protected: deleting e.g. the `httproutes` CRD would delete every `HTTPRoute` in the cluster, including those owned by other Kustomizations. The admission policy is not protected: it holds no data and Flux recreates it.

### cilium-config (added 2026-10-24)

- `infrastructure/cilium-config/`: Cilium custom resources, in a separate Flux Kustomization (`infrastructure/networking.yaml`) with `dependsOn: cilium`, because their CRDs are created by the Cilium operator at runtime. Both objects are cluster-scoped.
- `CiliumLoadBalancerIPPool` `lan` (`cilium.io/v2`): `192.168.0.240`–`192.168.0.250`. The router's DHCP range ends at `.229` (shortened on 2026-10-24); `.230`–`.239` stay free as a buffer. The pool has no `serviceSelector`, so **every** `type: LoadBalancer` Service gets an IP from it, including ones a Helm chart creates by default. Pin an IP with the Service annotation `lbipam.cilium.io/ips` (for a Gateway: `spec.infrastructure.annotations`).
- `CiliumL2AnnouncementPolicy` `lan` (`cilium.io/v2alpha1`): `loadBalancerIPs: true`, `interfaces: [^en.*]` (matches `eno1`, not `bond0`, `dummy0`, `lxc*`, `cilium_*`; check NIC names with `talosctl -n 192.168.0.192,192.168.0.193 get links`). No `nodeSelector`, so both nodes can announce.
- Not prune-protected on purpose: no cascade and no state. If the pool disappears, Services lose their external IP until it is back, and pinned IPs come back unchanged.
- Since 2026-10-25 the Gateway's Service `cilium-gateway-main` gets an unpinned IP from this pool (currently `192.168.0.240`, the first free one) and is announced on the LAN like every other LoadBalancer Service. That is a way around Cloudflare Access, see "cloudflared".
- Check: `kubectl get ciliumloadbalancerippool` (`IPS AVAILABLE`), `kubectl get svc -A --field-selector spec.type=LoadBalancer`, `kubectl -n kube-system get lease | grep cilium-l2announce` (which node announces which Service), and `arp -a` from a LAN machine.

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
- Deployment `sealed-secrets-controller` in `kube-system`. Resource requests and limits are set by a patch in `kustomization.yaml` (2026-10-23).
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
- Installed because the Barman Cloud plugin needs it: the plugin's gRPC connection to the CNPG operator uses mTLS certificates that cert-manager issues (the plugin brings its own `Issuer` `selfsigned-issuer` and two `Certificate`s in `cnpg-system`). Since 2026-10-24 it also issues public certificates, see "cert-manager-config".
- Resource requests and limits are set by patches in `kustomization.yaml` (2026-10-23).
- What the three Deployments do: `cert-manager` is the controller (drives each `Certificate` to an issued Secret and renews it), `cert-manager-webhook` validates every cert-manager object on admission (so the CRs need it *running*, see the controller/config split under "Flux"), `cert-manager-cainjector` copies CA bundles into webhook configurations and CRDs.
- Runs with `--cluster-resource-namespace=$(POD_NAMESPACE)`: Secrets referenced by a `ClusterIssuer` (API tokens, ACME account keys) must live in `cert-manager`.
- Flux Kustomization `cert-manager` in `infrastructure/security.yaml`, CRDs prune-protected.

### cert-manager-config (added 2026-10-24)

- `infrastructure/cert-manager-config/`, Flux Kustomization `cert-manager-config` in `infrastructure/security.yaml`, `dependsOn` `cert-manager` (CRDs and a running webhook) and `sealed-secrets` (to decrypt the token). Not prune-protected: issuers and the token are fully recreatable from Git (a new ACME account would be registered).
- `SealedSecret` `cloudflare-api-token` (namespace `cert-manager`, key `api-token`): Cloudflare API token with **Zone · DNS · Edit** and **Zone · Zone · Read** (cert-manager needs the latter to look up the zone ID), limited to the zone `superhellth.com`. Cloudflare tokens cannot be scoped below a zone, so the token can edit every record in `superhellth.com`, including `wg.superhellth.com` (accepted; options if that ever matters: client IP filtering on the token, or delegating `_acme-challenge` via CNAME to a separate domain with `cnameStrategy: Follow`). Sealed with `read -rs CF_TOKEN` so the token never lands in the shell history; checked with `kubeseal --validate` before the PR.
- `ClusterIssuer` `letsencrypt-staging` and `letsencrypt-prod`: ACME (staging and production endpoints), each with its own account key Secret (`letsencrypt-<env>-account-key` in `cert-manager`, generated by cert-manager on first use; `Ready` means the account is registered). One solver: DNS-01 via Cloudflare with the token above, `selector.dnsZones: [superhellth.com]`. The selector is not the Cloudflare zone, only the rule which names this solver handles; it must match **every** `dnsName` of a Certificate, otherwise the order fails with `no configured challenge solvers can be used`. Staging stays for experiments (generous limits, not browser-trusted); production allows 5 certificates per exact set of names per week.
- DNS-01: cert-manager creates `_acme-challenge.<name>` TXT records through the API, checks itself that they are visible (it looks up the zone via the normal pod DNS, then asks Cloudflare's nameservers directly), then lets Let's Encrypt validate, and deletes the records. Nothing has to be reachable from the internet, and it is the only ACME challenge that can issue wildcards. The optional cert-manager flags `--dns01-recursive-nameservers-only` / `--dns01-recursive-nameservers=1.1.1.1:53,9.9.9.9:53` were not needed; they help with split-horizon DNS or a self-check stuck at `DNS record ... not yet propagated`.

### gateway (added 2026-10-24)

- `infrastructure/gateway/`, Flux Kustomization `gateway` in `infrastructure/networking.yaml`, `dependsOn` `cert-manager-config`. With `wait: true` it only becomes `Ready` once the certificate is issued, so it shows whether the whole chain (token → issuer → challenge → Let's Encrypt) works.
- `Namespace` `gateway`: home of the shared entry point (the wildcard certificate and the `Gateway`). A `Gateway` may only reference TLS Secrets in its own namespace (otherwise a `ReferenceGrant` is needed), and keeping the private key out of app namespaces limits who can read it. Not prune-protected: nothing stateful in it.
- `Certificate` `wildcard` → Secret `wildcard-tls`, `dnsNames` `*.superhellth.com` (tunnel names, see "Decisions"). Until 2026-10-25 it also had `*.k8s.superhellth.com` for the LAN names; dropping it made cert-manager issue a new prod certificate (checked: issuer `Let's Encrypt YR2`, one SAN). Wildcards keep individual app names out of the public Certificate Transparency logs and need no new certificate per app; a wildcard covers exactly one label. Issued by `letsencrypt-staging` first (2026-10-24, about 3 minutes after the merge, issuer `(STAGING) Ersatz Emmer YR2`), then switched to `letsencrypt-prod`: cert-manager re-issues by itself when the Secret was issued by a different issuer than `issuerRef` names.
- Lifetime: Let's Encrypt certificates are now valid for 64 days (not 90); cert-manager renews after 2/3 of the lifetime by default.
- `Gateway` `main` (added 2026-10-25, PR #12): `gatewayClassName: cilium`, one listener `https` (HTTPS, port 443, hostname `*.superhellth.com`, `tls.mode: Terminate` with `wildcard-tls`, routes allowed from all namespaces). Cilium programs Envoy for it and creates the `LoadBalancer` Service `cilium-gateway-main` (ClusterIP plus an LB-IPAM IP, see "cilium-config"). No pinned `addresses` anymore: the first version pinned `192.168.0.240` for the LAN listener `https-lan` (`*.k8s.superhellth.com`), which was dropped before the tunnel went live.
- `HTTPRoute` `grafana` lives next to Grafana in `infrastructure/kube-prometheus-stack/`: parent `main`/`https`, hostname `grafana.superhellth.com`, backend `kube-prometheus-stack-grafana`. Grafana's `root_url` is `https://grafana.superhellth.com` to match.
- Checks: `kubectl get clusterissuer`, `kubectl -n gateway get certificate,certificaterequest,order,challenge`, `kubectl -n gateway describe certificate wildcard`, and the issued cert: `kubectl -n gateway get secret wildcard-tls -o jsonpath='{.data.tls\.crt}' | base64 -d | openssl x509 -noout -issuer -dates -ext subjectAltName`. If issuance hangs, the deepest object in Certificate → CertificateRequest → Order → Challenge has the reason (`kubectl -n gateway describe challenge`).

### cloudflared (deployed 2026-10-25)

- `infrastructure/cloudflared/`, Flux Kustomization `cloudflared` in `infrastructure/networking.yaml`, `dependsOn` `sealed-secrets` (credentials) and `gateway` (the origin). Not prune-protected: no CRDs, no state.
- **Locally managed tunnel** `225496ab-5394-45ec-b3a7-b44ceb3d13e8`: the routing table is `config.yaml` in Git, not the Cloudflare dashboard (a "remotely managed" tunnel with `--token` would keep the ingress rules at Cloudflare).
- How it works: `cloudflared` dials **out** to Cloudflare's edge (4 QUIC connections per replica, e.g. `fra06`, `dus01`), so no router port is open. The edge sends requests for a hostname into the tunnel, `cloudflared` matches the hostname against its `ingress` rules and opens a new TLS connection to the origin.
- `config.yaml` (turned into the ConfigMap `cloudflared-config-<hash>` by a `configMapGenerator`; the hash changes the pod template, so a config change rolls the pods):
  - `tunnel`, `credentials-file: /etc/cloudflared/creds/credentials.json`
  - `metrics: 0.0.0.0:2000`: Prometheus `/metrics` and `/ready` (200 only with an active edge connection). `0.0.0.0` because kubelet probes and Prometheus connect to the pod IP.
  - `no-autoupdate: true`: the image tag is the version, not cloudflared's self-update.
  - `ingress`: `grafana.superhellth.com` → `https://cilium-gateway-main.gateway.svc.cluster.local:443` (the Gateway's ClusterIP), then the mandatory catch-all `http_status:404`. First match wins. This is cloudflared's own routing table and has nothing to do with the Kubernetes `Ingress` API.
  - `originRequest.matchSNItoHost: true`: sends the request's `Host` as TLS SNI to the Gateway. Without it the SNI would be the `svc.cluster.local` name, which matches no listener and no certificate.
- `SealedSecret` `cloudflared-credentials` (key `credentials.json`: account tag, tunnel ID, tunnel secret). Mounted at `/etc/cloudflared/creds`. The tunnel ID alone is not secret, so it sits in the plain ConfigMap.
- `Deployment` `cloudflared`: 2 replicas, `cloudflare/cloudflared:2026.10.3`, `args: tunnel --config /etc/cloudflared/config/config.yaml run`.
  - `topologySpreadConstraints` on `kubernetes.io/hostname`, `maxSkew: 1`, `ScheduleAnyway` (soft: if one node is down both replicas run on the other), `matchLabelKeys: [pod-template-hash]` (only count pods of the same rollout). Only evaluated at scheduling time; nothing rebalances later. Verified: one replica per node.
  - Readiness and liveness on `/ready` (named port `metrics`); liveness tolerates ~1 min (`failureThreshold: 6`), because restarts do not help when the internet is down.
  - Resources 10m / 64Mi request, 256Mi memory limit, no CPU limit (estimates).
  - Fully PSA `restricted`: pod `runAsNonRoot`, `runAsUser`/`runAsGroup: 65532` (the image declares a named user, which `runAsNonRoot` cannot verify), `seccompProfile: RuntimeDefault`; container `allowPrivilegeEscalation: false`, `capabilities.drop: [ALL]`, `readOnlyRootFilesystem: true`. The namespace enforces `restricted`.
- **Access is a gate at the Cloudflare edge, before the tunnel.** Tunnel, DNS record and Access application are separate Cloudflare objects that are only linked by the hostname: if a proxied DNS record points at the tunnel and no Access application covers the hostname, the edge forwards everyone. Order for a new app: Access application first, cluster side second, DNS record last.
- **Known bypass:** the Gateway's LB IP is announced on the LAN, so any LAN device can reach Grafana directly (`curl --resolve grafana.superhellth.com:443:192.168.0.240 ...`) without passing Access. A DNS name that does not point there is not access control. See "Open items".
- Checks: `kubectl -n cloudflared logs deploy/cloudflared | grep 'Registered tunnel connection'`, `curl -sI https://grafana.superhellth.com` (expect `302` to `blue-hat-330a.cloudflareaccess.com`; Grafana's own `302 /login` means Access does not cover the hostname; `502`/error 1033 means no connected tunnel or no matching ingress rule).

### cnpg-barman-plugin (installed 2026-10-21)

- Upstream `plugin-barman-cloud` v0.15.1 `manifest.yaml`, referenced by URL in `infrastructure/cnpg-barman-plugin/kustomization.yaml`. Deployment `barman-cloud` in `cnpg-system`, plus the `ObjectStore` CRD (`barmancloud.cnpg.io/v1`).
- One local JSON patch lowers the log level from `debug` to `error`. It starts with an `op: test` on `args[6]`, so the render fails loudly if a future upstream release moves the argument, instead of silently replacing the wrong one.
- This is CNPG's CNPG-I plugin architecture: the operator talks to the plugin over gRPC, and the plugin injects a sidecar into each instance pod that ships WAL and base backups to object storage. It replaces the in-tree `barmanObjectStore` field, which is deprecated.
- Resource requests and limits are set by a patch in `kustomization.yaml` (2026-10-23). The same PR (#4) also set them for local-path-provisioner.

### kube-prometheus-stack (deployed 2026-10-23)

- Helm chart `kube-prometheus-stack` **82.16.0** (Prometheus Operator v0.89.0), installed by helm-controller from `infrastructure/kube-prometheus-stack/`. Everything runs in namespace `monitoring`. Flux Kustomization `kube-prometheus-stack` in `infrastructure/monitoring.yaml`, `dependsOn` `sealed-secrets` (Grafana secret) and `local-path-provisioner` (PVCs), `timeout: 15m`.
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
  - `kubeProxy.enabled: false` (2026-10-23): kube-proxy was replaced by Cilium.
  - `grafana.initChownData.enabled: false` (2026-10-23, PR #7): turns off the init container that `chown`s the Grafana data directory. The reason was not recorded in the commit.
- Verified 2026-10-24: Prometheus and Grafana PVCs `Bound` on `talos-i`, and every active target is up (apiserver, kubelet, coredns, etcd, controller-manager, scheduler, kube-state-metrics, node-exporter on both nodes, and the stack's own components).
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
  - Restore tested 2026-10-25 (see "Open items", item 3).
- History: the first attempt (2026-10-11) deadlocked and was deleted and recreated on 2026-10-20. See "Lessons learned (Kubernetes / tooling)".

### Decisions

- **CNPG runs in this cluster, not a separate one.** A second cluster would double the control-plane overhead and split the two nodes into two single-node clusters. Isolation happens inside the cluster instead: own namespace, Guaranteed QoS (requests = limits) for Postgres pods, optionally a PriorityClass. Network isolation via NetworkPolicy is possible since the move to Cilium (2026-10-23); `databases` has no policy yet.
- **Known limits of this setup:** etcd and Postgres share the NVMe on `talos-i` (watch for `slow fdatasync` warnings from etcd). CNPG failover runs through the API server, so if `talos-i` (the only control plane) dies, a replica on `talos-ii` is **not** promoted automatically.
- **Backups: CNPG Barman Cloud plugin to Cloudflare R2** (decided 2026-10-21). Volume snapshots are not an option, because local-path doesn't support them. The plugin gives continuous WAL archiving plus base backups, so point-in-time recovery, and the data leaves the house. The alternatives were `pg_dump` (only point-in-time snapshots, no WAL) and Velero file-level backups with Kopia (not consistent for a running database without extra hooks).
- **GitOps: Flux** (bootstrapped 2026-10-22), not Argo CD. The reasons were not recorded at the time. Relevant differences for this setup: Flux has no UI or API server of its own (less RAM), and everything is a CRD reconciled by a controller; Argo CD offers a web UI for visualizing sync state.
- **Move from Flux to Argo CD gradually** (roadmap written 2026-10-27), mainly to learn Argo CD; a full switch is optional. From the leaves inward, one component at a time, starting with `wg-app`. Argo CD was installed through a Flux `HelmRelease` rather than the roadmap's preferred upstream `install.yaml`.
- **App-of-apps with hand-written `Application`s and two `AppProject`s** (2026-10-30), set up before the first handover so that no `Application` has to move between projects later. Rejected: one `Application` in the `default` project first (faster, but the structure would be rebuilt later).
- **No deploy key for Argo CD** (2026-10-30): the repo is public, so HTTPS without credentials works and there is one secret less. This differs from the roadmap's first plan.
- **Control-plane metrics on `0.0.0.0`** (2026-10-23), see "Control-plane metrics endpoints".
- **Monitoring: kube-prometheus-stack via Flux `HelmRelease` from an `OCIRepository`** (2026-10-23). OCI instead of a `HelmRepository`, which downloads the repo's whole multi-MB `index.yaml` on every interval; OCI fetches one tag, like a container image.
- **One Prometheus replica, pinned to `talos-i`** (2026-10-23). Prometheus replicas don't replicate: each one scrapes everything independently, so an HA pair doubles memory and disk and needs a dedup layer (Thanos Query) to query cleanly. With a single control plane, losing `talos-i` takes the API down anyway. `talos-i` has the RAM and the SSD, and the `nodeSelector` makes the permanent `WaitForFirstConsumer` binding a deliberate choice.
- **Prometheus TSDB on local-path, not elsewhere** (2026-10-23). Prometheus only writes its own TSDB format; it cannot store data in Postgres. Durable metrics would mean shipping blocks to object storage (Thanos sidecar, Mimir), possibly a later learning project.
- **CNI: Cilium with kube-proxy replacement, instead of Flannel** (2026-10-23). Flannel only connects pods and ignores `NetworkPolicy`. Cilium enforces NetworkPolicies (plus its own L7-aware `CiliumNetworkPolicy`), replaces kube-proxy's iptables rules with eBPF, and brings Hubble for flow visibility. Installed by Flux as a `HelmRelease` like everything else, not as a Talos inline manifest; the trade-off is the rebuild ordering problem under "Open items".
- **Exposing services: Cilium LB-IPAM + L2 announcements, Cilium Gateway API, Cloudflare Tunnel** (2026-10-24). The goal is Grafana (and later other apps) reachable without `kubectl port-forward`, first on the LAN, then from outside.
  - L4: Cilium L2 announcements instead of MetalLB (same job, but MetalLB would be an extra component next to the CNI that already does it) and instead of BGP (the home router does not speak it). `NodePort` was rejected: high ports and a URL tied to one node.
  - L7: Gateway API instead of Ingress. The Ingress API is frozen and ingress-nginx is retired upstream. Gateway API splits the platform part (`Gateway`, listener, certificate) from per-app `HTTPRoute`s. Cilium implements it, so no separate controller (Traefik, Envoy Gateway).
  - TLS: cert-manager with Let's Encrypt DNS-01 through the Cloudflare API, so LAN-only names get real certificates without anything reachable from the internet.
  - From outside: Cloudflare Tunnel (`cloudflared` dials out, no router port forward, home IP hidden) with Cloudflare Access in front. Rejected: a router port forward (cluster directly on the internet) and Tailscale (fine for one person, but every client needs the app). Trade-off: Cloudflare terminates TLS and sees the traffic.
- **Hostnames: tunnel only, at `*.superhellth.com`** (2026-10-24, LAN part dropped 2026-10-25). The zone `superhellth.com` is on Cloudflare and already serves `wg.superhellth.com` (proxied); it has no CAA records and no wildcard record.
  - LAN (dropped 2026-10-25): the first plan had `<app>.k8s.superhellth.com` via a wildcard record `*.k8s.superhellth.com` → A `192.168.0.240` (DNS only), a second listener and a second SAN. Removed to keep one entry point behind Access; the DNS record was deleted. Trade-off: LAN clients also go through Cloudflare, so apps are unreachable while the internet is down.
  - Tunnel: `<app>.superhellth.com`, first level only, because Cloudflare's free Universal SSL covers `superhellth.com` and `*.superhellth.com` but not deeper names (that would need Advanced Certificate Manager). Opt-in per app: explicit hostnames in the cloudflared config and one proxied DNS record each, never a wildcard record pointing at the tunnel, so the tunnel cannot catch names like `wg`.
  - The cluster's `*.superhellth.com` key is technically valid for `wg.superhellth.com` too. Low risk while `wg` sits behind Cloudflare's proxy; step 4 may drop that SAN if cloudflared can reach the Gateway without it.
  - Replaces the `*.home.<domain>` naming from the first plan.
- **Grafana: SQLite on a local-path PVC, not `pg-main`** (2026-10-23). Storing Grafana's state in `pg-main` would tie the monitoring UI to the system it monitors: when Postgres is in trouble, Grafana can't log you in. Everything in Grafana's database is either rebuildable or should become dashboards-as-code anyway.

## Where things live

The Talos files moved from `~/talos` into `talos/` in this repo (2026-10-10). The rule for what gets committed: sources without secrets are committed, generated files are not, and secrets are committed only once they are encrypted.

- Committed: `patch-all.yaml`, `patch-controlplane.yaml`, `nodes/*.yaml`, `schematics/*.yaml`
- Committed in encrypted form: SealedSecrets (`databases/pg-main/sealedsecret-wg-app.yaml`, `databases/pg-main/sealedsecret-r2-backup.yaml`, `infrastructure/kube-prometheus-stack/sealedsecret-grafana-admin.yaml`, `infrastructure/cert-manager-config/` Cloudflare token, `infrastructure/cloudflared/sealedsecret-cloudflared-credentials.yaml`)
- Not committed: `.claude/` is gitignored, so `.claude/CLAUDE.md` (instructions for Claude Code) exists only in the working tree.
- Gitignored and plaintext in the working tree. **The secret files are backed up outside the working tree** (as of 2026-10-21):
  - `talos/secrets.yaml`: cluster CAs and tokens. Losing it means losing admin access. `git clean -fdx` would delete the working-tree copy.
  - `infrastructure/sealed-secrets/sealed-secrets-keys.yaml`: the Sealed Secrets controller's private key. It is needed to decrypt the committed SealedSecrets after a rebuild.
  - `talos/_out/`: generated machine configs (`controlplane.yaml`, `worker.yaml`, `talos-i.yaml`, `talos-ii.yaml`) and `talosconfig`. They all contain secrets and can be regenerated from the sources plus `secrets.yaml`.
- `talosctl` uses `~/.talos/config` (context `super-hellthalos`, endpoint `192.168.0.192`, client cert expires 2027-10-05). `TALOSCONFIG` is not set.
- kubeconfig merged into `~/.kube/config`

### State outside Git: Cloudflare (as of 2026-10-25)

Created by hand (CLI or dashboard), not reconciled by anything. On a rebuild the cluster side comes back from Git; these must still exist or be recreated:

| Object | Value | Created with |
|---|---|---|
| Tunnel | `225496ab-5394-45ec-b3a7-b44ceb3d13e8` (locally managed) | `cloudflared tunnel create` |
| DNS record | `grafana.superhellth.com` CNAME `225496ab-5394-45ec-b3a7-b44ceb3d13e8.cfargotunnel.com`, **proxied** | `cloudflared tunnel route dns <id> <hostname>` or dashboard |
| Access application | `grafana.superhellth.com`, team domain `blue-hat-330a.cloudflareaccess.com` (policy details not recorded yet) | Zero Trust dashboard |
| R2 bucket, API token | see "pg-main" and "cert-manager-config" | dashboard |

- The record must be proxied: `<id>.cfargotunnel.com` is only an identifier the edge understands, not a reachable address.
- One proxied record per app, never a wildcard pointing at the tunnel (see "Decisions").
- Deleted 2026-10-25: `*.k8s.superhellth.com` → `192.168.0.240`.
- `~/.cloudflared/cert.pem` (from `cloudflared tunnel login`) is an account-level credential that can create and delete tunnels and DNS records. The cluster does not need it. Treat it like a password or delete it and log in again when needed. `~/.cloudflared/<tunnel-id>.json` is the plaintext of the SealedSecret.
- Possible later: manage these with OpenTofu/Terraform (Cloudflare provider) or external-dns, see "Open items".

## Lessons learned (Talos 1.14)

- 1.14 generates multi-document configs. `.machine.install` is replaced by `UnattendedInstallConfig`, and node labels, taints, and `allowSchedulingOnControlPlanes` are replaced by `KubeNodeConfig`. Old-style fields in patches cause "already set in v1alpha1 config" errors.
- `--config-patch` applies to **both** controlplane and worker configs. A `$patch: delete` on a key missing from the worker config fails with `lookup failed`, so role-specific patches go in `--config-patch-control-plane` / `--config-patch-worker`.
- The generated `UnattendedInstallConfig` sets `wipe: false` explicitly. The install still repartitions the disk.
- Machine-specific settings (hostname, install disk) belong in per-node patches, not in `patch-all.yaml`. A disk selector like `dev_path == "/dev/nvme0n1"` doesn't match a laptop with a SATA disk.
- Worker nodes show ROLES `<none>`. That is normal Kubernetes behavior, not a Talos problem.
- `VolumeConfig` and `UserVolumeConfig` only take effect when a volume is provisioned. Applying a new `maxSize` to an existing EPHEMERAL does nothing. In the meantime, a user volume that does not fit shows `failed` in `talosctl get volumestatus`. To resize EPHEMERAL: apply the config, then run `talosctl reset --system-labels-to-wipe EPHEMERAL --reboot`. On boot, Talos recreates EPHEMERAL with the new size, and the user volume gets the freed space.
- Wiping EPHEMERAL on the only control plane deletes etcd, which means the whole cluster state. It needs `--graceful=false`, because a graceful reset tries to leave etcd, and the last member can't. Afterwards, wait for etcd to sit in "waiting for bootstrap" and run `talosctl bootstrap` once. Kubeconfig and talosconfig keep working because the CA comes from `secrets.yaml`, and Talos re-applies its bootstrap manifests by itself (at the time Flannel, CoreDNS, and kube-proxy; since 2026-10-23 only CoreDNS, see the Cilium rebuild item under "Open items").
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
- Pods that were running when a node shut down end in a final phase (`Completed`, or `Failed` with status `Error` and the message `Pod was terminated in response to imminent node shutdown`) and are replaced by their ReplicaSet, not restarted. The old pods stay around as clutter until deleted: the pod garbage collector only removes terminated pods above `--terminated-pod-gc-threshold` (12500 by default). Clean up with `kubectl delete pods -A --field-selector=status.phase=Failed` (and `=Succeeded` for the `Completed` ones).
- Cilium agent on a restarted node (2026-10-24, `talos-i`, ~18:35 UTC): the agent started before `eno1` had its DHCP address and logged `Failed to serve cilium-health API ... listen tcp 192.168.0.192:4240: bind: cannot assign requested address`. It does not retry that bind, so both nodes reported `Cluster health: 1/2 reachable` (only the probe to `talos-i`'s host `:4240` failed; pod traffic was fine). Fix: delete the agent pod on that node so the DaemonSet recreates it. Read `cilium-health status --verbose` before assuming a network problem.
- `kubectl run -i --rm` can lose the first lines of output: kubectl only attaches after the container has started ("If you don't see a command prompt…"). For a short test, run the pod without `-i`, wait for `Succeeded`, and read `kubectl logs`.
- Helm silently ignores unknown values keys, and a `HelmRelease`'s `spec.values` is free-form in the CRD, so `--dry-run=server --validate=strict` accepts typos there. `persistance:` instead of `persistence:` passed every check and would have left Grafana on an `emptyDir`. Compare value paths against the chart's `values.yaml` (kube-prometheus-stack ships no `values.schema.json`), and after deploying check that the setting reached the rendered object.
- Kustomize `nameReference` (including custom ones from `configurations:`) only rewrites a reference when the referenced object is in the same namespace as the referrer. A `configMapGenerator` output has no namespace unless the generator sets one, so a hashed ConfigMap name was not rewritten in a namespaced `HelmRelease`. Always look at the rendered output (`kubectl kustomize`, `flux build kustomization`), not just whether it builds.
- To inspect a Helm chart from an OCI registry without the `helm` CLI: get an anonymous token from `https://ghcr.io/token?scope=repository:<repo>:pull`, fetch `/v2/<repo>/manifests/<tag>` (Accept `application/vnd.oci.image.manifest.v1+json`), and download the chart layer from `/v2/<repo>/blobs/<digest>`.
- The Red Hat YAML extension for VS Code applies SchemaStore schemas by **filename**: `databases/pg-main/cluster.yaml` gets the RKE "Cluster Configuration" schema and reports `Property apiVersion is not allowed`. Fix with a `# yaml-language-server: $schema=<url>` modeline per file (CRD schemas from `datreeio/CRDs-catalog`) or workspace settings.
- Flux `spec.dependsOn` is a list of objects (`- name: <kustomization>`), not of strings. `- cilium` fails CRD validation (`spec.dependsOn[0] in body must be of type object`), and because every file in `clusters/super-hellthalos/` is applied by the one `flux-system` Kustomization, a single invalid object there blocks the rollout of all of them. `flux build` does not catch it; `kubectl apply --dry-run=server -f <file>` does (2026-10-24).
- **A `dependsOn` does not apply to the rollout that introduces it** (2026-10-24). When a commit adds `dependsOn` to an *existing* Flux Kustomization, that Kustomization reconciles the new revision right away with its old spec, in parallel with `flux-system` applying the new spec. On the Gateway API rollout, `cilium` upgraded the Helm release at 19:51:53, the Gateway API CRDs were created at 19:51:55, and the chart (`gatewayClass.create: auto`) rendered without a `GatewayClass`. The new `cilium-config` Kustomization had the dependency from the start and correctly waited. When an existing component needs something new first, merge the new dependency in a separate PR beforehand.
- Cilium agents and the operator read their ConfigMap only at startup, and the chart does not restart them on a values change by default (`rollOutCiliumPods: false`). A merged and `Ready` HelmRelease therefore does not mean the setting is active: check the pod start times against the release's `lastDeployed`, or the agent's `cilium-dbg status`.
- When copying a Flux Kustomization block, check `path`: two Kustomizations pointing at the same directory both claim the same objects (each stamps its own `kustomize.toolkit.fluxcd.io/name` label) and keep taking them over from each other, and the copy may lack the original's prune protection (caught in review on 2026-10-24).
- YAML that is valid but indented wrongly (e.g. `interval`, `path`, `prune` at column 0 instead of under `spec`) passes the editor and `flux build`; only the server dry run reports it (`strict decoding error: unknown field "interval"`). The same goes for a manifest without `apiVersion` (`apiVersion not set`) (2026-10-24).
- `kubeseal --validate < sealedsecret.yaml` asks the controller whether it can decrypt a SealedSecret (right key, name and namespace match the strict scope) without creating anything. Run it before a PR.
- `CiliumGatewayClassConfig.spec.service.type` (Cilium 1.20) only accepts `LoadBalancer` or `NodePort`, not `ClusterIP` (checked with `kubectl explain` on 2026-10-25). A Cilium Gateway cannot be made cluster-internal through its Service type. To hide it from the LAN, stop announcing its IP instead (L2 policy `serviceSelector`), or restrict `loadBalancerSourceRanges`.
- `kubectl kustomize <dir> | kubectl apply --dry-run=server -f -` on a directory that creates its own Namespace reports `namespaces "<ns>" not found` for every namespaced object: a dry run does not really create the Namespace. Expected, not an error in the manifests.
- `ERROR: cannot execute ... in a read-only transaction` in `psql` means the session is on a hot-standby replica (`SELECT pg_is_in_recovery()` returns `t`). Writes, DDL, and `GRANT`/`REVOKE` must go to the primary (`pg-main-rw`). Schema privileges are per database, so `\c <db>` first.
- When a node's IP changes, check `kubectl get ciliumnodes` against `kubectl get nodes -o wide`. After the `talos-ii` hardware swap on 2026-10-30, `CiliumNode` `talos-ii` kept the old IP `192.168.0.41` while the Node had `192.168.0.193`, so `talos-i` sent VXLAN traffic to the wrong address. Cluster health showed `1/2 reachable` on `talos-i` and `0/2 reachable` on `talos-ii`, and the Postgres replica stopped streaming. Deleting the agent pod on `talos-ii` fixed it: the new agent registered the current IP, and `pg-main-2` went back to `streaming` with 0 bytes of lag.
- `Cluster in healthy state` and both pods `Ready` do not prove that replication works: CNPG reported that throughout the outage above, while `pg_stat_replication` on the primary was empty. Query that view to check streaming.

## Open items / next steps

1. **Prometheus follow-ups** (deployed 2026-10-23): a `PodMonitor` for `pg-main` (CNPG exposes metrics on port 9187), check real memory usage and adjust requests/limits, and an Alertmanager receiver. Reaching Grafana without a port-forward: see item 9.
2. **Cilium follow-ups:**
   - Cluster health reports `2/2 reachable` on both nodes (2026-10-30). If the health listener goes missing after a reboot again, delete the agent pod on that node (`kubectl -n kube-system delete pod -l k8s-app=cilium --field-selector spec.nodeName=<node>`). If it happens after every reboot, find a durable fix (make the agent wait for the node IP, or pin `devices`).
   - **Rebuild ordering:** Talos no longer deploys a CNI, and the Flux controllers need pod networking, so on a fresh cluster Flux cannot install Cilium by itself. For now the manual path is documented in `README.md` ("Rebuild from scratch", 2026-10-24): apply the Gateway API CRDs, then `helm install` Cilium with the same values before `flux bootstrap`, and Flux continues the release (not yet tested). Under consideration: a Talos `KubeInlineManifestConfig` that creates the Cilium objects at bootstrap and lets Flux take ownership. Watch for the private keys the chart renders (`cilium-ca`, `hubble-server-certs`) and `upgrade-k8s` pruning; test on a throwaway `talosctl cluster create` cluster first.
   - A NetworkPolicy for `databases` (only the apps and the CNPG operator may reach Postgres), now that policies are enforced.
   - Optional: Hubble metrics plus a `ServiceMonitor`, Hubble Relay/UI, `bpf.masquerade` and eBPF host routing instead of iptables.
3. **Done 2026-10-25:** tested a restore of `pg-main` from R2.
4. Kubernetes Secrets use Sealed Secrets (decided 2026-10-20). Still open: whether to commit `talos/secrets.yaml` encrypted as well (it is not a Kubernetes object, so Sealed Secrets doesn't fit; SOPS + age would), and a script for the regenerate commands in "Cluster config" (half-run regenerations happened on 2026-10-23).
5. Optional: label `talos-ii` as a worker. On the old laptop, `talos-ii` was unreachable from 2026-10-09 ~21:42 until 2026-10-10 (`NotReady`, Talos API `no route to host`); the cause was not recorded. It was apparently not schedulable again from about 2026-10-11 until 2026-10-20 17:54 UTC (see the `pg-main` lesson). Because `talos-ii` holds a Postgres replica, its reliability matters.
6. Decide which schematic `talos-ii` uses. It runs the default one without `i915` (seen 2026-10-30; when and how that changed was not recorded), while `nodes/talos-ii.yaml` still sets the `i915` installer image, so Git and the node disagree. If `i915` is not needed, delete `schematics/talos-ii.yaml` and remove the installer override from `nodes/talos-ii.yaml`. If it is, run `talosctl upgrade` on `talos-ii` with the `i915` schematic ID.
7. Clean up the `Completed`/`Failed` pods left over from node shutdowns (as of 2026-10-24: `cilium-operator`, `kube-state-metrics`, Grafana and Prometheus Operator pods), and the local branches that are already merged into `main` (`feat/local-path-provisioner`, `feat/cnpg-operator`, `feat/talos-config`, `feat/cnpg-database`, `feat/flux`).
8. Add `# yaml-language-server: $schema=...` modelines to the CRD manifests (CNPG, Flux, SealedSecrets) for editor validation, as a separate small PR.
9. **Expose Grafana** (decision under "Decisions", started 2026-10-24), one PR per step:
   1. LoadBalancer IPs and Gateway API foundations: Gateway API CRDs, Cilium values, `cilium-config`. **Merged 2026-10-24** (`main@96ac6edb`): all Kustomizations `Ready`, pool `lan` with 11 IPs available, but no `GatewayClass` and the new Cilium settings not yet active (see the lesson on the 2026-10-24 rollout). To do: `kubectl -n kube-system rollout restart ds/cilium deploy/cilium-operator`, then `flux reconcile helmrelease cilium -n kube-system --force`, check `kubectl get gatewayclass` (`cilium`, `ACCEPTED True`), then test with a throwaway `type: LoadBalancer` Service. Follow-up PR: `rollOutCiliumPods: true`, `operator.rollOutPods: true`, `envoy.rollOutPods: true`, and `gatewayAPI.gatewayClass.create: "true"` (fail loudly instead of silently skipping the class when the CRDs are missing).
   2. TLS: Cloudflare token SealedSecret, `ClusterIssuer`s, wildcard `Certificate` (see "cert-manager-config" and "gateway"). Staging certificate issued 2026-10-24 (`main@41950886`). Switch to `letsencrypt-prod` in progress; afterwards check that the issuer no longer says `(STAGING)`.
   3. **Done 2026-10-25** (PR #12, `main@af15a0f2`): `Gateway` `main` and the Grafana `HTTPRoute`. The LAN path (`*.k8s.superhellth.com`, pinned `.240`) was dropped in the same PR, see "Decisions".
   4. **Done 2026-10-25** (same PR): `cloudflared`, Access application, DNS record (see "cloudflared" and "State outside Git"). Verified: 4 tunnel connections registered, `https://grafana.superhellth.com` answers with a redirect to the Access login.
   5. **Close the LAN bypass:** label the Gateway's Service through `spec.infrastructure.labels` (check that Cilium copies it onto `cilium-gateway-main`) and exclude that label in the `lan` L2 announcement policy's `serviceSelector` (`NotIn`). Afterwards `curl --resolve grafana.superhellth.com:443:<LB-IP> ...` from the LAN must time out, while the tunnel keeps working through the ClusterIP.
   6. **Check the Access JWT behind the edge**, so a deleted or mistyped Access application does not open Grafana: `originRequest.access` (`required: true`, `teamName: blue-hat-330a`, `audTag`) in `config.yaml`. That puts the rule in Git. Optional later: Grafana `auth.jwt` with the `Cf-Access-Jwt-Assertion` header (single sign-on from Access, and it also covers paths that skip cloudflared).
   7. Record the Access policy (who is allowed, identity provider, session duration) under "State outside Git". Optional: `PodMonitor` for cloudflared's `:2000/metrics`.
   8. Later: Cloudflare objects as code (OpenTofu with the Cloudflare provider, or external-dns for the DNS records).
   - The nodes also get public IPv6 addresses (SLAAC) from the router; confirm the router blocks inbound IPv6.
10. Later: metrics-server (`kubectl top` currently fails with "Metrics API not available").

Done: DHCP reservations for both nodes (renewed for the new `talos-ii` NIC, 2026-10-30); secret files backed up (2026-10-21); GitOps controller chosen and bootstrapped (Flux, 2026-10-22); stacked branches merged into `main`; backups for `pg-main` (Barman Cloud to R2, 2026-10-22); kube-prometheus-stack deployed (2026-10-23); resource requests and limits for sealed-secrets, cert-manager, the Barman Cloud plugin, and local-path-provisioner (2026-10-23); Cilium replaces Flannel and kube-proxy (2026-10-23); Grafana published through Gateway API, Cloudflare Tunnel and Access (2026-10-25).

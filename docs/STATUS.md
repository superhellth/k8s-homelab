# Cluster status

_Last updated: 2026-10-10_

## Summary

Two-node Talos cluster `super-hellthalos`: one control plane that also runs workloads (`talos-i`) and one worker (`talos-ii`, an old laptop). Both nodes report `Ready`. Each node has a dedicated 120 GiB data partition, and local-path-provisioner is installed on top of it as the default StorageClass. It is installed from Kustomize manifests in this repo (`infrastructure/`), and the end-to-end PVC test has passed. The CloudNativePG operator is installed and tested; the first Postgres `Cluster` is next. The Talos config (patches, per-node patches, Image Factory schematics) is in this repo under `talos/`; its secrets are not committed (see "Where things live").

The cluster was rebuilt on 2026-10-09: EPHEMERAL was wiped on both nodes to resize it, and etcd was re-bootstrapped (see "Disk layout" and "Lessons learned").

## Nodes

| | `talos-i` | `talos-ii` |
|---|---|---|
| Role | control plane (workloads allowed, no taint) | worker (ROLES shows `<none>`, see below) |
| IP | `192.168.0.192/24` | `192.168.0.220/24` |
| Gateway/DNS | `192.168.0.1` | `192.168.0.1` |
| DHCP reservation in router | **not yet confirmed** | **not yet confirmed** |
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
- `patch-controlplane.yaml`: `KubeNodeConfig` deleting the `node-role.kubernetes.io/control-plane` taint and the `node.kubernetes.io/exclude-from-external-load-balancers` label
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

Until a GitOps controller is chosen, manifests live in this repo as Kustomize directories and are applied by hand with `kubectl apply -k <dir>`. A GitOps controller can take over the same directories later. Upstream manifests are referenced by URL with a pinned tag, not copied into the repo, and only local patches are committed.

Repo layout (one directory per component, top-level directories ordered by dependency):
```
talos/                   machine config layer, applied with talosctl (not by a GitOps controller)
  patch-all.yaml, patch-controlplane.yaml
  nodes/                 per-node patches
  schematics/            Image Factory schematics
  _out/, secrets.yaml    gitignored
infrastructure/          cluster-wide foundations (storage, operators, CRDs)
  local-path-provisioner/
  cnpg-operator/
databases/               things that use the foundations (planned)
  pg-main/
```

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

### Decisions

- **CNPG runs in this cluster, not a separate one.** A second cluster would double the control-plane overhead and split the two nodes into two single-node clusters. Isolation happens inside the cluster instead: own namespace, Guaranteed QoS (requests = limits) for Postgres pods, optionally a PriorityClass. Flannel does not enforce NetworkPolicies, so network isolation needs a CNI change (e.g. Cilium).
- **Known limits of this setup:** etcd and Postgres share the NVMe on `talos-i` (watch for `slow fdatasync` warnings from etcd). CNPG failover runs through the API server, so if `talos-i` (the only control plane) dies, a replica on `talos-ii` is **not** promoted automatically.
- **Backups:** not decided yet, but they can't rely on volume snapshots, because local-path doesn't support them. Candidates: CNPG Barman Cloud plugin (WAL + base backups to object storage), `pg_dump`, Velero file-level (Kopia).

## Where things live

The Talos files moved from `~/talos` into `talos/` in this repo (2026-10-10). The rule for what gets committed: sources without secrets are committed, generated files are not, and secrets are committed only once they are encrypted.

- Committed: `patch-all.yaml`, `patch-controlplane.yaml`, `nodes/*.yaml`, `schematics/*.yaml`
- Gitignored, **plaintext, only copy is in the working tree**:
  - `talos/secrets.yaml`: cluster CAs and tokens. Losing it means losing admin access. It is not backed up yet, and `git clean -fdx` would delete it.
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
- Talos enables Pod Security Admission cluster-wide with `enforce: baseline`, `warn: restricted`, `audit: restricted`, and exempts `kube-system` (`KubeAdmissionControlConfig` `PodSecurity`). Workloads that need `hostPath` or privileged access need a namespace label like `pod-security.kubernetes.io/enforce: privileged`.

## Lessons learned (Kubernetes / tooling)

- Kustomize patches are matched by apiVersion + kind + name (+ namespace). A typo there (e.g. `storage.k82.io`) fails the render with `no resource matches strategic merge patch`, which is a useful safety net. Kustomize does not look inside string values such as `config.json`, so JSON errors only show up when the program reads the file.
- `kubectl kustomize` re-serialises objects with fields sorted alphabetically, so `kind:` can appear after a long `data:` block. Key order has no meaning in YAML.
- `kubectl get a,b,c -w` fails with `you may only specify a single resource type`. Watch one type at a time, use `kubectl get events -A -w`, or poll with `watch -n1 '...'`.
- CNPG's manifests must be applied with `kubectl apply --server-side`. Its CRDs exceed the 256 KiB annotation limit that client-side apply runs into.
- The CNPG operator pod restarted once on first install (exit code 143 = SIGTERM, about 27 s after start, nothing wrong in the logs). Most likely cause: the startup probe (6 × 5 s) checks the webhook port, and the certificate Secret the operator had just created had not been synced into the pod yet (it is mounted `optional: true`, and the kubelet only syncs Secret volumes periodically). The second start worked. This is harmless on first install.
- `kubectl rollout status deploy/<name>` waits until the Deployment's `.status` shows all replicas updated and Ready, and exits non-zero after `progressDeadlineSeconds` (600 s by default). Use it between applying an operator and applying its custom resources.
- Heredocs: the end marker (`EOF`) must start at column 0, and `<<'EOF'` (quoted) stops the local shell from expanding `$(...)` inside the manifest.

## Open items / next steps

1. Confirm DHCP reservations for `192.168.0.192` and `192.168.0.220` in the router.
2. Back up `talos/secrets.yaml` and `~/.talos/config` somewhere safe outside WSL. **Urgent**: the only copy of `secrets.yaml` is gitignored in the working tree.
3. Decide on secrets handling (SOPS + age is the leading option, because Flux decrypts it natively), then commit `secrets.yaml` encrypted. Also add a script for the regenerate commands in "Cluster config".
4. Choose a GitOps controller (Flux or Argo CD).
5. Optional: label `talos-ii` as a worker, and think about laptop specifics (lid-close behavior, battery as a built-in UPS, Wi-Fi vs. Ethernet). `talos-ii` was unreachable from 2026-10-09 ~21:42 until 2026-10-10 (`NotReady`, Talos API `no route to host`); the cause was not recorded.
6. Merge the stacked branches into `main` in order: `feat/local-path-provisioner`, then `feat/cnpg-operator`, then `feat/talos-config`.
7. First `Cluster` `pg-main` in namespace `databases` (`databases/pg-main/`) with `instances: 1`, `storageClass: local-path` and Guaranteed QoS (size it for the 8 GB on `talos-ii`). Next, choose and set up backups (see "Decisions"), then scale to `instances: 2` with required pod anti-affinity.
8. Decide whether `talos-ii` keeps `i915`. If not, delete `schematics/talos-ii.yaml`, remove the installer override from `nodes/talos-ii.yaml`, and run `talosctl upgrade` on `talos-ii` with the default schematic ID.
9. Later: ingress/load balancer, CNI choice (keep Flannel or move to Cilium; Flannel does not enforce NetworkPolicies). The nodes also get public IPv6 addresses (SLAAC) from the router, which matters once services are exposed.

# Cluster status

_Last updated: 2026-10-09_

## Summary

Two-node Talos cluster `super-hellthalos`: one control plane that also runs workloads (`talos-i`) and one worker (`talos-ii`, an old laptop). Both nodes report `Ready`. Each node has a dedicated 120 GiB data partition for local-path-provisioner, but the provisioner itself is not installed yet, so PVCs cannot bind. Nothing is deployed beyond the defaults yet, and none of the Talos config is in this repo yet.

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
| Image Factory schematic | `2d61dd07b20062062ea671b4d01873506103b67c0f7a4c3fb6cf4ee85585dcb8` (installer: `factory.talos.dev/metal-installer/<schematic>:v1.14.2`) |

Extensions in the schematic: `intel-ucode` (20260812), as reported by `talosctl get extensions`. The schematic YAML itself is not saved yet.

**Unresolved:** `nodes/talos-ii.yaml` overrides the installer image with a different schematic, `4b3cd373a192c8469e859b7a0cfbed3ecc3577c4a2d346a37b0aeff9cd17cdb0`. Which schematic `talos-ii` should use, and what extensions it contains, still needs to be recorded.

## Cluster config

- Cluster name `super-hellthalos`, Kubernetes endpoint `https://192.168.0.192:6443`
- Talos defaults are used for everything except the patches below: Flannel CNI, kube-proxy, CoreDNS.
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

The user volume is mounted under `/var/mnt/`. The exact mount path, `/var/mnt/local-path-provisioner` or `/var/mnt/u-local-path-provisioner`, is still to be confirmed with `talosctl get mountstatus`. local-path-provisioner needs the exact path.

## Where things live

All Talos files are currently in `~/talos` in WSL, **outside this repo and unencrypted**:
- `secrets.yaml`: cluster CAs and tokens. Losing it means losing admin access, so back it up.
- `_out/controlplane.yaml`, `_out/worker.yaml`: base machine configs (contain secrets)
- `_out/talos-i.yaml`, `_out/talos-ii.yaml`: final per-node machine configs (contain secrets)
- `_out/talosconfig`: Talos API client credentials (`TALOSCONFIG` env var points here)
- `patch-all.yaml`, `patch-controlplane.yaml`, `nodes/*.yaml`: safe to commit
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

## Open items / next steps

1. Confirm DHCP reservations for `192.168.0.192` and `192.168.0.220` in the router.
2. Back up `secrets.yaml` and `talosconfig` somewhere safe outside WSL.
3. Decide on secrets handling (e.g. SOPS + age) and the repo layout, then move patches (including `nodes/`) and the Image Factory schematic into the repo with a `.gitignore` for plaintext secrets.
4. Choose a GitOps controller (Flux or Argo CD).
5. Optional: label `talos-ii` as a worker, and think about laptop specifics (lid-close behavior, battery as a built-in UPS, Wi-Fi vs. Ethernet).
6. Install local-path-provisioner, pointed at the user volume mount path, with `volumeBindingMode: WaitForFirstConsumer`. First decide how to install things before GitOps exists: plain manifests, Helm, or GitOps first.
7. Then: CloudNativePG operator, and a first `Cluster` with `instances: 1` plus backups to object storage. Scale to `instances: 2` with required pod anti-affinity later.
8. Record the `talos-ii` installer schematic (see "Versions").
9. Later: ingress/load balancer, CNI choice (keep Flannel or move to Cilium). The nodes also get public IPv6 addresses (SLAAC) from the router, which matters once services are exposed.

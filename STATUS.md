# Cluster status

_Last updated: 2026-10-05_

## Summary

Single-node Talos cluster `super-hellthalos` is installed, bootstrapped, and the node reports `Ready`. Nothing is deployed on top of the defaults yet, and none of the Talos config is in this repo yet.

## Node

| | |
|---|---|
| Role | control plane (workloads allowed) |
| IP | `192.168.0.192/24`, gateway/DNS `192.168.0.1` (DHCP reservation in router: **not yet confirmed**) |
| Hardware | Intel i5 (6 logical CPUs), 16 GB RAM |
| Install disk | `/dev/nvme0n1`: WDC PC SN730 256 GB, serial `19422B800421` |
| Previous OS | Windows (wiped by the Talos install) |

## Versions

| Component | Version |
|---|---|
| Talos | v1.14.2 |
| Kubernetes | v1.37.1 |
| kubectl (WSL) | v1.37.1 |
| Image Factory schematic | `2d61dd07b20062062ea671b4d01873506103b67c0f7a4c3fb6cf4ee85585dcb8` (installer: `factory.talos.dev/metal-installer/<schematic>:v1.14.2`) |

The chosen extensions behind the schematic are not recorded yet; save the schematic YAML from Image Factory.

## Cluster config

- Cluster name `super-hellthalos`, Kubernetes endpoint `https://192.168.0.192:6443`
- Talos defaults are used for everything except the patches below: Flannel CNI, kube-proxy, CoreDNS.
- Generated with:
  ```bash
  talosctl gen config super-hellthalos https://192.168.0.192:6443 \
    --with-secrets secrets.yaml \
    --config-patch @patch-all.yaml \
    --config-patch-control-plane @patch-controlplane.yaml \
    --output-dir _out
  ```
- `patch-all.yaml`: `UnattendedInstallConfig` (Factory installer image, disk selector `disk.dev_path == "/dev/nvme0n1"`)
- `patch-controlplane.yaml`: `KubeNodeConfig` deleting the `node-role.kubernetes.io/control-plane` taint and the `node.kubernetes.io/exclude-from-external-load-balancers` label

## Where things live

All Talos files are currently in `~/talos` in WSL, **outside this repo and unencrypted**:
- `secrets.yaml`: cluster CAs and tokens. Losing it means losing admin access, so back it up.
- `_out/controlplane.yaml`, `_out/worker.yaml`: full machine configs (contain secrets)
- `_out/talosconfig`: Talos API client credentials (`TALOSCONFIG` env var points here)
- `patch-all.yaml`, `patch-controlplane.yaml`: safe to commit
- kubeconfig merged into `~/.kube/config`

## Lessons learned (Talos 1.14)

- 1.14 generates multi-document configs. `.machine.install` is replaced by `UnattendedInstallConfig`, and node labels, taints, and `allowSchedulingOnControlPlanes` are replaced by `KubeNodeConfig`. Old-style fields in patches cause "already set in v1alpha1 config" errors.
- `--config-patch` applies to **both** controlplane and worker configs. A `$patch: delete` on a key missing from the worker config fails with `lookup failed`, so role-specific patches go in `--config-patch-control-plane` / `--config-patch-worker`.
- The generated `UnattendedInstallConfig` sets `wipe: false` explicitly. The install still repartitions the disk.

## Open items / next steps

1. Confirm the DHCP reservation for `192.168.0.192` in the router.
2. Back up `secrets.yaml` and `talosconfig` somewhere safe outside WSL.
3. Decide on secrets handling (e.g. SOPS + age) and the repo layout, then move patches and the Image Factory schematic into the repo with a `.gitignore` for plaintext secrets.
4. Choose a GitOps controller (Flux or Argo CD).
5. Later: storage (local-path vs. Longhorn etc.), ingress/load balancer, CNI choice (keep Flannel or move to Cilium).

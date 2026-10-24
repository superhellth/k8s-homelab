# Nico's Homelab k8s GitOps Repo

## GitOps

Main branch is protected. Changes can only be made via PRs.

## Rebuild from scratch

How to get from blank machines to the full cluster. Talos does not deploy a CNI (Flannel and kube-proxy are disabled), and the Flux controllers need pod networking to run, so Cilium has to be installed once by hand before Flux. After that, Flux manages everything, including Cilium.

### Prerequisites

- `talosctl` v1.14.2, `kubectl`, `helm`, `flux` v2.9.6. Use exactly this Flux CLI version: `flux bootstrap` commits the component manifests of the CLI's version, and `main` is protected, so a version change would fail the push.
- The gitignored secret files, restored from the backup:
  - `talos/secrets.yaml` (cluster CAs and tokens)
  - `infrastructure/sealed-secrets/sealed-secrets-keys.yaml` (Sealed Secrets private key)
- Both nodes booted from the Talos installer ISO built from their Image Factory schematic (`talos/schematics/default.yaml` for `talos-i`, `talos/schematics/talos-ii.yaml` for `talos-ii`). They come up in maintenance mode with their DHCP-reserved IPs (`talos-i` `192.168.0.192`, `talos-ii` `192.168.0.220`).

### 1. Generate the machine configs

Run in `talos/`:

```bash
talosctl gen config super-hellthalos https://192.168.0.192:6443 \
  --with-secrets secrets.yaml \
  --config-patch @patch-all.yaml \
  --config-patch-control-plane @patch-controlplane.yaml \
  --output-dir _out --force
talosctl machineconfig patch _out/controlplane.yaml --patch @nodes/talos-i.yaml  -o _out/talos-i.yaml
talosctl machineconfig patch _out/worker.yaml       --patch @nodes/talos-ii.yaml -o _out/talos-ii.yaml
```

### 2. Install Talos and bootstrap Kubernetes

```bash
talosctl apply-config --insecure -n 192.168.0.192 -f _out/talos-i.yaml
talosctl apply-config --insecure -n 192.168.0.220 -f _out/talos-ii.yaml
# Both nodes install to disk and reboot.

talosctl --talosconfig _out/talosconfig config endpoint 192.168.0.192
talosctl --talosconfig _out/talosconfig config node 192.168.0.192
talosctl config merge _out/talosconfig      # into ~/.talos/config

talosctl -n 192.168.0.192 bootstrap         # once, on the control plane only
talosctl -n 192.168.0.192 kubeconfig        # merges into ~/.kube/config
```

`kubectl get nodes` shows both nodes as `NotReady` and the CoreDNS pods as `Pending`. This is expected: there is no CNI yet.

### 3. Install Cilium by hand

Run from the repo root. Apply the Gateway API CRDs first: Cilium only enables Gateway API (and creates `GatewayClass cilium`) if they exist when it is installed. Then install the chart version from `infrastructure/cilium/ocirepository.yaml` with the values Flux uses:

```bash
kubectl apply --server-side -k infrastructure/gateway-api-crds
helm install cilium oci://quay.io/cilium/charts/cilium --version 1.20.4 \
  -n kube-system -f infrastructure/cilium/values.yaml
kubectl -n kube-system rollout status ds/cilium
kubectl get nodes                           # both Ready
```

The release must be called `cilium` in `kube-system`: that is the name the Flux `HelmRelease` uses, so Flux continues with this release instead of creating a second one.

### 4. Restore the Sealed Secrets key

Before Flux deploys the controller, otherwise the controller generates a new key and none of the committed SealedSecrets can be decrypted:

```bash
kubectl apply -f infrastructure/sealed-secrets/sealed-secrets-keys.yaml
```

### 5. Bootstrap Flux

```bash
flux bootstrap git \
  --url=ssh://git@github.com/superhellth/k8s-homelab \
  --branch=main \
  --path=clusters/super-hellthalos
```

Flux generates a new SSH deploy key and asks you to add its public key to the GitHub repo (with write access). Then it installs the controllers and applies `clusters/super-hellthalos/`. The manifests in `flux-system/` are already in Git, so no commit should be needed.

Flux reconciles the rest in `dependsOn` order: the Gateway API CRDs (takes them over), Cilium (takes over the Helm release), `cilium-config` (LoadBalancer IP pool and L2 announcement policy), sealed-secrets, local-path-provisioner, cert-manager, CNPG operator, Barman Cloud plugin, kube-prometheus-stack, then `pg-main`. Follow it with:

```bash
flux get kustomizations -A --watch
flux get helmreleases -A
kubectl get pods -A
```

### 6. Check

- `kubectl -n kube-system exec ds/cilium -c cilium-agent -- cilium-dbg status`: `KubeProxyReplacement: True`, `Cluster health: 2/2 reachable`
- `flux get helmreleases -n kube-system cilium`: `Ready`, and its revision is a Flux upgrade of the manually installed release
- `kubectl get gatewayclass`: `cilium`, `ACCEPTED True`. If it is missing, the CRDs were not there when Cilium started: restart the operator (`kubectl -n kube-system rollout restart deploy/cilium-operator`)
- `kubectl get cluster -n databases pg-main`: `Cluster in healthy state`

### Caveats

- **`pg-main` comes back empty.** The `Cluster` bootstraps with `initdb`, not from the R2 backups. Restoring from R2 has not been tested yet (see `docs/STATUS.md`). A new cluster also archives WAL to the same bucket, so decide on the restore (or a new backup path) before `pg-main` is created, for example by suspending its Kustomization: `flux suspend kustomization pg-main`.
- Flux continuing a release that `helm install` created by hand has not been tested on this cluster yet. If `flux get helmreleases` shows an error for `cilium`, do not `helm uninstall` it (that cuts off all pod networking); look at `kubectl -n kube-system describe helmrelease cilium` first.
- Details, the reasoning behind each step, and lessons learned are in `docs/STATUS.md`.

## Postgres

The cluster runs one pg-main cnpg cluster.

Done for security reasons:

revoke all on schema public from public;
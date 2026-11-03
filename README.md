# Nico's Homelab k8s GitOps Repo

My home Kubernetes cluster, run entirely through GitOps from this repo.

## Why

Mainly to learn Kubernetes, Helm, GitOps with Flux and Argo CD, and Docker, using a real cluster instead of tutorials.

It also hosts my own apps:

- **[Wegweiser](https://github.com/superhellth/open-alps)** (`wegweiser.superhellth.com`)
- **[Cubing](https://github.com/superhellth/cubing)** (`cubing.superhellth.com`)
- **[wg-app](https://github.com/superhellth/wg-app)** (coming soon)

## Hardware

Two bare-metal nodes running [Talos Linux](https://www.talos.dev/):

| Node | Role | Machine | CPU | RAM |
|---|---|---|---|---|
| `talos-i` | control plane (also runs workloads) | Lenovo ThinkCentre M720q | i5 9th gen, 6 cores | 16 GB |
| `talos-ii` | worker | Lenovo ThinkCentre M720q | i5 9th gen, 6 cores | 16 GB |

One old Acer laptop that used to be `talos-ii` is kept as a backup node.

## Infrastructure stack

- **[Cilium](https://cilium.io/):** CNI and kube-proxy replacement. It also provides Gateway API ingress and LoadBalancer IPs.
- **[CloudNativePG](https://cloudnative-pg.io/):** a Postgres cluster with one instance per node, continuously backed up to Cloudflare R2.
- **[Cloudflare](https://www.cloudflare.com/):** a Cloudflare Tunnel publishes services without opening ports, and Cloudflare Access protects them. Cloudflare also handles DNS and the DNS-01 challenges for the wildcard certificate.

These are supported by cert-manager, Sealed Secrets, local-path-provisioner no NAS yet :( and kube-prometheus-stack.

## Git workflow

- `main` is protected: every change goes through a pull request.
- [Flux](https://fluxcd.io/) and [Argo CD](https://argo-cd.readthedocs.io/) both watch `main` and reconcile the cluster to whatever is merged. Flux bootstraps the cluster and runs the infrastructure. Argo CD runs the apps and is taking over more components over time.

## Docs

- [docs/SETUP.md](docs/SETUP.md): rebuilding the cluster from blank machines.
- [docs/STATUS.md](docs/STATUS.md): current state, decisions, and lessons learned (mainly for them agents though).

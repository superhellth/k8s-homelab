# Nico's Homelab k8s GitOps Repo

## GitOps

Main branch is protected. Changes can only be made via PRs.

## Rebuild from scratch

See [docs/SETUP.md](docs/SETUP.md): from blank machines to the full cluster (Talos, Cilium by hand, Sealed Secrets key, Flux bootstrap).

## Postgres

The cluster runs one pg-main cnpg cluster.

Done for security reasons:

revoke all on schema public from public;
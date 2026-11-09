# Argo CD migration roadmap

High-level plan for moving GitOps from Flux to Argo CD gradually, one component at a time. Primary goal is learning Argo CD. Written 2026-10-27; **since 2026-11-09 the goal is a full switch** (Flux uninstalled, see phase 7). Last updated 2026-11-09.

## Current state (2026-11-09)

- Done: phase 0 (except admin password hardening), phase 1. `cubing` and `open-alps` were deployed directly under Argo.
- Phase 2 (`kube-prometheus-stack`): handed over (PR #45), `Synced`/`Healthy`, etcd dashboard fixed. Left: cleanup of the Flux leftovers. Next after that: unblock the `argocd` HelmRelease (`docs/STATUS.md` open item 11), then phase 3.
- UI at `https://argocd.superhellth.com` behind Cloudflare Access since 2026-11-09. Before that, the route was live for a short time without an Access application (UI and API public, only the `admin` password in front). Lesson: Access application first, DNS record last, and check with `curl -sI` (expect a `302` to `blue-hat-330a.cloudflareaccess.com`). `argocd-initial-admin-secret` still exists.

## Principles

- **Migrate from the leaves inward.** Flux `dependsOn` can only reference Flux Kustomizations: moving something that a remaining Flux Kustomization depends on breaks that Kustomization ("dependency not found"). Only move components no remaining Flux Kustomization depends on.
- **Argo apps may implicitly depend on Flux-managed things.** No declared dependency needed: on the live cluster they already exist, on a rebuild Argo retries the sync with backoff until CRDs/Secrets show up.
- **One owner per object.** Never let both tools manage the same resource. Keep the ownership table in `docs/STATUS.md` current in every PR.
- **Leave Flux in charge of Argo CD itself** until the final phase.

Current dependency graph (Flux `dependsOn`, Flux-owned components only):

```
pg-main ──► cnpg-barman-plugin ──► cnpg-operator
   │                └────────────► cert-manager ◄── cert-manager-config ◄── gateway ◄── cloudflared
   └──► sealed-secrets, local-path-provisioner
kube-prometheus-stack ──► sealed-secrets, local-path-provisioner, gateway-api-crds   (handover in progress)
cilium-config ──► cilium ──► gateway-api-crds
argocd-config ──► argocd
```

Owned by Argo CD: `wg-app` (implicitly needs `pg-main`), `cubing`, `open-alps`.

## Layout

- `infrastructure/argocd-config/` (applied by Flux): `AppProject`s `apps` and `infrastructure`, root `Application`.
- The root syncs `argocd/` with `directory.recurse: true` (since #42): `argocd/apps/` holds workload `Application`s, `argocd/infrastructure/` holds infrastructure `Application`s. The root has `prune: true`: a wrong path or empty render would delete the child `Application`s (their workloads survive, no finalizer). Check `argocd app diff root` after every change to the root.
- Helm components: the plain manifests (Namespace, SealedSecret, HTTPRoute, ...) live in a `manifests/` subdirectory next to `values.yaml`. During the handover, Flux's `kustomization.yaml` references `manifests/...`; Argo applies the directory as a plain source.

## Handover recipe (per component)

Deleting a Flux Kustomization with `prune: true` deletes everything it manages, and deleting a `HelmRelease` runs `helm uninstall`. So every handover takes two PRs:

1. **PR A:** set `prune: false` on the Flux Kustomization (for HelmReleases additionally `spec.suspend: true`; verify against the Flux docs that a suspended HelmRelease is not uninstalled on deletion). Add the Argo `Application` with auto-sync **off**. Check `argocd app diff`: should be (nearly) empty.
2. **PR B:** delete the Flux Kustomization (resources are orphaned, keep running). Enable auto-sync on the `Application`; Argo adopts the resources. Clean up leftovers (Flux labels, `sh.helm.release.v1.*` secrets for Helm charts, orphaned `HelmRelease`/`OCIRepository`/values ConfigMaps).

Prune protection translates from `kustomize.toolkit.fluxcd.io/prune: disabled` to `argocd.argoproj.io/sync-options: Prune=false,Delete=false`. Also decide per `Application` whether it gets the `resources-finalizer.argocd.argoproj.io` finalizer (deleting the Application then cascades to its resources).

Checks before opening a PR:

- `kubectl kustomize <dir>` for every directory Flux builds (files not listed under `resources:` are silently ignored).
- `kubectl apply --dry-run=server --validate=strict -f <Application or AppProject>`: unknown fields (`valuesFiles`, `spec.directory`) are otherwise dropped silently by a normal apply.
- New namespaces in the project's `destinations`, new chart sources in `sourceRepos` (must match the `repoURL` exactly, including `oci://`).

Expected diff noise when Argo adopts a Flux Helm release:

- `argocd.argoproj.io/tracking-id` added to every resource (annotation-based tracking): this is the adoption itself.
- `chart`/`app.kubernetes.io/version` labels lose a `_<digest>` suffix: helm-controller appends the OCI digest as semver build metadata, Argo renders the plain version. Metadata only, but where the labels are in a pod template, those pods restart once.
- API-server defaults inside atomic lists (e.g. Gateway API `HTTPRoute`: `group`, `kind`, `weight`, `matches`) stay `OutOfSync` even with server-side diff: write them into the manifest before PR B.
- Objects Argo excludes by default (`Endpoints`, `EndpointSlice`, `Lease`, ... in `resource.exclusions`) show up as `ExcludedResourceWarning` and are never managed. If a chart renders one, decide how to handle it before PR B.

CLI: `argocd ... --core` uses the kubeconfig directly (no login, no Argo RBAC) and needs `argocd` as the kube namespace (separate kube context `argocd`, use `--kube-context argocd`). Since `server.insecure: true`, a login through a port-forward needs `--plaintext`.

## Phase 0: prerequisites and Argo CD install

- [x] Merge `refactor/cert-manager-to-helm` first, so the two changes don't get mixed up
- [x] Decide install method: Helm chart via `HelmRelease` (first preference was upstream `install.yaml` via kustomize)
- [x] Flux Kustomization `argocd` → `infrastructure/argocd/` (pinned version, resource requests/limits, RAM is tight)
- [x] ~~Separate read-only deploy key for the repo as a SealedSecret~~ Not needed: the repo is public, Argo reads it over HTTPS without credentials. Add a key if the repo goes private
- [x] Root `Application` (app-of-apps) applied by Flux from `infrastructure/argocd-config/` (Flux Kustomization `argocd-config`, `dependsOn: argocd`), pointing at `argocd/` (outside `clusters/super-hellthalos/`, which Flux applies entirely). One hand-written `Application` per file. No ApplicationSet yet
- [x] `AppProject`s instead of `default`: `infrastructure` (cluster-scoped resources allowed; destinations `argocd`, `monitoring`, `kube-system`) and `apps` (namespaced resources plus `Namespace` only, app namespaces)
- [x] Add an ownership table to `docs/STATUS.md`
- [x] UI route (PRs #40, #41): `server.insecure: true` (the Gateway terminates TLS, plain HTTP to the backend), `HTTPRoute` `argocd-server` in `infrastructure/argocd/`, cloudflared ingress rule, DNS record
- [x] Cloudflare Access application for `argocd.superhellth.com` (2026-11-09, verified: `302` to the Access login for UI and `/api/version`)
- [ ] Record the Access application and the `argocd` DNS record under "State outside Git" in `docs/STATUS.md`
- [ ] Change the `admin` password, delete `argocd-initial-admin-secret`
- [ ] Later: SSO through Cloudflare Access as OIDC provider (`oidc.config`), then disable `admin`
- [ ] Close the LAN bypass of the Gateway (`docs/STATUS.md` open item 9.5): more urgent now that Argo is behind it

## Phase 1: `wg-app` (done)

- [x] Hand over `wg-app`
- [x] `cubing`, `open-alps` deployed directly as Argo `Application`s (never under Flux)
- [ ] Image automation (wg-app roadmap phase 4): use Argo CD Image Updater instead of Flux image automation

## Phase 2: `kube-prometheus-stack` (in progress)

First Helm chart. Learn: how Argo renders Helm (`helm template`, no release secret, no `lookup`, Helm hooks become Argo hooks), `ServerSideApply=true` for the large CRDs, `ignoreDifferences`.

- [x] Directory layout: multi-source `Application` `argocd/infrastructure/kube-prometheus-stack.yaml`. Chart via Argo's native OCI source (`repoURL: oci://ghcr.io/prometheus-community/charts/kube-prometheus-stack`, `path: .`), no repository Secret needed for a public registry; `releaseName: kube-prometheus-stack` (resource names derive from it). Second source: this repo with `ref: values` (for `$values/.../values.yaml`) and `path: .../manifests`
- [x] `ignoreDifferences` for the webhooks' `caBundle` (patched in by the admission hook Job), `ServerSideApply=true`, `RespectIgnoreDifferences=true`
- [x] Namespace `monitoring` with `Prune=false,Delete=false`
- [x] PR A (#42): HelmRelease suspended, Flux Kustomization `prune: false`, `Application` with manual sync
- [x] Diff checked (2026-11-09): only tracking annotations and the chart version labels; `prometheus-operator` restarts once (labels in its pod template); CRDs unchanged
- [x] etcd `Endpoints` object (`kube-prometheus-stack-kube-etcd` in `kube-system`, rendered from `kubeEtcd.endpoints`): Argo excludes `Endpoints` by default, so after PR B nobody would manage it and a rebuild would not create it. Decided 2026-11-09: no `resource.exclusions` change in Argo; etcd is scraped through a `ScrapeConfig` with a static target instead (`manifests/scrapeconfig-etcd.yaml`, Argo only, not in Flux's `kustomization.yaml`). `kubeEtcd.service` and `kubeEtcd.serviceMonitor` disabled, `kubeEtcd.enabled` stays `true` (gates the etcd rules and dashboard)
- [x] Manual sync before PR B, done 2026-11-09 (`ScrapeConfig` created 15:26 UTC) (**required** since the `ScrapeConfig` change: the HelmRelease is suspended, so a values change only goes live through Argo). Check `argocd app diff` first (expected: tracking annotations, chart version labels, new `ScrapeConfig`). Watch the adoption and the hooks (PreSync createSecret, PostSync patchWebhook)
- [x] After the sync: etcd target up in Prometheus (`count by (job) (up{job=~".*etcd.*"})`, job `kube-prometheus-stack-kube-etcd`), then delete the old `ServiceMonitor` `kube-prometheus-stack-kube-etcd` in `monitoring` and the `Service` of the same name in `kube-system` (its `Endpoints` go with it). Argo never tracked them, so it will not prune them. Target up and old objects gone (checked 2026-11-09)
- [x] Etcd dashboard: empty after the switch because the old job label was `kube-etcd` (from the Service's `jobLabel`), not `kube-prometheus-stack-kube-etcd`. Fixed 2026-11-09 with `jobName: kube-etcd` in the `ScrapeConfig` (`db59f4e`)
- [x] `HTTPRoute` `grafana` permanently `OutOfSync` (self-heal loop): Gateway API defaults written into the manifest (`09b476d`, lesson in `docs/STATUS.md`)
- [x] PR B (#45, merged 2026-11-09): remove the Flux Kustomization from `monitoring.yaml`, delete `helmrelease.yaml`, `ocirepository.yaml`, `kustomization.yaml`, `kustomizeconfig.yaml`; `automated` (prune, selfHeal) plus `retry` on the `Application`; ownership table
- [ ] Cleanup: orphaned `HelmRelease` (check it is still suspended before deleting), `OCIRepository`, values ConfigMap(s), `sh.helm.release.v1.kube-prometheus-stack.*` Secrets, Flux labels. State 2026-11-09: all still present (HelmRelease suspended, 2 values ConfigMaps, release Secrets `v1`–`v5`, `kustomize.toolkit.fluxcd.io/*` labels on e.g. the `HTTPRoute`, `helm.toolkit.fluxcd.io/*` on the chart objects)
- [ ] Verify: Prometheus/Grafana pods not restarted, same PVCs, Grafana reachable, app `Synced`/`Healthy`. 2026-11-09: pods and PVCs unchanged, `Synced`/`Healthy` (after the `HTTPRoute` fix). Open: tick once the cleanup is done
- [ ] Later, separate PR: admission webhook certs via cert-manager (`prometheusOperator.admissionWebhooks.certManager.enabled`, removes the hook Jobs)

## Phase 3: `cloudflared`, `cilium-config`

Simple leaves. Learn: sync waves once several apps exist.

- [ ] Hand over `cloudflared`
- [ ] Hand over `cilium-config`

## Phase 4: `pg-main`

High stakes: state not in Git. Learn: Argo prune/delete protection, Application finalizers.

- [ ] Fresh base backup to R2 before starting
- [ ] `Prune=false,Delete=false` on `Cluster` and `Namespace`, no cascading finalizer on the `Application`
- [ ] Hand over, verify the cluster is untouched (`kubectl -n databases get cluster pg-main`, primary unchanged)

## Phase 5: base layer

`gateway`, `cert-manager-config`, `cert-manager`, `cnpg-barman-plugin`, `cnpg-operator`, `local-path-provisioner`, `gateway-api-crds`, `sealed-secrets` (roughly in this order). Learn: translating the `dependsOn` graph into sync waves; a custom Lua health check for `argoproj.io/Application` so that waves actually wait for child apps to be healthy.

- [ ] Wave layout derived from the dependency graph
- [ ] Application health check configured in `argocd-cm`
- [ ] Hand over components one by one

## Phase 6: ApplicationSet for `apps/`

Workloads under `apps/` are uniform (namespace, plain manifests, auto-sync), so "add a directory = deploy" fits there; infrastructure stays explicit. Learn: ApplicationSet generators and templates. Independent of the other phases: with three identical app `Application`s it can be done any time.

- [ ] `ApplicationSet` with a git directory generator over `apps/*`, project `apps`
- [ ] Replace the hand-written `Application`s (`wg-app`, `cubing`, `open-alps`): check how the ApplicationSet controller treats an existing Application with the same name; if unclear, delete the old ones without cascading (no finalizer) first
- [ ] Document in `.claude/CLAUDE.md` how to add a new app

## Phase 7: all Argo (decided 2026-11-09)

Argo manages everything including itself and Cilium; Flux is uninstalled. (The alternative was Flux as a thin bootstrap layer for Cilium, sealed-secrets and Argo CD.)

- [ ] Cilium: fix the Hubble certs first (`genCA` re-renders on every sync → permanent `OutOfSync`): `hubble.tls.auto.method: cronJob` or `certmanager`. Then hand over with `Prune=false,Delete=false` on everything (deleting the CNI cuts every pod off the network)
- [ ] Argo CD manages itself: `Application`s for the Argo CD chart and for `argocd-config` (projects, root). Document a manual recovery path (`helm upgrade` with the same values) for when Argo breaks itself
- [ ] Remove Flux: check nothing still carries `kustomize.toolkit.fluxcd.io/name` or `helm.toolkit.fluxcd.io/name` labels, `flux uninstall` (removes finalizers before deleting the CRDs, workloads keep running), delete `clusters/super-hellthalos/`, check the NetworkPolicies Flux shipped in `flux-system`
- [ ] Rebuild procedure in `docs/STATUS.md`/`docs/SETUP.md` updated and ideally tested on a throwaway `talosctl cluster create` cluster: Talos → sealed-secrets key → Gateway API CRDs + Cilium (`helm install` or Talos inline manifest) → `helm install` Argo CD → apply the root → Argo adopts everything
- [ ] `.claude/CLAUDE.md` updated (commands, prune protection conventions, repo layout)

## Costs to keep in mind

- Two tools and two mental models during the transition
- ~0.5–1 GB extra RAM for Argo CD (repo-server spikes when rendering large charts)
- Rebuild story is more complex while both are in use: document it
- Argo has no hard ordering between apps like Flux's `dependsOn`: waves plus the health check come close, but a rebuild converges through retries (noisy first sync)

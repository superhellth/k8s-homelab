# Argo CD migration roadmap

High-level plan for moving GitOps from Flux to Argo CD gradually, one component at a time. Primary goal is learning Argo CD; a full switch is optional. Written 2026-10-27.

## Principles

- **Migrate from the leaves inward.** Flux `dependsOn` can only reference Flux Kustomizations: moving something that a remaining Flux Kustomization depends on breaks that Kustomization ("dependency not found"). Only move components no remaining Flux Kustomization depends on.
- **Argo apps may implicitly depend on Flux-managed things.** No declared dependency needed: on the live cluster they already exist, on a rebuild Argo retries the sync with backoff until CRDs/Secrets show up.
- **One owner per object.** Never let both tools manage the same resource. Keep the ownership table below current in every PR.
- **Leave Flux in charge of Argo CD itself** until the final phase.

Current dependency graph (Flux `dependsOn`):

```
wg-app ──► pg-main ──► cnpg-barman-plugin ──► cnpg-operator
   │          │                └────────────► cert-manager ◄── cert-manager-config ◄── gateway ◄── cloudflared
   └──────────┴──► sealed-secrets, local-path-provisioner, gateway-api-crds
kube-prometheus-stack ──► sealed-secrets, local-path-provisioner, gateway-api-crds
cilium-config ──► cilium ──► gateway-api-crds
```

## Handover recipe (per component)

Deleting a Flux Kustomization with `prune: true` deletes everything it manages, and deleting a `HelmRelease` runs `helm uninstall`. So every handover takes two PRs:

1. **PR A:** set `prune: false` on the Flux Kustomization (for HelmReleases additionally `spec.suspend: true`; verify against the Flux docs that a suspended HelmRelease is not uninstalled on deletion). Add the Argo `Application` with auto-sync **off**. Check `argocd app diff`: should be (nearly) empty.
2. **PR B:** delete the Flux Kustomization (resources are orphaned, keep running). Enable auto-sync on the `Application`; Argo adopts the resources. Clean up leftovers (Flux labels, `sh.helm.release.v1.*` secrets for Helm charts).

Prune protection translates from `kustomize.toolkit.fluxcd.io/prune: disabled` to `argocd.argoproj.io/sync-options: Prune=false,Delete=false`. Also decide per `Application` whether it gets the `resources-finalizer.argocd.argoproj.io` finalizer (deleting the Application then cascades to its resources).

## Phase 0: prerequisites and Argo CD install

- [ ] Merge `refactor/cert-manager-to-helm` first, so the two changes don't get mixed up
- [ ] Decide install method: upstream `install.yaml` via kustomize (preferred: matches the rest of `infrastructure/`, raw objects visible) or Helm chart via `HelmRelease`
- [ ] Flux Kustomization `argocd` → `infrastructure/argocd/` (pinned version, resource requests/limits, RAM is tight)
- [ ] Separate read-only deploy key for the repo as a SealedSecret
- [ ] Root `Application` (app-of-apps) applied by Flux, pointing at a directory **outside** `clusters/super-hellthalos/` (Flux applies everything in there), e.g. `argocd/apps/`. One hand-written `Application` per file, mirroring `clusters/super-hellthalos/infrastructure/*.yaml`. No ApplicationSet yet: per-app settings differ during the handover (auto-sync off in PR A, prune/delete protection, `ServerSideApply`), which a single template expresses badly
- [ ] `AppProject`s instead of `default` (which allows everything): `infrastructure` (cluster-scoped resources allowed, destinations limited to the namespaces in use) and `apps` (namespaced resources plus `Namespace` only, destinations limited to app namespaces). Learn: Argo's guardrail/multi-tenancy model. Doing it now avoids moving Applications between projects later
- [ ] UI via `kubectl port-forward` first; later via Gateway + Cloudflare Access (Access application first, as for Grafana)
- [ ] Add an ownership table to `docs/STATUS.md`

## Phase 1: `wg-app`

Low stakes, nothing depends on it. Learn: `Application`, `syncPolicy` (automated, prune, selfHeal), the handover recipe.

- [ ] Hand over `wg-app`
- [ ] Image automation (wg-app roadmap phase 4): use Argo CD Image Updater instead of Flux image automation

## Phase 2: `kube-prometheus-stack`

First Helm chart. Learn: how Argo renders Helm (`helm template`, no release secret, no `lookup`, Helm hooks become Argo hooks), `ServerSideApply=true` for the large CRDs, `ignoreDifferences`.

- [ ] Decide the directory layout for Helm components. Plan: multi-source `Application` (chart from the OCI registry with `targetRevision`, `ref: values` source for `$values/.../values.yaml`, a plain git path source for extra manifests like Namespace, HTTPRoute, SealedSecret). The extras need their own directory, otherwise Argo also applies `helmrelease.yaml`/`ocirepository.yaml`. No umbrella/wrapper charts: they add `Chart.lock`/`helm dependency build`, force values to be nested under the subchart name, and render extras as Helm templates
- [ ] Hand over (HelmRelease + OCIRepository → `Application` with an OCI Helm source; the hashed ConfigMap workaround goes away)
- [ ] Grafana `HTTPRoute` still works

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

Workloads under `apps/` are uniform (namespace, plain manifests, auto-sync), so "add a directory = deploy" fits there; infrastructure stays explicit. Learn: ApplicationSet generators and templates.

- [ ] `ApplicationSet` with a git directory generator over `apps/*`, project `apps`
- [ ] Replace the hand-written `wg-app` `Application`: check how the ApplicationSet controller treats an existing Application with the same name; if unclear, delete the old one without cascading (no finalizer) first
- [ ] Document in `.claude/CLAUDE.md` how to add a new app

## Phase 7: decide the end state

Two options for `cilium` and Argo CD itself:

- **Flux as thin bootstrap layer:** Flux keeps Cilium, sealed-secrets, Argo CD; Argo handles everything else. Legitimate pattern, less risk.
- **All Argo:** Argo manages itself, Flux uninstalled (`flux uninstall`, after making sure nothing is still owned by Flux). Cilium under Argo needs a fix for the Hubble certs (`genCA` re-renders on every sync → permanent `OutOfSync`): `hubble.tls.auto.method: cronJob` or `certmanager`.

Either way:

- [ ] Rebuild procedure in `docs/STATUS.md` updated and ideally tested
- [ ] `.claude/CLAUDE.md` updated (commands, prune protection conventions, repo layout)

## Costs to keep in mind

- Two tools and two mental models during the transition
- ~0.5–1 GB extra RAM for Argo CD (repo-server spikes when rendering large charts)
- Rebuild story is more complex while both are in use: document it

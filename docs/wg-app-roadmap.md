# wg-app deployment roadmap

Rough plan for moving `../wg-app` (currently live on a VPS with real data, Docker Compose) into this cluster. Written 2026-10-25.

## Starting point

The compose stack and where each part goes:

| Compose service | In the cluster |
|---|---|
| `db` (postgres:16) | Already exists: `pg-main`, database and role `wg_app` |
| `backup` (pg_dump → R2) | Dropped: CNPG WAL archiving + daily base backups replace it (restore tested 2026-10-25) |
| `cloudflared` (token tunnel) | Already exists: add one ingress rule to `infrastructure/cloudflared/config.yaml` |
| `migrate` (one-shot) | initContainer on `api` (alternative: Job, but Jobs are immutable, awkward under Flux) |
| `api` | Deployment + Service `api` |
| `worker` (cron) | Deployment, `replicas: 1`, `strategy: Recreate` (singleton, never two copies) |
| `caddy` (PWA + proxy) | Deployment + Service; Caddyfile proxies to `api:3000`, which resolves to Service `api` in the same namespace |

Images: public on GHCR, `linux/amd64` only, tagged `latest` and `sha-<commit>`. Pin `sha-` tags in manifests.

## Phase 0: prerequisites

- [ ] Confirm both nodes are amd64: `kubectl get nodes -L kubernetes.io/arch`
- [ ] Repo layout: new top level `apps/wg-app/`, Flux Kustomization `wg-app` in `clusters/super-hellthalos/apps.yaml`, `dependsOn` `pg-main`, `sealed-secrets`, `gateway`

## Phase 1: run in the cluster, not exposed

- [ ] Namespace `wg-app`, workloads compatible with PSA `restricted` (non-root, no privilege escalation, drop all capabilities, seccomp `RuntimeDefault`)
- [ ] SealedSecret with `WG_TOKEN_SECRET`, `VAPID_PUBLIC_KEY`, `VAPID_PRIVATE_KEY`, `VAPID_SUBJECT` (same values as the VPS, otherwise devices lose their token and push subscriptions)
- [ ] `DATABASE_URL`: the DB password lives in Secret `pg-main-wg-app` in `databases`, Secrets cannot be referenced across namespaces. Decide: seal a copy into `wg-app`, or another approach. Build the URL with `$(VAR)` env interpolation pointing at `pg-main-rw.databases.svc:5432/wg_app`
- [ ] Deployments for `api` (with migrate initContainer), `worker`, `web`; Services `api` and `web`
- [ ] Probes and resource requests/limits (RAM is tight)
- [ ] Test via `kubectl -n wg-app port-forward svc/web 8080:80`

## Phase 2: expose

- [ ] Decide on Cloudflare Access: the app has its own token auth and is a PWA with a service worker and Web Push, which Access tends to break. Probably no Access (or a bypass policy) for this hostname
- [ ] `HTTPRoute` for `wg.superhellth.com` on Gateway `main` (wildcard cert already covers it)
- [ ] cloudflared ingress rule for `wg.superhellth.com`
- [ ] Optional: route `/api` straight to Service `api` via HTTPRoute path matching, leaving Caddy as a static file server

The DNS record is **not** changed in this phase: `wg.superhellth.com` still points at the VPS tunnel until cutover.

## Phase 3: data migration and cutover

- [ ] Rehearsal while the VPS stays live: `pg_dump` from the VPS, restore into `wg_app` with `--no-owner --no-acl` (VPS role `wg` vs. cluster role `wg_app`), check the app via port-forward, then wipe
- [ ] Announce a short downtime window
- [ ] Stop `api` and `worker` on the VPS (no more writes)
- [ ] Final `pg_dump`, restore into `wg_app`, check row counts / spot-check the data
- [ ] Repoint `wg.superhellth.com` to the cluster tunnel (rollback: point it back)
- [ ] Check the app end to end, including push notifications
- [ ] Remove the `deploy` job (SSH to VPS) from wg-app's CI, keep image build and push
- [ ] Keep the VPS around (stopped) for a while, then shut it down

## Phase 4: hardening

- [ ] NetworkPolicy: only `wg-app` (and the CNPG operator) may reach `databases`
- [ ] ServiceMonitor, if the API exposes metrics
- [ ] Flux image automation: new `sha-` tags open PRs automatically
- [ ] Update `docs/STATUS.md` and `.claude/CLAUDE.md` with the final state

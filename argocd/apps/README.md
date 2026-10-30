# Argo CD applications

Every YAML file in this directory is one Argo CD `Application`. The root `Application` (`infrastructure/argocd-config/root-application.yaml`) syncs this directory to the `argocd` namespace, so merging a file here creates the `Application`, and removing a file deletes it.

- One hand-written `Application` per file. Plan and handover recipe: `docs/argocd-migration-roadmap.md`.
- Set `spec.project` to `apps` or `infrastructure`, not `default`. A new namespace must first be added to the project's `destinations` in `infrastructure/argocd-config/`.
- Do not add `resources-finalizer.argocd.argoproj.io` by default. Without it, deleting an `Application` leaves its resources running.
- Only one tool may own a resource. Update the ownership table in `docs/STATUS.md` in every handover PR.

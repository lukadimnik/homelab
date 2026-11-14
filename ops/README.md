# ops

Manifests that are applied **manually** with `kubectl apply -f`. They are **not reconciled by Flux**: nothing under `clusters/staging/` points here, so nothing in this folder is deployed automatically.

Use this folder for one-off or on-demand resources such as restore tests, debug pods, and migration jobs. Set `metadata.namespace` explicitly in every file, because no Kustomization fills it in.

| File | Purpose |
|---|---|
| `health-vitals/db-restore.yaml` | Restores `health-vitals-db` from the R2 backup into a temporary cluster. See `docs/database-backups.md`. |

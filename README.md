# homelab

GitOps configuration for my Kubernetes homelab, continuously reconciled by [Flux](https://fluxcd.io/).

## How it works

Flux watches this repo and applies any changes to the cluster automatically — the repo is the source of truth. Secrets are encrypted in-place with [SOPS](https://github.com/getsops/sops) + [age](https://github.com/FiloSottile/age), so they're safe to commit.

Apps are exposed to the internet through a [Cloudflare Tunnel](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/), avoiding any inbound port forwarding.

## Layout

```
apps/
  base/        # environment-agnostic manifests (Kustomize bases)
  staging/     # overlays + secrets per app
clusters/
  staging/     # Flux entrypoint: sync config + bootstrap components
```

## Apps

| App | Description | URL |
|-----|-------------|-----|
| [linkding](https://github.com/sissbruecker/linkding) | Self-hosted bookmark manager | `linkdinghp.lukadimnik.com` |

## Working with secrets

Files are encrypted following the rules in `clusters/staging/.sops.yaml` (only `data`/`stringData` fields).

```sh
sops apps/staging/linkding/test-secret.yaml   # view / edit
```

Flux decrypts them on the cluster using the `sops-age` secret.

## Adding an app

1. Add base manifests under `apps/base/<app>/`.
2. Add a staging overlay under `apps/staging/<app>/` and reference it from the parent kustomization.
3. Encrypt any secrets with SOPS, then commit — Flux takes it from there.

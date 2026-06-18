# homelab

GitOps configuration for my Kubernetes homelab, continuously reconciled by [Flux](https://fluxcd.io/).

## How it works

Flux watches this repo and applies any changes to the cluster automatically — the repo is the source of truth. Secrets are encrypted in-place with [SOPS](https://github.com/getsops/sops) + [age](https://github.com/FiloSottile/age), so they're safe to commit.

The cluster is split into three reconciliation domains, each driven by its own Flux `Kustomization` under `clusters/staging/`:

- **apps** (`apps.yaml`) → workloads in `./apps/staging`
- **infrastructure** (`infrastructure.yaml`) → cluster tooling in `./infrastructure/controllers/staging`
- **monitoring** (`monitoring.yaml`) → the observability stack in `./monitoring/controllers/staging` and `./monitoring/configs/staging`

Services are exposed over HTTPS through a [Traefik](https://traefik.io/) `Ingress`. A [Cloudflare Tunnel](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/) (`cloudflared`) is also configured for inbound access without port forwarding, though it's currently disabled in the linkding overlay.

## Layout

Everything follows the same Kustomize pattern: environment-agnostic `base/` manifests, with a per-environment `staging/` overlay (currently the only environment) that layers in config and secrets.

```
apps/
  base/        # environment-agnostic app manifests (Kustomize bases)
  staging/     # app overlays + secrets
infrastructure/
  controllers/
    base/      # cluster tooling bases (Renovate)
    staging/   # tooling overlays
monitoring/
  controllers/
    base/      # kube-prometheus-stack HelmRelease + repo
    staging/   # overlay
  configs/
    staging/   # monitoring config + secrets (e.g. Grafana TLS)
clusters/
  staging/     # Flux entrypoint: GitRepository, sync config, per-domain Kustomizations
    flux-system/   # bootstrapped Flux components (generated — do not edit)
```

## Apps

| App | Description | URL |
|-----|-------------|-----|
| [linkding](https://github.com/sissbruecker/linkding) | Self-hosted bookmark manager | `lds.lukadimnik.com` |

## Infrastructure

| Component | Description |
|-----------|-------------|
| [Renovate](https://docs.renovatebot.com/) | Self-hosted dependency updates, run as an hourly `CronJob` in the `renovate` namespace |

[Renovate](https://docs.renovatebot.com/) keeps dependencies (container image tags, Helm charts, etc.) up to date by opening PRs against this repo. Instead of the hosted GitHub App, it runs in-cluster as a `CronJob` (`infrastructure/controllers/base/renovate/`) that scans `lukadimnik/homelab` every hour. Behaviour is configured two ways:

- `renovate.json` — the per-repo config Renovate reads. It enables the Kubernetes manager so all `*.yaml` manifests are scanned for updatable images.
- `renovate-configmap` / `renovate-container-env` — runtime settings (platform, git author) and the encrypted GitHub token Renovate uses to raise PRs.

## Monitoring

The [kube-prometheus-stack](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack) Helm chart is deployed as a Flux `HelmRelease` (`monitoring/controllers/base/kube-prometheus-stack/`), providing Prometheus, Alertmanager, Grafana, and node-exporter in the `monitoring` namespace.

| Component | URL |
|-----------|-----|
| Grafana | `grafana.lukadimnik.com` (Traefik ingress, TLS) |

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

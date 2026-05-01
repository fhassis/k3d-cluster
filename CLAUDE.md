# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

Infrastructure-as-code for a three-tier Kubernetes platform. There is no application code — everything here is Kubernetes/Helm configuration consumed by ArgoCD. Changes take effect by pushing to git; ArgoCD reconciles the cluster automatically.

Target environments:
- `dev` — local k3d (1 server + 3 agents) or Rancher Desktop. No ArgoCD, direct kubectl/helm.
- `staging` — 3-node k3s on Multipass VMs (node1/2/3). ArgoCD + Longhorn + TLS.
- `prod` — 5-node k3s on SLES. ArgoCD + Longhorn + TLS.

## Dev cluster lifecycle (k3d)

```bash
# Create (sets kubeconfig automatically)
k3d cluster create --config clusters/k3d.yaml

# Create the "longhorn" StorageClass alias (required once per cluster).
# All Helm values files use storageClass: longhorn; this makes PVCs resolve
# on k3d via the built-in local-path provisioner. See storage/longhorn-sc.yaml.
kubectl apply -f storage/longhorn-sc.yaml

# Delete
k3d cluster delete dev
```

## Staging cluster lifecycle (Multipass)

See [clusters/multipass/README.md](clusters/multipass/README.md) for full setup.

## ArgoCD

```bash
# Install (run once after cluster create — same for staging and prod)
helm repo add argo https://argoproj.github.io/argo-helm && helm repo update
kubectl create namespace argocd
helm install argocd argo/argo-cd -n argocd -f argocd/argocd.values.yaml
kubectl -n argocd rollout status deployment/argocd-server

# Bootstrap GitOps (choose the right file for the environment)
kubectl apply -f argocd/bootstrap.staging.yaml   # staging
kubectl apply -f argocd/bootstrap.prod.yaml       # prod

# Get admin password
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d && echo

# Watch sync status
kubectl -n argocd get applications -w
```

## Observability stack (manual install on dev, bypasses ArgoCD)

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add grafana-community https://grafana-community.github.io/helm-charts
helm repo update
kubectl create namespace observability

helm install prometheus prometheus-community/prometheus -n observability -f observability/prometheus.values.yaml
helm install loki grafana-community/loki -n observability -f observability/loki.values.yaml
helm install tempo grafana-community/tempo -n observability -f observability/tempo.values.yaml
helm install grafana grafana-community/grafana -n observability -f observability/grafana.values.yaml
helm install k8s-monitoring grafana/k8s-monitoring -n observability -f observability/k8s-monitoring.values.yaml
```

## Architecture

### GitOps flow

```
git push
  └── ArgoCD detects change
        └── ArgoCD applies Helm release with updated values
```

`argocd/bootstrap.staging.yaml` (or `bootstrap.prod.yaml`) is applied once manually. It points ArgoCD at `argocd/apps/staging/` (or `prod/`). Every manifest in the base or overlays is automatically discovered and managed.

### Multi-source Helm pattern

Every Application in `argocd/apps/` uses ArgoCD's multi-source feature (≥ 2.6):
- **source[0]**: upstream Helm chart repo
- **source[1]**: this git repo, referenced as `$values`, providing the `*.values.yaml` file

To change a chart's config: edit the relevant values file, push — ArgoCD reconciles.

### Observability sync waves

`argocd/apps/observability.yaml` deploys five Applications in dependency order:
- **Wave 0** — Prometheus, Loki, Tempo (independent backends)
- **Wave 1** — Grafana (datasources wired to wave 0)
- **Wave 2** — k8s-monitoring/Alloy (remote-writes to wave 0 backends)

### OTLP ingestion

Apps send traces to the Alloy singleton (not the DaemonSet):
- gRPC: `k8s-monitoring-alloy-singleton.observability.svc.cluster.local:4317`
- HTTP: `k8s-monitoring-alloy-singleton.observability.svc.cluster.local:4318`

## Key conventions

- `targetRevision: HEAD` is used in staging; pin to a semver for production.
- Staging and prod overlays are structurally identical — same Applications, same Longhorn, same TLS setup. Only hostnames and cert source differ.
- All values files use `storageClass: longhorn`. On dev, the `local-path` provisioner is the default and handles this automatically.
- Adding a new app: create a manifest in `argocd/apps/base/`, add env-specific patches in `staging/` and `prod/` overlays, commit, push.

## Production hardening notes (not implemented yet)

- ArgoCD: flip `server.insecure: false`, enable Dex SSO, increase replicas, `redis-ha.enabled: true`.
- Observability: larger PVCs, longer Prometheus retention, `alertmanager.enabled: true`, object storage for Loki/Tempo.
- Grafana Cloud migration: change only the `destinations` block in `k8s-monitoring.values.yaml`.

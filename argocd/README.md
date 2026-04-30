# ArgoCD

GitOps controller running in the `argocd` namespace. Manages all Application deployments by reconciling the cluster against this git repository.

## Architecture

```
argocd/bootstrap.staging.yaml   ← apply once manually (prod: bootstrap.prod.yaml)
  └── argocd/apps/staging/      ← Kustomize overlay ArgoCD watches
        kustomization.yaml        patches base/ apps with staging-specific values
  └── argocd/apps/base/         ← Application manifests, shared across envs
```

Each Application uses the **multi-source** pattern (ArgoCD ≥ 2.6):
- Source 1: upstream Helm chart
- Source 2: this git repo, providing the `*.values.yaml` file via `$values` reference

This means: edit a values file, push, ArgoCD reconciles — no manual `helm upgrade`.

## Install

```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update

kubectl create namespace argocd
helm install argocd argo/argo-cd \
  -n argocd -f argocd/argocd.values.yaml

kubectl -n argocd rollout status deployment/argocd-server
```

## Access

ArgoCD is exposed via Traefik ingress. On staging, add `argocd.staging` to your Windows hosts file (see root [README.md](../README.md) § Staging DNS), then open:

<https://argocd.staging> — log in as `admin` / `<password below>`.

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d && echo
```

If you need to bypass ingress:

```bash
kubectl -n argocd port-forward svc/argocd-server 8888:80
# → http://localhost:8888
```

## Bootstrap GitOps

```bash
# Staging
kubectl apply -f argocd/bootstrap.staging.yaml

# Prod
kubectl apply -f argocd/bootstrap.prod.yaml
```

ArgoCD discovers `argocd/apps/` and deploys all Applications automatically. Watch progress:

```bash
kubectl -n argocd get applications -w
```

The observability stack deploys in waves (0 → 1 → 2), respecting startup ordering.

## Adding new apps

Create a new Application manifest in `argocd/apps/base/`, add env-specific patches to both `staging/kustomization.yaml` and `prod/kustomization.yaml`, commit, and push. ArgoCD picks it up automatically.

## Uninstall

```bash
kubectl delete -f argocd/apps/
kubectl delete -f argocd/bootstrap.staging.yaml
helm uninstall argocd -n argocd
kubectl delete namespace argocd
```

## Upgrading ArgoCD

```bash
helm repo update
helm upgrade argocd argo/argo-cd \
  -n argocd -f argocd/argocd.values.yaml
```

## Adapting for production

- TLS: flip `configs.params.server.insecure` to `false` and add TLS to the ingress.
- SSO: `dex.enabled: true` + OIDC config.
- HA: increase `controller.replicas` and enable `redis-ha.enabled: true`.
- Pin `targetRevision` in all Application manifests to a specific chart version.

## Private repo credentials

If this repo is private, register it in ArgoCD after install:

```bash
argocd repo add https://github.com/fhassis/k8s-cluster.git \
  --username fhassis --password YOUR_PAT
```

# k3d-cluster

Platform repository for local and production Kubernetes cluster configuration. Manages a k3d dev cluster, ArgoCD GitOps bootstrap, and the Grafana observability stack.

Target environments:
- `dev` (local, 3-node: 1 server + 2 agents)
- Single-node VPS running k3s
- 5-node k3s cluster

## Repository layout

```
clusters/       k3d cluster config files
argocd/         ArgoCD install values + bootstrap Application
argocd/apps/    ArgoCD Application manifests (watched by bootstrap)
observability/  Helm values for the Grafana observability stack
```

## k3d Cluster Setup

```bash
# Create
k3d cluster create --config clusters/dev.yaml

# Delete
k3d cluster delete dev
```

## Ingress and DNS

k3s ships Traefik as the ingress controller. The cluster config maps host ports 80 and 443 to the k3d load balancer, so Traefik is reachable directly from the host without `port-forward`.

All services use `*.fhassis.top` as their hostname in every environment. In dev, resolve this to localhost via the Windows hosts file (since k3d runs inside WSL2):

**`C:\Windows\System32\drivers\etc\hosts`**
```
127.0.0.1  grafana.fhassis.top
127.0.0.1  argocd.fhassis.top
```

Add one line per service as you deploy them. In production, point the wildcard DNS A record `*.fhassis.top` to the cluster's public IP instead.

| Service | Dev URL |
|---|---|
| Grafana | <http://grafana.fhassis.top> |
| ArgoCD | <http://argocd.fhassis.top> |

## ArgoCD

GitOps controller running in the `argocd` namespace. See [argocd/README.md](argocd/README.md) for install, bootstrap, and access instructions.

## Observability

Grafana stack (Prometheus, Loki, Tempo, Grafana, Alloy) in the `observability` namespace. See [observability/README.md](observability/README.md) for install and access instructions.

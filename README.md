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

Create the cluster
```bash
k3d cluster create --config clusters/dev.yaml
```

Delete the cluster
```bash
k3d cluster delete dev
```

## Ingress and DNS

k3s ships Traefik as the ingress controller. The cluster config maps host ports 8080 and 8443 to the k3d load balancer, so Traefik is reachable directly from the host without `port-forward`.

Dev uses `*.localhost` hostnames to avoid browser HTTPS upgrades (no HSTS, no cert management). Add entries to the Windows hosts file (since k3d runs inside WSL2):

**`C:\Windows\System32\drivers\etc\hosts`**
```
127.0.0.1  grafana.localhost
127.0.0.1  argocd.localhost
```

Add one line per service as you deploy them. Production environments use real DNS and TLS.

| Service | Dev URL |
|---|---|
| Grafana | <http://grafana.localhost:8080> |
| ArgoCD | <http://argocd.localhost:8080> |

## ArgoCD

GitOps controller running in the `argocd` namespace. See [argocd/README.md](argocd/README.md) for install, bootstrap, and access instructions.

## Observability

Grafana stack (Prometheus, Loki, Tempo, Grafana, Alloy) in the `observability` namespace. See [observability/README.md](observability/README.md) for install and access instructions.

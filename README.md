# k8s-cluster

Infrastructure-as-code for a three-tier Kubernetes platform: local development, a staging cluster for drain and storage tests, and production.

**Primary goal**: eliminate monthly full-service outages caused by OS maintenance. With Longhorn block storage, nodes can be drained one at a time — each application pod migrates to another node and resumes from its existing data within minutes, instead of going fully offline.

## Environments

| Tier | Tool | ArgoCD | Storage | TLS |
|------|------|--------|---------|-----|
| `dev` | k3d or Rancher Desktop | No — direct kubectl/helm | local-path (k3s built-in) | No |
| `staging` | Multipass VMs (node1/2/3) | Yes | Longhorn | Self-signed `*.staging` cert |
| `prod` | 5-node k3s on SLES | Yes | Longhorn | `*.fhassis.top` cert |

## Repository Layout

```
clusters/        k3d config + Multipass cloud-init scripts
argocd/          ArgoCD install values + bootstrap Applications
argocd/apps/     ArgoCD Application manifests (base + staging/ and prod/ overlays)
observability/   Helm values for the Grafana stack (Prometheus, Loki, Tempo, Grafana, Alloy)
longhorn/        Longhorn Helm values (staging + prod)
on-premises/     Helm values + README for Jenkins, GitLab, Nexus, XWiki
certificates/    Traefik TLSStore manifest (staging + prod)
docs/            Step-by-step test guides
```

## Dev Cluster (k3d)

No ArgoCD — use `kubectl apply` or `helm install` directly for fast iteration.

```bash
k3d cluster create --config clusters/k3d.yaml
kubectl get nodes   # 1 server + 3 agents, all Ready

# Delete
k3d cluster delete dev
```

Alternatively, use **Rancher Desktop** on Windows for a simpler single-node setup.

### Dev DNS

k3d maps host ports 8080 (HTTP) and 8443 (HTTPS) to Traefik. Add to your Windows hosts file:

**`C:\Windows\System32\drivers\etc\hosts`**
```
127.0.0.1  grafana.localhost jenkins.localhost nexus.localhost
127.0.0.1  xwiki.localhost gitlab.localhost
```

## Staging Cluster (Multipass)

Three Ubuntu VMs with real Longhorn and HTTPS. See [clusters/multipass/README.md](clusters/multipass/README.md) for full setup instructions.

### Staging DNS

Add to Windows hosts file (replace with node1's actual IP):

```
192.168.x.x   argocd.staging grafana.staging prometheus.staging loki.staging
192.168.x.x   jenkins.staging nexus.staging xwiki.staging gitlab.staging
```

## Service URLs

| Service | Dev | Staging | Prod |
|---------|-----|---------|------|
| ArgoCD | — | https://argocd.staging | https://argocd.fhassis.top |
| Grafana | http://grafana.localhost:8080 | https://grafana.staging | https://grafana.fhassis.top |
| Jenkins | http://jenkins.localhost:8080 | https://jenkins.staging | https://jenkins.fhassis.top |
| GitLab | http://gitlab.localhost:8080 | https://gitlab.staging | https://gitlab.fhassis.top |
| Nexus | http://nexus.localhost:8080 | https://nexus.staging | https://nexus.fhassis.top |
| XWiki | http://xwiki.localhost:8080 | https://xwiki.staging | https://xwiki.fhassis.top |
| Longhorn UI | — | — | https://longhorn.fhassis.top |

## Storage Strategy

| Environment | Storage | How it works |
|-------------|---------|--------------|
| Dev (k3d / Rancher Desktop) | local-path (k3s default) | Dynamic provisioning, no config needed. |
| Staging / Prod | Longhorn | Block volume detaches from drained node, reattaches on the target node. Single replica — mobility, not redundancy. |

All on-premises tools use `storageClass: longhorn` in their Helm values. On dev, the `local-path` StorageClass is aliased as `longhorn` by the base values — no app changes needed across tiers.

See [longhorn/README.md](longhorn/README.md) for details, including SLES and Ubuntu prerequisites.

## ArgoCD

GitOps controller in the `argocd` namespace. See [argocd/README.md](argocd/README.md) for install, bootstrap, and access instructions.

### Sync Wave Order

```
Wave -1  longhorn + certificates — storage and TLS must exist first
Wave  0  prometheus, loki, tempo
Wave  1  grafana
Wave  2  k8s-monitoring (Alloy)
Wave  3  jenkins, nexus, xwiki
Wave  4  gitlab (heaviest — starts last)
```

## Observability

Grafana stack (Prometheus, Loki, Tempo, Grafana, Alloy) in the `observability` namespace. See [observability/README.md](observability/README.md) for install and access instructions.

Pod logs from all namespaces are collected automatically by Alloy and queryable in Grafana → Explore → Loki.

## On-Premises Tools

Jenkins, GitLab, Nexus OSS, and XWiki deployed in separate namespaces. See [on-premises/README.md](on-premises/README.md) for the overview and links to per-tool documentation.

## Tests

- [Drain Test](docs/drain-test.md) — drain a node and verify an app migrates without data loss.
- [Jenkins Agent Test](docs/jenkins-agent-test.md) — drain the Jenkins controller node mid-build and verify the build resumes.

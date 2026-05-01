# SonarQube

SonarQube is a static code analysis platform. It is deployed via the official [SonarSource Helm chart](https://github.com/SonarSource/helm-chart-sonarqube) (Community Edition — free, no license required).

## Chart Choice

The official `sonarqube/sonarqube` chart is the only maintained, production-grade option. As of version 2026.x, it no longer bundles PostgreSQL, so a separate `bitnami/postgresql` release is deployed alongside it.

## Two Releases per Environment

| Release name | Chart | Purpose |
|---|---|---|
| `sonarqube-db` | `bitnami/postgresql` | Database backend |
| `sonarqube` | `sonarqube/sonarqube` | Application server |

The JDBC URL in `sonarqube.values.yaml` targets the service `sonarqube-db-postgresql` in the `sonarqube` namespace.

## What Is Configured

- Community Edition (free, no license key)
- Longhorn-backed PVCs for both the app data and PostgreSQL
- Admin password set on first boot (`S0narQube@Dev!`) — override per env
- Prometheus annotations on the pod for Alloy autodiscovery (scrapes `/api/monitoring/metrics`)
- Ingress via Traefik; hostname overridden per environment

## What Is NOT Configured (and Why)

| Feature | Status | Reason |
|---------|--------|--------|
| LDAP / SSO | Not configured | Future work; SonarQube Community supports SAML via Settings UI |
| Alertmanager integration | Not configured | Future work |
| Backup | Not explicit | Longhorn PVC is retained on deletion; add Longhorn snapshot schedules via the UI |
| External PostgreSQL (prod) | Commented out | Enable `jdbcOverwrite.jdbcSecretName` in `sonarqube.values.prod.yaml` and remove the bundled DB |

## Dev Usage

```bash
# Add repos (one-time)
helm repo add sonarqube https://SonarSource.github.io/helm-chart-sonarqube
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update

# Deploy PostgreSQL first
helm install sonarqube-db bitnami/postgresql \
  -n sonarqube --create-namespace \
  -f on-premises/sonarqube/postgresql.values.yaml

# Deploy SonarQube
helm install sonarqube sonarqube/sonarqube \
  -n sonarqube \
  -f on-premises/sonarqube/sonarqube.values.yaml \
  -f on-premises/sonarqube/sonarqube.values.dev.yaml
```

Access at: `http://sonarqube.localhost:8080`  
Login: `admin` / `S0narQube@Dev!`

## Production Hardening

- Store DB password and admin password in Kubernetes Secrets; reference via `jdbcOverwrite.jdbcSecretName` and `setAdminPassword.passwordSecretName`.
- Pin `targetRevision` in the ArgoCD Application to a specific chart semver.
- Deploy a dedicated, externally managed PostgreSQL (not the bundled Bitnami release).
- Increase `persistence.size` based on project count and analysis volume.

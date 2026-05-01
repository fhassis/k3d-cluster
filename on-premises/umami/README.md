# Umami

Privacy-first, open-source web analytics. Collects page views and events from all on-premises tools via automatic script injection at the nginx ingress layer.

| Environment | URL |
|---|---|
| Dev | http://umami.localhost:8080 |
| Staging | https://umami.staging |
| Prod | https://umami.fhassis.top |

## Components

| Name | Chart | Namespace |
|---|---|---|
| `umami-db` | `bitnami/postgresql` | `umami` |
| `umami` | `christianhuth/umami` | `umami` |

`umami-db` must be running before `umami` starts (both at wave 3; DB readiness gates the app migration).

## First-Time Setup (post-deploy)

After the wave-3 sync completes:

1. Log in at `https://umami.fhassis.top` with `admin / umami`. **Change the password immediately.**
2. Settings → Websites → Add website. Name: `On-Premises Tools`, Domain: `*.fhassis.top`.
3. Copy the generated website UUID (e.g. `a1b2c3d4-e5f6-7890-abcd-ef1234567890`).
4. Replace `REPLACE_WITH_UUID` in every `configuration-snippet` annotation across all on-prem ingress overlay files. Run:
   ```bash
   grep -rl "REPLACE_WITH_UUID" on-premises/ observability/ | xargs sed -i 's/REPLACE_WITH_UUID/<your-uuid>/g'
   ```
5. Push → ArgoCD reconciles → nginx reloads → analytics are live.
6. Open any on-prem tool in the browser; its page view should appear in the Umami dashboard within seconds.

## Production Secret Management

The default `umami.appSecret.secret` in `umami.values.yaml` is a placeholder. In prod:

```bash
kubectl create secret generic umami-app-secret -n umami \
  --from-literal=app-secret=$(openssl rand -hex 32)
```

Then set in `umami.values.prod.yaml`:
```yaml
umami:
  appSecret:
    existingSecret: umami-app-secret
    secret: ""
```

## How Script Injection Works

nginx-ingress-controller `sub_filter` replaces `</head>` with the Umami tracking script tag before the response reaches the browser. This is configured via `nginx.ingress.kubernetes.io/configuration-snippet` on each on-prem ingress. Umami's own ingress is intentionally excluded to avoid circular tracking.

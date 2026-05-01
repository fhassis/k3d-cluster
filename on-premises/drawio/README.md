# draw.io (diagrams.net)

draw.io is a self-hosted diagramming tool for architecture, flowcharts, and technical drawings. It is deployed as plain Kubernetes manifests using the official `jgraph/drawio` Docker image.

## Chart Choice

draw.io has no official Helm chart. The app is stateless (all diagrams are stored client-side or in user-chosen storage), so a simple Deployment + Service + Ingress is sufficient. No PVC is required.

## What Is Configured

- Single-replica Deployment (`jgraph/drawio:latest`)
- ClusterIP Service routing port 80 → container port 8080
- Separate Ingress files per environment (`ingress.dev.yaml`, `ingress.staging.yaml`, `ingress.prod.yaml`)

## What Is NOT Configured (and Why)

| Feature | Status | Reason |
|---------|--------|--------|
| Persistence | Not needed | draw.io is stateless; diagrams are exported/saved by the user |
| Authentication | Not configured | draw.io has no built-in auth; add Traefik ForwardAuth for SSO if needed |
| Custom plugins | Not configured | Can be enabled via `DRAWIO_PLUGIN_LIST` env var if required |

## Dev Usage

```bash
kubectl apply -f on-premises/drawio/drawio.yaml
kubectl apply -f on-premises/drawio/ingress.dev.yaml
```

Access at: `http://drawio.localhost:8080`

## ArgoCD Integration (staging / prod)

Use an ArgoCD `Application` with a `directory` source pointing to `on-premises/drawio/`. Apply the env-specific ingress via a Kustomize overlay or manually after sync.

## Production Hardening

- Pin the image tag to a specific version instead of `latest`.
- Add Traefik ForwardAuth middleware for authentication.
- Set `replicas: 2` for availability during rolling updates (stateless, safe to run multiple).

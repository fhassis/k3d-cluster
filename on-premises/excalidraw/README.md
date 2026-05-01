# Excalidraw

Excalidraw is a self-hosted virtual whiteboard for sketching hand-drawn-style diagrams. It is deployed as plain Kubernetes manifests using the official `excalidraw/excalidraw` Docker image.

## Chart Choice

Excalidraw has no official Helm chart. The app is stateless (diagrams are stored in the browser's local storage or exported by the user), so a simple Deployment + Service + Ingress is sufficient. No PVC is required.

## What Is Configured

- Single-replica Deployment (`excalidraw/excalidraw:latest`)
- ClusterIP Service routing port 80 → container port 80
- Separate Ingress files per environment (`ingress.dev.yaml`, `ingress.staging.yaml`, `ingress.prod.yaml`)

## What Is NOT Configured (and Why)

| Feature | Status | Reason |
|---------|--------|--------|
| Persistence | Not needed | Excalidraw is stateless; drawings are saved by the user |
| Authentication | Not configured | No built-in auth; add Traefik ForwardAuth for SSO if needed |
| Collaboration backend | Not configured | Real-time collab requires the Excalidraw+ backend; the open-source version is single-user |

## Dev Usage

```bash
kubectl apply -f on-premises/excalidraw/excalidraw.yaml
kubectl apply -f on-premises/excalidraw/ingress.dev.yaml
```

Access at: `http://excalidraw.localhost:8080`

## ArgoCD Integration (staging / prod)

Use an ArgoCD `Application` with a `directory` source pointing to `on-premises/excalidraw/`. Apply the env-specific ingress via a Kustomize overlay or manually after sync.

## Production Hardening

- Pin the image tag to a specific version instead of `latest`.
- Add Traefik ForwardAuth middleware for authentication.
- Set `replicas: 2` for availability during rolling updates (stateless, safe to run multiple).

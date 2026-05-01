# TLS Certificate Management

All web services (Jenkins, Grafana, Nexus, XWiki, GitLab, ArgoCD) are exposed via nginx-ingress-controller, which handles TLS termination. A single wildcard certificate for `*.fhassis.top` covers every subdomain from one Secret in `kube-system`.

## How it works

```
Browser → port 443 → ingress-nginx (ingress-nginx namespace)
                          ↓ default-ssl-certificate flag
                     kube-system/ssl-certificate (Secret)
                          ↓ terminates TLS
                     forwards plain HTTP to each pod
```

The `ingress-nginx` controller is started with `--default-ssl-certificate=kube-system/ssl-certificate` (configured in `nginx-ingress/ingress-nginx.values.yaml`). Every Ingress resource that declares a `tls:` block but omits `secretName` automatically receives this certificate — no per-namespace copies needed.

## Importing a certificate (DigiCert or any CA)

**1. Obtain the certificate from your CA**

You need two files:
- `cert.pem` — your certificate **plus the full intermediate chain**, concatenated:
  ```
  -----BEGIN CERTIFICATE-----
  (your wildcard cert)
  -----END CERTIFICATE-----
  -----BEGIN CERTIFICATE-----
  (intermediate CA cert)
  -----END CERTIFICATE-----
  ```
- `key.pem` — your private key (the one you generated the CSR with)

**2. Import as a Kubernetes TLS Secret**

```bash
kubectl create secret tls ssl-certificate \
  --cert=cert.pem \
  --key=key.pem \
  --namespace kube-system
```

nginx-ingress-controller picks up the new cert within seconds — no restart needed.

**3. Verify**

```bash
# Confirm the Secret exists
kubectl get secret ssl-certificate -n kube-system

# Test TLS from the command line (replace with any service hostname)
curl -v https://jenkins.fhassis.top 2>&1 | grep -E "subject|issuer|expire"
```

## Renewal

When your cert expires, delete the old Secret and re-create it:

```bash
kubectl delete secret ssl-certificate -n kube-system
kubectl create secret tls ssl-certificate \
  --cert=new-cert.pem \
  --key=new-key.pem \
  --namespace kube-system
```

nginx-ingress-controller reloads within seconds. No application restarts needed.

## Staging: self-signed wildcard cert

For staging, generate a self-signed cert (see `clusters/multipass/README.md § TLS pre-flight`). The same `kubectl create secret tls ssl-certificate` command applies.

## Why not per-namespace secrets?

Kubernetes Secrets are namespace-scoped. With 5+ namespaces (jenkins, nexus, xwiki, gitlab, observability), per-namespace certs would require either a replication controller or manual copies on every renewal. The `default-ssl-certificate` flag avoids this entirely: one Secret, all services.

## Switching CAs

Because all Ingress resources omit `secretName` from their `tls:` block, switching CAs only requires replacing the `kube-system/ssl-certificate` Secret. Zero changes to any Helm values or application configs.

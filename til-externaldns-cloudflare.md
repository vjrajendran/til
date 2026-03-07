# TIL: Automating DNS with ExternalDNS + Cloudflare

> Date: 2026-03-07
> Tags: `kubernetes` `external-dns` `cloudflare` `helm` `automation`

---

## The Problem

Every time a developer adds an Ingress, someone has to manually create a DNS record.
ExternalDNS solves this by watching Ingress resources and creating DNS records automatically.

```
Developer pushes Ingress → ExternalDNS → Cloudflare API → DNS record created ✅
```

---

## Setup

### 1. Get Cloudflare API Token

- Cloudflare Dashboard → My Profile → API Tokens → Create Token
- Use template: **Edit zone DNS**
- Scope it to your domain only
- Copy the token — shown only once

### 2. Create the secret
```bash
kubectl create namespace external-dns

kubectl create secret generic cloudflare-api-token \
  --from-literal=api-token=<YOUR_TOKEN> \
  --namespace external-dns
```

### 3. Create `external-dns-values.yaml`
```yaml
provider:
  name: cloudflare

env:
  - name: CF_API_TOKEN
    valueFrom:
      secretKeyRef:
        name: cloudflare-api-token
        key: api-token

txtOwnerId: "k3s-jameson"   # unique ID — ExternalDNS uses this to track records it owns

domainFilters:
  - tecminal.com             # only manage records under this domain

policy: sync                 # create AND delete records automatically

sources:
  - ingress                  # watch Ingress resources

cloudflare:
  proxied: false             # must be false for cert-manager HTTP01 to work
```

### 4. Install via Helm
```bash
helm repo add external-dns https://kubernetes-sigs.github.io/external-dns/
helm repo update

helm upgrade --install external-dns external-dns/external-dns \
  --namespace external-dns \
  --values external-dns-values.yaml
```

---

## Verify It's Working

```bash
# Check logs for sync activity
kubectl logs -n external-dns deployment/external-dns | grep tecminal

# Watch live
kubectl logs -n external-dns deployment/external-dns -f
```

---

## Testing

```bash
# Terminal 1 — watch logs
kubectl logs -n external-dns deployment/external-dns -f

# Terminal 2 — create a test ingress
kubectl create ingress test-app \
  --rule="test.tecminal.com/*=jellyfin:8096" \
  --class=traefik \
  --namespace jellyfin-helm

# Cleanup — ExternalDNS will delete the DNS record automatically
kubectl delete ingress test-app -n jellyfin-helm
```

---

## CNAME vs A Record on Civo

On Civo, the Traefik LoadBalancer exposes a hostname instead of just an IP:
```
61f555ac-...k8s.civo.com → 74.220.23.21
```

ExternalDNS creates a `CNAME` to that hostname instead of an `A` record.
This is normal and resolves correctly.

To force an A record, annotate your Ingress:
```yaml
annotations:
  external-dns.alpha.kubernetes.io/target: "74.220.23.21"
```

---

## How It Fits in the Full Flow

```
Developer pushes Ingress
        ↓
ExternalDNS → creates DNS record in Cloudflare
        ↓
cert-manager → DNS resolves → HTTP01 challenge passes → TLS cert issued
        ↓
App is live with HTTPS — zero manual steps ✅
```

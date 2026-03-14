# TIL: External Secrets Operator + Doppler

> Date: 2026-03-14
> Tags: `kubernetes` `external-secrets` `doppler` `secrets` `gitops`

---

## The Problem

Secrets die with the cluster. Every new cluster requires manual secret recreation.
ESO + Doppler solves this — secrets live outside the cluster permanently.

---

## How It Works (3-step chain)


![ESO Flow](docs/images/eso-flow.png)

```
1. doppler-token secret (manual, one time)
        ↓
2. ClusterSecretStore — connects to Doppler using the token
        ↓
3. ExternalSecret — pulls specific secrets from Doppler
        ↓
Kubernetes Secret appears automatically ✅
```

Each step depends on the previous one. The `doppler-token` is the only
manual step — everything else is GitOps.

---

## Install ESO

```bash
helm repo add external-secrets https://charts.external-secrets.io
helm repo update

kubectl create namespace external-secrets

helm upgrade --install external-secrets \
  external-secrets/external-secrets \
  --namespace external-secrets
```

Verify 3 pods are running:
```bash
kubectl get pods -n external-secrets
# external-secrets
# external-secrets-cert-controller
# external-secrets-webhook
```

---

## Step 1 — Create Doppler Token Secret (manual, one time)

```bash
# Get token from: Doppler Dashboard → Project → prod → Access → Service Tokens
kubectl create secret generic doppler-token \
  --from-literal=dopplerToken="dp.st.xxxx" \
  --namespace external-secrets
```

⚠️ This is the bootstrap secret — must exist before anything else works.

---

## Step 2 — Create ClusterSecretStore

```yaml
apiVersion: external-secrets.io/v1
kind: ClusterSecretStore
metadata:
  name: doppler-backend
spec:
  provider:
    doppler:
      auth:
        secretRef:
          dopplerToken:
            name: doppler-token
            key: dopplerToken
            namespace: external-secrets
```

Verify it connected:
```bash
kubectl get clustersecretstore
# NAME             STATUS   READY
# doppler-backend  Valid    True  ✅
```

---

## Step 3 — Create ExternalSecret

### For a regular Opaque secret (e.g. Cloudflare token):
```yaml
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: cloudflare-api-token
  namespace: external-dns
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: doppler-backend
    kind: ClusterSecretStore
  target:
    name: cloudflare-api-token
    template:
      type: Opaque
  data:
    - secretKey: api-token
      remoteRef:
        key: CLOUDFLARE_API_TOKEN   # Doppler secret name
```

### For a Docker registry secret:
```yaml
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: dockerhub-secret
  namespace: jellyfin
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: doppler-backend
    kind: ClusterSecretStore
  target:
    name: dockerhub-secret
    template:
      type: kubernetes.io/dockerconfigjson
      data:
        .dockerconfigjson: |
          {"auths":{"https://index.docker.io/v1/":{"username":"{{ .DOCKERHUB_USERNAME }}","password":"{{ .DOCKERHUB_PASSWORD }}","auth":"{{ printf "%s:%s" .DOCKERHUB_USERNAME .DOCKERHUB_PASSWORD | b64enc }}"}}}
  data:
    - secretKey: DOCKERHUB_USERNAME
      remoteRef:
        key: DOCKERHUB_USERNAME
    - secretKey: DOCKERHUB_PASSWORD
      remoteRef:
        key: DOCKERHUB_PASSWORD
```

---

## Verify Secrets Are Syncing

```bash
# Check ExternalSecret status
kubectl get externalsecret -A
# STATUS should be: SecretSynced

# Check the actual secret was created
kubectl get secret dockerhub-secret -n jellyfin
kubectl get secret cloudflare-api-token -n external-dns
```

---

## Test: Delete and Watch It Recreate

```bash
kubectl delete secret dockerhub-secret -n jellyfin

# Watch ESO recreate it automatically
kubectl get secret -n jellyfin -w
```

Secret reappears within seconds — this is what happens on a new cluster. ✅

---

## Troubleshooting

**`ClusterSecretStore` not Valid:**
```bash
kubectl describe clustersecretstore doppler-backend
# Check the token is correct and has Read access in Doppler
```

**`ExternalSecret` not syncing:**
```bash
kubectl describe externalsecret <name> -n <namespace>
# Look for events at the bottom — shows exact error
```

**CRD version error on apply:**
```bash
# Check installed API version
kubectl api-resources | grep external-secrets
# Use v1 not v1beta1 for newer ESO versions
```

**Large CRD annotation error with ArgoCD:**
```yaml
# Add to ArgoCD Application syncOptions:
- ServerSideApply=true
- Replace=true
```

---

## Making ESO GitOps-Driven (ArgoCD)

Store your `ClusterSecretStore` and `ExternalSecret` manifests in Git
and create an ArgoCD Application pointing to them:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: eso-config
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/<user>/Kubernetes-infra
    targetRevision: main
    path: eso-config
  destination:
    server: https://kubernetes.default.svc
    namespace: external-secrets
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

ArgoCD manages the `ClusterSecretStore` and `ExternalSecrets` — only the
`doppler-token` secret remains manual.

---

## Notes

- `refreshInterval: 1h` — ESO re-syncs from Doppler every hour
- `ClusterSecretStore` works across all namespaces (vs `SecretStore` which is namespace-scoped)
- Doppler free tier supports unlimited secrets — perfect for homelab
- ESO is a Kubernetes CRD — `ExternalSecret` doesn't exist in vanilla Kubernetes

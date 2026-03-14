# TIL: App of Apps Pattern & Production GitOps Setup

> Date: 2026-03-14
> Tags: `kubernetes` `argocd` `helm` `gitops` `app-of-apps` `external-secrets`

---

## The Two-Repo Strategy

Separate concerns into two repos — industry standard pattern:

```
Kubernetes-learning/          ← App repo (Helm charts, app code)
    jellyfin-chart/
    custom-charts/
        cert-manager/
        external-dns/
        external-secrets/
        reflector/

Kubernetes-infra/             ← Infra repo (ArgoCD manifests, config)
    argocd-apps/
        cert-manager.yaml
        external-dns.yaml
        external-secrets.yaml
        reflector.yaml
        jellyfin.yaml
    bootstrap/
        app-of-apps.yaml
```

**Why two repos:**
- Developers touch app repo, ops touches infra repo
- Different permissions per repo
- Clean git history per concern
- Secrets config separate from application code

---

## Wrapper Charts

Instead of letting ArgoCD install community charts directly, wrap them
to own the `values.yaml` in Git:

```
custom-charts/external-dns/
├── Chart.yaml     ← declares community chart as dependency
└── values.yaml    ← YOUR values, fully in Git
```

```yaml
# Chart.yaml
apiVersion: v2
name: external-dns
version: 0.1.0
dependencies:
  - name: external-dns
    version: "1.15.0"
    repository: https://kubernetes-sigs.github.io/external-dns/
```

```yaml
# values.yaml — note: nested under the dependency name
external-dns:
  provider:
    name: cloudflare
  policy: sync
  sources:
    - ingress
```

**Key rule:** values must be nested under the dependency chart name.
Helm silently ignores values that aren't nested correctly.

Run after creating/updating Chart.yaml:
```bash
helm dependency update
helm template <name> . --namespace <ns>   # verify values are picked up
```

---

## App of Apps Pattern

One ArgoCD Application that manages all other Applications:

```yaml
# bootstrap/app-of-apps.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: app-of-apps
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/<user>/Kubernetes-infra
    targetRevision: main
    path: argocd-apps        # watches entire folder
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

ArgoCD reads `argocd-apps/` → finds all Application manifests → deploys them all.

### Bootstrap a new cluster — only 2 commands:
```bash
# 1. Install ArgoCD
helm install argocd argo/argo-cd -n argocd --create-namespace

# 2. Apply the parent app — everything else is automatic
kubectl apply -f bootstrap/app-of-apps.yaml
```

---

## ArgoCD Application Manifest

Standard template for each tool:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: cert-manager
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/<user>/Kubernetes-learning
    targetRevision: main
    path: custom-charts/cert-manager   # path to wrapper chart
  destination:
    server: https://kubernetes.default.svc
    namespace: cert-manager
  syncPolicy:
    automated:
      prune: true       # delete resources removed from Git
      selfHeal: true    # revert manual cluster changes
    syncOptions:
      - CreateNamespace=true
```

---

## Fix: ServerSideApply for Large CRDs

**Error:**
```
metadata.annotations: Too long: may not be more than 262144 bytes
```

**Cause:** ArgoCD uses client-side apply by default, which stores the full
manifest in an annotation. Large CRDs (ESO, cert-manager) exceed the 262144
byte limit.

**Fix:** Add `ServerSideApply=true` to syncOptions:
```yaml
syncOptions:
  - CreateNamespace=true
  - ServerSideApply=true
```

If the app was already installed with client-side apply, delete and recreate it:
```bash
kubectl delete application external-secrets -n argocd
kubectl apply -f argocd-apps/external-secrets.yaml
```

If still failing, also add `Replace=true`:
```yaml
syncOptions:
  - CreateNamespace=true
  - ServerSideApply=true
  - Replace=true
```

---

## Fix: Chart Version Mismatch

If a tool was already installed manually before ArgoCD took over,
the chart version in your wrapper must match what's running:

```bash
# Check what's installed
helm list -n <namespace>

# Update Chart.yaml to match
dependencies:
  - name: external-secrets
    version: "2.1.0"    ← must match installed version

# Re-download dependency
helm dependency update
git add . && git commit -m "fix chart version" && git push
```

---

## Ignore .tgz Files in Git

Chart dependencies are downloaded by `helm dependency update` — never commit them:

```bash
# .gitignore
**/charts/*.tgz
```

ArgoCD runs `helm dependency update` automatically during sync.

---

## Useful Commands

```bash
# List all ArgoCD applications
kubectl get application -n argocd

# Force immediate sync (don't wait for polling)
kubectl annotate application <app-name> \
  argocd.argoproj.io/refresh=normal -n argocd

# Delete app but keep deployed resources
kubectl delete application <app-name> -n argocd

# Verify helm template renders with your values
helm template <name> ./custom-charts/<name> --namespace <ns>
```

# TIL: ArgoCD GitOps with Helm

> Date: 2026-03-08
> Tags: `kubernetes` `argocd` `helm` `gitops`

---

## What is GitOps?

Git is the single source of truth. Instead of running `helm upgrade` manually,
you push to Git and ArgoCD applies the changes automatically.

```
Git commit → ArgoCD detects → helm upgrade → cluster updated ✅
```

---

## ArgoCD Application Manifest

Create this file and apply it once — ArgoCD handles everything after that:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: jellyfin-argo
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/<user>/<repo>
    targetRevision: main        # branch to watch
    path: jellyfin-chart        # path to helm chart inside repo
    helm:
      valueFiles:
        - values.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: jellyfin-argo    # namespace to deploy into
  syncPolicy:
    automated:
      prune: true       # delete resources removed from Git
      selfHeal: true    # revert manual changes that drift from Git
    syncOptions:
      - CreateNamespace=true    # create namespace if it doesn't exist
```

```bash
kubectl apply -f jellyfin-argo-app.yaml
```

---

## Key Fields Explained

| Field | Purpose |
|---|---|
| `repoURL` | Your GitHub repo |
| `targetRevision` | Branch ArgoCD watches |
| `path` | Where the Helm chart lives in the repo |
| `destination.namespace` | Where to deploy |
| `prune: true` | Resources deleted from Git get deleted from cluster |
| `selfHeal: true` | Manual cluster changes get reverted to match Git |
| `CreateNamespace=true` | No need to pre-create the namespace |

---

## Verify Deployment

```bash
# Check application status
kubectl get application -n argocd

# Check deployed resources
kubectl get all -n jellyfin-argo
```

Healthy output:
```
NAME            SYNC STATUS   HEALTH STATUS
jellyfin-argo   Synced        Healthy
```

---

## Testing GitOps Sync

**Test 1 — Git commit triggers deploy:**
1. Edit `values.yaml` in GitHub (e.g. change `replicaCount: 1` to `2`)
2. Commit to `main`
3. Watch ArgoCD apply it:
```bash
kubectl get pods -n jellyfin-argo -w
```
New pod appears automatically — no kubectl or helm needed ✅

---

**Test 2 — selfHeal reverts manual changes:**
```bash
# Manually break something
kubectl scale deployment jellyfin-argo --replicas=1 -n jellyfin-argo

# Watch ArgoCD revert it back to match Git
kubectl get pods -n jellyfin-argo -w
```
ArgoCD detects the drift and restores it within seconds ✅

---

## The Full Picture

```
Developer pushes to Git
        ↓
ArgoCD detects change (polls every 3 min or via webhook)
        ↓
ArgoCD renders Helm templates with values
        ↓
ArgoCD applies to cluster
        ↓
If someone manually changes cluster → selfHeal reverts it
```

---

## Useful Commands

```bash
# List all ArgoCD applications
kubectl get application -n argocd

# Force a manual sync (don't wait for polling)
kubectl annotate application jellyfin-argo \
  argocd.argoproj.io/refresh=normal -n argocd

# Delete an application (won't delete deployed resources by default)
kubectl delete application jellyfin-argo -n argocd
```

---

## Notes

- ArgoCD polls Git every 3 minutes by default — add a webhook for instant sync
- `prune: true` is powerful but dangerous — a bad Git commit can delete production resources
- Each namespace needs its own `Application` manifest with different `destination.namespace`
- Secrets are NOT managed by ArgoCD by default — use Reflector or ESO for that

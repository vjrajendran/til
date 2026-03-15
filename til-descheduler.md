# TIL: Kubernetes Descheduler

> Date: 2026-03-15
> Tags: `kubernetes` `descheduler` `cluster-operations` `scheduling`

---

## The Problem

Kubernetes scheduler places pods **once** when created — it never moves them.
Over time the cluster becomes unbalanced:

```
Node 1: 70% full
Node 2: 70% full
Node 3: 10% full  ← new node added, pods never move here
```

Descheduler fixes this by periodically evicting pods from bad positions
so the scheduler can replace them optimally.

---

## How It Works

Descheduler runs as a **CronJob** — not a permanent pod:

```
On schedule:
      ↓
Descheduler wakes up
      ↓
Evaluates all pods against policies
      ↓
Evicts pods in bad positions
      ↓
Kubernetes scheduler replaces them on better nodes ✅
      ↓
Descheduler goes back to sleep
```

---

## Install (Wrapper Chart)

```yaml
# Chart.yaml
apiVersion: v2
name: descheduler
version: 0.1.0
dependencies:
  - name: descheduler
    version: "0.31.0"
    repository: https://kubernetes-sigs.github.io/descheduler
```

---

## Production values.yaml

```yaml
descheduler:
  # Run at 1PM every Tuesday and Wednesday
  # Conservative schedule — not every 30 mins like dev
  schedule: "0 13 * * 2,4"

  # Run as non-root — security best practice
  podSecurityContext:
    fsGroup: 1000
    seccompProfile:
      type: RuntimeDefault
  securityContext:
    runAsGroup: 1000

  nodeSelector:
    kubernetes.io/os: linux

  deschedulerPolicy:
    profiles:
      - name: default
        pluginConfig:
          - name: DefaultEvictor
            args:
              ignorePvcPods: true
              evictLocalStoragePods: true
          - name: RemoveDuplicates
            args: {}
          - name: RemovePodsHavingTooManyRestarts
            args:
              podRestartThreshold: 100
              includingInitContainers: true
          - name: RemovePodsViolatingNodeAffinity
            args:
              nodeAffinityType:
                - requiredDuringSchedulingIgnoredDuringExecution
          - name: RemovePodsViolatingNodeTaints
            args: {}
          - name: RemovePodsViolatingInterPodAntiAffinity
            args: {}
          - name: RemovePodsViolatingTopologySpreadConstraint
            args: {}
          - name: LowNodeUtilization
            args:
              thresholds:
                cpu: 20
                memory: 30
                pods: 30
              targetThresholds:
                cpu: 50
                memory: 75
                pods: 50
        plugins:
          balance:
            enabled:
              - LowNodeUtilization
              - RemoveDuplicates
              - RemovePodsViolatingTopologySpreadConstraint
          deschedule:
            enabled:
              - RemovePodsHavingTooManyRestarts
              - RemovePodsViolatingNodeAffinity
              - RemovePodsViolatingNodeTaints
              - RemovePodsViolatingInterPodAntiAffinity
```

---

## All Plugins Explained

### Balance plugins — fix node distribution

| Plugin | What it does |
|---|---|
| `LowNodeUtilization` | Moves pods from busy nodes to idle ones |
| `RemoveDuplicates` | Spreads duplicate pods across nodes |
| `RemovePodsViolatingTopologySpreadConstraint` | Rebalances pods across zones/regions |

### Deschedule plugins — fix individual pod problems

| Plugin | What it does |
|---|---|
| `RemovePodsHavingTooManyRestarts` | Evicts crash-looping pods (100+ restarts) |
| `RemovePodsViolatingNodeAffinity` | Evicts pods that no longer match node affinity rules |
| `RemovePodsViolatingNodeTaints` | Evicts pods on tainted nodes they shouldn't be on |
| `RemovePodsViolatingInterPodAntiAffinity` | Fixes pods that violate anti-affinity rules |

### Safety plugin

| Plugin | What it does |
|---|---|
| `DefaultEvictor` | Guards — controls what can/can't be evicted |

---

## LowNodeUtilization Thresholds

```
thresholds       → nodes BELOW this % are targets (underutilized)
targetThresholds → nodes ABOVE this % are sources (overutilized)
```

Example with production values:
```
Node at 80% CPU → overutilized (above targetThreshold 50%) → pods evicted FROM here
Node at 10% CPU → underutilized (below threshold 20%)      → pods moved TO here
```

---

## ⚠️ Every Plugin Needs Config

Every plugin in `plugins.balance.enabled` or `plugins.deschedule.enabled`
**must** have a corresponding entry in `pluginConfig`. Use `args: {}` for
plugins with no required arguments.

Missing config → plugin skipped silently with this error:
```
unable to build LowNodeUtilization plugin: unable to find "LowNodeUtilization" plugin config
```

---

## Testing

```bash
# Trigger manually instead of waiting for cron
kubectl create job descheduler-test \
  --from=cronjob/descheduler \
  -n descheduler

# Watch logs
kubectl logs -n descheduler -l job-name=descheduler-test -f
```

Successful output:
```
"Number of evicted pods" totalEvicted=7
```

---

## Verify Config is Applied

If changes aren't taking effect, check what's actually deployed:
```bash
kubectl describe configmap descheduler -n descheduler
```

Shows the exact policy running — compare against your `values.yaml`.

---

## Schedule Reference

```
"*/30 * * * *"    → every 30 minutes (dev/testing)
"0 13 * * 2,4"   → 1PM every Tue & Wed (production)
"0 2 * * *"      → 2AM every day (off-hours)
```

---

## ArgoCD Application

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: descheduler
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/vjrajendran/Kubernetes-learning
    targetRevision: main
    path: custom-charts/descheduler
  destination:
    server: https://kubernetes.default.svc
    namespace: descheduler
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

---

## Notes

- Descheduler only **evicts** — it doesn't place pods. The scheduler handles placement.
- `ignorePvcPods: true` — never evict pods with persistent storage (databases etc.)
- `context canceled` error at job end is harmless — job finished before event could be written
- Production schedule should be off-peak hours to minimise disruption
- On small clusters evictions may be minimal — bigger impact on large multi-node clusters

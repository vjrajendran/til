# TIL: Auto-Mirror Secrets Across Namespaces with Reflector

> Date: 2026-03-08
> Tags: `kubernetes` `reflector` `secrets` `namespaces` `helm`

---

## The Problem

Every new namespace needs secrets copied manually. Reflector solves this by
automatically mirroring secrets to namespaces — existing and new ones.

---

## Install

```bash
helm repo add emberstack https://emberstack.github.io/helm-charts
helm repo update

kubectl create namespace reflector

helm upgrade --install reflector emberstack/reflector \
  --namespace reflector
```

---

## Annotate the Source Secret

Both annotation pairs are required — missing either one silently does nothing:

```bash
kubectl annotate secret <secret-name> \
  reflector.v1.k8s.emberstack.com/reflection-allowed="true" \
  reflector.v1.k8s.emberstack.com/reflection-allowed-namespaces=".*" \
  reflector.v1.k8s.emberstack.com/reflection-auto-enabled="true" \
  reflector.v1.k8s.emberstack.com/reflection-auto-namespaces=".*" \
  -n <source-namespace> --overwrite
```

### What each pair does

| Annotation | Purpose |
|---|---|
| `reflection-allowed` | **Permission** — unlocks the secret to be mirrored |
| `reflection-allowed-namespaces` | Which namespaces are allowed to receive it |
| `reflection-auto-enabled` | **Action** — automatically pushes to namespaces |
| `reflection-auto-namespaces` | Which namespaces to push to |

Use `.*` to match all namespaces, or be explicit: `"jellyfin,jellyfin-helm,jellyfin-argo"`

---

## Verify It's Working

```bash
# Existing namespace
kubectl get secret <secret-name> -n <target-namespace>

# Real test — create a brand new namespace
kubectl create namespace reflector-test
kubectl get secret <secret-name> -n reflector-test
# Should appear automatically ✅
```

---

## Troubleshooting

**Secret not reflecting — check the logs:**
```bash
kubectl logs -n reflector deployment/reflector | tail -20
```

**No activity in logs at all?**
Most likely missing one of the annotation pairs. Both `reflection-allowed`
and `reflection-auto-enabled` are required together.

**Restart to force a re-sync:**
```bash
kubectl rollout restart deployment/reflector -n reflector
```

---

## Notes

- Reflector watches for new namespaces and mirrors automatically — no action needed
- Keep `reflection-allowed-namespaces` explicit for sensitive secrets instead of `.*`
- The Helm repo may lag behind GitHub releases — check GitHub for latest if issues arise

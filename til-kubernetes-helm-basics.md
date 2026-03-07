# TIL: Helm Basics & Kubernetes Gotchas

> Date: 2026-03-07  
> Tags: `kubernetes` `helm` `secrets` `debugging`

---

## Helm Key Concepts

Helm is a package manager for Kubernetes. One chart, deploy anywhere by passing different values.

```yaml
# values.yaml — your config knobs
replicaCount: 1
image:
  repository: myrepo/myimage
  tag: "1.0"
```

```yaml
# template — reads from values
image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
replicas: {{ .Values.replicaCount }}
namespace: {{ .Release.Namespace }}  # comes from --namespace flag
name: {{ .Release.Name }}            # comes from helm install <name>
```

For lists/maps from values.yaml, always use `toYaml | nindent`:
```yaml
imagePullSecrets:
  {{- toYaml .Values.imagePullSecrets | nindent 8 }}
```

---

## Copy a Secret Between Namespaces

```bash
kubectl get secret <secret-name> -n <source> -o yaml \
  | sed 's/namespace: <source>/namespace: <target>/' \
  | kubectl apply -f -
```

---

## Private Image Runs Without Secret — Until It Doesn't

**The trap:** Pod runs fine without `imagePullSecrets` on a private image because the image is cached on the node from a previous deployment.

**It breaks when** the pod reschedules to a fresh node that never pulled the image → `ErrImagePull`.

**Fix:** Always create the secret in every namespace that needs it.
```bash
kubectl create secret docker-registry dockerhub-secret \
  --docker-username=YOUR_USERNAME \
  --docker-password=YOUR_PASSWORD \
  --namespace <namespace>
```

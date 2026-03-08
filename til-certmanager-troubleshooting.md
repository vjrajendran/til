# TIL: cert-manager Certificate Troubleshooting

> Date: 2026-03-07
> Tags: `kubernetes` `cert-manager` `traefik` `tls` `helm` `dns` `troubleshooting`

---

## How cert-manager Issues a Certificate (HTTP01)

Understanding this flow is key to debugging:

1. You create an `Ingress` with `cert-manager.io/cluster-issuer` annotation
2. cert-manager creates a `Certificate` → `CertificateRequest` → `Order` → `Challenge`
3. cert-manager spins up a temporary `cm-acme-http-solver` ingress
4. Let's Encrypt hits `http://<your-domain>/.well-known/acme-challenge/<token>`
5. If reachable → domain verified → certificate issued → stored as a `Secret`

**If DNS doesn't point to your cluster, step 4 fails. Full stop.**

---

## Debugging Commands (run in order)

```bash
# 1. Check the certificate state
kubectl get certificate -n <namespace>

# 2. See the full chain — challenge, order, certificate request
kubectl get challenge,order,certificaterequest -n <namespace>

# 3. Describe everything for detailed errors
kubectl describe challenge,order,certificaterequest -n <namespace>

# 4. Check if the http-solver ingress has an address
kubectl get ingress -n <namespace>

# 5. Check cert-manager logs
kubectl logs -n cert-manager deployment/cert-manager | tail -20
```

---

## Error: NXDOMAIN

```
400 urn:ietf:params:acme:error:dns: DNS problem: NXDOMAIN looking up A for <domain>
```

**Cause:** DNS record doesn't exist — Let's Encrypt can't resolve your domain.

**Fix:** Add an A record at your DNS provider:

| Type | Name | Value |
|---|---|---|
| A | `<subdomain>` | `<cluster external IP>` |

Find your cluster's external IP:
```bash
kubectl get svc -n kube-system | grep traefik
# Look for EXTERNAL-IP column
```

Verify DNS is live before retrying:
```bash
nslookup <your-domain>
# Must return your cluster IP
```

---

## Challenge Stuck / Not Retrying

If the challenge failed and won't retry on its own:

```bash
# Step 1 — delete the challenge
kubectl delete challenge -n <namespace> --all

# Step 2 — if still stuck, delete the full certificate chain
kubectl delete certificaterequest -n <namespace> --all
kubectl delete certificate <cert-name> -n <namespace>

# Step 3 — force Helm to recreate everything
helm upgrade --install <release> ./chart --namespace <namespace>
```

---

## Certificate Flow — What Each Resource Means

| Resource | What it is |
|---|---|
| `Certificate` | The goal — "I want a cert for this domain" |
| `CertificateRequest` | The actual CSR sent to Let's Encrypt |
| `Order` | Let's Encrypt's order for the certificate |
| `Challenge` | The HTTP01 verification step |
| `Secret` | Where the final cert+key is stored once issued |

They are all linked — deleting the `Certificate` cascades and recreates the whole chain.

---

## Success Indicators

```bash
kubectl get certificate -n <namespace>
# NAME          READY   SECRET        AGE
# jellyfin-tls  True    jellyfin-tls  2m   ← True means cert issued ✅
```

```bash
kubectl describe challenge -n <namespace>
# Reason: Successfully authorized domain  ✅
# State:  valid                           ✅
```

---

## Notes

- cert-manager retries failed challenges automatically, but slowly. Deleting resources forces an immediate retry.
- The `cm-acme-http-solver` ingress is temporary — cert-manager creates and deletes it automatically.
- On Civo, the external IP comes from the LoadBalancer assigned to Traefik.
- Always verify DNS with `nslookup` before assuming cert-manager is broken.

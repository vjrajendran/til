# TIL: Inspecting Container Images with Skopeo
> Date: 2026-03-13
> Tags: `containers` `skopeo` `docker` `registry` `images` `troubleshooting`

---

## What is Skopeo?

Skopeo is a CLI tool for working with container images and registries — without needing a Docker daemon or pulling the full image.

**Key advantage:** You can inspect tags, digests, and metadata remotely. Fast and lightweight.

---

## Install

```bash
# Ubuntu/Debian
sudo apt install skopeo

# Fedora/RHEL
sudo dnf install skopeo

# macOS
brew install skopeo
```

---

## Common Commands (run in order for debugging)

```bash
# 1. Inspect an image — returns metadata, labels, digest
skopeo inspect docker://nginx:latest

# 2. List all available tags for an image
skopeo list-tags docker://nginx

# 3. Check a specific registry image
skopeo inspect docker://ghcr.io/<org>/<image>:<tag>

# 4. Inspect a private registry (with credentials)
skopeo inspect --creds username:password docker://my-registry.com/<image>:<tag>

# 5. Copy an image between registries (no pull required)
skopeo copy docker://source-registry/<image>:<tag> docker://dest-registry/<image>:<tag>
```

---

## Get Image Digest

Useful for pinning exact versions in Helm charts or Kubernetes manifests:

```bash
skopeo inspect docker://nginx:latest | jq '.Digest'
# "sha256:abc123..."
```

Then use in manifests:
```yaml
image: nginx@sha256:abc123...
```

---

## Error: unauthorized

```
FATA[0000] Error parsing image name "docker://my-registry.com/image:tag":
unauthorized: authentication required
```

**Cause:** Registry requires authentication.

**Fix:**
```bash
skopeo inspect --creds <username>:<password> docker://my-registry.com/<image>:<tag>

# Or use a credentials file
skopeo inspect --authfile ~/.docker/config.json docker://my-registry.com/<image>:<tag>
```

---

## Error: manifest unknown

```
FATA[0000] Error parsing image name: manifest unknown
```

**Cause:** The tag doesn't exist in the registry.

**Fix:** List available tags first:
```bash
skopeo list-tags docker://<registry>/<image>
```

---

## Resource Comparison

| Tool | Needs Daemon | Pulls Full Image | List Tags | Copy Between Registries |
|---|---|---|---|---|
| `skopeo` | ❌ | ❌ | ✅ | ✅ |
| `docker pull` | ✅ | ✅ | ❌ | ❌ |
| `crane` | ❌ | ❌ | ✅ | ✅ |

---

## Success Indicators

```bash
skopeo inspect docker://nginx:latest | jq '.Tag, .Digest'
# "latest"
# "sha256:abc123..."   ← image found and inspected ✅
```

```bash
skopeo list-tags docker://nginx | jq '.Tags | length'
# 100+   ← tags listed successfully ✅
```

---

## Notes

- Skopeo works with any OCI-compliant registry: Docker Hub, ECR, GCR, Quay, GHCR.
- No daemon means it's safe and fast to use in CI/CD pipelines.
- Prefer digest pinning over tags in production — tags are mutable, digests are not.
- `--authfile` accepts the standard `~/.docker/config.json` so existing Docker credentials just work.

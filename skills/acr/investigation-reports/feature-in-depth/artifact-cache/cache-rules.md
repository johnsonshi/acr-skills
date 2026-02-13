# ACR Artifact Cache - Cache Rules Reference

## Overview

Cache rules define the mapping between upstream repositories and your Azure Container Registry namespaces. This document provides a comprehensive reference for creating and managing cache rules.

## Cache Rule Components

Every cache rule consists of four essential parts:

| Component | Description | Constraints |
|-----------|-------------|-------------|
| **Rule Name** | Unique identifier for the cache rule | Must be unique within the registry |
| **Source Registry** | The upstream registry to cache from | Must be a supported upstream |
| **Source Repository Path** | The full path to the upstream repository | Registry-specific format |
| **Target Repository Namespace** | Local ACR path for cached images | Cannot already exist in ACR |

## Source Registries and Path Formats

### Docker Hub

**Login Server:** `docker.io`

| Image Type | Source Repository Path Format | Example |
|------------|-------------------------------|---------|
| Official Images | `docker.io/library/<image>` | `docker.io/library/nginx` |
| User Images | `docker.io/<user>/<image>` | `docker.io/bitnami/redis` |

**Notes:**
- Credentials are **always required** for Docker Hub
- If you omit `library/` for official images, artifact cache adds it automatically
- Use Docker Hub access tokens instead of passwords

### Microsoft Artifact Registry (MCR)

**Login Server:** `mcr.microsoft.com`

| Source Repository Path Format | Example |
|-------------------------------|---------|
| `mcr.microsoft.com/<namespace>/<image>` | `mcr.microsoft.com/dotnet/runtime` |
| `mcr.microsoft.com/<image>` | `mcr.microsoft.com/azure-cli` |

**Notes:**
- Unauthenticated pulls **only** (no credentials supported)
- Public Microsoft images only

### GitHub Container Registry

**Login Server:** `ghcr.io`

| Source Repository Path Format | Example |
|-------------------------------|---------|
| `ghcr.io/<owner>/<image>` | `ghcr.io/actions/runner` |

**Notes:**
- Supports both authenticated and unauthenticated pulls
- Use Personal Access Token (PAT) for authentication
- Public images don't require credentials

### Quay.io

**Login Server:** `quay.io`

| Source Repository Path Format | Example |
|-------------------------------|---------|
| `quay.io/<namespace>/<image>` | `quay.io/coreos/etcd` |

**Notes:**
- Supports both authenticated and unauthenticated pulls
- Public images don't require credentials

### AWS ECR Public Gallery

**Login Server:** `public.ecr.aws`

| Source Repository Path Format | Example |
|-------------------------------|---------|
| `public.ecr.aws/<alias>/<image>` | `public.ecr.aws/amazonlinux/amazonlinux` |

**Notes:**
- Unauthenticated pulls **only**
- No credentials supported

### Kubernetes Registry

**Login Server:** `registry.k8s.io`

| Source Repository Path Format | Example |
|-------------------------------|---------|
| `registry.k8s.io/<component>` | `registry.k8s.io/pause` |
| `registry.k8s.io/<namespace>/<component>` | `registry.k8s.io/ingress-nginx/controller` |

**Notes:**
- Supports both authenticated and unauthenticated pulls
- Available via Azure CLI only

### Google Artifact Registry

**Login Server:** `*.pkg.dev` (region-specific)

| Source Repository Path Format | Example |
|-------------------------------|---------|
| `<region>-docker.pkg.dev/<project>/<repo>/<image>` | `us-docker.pkg.dev/myproject/myrepo/myimage` |

**Notes:**
- Authenticated pulls **only**
- Use Service Account Key for authentication
- Available via Azure CLI only

### Legacy Google Container Registry

**Login Server:** `gcr.io` or region-specific (`us.gcr.io`, etc.)

| Source Repository Path Format | Example |
|-------------------------------|---------|
| `gcr.io/<project>/<image>` | `gcr.io/google-samples/hello-app` |
| `<region>.gcr.io/<project>/<image>` | `us.gcr.io/my-project/my-image` |

**Notes:**
- Supports both authenticated and unauthenticated pulls
- Available via Azure CLI only

## Creating Cache Rules

### CLI Syntax

```bash
az acr cache create \
  --registry <registry-name> \
  --name <rule-name> \
  --source-repo <source-repository-path> \
  --target-repo <target-namespace> \
  [--cred-set <credential-set-name>]
```

### Examples by Registry

**Docker Hub (with credentials):**
```bash
az acr cache create -r MyRegistry -n nginx-cache \
  -s docker.io/library/nginx -t nginx -c MyDockerHubCreds
```

**MCR (unauthenticated):**
```bash
az acr cache create -r MyRegistry -n dotnet-cache \
  -s mcr.microsoft.com/dotnet/sdk -t dotnet/sdk
```

**GitHub Container Registry (unauthenticated):**
```bash
az acr cache create -r MyRegistry -n actions-runner \
  -s ghcr.io/actions/runner -t github/actions-runner
```

**Quay.io (unauthenticated):**
```bash
az acr cache create -r MyRegistry -n etcd-cache \
  -s quay.io/coreos/etcd -t coreos/etcd
```

## Cache Rule Limits

| Limit | Value | Notes |
|-------|-------|-------|
| Maximum cache rules per registry | 1,000 | Delete unused rules if limit reached |
| Rule name uniqueness | Must be unique | Within single registry |
| Target repository existence | Must NOT exist | Cannot cache to existing repo |

## Cache Rule Behavior

### First Pull (Cache Population)

When you pull an image through a cache rule for the first time:

1. ACR receives the pull request
2. ACR evaluates matching cache rules
3. ACR authenticates to upstream (if credentials configured)
4. ACR fetches the image from upstream
5. ACR stores the image locally
6. ACR returns the image to the client

### Subsequent Pulls (Cache Hit)

When you pull a previously cached image:

1. ACR receives the pull request
2. ACR evaluates matching cache rules
3. ACR finds the image in local cache
4. ACR returns the image directly (no upstream call)

### Tag Updates

**Important**: Cache does NOT automatically update when upstream tags change.

- If `nginx:latest` is cached and Docker Hub updates `latest`, your cache still has the old version
- You must explicitly pull the tag again to get the updated version
- Consider using specific version tags (e.g., `nginx:1.25.3`) for reproducibility

## Cache Rule Resolution

### Static vs Wildcard Rules

When multiple rules could match a pull request:

1. **Static rules** take precedence over wildcard rules
2. More specific rules match before broader rules
3. First matching rule is used

### Example Resolution

Given these rules:
- Rule A: `contoso.azurecr.io/*` => `mcr.microsoft.com/*`
- Rule B: `contoso.azurecr.io/library/dotnet` => `docker.io/library/dotnet`

| Pull Request | Matching Rule | Why |
|--------------|---------------|-----|
| `contoso.azurecr.io/library/dotnet:8.0` | Rule B | Static rule takes precedence |
| `contoso.azurecr.io/azure-cli:latest` | Rule A | Only wildcard matches |

## Managing Cache Rules

### List Cache Rules

```bash
az acr cache list --registry MyRegistry
```

### Show Cache Rule Details

```bash
az acr cache show --registry MyRegistry --name nginx-cache
```

### Update Cache Rule

Update credential association:
```bash
az acr cache update --registry MyRegistry --name nginx-cache --cred-set NewCredSet
```

Remove credentials:
```bash
az acr cache update --registry MyRegistry --name nginx-cache --remove-cred-set
```

### Delete Cache Rule

```bash
az acr cache delete --registry MyRegistry --name nginx-cache
```

## Common Cache Rule Patterns

### Pattern 1: Mirror Entire Upstream Registry

```bash
# Cache all MCR images
az acr cache create -r MyRegistry -n mcr-all \
  -s "mcr.microsoft.com/*" -t "mcr/*"
```

### Pattern 2: Cache Specific Namespace

```bash
# Cache all .NET images
az acr cache create -r MyRegistry -n dotnet-all \
  -s "mcr.microsoft.com/dotnet/*" -t "dotnet/*"
```

### Pattern 3: Namespace Reorganization

```bash
# Cache Docker Hub to vendor-prefixed namespace
az acr cache create -r MyRegistry -n nginx-vendor \
  -s docker.io/library/nginx -t vendor/nginx -c DockerCreds

az acr cache create -r MyRegistry -n redis-vendor \
  -s docker.io/library/redis -t vendor/redis -c DockerCreds
```

### Pattern 4: Multi-Registry Caching

```bash
# Docker Hub images
az acr cache create -r MyRegistry -n dh-nginx \
  -s docker.io/library/nginx -t dockerhub/nginx -c DockerCreds

# MCR images
az acr cache create -r MyRegistry -n mcr-dotnet \
  -s mcr.microsoft.com/dotnet/runtime -t mcr/dotnet/runtime

# GitHub images
az acr cache create -r MyRegistry -n gh-actions \
  -s ghcr.io/actions/runner -t github/actions-runner
```

## Best Practices

1. **Use descriptive rule names** that indicate source and purpose
2. **Plan namespace structure** before creating rules
3. **Document your cache rules** for team reference
4. **Use wildcards strategically** to reduce rule count
5. **Monitor rule usage** to identify unused rules
6. **Test rules** with manual pulls before automation
7. **Consider tag strategy** - specific tags vs latest
8. **Review credentials periodically** for expiration

## Troubleshooting Cache Rules

| Issue | Possible Cause | Solution |
|-------|---------------|----------|
| Rule creation fails | Target repo exists | Choose different target namespace |
| Rule creation fails | Overlapping wildcard | Review existing wildcard rules |
| Rule creation fails | Limit reached | Delete unused rules |
| Pulls fail | Invalid credentials | Check Key Vault secrets |
| Pulls fail | Key Vault access denied | Verify RBAC assignment |
| Cached image outdated | Manual refresh needed | Pull tag again |

## References

- [Artifact Cache Overview](https://learn.microsoft.com/azure/container-registry/artifact-cache-overview)
- [Wildcard Support](https://learn.microsoft.com/azure/container-registry/wildcards-artifact-cache)
- [CLI Reference - az acr cache](https://docs.microsoft.com/cli/azure/acr/cache)

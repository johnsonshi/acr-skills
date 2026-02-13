# ACR Tagging and Versioning - Best Practices

## Overview

A well-designed tagging strategy is essential for managing container images effectively. This guide covers recommended approaches for tagging and versioning container images in Azure Container Registry.

## Tagging Strategies

### 1. Stable Tags

**Definition**: Tags that are reused and updated over time (e.g., `:latest`, `:1.0`, `:v2`)

**Recommended Use**: Base images for container builds

**Not Recommended For**: Production deployments

#### How Stable Tags Work
- Tag points to the "latest" stable version
- Content changes over time
- Pulling by tag may get different content

#### Example Pattern
```
myimage:1      # Major version, always latest 1.x
myimage:1.0    # Minor version, always latest 1.0.x
myimage:latest # Always the newest release
```

#### Stable Tag Example
```bash
# Framework team releases version 1.0
docker push myregistry.azurecr.io/framework:1.0

# Later, security patch is applied
docker push myregistry.azurecr.io/framework:1.0  # Same tag, new content

# Major version tag updated
docker push myregistry.azurecr.io/framework:1    # Points to latest 1.x
```

### 2. Unique Tags

**Definition**: A different tag for each image pushed (never reused)

**Recommended Use**: Production deployments

#### Why Unique Tags for Deployments?
- Guarantees consistent version across all nodes
- Prevents accidental deployment of newer versions
- Enables reliable rollback
- Supports immutability guarantees

#### Unique Tag Patterns

| Pattern | Example | Pros | Cons |
|---------|---------|------|------|
| **Date-time stamp** | `myimage:20231215-143000` | Clear build time | Hard to correlate with source |
| **Git commit** | `myimage:abc123def` | Ties to source code | Doesn't reflect base image updates |
| **Build ID** | `myimage:build-12345` | Correlates to CI/CD | Requires CI/CD integration |
| **Semantic + Build** | `myimage:1.0.0-build-12345` | Version + traceability | Longer tags |
| **Manifest digest** | `myimage@sha256:...` | Cryptographically unique | Hard to read |

#### Best Practice: Combine Patterns
```bash
# Semantic version + build ID + Git SHA
docker tag myimage myregistry.azurecr.io/myimage:v1.2.3-build-456-abc123

# Or use build system prefix
docker tag myimage myregistry.azurecr.io/myimage:jenkins-456
docker tag myimage myregistry.azurecr.io/myimage:azdo-789
```

## Managing Untagged Manifests

### The Orphan Problem

When you push with a stable tag:
1. New image gets the tag
2. Old image becomes untagged (orphaned)
3. Orphaned images consume storage

```bash
# Push v1
docker push myregistry.azurecr.io/myimage:latest  # Creates manifest A

# Push v2 (overwrites tag)
docker push myregistry.azurecr.io/myimage:latest  # Creates manifest B
# Manifest A is now untagged but still exists!
```

### Solutions for Orphans

#### 1. Retention Policy (Premium)
```bash
az acr config retention update \
  --registry myregistry \
  --status enabled \
  --days 7 \
  --type UntaggedManifests
```

#### 2. ACR Purge
```bash
# Delete untagged manifests older than 7 days
az acr run \
  --cmd "acr purge --filter 'myrepo:.*' --untagged --ago 7d" \
  --registry myregistry \
  /dev/null
```

#### 3. Manual Cleanup
```bash
# List untagged manifests
az acr manifest list-metadata \
  --name myrepo \
  --registry myregistry \
  --query "[?tags==null].digest" -o tsv
```

## Lock Deployed Tags

**Best Practice**: Lock production images after deployment.

```bash
# After successful deployment
az acr repository update \
    --name myregistry \
    --image myapp:v1.2.3-build-456 \
    --write-enabled false
```

Benefits:
- Prevents accidental overwrite
- Survives aggressive purge policies
- Ensures deployment consistency

## Repository Namespaces

Organize images with namespaces:

```
contoso.azurecr.io/
├── base/
│   ├── dotnet:6.0
│   └── node:18
├── products/
│   ├── webapp/
│   │   ├── frontend:v1.0
│   │   └── api:v2.3
│   └── mobile/
│       └── backend:v1.1
├── marketing/
│   └── campaign-2023/
│       └── promo:latest
└── dev/
    └── experimental:latest
```

### Namespace Patterns

| Pattern | Example | Use Case |
|---------|---------|----------|
| Team-based | `team-a/myimage` | Multi-team organizations |
| Environment-based | `dev/myimage`, `prod/myimage` | Environment separation |
| Product-based | `product-x/myimage` | Product isolation |
| Hierarchical | `products/webapp/frontend` | Complex organizations |

## Semantic Versioning

Follow [SemVer](https://semver.org/) for meaningful versions:

```
MAJOR.MINOR.PATCH
```

- **MAJOR**: Breaking changes
- **MINOR**: New features, backward compatible
- **PATCH**: Bug fixes, backward compatible

### Example Tags
```
myapp:1.0.0      # Initial release
myapp:1.0.1      # Bug fix
myapp:1.1.0      # New feature
myapp:2.0.0      # Breaking change
myapp:1          # Latest 1.x.x
myapp:1.1        # Latest 1.1.x
myapp:latest     # Latest release
```

## CI/CD Integration

### Azure DevOps Example
```yaml
variables:
  imageTag: $(Build.BuildId)-$(Build.SourceVersion)

steps:
- task: Docker@2
  inputs:
    containerRegistry: 'myACRConnection'
    repository: 'myapp'
    command: 'buildAndPush'
    tags: |
      $(imageTag)
      latest

- script: |
    az acr repository update \
      --name $(ACR_NAME) \
      --image myapp:$(imageTag) \
      --write-enabled false
  displayName: 'Lock deployed image'
```

### GitHub Actions Example
```yaml
- name: Build and push
  uses: docker/build-push-action@v3
  with:
    push: true
    tags: |
      myregistry.azurecr.io/myapp:${{ github.sha }}
      myregistry.azurecr.io/myapp:latest

- name: Lock image
  run: |
    az acr repository update \
      --name myregistry \
      --image myapp:${{ github.sha }} \
      --write-enabled false
```

## Summary: Recommended Strategy

### For Base Images
- Use stable tags (`:1.0`, `:latest`)
- Accept that content changes
- Set up automated base image updates

### For Application Images
1. **Tag with unique identifier**: `myapp:build-123-abc456`
2. **Optionally add stable tag**: `myapp:latest`
3. **Lock after deployment**: `--write-enabled false`
4. **Clean up regularly**: ACR Purge or retention policy

### Decision Matrix

| Use Case | Tag Strategy | Lock? | Purge? |
|----------|--------------|-------|--------|
| Development | Stable (`:dev`) | No | Yes |
| Staging | Unique per build | No | Yes |
| Production | Unique per release | Yes | No (locked) |
| Base images | Stable (`:1.0`) | No | Retention policy |

## Source Documentation

- `/submodules/azure-management-docs/articles/container-registry/container-registry-image-tag-version.md`
- `/submodules/azure-management-docs/articles/container-registry/container-registry-best-practices.md`
- `/submodules/azure-management-docs/articles/container-registry/container-registry-image-lock.md`

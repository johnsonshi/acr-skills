# ACR Image Locking (Immutability) - Comprehensive Guide

## Overview

Azure Container Registry allows you to lock images and repositories to prevent accidental deletion or modification. This is essential for production deployments where image immutability is required.

## Important Distinction

**Image Lock vs Registry Lock**:
- **Image Lock** (this document): Controls data operations (push, pull, delete) on specific images/repositories
- **Registry Lock** (Azure Resource Lock): Controls management operations (replications, registry deletion)

Registry resource locks (`az lock` commands) do NOT prevent image data modifications.

## Lockable Attributes

| Attribute | Description | Default |
|-----------|-------------|---------|
| `write-enabled` | Allow pushing new versions | `true` |
| `delete-enabled` | Allow deletion | `true` |
| `read-enabled` | Allow pulling | `true` |
| `list-enabled` | Allow listing tags | `true` |

## Use Cases

### 1. Immutable Production Images
Lock images after deployment to prevent changes:
- Prevents accidental overwrites
- Ensures consistency across nodes
- Supports compliance requirements

### 2. Protect from Deletion
Allow updates but prevent removal:
- Keep historical versions
- Maintain audit trail

### 3. Quarantine Images
Prevent pulling of suspicious images:
- Block distribution while investigating
- Security incident response

## CLI Operations

### View Current Attributes

#### Repository Attributes
```bash
az acr repository show \
    --name myregistry \
    --repository myrepo \
    --output jsonc
```

#### Image Attributes
```bash
az acr repository show \
    --name myregistry \
    --image myrepo:tag \
    --output jsonc
```

### Lock an Image by Tag

```bash
# Make image immutable (cannot be overwritten or deleted)
az acr repository update \
    --name myregistry \
    --image myrepo:tag \
    --write-enabled false
```

### Lock an Image by Digest

```bash
# First, find the digest
az acr manifest list-metadata \
    --name myrepo \
    --registry myregistry

# Lock by digest
az acr repository update \
    --name myregistry \
    --image myrepo@sha256:123456abcdefg \
    --write-enabled false
```

### Lock an Entire Repository

```bash
# Lock all images in repository
az acr repository update \
    --name myregistry \
    --repository myrepo \
    --write-enabled false
```

### Protect from Deletion Only

Allow updates but prevent deletion:

```bash
# Image level
az acr repository update \
    --name myregistry \
    --image myrepo:tag \
    --delete-enabled false \
    --write-enabled true

# Repository level
az acr repository update \
    --name myregistry \
    --repository myrepo \
    --delete-enabled false \
    --write-enabled true
```

### Prevent Read Operations

```bash
# Quarantine image (cannot be pulled)
az acr repository update \
    --name myregistry \
    --image myrepo:tag \
    --read-enabled false

# Quarantine entire repository
az acr repository update \
    --name myregistry \
    --repository myrepo \
    --read-enabled false
```

### Hide from Listings

```bash
az acr repository update \
    --name myregistry \
    --repository myrepo \
    --list-enabled false

# Query hidden tags
az acr repository show-manifests \
    --name myregistry \
    --repository myrepo \
    --query "[?listEnabled==null].tags" \
    --output table
```

## Unlocking Images

### Unlock a Tag
```bash
az acr repository update \
    --name myregistry \
    --image myrepo:tag \
    --delete-enabled true \
    --write-enabled true
```

### Unlock a Repository
```bash
az acr repository update \
    --name myregistry \
    --repository myrepo \
    --delete-enabled true \
    --write-enabled true
```

### Unlock a Manifest
If the manifest itself is locked, you need an additional command:

```bash
# Get the digest
digest=$(az acr manifest show-metadata \
    -r myregistry \
    -n "myrepo:tag" \
    --query digest -o tsv)

# Unlock the manifest
az acr repository update \
    --name myregistry \
    --image myrepo@$digest \
    --delete-enabled true \
    --write-enabled true
```

## Tag vs Manifest Locking

**Important**: Tag and manifest attributes are managed separately.

Setting `deleteEnabled=false` on a tag does NOT automatically set it on the manifest.

### Check Both Attributes
```bash
#!/bin/bash
registry="myregistry"
repo="myrepo"
tag="mytag"

# Check tag attributes
az acr manifest show-metadata -r $registry -n "$repo:$tag"

# Check manifest attributes
digest=$(az acr manifest show-metadata -r $registry -n "$repo:$tag" --query digest -o tsv)
az acr manifest show-metadata -r $registry -n "$repo@$digest"
```

### Lock Both Tag and Manifest
```bash
# Lock tag
az acr repository update \
    --name myregistry \
    --image myrepo:tag \
    --write-enabled false \
    --delete-enabled false

# Lock manifest
digest=$(az acr manifest show-metadata \
    -r myregistry \
    -n "myrepo:tag" \
    --query digest -o tsv)

az acr repository update \
    --name myregistry \
    --image myrepo@$digest \
    --write-enabled false \
    --delete-enabled false
```

## Integration with ACR Purge

**ACR Purge will NOT delete locked images.**

Images with `write-enabled: false` or `delete-enabled: false` are automatically excluded from purge operations.

This allows you to:
1. Lock deployed images
2. Run aggressive purge policies
3. Retain only production-critical images

## Best Practices

### 1. Lock After Deployment
Include locking in your release pipeline:
```bash
# After deployment succeeds
az acr repository update \
    --name myregistry \
    --image myrepo:$BUILD_TAG \
    --write-enabled false
```

### 2. Use Unique Tags
Combine locking with unique tags for complete immutability:
- Tag: `myimage:build-12345`
- Lock: `write-enabled: false`

### 3. Automate in CI/CD
```yaml
# Azure DevOps example
- script: |
    az acr repository update \
      --name $(ACR_NAME) \
      --image $(IMAGE_NAME):$(Build.BuildId) \
      --write-enabled false
  displayName: 'Lock deployed image'
```

### 4. Audit Lock Status
Regularly review lock status:
```bash
# List all repositories
repos=$(az acr repository list --name myregistry -o tsv)

for repo in $repos; do
    echo "Repository: $repo"
    az acr repository show --name myregistry --repository $repo \
        --query "changeableAttributes" -o jsonc
done
```

## Lock Scenarios Summary

| Scenario | write-enabled | delete-enabled | read-enabled |
|----------|---------------|----------------|--------------|
| Full lock (immutable) | `false` | `false` | `true` |
| Delete protection only | `true` | `false` | `true` |
| Quarantine (no pull) | `true` | `false` | `false` |
| Hidden (not listed) | - | - | - (list-enabled: false) |

## Source Documentation

- `/submodules/azure-management-docs/articles/container-registry/container-registry-image-lock.md`
- `/submodules/azure-management-docs/articles/container-registry/container-registry-image-tag-version.md`
- `/submodules/azure-management-docs/articles/container-registry/container-registry-auto-purge.md`

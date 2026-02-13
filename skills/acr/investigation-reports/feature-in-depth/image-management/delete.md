# ACR Image Deletion - Comprehensive Guide

## Overview

To maintain registry size and manage costs, you should periodically delete stale image data from Azure Container Registry. This guide covers all methods for deleting image data and managing registry storage.

## Understanding Delete Operations

### Storage Impact
- Deleting images recovers storage space used by **unique layers**
- **Shared layers** are not deleted until no manifests reference them
- Storage billing stops immediately, but physical cleanup is asynchronous

### Key Concepts

| Term | Description |
|------|-------------|
| **Repository** | Collection of images with the same name |
| **Tag** | Human-readable reference to an image |
| **Manifest** | JSON describing image layers and configuration |
| **Manifest Digest** | SHA-256 hash uniquely identifying a manifest |
| **Untagged Manifest** | Manifest with no associated tags (orphaned) |

## Deletion Methods

### 1. Delete Repository

Deletes all images, tags, manifests, and unique layers in a repository.

```bash
# Delete entire repository
az acr repository delete \
  --name myregistry \
  --repository acr-helloworld
```

**Effect**: Removes all images and frees all storage for unique layers.

### 2. Delete by Tag

Deletes a specific image and its unique layers.

```bash
# Delete image by tag
az acr repository delete \
  --name myregistry \
  --image acr-helloworld:latest
```

**Important**: This deletes:
- The specified tag
- The underlying manifest
- All other tags pointing to the same manifest
- All unique layers

### 3. Delete by Manifest Digest

Deletes a specific manifest regardless of tags.

```bash
# First, list manifests to find the digest
az acr manifest list-metadata \
  --name acr-helloworld \
  --registry myregistry

# Delete by digest
az acr repository delete \
  --name myregistry \
  --image acr-helloworld@sha256:3168a21b98836dda7eb7a846b3d735286e09a32b0aa2401773da518e7eba3b57
```

### 4. Untag (Remove Tag Only)

Removes a tag without deleting the underlying image.

```bash
# Remove tag but keep manifest
az acr repository untag \
  --name myregistry \
  --image hello-world:latest
```

**Note**: This creates an untagged manifest. No storage is freed.

## Deleting by Timestamp

Delete manifests older than a specific date.

### List Old Manifests
```bash
az acr manifest list-metadata \
  --name <repositoryName> \
  --registry <acrName> \
  --orderby time_asc \
  -o tsv \
  --query "[?lastUpdateTime < '2023-04-05'].[digest, lastUpdateTime]"
```

### Automated Cleanup Script
```bash
#!/bin/bash
# WARNING: This script deletes data permanently!

ENABLE_DELETE=false  # Change to 'true' to enable
REGISTRY=myregistry
REPOSITORY=myrepository
TIMESTAMP=2023-04-05

if [ "$ENABLE_DELETE" = true ]; then
    az acr manifest list-metadata \
      --name $REPOSITORY \
      --registry $REGISTRY \
      --orderby time_asc \
      --query "[?lastUpdateTime < '$TIMESTAMP'].digest" -o tsv \
    | xargs -I% az acr repository delete \
      --name $REGISTRY \
      --image $REPOSITORY@% --yes
else
    echo "Dry run - would delete:"
    az acr manifest list-metadata \
      --name $REPOSITORY \
      --registry $REGISTRY \
      --orderby time_asc \
      --query "[?lastUpdateTime < '$TIMESTAMP'].[digest, lastUpdateTime]" -o tsv
fi
```

## Automatic Cleanup Methods

### 1. ACR Purge (Recommended)

Run `acr purge` as an ACR Task to automatically delete old images.

#### On-Demand Purge
```bash
PURGE_CMD="acr purge --filter 'hello-world:.*' \
  --untagged --ago 1d"

az acr run \
  --cmd "$PURGE_CMD" \
  --registry myregistry \
  /dev/null
```

#### Scheduled Purge
```bash
PURGE_CMD="acr purge --filter 'hello-world:.*' --ago 7d"

az acr task create \
  --name purgeTask \
  --cmd "$PURGE_CMD" \
  --schedule "0 0 * * *" \
  --registry myregistry \
  --context /dev/null
```

#### Purge Parameters

| Parameter | Description | Example |
|-----------|-------------|---------|
| `--filter` | Repository and tag regex | `'hello-world:.*'` |
| `--ago` | Delete older than duration | `7d`, `2d3h6m` |
| `--untagged` | Also delete untagged manifests | - |
| `--dry-run` | Preview without deleting | - |
| `--keep` | Keep N most recent tags | `--keep 5` |
| `--timeout` | Increase timeout for large operations | `3600` |

### 2. Retention Policy (Premium Only)

Automatically delete untagged manifests after a set period.

```bash
# Enable retention policy (30 days)
az acr config retention update \
  --registry myregistry \
  --status enabled \
  --days 30 \
  --type UntaggedManifests

# Show current policy
az acr config retention show \
  --registry myregistry

# Disable policy
az acr config retention update \
  --registry myregistry \
  --status disabled \
  --type UntaggedManifests
```

**Key Points**:
- Only affects manifests untagged **after** policy is enabled
- Default retention: 7 days (range: 0-365)
- Setting to 0 deletes immediately when untagged

### 3. Soft Delete Policy (Preview)

Recover accidentally deleted artifacts within a retention period.

```bash
# Enable soft delete (7 days retention)
az acr config soft-delete update \
  -r MyRegistry \
  --days 7 \
  --status enabled

# List soft-deleted repositories
az acr repository list-deleted -n MyRegistry

# List soft-deleted manifests
az acr manifest list-deleted -r MyRegistry -n hello-world

# Restore deleted image
az acr manifest restore \
  -r MyRegistry \
  -n hello-world:latest \
  -d sha256:abc123
```

**Limitations**:
- Cannot enable both retention policy and soft delete
- Not supported with zone redundancy, geo-replication, or artifact cache
- Soft-deleted artifacts still incur storage costs

## Handling Untagged Images

Untagged images (orphans) occur when:
- Pushing with an existing tag replaces the old image
- Using `az acr repository untag`

### Delete Untagged Images
```bash
# Using acr purge
az acr run \
  --cmd "acr purge --filter 'myrepo:.*' --untagged --ago 0d" \
  --registry myregistry \
  /dev/null
```

## Protecting Images from Deletion

Use image locking to prevent accidental deletion. See [locking.md](./locking.md).

```bash
# Protect from deletion
az acr repository update \
  --name myregistry \
  --image myrepo:tag \
  --delete-enabled false
```

## Best Practices

1. **Use Unique Tags for Deployments** - Prevents orphaned images from tag reuse
2. **Schedule Regular Purges** - Automate cleanup with ACR Tasks
3. **Lock Production Images** - Prevent accidental deletion
4. **Enable Soft Delete** - For recoverability during development
5. **Monitor Storage Usage** - Use `az acr show-usage` regularly
6. **Test Purge with --dry-run** - Preview before actual deletion

## Monitoring Storage

```bash
# Show storage usage
az acr show-usage \
  --resource-group myResourceGroup \
  --name myregistry \
  --output table
```

## Warnings

- **Deleted image data is UNRECOVERABLE** (unless soft delete is enabled)
- **Systems pulling by digest will fail** if those images are deleted
- **Retention policy only affects new untagged manifests** after enablement
- **acr purge does not delete locked images** (`write-enabled: false`)

## Source Documentation

- `/submodules/azure-management-docs/articles/container-registry/container-registry-delete.md`
- `/submodules/azure-management-docs/articles/container-registry/container-registry-auto-purge.md`
- `/submodules/azure-management-docs/articles/container-registry/container-registry-retention-policy.md`
- `/submodules/azure-management-docs/articles/container-registry/container-registry-soft-delete-policy.md`

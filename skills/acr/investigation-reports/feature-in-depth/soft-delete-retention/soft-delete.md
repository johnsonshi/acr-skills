# ACR Soft Delete Policy - Detailed Documentation

## Overview

The **Soft Delete Policy** is a preview feature in Azure Container Registry that allows you to recover accidentally deleted artifacts within a configurable retention period. When enabled, deleted artifacts are not immediately purged but are instead marked as "soft-deleted" and retained for a specified number of days.

## Feature Status

- **Status**: Preview
- **Availability**: All SKUs (Basic, Standard, Premium)
- **API Version**: Available in current Azure CLI and Portal

## How Soft Delete Works

### Lifecycle of Soft-Deleted Artifacts

```
[Active Artifact] ---(Delete Action)---> [Soft-Deleted] ---(Retention Period Expires)---> [Permanently Deleted]
                                              |
                                              +---(Restore Action)---> [Active Artifact]
```

1. When you delete an artifact (untag a manifest, delete an image), it becomes "soft-deleted"
2. Soft-deleted artifacts are retained for the configured retention period (1-90 days)
3. During the retention period, you can restore the artifact
4. After the retention period expires, the artifact is automatically purged (permanently deleted)

### Autopurge Process

- The autopurge runs **every 24 hours**
- It considers the **current value** of retention days
- If you change retention from 7 to 14 days, artifacts deleted 5 days ago will now expire at 14 days from deletion

## Configuration

### Default Settings

| Setting | Default Value | Range |
|---------|---------------|-------|
| Status | Disabled | enabled/disabled |
| Retention Days | 7 | 1-90 |

### Enabling via Azure Portal

1. Navigate to your Azure Container Registry
2. In **Overview**, check the status of **Soft delete (Preview)**
3. If **Status** is **Disabled**, select **Disabled** to open the **Properties** pane
4. Select the **Soft delete** checkbox
5. Enter a number of days between 1 and 90
6. Select **Save**

### Enabling via Azure CLI

```bash
# Enable soft delete with 14-day retention
az acr config soft-delete update -r MyRegistry --days 14 --status enabled

# Show current soft delete configuration
az acr config soft-delete show -r MyRegistry
```

## Viewing Soft-Deleted Artifacts

### Azure Portal

1. Navigate to your container registry
2. Under **Services**, select **Repositories**
3. Select a repository
4. Select **Manage deleted artifacts** to view soft-deleted items
5. For deleted repositories, select **Manage Deleted Repositories**

### Azure CLI

```bash
# List soft-deleted repositories
az acr repository list-deleted -n MyRegistry

# List soft-deleted manifests in a repository
az acr manifest list-deleted -r MyRegistry -n hello-world

# List soft-deleted tags in a repository
az acr manifest list-deleted-tags -r MyRegistry -n hello-world

# Filter soft-deleted tags by name
az acr manifest list-deleted-tags -r MyRegistry -n hello-world:latest
```

## Restoring Soft-Deleted Artifacts

### Azure Portal

1. Navigate to **Repositories** in your registry
2. Select a repository and click **Manage deleted artifacts**
3. In the row for the deleted artifact, select **Restore**
4. In the **Restore Artifact** pane, select the tag to restore
5. Click **Restore**

**Note**: You can only restore one tag at a time. Additional tags must be restored separately.

### Azure CLI

```bash
# Restore by tag and digest
az acr manifest restore -r MyRegistry -n hello-world:latest -d sha256:abc123

# Restore most recently deleted manifest by tag
az acr manifest restore -r MyRegistry -n hello-world:latest

# Force restore (overwrites existing tag with same name)
az acr manifest restore -r MyRegistry -n hello-world:latest -d sha256:abc123 -f
```

### Force Restore Behavior

When using `--force` (`-f`):
- Overwrites the existing tag with the same name
- If soft delete is enabled, the overwritten tag becomes soft-deleted
- The overwritten tag is retained according to the retention period

## Required Permissions

Users need the following permissions at the container registry level:

| Permission | Description |
|------------|-------------|
| `Microsoft.ContainerRegistry/registries/deleted/read` | List soft-deleted artifacts |
| `Microsoft.ContainerRegistry/registries/deleted/restore/action` | Restore soft-deleted artifacts |

## Limitations and Restrictions

### Current Limitations

1. **No manual purging**: Cannot manually purge soft-deleted artifacts before retention expires
2. **Mutual exclusivity with retention policy**: Cannot enable both soft delete and retention policy simultaneously
3. **Not supported with**:
   - Zone redundancy
   - Geo-replication
   - Artifact cache
4. **Import restrictions**: Cannot import a soft-deleted image at source or target
5. **Push restrictions**: Cannot push an image with the same manifest digest as a soft-deleted image (must restore first)

### Manifest List Restore Limitations

- Restoring a manifest list does NOT recursively restore underlying soft-deleted manifests
- Each manifest must be restored individually

### ORAS Artifact Restore Limitations

- Restoring a subject does NOT recursively restore the referrer chain
- The subject must be restored BEFORE you can restore a referrer manifest

## Billing Considerations

**Important**: Soft-deleted artifacts are billed at the same rate as active artifacts.

- Storage costs continue during the retention period
- Plan retention period based on cost vs. recovery needs
- Consider shorter retention periods for development registries

## Best Practices

### Recommended Retention Periods

| Environment | Recommended Retention | Rationale |
|-------------|----------------------|-----------|
| Production | 14-30 days | Allow time to detect accidental deletions |
| Staging | 7-14 days | Balance recovery needs with storage costs |
| Development | 1-7 days | Minimize storage costs |

### Operational Recommendations

1. **Monitor storage usage**: Soft-deleted artifacts count toward storage
2. **Document deletion procedures**: Ensure teams know how to restore artifacts
3. **Test restore process**: Periodically verify restore functionality works
4. **Consider image locking**: For critical images, use locking instead of relying on soft delete

### When to Use Soft Delete vs. Image Locking

| Scenario | Recommendation |
|----------|----------------|
| Protect production images from accidental deletion | Image Locking (preventive) |
| Recover from mistakes in development workflow | Soft Delete (reactive) |
| Compliance requirement for data retention | Soft Delete + appropriate retention period |
| Critical images that must never be deleted | Image Locking |

## Interaction with Other Features

### Pushing Images

- Pushing an image to a soft-deleted repository **restores** that repository
- Pushing an image that shares the same manifest digest as a soft-deleted image is **not allowed**
  - You must restore the soft-deleted image instead

### Deleting Images

When soft delete is enabled:
- Untagging a manifest creates a soft-deleted artifact
- Deleting a repository soft-deletes all artifacts in it
- Force restore of an existing tag soft-deletes the overwritten tag

## Example Workflows

### Workflow 1: Recover Accidentally Deleted Image

```bash
# 1. Discover the deletion
az acr manifest list-deleted-tags -r MyRegistry -n myapp

# 2. Find the specific digest
az acr manifest list-deleted -r MyRegistry -n myapp

# 3. Restore the image
az acr manifest restore -r MyRegistry -n myapp:v1.0 -d sha256:abc123

# 4. Verify restoration
az acr manifest list-metadata -r MyRegistry -n myapp
```

### Workflow 2: Recover Deleted Repository

```bash
# 1. List deleted repositories
az acr repository list-deleted -n MyRegistry

# 2. Check deleted manifests in repository
az acr manifest list-deleted -r MyRegistry -n deleted-repo

# 3. Restore specific image from repository
az acr manifest restore -r MyRegistry -n deleted-repo:latest

# 4. Repository is now active again
az acr repository list -n MyRegistry
```

## Troubleshooting

### Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| Cannot enable soft delete | Retention policy is enabled | Disable retention policy first |
| Cannot enable soft delete | Registry is geo-replicated | Remove geo-replications first |
| Cannot push image | Manifest digest matches soft-deleted image | Restore the soft-deleted image first |
| Artifact not found in deleted list | Retention period expired | Cannot recover - artifact is permanently deleted |

## Source References

- Primary documentation: `/submodules/azure-management-docs/articles/container-registry/container-registry-soft-delete-policy.md`
- Delete operations: `/submodules/azure-management-docs/articles/container-registry/container-registry-delete.md`
- Image locking: `/submodules/azure-management-docs/articles/container-registry/container-registry-image-lock.md`

# ACR Retention Policy - Detailed Documentation

## Overview

The **Retention Policy** in Azure Container Registry is a preview feature that automatically deletes untagged manifests after a specified period. This helps manage registry size by removing orphaned images that are no longer referenced by any tags.

## Feature Status

- **Status**: Preview
- **Availability**: Premium SKU only
- **Purpose**: Storage optimization through automatic cleanup of untagged manifests

## How Retention Policy Works

### Target: Untagged Manifests

Untagged manifests (also called "orphaned" or "dangling" images) are created when:
1. An image is pushed with a tag that already exists (the old image becomes untagged)
2. A tag is explicitly removed with `az acr repository untag`
3. The last tag referencing a manifest is deleted

### Lifecycle with Retention Policy

```
[Tagged Image] ---(Tag Updated/Removed)---> [Untagged Manifest] ---(Retention Period Expires)---> [Deleted]
```

### Reference Counting Mechanism

1. ACR performs reference counting for manifests
2. When a manifest becomes untagged, ACR checks if retention policy is enabled
3. If enabled AND `delete-enabled` attribute is `true`, ACR schedules deletion
4. Deletion occurs after the configured retention period

## Configuration

### Default Settings

| Setting | Default Value | Range |
|---------|---------------|-------|
| Status | Disabled | enabled/disabled |
| Retention Days | 7 | 0-365 |
| Type | UntaggedManifests | UntaggedManifests |

### Setting to 0 Days

When retention is set to **0 days**, untagged manifests are deleted **as soon as they become untagged** (within seconds).

## Enabling Retention Policy

### Azure Portal

1. Navigate to your Azure container registry
2. In the service menu, under **Policies**, select **Retention (Preview)**
3. In **Status**, select **Enabled**
4. Specify a number of days between 0 and 365
5. Select **Save**

### Azure CLI

```bash
# Enable retention policy with 30-day retention
az acr config retention update --registry myregistry --status enabled --days 30 --type UntaggedManifests

# Show current retention policy
az acr config retention show --registry myregistry
```

## Disabling Retention Policy

### Azure Portal

1. Navigate to your container registry
2. Under **Policies**, select **Retention (Preview)**
3. In **Status**, select **Disabled**
4. Select **Save**

### Azure CLI

```bash
az acr config retention update --registry myregistry --status disabled --type UntaggedManifests
```

## Important Behaviors

### Policy Application Scope

**Critical**: The retention policy only applies to untagged manifests with timestamps **AFTER** the policy is enabled.

- Existing untagged manifests before policy enablement are **NOT** affected
- Only newly untagged manifests are subject to the policy

### Manifest Type Limitations

The retention policy only supports:
- Docker `v2` manifests

**NOT supported**:
- OCI image index (`application/vnd.oci.image.index.v1+json`)

### Protecting Manifests from Retention Policy

You can exclude specific manifests from the retention policy by setting:

```bash
az acr repository update --name myregistry --image myrepo@sha256:abc123 --delete-enabled false
```

When `delete-enabled` is set to `false`:
- The manifest is protected from the retention policy
- The manifest is protected from manual deletion

## Verification and Testing

### Quick Verification (0-day retention)

```bash
# 1. Enable retention with 0 days
az acr config retention update --registry myregistry --status enabled --days 0 --type UntaggedManifests

# 2. Push a test image
docker push myregistry.azurecr.io/hello-world:latest

# 3. Untag the image (creates untagged manifest)
az acr repository untag --name myregistry --image hello-world:latest

# 4. Within seconds, the untagged manifest should be deleted
az acr manifest list-metadata --name hello-world --registry myregistry
```

### Monitoring Retention Policy Effects

```bash
# List all manifests in a repository
az acr manifest list-metadata --name myrepo --registry myregistry

# Check for untagged manifests (manifests with empty tags array)
az acr manifest list-metadata --name myrepo --registry myregistry --query "[?tags==null || length(tags)==\`0\`]"
```

## Comparison: Retention Policy vs. Soft Delete

| Aspect | Retention Policy | Soft Delete |
|--------|------------------|-------------|
| **SKU** | Premium only | All SKUs |
| **Target** | Untagged manifests only | All deleted artifacts |
| **Recovery** | Not possible | Yes, within retention period |
| **Purpose** | Storage optimization | Accident recovery |
| **Billing** | Reduces storage costs | Continues billing |
| **Mutual Exclusivity** | Cannot use with Soft Delete | Cannot use with Retention Policy |

## Comparison: Retention Policy vs. ACR Purge

| Aspect | Retention Policy | ACR Purge |
|--------|------------------|-----------|
| **Target** | Untagged manifests | Tagged images (by age/filter) |
| **Automation** | Automatic | Scheduled via ACR Tasks |
| **Configuration** | Simple (days only) | Flexible (filters, age, patterns) |
| **Scope** | Registry-wide | Per-task/repository |
| **Use Case** | Orphan cleanup | Lifecycle management |

## Best Practices

### Recommended Configuration by Environment

| Environment | Retention Days | Rationale |
|-------------|---------------|-----------|
| Production | 7-30 days | Allow time for issue detection |
| Staging | 1-7 days | Balance storage with recovery window |
| Development | 0-1 days | Aggressive cleanup acceptable |

### When to Use Retention Policy

**Use retention policy when**:
- You use stable tags (like `latest`) that frequently update
- You want automatic cleanup without scheduling tasks
- Storage costs are a concern
- You have a Premium SKU registry

**Don't use retention policy when**:
- Systems pull images by manifest digest (not by tag)
- You need the ability to recover deleted images
- You have a Basic or Standard SKU registry

### Warning for Digest-Based Pulls

**Critical Warning**: If your systems pull images by manifest digest:
- Do NOT enable retention policy
- Untagged images will be deleted even if still in use
- Consider using unique tags instead

```
# Bad pattern with retention policy:
docker pull myregistry.azurecr.io/myapp@sha256:abc123  # Image may be deleted!

# Good pattern:
docker pull myregistry.azurecr.io/myapp:v1.0.0-abc123  # Unique tag protects image
```

## Integration with Other Features

### Image Locking

Retention policy respects image locking:
- Manifests with `delete-enabled=false` are NOT deleted by retention policy
- Use locking to protect specific manifests that must remain even when untagged

```bash
# Protect a manifest from retention policy
az acr repository update --name myregistry --image myrepo@sha256:abc123 --delete-enabled false
```

### ACR Purge

Retention policy and ACR Purge can complement each other:
- Retention policy handles **untagged** manifests automatically
- ACR Purge handles **tagged** images based on age/filters

### Geo-Replication

Retention policy works with geo-replicated registries:
- Policy is applied across all replicas
- Deleted manifests are removed from all regions

## Example Scenarios

### Scenario 1: CI/CD Pipeline with Stable Tags

```bash
# Enable retention to clean up old builds pushed to :latest
az acr config retention update --registry myregistry --status enabled --days 7 --type UntaggedManifests

# Each CI build pushes to myapp:latest
# Previous :latest becomes untagged
# After 7 days, untagged images are automatically deleted
```

### Scenario 2: Multi-Environment Registry

```bash
# Production registry - longer retention for safety
az acr config retention update --registry prod-registry --status enabled --days 30 --type UntaggedManifests

# Development registry - aggressive cleanup
az acr config retention update --registry dev-registry --status enabled --days 1 --type UntaggedManifests
```

### Scenario 3: Protecting Critical Images

```bash
# Enable retention policy
az acr config retention update --registry myregistry --status enabled --days 7 --type UntaggedManifests

# Protect critical base images from deletion
az acr repository update --name myregistry --image base-images/python@sha256:abc123 --delete-enabled false
```

## Troubleshooting

### Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| Cannot enable retention | Not using Premium SKU | Upgrade to Premium SKU |
| Cannot enable retention | Soft delete is enabled | Disable soft delete first |
| Old untagged manifests not deleted | Created before policy enabled | Use ACR Purge or manual deletion |
| Manifest not deleted after retention period | `delete-enabled` is `false` | Update manifest attributes |

### Checking Why Manifest Wasn't Deleted

```bash
# Check manifest attributes
az acr repository show --name myregistry --image myrepo@sha256:abc123

# Look for:
# "changeableAttributes": {
#   "deleteEnabled": true/false  <-- if false, protected from deletion
# }
```

## Billing Impact

### Storage Cost Reduction

- Retention policy helps reduce storage costs
- Untagged manifests are automatically cleaned up
- No storage charges after deletion
- Consider shorter retention periods for cost optimization

### Comparison: Retention Policy vs. No Policy

| Scenario | Without Retention | With 7-day Retention |
|----------|-------------------|---------------------|
| 100 daily builds | 3000 manifests/month | ~700 manifests (rolling 7 days) |
| Storage growth | Linear, unbounded | Bounded by retention period |
| Manual cleanup | Required | Automatic |

## Source References

- Primary documentation: `/submodules/azure-management-docs/articles/container-registry/container-registry-retention-policy.md`
- Delete operations: `/submodules/azure-management-docs/articles/container-registry/container-registry-delete.md`
- Image locking: `/submodules/azure-management-docs/articles/container-registry/container-registry-image-lock.md`
- Tagging best practices: `/submodules/azure-management-docs/articles/container-registry/container-registry-image-tag-version.md`

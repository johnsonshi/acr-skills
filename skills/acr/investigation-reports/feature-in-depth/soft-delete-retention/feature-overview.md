# ACR Soft Delete and Retention - Feature Overview

## Executive Summary

Azure Container Registry (ACR) provides multiple mechanisms for managing the lifecycle of container images and artifacts. This document provides a comprehensive overview of the features available for managing deleted and obsolete artifacts in ACR.

## Feature Comparison Matrix

| Feature | Description | SKU Availability | Status | Recoverable |
|---------|-------------|------------------|--------|-------------|
| **Soft Delete Policy** | Recover accidentally deleted artifacts | All SKUs (Basic, Standard, Premium) | Preview | Yes (within retention period) |
| **Retention Policy** | Auto-delete untagged manifests | Premium only | Preview | No |
| **ACR Purge (Auto-Purge)** | Scheduled deletion based on age/filters | All SKUs | Preview | No |
| **Manual Deletion** | CLI/Portal commands to delete images | All SKUs | GA | No |
| **Image Locking** | Prevent deletion/modification | All SKUs | GA | N/A (preventive) |

## Key Concepts

### 1. Soft Delete Policy (Preview)
- **Purpose**: Recover accidentally deleted artifacts within a configurable retention period
- **Default Retention**: 7 days (configurable 1-90 days)
- **Billing**: Soft-deleted artifacts are billed at active SKU storage rates
- **Limitations**:
  - Cannot be enabled with retention policy simultaneously
  - Not supported with zone redundancy, geo-replication, or artifact cache
  - No manual purging of soft-deleted artifacts

### 2. Retention Policy (Preview)
- **Purpose**: Automatically delete untagged manifests after a specified period
- **Default Retention**: 7 days (configurable 0-365 days)
- **Target**: Only untagged manifests (orphaned images)
- **Billing**: Reduces storage costs by removing unused artifacts
- **Limitation**: Only applies to manifests created AFTER policy is enabled

### 3. ACR Purge (Auto-Purge)
- **Purpose**: Scheduled cleanup of images based on age and tag filters
- **Implementation**: Runs as an ACR Task container (`mcr.microsoft.com/acr/acr-cli:0.17`)
- **Scheduling**: Via cron expressions (timer triggers)
- **Flexibility**: Filter by repository, tag patterns, and age

### 4. Image Locking
- **Purpose**: Prevent accidental deletion or modification of critical images
- **Attributes**: `write-enabled`, `delete-enabled`, `read-enabled`, `list-enabled`
- **Scope**: Can be applied to individual images or entire repositories

## Decision Matrix: When to Use Each Feature

| Scenario | Recommended Feature |
|----------|-------------------|
| Need to recover accidentally deleted images | Soft Delete Policy |
| Automatic cleanup of orphaned/untagged images | Retention Policy |
| Regular cleanup based on age or naming patterns | ACR Purge |
| Protect production images from deletion | Image Locking |
| One-time cleanup of specific images | Manual Deletion |

## Architecture Considerations

### Storage Management Strategy
1. **Development registries**: Use ACR Purge with aggressive filters
2. **Production registries**: Enable Image Locking + Soft Delete Policy
3. **Mixed workloads**: Combine Image Locking (for critical images) with Retention Policy (for untagged cleanup)

### Mutual Exclusivity
- **Soft Delete Policy** and **Retention Policy** CANNOT be enabled simultaneously
- Choose based on priority:
  - Recoverability priority -> Soft Delete Policy
  - Storage optimization priority -> Retention Policy

## Related Features

### Export Policy (Data Loss Prevention)
- Prevents artifacts from being exported outside the virtual network
- Premium SKU only
- Useful for compliance and security scenarios

### Geo-Replication Impact
- Soft delete policy not supported with geo-replicated registries
- Storage usage in geo-replicated registries = home region usage x number of replicas

## Source Documentation

| Document | Path |
|----------|------|
| Soft Delete Policy | `submodules/azure-management-docs/articles/container-registry/container-registry-soft-delete-policy.md` |
| Retention Policy | `submodules/azure-management-docs/articles/container-registry/container-registry-retention-policy.md` |
| Auto-Purge | `submodules/azure-management-docs/articles/container-registry/container-registry-auto-purge.md` |
| Delete Images | `submodules/azure-management-docs/articles/container-registry/container-registry-delete.md` |
| Image Locking | `submodules/azure-management-docs/articles/container-registry/container-registry-image-lock.md` |
| Best Practices | `submodules/azure-management-docs/articles/container-registry/container-registry-best-practices.md` |
| SKUs | `submodules/azure-management-docs/articles/container-registry/container-registry-skus.md` |
| Storage | `submodules/azure-management-docs/articles/container-registry/container-registry-storage.md` |

## Version Information
- Documentation last updated: December 2025
- Soft Delete: Preview status
- Retention Policy: Preview status (Premium only)
- ACR Purge: Preview status

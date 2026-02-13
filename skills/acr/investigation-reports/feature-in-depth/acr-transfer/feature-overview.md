# ACR Transfer Feature Overview

## Summary

ACR Transfer is a feature in Azure Container Registry (ACR) that enables the transfer of container images and other registry artifacts between registries that may be in physically disconnected or air-gapped environments. It uses Azure Blob Storage as an intermediary to move artifacts between source and target registries.

## Key Use Cases

### Primary Use Case: Air-Gapped Environments

ACR Transfer is specifically designed for scenarios where:

- **Physically disconnected clouds**: Moving artifacts between Azure clouds that have no direct network connectivity (e.g., Azure Commercial to Azure Government)
- **Air-gapped registries**: Environments with strict network isolation requirements
- **Cross-tenant transfers**: Moving artifacts between different Active Directory tenants
- **Cross-subscription transfers**: Moving artifacts between different Azure subscriptions

### When NOT to Use ACR Transfer

For registries in connected clouds (including Docker Hub and other cloud vendors), use [ACR Import](container-registry-import-images.md) instead, which is simpler and faster.

## Feature Availability

| Requirement | Details |
|-------------|---------|
| **SKU** | Premium tier only |
| **Status** | Preview |
| **API Version** | 2019-12-01-preview and later |
| **Layer Size Limit** | 8 GB maximum per layer |
| **Artifacts per Run** | Maximum 50 artifacts per PipelineRun |

## How It Works

ACR Transfer operates through a three-step pipeline process:

```
Source Registry ──> Export Pipeline ──> Blob Storage ──> Transfer ──> Blob Storage ──> Import Pipeline ──> Target Registry
```

1. **Export Phase**: Artifacts from the source registry are exported to a blob in the source storage account
2. **Transfer Phase**: The blob is copied (manually or via AzCopy) from source to target storage account
3. **Import Phase**: The blob in the target storage account is imported as artifacts into the target registry

## Core Resources

ACR Transfer introduces three key Azure Resource Manager resources:

### 1. ExportPipeline

A long-lasting resource that defines:
- Source registry configuration
- Target storage account blob container URI
- Key Vault reference for SAS token secrets
- Export options (OverwriteBlobs, ContinueOnErrors)

### 2. ImportPipeline

A long-lasting resource that defines:
- Target registry configuration
- Source storage account blob container URI
- Key Vault reference for SAS token secrets
- Import options (OverwriteTags, DeleteSourceBlobOnSuccess, ContinueOnErrors)
- Source trigger configuration (automatic vs. manual import)

### 3. PipelineRun

A resource used to execute either an ExportPipeline or ImportPipeline:
- For exports: Specifies which artifacts to export
- For imports: Specifies which blob to import
- Can be triggered automatically or manually

## Authentication and Security

### Managed Identity

Pipelines use managed identities to access Key Vault secrets:

| Identity Type | Use Case |
|---------------|----------|
| **System-assigned** | Automatically created; simpler setup |
| **User-assigned** | Pre-existing identity; can be shared across resources |

### SAS Token Requirements

| Pipeline | Required Permissions |
|----------|---------------------|
| **Export** | Read, Write, List, Add |
| **Import** | Read, Delete, List (Delete only if DeleteSourceBlobOnSuccess is enabled) |

### Key Vault Integration

- SAS tokens are stored as secrets in Azure Key Vault
- Pipeline identities require `secret get` permissions on the Key Vault
- Recommended: Use Key Vault Managed Storage SAS Definition Secrets for production

## Export Policy (Data Loss Prevention)

ACR supports an `exportPolicy` property to prevent data exfiltration:

- When set to `disabled`, blocks creation of export pipelines
- Requires `publicNetworkAccess` to also be `disabled`
- Available in API version 2021-06-01-preview and later

## Implementation Options

### 1. Azure CLI Extension (Recommended)

The `acrtransfer` Az CLI extension provides a simplified interface:
- `az acr export-pipeline create`
- `az acr import-pipeline create`
- `az acr pipeline-run create`

### 2. ARM Templates

Azure Resource Manager templates are available for:
- ExportPipelines
- ImportPipelines
- PipelineRuns

### 3. .NET SDK

Sample applications demonstrating programmatic usage are available in the ACR GitHub repository.

## Source Trigger Behavior

The ImportPipeline can be configured with automatic triggering:

| Configuration | Behavior |
|---------------|----------|
| **sourceTriggerStatus: Enabled** (default) | Automatically imports blobs within 15 minutes of arrival |
| **sourceTriggerStatus: Disabled** | Requires manual PipelineRun creation |

### Source Trigger Requirements

- Blob must have `Last Modified` time within the last 60 days
- Blob must have valid `ContentMD5` property
- Blob must have metadata: `"category":"acr-transfer-blob"`

## Related Features

- [ACR Import](container-registry-import-images.md): For connected cloud scenarios
- [Data Loss Prevention](data-loss-prevention.md): Blocking export pipeline creation
- [Private Endpoints](container-registry-private-link.md): Network isolation

## References

### Documentation Sources
- `/submodules/azure-management-docs/articles/container-registry/container-registry-transfer-prerequisites.md`
- `/submodules/azure-management-docs/articles/container-registry/container-registry-transfer-images.md`
- `/submodules/azure-management-docs/articles/container-registry/container-registry-transfer-cli.md`
- `/submodules/azure-management-docs/articles/container-registry/container-registry-transfer-troubleshooting.md`

### ARM Templates
- `/submodules/acr/docs/image-transfer/ExportPipelines/`
- `/submodules/acr/docs/image-transfer/ImportPipelines/`
- `/submodules/acr/docs/image-transfer/PipelineRun/`

### Sample Code
- `/submodules/acr/samples/dotnetcore/image-transfer/`
- `/submodules/acr/samples/dotnetcore/registry-artifact-transfer/`

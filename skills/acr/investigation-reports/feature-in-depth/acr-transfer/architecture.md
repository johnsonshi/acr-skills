# ACR Transfer Architecture

## High-Level Architecture

ACR Transfer is designed to move container images and OCI artifacts between physically disconnected Azure environments using blob storage as an intermediary.

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                              SOURCE ENVIRONMENT                                  │
│  ┌──────────────────┐     ┌──────────────────┐     ┌──────────────────┐        │
│  │                  │     │                  │     │                  │        │
│  │  Source ACR      │────▶│  ExportPipeline  │────▶│  Source Storage  │        │
│  │  (Premium)       │     │                  │     │  Account         │        │
│  │                  │     │  - Identity      │     │  - Blob Container│        │
│  └──────────────────┘     │  - Target URI    │     │  - SAS Token     │        │
│                           │  - KeyVault Ref  │     │                  │        │
│                           └──────────────────┘     └────────┬─────────┘        │
│                                                              │                  │
│  ┌──────────────────┐                                       │                  │
│  │  Source KeyVault │◀──────────────────────────────────────┘                  │
│  │  - SAS Secret    │                                                          │
│  └──────────────────┘                                                          │
└─────────────────────────────────────────────────────────────────────────────────┘
                                        │
                                        │  AzCopy / Cross-Domain Solution
                                        │  (Blob Transfer)
                                        ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                              TARGET ENVIRONMENT                                  │
│                           ┌──────────────────┐                                  │
│                           │  Target Storage  │                                  │
│                           │  Account         │                                  │
│                           │  - Blob Container│                                  │
│                           │  - SAS Token     │                                  │
│                           └────────┬─────────┘                                  │
│                                    │                                            │
│  ┌──────────────────┐              │         ┌──────────────────┐              │
│  │  Target KeyVault │◀─────────────┼─────────│  ImportPipeline  │              │
│  │  - SAS Secret    │              │         │                  │              │
│  └──────────────────┘              │         │  - Identity      │              │
│                                    │         │  - Source URI    │              │
│                                    │         │  - KeyVault Ref  │              │
│                                    ▼         │  - Trigger       │              │
│  ┌──────────────────┐     ┌──────────────────┴──────────────────┐              │
│  │                  │     │                                      │              │
│  │  Target ACR      │◀────│            ImportPipeline            │              │
│  │  (Premium)       │     │                                      │              │
│  │                  │     └──────────────────────────────────────┘              │
│  └──────────────────┘                                                          │
└─────────────────────────────────────────────────────────────────────────────────┘
```

## Resource Types and Relationships

### Resource Provider

All ACR Transfer resources are under the `Microsoft.ContainerRegistry` resource provider:

```
Microsoft.ContainerRegistry/
├── registries/
│   ├── exportPipelines/
│   ├── importPipelines/
│   └── pipelineRuns/
```

### Resource Hierarchy

```
Azure Subscription
├── Resource Group (Source)
│   ├── Container Registry (Premium)
│   │   └── ExportPipeline
│   │       └── PipelineRun (Export)
│   ├── Storage Account
│   │   └── Blob Container
│   └── Key Vault
│       └── SAS Token Secret
│
└── Resource Group (Target)
    ├── Container Registry (Premium)
    │   └── ImportPipeline
    │       └── PipelineRun (Import)
    ├── Storage Account
    │   └── Blob Container
    └── Key Vault
        └── SAS Token Secret
```

## Data Flow Architecture

### Export Pipeline Data Flow

```
┌─────────────────────────────────────────────────────────────────┐
│                      EXPORT DATA FLOW                            │
│                                                                  │
│  1. PipelineRun Created                                         │
│         │                                                        │
│         ▼                                                        │
│  2. ExportPipeline Identity fetches SAS token from KeyVault     │
│         │                                                        │
│         ▼                                                        │
│  3. ExportPipeline reads artifacts from ACR                     │
│         │                                                        │
│         ▼                                                        │
│  4. Artifacts packaged into tar blob                            │
│         │                                                        │
│         ▼                                                        │
│  5. Blob uploaded to Storage Container using SAS token          │
│         │                                                        │
│         ▼                                                        │
│  6. PipelineRun status updated (Success/Failed)                 │
└─────────────────────────────────────────────────────────────────┘
```

### Import Pipeline Data Flow

```
┌─────────────────────────────────────────────────────────────────┐
│                      IMPORT DATA FLOW                            │
│                                                                  │
│  1. Source Trigger OR Manual PipelineRun                        │
│         │                                                        │
│         ▼                                                        │
│  2. ImportPipeline Identity fetches SAS token from KeyVault     │
│         │                                                        │
│         ▼                                                        │
│  3. ImportPipeline reads blob from Storage Container            │
│         │                                                        │
│         ▼                                                        │
│  4. Blob unpacked and artifacts extracted                       │
│         │                                                        │
│         ▼                                                        │
│  5. Artifacts pushed to target ACR                              │
│         │                                                        │
│         ▼                                                        │
│  6. (Optional) Source blob deleted if DeleteSourceBlobOnSuccess │
│         │                                                        │
│         ▼                                                        │
│  7. PipelineRun status updated (Success/Failed)                 │
└─────────────────────────────────────────────────────────────────┘
```

## Identity Architecture

### Managed Identity Flow

```
┌──────────────────────────────────────────────────────────────────────────┐
│                     MANAGED IDENTITY ARCHITECTURE                         │
│                                                                          │
│  ┌─────────────────┐         ┌─────────────────┐                        │
│  │                 │         │                 │                        │
│  │  ExportPipeline │         │  ImportPipeline │                        │
│  │                 │         │                 │                        │
│  │  Identity:      │         │  Identity:      │                        │
│  │  - System       │         │  - System       │                        │
│  │    Assigned     │         │    Assigned     │                        │
│  │      OR         │         │      OR         │                        │
│  │  - User         │         │  - User         │                        │
│  │    Assigned     │         │    Assigned     │                        │
│  │                 │         │                 │                        │
│  └────────┬────────┘         └────────┬────────┘                        │
│           │                           │                                  │
│           │ secret get                │ secret get                       │
│           │ permission                │ permission                       │
│           ▼                           ▼                                  │
│  ┌─────────────────┐         ┌─────────────────┐                        │
│  │                 │         │                 │                        │
│  │  Source         │         │  Target         │                        │
│  │  Key Vault      │         │  Key Vault      │                        │
│  │                 │         │                 │                        │
│  │  Access Policy: │         │  Access Policy: │                        │
│  │  - ObjectId:    │         │  - ObjectId:    │                        │
│  │    PrincipalId  │         │    PrincipalId  │                        │
│  │  - Permissions: │         │  - Permissions: │                        │
│  │    secrets: get │         │    secrets: get │                        │
│  │                 │         │                 │                        │
│  └─────────────────┘         └─────────────────┘                        │
└──────────────────────────────────────────────────────────────────────────┘
```

### Identity Types

| Type | Configuration | Lifecycle | Use Case |
|------|---------------|-----------|----------|
| **System-Assigned** | Auto-created with pipeline | Deleted with pipeline | Simple, single-use scenarios |
| **User-Assigned** | Pre-created, referenced by resource ID | Independent of pipeline | Shared identity, complex RBAC |

## Storage Architecture

### Blob Container Structure

```
Storage Account
└── transfer (container)
    ├── export-blob-1
    ├── export-blob-2
    └── ...
```

### Blob Metadata Requirements

For source trigger to work, blobs must have:

```json
{
  "ContentMD5": "<valid-md5-hash>",
  "LastModified": "<within-60-days>",
  "metadata": {
    "category": "acr-transfer-blob"
  }
}
```

## ARM Template Resource Structure

### ExportPipeline Resource

```json
{
  "type": "Microsoft.ContainerRegistry/registries/exportPipelines",
  "apiVersion": "2019-12-01-preview",
  "name": "[concat(registryName, '/', exportPipelineName)]",
  "location": "[location]",
  "identity": {
    "type": "SystemAssigned"  // or "UserAssigned"
  },
  "properties": {
    "target": {
      "type": "AzureStorageBlobContainer",
      "uri": "https://<storage>.blob.core.windows.net/<container>",
      "keyVaultUri": "https://<keyvault>.vault.azure.net/secrets/<secret>"
    },
    "options": ["OverwriteBlobs", "ContinueOnErrors"]
  }
}
```

### ImportPipeline Resource

```json
{
  "type": "Microsoft.ContainerRegistry/registries/importPipelines",
  "apiVersion": "2019-12-01-preview",
  "name": "[concat(registryName, '/', importPipelineName)]",
  "location": "[location]",
  "identity": {
    "type": "SystemAssigned"
  },
  "properties": {
    "source": {
      "type": "AzureStorageBlobContainer",
      "uri": "https://<storage>.blob.core.windows.net/<container>",
      "keyVaultUri": "https://<keyvault>.vault.azure.net/secrets/<secret>"
    },
    "trigger": {
      "sourceTrigger": {
        "status": "Enabled"
      }
    },
    "options": ["OverwriteTags", "DeleteSourceBlobOnSuccess", "ContinueOnErrors"]
  }
}
```

### PipelineRun Resource

```json
{
  "type": "Microsoft.ContainerRegistry/registries/pipelineRuns",
  "apiVersion": "2019-12-01-preview",
  "name": "[concat(registryName, '/', pipelineRunName)]",
  "location": "[location]",
  "properties": {
    "request": {
      "pipelineResourceId": "<export-or-import-pipeline-resource-id>",
      "artifacts": ["image:tag", "image@sha256:digest"],
      "target": {
        "type": "AzureStorageBlob",
        "name": "blobName"
      }
    },
    "forceUpdateTag": "<unique-value-for-redeploy>"
  }
}
```

## Cross-Environment Architecture

### Typical Deployment Pattern

```
┌──────────────────────────────────┐     ┌──────────────────────────────────┐
│       AZURE COMMERCIAL           │     │        AZURE GOVERNMENT          │
│                                  │     │                                  │
│  ┌──────────────────────────┐   │     │   ┌──────────────────────────┐  │
│  │ Resource Group: source-rg │   │     │   │ Resource Group: target-rg │  │
│  │                          │   │     │   │                          │  │
│  │  ┌────────────────────┐  │   │     │   │  ┌────────────────────┐  │  │
│  │  │ ACR: source-acr    │  │   │     │   │  │ ACR: target-acr    │  │  │
│  │  │ (Premium)          │  │   │     │   │  │ (Premium)          │  │  │
│  │  │                    │  │   │     │   │  │                    │  │  │
│  │  │  ExportPipeline    │──────────────────▶│  ImportPipeline    │  │  │
│  │  │                    │  │   │     │   │  │                    │  │  │
│  │  └────────────────────┘  │   │     │   │  └────────────────────┘  │  │
│  │                          │   │     │   │                          │  │
│  │  ┌────────────────────┐  │   │     │   │  ┌────────────────────┐  │  │
│  │  │ Storage Account    │──────┼─────┼───▶│ Storage Account     │  │  │
│  │  │ - transfer container│  │   │     │   │  │ - transfer container│  │  │
│  │  └────────────────────┘  │   │     │   │  └────────────────────┘  │  │
│  │                          │   │     │   │                          │  │
│  │  ┌────────────────────┐  │   │     │   │  ┌────────────────────┐  │  │
│  │  │ Key Vault          │  │   │     │   │  │ Key Vault          │  │  │
│  │  │ - export SAS       │  │   │     │   │  │ - import SAS       │  │  │
│  │  └────────────────────┘  │   │     │   │  └────────────────────┘  │  │
│  └──────────────────────────┘   │     │   └──────────────────────────┘  │
│                                  │     │                                  │
└──────────────────────────────────┘     └──────────────────────────────────┘
```

## Limits and Constraints

| Constraint | Value |
|------------|-------|
| Maximum artifacts per PipelineRun | 50 |
| Maximum layer size | 8 GB |
| Source trigger blob age limit | 60 days (Last Modified) |
| SKU requirement | Premium |
| Automatic import trigger delay | Up to 15 minutes |

## References

### Source Files
- ARM Template: `/submodules/acr/docs/image-transfer/ExportPipelines/azuredeploy.json`
- ARM Template: `/submodules/acr/docs/image-transfer/ImportPipelines/azuredeploy.json`
- ARM Template: `/submodules/acr/docs/image-transfer/PipelineRun/PipelineRun-Export/azuredeploy.json`
- Documentation: `/submodules/azure-management-docs/articles/container-registry/container-registry-transfer-images.md`

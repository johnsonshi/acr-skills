# ACR Transfer Skill

This skill provides comprehensive knowledge about Azure Container Registry transfer for air-gapped environments.

## When to Use This Skill

Use this skill when answering questions about:
- Air-gapped transfers
- Export/import pipelines
- Cross-cloud transfers
- Disconnected environments

## Overview

ACR Transfer enables artifact transfer between registries in disconnected or air-gapped environments using Azure Storage as an intermediary.

**Requirements:** Premium SKU

## Architecture

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   Source ACR    │───▶│  Azure Storage  │───▶│   Target ACR    │
│   (Export)      │    │     (Blobs)     │    │   (Import)      │
└─────────────────┘    └─────────────────┘    └─────────────────┘
```

## Export Pipeline

### Create Export Pipeline
```bash
az acr pipeline create \
  --registry sourceregistry \
  --name myexport \
  --type export \
  --storage-account-container-uri https://mystorageaccount.blob.core.windows.net/mycontainer \
  --storage-account-sas $(az keyvault secret show --vault-name myvault --name export-sas --query value -o tsv)
```

### Run Export
```bash
az acr pipeline-run create \
  --registry sourceregistry \
  --pipeline-name myexport \
  --name export-run-1 \
  --artifacts "myapp:v1" "myapp:v2"
```

## Import Pipeline

### Create Import Pipeline
```bash
az acr pipeline create \
  --registry targetregistry \
  --name myimport \
  --type import \
  --storage-account-container-uri https://mystorageaccount.blob.core.windows.net/mycontainer \
  --storage-account-sas $(az keyvault secret show --vault-name myvault --name import-sas --query value -o tsv)
```

### Run Import (Manual)
```bash
az acr pipeline-run create \
  --registry targetregistry \
  --pipeline-name myimport \
  --name import-run-1 \
  --artifacts "myapp:v1" "myapp:v2"
```

### Auto-Import (Source Trigger)
```bash
az acr pipeline create \
  --registry targetregistry \
  --name myimport \
  --type import \
  --storage-account-container-uri https://mystorageaccount.blob.core.windows.net/mycontainer \
  --storage-account-sas $SAS \
  --source-trigger-enabled true
```

## Prerequisites

### 1. Storage Account
```bash
az storage account create \
  --name mystorageaccount \
  --resource-group myRG \
  --sku Standard_LRS

az storage container create \
  --name mycontainer \
  --account-name mystorageaccount
```

### 2. SAS Token
```bash
# Export SAS (write)
az storage container generate-sas \
  --account-name mystorageaccount \
  --name mycontainer \
  --permissions dlrw \
  --expiry 2025-01-01

# Import SAS (read, delete)
az storage container generate-sas \
  --account-name mystorageaccount \
  --name mycontainer \
  --permissions dlr \
  --expiry 2025-01-01
```

### 3. Key Vault for SAS
```bash
az keyvault secret set \
  --vault-name myvault \
  --name export-sas \
  --value "?sv=2023-11-03&sr=c&sig=..."
```

## Pipeline Options

### Export Options
| Option | Description |
|--------|-------------|
| `OverwriteBlobs` | Overwrite existing blobs |
| `ContinueOnErrors` | Continue on individual artifact errors |

### Import Options
| Option | Description |
|--------|-------------|
| `OverwriteTags` | Overwrite existing tags |
| `DeleteSourceBlobOnSuccess` | Cleanup after import |
| `ContinueOnErrors` | Continue on individual artifact errors |

## Management

```bash
# List pipelines
az acr pipeline list --registry myregistry -o table

# Show pipeline
az acr pipeline show --registry myregistry --name mypipeline

# List runs
az acr pipeline-run list --registry myregistry -o table

# Delete pipeline
az acr pipeline delete --registry myregistry --name mypipeline
```

## Air-Gapped Workflow

1. **Export** artifacts to storage
2. **Transfer** blobs (AzCopy, physical media)
3. **Import** from storage in target environment

```bash
# Export to storage
az acr pipeline-run create --registry source --pipeline-name export ...

# Transfer blobs (e.g., using AzCopy)
azcopy copy \
  "https://source-storage.blob.core.windows.net/container/*?$SAS" \
  "https://target-storage.blob.core.windows.net/container/?$SAS"

# Import from storage
az acr pipeline-run create --registry target --pipeline-name import ...
```

## vs ACR Import

| Feature | ACR Transfer | ACR Import |
|---------|--------------|------------|
| Air-gapped | ✅ | ❌ |
| Network restricted | ✅ | Limited |
| Bulk transfer | ✅ | Single image |
| Automation | Pipelines | One-off |

## Reference Documentation

- **Investigation reports**: `agent-investigation-reports/acr-features/acr-transfer/`
- **MS Learn docs**: `azure-management-docs/articles/container-registry/container-registry-transfer-images.md`

# ACR Transfer CLI Commands Reference

## Overview

This document provides a comprehensive reference for all Azure CLI commands related to ACR Transfer, including the `acrtransfer` extension commands and supporting commands.

## Prerequisites

### Install the acrtransfer Extension

```azurecli
az extension add --name acrtransfer
```

### Verify Installation

```azurecli
az extension list --output table
```

### Update Extension

```azurecli
az extension update --name acrtransfer
```

## Environment Variables Setup

```bash
# Source environment
SOURCE_RG="source-resource-group"
SOURCE_ACR="sourceacr"
SOURCE_SA="sourcestorageaccount"
SOURCE_KV="sourcekeyvault"

# Target environment
TARGET_RG="target-resource-group"
TARGET_ACR="targetacr"
TARGET_SA="targetstorageaccount"
TARGET_KV="targetkeyvault"
```

---

## Export Pipeline Commands

### Create Export Pipeline

#### Basic (System-Assigned Identity)

```azurecli
az acr export-pipeline create \
  --resource-group $SOURCE_RG \
  --registry $SOURCE_ACR \
  --name myExportPipeline \
  --secret-uri https://$SOURCE_KV.vault.azure.net/secrets/acrexportsas \
  --storage-container-uri https://$SOURCE_SA.blob.core.windows.net/transfer
```

#### With Options and User-Assigned Identity

```azurecli
az acr export-pipeline create \
  --resource-group $SOURCE_RG \
  --registry $SOURCE_ACR \
  --name myExportPipeline \
  --secret-uri https://$SOURCE_KV.vault.azure.net/secrets/acrexportsas \
  --storage-container-uri https://$SOURCE_SA.blob.core.windows.net/transfer \
  --options OverwriteBlobs ContinueOnErrors \
  --assign-identity /subscriptions/$SUB_ID/resourceGroups/$SOURCE_RG/providers/Microsoft.ManagedIdentity/userAssignedIdentities/myIdentity
```

### Show Export Pipeline

```azurecli
az acr export-pipeline show \
  --resource-group $SOURCE_RG \
  --registry $SOURCE_ACR \
  --name myExportPipeline
```

### List Export Pipelines

```azurecli
az acr export-pipeline list \
  --resource-group $SOURCE_RG \
  --registry $SOURCE_ACR
```

### Delete Export Pipeline

```azurecli
az acr export-pipeline delete \
  --resource-group $SOURCE_RG \
  --registry $SOURCE_ACR \
  --name myExportPipeline
```

---

## Import Pipeline Commands

### Create Import Pipeline

#### Basic (System-Assigned Identity)

```azurecli
az acr import-pipeline create \
  --resource-group $TARGET_RG \
  --registry $TARGET_ACR \
  --name myImportPipeline \
  --secret-uri https://$TARGET_KV.vault.azure.net/secrets/acrimportsas \
  --storage-container-uri https://$TARGET_SA.blob.core.windows.net/transfer
```

#### With Options, Disabled Trigger, and User-Assigned Identity

```azurecli
az acr import-pipeline create \
  --resource-group $TARGET_RG \
  --registry $TARGET_ACR \
  --name myImportPipeline \
  --secret-uri https://$TARGET_KV.vault.azure.net/secrets/acrimportsas \
  --storage-container-uri https://$TARGET_SA.blob.core.windows.net/transfer \
  --options DeleteSourceBlobOnSuccess OverwriteTags ContinueOnErrors \
  --assign-identity /subscriptions/$SUB_ID/resourceGroups/$TARGET_RG/providers/Microsoft.ManagedIdentity/userAssignedIdentities/myIdentity \
  --source-trigger-enabled False
```

### Show Import Pipeline

```azurecli
az acr import-pipeline show \
  --resource-group $TARGET_RG \
  --registry $TARGET_ACR \
  --name myImportPipeline
```

### List Import Pipelines

```azurecli
az acr import-pipeline list \
  --resource-group $TARGET_RG \
  --registry $TARGET_ACR
```

### Delete Import Pipeline

```azurecli
az acr import-pipeline delete \
  --resource-group $TARGET_RG \
  --registry $TARGET_ACR \
  --name myImportPipeline
```

---

## Pipeline Run Commands

### Create Export Pipeline Run

```azurecli
az acr pipeline-run create \
  --resource-group $SOURCE_RG \
  --registry $SOURCE_ACR \
  --pipeline myExportPipeline \
  --name myExportRun \
  --pipeline-type export \
  --storage-blob myexportblob \
  --artifacts hello-world:latest nginx:v1 myimage@sha256:abc123 \
  --force-redeploy
```

### Create Import Pipeline Run

```azurecli
az acr pipeline-run create \
  --resource-group $TARGET_RG \
  --registry $TARGET_ACR \
  --pipeline myImportPipeline \
  --name myImportRun \
  --pipeline-type import \
  --storage-blob myexportblob \
  --force-redeploy
```

### Show Pipeline Run

```azurecli
az acr pipeline-run show \
  --resource-group $SOURCE_RG \
  --registry $SOURCE_ACR \
  --name myExportRun
```

### List Pipeline Runs

```azurecli
az acr pipeline-run list \
  --resource-group $SOURCE_RG \
  --registry $SOURCE_ACR
```

### Delete Pipeline Run

```azurecli
az acr pipeline-run delete \
  --resource-group $SOURCE_RG \
  --registry $SOURCE_ACR \
  --name myExportRun
```

---

## Supporting Commands

### Storage Commands

#### Generate Export SAS Token

```azurecli
EXPORT_SAS=?$(az storage container generate-sas \
  --name transfer \
  --account-name $SOURCE_SA \
  --expiry 2025-01-01 \
  --permissions alrw \
  --https-only \
  --output tsv)
```

#### Generate Import SAS Token

```azurecli
IMPORT_SAS=?$(az storage container generate-sas \
  --name transfer \
  --account-name $TARGET_SA \
  --expiry 2025-01-01 \
  --permissions dlr \
  --https-only \
  --output tsv)
```

#### List Blobs in Container

```azurecli
az storage blob list \
  --account-name $SOURCE_SA \
  --container-name transfer \
  --output table
```

### Key Vault Commands

#### Store SAS Token Secret

```azurecli
az keyvault secret set \
  --name acrexportsas \
  --value "$EXPORT_SAS" \
  --vault-name $SOURCE_KV
```

#### Set Key Vault Access Policy

```azurecli
az keyvault set-policy \
  --name $SOURCE_KV \
  --secret-permissions get \
  --object-id $PRINCIPAL_ID
```

### Identity Commands

#### Get Pipeline Principal ID

```azurecli
# Export Pipeline
PRINCIPAL_ID=$(az acr export-pipeline show \
  --resource-group $SOURCE_RG \
  --registry $SOURCE_ACR \
  --name myExportPipeline \
  --query identity.principalId \
  --output tsv)

# Import Pipeline
PRINCIPAL_ID=$(az acr import-pipeline show \
  --resource-group $TARGET_RG \
  --registry $TARGET_ACR \
  --name myImportPipeline \
  --query identity.principalId \
  --output tsv)
```

#### Create User-Assigned Identity

```azurecli
az identity create \
  --name myTransferIdentity \
  --resource-group $SOURCE_RG
```

### Registry Commands

#### List Repositories

```azurecli
az acr repository list --name $TARGET_ACR
```

#### Show Tags

```azurecli
az acr repository show-tags \
  --name $TARGET_ACR \
  --repository myimage
```

---

## ARM Template Deployment Commands

### Deploy Export Pipeline

```azurecli
az deployment group create \
  --resource-group $SOURCE_RG \
  --template-file ExportPipelines/azuredeploy.json \
  --name exportPipeline \
  --parameters ExportPipelines/azuredeploy.parameters.json
```

### Deploy Import Pipeline

```azurecli
az deployment group create \
  --resource-group $TARGET_RG \
  --template-file ImportPipelines/azuredeploy.json \
  --name importPipeline \
  --parameters ImportPipelines/azuredeploy.parameters.json
```

### Deploy Pipeline Run (Export)

```azurecli
az deployment group create \
  --resource-group $SOURCE_RG \
  --template-file PipelineRun/PipelineRun-Export/azuredeploy.json \
  --name exportPipelineRun \
  --parameters PipelineRun/PipelineRun-Export/azuredeploy.parameters.json
```

### Deploy Pipeline Run (Import)

```azurecli
az deployment group create \
  --resource-group $TARGET_RG \
  --template-file PipelineRun/PipelineRun-Import/azuredeploy.json \
  --name importPipelineRun \
  --parameters PipelineRun/PipelineRun-Import/azuredeploy.parameters.json
```

### Redeploy with Force Update Tag

```azurecli
CURRENT_DATETIME=$(date +"%Y-%m-%d:%T")

az deployment group create \
  --resource-group $SOURCE_RG \
  --template-file PipelineRun/PipelineRun-Export/azuredeploy.json \
  --name exportPipelineRun \
  --parameters PipelineRun/PipelineRun-Export/azuredeploy.parameters.json \
  --parameters forceUpdateTag=$CURRENT_DATETIME
```

---

## Resource Deletion Commands

### Delete Using az resource delete

```azurecli
# Get resource IDs
EXPORT_PIPELINE_ID=$(az acr export-pipeline show \
  --resource-group $SOURCE_RG \
  --registry $SOURCE_ACR \
  --name myExportPipeline \
  --query id \
  --output tsv)

IMPORT_PIPELINE_ID=$(az acr import-pipeline show \
  --resource-group $TARGET_RG \
  --registry $TARGET_ACR \
  --name myImportPipeline \
  --query id \
  --output tsv)

# Delete resources
az resource delete \
  --resource-group $SOURCE_RG \
  --ids $EXPORT_PIPELINE_ID \
  --api-version 2019-12-01-preview

az resource delete \
  --resource-group $TARGET_RG \
  --ids $IMPORT_PIPELINE_ID \
  --api-version 2019-12-01-preview
```

---

## Data Loss Prevention Commands

### Disable Export Policy

```azurecli
az resource update \
  --resource-group $SOURCE_RG \
  --name $SOURCE_ACR \
  --resource-type "Microsoft.ContainerRegistry/registries" \
  --api-version "2021-06-01-preview" \
  --set "properties.policies.exportPolicy.status=disabled" \
  --set "properties.publicNetworkAccess=disabled"
```

### Enable Export Policy

```azurecli
az resource update \
  --resource-group $SOURCE_RG \
  --name $SOURCE_ACR \
  --resource-type "Microsoft.ContainerRegistry/registries" \
  --api-version "2021-06-01-preview" \
  --set "properties.policies.exportPolicy.status=enabled"
```

---

## Blob Transfer Commands (AzCopy)

### Copy Blob Between Storage Accounts

```bash
azcopy copy \
  "https://$SOURCE_SA.blob.core.windows.net/transfer/myblob$SOURCE_SAS" \
  "https://$TARGET_SA.blob.core.windows.net/transfer/myblob$TARGET_SAS" \
  --overwrite true
```

---

## Command Parameters Reference

### Export Pipeline Create Parameters

| Parameter | Required | Description |
|-----------|----------|-------------|
| `--resource-group` | Yes | Resource group name |
| `--registry` | Yes | Registry name |
| `--name` | Yes | Pipeline name |
| `--secret-uri` | Yes | Key Vault secret URI for SAS token |
| `--storage-container-uri` | Yes | Target storage container URI |
| `--options` | No | Space-separated list: OverwriteBlobs, ContinueOnErrors |
| `--assign-identity` | No | User-assigned identity resource ID |

### Import Pipeline Create Parameters

| Parameter | Required | Description |
|-----------|----------|-------------|
| `--resource-group` | Yes | Resource group name |
| `--registry` | Yes | Registry name |
| `--name` | Yes | Pipeline name |
| `--secret-uri` | Yes | Key Vault secret URI for SAS token |
| `--storage-container-uri` | Yes | Source storage container URI |
| `--options` | No | Space-separated list: OverwriteTags, DeleteSourceBlobOnSuccess, ContinueOnErrors |
| `--assign-identity` | No | User-assigned identity resource ID |
| `--source-trigger-enabled` | No | True/False to enable automatic import |

### Pipeline Run Create Parameters

| Parameter | Required | Description |
|-----------|----------|-------------|
| `--resource-group` | Yes | Resource group name |
| `--registry` | Yes | Registry name |
| `--pipeline` | Yes | Pipeline name |
| `--name` | Yes | Pipeline run name |
| `--pipeline-type` | Yes | export or import |
| `--storage-blob` | Yes | Target/source blob name |
| `--artifacts` | Export only | Space-separated list of artifacts |
| `--force-redeploy` | No | Force recreation of identical pipeline run |

## References

### Source Files
- Documentation: `/submodules/azure-management-docs/articles/container-registry/container-registry-transfer-cli.md`
- Documentation: `/submodules/azure-management-docs/articles/container-registry/container-registry-transfer-images.md`
- ARM Templates: `/submodules/acr/docs/image-transfer/`

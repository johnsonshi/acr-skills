# ACR Transfer Prerequisites

## Overview

Before using ACR Transfer, you must set up several Azure resources in both the source and target environments. This document provides a comprehensive checklist and configuration details.

## Prerequisites Checklist

### Required Resources

| Resource | Source Environment | Target Environment | Notes |
|----------|-------------------|-------------------|-------|
| Container Registry | Yes (Premium) | Yes (Premium) | Must be Premium SKU |
| Storage Account | Yes | Yes | With blob container |
| Key Vault | Yes | Yes | For SAS token secrets |
| Azure CLI | Yes (v2.2.0+) | Yes (v2.2.0+) | With acrtransfer extension |

## Container Registry Requirements

### SKU Requirement

ACR Transfer requires **Premium SKU** registries for both source and target.

```azurecli
# Check registry SKU
az acr show --name <registry-name> --query sku.name

# Upgrade to Premium if needed
az acr update --name <registry-name> --sku Premium
```

### Registry Permissions

Users creating ACR Transfer resources need at minimum:
- `Container Registry Transfer Pipeline Contributor` role on the subscription
- Or custom role with equivalent permissions

## Storage Account Setup

### Create Storage Account

```azurecli
# Source storage account
az storage account create \
  --name $SOURCE_SA \
  --resource-group $SOURCE_RG \
  --location $LOCATION \
  --sku Standard_LRS

# Target storage account
az storage account create \
  --name $TARGET_SA \
  --resource-group $TARGET_RG \
  --location $LOCATION \
  --sku Standard_LRS
```

### Create Blob Container

```azurecli
# Create transfer container in source storage
az storage container create \
  --name transfer \
  --account-name $SOURCE_SA

# Create transfer container in target storage
az storage container create \
  --name transfer \
  --account-name $TARGET_SA
```

## Key Vault Setup

### Create Key Vaults

```azurecli
# Source key vault
az keyvault create \
  --name $SOURCE_KV \
  --resource-group $SOURCE_RG \
  --location $LOCATION

# Target key vault
az keyvault create \
  --name $TARGET_KV \
  --resource-group $TARGET_RG \
  --location $LOCATION
```

## SAS Token Configuration

### Generate Export SAS Token

Required permissions for export: **Read, Write, List, Add**

```azurecli
EXPORT_SAS=?$(az storage container generate-sas \
  --name transfer \
  --account-name $SOURCE_SA \
  --expiry 2025-01-01 \
  --permissions alrw \
  --https-only \
  --output tsv)
```

### Store Export SAS Token in Key Vault

```azurecli
az keyvault secret set \
  --name acrexportsas \
  --value $EXPORT_SAS \
  --vault-name $SOURCE_KV
```

### Generate Import SAS Token

Required permissions for import: **Read, Delete, List**

Note: `Delete` permission is only required if using `DeleteSourceBlobOnSuccess` option.

```azurecli
IMPORT_SAS=?$(az storage container generate-sas \
  --name transfer \
  --account-name $TARGET_SA \
  --expiry 2025-01-01 \
  --permissions dlr \
  --https-only \
  --output tsv)
```

### Store Import SAS Token in Key Vault

```azurecli
az keyvault secret set \
  --name acrimportsas \
  --value $IMPORT_SAS \
  --vault-name $TARGET_KV
```

## SAS Token Best Practices

### Production Recommendation

For production workloads, use **Key Vault Managed Storage SAS Definition Secrets** instead of manually generated SAS tokens:

- Automatic rotation
- No manual token management
- Better security posture

Documentation: [Key Vault Managed Storage SAS](https://docs.microsoft.com/azure/key-vault/secrets/overview-storage-keys)

### SAS Token Properties

| Property | Export SAS | Import SAS |
|----------|-----------|-----------|
| Permissions | `alrw` (add, list, read, write) | `dlr` (delete, list, read) |
| HTTPS Only | Yes (`spr=https`) | Yes (`spr=https`) |
| Allowed Resource Types | Service, Container, Object (`srt=sco`) | Service, Container, Object (`srt=sco`) |
| Expiry | Set appropriate expiry date | Set appropriate expiry date |

### Validating SAS Token

To test if a SAS token is valid:

```bash
# Test export SAS by listing blobs
az storage blob list \
  --container-name transfer \
  --account-name $SOURCE_SA \
  --sas-token "$EXPORT_SAS"

# Test by uploading a test blob
az storage blob upload \
  --container-name transfer \
  --account-name $SOURCE_SA \
  --name test-blob \
  --file /path/to/test/file \
  --sas-token "$EXPORT_SAS"
```

## Environment Variables Template

For consistency across commands, set these environment variables:

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

## Azure CLI Extension Installation

### Install acrtransfer Extension

```azurecli
az extension add --name acrtransfer
```

### Verify Installation

```azurecli
az extension list --output table
```

## Managed Identity Considerations

### System-Assigned Identity

- Automatically created when pipeline is created
- No additional setup required
- Identity is deleted when pipeline is deleted

### User-Assigned Identity (Optional)

If using user-assigned identity:

```azurecli
# Create user-assigned identity
az identity create \
  --name myTransferIdentity \
  --resource-group $SOURCE_RG

# Get the identity resource ID
IDENTITY_ID=$(az identity show \
  --name myTransferIdentity \
  --resource-group $SOURCE_RG \
  --query id \
  --output tsv)
```

## Key Vault Access Policy Setup

After creating pipelines, you must grant their identities access to Key Vault:

### For System-Assigned Identity

```azurecli
# Get the pipeline's principal ID
PRINCIPAL_ID=$(az acr export-pipeline show \
  --resource-group $SOURCE_RG \
  --registry $SOURCE_ACR \
  --name myExportPipeline \
  --query identity.principalId \
  --output tsv)

# Grant secret get permission
az keyvault set-policy \
  --name $SOURCE_KV \
  --secret-permissions get \
  --object-id $PRINCIPAL_ID
```

### For User-Assigned Identity

```azurecli
# Get the identity's principal ID
PRINCIPAL_ID=$(az identity show \
  --name myTransferIdentity \
  --resource-group $SOURCE_RG \
  --query principalId \
  --output tsv)

# Grant secret get permission
az keyvault set-policy \
  --name $SOURCE_KV \
  --secret-permissions get \
  --object-id $PRINCIPAL_ID
```

## Network Considerations

### Firewall Configuration

If using storage account firewalls or Key Vault network restrictions:

1. Allow Azure services to access the storage account
2. Add specific IP ranges if needed
3. Consider using private endpoints for enhanced security

### Private Endpoints

For air-gapped scenarios, consider:
- Private endpoints for ACR
- Private endpoints for Storage Account
- Private endpoints for Key Vault

## Pre-Flight Validation Checklist

Before creating pipelines, verify:

- [ ] Source ACR exists and is Premium SKU
- [ ] Target ACR exists and is Premium SKU
- [ ] Source storage account exists with `transfer` container
- [ ] Target storage account exists with `transfer` container
- [ ] Source Key Vault exists with export SAS secret
- [ ] Target Key Vault exists with import SAS secret
- [ ] Azure CLI is version 2.2.0 or later
- [ ] acrtransfer extension is installed
- [ ] User has `Container Registry Transfer Pipeline Contributor` role

## References

### Source Files
- `/submodules/azure-management-docs/articles/container-registry/container-registry-transfer-prerequisites.md`
- `/submodules/acr/samples/dotnetcore/image-transfer/README.md`
- `/submodules/acr/samples/dotnetcore/registry-artifact-transfer/README.md`

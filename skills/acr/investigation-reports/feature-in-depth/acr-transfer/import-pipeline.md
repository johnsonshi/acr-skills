# ACR Import Pipeline

## Overview

An ImportPipeline is a long-lasting Azure resource that imports container images and OCI artifacts from an Azure Blob Storage container into an Azure Container Registry. It serves as the final step in the ACR Transfer workflow.

## Resource Definition

### Resource Type
```
Microsoft.ContainerRegistry/registries/importPipelines
```

### API Version
```
2019-12-01-preview
```

## Creating an Import Pipeline

### Using Azure CLI Extension (Recommended)

#### Basic Import Pipeline with System-Assigned Identity

```azurecli
az acr import-pipeline create \
  --resource-group $MyRG \
  --registry $MyReg \
  --name $MyPipeline \
  --secret-uri https://$MyKV.vault.azure.net/secrets/$MySecret \
  --storage-container-uri https://$MyStorage.blob.core.windows.net/$MyContainer
```

#### Import Pipeline with All Options and Disabled Trigger

```azurecli
az acr import-pipeline create \
  --resource-group $MyRG \
  --registry $MyReg \
  --name $MyPipeline \
  --secret-uri https://$MyKV.vault.azure.net/secrets/$MySecret \
  --storage-container-uri https://$MyStorage.blob.core.windows.net/$MyContainer \
  --options DeleteSourceBlobOnSuccess OverwriteTags ContinueOnErrors \
  --assign-identity /subscriptions/$MySubID/resourceGroups/$MyRG/providers/Microsoft.ManagedIdentity/userAssignedIdentities/$MyIdentity \
  --source-trigger-enabled False
```

### Using ARM Template

#### Template Structure

```json
{
  "$schema": "https://schema.management.azure.com/schemas/2015-01-01/deploymentTemplate.json#",
  "contentVersion": "1.0.0.0",
  "parameters": {
    "registryName": {
      "type": "string",
      "metadata": {
        "description": "Name of your Azure Container Registry"
      }
    },
    "importPipelineName": {
      "type": "string",
      "metadata": {
        "description": "Name of your import pipeline"
      }
    },
    "userAssignedIdentity": {
      "type": "string",
      "defaultValue": "",
      "metadata": {
        "description": "User-assigned identity resource ID (optional)"
      }
    },
    "sourceUri": {
      "type": "string",
      "metadata": {
        "description": "Storage container URI"
      }
    },
    "keyVaultName": {
      "type": "string",
      "metadata": {
        "description": "Key vault name"
      }
    },
    "sasTokenSecretName": {
      "type": "string",
      "metadata": {
        "description": "SAS token secret name in key vault"
      }
    },
    "sourceTriggerStatus": {
      "type": "string",
      "defaultValue": "Enabled",
      "allowedValues": ["Enabled", "Disabled"],
      "metadata": {
        "description": "Enable or disable automatic source trigger"
      }
    },
    "options": {
      "type": "array",
      "defaultValue": [],
      "metadata": {
        "description": "Import options"
      }
    }
  },
  "resources": [
    {
      "type": "Microsoft.ContainerRegistry/registries/importPipelines",
      "name": "[concat(parameters('registryName'), '/', parameters('importPipelineName'))]",
      "location": "[resourceGroup().location]",
      "apiVersion": "2019-12-01-preview",
      "identity": {
        "type": "[if(empty(parameters('userAssignedIdentity')), 'SystemAssigned', 'UserAssigned')]"
      },
      "properties": {
        "source": {
          "type": "AzureStorageBlobContainer",
          "uri": "[parameters('sourceUri')]",
          "keyVaultUri": "[concat(reference(resourceId('Microsoft.KeyVault/vaults', parameters('keyVaultName')), '2019-09-01').vaultUri, 'secrets/', parameters('sasTokenSecretName'))]"
        },
        "trigger": {
          "sourceTrigger": {
            "status": "[parameters('sourceTriggerStatus')]"
          }
        },
        "options": "[parameters('options')]"
      }
    }
  ]
}
```

#### Parameters File Example

```json
{
  "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentParameters.json#",
  "contentVersion": "1.0.0.0",
  "parameters": {
    "registryName": {
      "value": "myregistry"
    },
    "importPipelineName": {
      "value": "myImportPipeline"
    },
    "sourceUri": {
      "value": "https://mystorageaccount.blob.core.windows.net/transfer"
    },
    "keyVaultName": {
      "value": "mykeyvault"
    },
    "sasTokenSecretName": {
      "value": "acrimportsas"
    },
    "options": {
      "value": [
        "OverwriteTags",
        "DeleteSourceBlobOnSuccess",
        "ContinueOnErrors"
      ]
    }
  }
}
```

#### Deploy ARM Template

```azurecli
az deployment group create \
  --resource-group $TARGET_RG \
  --template-file azuredeploy.json \
  --name importPipeline \
  --parameters azuredeploy.parameters.json
```

## Import Pipeline Options

| Option | Description | Default |
|--------|-------------|---------|
| `OverwriteTags` | Overwrite existing tags in the target registry | Not enabled |
| `DeleteSourceBlobOnSuccess` | Delete the source blob after successful import | Not enabled |
| `ContinueOnErrors` | Continue importing remaining artifacts if one fails | Not enabled |

## Import Pipeline Properties

### Source Properties

| Property | Type | Description |
|----------|------|-------------|
| `type` | string | Always `AzureStorageBlobContainer` |
| `uri` | string | Storage container URI |
| `keyVaultUri` | string | Full URI to Key Vault secret containing SAS token |

### Trigger Properties

| Property | Type | Description |
|----------|------|-------------|
| `sourceTrigger.status` | string | `Enabled` or `Disabled` |

### Identity Properties

| Property | Type | Description |
|----------|------|-------------|
| `type` | string | `SystemAssigned` or `UserAssigned` |
| `principalId` | string | (Output) The principal ID of system-assigned identity |
| `userAssignedIdentities` | object | Map of user-assigned identity resource IDs |

## Source Trigger Behavior

When source trigger is **Enabled** (default):

1. Pipeline monitors the storage container for new blobs
2. Automatically imports blobs within **15 minutes** of arrival
3. Only imports blobs with:
   - `Last Modified` time within the last **60 days**
   - Valid `ContentMD5` property
   - Metadata containing `"category":"acr-transfer-blob"`

### Refreshing Old Blobs

For blobs older than 60 days, you can refresh the `Last Modified` time:

```azurecli
# Add metadata to refresh the blob
az storage blob metadata update \
  --account-name $TARGET_SA \
  --container-name transfer \
  --name myblob \
  --metadata refreshed=true
```

## Managing Import Pipelines

### View Import Pipeline

```azurecli
az acr import-pipeline show \
  --resource-group $MyRG \
  --registry $MyReg \
  --name $MyPipeline
```

### List Import Pipelines

```azurecli
az acr import-pipeline list \
  --resource-group $MyRG \
  --registry $MyReg
```

### Delete Import Pipeline

```azurecli
az acr import-pipeline delete \
  --resource-group $MyRG \
  --registry $MyReg \
  --name $MyPipeline
```

## Running an Import Pipeline Manually

If source trigger is disabled, or to manually trigger an import:

### Using Azure CLI

```azurecli
az acr pipeline-run create \
  --resource-group $MyRG \
  --registry $MyReg \
  --pipeline $MyPipeline \
  --name $MyPipelineRun \
  --pipeline-type import \
  --storage-blob $MyBlob \
  --force-redeploy
```

### Using ARM Template

```json
{
  "type": "Microsoft.ContainerRegistry/registries/pipelineRuns",
  "apiVersion": "2019-12-01-preview",
  "name": "[concat(parameters('registryName'), '/', parameters('pipelineRunName'))]",
  "properties": {
    "request": {
      "pipelineResourceId": "[parameters('importPipelineResourceId')]",
      "source": {
        "type": "AzureStorageBlob",
        "name": "myexportblob"
      }
    }
  }
}
```

## Key Vault Access Policy

After creating the pipeline, grant its identity access to Key Vault:

```azurecli
# Get the pipeline's principal ID
PRINCIPAL_ID=$(az acr import-pipeline show \
  --resource-group $MyRG \
  --registry $MyReg \
  --name $MyPipeline \
  --query identity.principalId \
  --output tsv)

# Grant secret get permission
az keyvault set-policy \
  --name $MyKeyvault \
  --secret-permissions get \
  --object-id $PRINCIPAL_ID
```

## Verifying Import

After the pipeline runs, verify the import:

```azurecli
# List repositories in the target registry
az acr repository list --name $TARGET_ACR

# Show tags for a specific repository
az acr repository show-tags --name $TARGET_ACR --repository myimage
```

## .NET SDK Example

```csharp
var importPipeline = new ImportPipeline(
    name: "myImportPipeline",
    location: registry.Location,
    identity: new IdentityProperties { Type = ResourceIdentityType.SystemAssigned },
    source: new ImportPipelineSourceProperties
    {
        Type = "AzureStorageBlobContainer",
        Uri = "https://mystorageaccount.blob.core.windows.net/transfer",
        KeyVaultUri = "https://mykeyvault.vault.azure.net/secrets/acrimportsas"
    },
    trigger: new PipelineTriggerProperties
    {
        SourceTrigger = new PipelineSourceTriggerProperties
        {
            Status = "Enabled"
        }
    },
    options: new List<string> { "OverwriteTags", "DeleteSourceBlobOnSuccess", "ContinueOnErrors" }
);

var result = await registryClient.ImportPipelines.CreateAsync(
    registryName: "myregistry",
    resourceGroupName: "myResourceGroup",
    importPipelineName: "myImportPipeline",
    importPipelineCreateParameters: importPipeline
);
```

## Import Pipeline vs ACR Import

| Feature | Import Pipeline | ACR Import |
|---------|----------------|------------|
| **Use Case** | Air-gapped/disconnected clouds | Connected clouds |
| **Source** | Azure Blob Storage | Container Registry |
| **Trigger** | Automatic or manual | Manual only |
| **Requires Premium** | Yes | No (Basic+) |
| **Network** | Works across disconnected networks | Requires network connectivity |

## Import Pipeline Lifecycle

```
1. Create ImportPipeline Resource
         │
         ▼
2. Configure Key Vault Access Policy
         │
         ▼
3. (If source trigger enabled)
   └── Automatic import when blob arrives
         │
   (If source trigger disabled)
   └── Create PipelineRun manually
         │
         ▼
4. Verify Import in Target Registry
         │
         ▼
5. (Optional) Delete when no longer needed
```

## Source Trigger Requirements Checklist

For automatic imports to work, ensure:

- [ ] Source trigger status is `Enabled`
- [ ] Blob `Last Modified` is within 60 days
- [ ] Blob has valid `ContentMD5` property
- [ ] Blob has metadata: `"category":"acr-transfer-blob"`
- [ ] SAS token has `List` permission
- [ ] Pipeline identity has Key Vault access

## References

### Source Files
- ARM Template: `/submodules/acr/docs/image-transfer/ImportPipelines/azuredeploy.json`
- Parameters: `/submodules/acr/docs/image-transfer/ImportPipelines/azuredeploy.parameters.json`
- .NET Client: `/submodules/acr/samples/dotnetcore/image-transfer/ContainerRegistryTransfer/Clients/ImportClient.cs`
- Documentation: `/submodules/azure-management-docs/articles/container-registry/container-registry-transfer-cli.md`

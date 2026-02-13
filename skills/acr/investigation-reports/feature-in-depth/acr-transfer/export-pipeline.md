# ACR Export Pipeline

## Overview

An ExportPipeline is a long-lasting Azure resource that exports container images and OCI artifacts from an Azure Container Registry to an Azure Blob Storage container. It serves as the first step in the ACR Transfer workflow.

## Resource Definition

### Resource Type
```
Microsoft.ContainerRegistry/registries/exportPipelines
```

### API Version
```
2019-12-01-preview
```

## Creating an Export Pipeline

### Using Azure CLI Extension (Recommended)

#### Basic Export Pipeline with System-Assigned Identity

```azurecli
az acr export-pipeline create \
  --resource-group $MyRG \
  --registry $MyReg \
  --name $MyPipeline \
  --secret-uri https://$MyKV.vault.azure.net/secrets/$MySecret \
  --storage-container-uri https://$MyStorage.blob.core.windows.net/$MyContainer
```

#### Export Pipeline with All Options and User-Assigned Identity

```azurecli
az acr export-pipeline create \
  --resource-group $MyRG \
  --registry $MyReg \
  --name $MyPipeline \
  --secret-uri https://$MyKV.vault.azure.net/secrets/$MySecret \
  --storage-container-uri https://$MyStorage.blob.core.windows.net/$MyContainer \
  --options OverwriteBlobs ContinueOnErrors \
  --assign-identity /subscriptions/$MySubID/resourceGroups/$MyRG/providers/Microsoft.ManagedIdentity/userAssignedIdentities/$MyIdentity
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
    "exportPipelineName": {
      "type": "string",
      "metadata": {
        "description": "Name of your export pipeline"
      }
    },
    "userAssignedIdentity": {
      "type": "string",
      "defaultValue": "",
      "metadata": {
        "description": "User-assigned identity resource ID (optional)"
      }
    },
    "targetUri": {
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
    "options": {
      "type": "array",
      "defaultValue": [],
      "metadata": {
        "description": "Export options"
      }
    }
  },
  "resources": [
    {
      "type": "Microsoft.ContainerRegistry/registries/exportPipelines",
      "name": "[concat(parameters('registryName'), '/', parameters('exportPipelineName'))]",
      "location": "[resourceGroup().location]",
      "apiVersion": "2019-12-01-preview",
      "identity": {
        "type": "[if(empty(parameters('userAssignedIdentity')), 'SystemAssigned', 'UserAssigned')]"
      },
      "properties": {
        "target": {
          "type": "AzureStorageBlobContainer",
          "uri": "[parameters('targetUri')]",
          "keyVaultUri": "[concat(reference(resourceId('Microsoft.KeyVault/vaults', parameters('keyVaultName')), '2019-09-01').vaultUri, 'secrets/', parameters('sasTokenSecretName'))]"
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
    "exportPipelineName": {
      "value": "myExportPipeline"
    },
    "targetUri": {
      "value": "https://mystorageaccount.blob.core.windows.net/transfer"
    },
    "keyVaultName": {
      "value": "mykeyvault"
    },
    "sasTokenSecretName": {
      "value": "acrexportsas"
    },
    "options": {
      "value": [
        "OverwriteBlobs",
        "ContinueOnErrors"
      ]
    }
  }
}
```

#### Deploy ARM Template

```azurecli
az deployment group create \
  --resource-group $SOURCE_RG \
  --template-file azuredeploy.json \
  --name exportPipeline \
  --parameters azuredeploy.parameters.json
```

## Export Pipeline Options

| Option | Description | Default |
|--------|-------------|---------|
| `OverwriteBlobs` | Overwrite existing target blobs with the same name | Not enabled |
| `ContinueOnErrors` | Continue exporting remaining artifacts if one fails | Not enabled |

## Export Pipeline Properties

### Target Properties

| Property | Type | Description |
|----------|------|-------------|
| `type` | string | Always `AzureStorageBlobContainer` |
| `uri` | string | Storage container URI (e.g., `https://account.blob.core.windows.net/container`) |
| `keyVaultUri` | string | Full URI to Key Vault secret containing SAS token |

### Identity Properties

| Property | Type | Description |
|----------|------|-------------|
| `type` | string | `SystemAssigned` or `UserAssigned` |
| `principalId` | string | (Output) The principal ID of system-assigned identity |
| `userAssignedIdentities` | object | Map of user-assigned identity resource IDs |

## Managing Export Pipelines

### View Export Pipeline

```azurecli
az acr export-pipeline show \
  --resource-group $MyRG \
  --registry $MyReg \
  --name $MyPipeline
```

### List Export Pipelines

```azurecli
az acr export-pipeline list \
  --resource-group $MyRG \
  --registry $MyReg
```

### Delete Export Pipeline

```azurecli
az acr export-pipeline delete \
  --resource-group $MyRG \
  --registry $MyReg \
  --name $MyPipeline
```

## Running an Export Pipeline

To execute an export, create a PipelineRun resource:

### Using Azure CLI

```azurecli
az acr pipeline-run create \
  --resource-group $MyRG \
  --registry $MyReg \
  --pipeline $MyPipeline \
  --name $MyPipelineRun \
  --pipeline-type export \
  --storage-blob $MyBlob \
  --artifacts hello-world:latest nginx:v1 myimage@sha256:abc123... \
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
      "pipelineResourceId": "[parameters('exportPipelineResourceId')]",
      "artifacts": [
        "hello-world:latest",
        "nginx:v1"
      ],
      "target": {
        "type": "AzureStorageBlob",
        "name": "myexportblob"
      }
    }
  }
}
```

## Artifacts Specification

Artifacts can be specified using:

| Format | Example | Description |
|--------|---------|-------------|
| Tag | `myimage:v1` | Image with specific tag |
| Digest | `myimage@sha256:abc123...` | Image with specific digest |
| Repository:Tag | `samples/hello-world:latest` | Full repository path with tag |

### Limits

- **Maximum 50 artifacts per PipelineRun**
- **Maximum 8 GB per layer**

## Key Vault Access Policy

After creating the pipeline, grant its identity access to Key Vault:

```azurecli
# Get the pipeline's principal ID
PRINCIPAL_ID=$(az acr export-pipeline show \
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

## Verifying Export

After running the pipeline, verify the export:

```azurecli
# List blobs in the storage container
az storage blob list \
  --account-name $SOURCE_SA \
  --container transfer \
  --output table
```

## .NET SDK Example

```csharp
var exportPipeline = new ExportPipeline(
    name: "myExportPipeline",
    location: registry.Location,
    identity: new IdentityProperties { Type = ResourceIdentityType.SystemAssigned },
    target: new ExportPipelineTargetProperties
    {
        Type = "AzureStorageBlobContainer",
        Uri = "https://mystorageaccount.blob.core.windows.net/transfer",
        KeyVaultUri = "https://mykeyvault.vault.azure.net/secrets/acrexportsas"
    },
    options: new List<string> { "OverwriteBlobs", "ContinueOnErrors" }
);

var result = await registryClient.ExportPipelines.CreateAsync(
    registryName: "myregistry",
    resourceGroupName: "myResourceGroup",
    exportPipelineName: "myExportPipeline",
    exportPipelineCreateParameters: exportPipeline
);
```

## Export Pipeline Lifecycle

```
1. Create ExportPipeline Resource
         │
         ▼
2. Configure Key Vault Access Policy
         │
         ▼
3. Create PipelineRun (one or more times)
         │
         ▼
4. Verify Blob Export
         │
         ▼
5. (Optional) Delete when no longer needed
```

## References

### Source Files
- ARM Template: `/submodules/acr/docs/image-transfer/ExportPipelines/azuredeploy.json`
- Parameters: `/submodules/acr/docs/image-transfer/ExportPipelines/azuredeploy.parameters.json`
- .NET Client: `/submodules/acr/samples/dotnetcore/image-transfer/ContainerRegistryTransfer/Clients/ExportClient.cs`
- Documentation: `/submodules/azure-management-docs/articles/container-registry/container-registry-transfer-cli.md`

# Azure Container Registry Data Loss Prevention - CLI Commands Reference

## Overview

This document provides a comprehensive reference for all CLI commands related to Data Loss Prevention features in Azure Container Registry, including export policy management, network configuration, and related operations.

## Export Policy Commands

### View Current Export Policy Status

```bash
# View export policy status
az acr show \
  --name myregistry \
  --query "properties.policies.exportPolicy" \
  --output json

# View only the status value
az acr show \
  --name myregistry \
  --query "properties.policies.exportPolicy.status" \
  --output tsv
```

### Disable Export Policy (Enable DLP)

```bash
# Disable export policy (requires public network access to also be disabled)
az resource update \
  --resource-group myResourceGroup \
  --name myregistry \
  --resource-type "Microsoft.ContainerRegistry/registries" \
  --api-version "2021-06-01-preview" \
  --set "properties.policies.exportPolicy.status=disabled" \
  --set "properties.publicNetworkAccess=disabled"
```

### Enable Export Policy (Disable DLP)

```bash
# Re-enable export policy
az resource update \
  --resource-group myResourceGroup \
  --name myregistry \
  --resource-type "Microsoft.ContainerRegistry/registries" \
  --api-version "2021-06-01-preview" \
  --set "properties.policies.exportPolicy.status=enabled"
```

### View All Registry Policies

```bash
# View all policies including export, retention, trust, quarantine
az acr show \
  --name myregistry \
  --query "properties.policies" \
  --output json
```

Expected output:
```json
{
  "exportPolicy": {
    "status": "disabled"
  },
  "quarantinePolicy": {
    "status": "disabled"
  },
  "retentionPolicy": {
    "days": 7,
    "lastUpdatedTime": "2023-07-20T23:20:30.9985256+00:00",
    "status": "disabled"
  },
  "trustPolicy": {
    "status": "disabled",
    "type": "Notary"
  }
}
```

## Network Security Commands

### Public Network Access

```bash
# Check public network access status
az acr show \
  --name myregistry \
  --query "publicNetworkAccess" \
  --output tsv

# Disable public network access
az acr update \
  --name myregistry \
  --public-network-enabled false

# Enable public network access
az acr update \
  --name myregistry \
  --public-network-enabled true
```

### Private Endpoint Commands

#### List Private Endpoints

```bash
# List all private endpoint connections
az acr private-endpoint-connection list \
  --registry-name myregistry \
  --output table
```

#### Approve Private Endpoint Connection

```bash
# Approve a pending private endpoint connection
az acr private-endpoint-connection approve \
  --registry-name myregistry \
  --name myPrivateEndpoint
```

#### Reject Private Endpoint Connection

```bash
# Reject a private endpoint connection
az acr private-endpoint-connection reject \
  --registry-name myregistry \
  --name myPrivateEndpoint
```

#### Delete Private Endpoint Connection

```bash
# Delete a private endpoint connection
az acr private-endpoint-connection delete \
  --registry-name myregistry \
  --name myPrivateEndpoint
```

#### Show Private Endpoint Details

```bash
# Show details of a specific private endpoint connection
az acr private-endpoint-connection show \
  --registry-name myregistry \
  --name myPrivateEndpoint
```

### Create Private Endpoint (Full Workflow)

```bash
# Set variables
REGISTRY_NAME="myregistry"
RESOURCE_GROUP="myResourceGroup"
VNET_NAME="myVNet"
SUBNET_NAME="mySubnet"
REGISTRY_LOCATION="eastus"

# Get registry ID
REGISTRY_ID=$(az acr show --name $REGISTRY_NAME --query 'id' --output tsv)

# Create private DNS zone
az network private-dns zone create \
  --resource-group $RESOURCE_GROUP \
  --name "privatelink.azurecr.io"

# Create virtual network link
az network private-dns link vnet create \
  --resource-group $RESOURCE_GROUP \
  --zone-name "privatelink.azurecr.io" \
  --name MyDNSLink \
  --virtual-network $VNET_NAME \
  --registration-enabled false

# Create private endpoint
az network private-endpoint create \
  --name myPrivateEndpoint \
  --resource-group $RESOURCE_GROUP \
  --vnet-name $VNET_NAME \
  --subnet $SUBNET_NAME \
  --private-connection-resource-id $REGISTRY_ID \
  --group-ids registry \
  --connection-name myConnection
```

## Dedicated Data Endpoints Commands

### Enable Dedicated Data Endpoints

```bash
# Enable dedicated data endpoints
az acr update \
  --name myregistry \
  --data-endpoint-enabled
```

### Disable Dedicated Data Endpoints

```bash
# Disable dedicated data endpoints
az acr update \
  --name myregistry \
  --data-endpoint-enabled false
```

### View Data Endpoints

```bash
# Show all endpoints including data endpoints
az acr show-endpoints \
  --name myregistry \
  --output json
```

Expected output:
```json
{
  "loginServer": "myregistry.azurecr.io",
  "dataEndpoints": [
    {
      "region": "eastus",
      "endpoint": "myregistry.eastus.data.azurecr.io"
    },
    {
      "region": "westus",
      "endpoint": "myregistry.westus.data.azurecr.io"
    }
  ]
}
```

## ACR Transfer Pipeline Commands (Blocked by Export Policy)

### Export Pipeline Commands

```bash
# Install the acrtransfer extension
az extension add --name acrtransfer

# List export pipelines
az acr export-pipeline list \
  --registry myregistry \
  --resource-group myResourceGroup

# Create export pipeline (will fail if export policy is disabled)
az acr export-pipeline create \
  --resource-group myResourceGroup \
  --registry myregistry \
  --name myExportPipeline \
  --secret-uri https://mykv.vault.azure.net/secrets/mysecret \
  --storage-container-uri https://mystorage.blob.core.windows.net/transfer

# Delete export pipeline (must be done before disabling export policy)
az acr export-pipeline delete \
  --resource-group myResourceGroup \
  --registry myregistry \
  --name myExportPipeline
```

### Import Pipeline Commands

```bash
# List import pipelines (these continue to work)
az acr import-pipeline list \
  --registry myregistry \
  --resource-group myResourceGroup

# Create import pipeline (imports INTO registry still allowed)
az acr import-pipeline create \
  --resource-group myResourceGroup \
  --registry myregistry \
  --name myImportPipeline \
  --secret-uri https://mykv.vault.azure.net/secrets/mysecret \
  --storage-container-uri https://mystorage.blob.core.windows.net/transfer
```

### Pipeline Run Commands

```bash
# Create export pipeline run (will fail if export policy is disabled)
az acr pipeline-run create \
  --resource-group myResourceGroup \
  --registry myregistry \
  --pipeline myExportPipeline \
  --name myExportRun \
  --pipeline-type export \
  --storage-blob myblob \
  --artifacts "hello-world:latest"

# Create import pipeline run (works regardless of export policy)
az acr pipeline-run create \
  --resource-group myResourceGroup \
  --registry myregistry \
  --pipeline myImportPipeline \
  --name myImportRun \
  --pipeline-type import \
  --storage-blob myblob
```

## Image Import Commands (Affected by Export Policy)

### Import TO Protected Registry (Allowed)

```bash
# Import from Docker Hub (works even with export policy disabled)
az acr import \
  --name myprotectedregistry \
  --source docker.io/library/hello-world:latest \
  --image hello-world:latest
```

### Import FROM Protected Registry (Blocked)

```bash
# This will FAIL if source registry has export policy disabled
az acr import \
  --name targetregistry \
  --source myprotectedregistry.azurecr.io/hello-world:latest \
  --image hello-world:latest
```

### Import with Service Principal

```bash
# Using service principal credentials
az acr import \
  --name targetregistry \
  --source myprotectedregistry.azurecr.io/hello-world:latest \
  --image hello-world:latest \
  --username <SP_App_ID> \
  --password <SP_Passwd>
```

## Diagnostic and Monitoring Commands

### Enable Diagnostic Settings

```bash
# Create Log Analytics workspace
az monitor log-analytics workspace create \
  --resource-group myResourceGroup \
  --workspace-name myLogWorkspace

# Get workspace ID
WORKSPACE_ID=$(az monitor log-analytics workspace show \
  --resource-group myResourceGroup \
  --workspace-name myLogWorkspace \
  --query id --output tsv)

# Get registry ID
REGISTRY_ID=$(az acr show --name myregistry --query id --output tsv)

# Enable diagnostic settings
az monitor diagnostic-settings create \
  --name acrDiagnostics \
  --resource $REGISTRY_ID \
  --workspace $WORKSPACE_ID \
  --logs '[
    {"category": "ContainerRegistryRepositoryEvents", "enabled": true},
    {"category": "ContainerRegistryLoginEvents", "enabled": true}
  ]' \
  --metrics '[{"category": "AllMetrics", "enabled": true}]'
```

### View Diagnostic Settings

```bash
# List diagnostic settings
az monitor diagnostic-settings list \
  --resource $(az acr show --name myregistry --query id --output tsv)
```

### Health Check

```bash
# Check registry health (includes network connectivity)
az acr check-health \
  --name myregistry \
  --vnet myVNet
```

## Full DLP Configuration Script

```bash
#!/bin/bash
# Complete DLP configuration script

# Variables
REGISTRY_NAME="myregistry"
RESOURCE_GROUP="myResourceGroup"
LOCATION="eastus"

# 1. Verify Premium SKU
echo "Checking registry SKU..."
SKU=$(az acr show --name $REGISTRY_NAME --query sku.name --output tsv)
if [ "$SKU" != "Premium" ]; then
    echo "Upgrading to Premium SKU..."
    az acr update --name $REGISTRY_NAME --sku Premium
fi

# 2. Delete any existing export pipelines
echo "Checking for existing export pipelines..."
PIPELINES=$(az acr export-pipeline list --registry $REGISTRY_NAME --resource-group $RESOURCE_GROUP --query "[].name" --output tsv)
for PIPELINE in $PIPELINES; do
    echo "Deleting export pipeline: $PIPELINE"
    az acr export-pipeline delete --registry $REGISTRY_NAME --resource-group $RESOURCE_GROUP --name $PIPELINE --yes
done

# 3. Enable dedicated data endpoints
echo "Enabling dedicated data endpoints..."
az acr update --name $REGISTRY_NAME --data-endpoint-enabled

# 4. Disable exports and public network access
echo "Disabling export policy and public network access..."
az resource update \
  --resource-group $RESOURCE_GROUP \
  --name $REGISTRY_NAME \
  --resource-type "Microsoft.ContainerRegistry/registries" \
  --api-version "2021-06-01-preview" \
  --set "properties.policies.exportPolicy.status=disabled" \
  --set "properties.publicNetworkAccess=disabled"

# 5. Verify configuration
echo "Verifying configuration..."
az acr show --name $REGISTRY_NAME --query "{
  exportPolicy: properties.policies.exportPolicy.status,
  publicNetworkAccess: publicNetworkAccess,
  dataEndpointEnabled: dataEndpointEnabled
}" --output json

echo "DLP configuration complete!"
```

## Azure PowerShell Equivalents

### View Export Policy

```powershell
Get-AzContainerRegistry -ResourceGroupName myResourceGroup -Name myregistry |
    Select-Object -ExpandProperty Policies
```

### Disable Export Policy

```powershell
$uri = "/subscriptions/$subscriptionId/resourceGroups/myResourceGroup/providers/Microsoft.ContainerRegistry/registries/myregistry?api-version=2021-06-01-preview"

$body = @{
    properties = @{
        publicNetworkAccess = "disabled"
        policies = @{
            exportPolicy = @{
                status = "disabled"
            }
        }
    }
} | ConvertTo-Json -Depth 10

Invoke-AzRestMethod -Method PATCH -Path $uri -Payload $body
```

### Import Image

```powershell
Import-AzContainerRegistryImage `
    -RegistryName myregistry `
    -ResourceGroupName myResourceGroup `
    -SourceRegistryUri docker.io `
    -SourceImage library/hello-world:latest
```

## Error Reference

| Error Message | Cause | Solution |
|---------------|-------|----------|
| `Export policy can only be disabled for registries with private endpoints` | No private endpoint configured | Create private endpoint first |
| `Cannot disable export policy. Registry has existing export pipelines` | Export pipelines exist | Delete all export pipelines |
| `Import from source registry with export policy disabled is not allowed` | Source registry has exports disabled | Cannot import from this registry |
| `Cannot create export pipeline. Export policy is disabled` | Export policy is disabled | Enable export policy first |

## Source References

- Primary documentation: `/Users/johnsonshi/repos/acr-skills/submodules/azure-management-docs/articles/container-registry/data-loss-prevention.md`
- ACR Transfer CLI: `/Users/johnsonshi/repos/acr-skills/submodules/azure-management-docs/articles/container-registry/container-registry-transfer-cli.md`
- Private link: `/Users/johnsonshi/repos/acr-skills/submodules/azure-management-docs/articles/container-registry/container-registry-private-link.md`
- Import images: `/Users/johnsonshi/repos/acr-skills/submodules/azure-management-docs/articles/container-registry/container-registry-import-images.md`

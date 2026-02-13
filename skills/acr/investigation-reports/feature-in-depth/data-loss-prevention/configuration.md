# Azure Container Registry Data Loss Prevention - Configuration Guide

## Overview

This guide provides comprehensive instructions for configuring Data Loss Prevention (DLP) features in Azure Container Registry, including export policy, network restrictions, and dedicated data endpoints.

## Complete DLP Configuration Workflow

### Step 1: Verify Prerequisites

#### 1.1 Check Registry SKU

```bash
# Verify registry is Premium SKU
az acr show --name myregistry --query sku.name --output tsv
# Expected output: Premium
```

If not Premium, upgrade:
```bash
az acr update --name myregistry --sku Premium
```

#### 1.2 Check Existing Export Pipelines

```bash
# List all export pipelines (must be empty before disabling exports)
az acr export-pipeline list \
  --registry myregistry \
  --resource-group myResourceGroup \
  --output table
```

Delete any existing pipelines:
```bash
az acr export-pipeline delete \
  --resource-group myResourceGroup \
  --registry myregistry \
  --name existingPipeline
```

### Step 2: Configure Network Security

#### 2.1 Create Private Endpoint

Using Azure Portal:
1. Navigate to your container registry
2. Select **Networking** > **Private access**
3. Click **Create a private endpoint connection**
4. Configure:
   - Subscription and Resource Group
   - Name for the endpoint
   - Virtual network and subnet
   - Private DNS zone integration: **Yes**

Using Azure CLI:
```bash
# Set variables
REGISTRY_NAME="myregistry"
REGISTRY_LOCATION="eastus"
RESOURCE_GROUP="myResourceGroup"
VNET_NAME="myVNet"
SUBNET_NAME="mySubnet"

# Get registry resource ID
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

#### 2.2 Configure DNS Records

```bash
# Get network interface ID
NETWORK_INTERFACE_ID=$(az network private-endpoint show \
  --name myPrivateEndpoint \
  --resource-group $RESOURCE_GROUP \
  --query 'networkInterfaces[0].id' \
  --output tsv)

# Get private IP addresses
REGISTRY_PRIVATE_IP=$(az network nic show \
  --ids $NETWORK_INTERFACE_ID \
  --query "ipConfigurations[?privateLinkConnectionProperties.requiredMemberName=='registry'].privateIPAddress" \
  --output tsv)

DATA_ENDPOINT_PRIVATE_IP=$(az network nic show \
  --ids $NETWORK_INTERFACE_ID \
  --query "ipConfigurations[?privateLinkConnectionProperties.requiredMemberName=='registry_data_${REGISTRY_LOCATION}'].privateIPAddress" \
  --output tsv)

# Create DNS A records
az network private-dns record-set a create \
  --name $REGISTRY_NAME \
  --zone-name privatelink.azurecr.io \
  --resource-group $RESOURCE_GROUP

az network private-dns record-set a add-record \
  --record-set-name $REGISTRY_NAME \
  --zone-name privatelink.azurecr.io \
  --resource-group $RESOURCE_GROUP \
  --ipv4-address $REGISTRY_PRIVATE_IP

az network private-dns record-set a create \
  --name ${REGISTRY_NAME}.${REGISTRY_LOCATION}.data \
  --zone-name privatelink.azurecr.io \
  --resource-group $RESOURCE_GROUP

az network private-dns record-set a add-record \
  --record-set-name ${REGISTRY_NAME}.${REGISTRY_LOCATION}.data \
  --zone-name privatelink.azurecr.io \
  --resource-group $RESOURCE_GROUP \
  --ipv4-address $DATA_ENDPOINT_PRIVATE_IP
```

### Step 3: Disable Public Network Access and Export Policy

#### Option A: Using Azure CLI (Recommended)

```bash
az resource update \
  --resource-group myResourceGroup \
  --name myregistry \
  --resource-type "Microsoft.ContainerRegistry/registries" \
  --api-version "2021-06-01-preview" \
  --set "properties.policies.exportPolicy.status=disabled" \
  --set "properties.publicNetworkAccess=disabled"
```

#### Option B: Using ARM Template

Create a file `dlp-config.json`:

```json
{
  "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
  "contentVersion": "1.0.0.0",
  "parameters": {
    "registryName": {
      "type": "string",
      "metadata": {
        "description": "Name of the container registry"
      }
    },
    "location": {
      "type": "string",
      "defaultValue": "[resourceGroup().location]"
    }
  },
  "resources": [
    {
      "type": "Microsoft.ContainerRegistry/registries",
      "apiVersion": "2021-06-01-preview",
      "name": "[parameters('registryName')]",
      "location": "[parameters('location')]",
      "sku": {
        "name": "Premium"
      },
      "properties": {
        "publicNetworkAccess": "disabled",
        "policies": {
          "exportPolicy": {
            "status": "disabled"
          }
        }
      }
    }
  ]
}
```

Deploy the template:
```bash
az deployment group create \
  --resource-group myResourceGroup \
  --template-file dlp-config.json \
  --parameters registryName=myregistry
```

#### Option C: Using Azure PowerShell

```powershell
$registry = Get-AzContainerRegistry -ResourceGroupName myResourceGroup -Name myregistry

# Note: Direct PowerShell cmdlets for export policy may require REST API calls
# Use Invoke-AzRestMethod for fine-grained control

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

### Step 4: Enable Dedicated Data Endpoints (Optional but Recommended)

```bash
# Enable dedicated data endpoints
az acr update --name myregistry --data-endpoint-enabled

# View the data endpoints
az acr show-endpoints --name myregistry
```

Expected output:
```json
{
  "loginServer": "myregistry.azurecr.io",
  "dataEndpoints": [
    {
      "region": "eastus",
      "endpoint": "myregistry.eastus.data.azurecr.io"
    }
  ]
}
```

### Step 5: Configure Diagnostic Logging

```bash
# Create Log Analytics workspace if needed
az monitor log-analytics workspace create \
  --resource-group myResourceGroup \
  --workspace-name myLogWorkspace

# Get workspace ID
WORKSPACE_ID=$(az monitor log-analytics workspace show \
  --resource-group myResourceGroup \
  --workspace-name myLogWorkspace \
  --query id \
  --output tsv)

# Enable diagnostic settings
az monitor diagnostic-settings create \
  --name acrDiagnostics \
  --resource $(az acr show --name myregistry --query id --output tsv) \
  --workspace $WORKSPACE_ID \
  --logs '[
    {"category": "ContainerRegistryRepositoryEvents", "enabled": true},
    {"category": "ContainerRegistryLoginEvents", "enabled": true}
  ]' \
  --metrics '[{"category": "AllMetrics", "enabled": true}]'
```

## Verification Steps

### Verify Export Policy is Disabled

```bash
az acr show --name myregistry \
  --query "properties.policies.exportPolicy" \
  --output json
```

Expected output:
```json
{
  "status": "disabled"
}
```

### Verify Public Network Access is Disabled

```bash
az acr show --name myregistry \
  --query "publicNetworkAccess" \
  --output tsv
```

Expected output: `Disabled`

### Verify Private Endpoint Connection

```bash
az acr private-endpoint-connection list \
  --registry-name myregistry \
  --output table
```

### Test Blocked Export Operation

From another registry, attempt to import (should fail):
```bash
# This should fail with permission denied
az acr import \
  --name otherregistry \
  --source myregistry.azurecr.io/test:latest \
  --image test:latest
```

## Re-enabling Exports

If you need to temporarily re-enable exports:

```bash
# Re-enable export policy
az resource update \
  --resource-group myResourceGroup \
  --name myregistry \
  --resource-type "Microsoft.ContainerRegistry/registries" \
  --api-version "2021-06-01-preview" \
  --set "properties.policies.exportPolicy.status=enabled"
```

Or using ARM template:
```json
{
  "properties": {
    "policies": {
      "exportPolicy": {
        "status": "enabled"
      }
    }
  }
}
```

## Configuration Summary Table

| Setting | Value for DLP | Command to Verify |
|---------|---------------|-------------------|
| Registry SKU | Premium | `az acr show --name myregistry --query sku.name` |
| Export Policy | disabled | `az acr show --name myregistry --query "properties.policies.exportPolicy.status"` |
| Public Network Access | disabled | `az acr show --name myregistry --query publicNetworkAccess` |
| Private Endpoint | Configured | `az acr private-endpoint-connection list --registry-name myregistry` |
| Data Endpoints | enabled (recommended) | `az acr show-endpoints --name myregistry` |
| Diagnostic Logging | enabled | `az monitor diagnostic-settings list --resource <registry-id>` |

## Troubleshooting Configuration Issues

### Error: Cannot disable export policy without private endpoint

**Cause**: Private endpoint is required before disabling exports.

**Solution**: Configure a private endpoint first (see Step 2).

### Error: Export pipelines exist

**Cause**: Cannot disable exports while export pipelines exist.

**Solution**: Delete all export pipelines first:
```bash
az acr export-pipeline list --registry myregistry --resource-group myRG
az acr export-pipeline delete --registry myregistry --resource-group myRG --name <pipeline-name>
```

### Error: Registry is not Premium SKU

**Cause**: Export policy requires Premium tier.

**Solution**: Upgrade the registry:
```bash
az acr update --name myregistry --sku Premium
```

## Source References

- Export policy documentation: `/Users/johnsonshi/repos/acr-skills/submodules/azure-management-docs/articles/container-registry/data-loss-prevention.md`
- Private link setup: `/Users/johnsonshi/repos/acr-skills/submodules/azure-management-docs/articles/container-registry/container-registry-private-link.md`
- Dedicated data endpoints: `/Users/johnsonshi/repos/acr-skills/submodules/azure-management-docs/articles/container-registry/container-registry-dedicated-data-endpoints.md`

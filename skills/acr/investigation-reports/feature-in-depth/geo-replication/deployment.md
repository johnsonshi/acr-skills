# ACR Geo-Replication Deployment Guide

## Overview

This guide covers the deployment and setup of geo-replicated Azure Container Registries using various methods: Azure Portal, Azure CLI, PowerShell, and Infrastructure as Code (ARM/Bicep).

## Prerequisites

### General Requirements
- Azure subscription with appropriate permissions
- Premium SKU container registry (required for geo-replication)
- Permissions: `Microsoft.ContainerRegistry/registries/write` and `Microsoft.ContainerRegistry/registries/replications/write`

### For CLI/PowerShell
- Azure CLI version 2.0.31 or later
- Azure PowerShell Az module 5.9.0 or later

## Deployment Methods

### Method 1: Azure Portal

#### Step 1: Create Premium Registry

1. Sign in to the [Azure portal](https://portal.azure.com)
2. Select **Create a resource** > **Containers** > **Container Registry**
3. In the **Basics** tab:
   - Select or create a **Resource group**
   - Enter a unique **Registry name**
   - Select a **Location** for the home region
   - Select **Premium** for **Pricing Plan** (required for geo-replication)
4. Select **Review + create**, then **Create**

#### Step 2: Configure Geo-Replication

1. Navigate to your container registry in the Azure portal
2. In the service menu, under **Services**, select **Geo-replications**
3. You see a map showing Azure regions:
   - **Blue hexagon**: Home region (where registry was created)
   - **Green hexagons**: Available regions for replication
   - **Gray hexagons**: Regions not yet available
4. Select a green hexagon for the desired replica region
5. In the **Create replication** pane:
   - Confirm the **Location**
   - Optionally enable **Use availability zones**
6. Select **Create**
7. Wait for status to show **Ready** (refresh the page if needed)

#### Step 3: Enable Dedicated Data Endpoints (Optional)

1. Navigate to your container registry
2. Select **Networking** > **Public access**
3. Check **Use dedicated data endpoint**
4. Select **Save**

### Method 2: Azure CLI

#### Step 1: Create Resource Group and Registry

```bash
# Create resource group
az group create \
  --name myResourceGroup \
  --location westus

# Create Premium registry
az acr create \
  --name myregistry \
  --resource-group myResourceGroup \
  --sku Premium \
  --location westus
```

#### Step 2: Add Geo-Replications

```bash
# Add replica in East US
az acr replication create \
  --registry myregistry \
  --location eastus

# Add replica in West Europe
az acr replication create \
  --registry myregistry \
  --location westeurope

# Add zone-redundant replica
az acr replication create \
  --registry myregistry \
  --location australiaeast \
  --zone-redundancy enabled
```

#### Step 3: Verify Configuration

```bash
# List all replications
az acr replication list --registry myregistry --output table

# View endpoints
az acr show-endpoints --name myregistry
```

### Method 3: Azure PowerShell

#### Step 1: Create Resource Group and Registry

```powershell
# Create resource group
New-AzResourceGroup -Name myResourceGroup -Location westus

# Create Premium registry
New-AzContainerRegistry `
  -ResourceGroupName myResourceGroup `
  -Name myregistry `
  -Sku Premium `
  -Location westus
```

#### Step 2: Add Geo-Replications

```powershell
# Add replica
New-AzContainerRegistryReplication `
  -ResourceGroupName myResourceGroup `
  -RegistryName myregistry `
  -Location eastus

# Add another replica
New-AzContainerRegistryReplication `
  -ResourceGroupName myResourceGroup `
  -RegistryName myregistry `
  -Location westeurope
```

#### Step 3: Verify Configuration

```powershell
# List replications
Get-AzContainerRegistryReplication `
  -ResourceGroupName myResourceGroup `
  -RegistryName myregistry
```

### Method 4: ARM Template

#### Deploy Using Azure CLI

```bash
# Create parameter file
cat > parameters.json << 'EOF'
{
  "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentParameters.json#",
  "contentVersion": "1.0.0.0",
  "parameters": {
    "acrName": {
      "value": "myuniqueregistry"
    },
    "location": {
      "value": "westus"
    },
    "acrReplicaLocation": {
      "value": "eastus"
    }
  }
}
EOF

# Deploy template
az deployment group create \
  --resource-group myResourceGroup \
  --template-uri "https://raw.githubusercontent.com/Azure/azure-quickstart-templates/master/quickstarts/microsoft.containerregistry/container-registry-geo-replication/azuredeploy.json" \
  --parameters @parameters.json
```

### Method 5: Bicep

#### Deploy Using Bicep File

```bicep
// main.bicep
@description('Container registry name')
param acrName string = 'acr${uniqueString(resourceGroup().id)}'

@description('Home region')
param location string = resourceGroup().location

@description('Replica region')
param replicaLocation string = 'eastus'

resource containerRegistry 'Microsoft.ContainerRegistry/registries@2023-01-01-preview' = {
  name: acrName
  location: location
  sku: {
    name: 'Premium'
  }
  properties: {
    adminUserEnabled: false
    zoneRedundancy: 'Enabled'
  }
}

resource replica 'Microsoft.ContainerRegistry/registries/replications@2023-01-01-preview' = {
  parent: containerRegistry
  name: replicaLocation
  location: replicaLocation
  properties: {
    zoneRedundancy: 'Enabled'
  }
}

output loginServer string = containerRegistry.properties.loginServer
```

```bash
# Deploy Bicep template
az deployment group create \
  --resource-group myResourceGroup \
  --template-file main.bicep \
  --parameters acrName=myregistry replicaLocation=eastus
```

## Post-Deployment Setup

### 1. Login to Registry

```bash
# Login using Azure CLI
az acr login --name myregistry
```

### 2. Push Test Image

```bash
# Pull a test image
docker pull mcr.microsoft.com/hello-world

# Tag for your registry
docker tag mcr.microsoft.com/hello-world myregistry.azurecr.io/hello-world:v1

# Push to registry (automatically replicated)
docker push myregistry.azurecr.io/hello-world:v1
```

### 3. Configure Webhooks (Optional)

```bash
# Create regional webhook
az acr webhook create \
  --registry myregistry \
  --name webhook-eastus \
  --location eastus \
  --uri https://myserver.com/webhook \
  --actions push
```

### 4. Enable Admin Account (For Web Apps)

```bash
# Enable admin user (if needed for Web App deployment)
az acr update --name myregistry --admin-enabled true

# Get credentials
az acr credential show --name myregistry
```

## Configuring Private Endpoints

### For Each Replica Region

When using private endpoints with geo-replication, configure endpoints in each region:

```bash
# Example: Create private endpoint in each region
# (Requires existing VNet and subnet in each region)

# East US region
az network private-endpoint create \
  --name myregistry-pe-eastus \
  --resource-group myResourceGroup \
  --vnet-name eastus-vnet \
  --subnet default \
  --private-connection-resource-id $(az acr show --name myregistry --query id -o tsv) \
  --group-ids registry \
  --connection-name eastus-connection
```

### DNS Configuration

For each replica, create DNS records:
- `myregistry.azurecr.io` -> Private IP
- `myregistry.eastus.data.azurecr.io` -> Regional data endpoint private IP
- `myregistry.westus.data.azurecr.io` -> Regional data endpoint private IP

## Upgrading Existing Registry

### From Standard to Premium

```bash
# Check current SKU
az acr show --name myregistry --query sku.name

# Upgrade to Premium
az acr update --name myregistry --sku Premium

# Verify upgrade
az acr show --name myregistry --query sku.name
```

**Note**: Upgrading SKU has no downtime or impact on registry operations.

### Add Replications After Upgrade

```bash
# Add replications after upgrading to Premium
az acr replication create --registry myregistry --location eastus
az acr replication create --registry myregistry --location westeurope
```

## Monitoring Deployment

### Check Replication Status

```bash
# List replications with status
az acr replication list \
  --registry myregistry \
  --query "[].{Name:name, Location:location, Status:provisioningState, Ready:status.displayStatus}" \
  --output table
```

**Expected Output:**
```
Name        Location    Status       Ready
----------  ----------  -----------  ------
westus      westus      Succeeded    Ready
eastus      eastus      Succeeded    Ready
westeurope  westeurope  Succeeded    Ready
```

### View Storage Usage

```bash
az acr show-usage \
  --resource-group myResourceGroup \
  --name myregistry \
  --output table
```

## Cleanup / Removal

### Remove Specific Replica

```bash
# Delete a specific replication
az acr replication delete \
  --name eastus \
  --registry myregistry \
  --yes
```

### Remove All Replications (Downgrade)

```bash
# Remove all replicas before downgrading SKU
az acr replication list --registry myregistry --query "[].name" -o tsv | \
  xargs -I {} az acr replication delete --name {} --registry myregistry --yes

# Downgrade to Standard (after removing replications)
az acr update --name myregistry --sku Standard
```

## Troubleshooting Deployment

### Common Issues

| Issue | Solution |
|-------|----------|
| "SKU does not support geo-replication" | Upgrade to Premium SKU first |
| "Region not available" | Check Azure region availability |
| "Provisioning stuck" | Check permissions for private endpoints |
| "Replication failed" | Verify network connectivity and permissions |

### Validate Permissions

```bash
# Check role assignments
az role assignment list \
  --scope $(az acr show --name myregistry --query id -o tsv) \
  --output table
```

### Debug Network Issues

```bash
# Check registry health
az acr check-health --name myregistry --yes

# Verify endpoints
az acr show-endpoints --name myregistry
```

## Sources

- `/Users/johnsonshi/repos/acr-skills/submodules/azure-management-docs/articles/container-registry/container-registry-geo-replication.md`
- `/Users/johnsonshi/repos/acr-skills/submodules/azure-management-docs/articles/container-registry/container-registry-get-started-geo-replication-template.md`
- `/Users/johnsonshi/repos/acr-skills/submodules/azure-management-docs/articles/container-registry/container-registry-tutorial-prepare-registry.md`
- `/Users/johnsonshi/repos/acr-skills/submodules/azure-management-docs/articles/container-registry/container-registry-tutorial-deploy-app.md`
- `/Users/johnsonshi/repos/acr-skills/submodules/azure-management-docs/articles/container-registry/container-registry-tutorial-deploy-update.md`
- `/Users/johnsonshi/repos/acr-skills/submodules/azure-management-docs/articles/container-registry/zone-redundancy.md`

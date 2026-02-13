# ACR Zone Redundancy - Deployment Guide

## Overview

This guide covers how to deploy zone-redundant Azure Container Registries and configure zone-redundant geo-replicas. Note that as of 2026, zone redundancy is enabled by default for all registries in supported regions, but explicit configuration is still available for Premium SKU registries.

## Prerequisites

Before deploying a zone-redundant registry:

1. **Azure Subscription**: Active Azure subscription
2. **Region Selection**: Choose a region that [supports availability zones](/azure/reliability/regions-list)
3. **SKU Selection**:
   - All SKUs benefit from automatic zone redundancy
   - Premium SKU required for explicit configuration and geo-replication
4. **Tools**: Azure CLI, Azure portal access, or infrastructure-as-code tools

## Deployment Methods

### Method 1: Azure Portal

#### Create Zone-Redundant Registry

1. Sign in to the [Azure portal](https://portal.azure.com)
2. Select **Create a resource** > **Containers** > **Container Registry**
3. In the **Basics** tab:
   - Select or create a resource group
   - Enter a unique registry name
   - Select a **Location** that supports availability zones (e.g., *East US*)
   - Select **Premium** SKU
4. In **Use availability zones**, ensure **Enabled** is selected
5. Configure additional settings as needed
6. Select **Review + create**
7. Select **Create** to deploy

#### Add Zone-Redundant Replica (Geo-Replication)

1. Navigate to your Premium registry in the Azure portal
2. In the service menu, under **Services**, select **Geo-replications**
3. On the map:
   - Select a green hexagon in a region that supports availability zones
   - Or select **+ Add**
4. In the **Create replication** window:
   - Confirm the **Location**
   - In **Use availability zones**, select **Enabled**
5. Select **Create**

### Method 2: Azure CLI

#### Create Resource Group

```azurecli
# Create resource group in a supported region
az group create --name <resource-group-name> --location <location>

# Example
az group create --name myACRResourceGroup --location eastus
```

#### Create Zone-Redundant Registry

```azurecli
# Create zone-redundant Premium registry
az acr create \
  --resource-group <resource-group-name> \
  --name <container-registry-name> \
  --location <region-name> \
  --zone-redundancy enabled \
  --sku Premium

# Example
az acr create \
  --resource-group myACRResourceGroup \
  --name myZoneRedundantRegistry \
  --location eastus \
  --zone-redundancy enabled \
  --sku Premium
```

#### Verify Zone Redundancy Status

Check the `zoneRedundancy` property in the output:

```json
{
  "zoneRedundancy": "Enabled"
}
```

#### Create Zone-Redundant Geo-Replica

```azurecli
# Add zone-redundant replication to another region
az acr replication create \
  --location <region-name> \
  --resource-group <resource-group-name> \
  --registry <container-registry-name> \
  --zone-redundancy enabled

# Example
az acr replication create \
  --location westus2 \
  --resource-group myACRResourceGroup \
  --registry myZoneRedundantRegistry \
  --zone-redundancy enabled
```

### Method 3: Bicep/ARM Template

#### Zone-Redundant Registry Template

Save as `registryZone.bicep`:

```bicep
@description('Globally unique name of your Azure Container Registry')
@minLength(5)
@maxLength(50)
param containerRegistryName string = 'acr${uniqueString(resourceGroup().id)}'

@description('Location for registry home replica.')
param location string = resourceGroup().location

@description('Enable admin user for registry. This is not recommended for production use.')
param adminUserEnabled bool = false

@description('Enable zone redundancy of registry\'s home replica. Requires the registry\'s region supports availability zones.')
@allowed([
  'Enabled'
  'Disabled'
])
param containerRegistryZoneRedundancy string = 'Enabled'

// Tier of your Azure Container Registry. Geo-replication and zone redundancy require Premium SKU.
var acrSku = 'Premium'

resource containerRegistry 'Microsoft.ContainerRegistry/registries@2025-04-01' = {
  name: containerRegistryName
  location: location
  sku: {
    name: acrSku
  }
  properties: {
    adminUserEnabled: adminUserEnabled
    zoneRedundancy: containerRegistryZoneRedundancy
  }
}

output containerRegistryLoginServer string = containerRegistry.properties.loginServer
```

#### Deploy Registry Template

```azurecli
az deployment group create \
  --resource-group <resource-group-name> \
  --template-file registryZone.bicep \
  --parameters containerRegistryName=<registry-name>
```

#### Zone-Redundant Replica Template

Save as `replicaZone.bicep`:

```bicep
@description('Globally unique name of your Azure Container Registry')
param containerRegistryName string

@description('Short name for registry replica location, such as australiaeast or westus.')
param replicaLocation string

@description('Enable zone redundancy of registry replica. Requires replica location to support availability zones.')
@allowed([
  'Enabled'
  'Disabled'
])
param replicaZoneRedundancy string = 'Enabled'

resource containerRegistry 'Microsoft.ContainerRegistry/registries@2025-04-01' existing = {
  name: containerRegistryName
}

resource containerRegistryReplica 'Microsoft.ContainerRegistry/registries/replications@2025-04-01' = {
  parent: containerRegistry
  name: replicaLocation
  location: replicaLocation
  properties: {
    zoneRedundancy: replicaZoneRedundancy
  }
}
```

#### Deploy Replica Template

```azurecli
az deployment group create \
  --resource-group <resource-group-name> \
  --template-file replicaZone.bicep \
  --parameters containerRegistryName=<registry-name> replicaLocation=<replica-location>

# Example
az deployment group create \
  --resource-group myACRResourceGroup \
  --template-file replicaZone.bicep \
  --parameters containerRegistryName=myZoneRedundantRegistry replicaLocation=westeurope
```

### Method 4: Terraform

```hcl
resource "azurerm_container_registry" "acr" {
  name                = "myZoneRedundantRegistry"
  resource_group_name = azurerm_resource_group.rg.name
  location            = azurerm_resource_group.rg.location
  sku                 = "Premium"
  zone_redundancy_enabled = true

  georeplications {
    location                = "West US 2"
    zone_redundancy_enabled = true
  }

  georeplications {
    location                = "West Europe"
    zone_redundancy_enabled = true
  }
}
```

## Post-Deployment Verification

### Verify Registry Zone Redundancy

```azurecli
# Show registry details
az acr show --name <registry-name> --resource-group <resource-group-name> --query zoneRedundancy

# Show full registry properties
az acr show --name <registry-name> --output json
```

### Verify Replica Zone Redundancy

```azurecli
# List all replications
az acr replication list --registry <registry-name> --output table

# Show specific replication details
az acr replication show --name <location> --registry <registry-name> --query zoneRedundancy
```

## Deployment Best Practices

### 1. Region Selection

- Choose regions with availability zone support
- Consider proximity to deployment targets
- Plan for multi-region deployments for global applications

### 2. SKU Considerations

- Use Premium SKU for geo-replication with zone redundancy
- All SKUs benefit from automatic zone redundancy in supported regions
- Premium provides additional features like private endpoints

### 3. Network Configuration

When deploying with private endpoints:
- Dedicated data endpoints are enabled automatically per region
- Configure private endpoints in each zone-redundant region
- Update firewall rules for dedicated data endpoints

### 4. Naming Conventions

- Use consistent naming for registries and replicas
- Include region identifiers where helpful
- Document deployment configurations

## Troubleshooting Deployment

### Common Issues

1. **Zone redundancy shows as disabled**
   - Portal may not reflect actual status
   - Zone redundancy is automatically enabled in supported regions
   - Use CLI to verify actual configuration

2. **Deployment fails in region**
   - Verify region supports availability zones
   - Check Azure subscription quotas
   - Ensure Premium SKU for explicit zone redundancy

3. **Replication stuck in provisioning**
   - For private endpoint registries, verify identity permissions
   - Add `Microsoft.Network/privateEndpoints/privateLinkServiceProxies/write` permission
   - Delete stuck replication and recreate

## Sources

- `/Users/johnsonshi/repos/acr-skills/submodules/azure-management-docs/articles/container-registry/zone-redundancy.md`
- `/Users/johnsonshi/repos/acr-skills/submodules/azure-management-docs/articles/container-registry/container-registry-geo-replication.md`
- `/Users/johnsonshi/repos/acr-skills/submodules/azure-management-docs/articles/container-registry/container-registry-get-started-geo-replication-template.md`
- `/Users/johnsonshi/repos/acr-skills/submodules/azure-management-docs/articles/container-registry/container-registry-get-started-bicep.md`

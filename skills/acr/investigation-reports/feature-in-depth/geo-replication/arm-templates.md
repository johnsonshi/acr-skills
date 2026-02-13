# ACR Geo-Replication ARM Templates and Bicep

## Overview

This document provides comprehensive ARM templates and Bicep files for deploying geo-replicated Azure Container Registries.

## Important Notes

- Geo-replicated registries **do not support** ARM/Bicep template Complete mode deployments
- Use Incremental deployment mode for registries with replications
- Premium SKU is required for geo-replication

## ARM Templates

### Basic Geo-Replicated Registry

**File: azuredeploy.json**
```json
{
  "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
  "contentVersion": "1.0.0.0",
  "parameters": {
    "acrName": {
      "type": "string",
      "defaultValue": "[concat('acr', uniqueString(resourceGroup().id))]",
      "minLength": 5,
      "maxLength": 50,
      "metadata": {
        "description": "Globally unique name for the container registry"
      }
    },
    "location": {
      "type": "string",
      "defaultValue": "[resourceGroup().location]",
      "metadata": {
        "description": "Location for the registry home replica"
      }
    },
    "acrAdminUserEnabled": {
      "type": "bool",
      "defaultValue": false,
      "metadata": {
        "description": "Enable admin user for the registry"
      }
    },
    "acrSku": {
      "type": "string",
      "defaultValue": "Premium",
      "allowedValues": [
        "Premium"
      ],
      "metadata": {
        "description": "SKU for the registry (must be Premium for geo-replication)"
      }
    },
    "acrReplicaLocation": {
      "type": "string",
      "metadata": {
        "description": "Location for the registry replica (must differ from home location)"
      }
    }
  },
  "resources": [
    {
      "type": "Microsoft.ContainerRegistry/registries",
      "apiVersion": "2023-01-01-preview",
      "name": "[parameters('acrName')]",
      "location": "[parameters('location')]",
      "sku": {
        "name": "[parameters('acrSku')]"
      },
      "properties": {
        "adminUserEnabled": "[parameters('acrAdminUserEnabled')]"
      }
    },
    {
      "type": "Microsoft.ContainerRegistry/registries/replications",
      "apiVersion": "2023-01-01-preview",
      "name": "[concat(parameters('acrName'), '/', parameters('acrReplicaLocation'))]",
      "location": "[parameters('acrReplicaLocation')]",
      "dependsOn": [
        "[resourceId('Microsoft.ContainerRegistry/registries', parameters('acrName'))]"
      ],
      "properties": {}
    }
  ],
  "outputs": {
    "acrLoginServer": {
      "type": "string",
      "value": "[reference(resourceId('Microsoft.ContainerRegistry/registries', parameters('acrName'))).loginServer]"
    },
    "acrResourceId": {
      "type": "string",
      "value": "[resourceId('Microsoft.ContainerRegistry/registries', parameters('acrName'))]"
    }
  }
}
```

### Multi-Region Geo-Replicated Registry

**File: azuredeploy-multi-region.json**
```json
{
  "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
  "contentVersion": "1.0.0.0",
  "parameters": {
    "acrName": {
      "type": "string",
      "defaultValue": "[concat('acr', uniqueString(resourceGroup().id))]",
      "minLength": 5,
      "maxLength": 50,
      "metadata": {
        "description": "Globally unique name for the container registry"
      }
    },
    "location": {
      "type": "string",
      "defaultValue": "[resourceGroup().location]",
      "metadata": {
        "description": "Location for the registry home replica"
      }
    },
    "replicaLocations": {
      "type": "array",
      "defaultValue": [
        "eastus",
        "westeurope",
        "southeastasia"
      ],
      "metadata": {
        "description": "Array of regions for geo-replication"
      }
    },
    "adminUserEnabled": {
      "type": "bool",
      "defaultValue": false
    }
  },
  "resources": [
    {
      "type": "Microsoft.ContainerRegistry/registries",
      "apiVersion": "2023-01-01-preview",
      "name": "[parameters('acrName')]",
      "location": "[parameters('location')]",
      "sku": {
        "name": "Premium"
      },
      "properties": {
        "adminUserEnabled": "[parameters('adminUserEnabled')]"
      }
    },
    {
      "type": "Microsoft.ContainerRegistry/registries/replications",
      "apiVersion": "2023-01-01-preview",
      "name": "[concat(parameters('acrName'), '/', parameters('replicaLocations')[copyIndex()])]",
      "location": "[parameters('replicaLocations')[copyIndex()]]",
      "dependsOn": [
        "[resourceId('Microsoft.ContainerRegistry/registries', parameters('acrName'))]"
      ],
      "copy": {
        "name": "replicationCopy",
        "count": "[length(parameters('replicaLocations'))]"
      },
      "properties": {}
    }
  ],
  "outputs": {
    "loginServer": {
      "type": "string",
      "value": "[reference(resourceId('Microsoft.ContainerRegistry/registries', parameters('acrName'))).loginServer]"
    },
    "replicaCount": {
      "type": "int",
      "value": "[add(length(parameters('replicaLocations')), 1)]"
    }
  }
}
```

### Zone-Redundant Geo-Replicated Registry

**File: azuredeploy-zone-redundant.json**
```json
{
  "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
  "contentVersion": "1.0.0.0",
  "parameters": {
    "acrName": {
      "type": "string",
      "defaultValue": "[concat('acr', uniqueString(resourceGroup().id))]",
      "metadata": {
        "description": "Globally unique name for the container registry"
      }
    },
    "location": {
      "type": "string",
      "defaultValue": "eastus",
      "metadata": {
        "description": "Location for the registry home replica (must support availability zones)"
      }
    },
    "homeZoneRedundancy": {
      "type": "string",
      "defaultValue": "Enabled",
      "allowedValues": [
        "Enabled",
        "Disabled"
      ],
      "metadata": {
        "description": "Enable zone redundancy for home replica"
      }
    },
    "replicaLocation": {
      "type": "string",
      "defaultValue": "westeurope",
      "metadata": {
        "description": "Location for the registry replica"
      }
    },
    "replicaZoneRedundancy": {
      "type": "string",
      "defaultValue": "Enabled",
      "allowedValues": [
        "Enabled",
        "Disabled"
      ],
      "metadata": {
        "description": "Enable zone redundancy for replica"
      }
    }
  },
  "resources": [
    {
      "type": "Microsoft.ContainerRegistry/registries",
      "apiVersion": "2023-01-01-preview",
      "name": "[parameters('acrName')]",
      "location": "[parameters('location')]",
      "sku": {
        "name": "Premium"
      },
      "properties": {
        "adminUserEnabled": false,
        "zoneRedundancy": "[parameters('homeZoneRedundancy')]"
      }
    },
    {
      "type": "Microsoft.ContainerRegistry/registries/replications",
      "apiVersion": "2023-01-01-preview",
      "name": "[concat(parameters('acrName'), '/', parameters('replicaLocation'))]",
      "location": "[parameters('replicaLocation')]",
      "dependsOn": [
        "[resourceId('Microsoft.ContainerRegistry/registries', parameters('acrName'))]"
      ],
      "properties": {
        "zoneRedundancy": "[parameters('replicaZoneRedundancy')]"
      }
    }
  ],
  "outputs": {
    "loginServer": {
      "type": "string",
      "value": "[reference(resourceId('Microsoft.ContainerRegistry/registries', parameters('acrName'))).loginServer]"
    }
  }
}
```

### Geo-Replicated Registry with Private Endpoint

**File: azuredeploy-private-endpoint.json**
```json
{
  "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
  "contentVersion": "1.0.0.0",
  "parameters": {
    "acrName": {
      "type": "string",
      "metadata": {
        "description": "Globally unique name for the container registry"
      }
    },
    "location": {
      "type": "string",
      "defaultValue": "[resourceGroup().location]"
    },
    "replicaLocation": {
      "type": "string",
      "metadata": {
        "description": "Location for the registry replica"
      }
    },
    "vnetName": {
      "type": "string",
      "metadata": {
        "description": "Name of the virtual network"
      }
    },
    "subnetName": {
      "type": "string",
      "defaultValue": "default",
      "metadata": {
        "description": "Name of the subnet for private endpoint"
      }
    },
    "privateDnsZoneName": {
      "type": "string",
      "defaultValue": "privatelink.azurecr.io"
    }
  },
  "variables": {
    "privateEndpointName": "[concat(parameters('acrName'), '-pe')]",
    "privateDnsZoneGroupName": "[concat(variables('privateEndpointName'), '/default')]"
  },
  "resources": [
    {
      "type": "Microsoft.ContainerRegistry/registries",
      "apiVersion": "2023-01-01-preview",
      "name": "[parameters('acrName')]",
      "location": "[parameters('location')]",
      "sku": {
        "name": "Premium"
      },
      "properties": {
        "adminUserEnabled": false,
        "publicNetworkAccess": "Disabled"
      }
    },
    {
      "type": "Microsoft.ContainerRegistry/registries/replications",
      "apiVersion": "2023-01-01-preview",
      "name": "[concat(parameters('acrName'), '/', parameters('replicaLocation'))]",
      "location": "[parameters('replicaLocation')]",
      "dependsOn": [
        "[resourceId('Microsoft.ContainerRegistry/registries', parameters('acrName'))]"
      ],
      "properties": {}
    },
    {
      "type": "Microsoft.Network/privateEndpoints",
      "apiVersion": "2021-05-01",
      "name": "[variables('privateEndpointName')]",
      "location": "[parameters('location')]",
      "dependsOn": [
        "[resourceId('Microsoft.ContainerRegistry/registries', parameters('acrName'))]"
      ],
      "properties": {
        "subnet": {
          "id": "[resourceId('Microsoft.Network/virtualNetworks/subnets', parameters('vnetName'), parameters('subnetName'))]"
        },
        "privateLinkServiceConnections": [
          {
            "name": "[variables('privateEndpointName')]",
            "properties": {
              "privateLinkServiceId": "[resourceId('Microsoft.ContainerRegistry/registries', parameters('acrName'))]",
              "groupIds": [
                "registry"
              ]
            }
          }
        ]
      }
    },
    {
      "type": "Microsoft.Network/privateDnsZones",
      "apiVersion": "2020-06-01",
      "name": "[parameters('privateDnsZoneName')]",
      "location": "global",
      "properties": {}
    },
    {
      "type": "Microsoft.Network/privateDnsZones/virtualNetworkLinks",
      "apiVersion": "2020-06-01",
      "name": "[concat(parameters('privateDnsZoneName'), '/', parameters('vnetName'), '-link')]",
      "location": "global",
      "dependsOn": [
        "[resourceId('Microsoft.Network/privateDnsZones', parameters('privateDnsZoneName'))]"
      ],
      "properties": {
        "registrationEnabled": false,
        "virtualNetwork": {
          "id": "[resourceId('Microsoft.Network/virtualNetworks', parameters('vnetName'))]"
        }
      }
    },
    {
      "type": "Microsoft.Network/privateEndpoints/privateDnsZoneGroups",
      "apiVersion": "2021-05-01",
      "name": "[variables('privateDnsZoneGroupName')]",
      "dependsOn": [
        "[resourceId('Microsoft.Network/privateEndpoints', variables('privateEndpointName'))]",
        "[resourceId('Microsoft.Network/privateDnsZones', parameters('privateDnsZoneName'))]"
      ],
      "properties": {
        "privateDnsZoneConfigs": [
          {
            "name": "config1",
            "properties": {
              "privateDnsZoneId": "[resourceId('Microsoft.Network/privateDnsZones', parameters('privateDnsZoneName'))]"
            }
          }
        ]
      }
    }
  ],
  "outputs": {
    "loginServer": {
      "type": "string",
      "value": "[reference(resourceId('Microsoft.ContainerRegistry/registries', parameters('acrName'))).loginServer]"
    }
  }
}
```

## Bicep Templates

### Basic Geo-Replicated Registry

**File: main.bicep**
```bicep
@description('Globally unique name of your Azure Container Registry')
@minLength(5)
@maxLength(50)
param acrName string = 'acr${uniqueString(resourceGroup().id)}'

@description('Location for registry home replica')
param location string = resourceGroup().location

@description('Location for registry replica')
param replicaLocation string

@description('Enable admin user')
param adminUserEnabled bool = false

resource containerRegistry 'Microsoft.ContainerRegistry/registries@2023-01-01-preview' = {
  name: acrName
  location: location
  sku: {
    name: 'Premium'
  }
  properties: {
    adminUserEnabled: adminUserEnabled
  }
}

resource containerRegistryReplica 'Microsoft.ContainerRegistry/registries/replications@2023-01-01-preview' = {
  parent: containerRegistry
  name: replicaLocation
  location: replicaLocation
  properties: {}
}

output loginServer string = containerRegistry.properties.loginServer
output registryResourceId string = containerRegistry.id
```

### Multi-Region Geo-Replicated Registry

**File: multi-region.bicep**
```bicep
@description('Globally unique name of your Azure Container Registry')
@minLength(5)
@maxLength(50)
param acrName string = 'acr${uniqueString(resourceGroup().id)}'

@description('Location for registry home replica')
param location string = resourceGroup().location

@description('Array of locations for replicas')
param replicaLocations array = [
  'eastus'
  'westeurope'
  'southeastasia'
]

@description('Enable admin user')
param adminUserEnabled bool = false

resource containerRegistry 'Microsoft.ContainerRegistry/registries@2023-01-01-preview' = {
  name: acrName
  location: location
  sku: {
    name: 'Premium'
  }
  properties: {
    adminUserEnabled: adminUserEnabled
  }
}

resource replications 'Microsoft.ContainerRegistry/registries/replications@2023-01-01-preview' = [for replicaLocation in replicaLocations: {
  parent: containerRegistry
  name: replicaLocation
  location: replicaLocation
  properties: {}
}]

output loginServer string = containerRegistry.properties.loginServer
output totalReplicas int = length(replicaLocations) + 1
```

### Zone-Redundant Geo-Replicated Registry

**File: zone-redundant.bicep**
```bicep
@description('Globally unique name of your Azure Container Registry')
@minLength(5)
@maxLength(50)
param acrName string = 'acr${uniqueString(resourceGroup().id)}'

@description('Location for registry home replica (must support availability zones)')
param location string = resourceGroup().location

@description('Enable admin user')
param adminUserEnabled bool = false

@description('Enable zone redundancy for home replica')
@allowed([
  'Enabled'
  'Disabled'
])
param homeZoneRedundancy string = 'Enabled'

@description('Location for registry replica')
param replicaLocation string

@description('Enable zone redundancy for replica')
@allowed([
  'Enabled'
  'Disabled'
])
param replicaZoneRedundancy string = 'Enabled'

resource containerRegistry 'Microsoft.ContainerRegistry/registries@2023-01-01-preview' = {
  name: acrName
  location: location
  sku: {
    name: 'Premium'
  }
  properties: {
    adminUserEnabled: adminUserEnabled
    zoneRedundancy: homeZoneRedundancy
  }
}

resource containerRegistryReplica 'Microsoft.ContainerRegistry/registries/replications@2023-01-01-preview' = {
  parent: containerRegistry
  name: replicaLocation
  location: replicaLocation
  properties: {
    zoneRedundancy: replicaZoneRedundancy
  }
}

output loginServer string = containerRegistry.properties.loginServer
```

### Adding Replica to Existing Registry

**File: add-replica.bicep**
```bicep
@description('Name of existing Azure Container Registry')
param containerRegistryName string

@description('Short name for registry replica location')
param replicaLocation string

@description('Enable zone redundancy of registry replica')
@allowed([
  'Enabled'
  'Disabled'
])
param replicaZoneRedundancy string = 'Enabled'

resource containerRegistry 'Microsoft.ContainerRegistry/registries@2023-01-01-preview' existing = {
  name: containerRegistryName
}

resource containerRegistryReplica 'Microsoft.ContainerRegistry/registries/replications@2023-01-01-preview' = {
  parent: containerRegistry
  name: replicaLocation
  location: replicaLocation
  properties: {
    zoneRedundancy: replicaZoneRedundancy
  }
}

output replicaId string = containerRegistryReplica.id
```

## Deployment Commands

### Deploy ARM Template

```bash
# Deploy basic geo-replicated registry
az deployment group create \
  --resource-group myResourceGroup \
  --template-file azuredeploy.json \
  --parameters acrName=myregistry acrReplicaLocation=eastus

# Deploy multi-region registry
az deployment group create \
  --resource-group myResourceGroup \
  --template-file azuredeploy-multi-region.json \
  --parameters acrName=myregistry

# Deploy zone-redundant registry
az deployment group create \
  --resource-group myResourceGroup \
  --template-file azuredeploy-zone-redundant.json \
  --parameters acrName=myregistry location=eastus replicaLocation=westeurope
```

### Deploy Bicep Template

```bash
# Deploy basic geo-replicated registry
az deployment group create \
  --resource-group myResourceGroup \
  --template-file main.bicep \
  --parameters acrName=myregistry replicaLocation=eastus

# Deploy multi-region registry
az deployment group create \
  --resource-group myResourceGroup \
  --template-file multi-region.bicep \
  --parameters acrName=myregistry

# Add replica to existing registry
az deployment group create \
  --resource-group myResourceGroup \
  --template-file add-replica.bicep \
  --parameters containerRegistryName=myregistry replicaLocation=westeurope
```

### Deploy with What-If Preview

```bash
# Preview deployment changes
az deployment group what-if \
  --resource-group myResourceGroup \
  --template-file main.bicep \
  --parameters acrName=myregistry replicaLocation=eastus
```

## Parameter Files

### parameters.json
```json
{
  "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentParameters.json#",
  "contentVersion": "1.0.0.0",
  "parameters": {
    "acrName": {
      "value": "myuniqueacrname"
    },
    "location": {
      "value": "westus"
    },
    "replicaLocation": {
      "value": "eastus"
    },
    "adminUserEnabled": {
      "value": false
    }
  }
}
```

### Deploy with Parameter File
```bash
az deployment group create \
  --resource-group myResourceGroup \
  --template-file azuredeploy.json \
  --parameters @parameters.json
```

## Resource Types Reference

| Resource Type | API Version | Description |
|---------------|-------------|-------------|
| `Microsoft.ContainerRegistry/registries` | `2023-01-01-preview` | Container registry |
| `Microsoft.ContainerRegistry/registries/replications` | `2023-01-01-preview` | Registry replication |

## Common Properties

### Registry Properties

| Property | Type | Description |
|----------|------|-------------|
| `adminUserEnabled` | bool | Enable admin user |
| `zoneRedundancy` | string | Enable zone redundancy (`Enabled`/`Disabled`) |
| `publicNetworkAccess` | string | Public network access (`Enabled`/`Disabled`) |

### Replication Properties

| Property | Type | Description |
|----------|------|-------------|
| `zoneRedundancy` | string | Enable zone redundancy (`Enabled`/`Disabled`) |
| `regionEndpointEnabled` | bool | Enable/disable routing to this replica |

## Sources

- `/Users/johnsonshi/repos/acr-skills/submodules/azure-management-docs/articles/container-registry/container-registry-get-started-geo-replication-template.md`
- `/Users/johnsonshi/repos/acr-skills/submodules/azure-management-docs/articles/container-registry/zone-redundancy.md`
- Azure Quickstart Templates: https://github.com/Azure/azure-quickstart-templates/tree/master/quickstarts/microsoft.containerregistry/container-registry-geo-replication

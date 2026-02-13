# Azure Container Registry Private Endpoints - Deployment Guide

## Prerequisites

Before deploying private endpoints for ACR, ensure you have:

1. **Premium SKU Registry**: Private endpoints require the Premium service tier
2. **Virtual Network and Subnet**: Network infrastructure for the private endpoint
3. **Azure CLI 2.6.0+**: Required for CLI-based deployment
4. **Permissions**:
   - `Microsoft.ContainerRegistry/registries/write`
   - `Microsoft.Network/privateEndpoints/write`
   - `Microsoft.Network/privateEndpoints/privateLinkServiceProxies/write`

## Method 1: Azure Portal (Recommended)

### Create Private Endpoint with New Registry

1. Navigate to **Create a resource** > **Containers** > **Container Registry**

2. On the **Basics** tab:
   - Select **Premium** for **Pricing plan** (required for private endpoints)
   - Complete other basic settings

3. On the **Networking** tab:
   - For **Connectivity configuration**, select **Private access (recommended)**
   - Click **Create a private endpoint connection**

4. Configure the private endpoint:

   | Setting | Value |
   |---------|-------|
   | Subscription | Select your subscription |
   | Resource group | Select existing or create new |
   | Name | Enter unique endpoint name (e.g., `myregistry-pe`) |
   | Registry subresource | Ensure `registry` is selected |
   | Virtual network | Select target VNet |
   | Subnet | Select target subnet |
   | Integrate with private DNS zone | **Yes** (recommended) |
   | Private DNS Zone | Select `(New) privatelink.azurecr.io` |

5. Click **OK** to add the private endpoint

6. Complete remaining tabs and select **Review + create**

### Add Private Endpoint to Existing Registry

1. Navigate to your container registry in the Azure portal

2. In the service menu, under **Settings**, select **Networking**

3. On the **Private access** tab, click **Create a private endpoint connection**

4. Complete the wizard:
   - **Basics**: Select subscription, resource group, region, name
   - **Resource**: Pre-selected (registry resource and subresource)
   - **Virtual Network**: Select VNet and subnet
   - **DNS**: Ensure **Integrate with private DNS zone** is **Yes**
   - **Tags**: Add optional tags

5. Click **Create** after validation passes

### Verify Configuration

1. Go to registry > **Settings** > **Networking** > **Private access**
2. Select your private endpoint
3. Under **Settings**, select **DNS configuration**
4. Verify the DNS settings show correct FQDNs and private IPs

## Method 2: Azure CLI

### Step 1: Set Environment Variables

```bash
# Registry configuration
REGISTRY_NAME=mycontainerregistry
REGISTRY_LOCATION=westus2
RESOURCE_GROUP=myResourceGroup

# Network configuration
VNET_NAME=myVNet
SUBNET_NAME=mySubnet

# Private endpoint configuration
PRIVATE_ENDPOINT_NAME=myPrivateEndpoint
CONNECTION_NAME=myConnection
DNS_ZONE_NAME=privatelink.azurecr.io
DNS_LINK_NAME=myDNSLink
```

### Step 2: Create Premium Registry (if not exists)

```bash
az acr create \
  --name $REGISTRY_NAME \
  --resource-group $RESOURCE_GROUP \
  --location $REGISTRY_LOCATION \
  --sku Premium
```

### Step 3: Disable Network Policies in Subnet

Private endpoints require network policies to be disabled:

```bash
az network vnet subnet update \
  --name $SUBNET_NAME \
  --vnet-name $VNET_NAME \
  --resource-group $RESOURCE_GROUP \
  --disable-private-endpoint-network-policies
```

### Step 4: Create Private DNS Zone

```bash
az network private-dns zone create \
  --resource-group $RESOURCE_GROUP \
  --name $DNS_ZONE_NAME
```

### Step 5: Link DNS Zone to Virtual Network

```bash
az network private-dns link vnet create \
  --resource-group $RESOURCE_GROUP \
  --zone-name $DNS_ZONE_NAME \
  --name $DNS_LINK_NAME \
  --virtual-network $VNET_NAME \
  --registration-enabled false
```

### Step 6: Get Registry Resource ID

```bash
REGISTRY_ID=$(az acr show \
  --name $REGISTRY_NAME \
  --query 'id' \
  --output tsv)

echo "Registry ID: $REGISTRY_ID"
```

### Step 7: Create Private Endpoint

```bash
az network private-endpoint create \
  --name $PRIVATE_ENDPOINT_NAME \
  --resource-group $RESOURCE_GROUP \
  --vnet-name $VNET_NAME \
  --subnet $SUBNET_NAME \
  --private-connection-resource-id $REGISTRY_ID \
  --group-ids registry \
  --connection-name $CONNECTION_NAME
```

### Step 8: Get Private IP Addresses

```bash
# Get network interface ID
NETWORK_INTERFACE_ID=$(az network private-endpoint show \
  --name $PRIVATE_ENDPOINT_NAME \
  --resource-group $RESOURCE_GROUP \
  --query 'networkInterfaces[0].id' \
  --output tsv)

# Get registry private IP
REGISTRY_PRIVATE_IP=$(az network nic show \
  --ids $NETWORK_INTERFACE_ID \
  --query "ipConfigurations[?privateLinkConnectionProperties.requiredMemberName=='registry'].privateIPAddress" \
  --output tsv)

# Get data endpoint private IP
DATA_ENDPOINT_PRIVATE_IP=$(az network nic show \
  --ids $NETWORK_INTERFACE_ID \
  --query "ipConfigurations[?privateLinkConnectionProperties.requiredMemberName=='registry_data_$REGISTRY_LOCATION'].privateIPAddress" \
  --output tsv)

echo "Registry IP: $REGISTRY_PRIVATE_IP"
echo "Data Endpoint IP: $DATA_ENDPOINT_PRIVATE_IP"
```

### Step 9: Create DNS Records

```bash
# Create A-record sets
az network private-dns record-set a create \
  --name $REGISTRY_NAME \
  --zone-name $DNS_ZONE_NAME \
  --resource-group $RESOURCE_GROUP

az network private-dns record-set a create \
  --name ${REGISTRY_NAME}.${REGISTRY_LOCATION}.data \
  --zone-name $DNS_ZONE_NAME \
  --resource-group $RESOURCE_GROUP

# Add A-records
az network private-dns record-set a add-record \
  --record-set-name $REGISTRY_NAME \
  --zone-name $DNS_ZONE_NAME \
  --resource-group $RESOURCE_GROUP \
  --ipv4-address $REGISTRY_PRIVATE_IP

az network private-dns record-set a add-record \
  --record-set-name ${REGISTRY_NAME}.${REGISTRY_LOCATION}.data \
  --zone-name $DNS_ZONE_NAME \
  --resource-group $RESOURCE_GROUP \
  --ipv4-address $DATA_ENDPOINT_PRIVATE_IP
```

## Method 3: ARM Template

### Complete ARM Template

```json
{
  "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
  "contentVersion": "1.0.0.0",
  "parameters": {
    "registryName": {
      "type": "string",
      "metadata": {
        "description": "Name of the Azure Container Registry"
      }
    },
    "location": {
      "type": "string",
      "defaultValue": "[resourceGroup().location]",
      "metadata": {
        "description": "Location for all resources"
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
      "metadata": {
        "description": "Name of the subnet for private endpoint"
      }
    },
    "privateEndpointName": {
      "type": "string",
      "defaultValue": "[concat(parameters('registryName'), '-pe')]",
      "metadata": {
        "description": "Name of the private endpoint"
      }
    }
  },
  "variables": {
    "privateDnsZoneName": "privatelink.azurecr.io",
    "privateEndpointDnsGroupName": "[concat(parameters('privateEndpointName'), '/default')]"
  },
  "resources": [
    {
      "type": "Microsoft.ContainerRegistry/registries",
      "apiVersion": "2023-01-01-preview",
      "name": "[parameters('registryName')]",
      "location": "[parameters('location')]",
      "sku": {
        "name": "Premium"
      },
      "properties": {
        "publicNetworkAccess": "Disabled",
        "networkRuleBypassOptions": "AzureServices"
      }
    },
    {
      "type": "Microsoft.Network/privateEndpoints",
      "apiVersion": "2023-04-01",
      "name": "[parameters('privateEndpointName')]",
      "location": "[parameters('location')]",
      "dependsOn": [
        "[resourceId('Microsoft.ContainerRegistry/registries', parameters('registryName'))]"
      ],
      "properties": {
        "subnet": {
          "id": "[resourceId('Microsoft.Network/virtualNetworks/subnets', parameters('vnetName'), parameters('subnetName'))]"
        },
        "privateLinkServiceConnections": [
          {
            "name": "[concat(parameters('privateEndpointName'), '-connection')]",
            "properties": {
              "privateLinkServiceId": "[resourceId('Microsoft.ContainerRegistry/registries', parameters('registryName'))]",
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
      "name": "[variables('privateDnsZoneName')]",
      "location": "global",
      "properties": {}
    },
    {
      "type": "Microsoft.Network/privateDnsZones/virtualNetworkLinks",
      "apiVersion": "2020-06-01",
      "name": "[concat(variables('privateDnsZoneName'), '/', parameters('vnetName'), '-link')]",
      "location": "global",
      "dependsOn": [
        "[resourceId('Microsoft.Network/privateDnsZones', variables('privateDnsZoneName'))]"
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
      "apiVersion": "2023-04-01",
      "name": "[variables('privateEndpointDnsGroupName')]",
      "dependsOn": [
        "[resourceId('Microsoft.Network/privateEndpoints', parameters('privateEndpointName'))]",
        "[resourceId('Microsoft.Network/privateDnsZones', variables('privateDnsZoneName'))]"
      ],
      "properties": {
        "privateDnsZoneConfigs": [
          {
            "name": "config1",
            "properties": {
              "privateDnsZoneId": "[resourceId('Microsoft.Network/privateDnsZones', variables('privateDnsZoneName'))]"
            }
          }
        ]
      }
    }
  ],
  "outputs": {
    "registryLoginServer": {
      "type": "string",
      "value": "[reference(resourceId('Microsoft.ContainerRegistry/registries', parameters('registryName'))).loginServer]"
    },
    "privateEndpointId": {
      "type": "string",
      "value": "[resourceId('Microsoft.Network/privateEndpoints', parameters('privateEndpointName'))]"
    }
  }
}
```

### Deploy ARM Template

```bash
az deployment group create \
  --resource-group myResourceGroup \
  --template-file acr-private-endpoint.json \
  --parameters registryName=myregistry vnetName=myVNet subnetName=mySubnet
```

## Method 4: Bicep Template

```bicep
@description('Name of the Azure Container Registry')
param registryName string

@description('Location for all resources')
param location string = resourceGroup().location

@description('Name of the virtual network')
param vnetName string

@description('Name of the subnet for private endpoint')
param subnetName string

@description('Name of the private endpoint')
param privateEndpointName string = '${registryName}-pe'

var privateDnsZoneName = 'privatelink.azurecr.io'

resource acr 'Microsoft.ContainerRegistry/registries@2023-01-01-preview' = {
  name: registryName
  location: location
  sku: {
    name: 'Premium'
  }
  properties: {
    publicNetworkAccess: 'Disabled'
    networkRuleBypassOptions: 'AzureServices'
  }
}

resource privateEndpoint 'Microsoft.Network/privateEndpoints@2023-04-01' = {
  name: privateEndpointName
  location: location
  properties: {
    subnet: {
      id: resourceId('Microsoft.Network/virtualNetworks/subnets', vnetName, subnetName)
    }
    privateLinkServiceConnections: [
      {
        name: '${privateEndpointName}-connection'
        properties: {
          privateLinkServiceId: acr.id
          groupIds: [
            'registry'
          ]
        }
      }
    ]
  }
}

resource privateDnsZone 'Microsoft.Network/privateDnsZones@2020-06-01' = {
  name: privateDnsZoneName
  location: 'global'
}

resource privateDnsZoneLink 'Microsoft.Network/privateDnsZones/virtualNetworkLinks@2020-06-01' = {
  parent: privateDnsZone
  name: '${vnetName}-link'
  location: 'global'
  properties: {
    registrationEnabled: false
    virtualNetwork: {
      id: resourceId('Microsoft.Network/virtualNetworks', vnetName)
    }
  }
}

resource privateDnsZoneGroup 'Microsoft.Network/privateEndpoints/privateDnsZoneGroups@2023-04-01' = {
  parent: privateEndpoint
  name: 'default'
  properties: {
    privateDnsZoneConfigs: [
      {
        name: 'config1'
        properties: {
          privateDnsZoneId: privateDnsZone.id
        }
      }
    ]
  }
}

output registryLoginServer string = acr.properties.loginServer
output privateEndpointId string = privateEndpoint.id
```

## Geo-Replicated Registry Deployment

For geo-replicated registries, additional configuration is required:

### Add DNS Records for Geo-Replicas

```bash
# Set replica location
REPLICA_LOCATION=eastus

# Get geo-replica data endpoint IP
GEO_REPLICA_DATA_ENDPOINT_PRIVATE_IP=$(az network nic show \
  --ids $NETWORK_INTERFACE_ID \
  --query "ipConfigurations[?privateLinkConnectionProperties.requiredMemberName=='registry_data_$REPLICA_LOCATION'].privateIPAddress" \
  --output tsv)

# Create DNS record for replica
az network private-dns record-set a create \
  --name ${REGISTRY_NAME}.${REPLICA_LOCATION}.data \
  --zone-name $DNS_ZONE_NAME \
  --resource-group $RESOURCE_GROUP

az network private-dns record-set a add-record \
  --record-set-name ${REGISTRY_NAME}.${REPLICA_LOCATION}.data \
  --zone-name $DNS_ZONE_NAME \
  --resource-group $RESOURCE_GROUP \
  --ipv4-address $GEO_REPLICA_DATA_ENDPOINT_PRIVATE_IP
```

### Approve Pending Connections

When adding geo-replications, private endpoint connections may require manual approval:

```bash
# List pending connections
az acr private-endpoint-connection list \
  --registry-name $REGISTRY_NAME

# Approve connection
az acr private-endpoint-connection approve \
  --registry-name $REGISTRY_NAME \
  --name <connection-name>
```

## Post-Deployment Steps

### 1. Disable Public Network Access (Optional but Recommended)

**Portal:**
1. Go to registry > **Settings** > **Networking**
2. On **Public access** tab, select **Disabled**
3. Click **Save**

**CLI:**
```bash
az acr update \
  --name $REGISTRY_NAME \
  --public-network-enabled false
```

### 2. Enable Trusted Services (Optional)

Allow trusted Azure services to access the registry:

```bash
az acr update \
  --name $REGISTRY_NAME \
  --allow-trusted-services true
```

### 3. Configure Export Policy (Optional)

Prevent data exfiltration by disabling exports:

```bash
az resource update \
  --resource-group $RESOURCE_GROUP \
  --name $REGISTRY_NAME \
  --resource-type "Microsoft.ContainerRegistry/registries" \
  --api-version "2021-06-01-preview" \
  --set "properties.policies.exportPolicy.status=disabled" \
  --set "properties.publicNetworkAccess=disabled"
```

## Cross-Subscription Deployment

When deploying private endpoints in a different subscription:

```bash
# Register resource provider in the private link subscription
az account set --subscription <private-link-subscription-id>
az provider register --namespace Microsoft.ContainerRegistry

# Then proceed with private endpoint creation
```

## Validation

After deployment, validate the configuration:

```bash
# From a VM in the VNet
dig $REGISTRY_NAME.azurecr.io

# Expected output should show private IP, e.g.:
# myregistry.azurecr.io. -> myregistry.privatelink.azurecr.io. -> 10.0.0.5

# Test connectivity
az acr login --name $REGISTRY_NAME
docker pull $REGISTRY_NAME.azurecr.io/hello-world:latest
```

## Sources

- [Azure Container Registry Private Link Documentation](https://aka.ms/acr/privatelink)
- [ARM Template Reference](/azure/templates/microsoft.containerregistry/registries)
- [Private Endpoint Documentation](/azure/private-link/create-private-endpoint-cli)

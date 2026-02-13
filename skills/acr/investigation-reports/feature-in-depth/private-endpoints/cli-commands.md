# Azure Container Registry Private Endpoints - CLI Commands Reference

## Overview

This document provides a comprehensive reference of Azure CLI commands for managing private endpoints with Azure Container Registry.

## Prerequisites

```bash
# Install or upgrade Azure CLI (version 2.6.0+ required)
az upgrade

# Verify version
az version

# Login to Azure
az login

# Set subscription
az account set --subscription <subscription-id>
```

## Registry Commands

### Create Premium Registry

```bash
az acr create \
  --name <registry-name> \
  --resource-group <resource-group> \
  --location <location> \
  --sku Premium
```

### Update Registry Public Network Access

```bash
# Disable public network access
az acr update \
  --name <registry-name> \
  --public-network-enabled false

# Enable public network access
az acr update \
  --name <registry-name> \
  --public-network-enabled true
```

### Enable/Disable Trusted Services

```bash
# Allow trusted Azure services
az acr update \
  --name <registry-name> \
  --allow-trusted-services true

# Disable trusted services access
az acr update \
  --name <registry-name> \
  --allow-trusted-services false
```

### Show Registry Details

```bash
az acr show \
  --name <registry-name> \
  --query '{name:name, loginServer:loginServer, publicNetworkAccess:publicNetworkAccess}' \
  --output table
```

### Get Registry Resource ID

```bash
REGISTRY_ID=$(az acr show \
  --name <registry-name> \
  --query 'id' \
  --output tsv)
```

## Private Endpoint Connection Commands

### List Private Endpoint Connections

```bash
az acr private-endpoint-connection list \
  --registry-name <registry-name> \
  --output table
```

### Show Private Endpoint Connection Details

```bash
az acr private-endpoint-connection show \
  --registry-name <registry-name> \
  --name <connection-name>
```

### Approve Private Endpoint Connection

```bash
az acr private-endpoint-connection approve \
  --registry-name <registry-name> \
  --name <connection-name> \
  --description "Approved for production use"
```

### Reject Private Endpoint Connection

```bash
az acr private-endpoint-connection reject \
  --registry-name <registry-name> \
  --name <connection-name> \
  --description "Connection rejected - not authorized"
```

### Delete Private Endpoint Connection

```bash
az acr private-endpoint-connection delete \
  --registry-name <registry-name> \
  --name <connection-name>
```

## Network Commands

### Subnet Configuration

```bash
# Disable network policies for private endpoints
az network vnet subnet update \
  --name <subnet-name> \
  --vnet-name <vnet-name> \
  --resource-group <resource-group> \
  --disable-private-endpoint-network-policies

# Show subnet configuration
az network vnet subnet show \
  --name <subnet-name> \
  --vnet-name <vnet-name> \
  --resource-group <resource-group> \
  --output table
```

### Create Private Endpoint

```bash
az network private-endpoint create \
  --name <endpoint-name> \
  --resource-group <resource-group> \
  --vnet-name <vnet-name> \
  --subnet <subnet-name> \
  --private-connection-resource-id <registry-resource-id> \
  --group-ids registry \
  --connection-name <connection-name>
```

### Show Private Endpoint Details

```bash
az network private-endpoint show \
  --name <endpoint-name> \
  --resource-group <resource-group>
```

### List Private Endpoints

```bash
az network private-endpoint list \
  --resource-group <resource-group> \
  --output table
```

### Delete Private Endpoint

```bash
az network private-endpoint delete \
  --name <endpoint-name> \
  --resource-group <resource-group>
```

### Get Private Endpoint Network Interface ID

```bash
NETWORK_INTERFACE_ID=$(az network private-endpoint show \
  --name <endpoint-name> \
  --resource-group <resource-group> \
  --query 'networkInterfaces[0].id' \
  --output tsv)
```

### Show Network Interface IP Configurations

```bash
az network nic show \
  --ids $NETWORK_INTERFACE_ID \
  --query "ipConfigurations[].{Name:name, IP:privateIPAddress, Member:privateLinkConnectionProperties.requiredMemberName}" \
  --output table
```

### Get Specific IP Addresses

```bash
# Registry endpoint IP
az network nic show \
  --ids $NETWORK_INTERFACE_ID \
  --query "ipConfigurations[?privateLinkConnectionProperties.requiredMemberName=='registry'].privateIPAddress" \
  --output tsv

# Data endpoint IP (replace <region> with actual region)
az network nic show \
  --ids $NETWORK_INTERFACE_ID \
  --query "ipConfigurations[?privateLinkConnectionProperties.requiredMemberName=='registry_data_<region>'].privateIPAddress" \
  --output tsv

# Get FQDNs
az network nic show \
  --ids $NETWORK_INTERFACE_ID \
  --query "ipConfigurations[?privateLinkConnectionProperties.requiredMemberName=='registry'].privateLinkConnectionProperties.fqdns" \
  --output tsv
```

## Private DNS Zone Commands

### Create Private DNS Zone

```bash
az network private-dns zone create \
  --resource-group <resource-group> \
  --name "privatelink.azurecr.io"
```

### List Private DNS Zones

```bash
az network private-dns zone list \
  --resource-group <resource-group> \
  --output table
```

### Show Private DNS Zone

```bash
az network private-dns zone show \
  --resource-group <resource-group> \
  --name "privatelink.azurecr.io"
```

### Delete Private DNS Zone

```bash
az network private-dns zone delete \
  --resource-group <resource-group> \
  --name "privatelink.azurecr.io"
```

## VNet Link Commands

### Create VNet Link

```bash
az network private-dns link vnet create \
  --resource-group <resource-group> \
  --zone-name "privatelink.azurecr.io" \
  --name <link-name> \
  --virtual-network <vnet-name> \
  --registration-enabled false
```

### List VNet Links

```bash
az network private-dns link vnet list \
  --resource-group <resource-group> \
  --zone-name "privatelink.azurecr.io" \
  --output table
```

### Show VNet Link

```bash
az network private-dns link vnet show \
  --resource-group <resource-group> \
  --zone-name "privatelink.azurecr.io" \
  --name <link-name>
```

### Delete VNet Link

```bash
az network private-dns link vnet delete \
  --resource-group <resource-group> \
  --zone-name "privatelink.azurecr.io" \
  --name <link-name>
```

## DNS Record Commands

### Create A-Record Set

```bash
az network private-dns record-set a create \
  --name <record-name> \
  --zone-name "privatelink.azurecr.io" \
  --resource-group <resource-group>
```

### Add A-Record

```bash
az network private-dns record-set a add-record \
  --record-set-name <record-name> \
  --zone-name "privatelink.azurecr.io" \
  --resource-group <resource-group> \
  --ipv4-address <ip-address>
```

### List A-Records

```bash
az network private-dns record-set a list \
  --zone-name "privatelink.azurecr.io" \
  --resource-group <resource-group> \
  --output table
```

### Remove A-Record

```bash
az network private-dns record-set a remove-record \
  --record-set-name <record-name> \
  --zone-name "privatelink.azurecr.io" \
  --resource-group <resource-group> \
  --ipv4-address <ip-address>
```

### Delete A-Record Set

```bash
az network private-dns record-set a delete \
  --name <record-name> \
  --zone-name "privatelink.azurecr.io" \
  --resource-group <resource-group>
```

## Health Check Commands

### Check Registry Health

```bash
az acr check-health \
  --name <registry-name>
```

### Check Health with VNet Validation

```bash
# Verify DNS configuration in a virtual network
az acr check-health \
  --name <registry-name> \
  --vnet <vnet-name>

# Using VNet resource ID (for cross-subscription)
az acr check-health \
  --name <registry-name> \
  --vnet /subscriptions/<sub-id>/resourceGroups/<rg>/providers/Microsoft.Network/virtualNetworks/<vnet-name>
```

### Check Health with All Diagnostics

```bash
az acr check-health \
  --name <registry-name> \
  --ignore-errors \
  --yes
```

## Network Rule Commands

### List Network Rules

```bash
az acr network-rule list \
  --name <registry-name> \
  --output table
```

### Add IP Network Rule

```bash
az acr network-rule add \
  --name <registry-name> \
  --ip-address <ip-address-or-range>
```

### Remove IP Network Rule

```bash
az acr network-rule remove \
  --name <registry-name> \
  --ip-address <ip-address-or-range>
```

### Set Default Action

```bash
# Deny by default (recommended with private endpoints)
az acr update \
  --name <registry-name> \
  --default-action Deny

# Allow by default
az acr update \
  --name <registry-name> \
  --default-action Allow
```

## Export Policy Commands

### Disable Export (Data Loss Prevention)

```bash
az resource update \
  --resource-group <resource-group> \
  --name <registry-name> \
  --resource-type "Microsoft.ContainerRegistry/registries" \
  --api-version "2021-06-01-preview" \
  --set "properties.policies.exportPolicy.status=disabled" \
  --set "properties.publicNetworkAccess=disabled"
```

### Enable Export

```bash
az resource update \
  --resource-group <resource-group> \
  --name <registry-name> \
  --resource-type "Microsoft.ContainerRegistry/registries" \
  --api-version "2021-06-01-preview" \
  --set "properties.policies.exportPolicy.status=enabled"
```

## Data Endpoint Commands

### Enable Dedicated Data Endpoints

```bash
az acr update \
  --name <registry-name> \
  --data-endpoint-enabled
```

### Show Data Endpoints

```bash
az acr show-endpoints \
  --name <registry-name>
```

## Complete Workflow Scripts

### Full Private Endpoint Setup Script

```bash
#!/bin/bash
set -e

# Configuration
REGISTRY_NAME="myregistry"
RESOURCE_GROUP="myResourceGroup"
LOCATION="westus2"
VNET_NAME="myVNet"
SUBNET_NAME="mySubnet"
PE_NAME="${REGISTRY_NAME}-pe"
DNS_ZONE="privatelink.azurecr.io"

# Create Premium registry
az acr create \
  --name $REGISTRY_NAME \
  --resource-group $RESOURCE_GROUP \
  --location $LOCATION \
  --sku Premium

# Disable network policies
az network vnet subnet update \
  --name $SUBNET_NAME \
  --vnet-name $VNET_NAME \
  --resource-group $RESOURCE_GROUP \
  --disable-private-endpoint-network-policies

# Create private DNS zone
az network private-dns zone create \
  --resource-group $RESOURCE_GROUP \
  --name $DNS_ZONE

# Link DNS zone to VNet
az network private-dns link vnet create \
  --resource-group $RESOURCE_GROUP \
  --zone-name $DNS_ZONE \
  --name "${VNET_NAME}-link" \
  --virtual-network $VNET_NAME \
  --registration-enabled false

# Get registry ID
REGISTRY_ID=$(az acr show --name $REGISTRY_NAME --query 'id' --output tsv)

# Create private endpoint
az network private-endpoint create \
  --name $PE_NAME \
  --resource-group $RESOURCE_GROUP \
  --vnet-name $VNET_NAME \
  --subnet $SUBNET_NAME \
  --private-connection-resource-id $REGISTRY_ID \
  --group-ids registry \
  --connection-name "${PE_NAME}-connection"

# Get network interface ID
NIC_ID=$(az network private-endpoint show \
  --name $PE_NAME \
  --resource-group $RESOURCE_GROUP \
  --query 'networkInterfaces[0].id' \
  --output tsv)

# Get IPs
REGISTRY_IP=$(az network nic show --ids $NIC_ID \
  --query "ipConfigurations[?privateLinkConnectionProperties.requiredMemberName=='registry'].privateIPAddress" \
  --output tsv)

DATA_IP=$(az network nic show --ids $NIC_ID \
  --query "ipConfigurations[?privateLinkConnectionProperties.requiredMemberName=='registry_data_${LOCATION}'].privateIPAddress" \
  --output tsv)

# Create DNS records
az network private-dns record-set a create \
  --name $REGISTRY_NAME \
  --zone-name $DNS_ZONE \
  --resource-group $RESOURCE_GROUP

az network private-dns record-set a add-record \
  --record-set-name $REGISTRY_NAME \
  --zone-name $DNS_ZONE \
  --resource-group $RESOURCE_GROUP \
  --ipv4-address $REGISTRY_IP

az network private-dns record-set a create \
  --name "${REGISTRY_NAME}.${LOCATION}.data" \
  --zone-name $DNS_ZONE \
  --resource-group $RESOURCE_GROUP

az network private-dns record-set a add-record \
  --record-set-name "${REGISTRY_NAME}.${LOCATION}.data" \
  --zone-name $DNS_ZONE \
  --resource-group $RESOURCE_GROUP \
  --ipv4-address $DATA_IP

# Disable public access
az acr update --name $REGISTRY_NAME --public-network-enabled false

echo "Private endpoint setup complete!"
echo "Registry: $REGISTRY_NAME.azurecr.io"
echo "Registry IP: $REGISTRY_IP"
echo "Data Endpoint IP: $DATA_IP"
```

### Cleanup Script

```bash
#!/bin/bash
set -e

REGISTRY_NAME="myregistry"
RESOURCE_GROUP="myResourceGroup"
PE_NAME="${REGISTRY_NAME}-pe"
DNS_ZONE="privatelink.azurecr.io"
VNET_NAME="myVNet"

# Delete private endpoint
az network private-endpoint delete \
  --name $PE_NAME \
  --resource-group $RESOURCE_GROUP

# Delete DNS records
az network private-dns record-set a delete \
  --name $REGISTRY_NAME \
  --zone-name $DNS_ZONE \
  --resource-group $RESOURCE_GROUP \
  --yes

az network private-dns record-set a delete \
  --name "${REGISTRY_NAME}.westus2.data" \
  --zone-name $DNS_ZONE \
  --resource-group $RESOURCE_GROUP \
  --yes

# Delete VNet link
az network private-dns link vnet delete \
  --resource-group $RESOURCE_GROUP \
  --zone-name $DNS_ZONE \
  --name "${VNET_NAME}-link" \
  --yes

# Delete DNS zone
az network private-dns zone delete \
  --resource-group $RESOURCE_GROUP \
  --name $DNS_ZONE \
  --yes

echo "Cleanup complete!"
```

## Sources

- [Azure CLI ACR Reference](/cli/azure/acr)
- [Azure CLI Network Reference](/cli/azure/network)
- [ACR Private Link Documentation](https://aka.ms/acr/privatelink)

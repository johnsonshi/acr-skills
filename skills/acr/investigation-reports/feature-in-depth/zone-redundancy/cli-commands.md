# ACR Zone Redundancy - CLI Commands Reference

## Overview

This document provides a comprehensive reference for Azure CLI commands related to zone redundancy in Azure Container Registry.

## Prerequisites

Ensure you have the latest Azure CLI installed:

```azurecli
# Check current version
az --version

# Install or upgrade Azure CLI
# See: https://docs.microsoft.com/cli/azure/install-azure-cli
```

Login to Azure:

```azurecli
az login
```

## Resource Group Commands

### Create Resource Group

```azurecli
# Create resource group in availability zone supported region
az group create \
  --name <resource-group-name> \
  --location <location>

# Example
az group create \
  --name myACRResourceGroup \
  --location eastus
```

## Registry Creation Commands

### Create Zone-Redundant Registry

```azurecli
# Full command with zone redundancy
az acr create \
  --resource-group <resource-group-name> \
  --name <container-registry-name> \
  --location <region-name> \
  --zone-redundancy enabled \
  --sku Premium

# Example
az acr create \
  --resource-group myACRResourceGroup \
  --name myZoneRedundantACR \
  --location eastus \
  --zone-redundancy enabled \
  --sku Premium

# With JSON output for verification
az acr create \
  --resource-group myACRResourceGroup \
  --name myZoneRedundantACR \
  --location eastus \
  --zone-redundancy enabled \
  --sku Premium \
  --output jsonc
```

### Create Registry with Additional Options

```azurecli
# Zone-redundant registry with admin user disabled (recommended)
az acr create \
  --resource-group <resource-group-name> \
  --name <container-registry-name> \
  --location <region-name> \
  --zone-redundancy enabled \
  --sku Premium \
  --admin-enabled false

# Zone-redundant registry with public network access disabled
az acr create \
  --resource-group <resource-group-name> \
  --name <container-registry-name> \
  --location <region-name> \
  --zone-redundancy enabled \
  --sku Premium \
  --public-network-enabled false
```

## Registry Query Commands

### Show Registry Details

```azurecli
# Show full registry details
az acr show \
  --name <registry-name> \
  --resource-group <resource-group-name>

# Show only zone redundancy status
az acr show \
  --name <registry-name> \
  --query zoneRedundancy

# Show zone redundancy and location
az acr show \
  --name <registry-name> \
  --query "[location, zoneRedundancy]"

# Show registry in table format
az acr show \
  --name <registry-name> \
  --output table
```

### List All Registries

```azurecli
# List registries in resource group
az acr list \
  --resource-group <resource-group-name> \
  --output table

# List with zone redundancy status
az acr list \
  --resource-group <resource-group-name> \
  --query "[].{Name:name, Location:location, SKU:sku.name, ZoneRedundancy:zoneRedundancy}" \
  --output table
```

## Geo-Replication Commands

### Create Zone-Redundant Replica

```azurecli
# Create zone-redundant replication
az acr replication create \
  --location <region-name> \
  --resource-group <resource-group-name> \
  --registry <container-registry-name> \
  --zone-redundancy enabled

# Example
az acr replication create \
  --location westus2 \
  --resource-group myACRResourceGroup \
  --registry myZoneRedundantACR \
  --zone-redundancy enabled

# With specific replication name
az acr replication create \
  --name westus2 \
  --location westus2 \
  --resource-group myACRResourceGroup \
  --registry myZoneRedundantACR \
  --zone-redundancy enabled
```

### List Replications

```azurecli
# List all replications for registry
az acr replication list \
  --registry <registry-name> \
  --output table

# List with zone redundancy status
az acr replication list \
  --registry <registry-name> \
  --query "[].{Name:name, Location:location, Status:provisioningState, ZoneRedundancy:zoneRedundancy}" \
  --output table
```

### Show Replication Details

```azurecli
# Show specific replication
az acr replication show \
  --name <location> \
  --registry <registry-name>

# Show replication zone redundancy
az acr replication show \
  --name <location> \
  --registry <registry-name> \
  --query zoneRedundancy
```

### Update Replication

```azurecli
# Disable routing to replication (for troubleshooting)
az acr replication update \
  --name <location> \
  --registry <registry-name> \
  --resource-group <resource-group-name> \
  --region-endpoint-enabled false

# Re-enable routing
az acr replication update \
  --name <location> \
  --registry <registry-name> \
  --resource-group <resource-group-name> \
  --region-endpoint-enabled true
```

### Delete Replication

```azurecli
# Delete a replication
az acr replication delete \
  --name <location> \
  --registry <registry-name>

# Example
az acr replication delete \
  --name westus2 \
  --registry myZoneRedundantACR
```

## Registry Management Commands

### Update Registry SKU

```azurecli
# Upgrade to Premium (required for explicit zone redundancy config)
az acr update \
  --name <registry-name> \
  --sku Premium

# Note: Zone redundancy is automatic in supported regions for all SKUs
```

### Show Registry Usage

```azurecli
# Show storage and resource usage
az acr show-usage \
  --name <registry-name> \
  --resource-group <resource-group-name> \
  --output table

# Show detailed usage
az acr show-usage \
  --name <registry-name> \
  --output jsonc
```

## Deployment Template Commands

### Deploy Bicep Template

```azurecli
# Deploy zone-redundant registry from Bicep
az deployment group create \
  --resource-group <resource-group-name> \
  --template-file registryZone.bicep \
  --parameters containerRegistryName=<registry-name>

# Deploy with specific parameters
az deployment group create \
  --resource-group <resource-group-name> \
  --template-file registryZone.bicep \
  --parameters containerRegistryName=<registry-name> \
               containerRegistryZoneRedundancy=Enabled \
               location=eastus
```

### Deploy Replication Template

```azurecli
# Deploy zone-redundant replica from Bicep
az deployment group create \
  --resource-group <resource-group-name> \
  --template-file replicaZone.bicep \
  --parameters containerRegistryName=<registry-name> \
               replicaLocation=<replica-location>

# Example
az deployment group create \
  --resource-group myACRResourceGroup \
  --template-file replicaZone.bicep \
  --parameters containerRegistryName=myZoneRedundantACR \
               replicaLocation=westeurope \
               replicaZoneRedundancy=Enabled
```

## Verification Commands

### Verify Zone Redundancy Status

```azurecli
# Check registry zone redundancy
az acr show \
  --name <registry-name> \
  --query "{Name:name, Location:location, SKU:sku.name, ZoneRedundancy:zoneRedundancy}" \
  --output table

# Check all replicas zone redundancy
az acr replication list \
  --registry <registry-name> \
  --query "[].{Location:location, ZoneRedundancy:zoneRedundancy, Status:provisioningState}" \
  --output table
```

### Check Region Availability Zone Support

```azurecli
# List regions with availability zones
az account list-locations \
  --query "[?availabilityZoneMappings != null].{Name:name, DisplayName:displayName}" \
  --output table
```

## Complete Example Workflow

```azurecli
#!/bin/bash

# Variables
RESOURCE_GROUP="myACRResourceGroup"
REGISTRY_NAME="myZoneRedundantACR"
HOME_LOCATION="eastus"
REPLICA_LOCATION="westeurope"

# 1. Create resource group
az group create \
  --name $RESOURCE_GROUP \
  --location $HOME_LOCATION

# 2. Create zone-redundant registry
az acr create \
  --resource-group $RESOURCE_GROUP \
  --name $REGISTRY_NAME \
  --location $HOME_LOCATION \
  --zone-redundancy enabled \
  --sku Premium

# 3. Verify registry creation
az acr show \
  --name $REGISTRY_NAME \
  --query "{Name:name, Location:location, ZoneRedundancy:zoneRedundancy}" \
  --output table

# 4. Add zone-redundant replica
az acr replication create \
  --location $REPLICA_LOCATION \
  --registry $REGISTRY_NAME \
  --zone-redundancy enabled

# 5. Verify replication
az acr replication list \
  --registry $REGISTRY_NAME \
  --query "[].{Location:location, ZoneRedundancy:zoneRedundancy, Status:provisioningState}" \
  --output table

# 6. Show usage
az acr show-usage \
  --name $REGISTRY_NAME \
  --output table

echo "Zone-redundant registry $REGISTRY_NAME created successfully!"
```

## Troubleshooting Commands

### Check Provisioning State

```azurecli
# Check registry provisioning state
az acr show \
  --name <registry-name> \
  --query provisioningState

# Check replication provisioning state
az acr replication show \
  --name <location> \
  --registry <registry-name> \
  --query provisioningState
```

### View Diagnostic Settings

```azurecli
# List diagnostic settings
az monitor diagnostic-settings list \
  --resource $(az acr show --name <registry-name> --query id -o tsv)
```

## Sources

- `/Users/johnsonshi/repos/acr-skills/submodules/azure-management-docs/articles/container-registry/zone-redundancy.md`
- `/Users/johnsonshi/repos/acr-skills/submodules/azure-management-docs/articles/container-registry/container-registry-geo-replication.md`
- `/Users/johnsonshi/repos/acr-skills/submodules/azure-management-docs/articles/container-registry/container-registry-get-started-azure-cli.md`
- `/Users/johnsonshi/repos/acr-skills/submodules/acr/docs/container-registry-oras-artifacts.md`

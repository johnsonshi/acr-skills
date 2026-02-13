# ACR Geo-Replication CLI Commands Reference

## Overview

This document provides a comprehensive reference for Azure CLI commands related to ACR geo-replication management.

## Prerequisites

```bash
# Ensure Azure CLI is installed (version 2.0.31 or later)
az --version

# Log in to Azure
az login

# Set subscription context
az account set --subscription <subscription-id>
```

## Registry Management Commands

### Create a Premium Registry

```bash
# Create resource group
az group create \
  --name myResourceGroup \
  --location westus

# Create Premium registry (required for geo-replication)
az acr create \
  --name myregistry \
  --resource-group myResourceGroup \
  --sku Premium \
  --location westus
```

### Upgrade Existing Registry to Premium

```bash
# Upgrade to Premium SKU
az acr update \
  --name myregistry \
  --sku Premium
```

### Create Zone-Redundant Premium Registry

```bash
az acr create \
  --resource-group myResourceGroup \
  --name myregistry \
  --location eastus \
  --zone-redundancy enabled \
  --sku Premium
```

## Replication Commands

### az acr replication create

Creates a new geo-replication for a container registry.

**Syntax:**
```bash
az acr replication create \
  --registry <registry-name> \
  --location <region> \
  [--name <replication-name>] \
  [--resource-group <resource-group>] \
  [--zone-redundancy {Disabled, Enabled}]
```

**Examples:**

```bash
# Create basic replication
az acr replication create \
  --name eastus \
  --registry myregistry \
  --resource-group myResourceGroup \
  --location eastus

# Create zone-redundant replication
az acr replication create \
  --location westeurope \
  --registry myregistry \
  --resource-group myResourceGroup \
  --zone-redundancy enabled

# Create multiple replications
az acr replication create --registry myregistry --location eastus
az acr replication create --registry myregistry --location westeurope
az acr replication create --registry myregistry --location southeastasia
```

### az acr replication list

Lists all replications for a container registry.

**Syntax:**
```bash
az acr replication list \
  --registry <registry-name> \
  [--resource-group <resource-group>] \
  [--output {json, jsonc, table, tsv}]
```

**Examples:**

```bash
# List replications in table format
az acr replication list --registry myregistry --output table

# List replications in JSON format
az acr replication list --registry myregistry --output json

# List with specific fields
az acr replication list \
  --registry myregistry \
  --query "[].{Name:name, Location:location, Status:provisioningState, ZoneRedundancy:zoneRedundancy}" \
  --output table
```

**Sample Output:**
```
NAME        LOCATION    STATUS       ZONE-REDUNDANCY
----------  ----------  -----------  ---------------
westus      westus      Succeeded    Disabled
eastus      eastus      Succeeded    Enabled
westeurope  westeurope  Succeeded    Disabled
```

### az acr replication show

Shows the details of a specific replication.

**Syntax:**
```bash
az acr replication show \
  --name <replication-name> \
  --registry <registry-name> \
  [--resource-group <resource-group>]
```

**Example:**
```bash
az acr replication show \
  --name eastus \
  --registry myregistry
```

**Sample Output:**
```json
{
  "id": "/subscriptions/.../replications/eastus",
  "location": "eastus",
  "name": "eastus",
  "provisioningState": "Succeeded",
  "regionEndpointEnabled": true,
  "status": {
    "displayStatus": "Ready",
    "message": null,
    "timestamp": "2024-01-15T10:30:00Z"
  },
  "type": "Microsoft.ContainerRegistry/registries/replications",
  "zoneRedundancy": "Enabled"
}
```

### az acr replication update

Updates a replication configuration.

**Syntax:**
```bash
az acr replication update \
  --name <replication-name> \
  --registry <registry-name> \
  [--resource-group <resource-group>] \
  [--region-endpoint-enabled {false, true}]
```

**Examples:**

```bash
# Disable routing to a replication (for troubleshooting)
az acr replication update \
  --name westus \
  --registry myregistry \
  --resource-group myResourceGroup \
  --region-endpoint-enabled false

# Re-enable routing to a replication
az acr replication update \
  --name westus \
  --registry myregistry \
  --resource-group myResourceGroup \
  --region-endpoint-enabled true
```

**Note**: The `--region-endpoint-enabled` option is in preview. When disabled:
- Traffic Manager no longer routes docker push/pull requests to that region
- Data synchronization continues regardless of routing status

### az acr replication delete

Deletes a replication from a container registry.

**Syntax:**
```bash
az acr replication delete \
  --name <replication-name> \
  --registry <registry-name> \
  [--resource-group <resource-group>] \
  [--yes]
```

**Examples:**

```bash
# Delete with confirmation prompt
az acr replication delete \
  --name eastus \
  --registry myregistry

# Delete without confirmation
az acr replication delete \
  --name eastus \
  --registry myregistry \
  --yes
```

## Endpoint Commands

### az acr show-endpoints

Shows all endpoints for a container registry.

**Syntax:**
```bash
az acr show-endpoints \
  --name <registry-name> \
  [--resource-group <resource-group>]
```

**Example:**
```bash
az acr show-endpoints --name myregistry
```

**Sample Output:**
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
    },
    {
      "region": "westeurope",
      "endpoint": "myregistry.westeurope.data.azurecr.io"
    }
  ]
}
```

### Enable Dedicated Data Endpoints

```bash
# Enable dedicated data endpoints
az acr update --name myregistry --data-endpoint-enabled

# Verify endpoints
az acr show-endpoints --name myregistry
```

## Registry Information Commands

### az acr show

Shows details about a container registry.

**Example:**
```bash
az acr show \
  --name myregistry \
  --query "{LoginServer:loginServer, Sku:sku.name, Location:location}" \
  --output table
```

### az acr show-usage

Shows the current usage of a container registry.

**Example:**
```bash
az acr show-usage \
  --resource-group myResourceGroup \
  --name myregistry \
  --output table
```

**Sample Output:**
```
NAME                        LIMIT         CURRENT VALUE    UNIT
--------------------------  ------------  ---------------  ------
Size                        536870912000  215629144        Bytes
Webhooks                    500           2                Count
Geo-replications            -1            3                Count
IPRules                     100           1                Count
VNetRules                   100           0                Count
PrivateEndpointConnections  10            0                Count
```

## Webhook Commands for Geo-Replicated Registries

### Create Regional Webhook

```bash
# Create webhook for specific region
az acr webhook create \
  --registry myregistry \
  --name mywebhook \
  --location eastus \
  --uri https://myserver.com/webhook \
  --actions push delete
```

### List Webhooks

```bash
az acr webhook list --registry myregistry --output table
```

## Authentication Commands

### Login to Registry

```bash
# Login to registry
az acr login --name myregistry

# Login with specific resource group
az acr login \
  --name myregistry \
  --resource-group myResourceGroup
```

## Troubleshooting Commands

### Check Registry Health

```bash
# Check registry health
az acr check-health \
  --name myregistry \
  --yes
```

### Check Replication Status

```bash
# Check all replications status
az acr replication list \
  --registry myregistry \
  --query "[].{Name:name, Location:location, Status:status.displayStatus}" \
  --output table
```

### DNS Resolution Troubleshooting

```bash
# Use nslookup or dig to check DNS resolution
nslookup myregistry.azurecr.io

# On Linux
dig myregistry.azurecr.io
```

## PowerShell Equivalents

| Azure CLI Command | Azure PowerShell Equivalent |
|-------------------|----------------------------|
| `az acr create` | `New-AzContainerRegistry` |
| `az acr update` | `Update-AzContainerRegistry` |
| `az acr replication create` | `New-AzContainerRegistryReplication` |
| `az acr replication list` | `Get-AzContainerRegistryReplication` |
| `az acr replication delete` | `Remove-AzContainerRegistryReplication` |
| `az acr show-usage` | `Get-AzContainerRegistryUsage` |

### PowerShell Examples

```powershell
# Create replication
New-AzContainerRegistryReplication `
  -ResourceGroupName myResourceGroup `
  -RegistryName myregistry `
  -Location eastus

# List replications
Get-AzContainerRegistryReplication `
  -ResourceGroupName myResourceGroup `
  -RegistryName myregistry

# Delete replication
Remove-AzContainerRegistryReplication `
  -ResourceGroupName myResourceGroup `
  -RegistryName myregistry `
  -Name eastus

# Change SKU
Update-AzContainerRegistry `
  -ResourceGroupName myResourceGroup `
  -Name myregistry `
  -Sku Premium
```

## Common Command Patterns

### Set Up Complete Geo-Replicated Registry

```bash
# 1. Create resource group
az group create --name myResourceGroup --location westus

# 2. Create Premium registry with zone redundancy
az acr create \
  --name myregistry \
  --resource-group myResourceGroup \
  --sku Premium \
  --location westus \
  --zone-redundancy enabled

# 3. Add replications
az acr replication create --registry myregistry --location eastus --zone-redundancy enabled
az acr replication create --registry myregistry --location westeurope --zone-redundancy enabled

# 4. Enable dedicated data endpoints
az acr update --name myregistry --data-endpoint-enabled

# 5. Verify configuration
az acr replication list --registry myregistry --output table
az acr show-endpoints --name myregistry
```

### Clean Up Geo-Replication

```bash
# 1. Delete all replications
az acr replication delete --name eastus --registry myregistry --yes
az acr replication delete --name westeurope --registry myregistry --yes

# 2. Optionally downgrade SKU
az acr update --name myregistry --sku Standard
```

## Sources

- `/Users/johnsonshi/repos/acr-skills/submodules/azure-management-docs/articles/container-registry/container-registry-geo-replication.md`
- `/Users/johnsonshi/repos/acr-skills/submodules/azure-management-docs/articles/container-registry/zone-redundancy.md`
- `/Users/johnsonshi/repos/acr-skills/submodules/azure-management-docs/articles/container-registry/container-registry-skus.md`
- Azure CLI documentation: https://docs.microsoft.com/cli/azure/acr/replication

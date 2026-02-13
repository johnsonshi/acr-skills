# ACR Dedicated Data Endpoints - CLI Commands Reference

## Overview

This document provides a comprehensive reference of Azure CLI commands for managing dedicated data endpoints in Azure Container Registry.

## Prerequisites

- Azure CLI version 2.4.0 or later
- Premium SKU container registry
- Appropriate permissions (Contributor or Owner role on the registry)

```bash
# Check Azure CLI version
az --version

# Install or upgrade Azure CLI
az upgrade

# Login to Azure
az login

# Set subscription (if needed)
az account set --subscription <subscription-name-or-id>
```

## Core Commands

### Enable Dedicated Data Endpoints

```bash
az acr update --name <registry-name> --data-endpoint-enabled
```

**Parameters:**
- `--name`, `-n`: Name of the container registry (required)
- `--data-endpoint-enabled`: Flag to enable dedicated data endpoints

**Example:**
```bash
az acr update --name contoso --data-endpoint-enabled
```

**Output:**
```json
{
  "adminUserEnabled": false,
  "dataEndpointEnabled": true,
  "dataEndpointHostNames": [
    "contoso.eastus.data.azurecr.io"
  ],
  "encryption": {...},
  "id": "/subscriptions/.../resourceGroups/.../providers/Microsoft.ContainerRegistry/registries/contoso",
  "location": "eastus",
  "loginServer": "contoso.azurecr.io",
  "name": "contoso",
  ...
}
```

### Disable Dedicated Data Endpoints

```bash
az acr update --name <registry-name> --data-endpoint-enabled false
```

**Example:**
```bash
az acr update --name contoso --data-endpoint-enabled false
```

**Warning:** Disabling will break clients relying on dedicated data endpoint firewall rules.

### View Data Endpoints

```bash
az acr show-endpoints --name <registry-name>
```

**Parameters:**
- `--name`, `-n`: Name of the container registry (required)

**Example:**
```bash
az acr show-endpoints --name contoso
```

**Output:**
```json
{
  "loginServer": "contoso.azurecr.io",
  "dataEndpoints": [
    {
      "region": "eastus",
      "endpoint": "contoso.eastus.data.azurecr.io"
    },
    {
      "region": "westus",
      "endpoint": "contoso.westus.data.azurecr.io"
    }
  ]
}
```

### Check Data Endpoint Status

```bash
az acr show --name <registry-name> --query "dataEndpointEnabled"
```

**Example:**
```bash
az acr show --name contoso --query "dataEndpointEnabled"
```

**Output:**
```
true
```

### Show Full Registry Configuration

```bash
az acr show --name <registry-name>
```

**Relevant fields in output:**
```json
{
  "dataEndpointEnabled": true,
  "dataEndpointHostNames": [
    "contoso.eastus.data.azurecr.io",
    "contoso.westus.data.azurecr.io"
  ],
  ...
}
```

## Geo-Replication Commands

### Create Replication (Auto-Creates Data Endpoint)

```bash
az acr replication create --registry <registry-name> --location <region>
```

**Example:**
```bash
az acr replication create --registry contoso --location westus
```

When dedicated data endpoints are enabled, a data endpoint is automatically created in the new region.

### List Replications

```bash
az acr replication list --registry <registry-name> --output table
```

**Example:**
```bash
az acr replication list --registry contoso --output table
```

**Output:**
```
NAME     LOCATION    PROVISIONING STATE    STATUS
-------  ----------  --------------------  --------
eastus   eastus      Succeeded             Ready
westus   westus      Succeeded             Ready
```

### Delete Replication

```bash
az acr replication delete --registry <registry-name> --name <region>
```

**Example:**
```bash
az acr replication delete --registry contoso --name westus
```

This also removes the data endpoint for that region.

## Private Endpoint Commands

When Private Link is configured, dedicated data endpoints are automatically enabled.

### Create Private Endpoint

```bash
# Get registry resource ID
REGISTRY_ID=$(az acr show --name <registry-name> --query 'id' --output tsv)

# Create private endpoint
az network private-endpoint create \
    --name <endpoint-name> \
    --resource-group <resource-group> \
    --vnet-name <vnet-name> \
    --subnet <subnet-name> \
    --private-connection-resource-id $REGISTRY_ID \
    --group-ids registry \
    --connection-name <connection-name>
```

### View Private Endpoint IP Configuration

```bash
# Get network interface ID
NETWORK_INTERFACE_ID=$(az network private-endpoint show \
  --name <endpoint-name> \
  --resource-group <resource-group> \
  --query 'networkInterfaces[0].id' \
  --output tsv)

# Get registry private IP
az network nic show \
  --ids $NETWORK_INTERFACE_ID \
  --query "ipConfigurations[?privateLinkConnectionProperties.requiredMemberName=='registry'].privateIPAddress" \
  --output tsv

# Get data endpoint private IP
az network nic show \
  --ids $NETWORK_INTERFACE_ID \
  --query "ipConfigurations[?privateLinkConnectionProperties.requiredMemberName=='registry_data_<region>'].privateIPAddress" \
  --output tsv
```

## Connected Registry Commands

Connected registries require dedicated data endpoints on the parent registry.

### Enable Data Endpoint for Connected Registry

```bash
az acr update --name <parent-registry> --data-endpoint-enabled
```

### Create Connected Registry

```bash
az acr connected-registry create \
    --registry <parent-registry> \
    --name <connected-registry-name> \
    --repository <repo1> <repo2>
```

**Example:**
```bash
az acr connected-registry create \
    --registry contoso \
    --name myconnectedregistry \
    --repository "hello-world" "nginx"
```

### Get Connected Registry Connection String

```bash
az acr connected-registry get-settings \
    --registry <parent-registry> \
    --name <connected-registry-name> \
    --generate-password 1 \
    --parent-protocol https
```

**Output includes data endpoint:**
```json
{
  "ACR_REGISTRY_CONNECTION_STRING": "ConnectedRegistryName=myconnectedregistry;SyncTokenName=myconnectedregistry-sync-token;SyncTokenPassword=xxx;ParentGatewayEndpoint=contoso.eastus.data.azurecr.io;ParentEndpointProtocol=https"
}
```

## DNS Configuration Commands (for Private Link)

### Create Private DNS Zone

```bash
az network private-dns zone create \
    --resource-group <resource-group> \
    --name "privatelink.azurecr.io"
```

### Create DNS Records for Data Endpoints

```bash
# Create A-record set for data endpoint
az network private-dns record-set a create \
    --name <registry>.<region>.data \
    --zone-name privatelink.azurecr.io \
    --resource-group <resource-group>

# Add A-record
az network private-dns record-set a add-record \
    --record-set-name <registry>.<region>.data \
    --zone-name privatelink.azurecr.io \
    --resource-group <resource-group> \
    --ipv4-address <data-endpoint-private-ip>
```

## Diagnostic Commands

### Check Registry Health

```bash
az acr check-health --name <registry-name> --yes
```

### Check Health with VNet Validation

```bash
az acr check-health --name <registry-name> --vnet <vnet-resource-id> --yes
```

### Enable Diagnostic Logging

```bash
az monitor diagnostic-settings create \
    --name ACRDiagnostics \
    --resource $(az acr show --name <registry-name> --query id -o tsv) \
    --logs '[{"category": "ContainerRegistryLoginEvents", "enabled": true}]' \
    --workspace <log-analytics-workspace-id>
```

## Command Quick Reference Table

| Task | Command |
|------|---------|
| Enable data endpoints | `az acr update -n <name> --data-endpoint-enabled` |
| Disable data endpoints | `az acr update -n <name> --data-endpoint-enabled false` |
| View data endpoints | `az acr show-endpoints -n <name>` |
| Check if enabled | `az acr show -n <name> --query "dataEndpointEnabled"` |
| Show endpoint hostnames | `az acr show -n <name> --query "dataEndpointHostNames"` |
| Create replication | `az acr replication create -r <name> -l <location>` |
| List replications | `az acr replication list -r <name>` |
| Check health | `az acr check-health -n <name> --yes` |

## Output Formatting Options

### Table Format
```bash
az acr show-endpoints --name contoso --output table
```

### JSON Format (default)
```bash
az acr show-endpoints --name contoso --output json
```

### YAML Format
```bash
az acr show-endpoints --name contoso --output yaml
```

### TSV Format (for scripting)
```bash
az acr show --name contoso --query "dataEndpointHostNames" --output tsv
```

## Common Scripting Patterns

### Get All Data Endpoints as Array

```bash
DATA_ENDPOINTS=$(az acr show-endpoints --name contoso --query "dataEndpoints[].endpoint" -o tsv)
echo "$DATA_ENDPOINTS"
```

### Check and Enable in Script

```bash
#!/bin/bash
REGISTRY_NAME="contoso"

# Check if already enabled
ENABLED=$(az acr show --name $REGISTRY_NAME --query "dataEndpointEnabled" -o tsv)

if [ "$ENABLED" != "true" ]; then
    echo "Enabling dedicated data endpoints..."
    az acr update --name $REGISTRY_NAME --data-endpoint-enabled
else
    echo "Dedicated data endpoints already enabled"
fi

# Show endpoints
az acr show-endpoints --name $REGISTRY_NAME
```

### Export Endpoints for Firewall Configuration

```bash
#!/bin/bash
REGISTRY_NAME="contoso"

# Get login server
LOGIN_SERVER=$(az acr show --name $REGISTRY_NAME --query "loginServer" -o tsv)
echo "Login Server: $LOGIN_SERVER"

# Get data endpoints
az acr show-endpoints --name $REGISTRY_NAME --query "dataEndpoints[].endpoint" -o tsv | while read endpoint; do
    echo "Data Endpoint: $endpoint"
done
```

## Error Handling

### Common Errors and Solutions

| Error | Cause | Solution |
|-------|-------|----------|
| `SKU Premium required` | Not Premium tier | Upgrade to Premium SKU |
| `Resource not found` | Registry doesn't exist | Check registry name |
| `Authorization failed` | Insufficient permissions | Request Contributor role |
| `Data endpoint not enabled` | Feature not enabled | Run `az acr update --data-endpoint-enabled` |

### Verify Premium SKU

```bash
az acr show --name <registry-name> --query "sku.tier"
```

**Expected output for Premium:**
```
"Premium"
```

## Sources

- MS Learn: `/articles/container-registry/container-registry-dedicated-data-endpoints.md`
- MS Learn: `/articles/container-registry/container-registry-firewall-access-rules.md`
- MS Learn: `/articles/container-registry/container-registry-private-link.md`
- Azure CLI Reference: `az acr update`, `az acr show-endpoints`
- ACR Repo: `/docs/preview/connected-registry/quickstart-connected-registry-cli.md`

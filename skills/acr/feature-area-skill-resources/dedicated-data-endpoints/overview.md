# ACR Dedicated Data Endpoints Skill

This skill provides comprehensive knowledge about Azure Container Registry dedicated data endpoints.

## When to Use This Skill

Use this skill when answering questions about:
- Regional data endpoints
- Firewall configuration
- Data exfiltration mitigation
- Blob storage access

## Overview

Dedicated data endpoints provide regional endpoints for registry data, enabling tighter firewall rules.

**Requirements:** Premium SKU

## Endpoint Types

| Endpoint | Format | Purpose |
|----------|--------|---------|
| REST API | `myregistry.azurecr.io` | Authentication, management |
| Data | `myregistry.<region>.data.azurecr.io` | Blob storage access |

## Why Use Dedicated Data Endpoints?

### Without Dedicated Endpoints
```
Firewall must allow: *.blob.core.windows.net
(Too broad, allows any Azure Storage)
```

### With Dedicated Endpoints
```
Firewall allows: myregistry.eastus.data.azurecr.io
(Specific to your registry)
```

## Configuration

### Enable Dedicated Data Endpoints
```bash
az acr update --name myregistry --data-endpoint-enabled true
```

### Check Status
```bash
az acr show --name myregistry --query dataEndpointEnabled
```

### Show Endpoints
```bash
az acr show-endpoints --name myregistry -o table
```

## Firewall Rules

### Required Endpoints
```
# REST API endpoint
myregistry.azurecr.io

# Data endpoint (per region)
myregistry.eastus.data.azurecr.io

# For geo-replicated registries
myregistry.westeurope.data.azurecr.io
myregistry.southeastasia.data.azurecr.io
```

### Azure Firewall Rule
```bash
az network firewall application-rule create \
  --firewall-name myFirewall \
  --resource-group myRG \
  --collection-name acr-rules \
  --name acr-data \
  --protocols https=443 \
  --target-fqdns "myregistry.azurecr.io" "myregistry.*.data.azurecr.io"
```

### NSG Rule with Service Tag
```bash
az network nsg rule create \
  --nsg-name myNSG \
  --name AllowACR \
  --priority 100 \
  --destination-service-tag AzureContainerRegistry \
  --destination-port-ranges 443 \
  --access Allow
```

## With Private Link

When using private endpoints:
- Dedicated data endpoints are **automatically enabled**
- Each region gets private IP for data endpoint
- DNS records needed for each data endpoint

```
# Private DNS records
myregistry.azurecr.io → 10.0.0.5
myregistry.eastus.data.azurecr.io → 10.0.0.6
```

## Geo-Replication

Each replica region gets its own data endpoint:

```bash
# Show all endpoints including replicas
az acr show-endpoints --name myregistry
```

Output:
```
Data Endpoint                                   Region
----------------------------------------------  --------------
myregistry.eastus.data.azurecr.io               East US
myregistry.westeurope.data.azurecr.io           West Europe
```

## Connected Registry

Connected registries require dedicated data endpoints for sync:

```bash
# Enable before creating connected registry
az acr update --name myregistry --data-endpoint-enabled true

# Then create connected registry
az acr connected-registry create ...
```

## Temporary Download Links

Data endpoints serve temporary download links:
- **Validity**: 20 minutes
- **Purpose**: Authenticate once, download directly
- **Security**: Links are registry-specific

## Migration

### Before Enabling
1. Document current firewall rules
2. Add new data endpoint rules
3. Test connectivity

### After Enabling
1. Verify pulls work
2. Remove broad `*.blob.core.windows.net` rules

```bash
# Test pull after enabling
docker pull myregistry.azurecr.io/myapp:v1
```

## Reference Documentation

- **Investigation reports**: `agent-investigation-reports/acr-features/dedicated-data-endpoints/`
- **MS Learn docs**: `azure-management-docs/articles/container-registry/container-registry-dedicated-data-endpoints.md`

# ACR Geo-Replication Skill

This skill provides comprehensive knowledge about Azure Container Registry geo-replication.

## When to Use This Skill

Use this skill when answering questions about:
- Multi-region registry replication
- Regional endpoints
- High availability
- Global distribution

## Overview

Geo-replication enables a single Premium registry to serve multiple regions with local endpoints.

**Requirements:** Premium SKU only

## Quick Setup

```bash
# Add replication
az acr replication create \
  --registry myregistry \
  --location westeurope

# List replications
az acr replication list --registry myregistry -o table

# Show endpoints
az acr show-endpoints --name myregistry
```

## Architecture

```
┌─────────────────────────────────────────────────┐
│              myregistry.azurecr.io              │
├─────────────────┬───────────────────────────────┤
│   East US       │      West Europe              │
│   (Home)        │      (Replica)                │
├─────────────────┼───────────────────────────────┤
│ myregistry.     │ myregistry.                   │
│ eastus.data.    │ westeurope.data.              │
│ azurecr.io      │ azurecr.io                    │
└─────────────────┴───────────────────────────────┘
```

## Benefits

| Benefit | Description |
|---------|-------------|
| **Network-Close Access** | Clients pull from nearest region |
| **High Availability** | Survives regional outages |
| **Single Management** | One registry, multiple regions |
| **Traffic Manager** | Automatic routing to nearest replica |

## Regional Endpoints

Each replica gets its own data endpoint:

| Region | Data Endpoint |
|--------|---------------|
| East US | `myregistry.eastus.data.azurecr.io` |
| West Europe | `myregistry.westeurope.data.azurecr.io` |
| Southeast Asia | `myregistry.southeastasia.data.azurecr.io` |

## Replication Commands

```bash
# Create replica with zone redundancy
az acr replication create \
  --registry myregistry \
  --location westeurope \
  --zone-redundancy enabled

# Update replica
az acr replication update \
  --registry myregistry \
  --location westeurope \
  --tags environment=production

# Delete replica
az acr replication delete \
  --registry myregistry \
  --location westeurope
```

## Consistency Model

Geo-replication uses **eventual consistency**:
- Pushes sync to all replicas asynchronously
- Typical sync time: seconds to minutes
- Each region operates independently

## Webhooks per Region

Configure webhooks to trigger on specific regions:

```bash
az acr webhook create \
  --registry myregistry \
  --name notify-westeurope \
  --uri https://webhook.example.com \
  --location westeurope \
  --actions push
```

## ARM/Bicep Deployment

```bicep
resource registry 'Microsoft.ContainerRegistry/registries@2023-07-01' = {
  name: 'myregistry'
  location: 'eastus'
  sku: { name: 'Premium' }
}

resource replication 'Microsoft.ContainerRegistry/registries/replications@2023-07-01' = {
  parent: registry
  name: 'westeurope'
  location: 'westeurope'
  properties: {
    zoneRedundancy: 'Enabled'
  }
}
```

## Best Practices

1. **Region Selection**: Choose regions close to your users/workloads
2. **Zone Redundancy**: Enable for high availability within regions
3. **Private Endpoints**: Create in each replica region for private access
4. **Webhooks**: Configure region-specific for deployment triggers
5. **Monitoring**: Set up alerts per region

## Compatibility

| Feature | Compatible |
|---------|------------|
| Zone Redundancy | ✅ |
| Private Endpoints | ✅ (per region) |
| Customer-Managed Keys | ✅ |
| Soft Delete | ❌ |

## Troubleshooting

```bash
# Check replication status
az acr replication show \
  --registry myregistry \
  --location westeurope \
  --query provisioningState

# Check sync status
az acr replication list --registry myregistry -o table
```

## Reference Documentation

- **Investigation reports**: `agent-investigation-reports/acr-features/geo-replication/`
- **MS Learn docs**: `azure-management-docs/articles/container-registry/container-registry-geo-replication.md`

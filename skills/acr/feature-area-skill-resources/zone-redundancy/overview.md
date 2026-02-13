# ACR Zone Redundancy Skill

This skill provides comprehensive knowledge about Azure Container Registry zone redundancy.

## When to Use This Skill

Use this skill when answering questions about:
- Availability zones
- High availability configuration
- Zone-redundant storage

## Overview

Zone redundancy distributes registry resources across availability zones within a region for high availability.

**Key Update (2026):** Zone redundancy is now enabled by default for ALL SKUs in supported regions at no additional cost!

## Feature Availability

| SKU | Zone Redundancy | Configuration |
|-----|-----------------|---------------|
| Basic | ✅ (default) | Automatic |
| Standard | ✅ (default) | Automatic |
| Premium | ✅ (default) | Configurable |

## Quick Setup

```bash
# Create zone-redundant registry
az acr create \
  --name myregistry \
  --resource-group myRG \
  --sku Premium \
  --location eastus \
  --zone-redundancy enabled

# Check status
az acr show --name myregistry --query zoneRedundancy
```

## Architecture

Zone redundancy ensures:
- Registry metadata replicated across zones
- Storage uses Zone-Redundant Storage (ZRS)
- Automatic failover within region

```
┌─────────────────────────────────────────┐
│           East US Region                │
├─────────────┬─────────────┬─────────────┤
│   Zone 1    │   Zone 2    │   Zone 3    │
│   Registry  │   Registry  │   Registry  │
│   Replica   │   Replica   │   Replica   │
└─────────────┴─────────────┴─────────────┘
```

## With Geo-Replication

Combine zone redundancy with geo-replication:

```bash
# Create zone-redundant replica
az acr replication create \
  --registry myregistry \
  --location westeurope \
  --zone-redundancy enabled
```

## Supported Regions

Zone redundancy is available in regions that support availability zones. Check current support:
```bash
az account list-locations --query "[?availabilityZoneMappings != null].name"
```

Common regions: East US, West Europe, Southeast Asia, Japan East, etc.

## Limitations

| Feature | Zone Redundancy Compatible |
|---------|---------------------------|
| Geo-Replication | ✅ |
| Private Endpoints | ✅ |
| Customer-Managed Keys | ✅ |
| Soft Delete Policy | ❌ |
| Artifact Cache | ❌ |

## Verification

```bash
# Check zone redundancy status
az acr show --name myregistry \
  --query "{zoneRedundancy:zoneRedundancy,location:location}"

# Check replica zone redundancy
az acr replication show \
  --registry myregistry \
  --location westeurope \
  --query zoneRedundancy
```

## Migration

Existing registries in supported regions automatically benefit from zone redundancy for new data. For explicit configuration:

```bash
# Update existing Premium registry
az acr update \
  --name myregistry \
  --zone-redundancy enabled
```

## Reference Documentation

- **Investigation reports**: `agent-investigation-reports/acr-features/zone-redundancy/`
- **MS Learn docs**: `azure-management-docs/articles/container-registry/zone-redundancy.md`

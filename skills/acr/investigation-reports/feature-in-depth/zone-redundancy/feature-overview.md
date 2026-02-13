# ACR Zone Redundancy - Feature Overview

## Summary

Zone redundancy is a high-availability feature in Azure Container Registry (ACR) that replicates registry data across multiple availability zones within an Azure region. This provides resilience against zone-level failures and ensures continuous availability of container images.

**Key Update (2026):** Zone redundancy is now enabled by default for ALL Azure Container Registries in regions that support availability zones. This applies to all SKUs (Basic, Standard, and Premium) and has been rolled out to both new and existing registries at no additional cost.

## What is Zone Redundancy?

Zone redundancy uses Azure [availability zones](/azure/reliability/availability-zones-overview) to replicate your registry to a minimum of three separate zones in each enabled region. This ensures:

- **High Availability**: Registry remains operational even if one availability zone experiences an outage
- **Data Durability**: Registry data is replicated across physically separated datacenters
- **Automatic Failover**: No manual intervention required during zone failures

## Key Benefits

1. **Enhanced Resilience**: Protects against zone-level infrastructure failures
2. **Automatic Protection**: Now enabled by default in supported regions
3. **No Additional Cost**: Zone redundancy comes at no extra charge
4. **Transparent to Users**: Works seamlessly with existing workflows
5. **Combined with Geo-Replication**: Can be used alongside geo-replication for multi-region resilience

## Feature Availability by SKU

| Feature | Basic | Standard | Premium |
|---------|-------|----------|---------|
| Zone Redundancy (default) | Yes* | Yes* | Yes* |
| Explicit Zone Redundancy Configuration | No | No | Yes |
| Geo-Replicated Zone Redundancy | No | No | Yes |

*Zone redundancy is now automatically enabled for all SKUs in supported regions.

## Important Notes

> **IMPORTANT**: The Azure portal and other tooling might not yet reflect the zone redundancy update accurately. The `zoneRedundancy` property in your registry's configuration might still show as `false`, but zone redundancy is active for all registries in supported regions.

- Zone redundancy was previously a Premium-only feature requiring explicit configuration
- The feature has been expanded to all SKUs with automatic enablement
- For Premium registries using geo-replication, all replicas in supported regions are also zone redundant by default

## Feature Limitations

Zone redundancy does not support:
- Registries configured with the **soft delete policy** (preview)
- The `zoneRedundancy` property display may not accurately reflect actual status

## Relationship to Other Features

### Geo-Replication
- Geo-replication replicates across Azure regions
- Zone redundancy replicates within a single region
- Combining both provides maximum resilience

### Artifact Cache
- Artifact cache uses features like availability zone support for higher availability
- Cache registries benefit from zone redundancy in supported regions

### Storage
- Registry data is stored in Azure Storage
- Zone redundancy ensures storage is replicated across zones

## Sources

- `/Users/johnsonshi/repos/acr-skills/submodules/azure-management-docs/articles/container-registry/zone-redundancy.md`
- `/Users/johnsonshi/repos/acr-skills/submodules/azure-management-docs/articles/container-registry/container-registry-storage.md`
- `/Users/johnsonshi/repos/acr-skills/submodules/azure-management-docs/articles/container-registry/container-registry-geo-replication.md`
- `/Users/johnsonshi/repos/acr-skills/submodules/acr/README.md`

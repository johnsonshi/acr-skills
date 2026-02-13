# ACR Geo-Replication Feature Overview

## Summary

Azure Container Registry (ACR) Geo-Replication is a Premium tier feature that enables a single container registry to serve multiple Azure regions through automatic content synchronization. This feature provides network-close registry access, reduced latency, improved reliability, and simplified management for multi-region deployments.

## Key Benefits

### 1. Single Registry, Multiple Regions
- Manage one registry and image name across all regions
- Push to a single registry endpoint; Azure automatically replicates content to configured regions
- Eliminate the need to maintain separate registries per region

### 2. Network-Close Performance
- Pull images from the nearest regional replica
- Reduced latency for container deployments
- Azure Traffic Manager routes requests to the network-closest replicated registry

### 3. Cost Efficiency
- Lower data transfer costs by pulling from local/nearby replicas
- Eliminate network egress fees between regions
- Pay Premium registry fees per region (no additional transfer charges)

### 4. High Availability
- Registry remains available if a regional outage occurs
- Data plane operations (push/pull) continue from remaining regions
- SLAs apply independently to each geo-replicated region

### 5. Simplified Management
- Single configuration for image deployments across regions
- Centralized access management
- Unified image URLs for all regional deployments

## Premium SKU Requirement

Geo-replication is **exclusively available** in the Premium service tier of Azure Container Registry.

| SKU | Geo-Replication Support |
|-----|------------------------|
| Basic | No |
| Standard | No |
| **Premium** | **Yes** |

To upgrade a registry: `az acr update --name myregistry --sku Premium`

## Feature Availability

### Supported Regions
- All Azure public cloud regions with availability zones support optimal geo-replication
- Gray regions on the Azure portal map indicate regions not yet available for replication

### Regional Data Endpoints
Each geo-replicated region gets its own data endpoint:
- Pattern: `<registry-name>.<region>.data.azurecr.io`
- Example: `contoso.eastus.data.azurecr.io`

## Use Case Example: Contoso

**Before Geo-Replication:**
- US-based registry in West US
- Separate registry in West Europe
- Required pushing to multiple registries:
  ```bash
  docker push contoso.azurecr.io/public/products/web:1.2
  docker push contosowesteu.azurecr.io/public/products/web:1.2
  ```
- East US, West US, and Canada Central clusters pulled from West US (incurring egress fees)
- Different image URL configurations per region

**After Geo-Replication:**
- Single registry: `contoso.azurecr.io`
- Replicas in East US, West US, Canada Central, and West Europe
- Single push command:
  ```bash
  docker push contoso.azurecr.io/public/products/web:1.2
  ```
- Each region pulls from nearest replica
- Unified image URL: `contoso.azurecr.io/public/products/web:1.2`

## Key Considerations

### Eventual Consistency Model
- Images and tags synchronize across regions with eventual consistency
- Larger images take longer to replicate than smaller ones
- After pushing to the nearest region, ACR replicates manifests and layers to other regions

### Webhooks for Event Tracking
- Configure regional webhooks to track push events per replica
- Set up workflows that respond to replication completion events
- Monitor push events as they complete across geo-replicated regions

### Home Region Importance
- The "home region" is where the registry was created
- If the home region becomes unavailable:
  - Data plane operations (push/pull) continue from replicas
  - Management operations may be unavailable (configuring network rules, managing replicas)

### Integration with Other Features

| Feature | Compatibility |
|---------|--------------|
| Zone Redundancy | Fully compatible - enhances regional reliability |
| Private Link | Compatible - dedicated data endpoints enabled automatically |
| Customer-Managed Keys | Compatible - requires key vault failover planning |
| Soft Delete Policy | **Not compatible** with geo-replication |
| Webhooks | Supports regional webhook configuration |

## Permissions Required

To create or delete replications, you need:
- `Microsoft.ContainerRegistry/registries/write` - Create a replication
- `Microsoft.ContainerRegistry/registries/replications/write` - Delete a replication

## Sources

- `/Users/johnsonshi/repos/acr-skills/submodules/azure-management-docs/articles/container-registry/container-registry-geo-replication.md`
- `/Users/johnsonshi/repos/acr-skills/submodules/azure-management-docs/articles/container-registry/container-registry-skus.md`
- `/Users/johnsonshi/repos/acr-skills/submodules/azure-management-docs/articles/container-registry/container-registry-best-practices.md`
- `/Users/johnsonshi/repos/acr-skills/submodules/azure-management-docs/articles/container-registry/container-registry-storage.md`

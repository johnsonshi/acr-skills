# ACR Zone Redundancy - Architecture

## Overview

Zone redundancy in Azure Container Registry leverages Azure Availability Zones to provide high availability and resilience within a single Azure region. The architecture ensures that registry data and services remain accessible even during zone-level failures.

## Azure Availability Zones Foundation

### What are Availability Zones?

Availability zones are physically separate datacenters within an Azure region. Each zone has:

- Independent power supply
- Independent cooling systems
- Independent networking
- High-speed, low-latency connections between zones

Zone-redundant services automatically replicate data and applications across these zones.

## ACR Zone Redundancy Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Azure Region (e.g., East US)                  │
│                                                                      │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐      │
│  │  Availability   │  │  Availability   │  │  Availability   │      │
│  │    Zone 1       │  │    Zone 2       │  │    Zone 3       │      │
│  │                 │  │                 │  │                 │      │
│  │ ┌─────────────┐ │  │ ┌─────────────┐ │  │ ┌─────────────┐ │      │
│  │ │ Registry    │ │  │ │ Registry    │ │  │ │ Registry    │ │      │
│  │ │ Replica     │ │  │ │ Replica     │ │  │ │ Replica     │ │      │
│  │ └─────────────┘ │  │ └─────────────┘ │  │ └─────────────┘ │      │
│  │                 │  │                 │  │                 │      │
│  │ ┌─────────────┐ │  │ ┌─────────────┐ │  │ ┌─────────────┐ │      │
│  │ │ Storage     │ │  │ │ Storage     │ │  │ │ Storage     │ │      │
│  │ │ (ZRS)       │ │  │ │ (ZRS)       │ │  │ │ (ZRS)       │ │      │
│  │ └─────────────┘ │  │ └─────────────┘ │  │ └─────────────┘ │      │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘      │
│                                                                      │
│              ▲                  ▲                  ▲                  │
│              │                  │                  │                  │
│              └──────────────────┼──────────────────┘                  │
│                                 │                                     │
│                    ┌────────────────────────┐                         │
│                    │    Load Balancer       │                         │
│                    │    (Traffic Manager)   │                         │
│                    └────────────────────────┘                         │
│                                 ▲                                     │
└─────────────────────────────────│─────────────────────────────────────┘
                                  │
                            Client Requests
```

## Component Details

### 1. Registry Data Replication

Zone-redundant registries store data across minimum three availability zones:

- **Container Images**: Manifests and layers are replicated across zones
- **Metadata**: Repository information, tags, and configuration
- **Artifacts**: OCI artifacts, Helm charts, and signatures

### 2. Storage Layer (Zone-Redundant Storage - ZRS)

Azure Container Registry uses Azure Storage for image data:

- **Zone-Redundant Storage (ZRS)**: Synchronously replicates data across three AZs
- **Encryption at Rest**: All data encrypted automatically
- **Durability**: Designed for 99.9999999999% (12 9s) durability

### 3. Service Endpoints

ACR provides multiple endpoint types that benefit from zone redundancy:

- **Login Server**: `<registry-name>.azurecr.io`
- **Data Endpoints**: For pulling/pushing image layers
- **Dedicated Data Endpoints** (Premium): Per-region endpoints for firewall configurations

## Failover Behavior

### Automatic Zone Failover

When a zone experiences an outage:

1. **Detection**: Azure automatically detects zone unavailability
2. **Traffic Rerouting**: Requests are routed to healthy zones
3. **Transparent**: No changes required in client configurations
4. **Recovery**: Service restores automatically when zone returns

### Data Consistency

- **Synchronous Replication**: Data written to all zones before acknowledgment
- **No Data Loss**: Committed writes preserved during zone failures
- **Read Consistency**: All zones serve identical data

## Architecture with Geo-Replication

For Premium registries, zone redundancy can be combined with geo-replication:

```
┌─────────────────────────────────┐     ┌─────────────────────────────────┐
│         Region 1 (Home)         │     │         Region 2 (Replica)      │
│                                 │     │                                 │
│  ┌───────┐ ┌───────┐ ┌───────┐  │     │  ┌───────┐ ┌───────┐ ┌───────┐  │
│  │ Zone1 │ │ Zone2 │ │ Zone3 │  │     │  │ Zone1 │ │ Zone2 │ │ Zone3 │  │
│  │  ZR   │ │  ZR   │ │  ZR   │  │     │  │  ZR   │ │  ZR   │ │  ZR   │  │
│  └───────┘ └───────┘ └───────┘  │     │  └───────┘ └───────┘ └───────┘  │
│                                 │     │                                 │
│       Zone Redundant            │     │       Zone Redundant            │
└─────────────────────────────────┘     └─────────────────────────────────┘
              │                                        │
              └───────────── Geo-Replication ──────────┘
```

Benefits of combined architecture:
- **Regional Resilience**: Survives entire region outages
- **Zone Resilience**: Survives zone-level failures within each region
- **Low Latency**: Network-close access from multiple regions
- **Disaster Recovery**: Complete data redundancy across geographies

## Private Endpoint Integration

Zone redundancy works with private endpoints:

- Dedicated data endpoints enabled automatically per geo-replicated region
- Private endpoint traffic benefits from zone-redundant backend
- Firewall rules can be configured per-region with zone-redundant support

## Performance Characteristics

### Latency

- Zone-to-zone latency: Sub-millisecond within same region
- Read operations: Served from nearest healthy zone
- Write operations: Synchronous to all zones (slight overhead)

### Throughput

- Aggregate throughput increases with zone redundancy
- Premium SKU provides highest concurrent operations
- No performance penalty for zone redundancy

## API Resource Model

Zone redundancy is represented in the ARM/Bicep resource model:

```json
{
  "type": "Microsoft.ContainerRegistry/registries",
  "apiVersion": "2025-04-01",
  "properties": {
    "zoneRedundancy": "Enabled"
  }
}
```

For geo-replicated replicas:

```json
{
  "type": "Microsoft.ContainerRegistry/registries/replications",
  "apiVersion": "2025-04-01",
  "properties": {
    "zoneRedundancy": "Enabled"
  }
}
```

## Sources

- `/Users/johnsonshi/repos/acr-skills/submodules/azure-management-docs/articles/container-registry/zone-redundancy.md`
- `/Users/johnsonshi/repos/acr-skills/submodules/azure-management-docs/articles/container-registry/container-registry-geo-replication.md`
- `/Users/johnsonshi/repos/acr-skills/submodules/azure-management-docs/articles/container-registry/container-registry-storage.md`
- `/Users/johnsonshi/repos/acr-skills/submodules/azure-management-docs/articles/container-registry/container-registry-dedicated-data-endpoints.md`

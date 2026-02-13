# ACR Geo-Replication Architecture

## Overview

Azure Container Registry geo-replication architecture enables a single registry to serve multiple Azure regions through automated content synchronization. The architecture leverages Azure Traffic Manager for intelligent request routing and dedicated data endpoints for each regional replica.

## Core Architectural Components

### 1. Registry Structure

```
                              +---------------------------+
                              |     Azure Container       |
                              |     Registry (Premium)    |
                              +---------------------------+
                              |    Login Server:          |
                              |    myregistry.azurecr.io  |
                              +-------------+-------------+
                                            |
                        +-------------------+-------------------+
                        |                   |                   |
              +---------v---------+ +-------v-------+ +--------v--------+
              |   Home Region     | | Replica       | | Replica         |
              |   (West US)       | | (East US)     | | (West Europe)   |
              +---------+---------+ +-------+-------+ +--------+--------+
                        |                   |                   |
              +---------v---------+ +-------v-------+ +--------v--------+
              | Data Endpoint:    | | Data Endpoint:| | Data Endpoint:  |
              | myregistry.       | | myregistry.   | | myregistry.     |
              | westus.data.      | | eastus.data.  | | westeurope.data.|
              | azurecr.io        | | azurecr.io    | | azurecr.io      |
              +-------------------+ +---------------+ +-----------------+
```

### 2. Traffic Manager Integration

Azure Traffic Manager is central to geo-replication functionality:

- **DNS-Based Routing**: Routes registry requests to the network-closest replicated registry
- **Automatic Selection**: For every push or pull operation, Traffic Manager sends requests to the nearest location
- **Health Monitoring**: Monitors replica health and routes around failures

```
Client Request: docker pull myregistry.azurecr.io/image:tag
        |
        v
+------------------+
| Azure Traffic    |
| Manager          |
+--------+---------+
         |
         | (DNS Resolution to nearest replica)
         |
         v
+--------+---------+
| Nearest Registry |
| Replica          |
+------------------+
```

### 3. Data Endpoints Architecture

Each geo-replicated region has dedicated data endpoints:

| Component | Purpose | Format |
|-----------|---------|--------|
| Login Server | Authentication, content discovery | `<registry>.azurecr.io` |
| Data Endpoint | Blob storage, layer content | `<registry>.<region>.data.azurecr.io` |

**Why Dedicated Data Endpoints?**
- Tightly scoped firewall rules
- Minimize data exfiltration risks
- Regional blob serving for performance

### 4. Storage Architecture

```
+-------------------+      +-------------------+      +-------------------+
|   Region: West US |      |  Region: East US  |      | Region: West EU   |
+-------------------+      +-------------------+      +-------------------+
| Azure Storage     |<---->| Azure Storage     |<---->| Azure Storage     |
| (Managed by ACR)  |      | (Managed by ACR)  |      | (Managed by ACR)  |
|                   |      |                   |      |                   |
| - Image layers    |      | - Image layers    |      | - Image layers    |
| - Manifests       |      | - Manifests       |      | - Manifests       |
| - Artifacts       |      | - Artifacts       |      | - Artifacts       |
+-------------------+      +-------------------+      +-------------------+
```

**Key Storage Features:**
- Managed storage accounts by ACR service
- Load balancing across storage
- Multiple copies for concurrent content delivery
- Encryption at rest (automatic)

## Replication Flow

### Push Operation Flow

```
1. Client: docker push myregistry.azurecr.io/image:tag
        |
        v
2. Traffic Manager resolves to nearest replica
        |
        v
3. Image pushed to nearest replica storage
        |
        v
4. ACR initiates background replication
        |
   +----+----+----+
   |    |    |    |
   v    v    v    v
5. Layers replicated to all configured regions
        |
        v
6. Regional webhooks fire on completion
```

### Pull Operation Flow

```
1. Client: docker pull myregistry.azurecr.io/image:tag
        |
        v
2. Traffic Manager DNS resolution
        |
        v
3. Request routed to nearest replica
        |
        v
4. Image layers served from local storage
        |
        v
5. Client receives image (minimal network hops)
```

## Consistency Model

### Eventual Consistency

Geo-replication operates on an **eventual consistency** model:

1. **Initial Push**: Content lands in the nearest region first
2. **Manifest Sync**: ACR replicates manifests to other regions
3. **Layer Sync**: Layers are synchronized across all replicas
4. **Completion**: All regions have consistent content

### Timing Factors

| Factor | Impact |
|--------|--------|
| Image size | Larger images take longer to replicate |
| Number of layers | More layers = longer sync time |
| Network conditions | Cross-region bandwidth affects speed |
| Regional distance | Farther regions may take longer |

## High Availability Architecture

### Region Independence

Each geo-replicated region operates independently:

```
+------------------+
| Home Region      |  <-- Registry management operations
| (West US)        |  <-- Data plane operations
+------------------+

+------------------+     +------------------+
| Replica Region   |     | Replica Region   |
| (East US)        |     | (West Europe)    |
| - Independent    |     | - Independent    |
| - Own SLA        |     | - Own SLA        |
| - Own storage    |     | - Own storage    |
+------------------+     +------------------+
```

### Failover Behavior

| Scenario | Impact |
|----------|--------|
| Replica region outage | Traffic routes to remaining regions |
| Home region outage | Data plane continues; management operations paused |
| Multiple region outage | Remaining regions serve traffic |

## Zone Redundancy Integration

For maximum resilience, combine geo-replication with zone redundancy:

```
+---------------------------------------------+
|              Geo-Replicated Registry        |
+---------------------------------------------+
|                                             |
|  +---------------+    +------------------+  |
|  | Region: EUS   |    | Region: WEU      |  |
|  +---------------+    +------------------+  |
|  | Zone 1        |    | Zone 1           |  |
|  | Zone 2        |    | Zone 2           |  |
|  | Zone 3        |    | Zone 3           |  |
|  +---------------+    +------------------+  |
|                                             |
|  Cross-Region Redundancy + Zone Redundancy  |
+---------------------------------------------+
```

**Benefits:**
- Zone redundancy: Within-region resilience (availability zones)
- Geo-replication: Cross-region resilience
- Combined: Maximum availability guarantee

## Private Link Architecture

When combined with Private Link:

```
+--------------------+       +--------------------+
| Virtual Network A  |       | Virtual Network B  |
| (East US)          |       | (West Europe)      |
+--------------------+       +--------------------+
| Private Endpoint   |       | Private Endpoint   |
| 10.0.0.7           |       | 10.1.0.7           |
+--------+-----------+       +---------+----------+
         |                             |
         v                             v
+--------+-----------+       +---------+----------+
| Data Endpoint:     |       | Data Endpoint:     |
| myregistry.eastus. |       | myregistry.        |
| data.azurecr.io    |       | westeurope.data.   |
|                    |       | azurecr.io         |
+--------------------+       +--------------------+
```

**Key Points:**
- Private endpoints required in each region
- Each replica needs DNS records
- Dedicated data endpoints enabled automatically with Private Link

## Customer-Managed Key Architecture

For CMK-encrypted geo-replicated registries:

```
+------------------------+
| Registry (Geo-Replicated)
+------------------------+
           |
           v
+------------------------+
| Azure Key Vault        |
| - Primary region       |
| - CMK for encryption   |
+------------------------+
           |
           | (Key vault failover required
           |  for disaster recovery)
           v
+------------------------+
| Backup Key Vault       |
| - Secondary region     |
+------------------------+
```

**Planning Considerations:**
- Review key vault failover and redundancy guidance
- Ensure key availability across regions
- Plan for key rotation across geo-replicated registries

## Sources

- `/Users/johnsonshi/repos/acr-skills/submodules/azure-management-docs/articles/container-registry/container-registry-geo-replication.md`
- `/Users/johnsonshi/repos/acr-skills/submodules/azure-management-docs/articles/container-registry/container-registry-dedicated-data-endpoints.md`
- `/Users/johnsonshi/repos/acr-skills/submodules/azure-management-docs/articles/container-registry/zone-redundancy.md`
- `/Users/johnsonshi/repos/acr-skills/submodules/azure-management-docs/articles/container-registry/container-registry-private-link.md`
- `/Users/johnsonshi/repos/acr-skills/submodules/azure-management-docs/articles/container-registry/container-registry-storage.md`

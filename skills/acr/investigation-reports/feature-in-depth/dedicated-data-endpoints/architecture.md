# ACR Dedicated Data Endpoints - Architecture

## High-Level Architecture

### Registry Managed Storage Architecture

Azure Container Registry is a multi-tenant service where the registry service manages data endpoint storage accounts. This architecture provides:

- **Load Balancing**: Distribution of requests across storage infrastructure
- **Content Splitting**: Efficient handling of contentious content
- **Multiple Copies**: Higher concurrent content delivery capacity
- **Multi-Region Support**: Foundation for geo-replication

```
                                   Azure Container Registry Service
                                              |
                    +-------------------------+-------------------------+
                    |                         |                         |
              Registry API              Data Endpoint             Management
         (contoso.azurecr.io)    (contoso.<region>.data.azurecr.io)   Plane
                    |                         |
                    v                         v
             Authentication            Managed Storage
              & Discovery               (Blob Layer)
```

## Endpoint Architecture

### Standard Data Flow (Without Dedicated Endpoints)

```
+--------+     1. Login Request      +----------------+
| Client |-------------------------->| Registry API   |
|        |<--------------------------| *.azurecr.io   |
+--------+     2. Auth Token         +----------------+
    |
    |          3. Manifest Request   +----------------+
    +----------------------------->  | Registry API   |
    <-----------------------------   | *.azurecr.io   |
    |          4. Manifest + Layer   +----------------+
    |             References
    |
    |          5. Layer Download     +-------------------+
    +----------------------------->  | Azure Blob Storage|
    <-----------------------------   | *.blob.core.      |
               6. Layer Data         | windows.net       |
                                     +-------------------+
```

**Security Concern**: Wildcard `*.blob.core.windows.net` allows access to ANY Azure storage.

### Dedicated Data Endpoint Flow

```
+--------+     1. Login Request      +----------------+
| Client |-------------------------->| Registry API   |
|        |<--------------------------| contoso.       |
+--------+     2. Auth Token         | azurecr.io     |
    |                                +----------------+
    |
    |          3. Manifest Request   +----------------+
    +----------------------------->  | Registry API   |
    <-----------------------------   | contoso.       |
    |          4. Manifest + Refs    | azurecr.io     |
    |             (to data endpoint) +----------------+
    |
    |          5. Layer Download     +-------------------+
    +----------------------------->  | Dedicated Data    |
    <-----------------------------   | Endpoint          |
               6. Layer Data         | contoso.eastus.   |
               (via temp link)       | data.azurecr.io   |
                                     +-------------------+
```

**Security Benefit**: Firewall rules can be specific to `contoso.*.data.azurecr.io`.

## Regional Architecture

### Single Region Deployment

```
                    +------------------+
                    |   Registry API   |
                    | contoso.azurecr.io|
                    +--------+---------+
                             |
                    +--------v---------+
                    | Data Endpoint    |
                    | contoso.eastus.  |
                    | data.azurecr.io  |
                    +------------------+
                             |
                    +--------v---------+
                    | Managed Storage  |
                    | (East US)        |
                    +------------------+
```

### Geo-Replicated Registry Architecture

```
                         +------------------+
                         |   Registry API   |
                         | contoso.azurecr.io|
                         +--------+---------+
                                  |
           +----------------------+----------------------+
           |                      |                      |
  +--------v---------+  +--------v---------+  +--------v---------+
  | Data Endpoint    |  | Data Endpoint    |  | Data Endpoint    |
  | contoso.eastus.  |  | contoso.westus.  |  | contoso.westeu.  |
  | data.azurecr.io  |  | data.azurecr.io  |  | data.azurecr.io  |
  +--------+---------+  +--------+---------+  +--------+---------+
           |                      |                      |
  +--------v---------+  +--------v---------+  +--------v---------+
  | Managed Storage  |  | Managed Storage  |  | Managed Storage  |
  | (East US)        |  | (West US)        |  | (West Europe)    |
  +------------------+  +------------------+  +------------------+
```

## Private Link Integration Architecture

When Private Link is configured, the architecture extends to include private endpoints:

```
+------------------------+
|    Virtual Network     |
|  +------------------+  |
|  | Private Endpoint |  |
|  | (Registry)       |  |
|  +--------+---------+  |
|           |            |
|  +--------v---------+  |         +------------------+
|  | Private IP       |  |         |   Registry API   |
|  | 10.0.0.7         +----------->| contoso.azurecr.io|
|  +------------------+  |         +------------------+
|                        |
|  +------------------+  |
|  | Private Endpoint |  |
|  | (Data)           |  |
|  +--------+---------+  |
|           |            |         +-------------------+
|  +--------v---------+  |         | Data Endpoint     |
|  | Private IP       +----------->| contoso.eastus.   |
|  | 10.0.0.8         |  |         | data.azurecr.io   |
|  +------------------+  |         +-------------------+
+------------------------+
```

Key Points:
- Registry endpoint gets its own private IP
- Each regional data endpoint gets a separate private IP
- Geo-replicated registries have additional IPs per replica region
- DNS records map FQDNs to private IPs

## Connected Registry Integration

Connected registries require dedicated data endpoints for synchronization:

```
                    Cloud (Azure)
+--------------------------------------------------+
|                                                  |
|  +----------------+    +--------------------+    |
|  | Registry API   |    | Data Endpoint      |    |
|  | contoso.       |    | contoso.eastus.    |    |
|  | azurecr.io     |    | data.azurecr.io    |    |
|  +-------+--------+    +---------+----------+    |
|          |                       |               |
+--------------------------------------------------+
           |                       |
           | Authentication        | Sync Messages
           |                       | & Content
           |                       |
+----------v-----------------------v---------------+
|                On-Premises                       |
|                                                  |
|  +------------------------------------------+   |
|  |          Connected Registry              |   |
|  |  +----------------+  +----------------+  |   |
|  |  | Local Registry |  | Local Storage  |  |   |
|  |  | Endpoint       |  |                |  |   |
|  |  +----------------+  +----------------+  |   |
|  +------------------------------------------+   |
|                        |                        |
+------------------------+------------------------+
                         |
              +----------v----------+
              |  On-Premises        |
              |  Container Clients  |
              +---------------------+
```

## DNS Architecture

### Standard (Public) DNS Resolution

```
contoso.azurecr.io
    --> CNAME --> contoso.privatelink.azurecr.io
    --> CNAME --> xxxx.xx.azcr.io
    --> CNAME --> xxxx-xxx-reg.trafficmanager.net
    --> A Record --> Public IP (e.g., 20.45.122.144)

contoso.eastus.data.azurecr.io
    --> A Record --> Public IP
```

### Private DNS Zone Resolution

```
contoso.azurecr.io
    --> CNAME --> contoso.privatelink.azurecr.io
    --> A Record --> Private IP (e.g., 10.0.0.7)

contoso.eastus.data.azurecr.io
    --> A Record --> Private IP (e.g., 10.0.0.8)
```

## Temporary Download Link Architecture

```
1. Client requests layer
   GET /v2/repo/blobs/sha256:xxx

2. Registry returns redirect to temporary URL
   302 Found
   Location: https://contoso.eastus.data.azurecr.io/blob/sha256:xxx?se=2024-01-01T12:20:00Z&sig=xxx

3. Client follows redirect to data endpoint
   GET https://contoso.eastus.data.azurecr.io/blob/sha256:xxx?se=...&sig=...

4. Data endpoint serves content
   200 OK
   [binary layer data]

Note: Temporary links expire after 20 minutes
```

## Network Traffic Flow Summary

| Operation | Without Dedicated Endpoints | With Dedicated Endpoints |
|-----------|---------------------------|-------------------------|
| Login | `*.azurecr.io` | `<registry>.azurecr.io` |
| Manifest | `*.azurecr.io` | `<registry>.azurecr.io` |
| Layer Pull | `*.blob.core.windows.net` | `<registry>.<region>.data.azurecr.io` |
| Layer Push | `*.blob.core.windows.net` | `<registry>.<region>.data.azurecr.io` |

## Sources

- MS Learn: `/articles/container-registry/container-registry-dedicated-data-endpoints.md`
- MS Learn: `/articles/container-registry/container-registry-private-link.md`
- MS Learn: `/articles/container-registry/container-registry-geo-replication.md`
- MS Learn: `/articles/container-registry/intro-connected-registry.md`
- ACR Repo: `/docs/preview/connected-registry/overview-connected-registry-access.md`

# ACR Artifact Cache - Architecture

## Overview

The ACR Artifact Cache feature provides a pull-through caching mechanism that sits between your Azure resources and upstream container registries. This document describes the architectural components and how they work together.

## Architecture Diagram

```
                                    +---------------------------+
                                    |    Upstream Registries    |
                                    |---------------------------|
                                    | - Docker Hub              |
                                    | - MCR                     |
                                    | - GitHub Container Reg    |
                                    | - Google Artifact Reg     |
                                    | - Quay                    |
                                    | - AWS ECR Public          |
                                    | - registry.k8s.io         |
                                    +-------------+-------------+
                                                  |
                                                  | (Authenticated/
                                                  |  Unauthenticated)
                                                  v
+-------------------+           +------------------------------------------+
|                   |           |         Azure Container Registry         |
|  Azure Key Vault  |<--------->|------------------------------------------|
|-------------------|  Secrets  |                                          |
| - Username Secret |           |  +----------------------------------+    |
| - Password Secret |           |  |        Artifact Cache Engine     |    |
+-------------------+           |  |----------------------------------|    |
                                |  | - Cache Rules (max 1000)        |    |
                                |  | - Credential Sets               |    |
                                |  | - Wildcard Matching             |    |
                                |  +----------------+-----------------+    |
                                |                   |                      |
                                |                   v                      |
                                |  +----------------------------------+    |
                                |  |        Cached Repositories        |    |
                                |  |----------------------------------|    |
                                |  | - Local copies of images         |    |
                                |  | - Geo-replicated (if enabled)   |    |
                                |  | - Zone redundant (if enabled)   |    |
                                |  +----------------+-----------------+    |
                                +---------------------|--------------------+
                                                      |
                    +----------------+----------------+----------------+
                    |                |                |                |
                    v                v                v                v
            +-------------+  +-------------+  +-------------+  +-------------+
            |    AKS      |  |    ACI      |  |   VMs/Dev   |  |  CI/CD      |
            |   Cluster   |  |  Instances  |  |   Machines  |  |  Pipelines  |
            +-------------+  +-------------+  +-------------+  +-------------+
```

## Core Components

### 1. Cache Rules Engine

The cache rules engine manages the mapping between source (upstream) repositories and target (ACR) repositories.

**Components:**
- **Rule Store**: Persists up to 1,000 cache rules per registry
- **Rule Matcher**: Evaluates incoming pull requests against defined rules
- **Wildcard Processor**: Handles pattern matching for wildcard rules

**Rule Resolution Order:**
1. Static (fixed) rules take precedence over wildcard rules
2. More specific wildcard rules are matched before broader ones
3. First matching rule is used

### 2. Credential Management System

Manages authentication credentials for upstream registries using Azure Key Vault integration.

**Components:**
- **Credential Sets**: Named collections of authentication credentials
- **Key Vault Integration**: Secure storage and retrieval of secrets
- **System-Assigned Managed Identity**: Each credential set gets its own identity

**Authentication Flow:**
```
1. Credential Set Created
        |
        v
2. System Identity Assigned (Principal ID)
        |
        v
3. Identity granted Key Vault access (Key Vault Secrets User role)
        |
        v
4. ACR retrieves secrets during pull operations
        |
        v
5. ACR authenticates to upstream registry
```

### 3. Image Caching Layer

Handles the actual storage and retrieval of cached container images.

**Characteristics:**
- Images stored in ACR's managed storage
- Benefits from geo-replication if enabled
- Zone redundancy support for high availability
- Standard ACR image storage model applies

### 4. Pull Request Handler

Intercepts pull requests and determines whether to serve from cache or fetch from upstream.

**Decision Flow:**
```
Client Pull Request
        |
        v
+-------+-------+
|   Cache Rule  |
|    Exists?    |
+-------+-------+
    |       |
   Yes      No
    |       |
    v       v
+---+---+  Standard ACR
| Image |  Pull (or 404)
|Cached?|
+---+---+
 |     |
Yes    No
 |     |
 v     v
Serve  Fetch from
Local  Upstream
       & Cache
```

## Data Flow

### Initial Pull (Cache Miss)

```
1. Client requests: docker pull myacr.azurecr.io/library/nginx:latest

2. ACR checks cache rules:
   - Finds rule: library/nginx -> docker.io/library/nginx

3. ACR checks local cache:
   - Image not found locally

4. ACR authenticates to upstream (if credentials configured):
   - Retrieves secrets from Key Vault
   - Authenticates to Docker Hub

5. ACR pulls from upstream:
   - Fetches nginx:latest from docker.io

6. ACR stores image locally:
   - Image cached in ACR storage

7. ACR serves image to client:
   - Client receives image
```

### Subsequent Pull (Cache Hit)

```
1. Client requests: docker pull myacr.azurecr.io/library/nginx:latest

2. ACR checks cache rules:
   - Finds rule: library/nginx -> docker.io/library/nginx

3. ACR checks local cache:
   - Image found locally

4. ACR serves image directly:
   - No upstream call needed
   - Fast, reliable delivery
```

## Integration Points

### Azure Key Vault Integration

```
+-------------------+
|  Azure Key Vault  |
+-------------------+
         |
         | RBAC: Key Vault Secrets User
         |
         v
+-------------------+
|  Credential Set   |
|  System Identity  |
+-------------------+
         |
         | Microsoft.KeyVault/vaults/secrets/getSecret/action
         |
         v
+-------------------+
| Username Secret   |
| Password Secret   |
+-------------------+
```

**Required Permission:** `Microsoft.KeyVault/vaults/secrets/getSecret/action`

**Recommended Role:** Key Vault Secrets User

### Geo-Replication Integration

When artifact cache is combined with geo-replication:

```
                    +-------------------+
                    | Upstream Registry |
                    +--------+----------+
                             |
                             v
                    +--------+----------+
                    |    Primary ACR    |
                    |    (Home Region)  |
                    +--------+----------+
                             |
          +------------------+------------------+
          |                  |                  |
          v                  v                  v
   +------+------+    +------+------+    +------+------+
   |   Replica   |    |   Replica   |    |   Replica   |
   |  (Region A) |    |  (Region B) |    |  (Region C) |
   +-------------+    +-------------+    +-------------+
```

- Images cached once are automatically replicated
- Clients pull from nearest replica
- Reduces latency and improves reliability

### Private Network Integration

```
+-------------------+     Private Endpoint      +-------------------+
|  Virtual Network  |<------------------------->|        ACR        |
|-------------------|                           |-------------------|
| - AKS Cluster     |                           | - Artifact Cache  |
| - VMs             |                           | - Private IP      |
| - CI/CD Agents    |                           +-------------------+
+-------------------+                                    |
                                                         | Outbound
                                                         v
                                                +-------------------+
                                                | Upstream Registry |
                                                +-------------------+
```

## Security Architecture

### Credential Security

1. **No credentials stored in ACR** - All secrets in Key Vault
2. **Managed Identity access** - No shared credentials
3. **RBAC-controlled access** - Fine-grained permissions
4. **Audit logging** - All access logged

### Network Security

1. **Private endpoints** - Restrict inbound access
2. **Firewall rules** - Control source IPs
3. **Service endpoints** - VNet integration
4. **Outbound calls** - ACR to upstream only

### Identity Security

```
+-------------------+
|  Credential Set   |
|-------------------|
| - System Identity |
| - Principal ID    |
| - Resource ID     |
+--------+----------+
         |
         | Assigned Permissions
         v
+-------------------+
|   Azure Key Vault |
|-------------------|
| - Secrets access  |
| - Limited scope   |
+-------------------+
```

## Scalability Considerations

### Cache Rule Limits
- Maximum: 1,000 cache rules per registry
- Use wildcards to reduce rule count
- Plan namespace strategy for efficient rule usage

### Storage Scaling
- Cached images consume ACR storage
- Storage limits apply per SKU
- Geo-replicated storage multiplies usage

### Performance Scaling
- Pull performance based on SKU tier
- Premium SKU for highest throughput
- Geo-replication for distributed load

## High Availability Architecture

```
+-------------------+
|    Availability   |
|       Zone 1      |
+--------+----------+
         |
         +----------+-------------------+
                    |                   |
         +----------+----------+   +----+----+
         |    Availability     |   | Storage |
         |       Zone 2        |   |  Zone   |
         +----------+----------+   +----+----+
                    |                   |
         +----------+----------+   +----+----+
         |    Availability     |   | Storage |
         |       Zone 3        |   |  Zone   |
         +---------------------+   +---------+
```

- Zone redundancy available in Premium SKU
- Automatic failover between zones
- Storage replicated across zones

## References

- [ACR Architecture Overview](https://learn.microsoft.com/azure/container-registry/container-registry-concepts)
- [Azure Key Vault Integration](https://learn.microsoft.com/azure/key-vault/)
- [Geo-Replication Architecture](https://learn.microsoft.com/azure/container-registry/container-registry-geo-replication)
- [Private Link Architecture](https://learn.microsoft.com/azure/container-registry/container-registry-private-link)

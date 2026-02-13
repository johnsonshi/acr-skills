# Azure Container Registry - Limits and Quotas

## Overview

Azure Container Registry enforces various limits and quotas based on the service tier. These limits ensure fair resource allocation and optimal performance across all users. Some limits can be increased by contacting Azure Support.

## Storage Limits

### Included Storage vs Maximum Capacity

All ACR tiers support the same **maximum storage capacity of 40 TiB**. The difference between tiers is the "included storage" - the amount covered by the base tier price.

| Tier | Included Storage | Max Capacity |
|------|------------------|--------------|
| Basic | 10 GiB | 40 TiB |
| Standard | 100 GiB | 40 TiB |
| Premium | 500 GiB | 40 TiB |

### Storage Characteristics
- Additional storage beyond included amount is charged per-GB
- Storage usage includes all repositories, images, layers, and artifacts
- For geo-replicated registries, storage is counted in the home region
- Multiply by number of replicas for total storage consumption
- Storage usage may not reflect recent operations immediately (monitor via metrics)

### Storage Best Practices
- Periodically delete unused repositories, tags, and images
- Use retention policies for untagged manifests (Premium)
- Use soft delete policy for accident recovery
- Monitor storage consumption via Azure Portal or CLI

## Resource Limits

### Webhooks

| Tier | Maximum Webhooks |
|------|------------------|
| Basic | 10 |
| Standard | 100 |
| Premium | 500 |

### Network Rules (Premium Only)

| Resource | Limit |
|----------|-------|
| IP Access Rules | 200 |
| Virtual Network Rules | 100 |
| Private Endpoints | 200 |

### Tokens and Scope Maps

| Resource | Limit (All Tiers) |
|----------|-------------------|
| Tokens | 20,000 |
| Scope Maps | 20,000 |
| Repository Permissions per Scope Map | 500 |

### Connected Registry (Premium Only)

| Resource | Limit |
|----------|-------|
| Connected Registries per Registry | 50 |
| Clients per Connected Registry | 50 |
| Sync Token | 1 per connected registry |
| Client Tokens | As needed (within token limit) |

### Connected Registry Sync Limits

**Continuous Sync Mode:**
| Setting | Minimum | Maximum |
|---------|---------|---------|
| Message TTL | 1 day | 90 days |

**Scheduled Sync Mode:**
| Setting | Minimum | Maximum |
|---------|---------|---------|
| Sync Window | 1 hour | 7 days |

## Performance Limits

### API Concurrency and Throughput

Performance varies by tier, with Premium providing the highest capacity:

| Tier | API Concurrency | Bandwidth Throughput | Concurrent Operations |
|------|-----------------|----------------------|-----------------------|
| Basic | Lower | Lower | Limited |
| Standard | Moderate | Moderate | Moderate |
| Premium | Highest | Highest | High |

### Factors Affecting Performance

**Image/Artifact Factors:**
- Number and size of image layers
- Layer reuse across images
- Additional API calls per operation
- Scale of concurrent deployments

**Client Environment Factors:**
- Docker daemon/Podman configuration
- Container runtime settings (containerd, CRI-O)
- Cluster configuration and data plane settings

**Network Factors:**
- Network bandwidth and latency
- Firewall rules and proxy settings
- Geographic distance to registry or nearest replica

### Throttling

During high request volumes, you may encounter:
- HTTP 429 `Too many requests` errors
- Slow bandwidth throughput

**Mitigation Strategies:**
1. Implement retry logic with exponential backoff
2. Reduce concurrent request rate
3. Space out large-scale deployments
4. Upgrade to a higher SKU
5. Contact Azure Support for limit increases

## ACR Tasks Limits

### Agent Pool Limits (Premium Only)

| Resource | Default Quota |
|----------|---------------|
| Total vCPU (Standard Pools) | 16 per registry |
| Total vCPU (Isolated Pools) | 0 per registry |

**Note**: Contact Azure Support to increase quotas.

### Agent Pool Instance Limits

| Tier | CPU | Memory (GB) |
|------|-----|-------------|
| S1 | 2 | 3 |
| S2 | 4 | 8 |
| S3 | 8 | 16 |
| I6 | 64 | 216 |

### Task Execution
- Task logs are stored for later retrieval
- Build contexts can be from GitHub, Azure DevOps, local filesystem, or remote tarball
- Personal access tokens required for private repository triggers

## Repository and Artifact Limits

### General Limits
- No hard limit on number of repositories
- No hard limit on number of images
- No hard limit on number of tags
- No hard limit on number of layers

**Performance Considerations:**
- High numbers of repositories and tags can affect registry performance
- Regular maintenance recommended (delete unused content)
- Consider using retention policies

### Soft Delete Policy (All Tiers)

| Setting | Range |
|---------|-------|
| Retention Period | 1-90 days |
| Default Retention | 7 days |

### Retention Policy - Untagged Manifests (Premium Only, Preview)

| Setting | Range |
|---------|-------|
| Retention Period | 0-365 days |
| Default Retention | 7 days |

**Note**: Setting to 0 removes untagged manifests immediately.

## Geo-replication Limits (Premium Only)

| Resource | Limit |
|----------|-------|
| Maximum Replicas | Configurable (contact support for high limits) |
| Regions | Most Azure regions supported |

### Considerations
- Each replica incurs Premium registry fees
- Storage usage shown for home region
- Multiply by replicas for total storage

## Private Link Limits (Premium Only)

| Resource | Limit |
|----------|-------|
| Private Endpoints | 200 per registry |
| Data Endpoints per Region | 1 per geo-replicated region |

## Customer-Managed Keys (Premium Only)

| Resource | Requirement |
|----------|-------------|
| Azure Key Vault | Required |
| Managed Identity | User-assigned required |
| Key Permissions | get, unwrapKey, wrapKey |

## Content Trust Limits (Premium Only)

- Repository-scoped tokens cannot push/pull signed images
- Root key loss results in permanent loss of access to signed tags
- DCT being deprecated (May 31, 2026 - no new enablement; March 31, 2028 - complete removal)

## Request Limit Increases

The following limits can be increased by contacting Azure Support:

1. **Storage Limits** - Request additional storage beyond tier defaults
2. **Private Endpoint Limits** - Increase beyond 200 if needed
3. **API Throttling** - Request higher throughput limits
4. **Agent Pool vCPU Quota** - Increase standard or isolated pool quotas
5. **Geo-replication Regions** - Add replicas in additional regions

### How to Request Increases

1. Open a support ticket at https://azure.microsoft.com/support/create-ticket/
2. Provide registry name and subscription
3. Specify the limit increase needed
4. Justify the business requirement

## Monitoring Usage

### Azure Portal
- Registry Overview page shows current consumption vs. limits
- Storage usage displayed
- Webhook count
- Geo-replication count
- Private endpoint count
- IP access rule count
- VNet rule count

### Azure CLI
```bash
az acr show-usage --resource-group myResourceGroup --name myregistry --output table
```

### Sample Output
```
NAME                        LIMIT         CURRENT VALUE    UNIT
--------------------------  ------------  ---------------  ------
Size                        536870912000  215629144        Bytes
Webhooks                    500           1                Count
Geo-replications            -1            3                Count
IPRules                     100           1                Count
VNetRules                   100           0                Count
PrivateEndpointConnections  10            0                Count
```

### Azure Monitor Metrics
- Monitor `StorageUsed` metric for up-to-date storage data
- Set up alerts for approaching limits
- Use diagnostic logs for detailed analysis

## Related Documentation

- [Azure Container Registry SKU Features and Limits](container-registry-skus.md)
- [Show Registry Usage](container-registry-skus.md#show-registry-usage)
- [Troubleshoot Registry Performance](container-registry-troubleshoot-performance.md)
- [Azure Support](https://azure.microsoft.com/support/create-ticket/)

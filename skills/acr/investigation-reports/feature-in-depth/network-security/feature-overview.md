# ACR Network Security - Feature Overview

## Executive Summary

Azure Container Registry (ACR) provides a comprehensive suite of network security features to protect container registries from unauthorized access. These features enable organizations to implement defense-in-depth strategies, controlling both inbound and outbound network traffic to their registries.

## Network Security Architecture

ACR network security operates on multiple layers:

```
                    +---------------------------+
                    |     Internet/Public       |
                    +------------+--------------+
                                 |
                    +------------v--------------+
                    |    Public IP Rules        |
                    |   (Firewall/IP Filtering) |
                    +------------+--------------+
                                 |
          +----------------------+----------------------+
          |                      |                      |
+---------v--------+  +---------v---------+  +---------v---------+
|  Service Tags    |  | Service Endpoints |  |  Private Link     |
| (NSG Outbound)   |  |  (VNet Inbound)   |  | (Private Endpoint)|
+------------------+  +-------------------+  +-------------------+
          |                      |                      |
          +----------------------+----------------------+
                                 |
                    +------------v--------------+
                    |   Trusted Services        |
                    |   (Azure Service Bypass)  |
                    +------------+--------------+
                                 |
                    +------------v--------------+
                    |   Azure Container         |
                    |   Registry (ACR)          |
                    +---------------------------+
```

## Core Network Security Features

### 1. Public IP Network Rules (Firewall Rules)

**Purpose**: Control which public IP addresses can access the registry.

**Key Capabilities**:
- Configure IP allowlists using CIDR notation
- Maximum 200 IP access rules per registry
- Change default action to Deny (block all) or Allow (permit all)
- Available in Premium tier only

**Documentation Source**: `/submodules/azure-management-docs/articles/container-registry/container-registry-access-selected-networks.md`

### 2. VNet Service Endpoints

**Purpose**: Restrict registry access to specific Azure Virtual Network subnets.

**Key Capabilities**:
- Secure public IP to only traffic from VNet subnets
- Optimal routing over Azure backbone network
- Identity of VNet and subnet transmitted with requests
- Maximum 100 VNet rules per Premium registry
- Preview feature with limitations

**Documentation Source**: `/submodules/azure-management-docs/articles/container-registry/container-registry-vnet.md`

### 3. Azure Private Link (Private Endpoints)

**Purpose**: Provide private connectivity to registry from within a VNet using private IPs.

**Key Capabilities**:
- Traffic traverses Microsoft backbone network
- Eliminates public internet exposure
- Supports on-premises access via ExpressRoute or VPN
- Maximum 200 private endpoints per Premium registry
- Integrates with Private DNS Zones

**Documentation Source**: `/submodules/azure-management-docs/articles/container-registry/container-registry-private-link.md`

### 4. Azure Service Tags

**Purpose**: Simplify NSG rules for ACR traffic using service tag groups.

**Key Capabilities**:
- `AzureContainerRegistry` service tag for registry endpoints
- Regional variants (e.g., `AzureContainerRegistry.EastUS`)
- Automatic IP range updates managed by Microsoft
- Used for image import, webhooks, and ACR Tasks

**Documentation Source**: `/submodules/azure-management-docs/articles/container-registry/container-registry-service-tag.md`

### 5. Trusted Azure Services

**Purpose**: Allow specific Azure services to bypass network restrictions.

**Key Capabilities**:
- Enabled by default on new registries
- Supports Azure Container Instances, Microsoft Defender, Machine Learning
- Requires managed identity configuration for some services
- Works with Private Link and firewall rules (not service endpoints)

**Documentation Source**: `/submodules/azure-management-docs/articles/container-registry/allow-access-trusted-services.md`

### 6. Dedicated Data Endpoints

**Purpose**: Mitigate data exfiltration risks with registry-specific storage endpoints.

**Key Capabilities**:
- Regional pattern: `<registry>.<region>.data.azurecr.io`
- Tightly scoped firewall rules
- Eliminates need for wildcard `*.blob.core.windows.net` rules
- Premium tier only

**Documentation Source**: `/submodules/azure-management-docs/articles/container-registry/container-registry-dedicated-data-endpoints.md`

## SKU Requirements

| Feature | Basic | Standard | Premium |
|---------|-------|----------|---------|
| Public IP Rules | No | No | Yes (200 max) |
| VNet Service Endpoints | No | No | Yes (100 max) |
| Private Link | No | No | Yes (200 max) |
| Service Tags | Yes | Yes | Yes |
| Trusted Services | Yes | Yes | Yes |
| Dedicated Data Endpoints | No | No | Yes |

## Network Access Decision Matrix

| Scenario | Recommended Approach |
|----------|---------------------|
| Internet-accessible with IP restrictions | Public IP Rules (Firewall) |
| Access from Azure VNet only | Private Link (recommended) or Service Endpoints |
| Hybrid cloud with on-premises | Private Link + ExpressRoute/VPN |
| Allow specific Azure services | Trusted Services setting |
| Outbound rules from VNet | Service Tags in NSG |
| Prevent data exfiltration | Dedicated Data Endpoints + Export Policy |

## Registry Endpoints

ACR uses two primary endpoint types:

1. **REST API Endpoint** (Login Server)
   - Format: `<registry>.azurecr.io`
   - Used for authentication and content discovery
   - Port 443 (HTTPS)

2. **Data Endpoint** (Storage)
   - Default: `*.blob.core.windows.net`
   - Dedicated: `<registry>.<region>.data.azurecr.io`
   - Used for image layer transfers
   - Port 443 (HTTPS)

## Security Best Practices

1. **Use Private Link** - Most secure option for VNet scenarios
2. **Enable Dedicated Data Endpoints** - Prevents data exfiltration
3. **Disable Public Access** - When only private access is needed
4. **Use Service Tags** - Simplify NSG management
5. **Configure Trusted Services** - Only when needed for Azure integrations
6. **Monitor Network Traffic** - Use Azure Monitor and Network Watcher
7. **Implement Export Policy** - Disable artifact export for sensitive registries

## Related Documentation

- [Firewall Rules Configuration](./firewall-rules.md)
- [Service Endpoints Setup](./service-endpoints.md)
- [Service Tags Reference](./service-tags.md)
- [Trusted Services Configuration](./trusted-services.md)
- [CLI Commands Reference](./cli-commands.md)

## Source Files Referenced

- `/submodules/azure-management-docs/articles/container-registry/container-registry-access-selected-networks.md`
- `/submodules/azure-management-docs/articles/container-registry/container-registry-firewall-access-rules.md`
- `/submodules/azure-management-docs/articles/container-registry/container-registry-vnet.md`
- `/submodules/azure-management-docs/articles/container-registry/container-registry-private-link.md`
- `/submodules/azure-management-docs/articles/container-registry/container-registry-service-tag.md`
- `/submodules/azure-management-docs/articles/container-registry/allow-access-trusted-services.md`
- `/submodules/azure-management-docs/articles/container-registry/container-registry-dedicated-data-endpoints.md`
- `/submodules/azure-management-docs/articles/container-registry/container-registry-skus.md`

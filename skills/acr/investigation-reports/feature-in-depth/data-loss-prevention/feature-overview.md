# Azure Container Registry Data Loss Prevention - Feature Overview

## Executive Summary

Azure Container Registry (ACR) provides comprehensive Data Loss Prevention (DLP) capabilities to prevent unauthorized exfiltration of container images and artifacts from a registry. The primary mechanism is the **export policy** feature, which disables the ability to export artifacts from a network-restricted Premium container registry.

## What is Data Loss Prevention in ACR?

Data Loss Prevention in ACR is a security feature that prevents registry users from maliciously or accidentally leaking container artifacts outside a virtual network. When enabled, it blocks attempts to:

1. **Import artifacts to another registry** - Prevents using `az acr import` to copy artifacts from the protected registry to another Azure container registry
2. **Create export pipelines** - Blocks creation of ACR Transfer export pipelines that would transfer artifacts to external storage accounts

## Key Concepts

### Export Policy Property

The `exportPolicy` property is the core mechanism for DLP in ACR:

- **API Version**: Introduced in `2021-06-01-preview`
- **SKU Requirement**: Premium container registries only
- **Default State**: `enabled` (exports allowed)
- **DLP State**: `disabled` (exports blocked)

### Relationship with Network Restrictions

Export policy works in conjunction with network security features:

| Feature | Purpose |
|---------|---------|
| Export Policy | Prevents artifact export via import/transfer |
| Public Network Access | Controls external access to registry endpoints |
| Private Endpoints | Enables access only from virtual networks |
| Dedicated Data Endpoints | Provides registry-specific data URLs for firewall rules |

## Architecture Overview

```
                                    PROTECTED REGISTRY
                                    (Export Policy Disabled)
                                    +-------------------+
                                    |                   |
    VIRTUAL NETWORK                 |   ACR Premium     |
    +---------------------------+   |                   |
    |                           |   | exportPolicy:     |
    |  Authorized Clients       |   |   disabled        |
    |  - Pull allowed           |   |                   |
    |  - Push allowed           |   | publicNetworkAccess:|
    |  - Import FROM: allowed   |   |   disabled        |
    |                           |   |                   |
    +---------------------------+   +-------------------+
            |                               |
            | Private Endpoint              |
            +-------------------------------+

    BLOCKED OPERATIONS:
    X Import TO another registry
    X Export pipeline creation
    X Any export outside the VNet
```

## Prerequisites for Using Export Policy

To disable artifact exports, you must meet these requirements:

1. **Premium SKU** - Export policy is only available on Premium container registries
2. **Private Endpoint** - Registry must be configured with a private endpoint
3. **Public Network Access Disabled** - The registry's `publicNetworkAccess` property must be set to `disabled`
4. **No Existing Export Pipelines** - All export pipelines must be deleted before disabling exports

## What Export Policy Does NOT Block

The export policy does not restrict:

- **Pull operations** - Authorized users within the virtual network can still pull artifacts
- **Push operations** - Authorized users can still push new artifacts
- **Import INTO the registry** - Importing images from external sources is still allowed
- **Docker client operations** - Standard `docker pull`, `docker push` commands work normally within the VNet
- **Other data-plane operations** - Reading manifests, listing tags, etc. all remain functional

## Related DLP Features

### Dedicated Data Endpoints

Dedicated data endpoints provide an additional layer of protection against data exfiltration:

- Creates registry-specific FQDNs for data access (e.g., `myregistry.eastus.data.azurecr.io`)
- Allows tightly scoped firewall rules instead of wildcard `*.blob.core.windows.net` rules
- Prevents attackers from writing to arbitrary storage accounts through firewall loopholes

### Private Link Integration

Private Link ensures:
- Registry endpoints resolve to private IP addresses within the VNet
- Traffic traverses the Microsoft backbone network
- No exposure to the public internet

## Use Cases

### Scenario 1: Highly Regulated Industries
Organizations in healthcare, finance, or government sectors with strict data sovereignty requirements can use export policy to ensure container images never leave their controlled network environment.

### Scenario 2: Intellectual Property Protection
Companies with proprietary containerized applications can prevent accidental or malicious exfiltration of their container images.

### Scenario 3: Compliance Requirements
Organizations needing to comply with regulations like GDPR, HIPAA, or SOC 2 can use export policy as part of their data protection controls.

## Limitations

1. **Premium SKU Required** - Not available on Basic or Standard tiers
2. **Network Restrictions Required** - Must disable public network access
3. **Private Endpoint Required** - Must have private endpoint configured
4. **No Granular Control** - Export policy is registry-wide; cannot be applied to specific repositories
5. **Doesn't Block Pull** - Authorized users can still pull images (audit logging recommended)

## Monitoring and Auditing

Since export policy doesn't prevent pull operations, Microsoft recommends:

- Configure **diagnostic settings** to monitor registry operations
- Use Azure Monitor to track all data-plane activities
- Set up alerts for suspicious access patterns
- Review audit logs regularly for compliance purposes

## Related Documentation

- [Export Policy Configuration](./export-policy.md)
- [Configuration Guide](./configuration.md)
- [CLI Commands Reference](./cli-commands.md)

## Source References

- Primary documentation: `submodules/azure-management-docs/articles/container-registry/data-loss-prevention.md`
- Dedicated data endpoints: `submodules/azure-management-docs/articles/container-registry/container-registry-dedicated-data-endpoints.md`
- Private link: `submodules/azure-management-docs/articles/container-registry/container-registry-private-link.md`
- Transfer/Pipelines: `submodules/azure-management-docs/articles/container-registry/container-registry-transfer-images.md`

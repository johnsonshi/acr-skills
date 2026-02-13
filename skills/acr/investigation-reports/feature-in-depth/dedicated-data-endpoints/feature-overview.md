# ACR Dedicated Data Endpoints - Feature Overview

## Summary

Azure Container Registry (ACR) Dedicated Data Endpoints is a Premium SKU feature that enables tightly scoped client firewall rules to specific registries, minimizing data exfiltration concerns. This feature provides dedicated, registry-specific endpoints for downloading container image layers and other OCI artifacts.

## Key Concepts

### What Are Dedicated Data Endpoints?

Dedicated data endpoints are fully qualified domain names (FQDNs) that represent registry-specific endpoints for data transfer operations. They replace the generic Azure blob storage endpoints (`*.blob.core.windows.net`) with registry-specific endpoints following the pattern:

```
<registry-name>.<region>.data.azurecr.io
```

### Two Types of Registry Endpoints

Pulling content from a registry involves two distinct endpoints:

1. **Registry REST API Endpoint (Login URL)**
   - Used for authentication and content discovery
   - Format: `<registry-name>.azurecr.io`
   - Handles REST requests, authentication, and layer negotiation
   - Example: `contoso.azurecr.io`

2. **Data Endpoint (Storage Endpoint)**
   - Serves blobs representing content layers
   - Without dedicated endpoints: `*.blob.core.windows.net`
   - With dedicated endpoints: `<registry-name>.<region>.data.azurecr.io`
   - Example: `contoso.eastus.data.azurecr.io`

## Service Tier Availability

| Feature | Basic | Standard | Premium |
|---------|-------|----------|---------|
| Dedicated Data Endpoints | No | No | **Yes** |

Dedicated data endpoints are only available in the **Premium** service tier.

## Benefits

### 1. Enhanced Security and Data Exfiltration Mitigation
- Eliminates need for wildcard firewall rules (`*.blob.core.windows.net`)
- Allows precise firewall rules scoped to specific registries
- Prevents bad actors from writing to unauthorized storage accounts
- Provides "secure knowledge" of what resources are accessible from each client

### 2. Simplified Firewall Configuration
- Instead of allowing all Azure blob storage, allow only your registry's data endpoints
- Reduces attack surface significantly
- Provides predictable, registry-specific URLs

### 3. Geo-Replication Support
- Data endpoints are created per region
- Geo-replicated registries get dedicated endpoints in each replica region
- Enables region-specific firewall rules

### 4. Private Link Integration
- When Private Link is enabled, dedicated data endpoints are enabled automatically
- Both registry and data endpoints become accessible via private IPs
- Public endpoints can be removed for complete network isolation

## How It Works

### Without Dedicated Data Endpoints

```
Client --> contoso.azurecr.io (authentication)
       --> *.blob.core.windows.net (layer download)
```

Risk: Wildcard allows access to ANY Azure blob storage, potential for data exfiltration.

### With Dedicated Data Endpoints

```
Client --> contoso.azurecr.io (authentication)
       --> contoso.eastus.data.azurecr.io (layer download)
```

Benefit: Firewall rules can be specific to the registry, blocking unauthorized storage access.

## Temporary Download Links

When dedicated data endpoints are enabled, ACR provides clients with temporary download links for image layers:
- Each layer download receives a short-lived URL
- Links are valid for **20 minutes**
- After expiration, clients request new links as needed
- This provides secure, time-limited access to content

## Use Cases

1. **Enterprise Security Compliance**: Organizations requiring strict data exfiltration controls
2. **On-Premises Client Access**: IoT devices, build agents behind firewalls
3. **Hybrid Cloud Deployments**: Clients accessing registry from on-premises networks
4. **Connected Registry Synchronization**: Required for connected registries to sync with cloud registry
5. **Custom Domain Scenarios**: Foundation for custom domain configurations

## Related Features

- **Azure Private Link**: Most secure option when VNet connectivity is available
- **Geo-Replication**: Regional data endpoints for each replica
- **Connected Registry**: Requires dedicated data endpoints for synchronization
- **Export Policy**: Additional data loss prevention control
- **Custom Domains**: Builds upon dedicated data endpoints

## Sources

- MS Learn: `/articles/container-registry/container-registry-dedicated-data-endpoints.md`
- MS Learn: `/articles/container-registry/container-registry-firewall-access-rules.md`
- ACR Repo: `/docs/custom-domain/README.md`
- ACR Repo: `README.md` (Security Links section)

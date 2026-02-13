# Azure Container Registry Private Endpoints - Feature Overview

## What Are Private Endpoints?

Private endpoints enable you to connect privately to an Azure Container Registry using Azure Private Link. By assigning virtual network private IP addresses to the registry endpoints, network traffic between clients on the virtual network and the registry's private endpoints traverses the virtual network and a private link on the Microsoft backbone network, eliminating exposure from the public internet.

## Key Benefits

### 1. Enhanced Security
- Network traffic stays entirely within the Microsoft backbone network
- Eliminates exposure to the public internet
- Registry endpoints are assigned private IP addresses within your VNet

### 2. Private Connectivity Options
- **Azure Virtual Network**: Access registry from VMs and services within the VNet
- **Azure ExpressRoute**: Private peering for on-premises connectivity
- **VPN Gateway**: Secure tunnel from on-premises networks

### 3. Seamless DNS Integration
- Configure DNS settings so the registry's public FQDN resolves to private IP addresses
- Clients continue to access the registry using its fully qualified domain name (e.g., `myregistry.azurecr.io`)
- No application code changes required

## Premium SKU Requirement

**Important**: Private endpoints are available **only** in the Premium container registry service tier.

| Feature | Basic | Standard | Premium |
|---------|-------|----------|---------|
| Private Endpoints | No | No | Yes |
| Maximum Private Endpoints | N/A | N/A | 200 |

## Private Endpoint Limits

- Maximum of **200 private endpoints** per Premium registry
- This limit can be increased by contacting Azure Support

## Registry Endpoints Involved

When you create a private endpoint for a container registry, two endpoints receive private IP addresses:

1. **Registry REST API endpoint**: `<registry-name>.azurecr.io`
   - Handles authentication and registry management operations

2. **Data endpoint**: `<registry-name>.<region>.data.azurecr.io`
   - Serves blob data for container image layers
   - One data endpoint per region (important for geo-replicated registries)

## Comparison with Service Endpoints

| Feature | Private Endpoint (Private Link) | Service Endpoint |
|---------|--------------------------------|------------------|
| SKU Required | Premium | Premium |
| Network Access | Private IP within VNet | Public IP routed through Azure backbone |
| On-premises Access | Yes (via ExpressRoute/VPN) | No |
| Cross-subscription Support | Yes | Limited |
| DNS Resolution | Private IP | Public IP |
| Status | Generally Available | Preview |
| Portal Configuration | Yes | No (CLI only) |

**Microsoft Recommendation**: Use private endpoints instead of service endpoints in most network scenarios. Private endpoints are accessible from within the virtual network using private IP addresses.

> **Important**: You cannot use both private link and service endpoint features configured from a virtual network in your container registry simultaneously.

## Related Features

### Dedicated Data Endpoints
- When private link is enabled, dedicated data endpoints are **automatically enabled**
- Provide tightly scoped access to registry storage
- Follow regional pattern: `<registry-name>.<region>.data.azurecr.io`

### Public Network Access
- Can be completely disabled when using private endpoints
- Ensures registry is only accessible through private connectivity
- Firewall rules don't apply to private endpoints

### Export Policy (Data Loss Prevention)
- Can disable artifact export from network-restricted registries
- Prevents data exfiltration via import operations or export pipelines
- Requires Premium SKU and private endpoint configuration

## Trusted Services Access

When a registry is configured with private endpoints, you can still allow access from trusted Azure services:

- **Azure Container Instances**: Deploy containers using managed identity
- **Microsoft Defender for Cloud**: Vulnerability scanning
- **Azure Machine Learning**: Deploy/train models with custom images
- **Azure Container Registry**: Import images between registries

Enable with:
```bash
az acr update --name myregistry --allow-trusted-services true
```

## Use Cases

1. **Enterprise Security Compliance**: Meet regulatory requirements for network isolation
2. **Hybrid Cloud Scenarios**: Access registry from on-premises via ExpressRoute/VPN
3. **Multi-Tenant Environments**: Isolate registry access per virtual network
4. **CI/CD Pipelines**: Secure access from build agents in private networks
5. **Kubernetes Deployments**: Private access from AKS clusters

## Key Considerations

- Private endpoint creation requires the `Microsoft.Network/privateEndpoints/privateLinkServiceProxies/write` permission
- DNS configuration is critical for proper functionality
- Geo-replicated registries require private endpoints in each replica region
- Some Azure services (App Service, Azure Container Instances without managed identity) may have limitations with network-restricted registries

## Sources

- [Azure Container Registry Private Link Documentation](https://aka.ms/acr/privatelink)
- [Azure Private Link Overview](/azure/private-link/private-link-overview)
- [ACR SKU Features and Limits](https://aka.ms/acr/tiers)

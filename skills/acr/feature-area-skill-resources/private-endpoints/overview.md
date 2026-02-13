# ACR Private Endpoints Skill

This skill provides comprehensive knowledge about Azure Container Registry private endpoints and Private Link configuration.

## When to Use This Skill

Use this skill when answering questions about:
- Private Link configuration for ACR
- Private endpoint setup and DNS
- Network isolation for registries
- ExpressRoute/VPN connectivity
- Eliminating public internet exposure

## Overview

Private endpoints enable private connectivity to ACR using Azure Private Link. Traffic stays on Microsoft's backbone network, eliminating public internet exposure.

**Requirements:**
- Premium SKU only
- Maximum 200 private endpoints per registry

## Key Benefits

| Benefit | Description |
|---------|-------------|
| Enhanced Security | Traffic stays on Microsoft backbone |
| Private IPs | Registry endpoints get VNet IPs |
| On-Premises Access | Works with ExpressRoute/VPN |
| Seamless DNS | Clients use same FQDN |

## Quick Setup

### Azure CLI
```bash
# Create private endpoint
az network private-endpoint create \
  --name myPrivateEndpoint \
  --resource-group myRG \
  --vnet-name myVNet \
  --subnet mySubnet \
  --private-connection-resource-id $(az acr show --name myregistry --query id -o tsv) \
  --group-id registry \
  --connection-name myConnection

# Create private DNS zone
az network private-dns zone create \
  --resource-group myRG \
  --name privatelink.azurecr.io

# Link DNS zone to VNet
az network private-dns link vnet create \
  --resource-group myRG \
  --zone-name privatelink.azurecr.io \
  --name myDNSLink \
  --virtual-network myVNet \
  --registration-enabled false
```

### Disable Public Access
```bash
az acr update --name myregistry --public-network-enabled false
```

## Registry Endpoints

When you create a private endpoint, two endpoints receive private IPs:

| Endpoint | Format | Purpose |
|----------|--------|---------|
| REST API | `myregistry.azurecr.io` | Authentication, management |
| Data | `myregistry.eastus.data.azurecr.io` | Image layer blobs |

## DNS Configuration

### Required DNS Records
```
# For single region
myregistry.azurecr.io → 10.0.0.5
myregistry.eastus.data.azurecr.io → 10.0.0.6

# For geo-replicated (add per region)
myregistry.westeurope.data.azurecr.io → 10.0.1.6
```

### DNS Verification
```bash
# Test resolution
nslookup myregistry.azurecr.io
dig myregistry.azurecr.io

# Should resolve to private IP (10.x.x.x)
```

## Private Endpoint vs Service Endpoint

| Feature | Private Endpoint | Service Endpoint |
|---------|------------------|------------------|
| Status | GA | Preview |
| IP Type | Private | Public (Azure routed) |
| On-Premises | Yes | No |
| Portal Config | Yes | CLI only |
| Recommendation | **Preferred** | Limited use |

> **Note:** Cannot use both simultaneously on the same registry.

## Trusted Services Access

Allow Azure services to access network-restricted registry:
```bash
az acr update --name myregistry --allow-trusted-services true
```

Supported services:
- Azure Container Instances (with managed identity)
- Microsoft Defender for Cloud
- Azure Machine Learning
- ACR (import operations)

## Geo-Replication Considerations

For geo-replicated registries:
1. Create private endpoints in each replica region
2. Add DNS records for each regional data endpoint
3. Configure DNS zone links in each VNet

## Troubleshooting

### Common Issues
| Issue | Cause | Solution |
|-------|-------|----------|
| DNS resolves to public IP | DNS not configured | Add private DNS zone |
| Auth succeeds, pull fails | Data endpoint issue | Check data endpoint DNS |
| 403 Forbidden | Missing permissions | Check RBAC, trusted services |
| ACR Tasks fail | No private access | Use agent pools or IP allowlist |

### Diagnostic Commands
```bash
# Check health
az acr check-health --name myregistry --vnet myVNet

# Test connectivity from VNet
curl -I https://myregistry.azurecr.io/v2/
```

## Reference Documentation

- **Investigation reports**: `agent-investigation-reports/acr-features/private-endpoints/`
- **MS Learn docs**: `azure-management-docs/articles/container-registry/container-registry-private-link.md`

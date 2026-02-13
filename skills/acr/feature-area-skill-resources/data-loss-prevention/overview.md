# ACR Data Loss Prevention Skill

This skill provides comprehensive knowledge about Azure Container Registry data loss prevention features.

## When to Use This Skill

Use this skill when answering questions about:
- Export policy
- Data exfiltration prevention
- Disabling artifact export

## Overview

Data Loss Prevention (DLP) prevents artifact exfiltration from network-restricted registries.

**Requirements:** Premium SKU + Private endpoint + Public network disabled

## Export Policy

### What It Blocks
| Operation | Blocked |
|-----------|---------|
| `az acr import` FROM this registry | ✅ |
| Export pipelines | ✅ |
| Pull from public internet | ✅ (if public disabled) |

### What It Allows
| Operation | Allowed |
|-----------|---------|
| Push to registry | ✅ |
| Pull from private endpoint | ✅ |
| `az acr import` INTO this registry | ✅ |

## Configuration

### Prerequisites
```bash
# 1. Ensure Premium SKU
az acr show --name myregistry --query sku.name

# 2. Create private endpoint
az network private-endpoint create ...

# 3. Disable public access
az acr update --name myregistry --public-network-enabled false
```

### Disable Export
```bash
# Check current status
az acr show --name myregistry --query policies.exportPolicy

# Disable export (enable DLP)
az acr update --name myregistry --allow-exports false

# Enable export (disable DLP)
az acr update --name myregistry --allow-exports true
```

## Verification

```bash
# Attempt import FROM registry (should fail if DLP enabled)
az acr import \
  --name otherregistry \
  --source myregistry.azurecr.io/myapp:v1 \
  --image myapp:v1
# Error: Export policy prevents this operation
```

## Complete DLP Setup

```bash
# 1. Create Premium registry
az acr create --name myregistry --resource-group myRG --sku Premium

# 2. Create private endpoint
az network private-endpoint create \
  --name myPrivateEndpoint \
  --resource-group myRG \
  --vnet-name myVNet \
  --subnet mySubnet \
  --private-connection-resource-id $(az acr show --name myregistry --query id -o tsv) \
  --group-id registry

# 3. Disable public access
az acr update --name myregistry --public-network-enabled false

# 4. Enable trusted services
az acr update --name myregistry --allow-trusted-services true

# 5. Disable export
az acr update --name myregistry --allow-exports false

# 6. Enable dedicated data endpoints
az acr update --name myregistry --data-endpoint-enabled true
```

## ARM Template

```json
{
  "type": "Microsoft.ContainerRegistry/registries",
  "apiVersion": "2023-07-01",
  "properties": {
    "publicNetworkAccess": "Disabled",
    "policies": {
      "exportPolicy": {
        "status": "disabled"
      }
    }
  }
}
```

## Related Features

| Feature | Purpose |
|---------|---------|
| Private endpoints | Private connectivity |
| Dedicated data endpoints | Granular firewall rules |
| Export policy | Block exports |
| Trusted services | Allow Azure services |

## Use Cases

1. **Compliance**: Prevent data leaving network boundary
2. **Security**: Protect proprietary images
3. **Regulatory**: Meet data residency requirements

## Reference Documentation

- **Investigation reports**: `agent-investigation-reports/acr-features/data-loss-prevention/`
- **MS Learn docs**: `azure-management-docs/articles/container-registry/data-loss-prevention.md`

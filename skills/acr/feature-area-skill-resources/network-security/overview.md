# ACR Network Security Skill

This skill provides comprehensive knowledge about Azure Container Registry network security features.

## When to Use This Skill

Use this skill when answering questions about:
- Firewall rules and IP restrictions
- VNet service endpoints
- Service tags
- Trusted services bypass
- Network access control

## Network Security Features

| Feature | SKU | Status |
|---------|-----|--------|
| Public IP Rules | All | GA |
| Service Endpoints | Premium | Preview |
| Private Endpoints | Premium | GA |
| Service Tags | All | GA |
| Trusted Services | Premium | GA |

## Firewall Rules (IP-Based Access)

### Allow Specific IPs
```bash
# Add IP rule
az acr network-rule add \
  --name myregistry \
  --ip-address 203.0.113.0/24

# List rules
az acr network-rule list --name myregistry

# Remove rule
az acr network-rule remove \
  --name myregistry \
  --ip-address 203.0.113.0/24
```

### Default Action
```bash
# Deny by default (Premium)
az acr update --name myregistry --default-action Deny

# Allow by default
az acr update --name myregistry --default-action Allow
```

**Limits:** Maximum 200 IP rules per Premium registry

## Service Endpoints (Preview)

> **Note:** Service endpoints are in preview with limitations. Private endpoints are recommended.

```bash
# Add VNet rule
az acr network-rule add \
  --name myregistry \
  --vnet-name myVNet \
  --subnet mySubnet

# Enable service endpoint on subnet
az network vnet subnet update \
  --name mySubnet \
  --vnet-name myVNet \
  --resource-group myRG \
  --service-endpoints Microsoft.ContainerRegistry
```

**Limitations:**
- CLI only (no portal)
- VM/AKS hosts only
- Max 100 VNet rules

## Service Tags

Use `AzureContainerRegistry` service tag in NSGs and Azure Firewall:

### NSG Rule
```bash
az network nsg rule create \
  --nsg-name myNSG \
  --name AllowACR \
  --priority 100 \
  --destination-service-tag AzureContainerRegistry \
  --destination-port-ranges 443 \
  --access Allow
```

### Regional Service Tags
- `AzureContainerRegistry.EastUS`
- `AzureContainerRegistry.WestEurope`

### Required for Import/Tasks
When importing images or running tasks, also allow:
- `AzureActiveDirectory`
- `Storage`

## Trusted Services

Allow Azure services to bypass network restrictions:

```bash
# Enable trusted services
az acr update --name myregistry --allow-trusted-services true
```

### Supported Services
| Service | Requirements |
|---------|-------------|
| Azure Container Instances | Managed identity + RBAC |
| Microsoft Defender for Cloud | None |
| Azure Machine Learning | Managed identity + RBAC |
| ACR (import operations) | None |

## Network Bypass Policy for Tasks

> **Breaking Change:** As of June 2025, `networkRuleBypassAllowedForTasks` defaults to `false`

```bash
# Check policy
az acr show --name myregistry --query networkRuleBypassAllowedForTasks

# Enable bypass (not recommended)
az acr update --name myregistry --allow-network-bypass-for-tasks true
```

**Alternative:** Use agent pools for VNet-integrated task execution.

## Dedicated Data Endpoints

Mitigate data exfiltration with regional data endpoints:

```bash
# Enable
az acr update --name myregistry --data-endpoint-enabled true
```

Creates endpoint: `myregistry.eastus.data.azurecr.io`

## Complete Network Lockdown

```bash
# 1. Create private endpoint
az network private-endpoint create ...

# 2. Disable public access
az acr update --name myregistry --public-network-enabled false

# 3. Enable trusted services
az acr update --name myregistry --allow-trusted-services true

# 4. Enable dedicated data endpoints
az acr update --name myregistry --data-endpoint-enabled true
```

## Troubleshooting

```bash
# Check network configuration
az acr show --name myregistry --query "{publicNetwork:publicNetworkAccess,defaultAction:networkRuleSet.defaultAction,ipRules:networkRuleSet.ipRules}"

# Health check
az acr check-health --name myregistry
```

## Reference Documentation

- **Investigation reports**: `agent-investigation-reports/acr-features/network-security/`
- **MS Learn docs**: `azure-management-docs/articles/container-registry/container-registry-access-selected-networks.md`

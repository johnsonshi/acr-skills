# Azure Container Registry - SKU Upgrade and Downgrade Path

## Overview

Azure Container Registry allows you to change between service tiers (SKUs) as your requirements evolve. You can upgrade to access more features and capacity, or downgrade to reduce costs when advanced features are no longer needed.

## Upgrade and Downgrade Matrix

| From | To | Supported | Considerations |
|------|-----|-----------|----------------|
| Basic | Standard | Yes | No restrictions |
| Basic | Premium | Yes | No restrictions |
| Standard | Basic | Yes | Storage must fit within Basic limits |
| Standard | Premium | Yes | No restrictions |
| Premium | Standard | Yes | Must remove Premium features first |
| Premium | Basic | Yes | Must remove Premium features first |

## Upgrading Your Registry

### Basic to Standard

**Process:**
1. No downtime during upgrade
2. All existing content preserved
3. Immediate access to Standard limits
4. No feature changes (same feature set)

**When to Upgrade:**
- Need more webhooks (Basic: 2, Standard: 10)
- Experiencing throughput limitations

**CLI Command:**
```bash
az acr update --name myContainerRegistry --sku Standard
```

**PowerShell Command:**
```powershell
Update-AzContainerRegistry -ResourceGroupName myResourceGroup -Name myContainerRegistry -Sku Standard
```

### Basic/Standard to Premium

**Process:**
1. No downtime during upgrade
2. All existing content preserved
3. Immediate access to Premium features
4. Premium limits take effect immediately

**When to Upgrade:**
- Need geo-replication for multi-region deployments
- Require private endpoints for network isolation
- Need customer-managed encryption keys
- Want to use connected registries for edge scenarios
- Need dedicated agent pools for ACR Tasks
- Require retention policies for untagged manifests
- Need IP/VNet network rules
- Want artifact streaming for AKS

**CLI Command:**
```bash
az acr update --name myContainerRegistry --sku Premium
```

**PowerShell Command:**
```powershell
Update-AzContainerRegistry -ResourceGroupName myResourceGroup -Name myContainerRegistry -Sku Premium
```

### Upgrade Best Practices

1. **Plan for Feature Enablement**: After upgrading to Premium, plan which features to enable
2. **Test in Non-production**: If possible, test the upgrade process in a non-production registry first
3. **Review New Limits**: Familiarize yourself with the new tier's limits
4. **Update Documentation**: Document the change for your team

## Downgrading Your Registry

### Premium to Standard or Basic

Downgrading from Premium requires removing all Premium-exclusive resources before the SKU change will succeed.

**Required Steps Before Downgrade:**

1. **Delete All Geo-replications**
   ```bash
   # List replications
   az acr replication list --registry myregistry --output table

   # Delete each replication (except home region)
   az acr replication delete --name eastus --registry myregistry
   ```

2. **Delete All Connected Registries**
   ```bash
   # List connected registries
   az acr connected-registry list --registry myregistry --output table

   # Delete each connected registry
   az acr connected-registry delete --name myconnectedreg --registry myregistry
   ```

3. **Remove All Private Endpoints**
   - Delete private endpoint connections from the registry
   - Remove private DNS zone records
   - Delete the virtual network link if no longer needed

4. **Remove IP and VNet Rules**
   ```bash
   # List network rules
   az acr network-rule list --name myregistry

   # Remove IP rules
   az acr network-rule remove --name myregistry --ip-address <ip-address>

   # Remove VNet rules
   az acr network-rule remove --name myregistry --subnet <subnet-id>
   ```

5. **Disable Dedicated Data Endpoints** (if enabled)
   ```bash
   az acr update --name myregistry --data-endpoint-enabled false
   ```

6. **Disable Retention Policy** (if enabled)
   ```bash
   az acr config retention update --registry myregistry --status disabled --type UntaggedManifests
   ```

7. **Disable Content Trust** (if enabled)
   - Go to registry in Azure portal
   - Under Policies, select Content Trust
   - Set Status to Disabled

8. **Remove Customer-Managed Key Encryption** (if enabled)
   - This may require recreating the registry
   - Contact Azure Support for guidance

9. **Delete Agent Pools** (if created)
   ```bash
   # List agent pools
   az acr agentpool list --registry myregistry --output table

   # Delete agent pools
   az acr agentpool delete --name myagentpool --registry myregistry
   ```

**After Removing Premium Features:**

```bash
# Downgrade to Standard
az acr update --name myContainerRegistry --sku Standard

# Or downgrade to Basic
az acr update --name myContainerRegistry --sku Basic
```

### Standard to Basic

**Required Steps:**
1. Reduce webhook count if exceeding Basic limit (100 -> 2)

> **Note**: All tiers support up to 40 TiB max storage capacity. The "included storage" differs by tier (Basic: 10 GiB, Standard: 100 GiB), but storage limits do not block downgrades—you'll simply pay overage charges for storage beyond the included amount.

**CLI Command:**
```bash
az acr update --name myContainerRegistry --sku Basic
```

### Downgrade Checklist

Before downgrading from Premium:

- [ ] All geo-replications deleted (except home region)
- [ ] All connected registries deleted
- [ ] All private endpoints removed
- [ ] All IP network rules removed
- [ ] All VNet rules removed
- [ ] Dedicated data endpoints disabled
- [ ] Retention policy disabled
- [ ] Content trust disabled
- [ ] Agent pools deleted
- [ ] Storage within target tier limits
- [ ] Webhook count within target tier limits

## Impact of SKU Changes

### No Downtime
- SKU changes do not cause registry downtime
- Push and pull operations continue normally during the change

### Feature Availability
- **Upgrade**: New features available immediately after upgrade completes
- **Downgrade**: Premium features disabled; must be removed before downgrade

### Storage
- **Upgrade**: Higher storage limits apply immediately
- **Downgrade**: Must ensure current usage fits within new limits

### Billing
- **Upgrade**: New tier pricing starts from the change
- **Downgrade**: Reduced pricing starts from the change

## Common Upgrade/Downgrade Scenarios

### Scenario 1: Scaling for Production

**Starting Point:** Basic tier for development
**Need:** Production deployment with moderate traffic

**Action:**
```bash
az acr update --name myContainerRegistry --sku Standard
```

**Benefits:**
- 100 GiB included storage (vs 10 GiB)
- 10 webhooks (vs 2)
- Higher throughput

### Scenario 2: Adding Multi-region Support

**Starting Point:** Standard tier, single region
**Need:** Deploy to multiple Azure regions

**Action:**
```bash
# Upgrade to Premium
az acr update --name myContainerRegistry --sku Premium

# Add geo-replication
az acr replication create --registry myContainerRegistry --location westeurope
az acr replication create --registry myContainerRegistry --location eastasia
```

### Scenario 3: Enabling Network Isolation

**Starting Point:** Standard tier, public access
**Need:** Private network access only

**Action:**
```bash
# Upgrade to Premium
az acr update --name myContainerRegistry --sku Premium

# Configure private endpoint
az network private-endpoint create \
    --name myPrivateEndpoint \
    --resource-group myResourceGroup \
    --vnet-name myVNet \
    --subnet mySubnet \
    --private-connection-resource-id $(az acr show --name myContainerRegistry --query 'id' -o tsv) \
    --group-ids registry \
    --connection-name myConnection

# Disable public access
az acr update --name myContainerRegistry --public-network-enabled false
```

### Scenario 4: Cost Reduction

**Starting Point:** Premium tier with no Premium features in use
**Need:** Reduce costs

**Pre-check:**
```bash
# Verify no geo-replications
az acr replication list --registry myContainerRegistry

# Verify no private endpoints
az acr private-endpoint-connection list --registry-name myContainerRegistry

# Check storage usage
az acr show-usage --name myContainerRegistry
```

**Action (if no Premium features):**
```bash
az acr update --name myContainerRegistry --sku Standard
```

### Scenario 5: Edge/IoT Deployment

**Starting Point:** Standard tier
**Need:** Connected registry for edge locations

**Action:**
```bash
# Upgrade to Premium
az acr update --name myContainerRegistry --sku Premium

# Create connected registry
az acr connected-registry create \
    --registry myContainerRegistry \
    --name myConnectedRegistry \
    --repository "myrepo/*"
```

## Troubleshooting SKU Changes

### Error: Storage Exceeds Limit

**Error Message:** Cannot downgrade because storage exceeds target tier limit

**Solution:**
1. Check current storage usage:
   ```bash
   az acr show-usage --name myContainerRegistry
   ```
2. Delete unused content to reduce storage
3. Wait for storage metrics to update (may take time)
4. Retry the downgrade

### Error: Feature Requires Premium

**Error Message:** Cannot downgrade while Premium features are enabled

**Solution:**
1. Identify Premium features in use:
   - Geo-replications
   - Connected registries
   - Private endpoints
   - Network rules
   - Retention policy
2. Remove each feature as described above
3. Retry the downgrade

### Error: Active Connected Registry

**Error Message:** Cannot delete connected registry while active

**Solution:**
1. Deactivate the connected registry deployment
2. Wait for status to change to Inactive
3. Delete the connected registry resource
4. Retry the downgrade

## Automation and Infrastructure as Code

### Terraform Example

```hcl
resource "azurerm_container_registry" "acr" {
  name                = "myContainerRegistry"
  resource_group_name = azurerm_resource_group.rg.name
  location            = azurerm_resource_group.rg.location
  sku                 = "Premium"  # Change this to upgrade/downgrade
  admin_enabled       = false
}
```

### Bicep Example

```bicep
resource containerRegistry 'Microsoft.ContainerRegistry/registries@2023-01-01-preview' = {
  name: 'myContainerRegistry'
  location: location
  sku: {
    name: 'Premium'  // Change to 'Standard' or 'Basic'
  }
  properties: {
    adminUserEnabled: false
  }
}
```

### ARM Template Example

```json
{
  "type": "Microsoft.ContainerRegistry/registries",
  "apiVersion": "2023-01-01-preview",
  "name": "myContainerRegistry",
  "location": "[resourceGroup().location]",
  "sku": {
    "name": "Premium"
  },
  "properties": {
    "adminUserEnabled": false
  }
}
```

## Related Documentation

- [Azure Container Registry SKU Features and Limits](container-registry-skus.md)
- [Change Registry SKU](container-registry-skus.md#change-registry-sku)
- [Geo-replication](container-registry-geo-replication.md)
- [Private Link](container-registry-private-link.md)
- [Connected Registry](intro-connected-registry.md)
- [Azure Container Registry Pricing](https://azure.microsoft.com/pricing/details/container-registry/)

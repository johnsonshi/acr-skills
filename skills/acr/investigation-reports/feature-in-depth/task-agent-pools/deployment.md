# ACR Task Agent Pools - Deployment Guide

## Prerequisites

### Service Requirements

1. **Azure CLI**: Version 2.3.1 or later
   ```bash
   az --version
   # If upgrade needed:
   # brew upgrade azure-cli (macOS)
   # apt-get update && apt-get install azure-cli (Linux)
   ```

2. **Premium Container Registry**: Agent pools require Premium tier
   ```bash
   # Check registry tier
   az acr show --name myregistry --query sku.name --output tsv
   # Should return: Premium
   ```

3. **Registry in Supported Region**: Must be in a preview region
   - West US 2, South Central US, East US 2, East US, Central US
   - West Europe, North Europe, Canada Central, East Asia
   - Switzerland North, Sweden Central
   - Jio India West, Jio India Central
   - USGov Arizona, USGov Texas, USGov Virginia

### For VNet Deployments

4. **Virtual Network and Subnet**: Existing VNet/subnet for agent pool placement
5. **Network Security Group (NSG)**: Configured with required outbound rules
6. **Service Endpoints** (recommended): For fine-grained network control

## Basic Deployment

### Step 1: Set Default Registry (Optional)

Simplify subsequent commands by setting a default registry:

```bash
az config set defaults.acr=myregistry
```

Or use `az configure`:
```bash
az configure --defaults acr=myregistry
```

### Step 2: Create Agent Pool

Create a basic agent pool with default settings (1 instance, S1 tier):

```bash
az acr agentpool create \
    --name myagentpool \
    --registry myregistry
```

Create with specific tier:

```bash
# S2 tier (4 vCPU, 8 GB memory)
az acr agentpool create \
    --registry myregistry \
    --name myagentpool \
    --tier S2
```

### Step 3: Verify Pool Creation

```bash
az acr agentpool show \
    --registry myregistry \
    --name myagentpool
```

> **Note**: Pool creation takes several minutes to complete.

## Scaling Configuration

### Scale Pool Instances

Increase capacity for parallel task execution:

```bash
# Scale to 2 instances
az acr agentpool update \
    --registry myregistry \
    --name myagentpool \
    --count 2
```

### Scale to Zero

Reduce costs when pool is not needed:

```bash
az acr agentpool update \
    --registry myregistry \
    --name myagentpool \
    --count 0
```

> **Warning**: Tasks cannot run when count is 0. Tasks queued on an empty pool will fail.

## VNet Deployment

### Step 1: Configure Network Security Group

Add required outbound rules to your NSG:

| Priority | Direction | Protocol | Source | Source Port | Destination | Dest Port | Action |
|----------|-----------|----------|--------|-------------|-------------|-----------|--------|
| 100 | Outbound | TCP | VirtualNetwork | Any | AzureKeyVault | 443 | Allow |
| 110 | Outbound | TCP | VirtualNetwork | Any | Storage | 443 | Allow |
| 120 | Outbound | TCP | VirtualNetwork | Any | EventHub | 443 | Allow |
| 130 | Outbound | TCP | VirtualNetwork | Any | AzureActiveDirectory | 443 | Allow |
| 140 | Outbound | TCP | VirtualNetwork | Any | AzureMonitor | 443,12000 | Allow |

### Step 2: Enable Service Endpoints (Recommended)

For fine-grained control, enable service endpoints on the subnet:

```bash
az network vnet subnet update \
    --resource-group myresourcegroup \
    --vnet-name myvnet \
    --name mysubnet \
    --service-endpoints \
        Microsoft.AzureActiveDirectory \
        Microsoft.ContainerRegistry \
        Microsoft.EventHub \
        Microsoft.KeyVault \
        Microsoft.Storage
```

### Step 3: Get Subnet ID

```bash
subnetId=$(az network vnet subnet show \
    --resource-group myresourcegroup \
    --vnet-name myvnet \
    --name mysubnetname \
    --query id --output tsv)

echo $subnetId
```

### Step 4: Create VNet-Connected Agent Pool

```bash
az acr agentpool create \
    --registry myregistry \
    --name myagentpool \
    --tier S2 \
    --subnet-id $subnetId
```

### Complete VNet Deployment Script

```bash
#!/bin/bash

# Variables
REGISTRY_NAME="myregistry"
RESOURCE_GROUP="myresourcegroup"
VNET_NAME="myvnet"
SUBNET_NAME="mysubnet"
POOL_NAME="myagentpool"
POOL_TIER="S2"
POOL_COUNT=2

# Get subnet ID
subnetId=$(az network vnet subnet show \
    --resource-group $RESOURCE_GROUP \
    --vnet-name $VNET_NAME \
    --name $SUBNET_NAME \
    --query id --output tsv)

# Create agent pool in VNet
az acr agentpool create \
    --registry $REGISTRY_NAME \
    --name $POOL_NAME \
    --tier $POOL_TIER \
    --subnet-id $subnetId \
    --count $POOL_COUNT

# Verify creation
az acr agentpool show \
    --registry $REGISTRY_NAME \
    --name $POOL_NAME
```

## Deployment for Network-Restricted Registries

When your registry has public network access disabled:

### Option 1: Use VNet-Connected Agent Pool

Deploy agent pool in VNet that can access the private registry:

```bash
# Registry with private endpoint in same VNet
az acr agentpool create \
    --registry myregistry \
    --name myagentpool \
    --tier S2 \
    --subnet-id /subscriptions/<sub>/resourceGroups/<rg>/providers/Microsoft.Network/virtualNetworks/<vnet>/subnets/<subnet>
```

### Option 2: Enable Network Bypass Policy

If using System-Assigned Managed Identity (SAMI) without VNet:

```bash
az resource update \
    --namespace Microsoft.ContainerRegistry \
    --resource-type registries \
    --name myregistry \
    --resource-group myresourcegroup \
    --api-version 2025-06-01-preview \
    --set properties.networkRuleBypassAllowedForTasks=true
```

## Multiple Pool Configurations

### Create Pools for Different Workloads

```bash
# Pool for quick builds (cost-optimized)
az acr agentpool create \
    --registry myregistry \
    --name pool-quick \
    --tier S1 \
    --count 1

# Pool for standard builds
az acr agentpool create \
    --registry myregistry \
    --name pool-standard \
    --tier S2 \
    --count 2

# Pool for heavy builds
az acr agentpool create \
    --registry myregistry \
    --name pool-heavy \
    --tier S3 \
    --count 2

# Pool for isolated/secure builds
az acr agentpool create \
    --registry myregistry \
    --name pool-isolated \
    --tier I6 \
    --count 1
```

## Quota Management

### Check Current Quota Usage

Contact Azure support or check your subscription limits.

### Request Quota Increase

For isolated pools (I6) or when exceeding default 16 vCPU limit:

1. Go to Azure Portal > Help + Support
2. Create new support request
3. Select "Service and subscription limits (quotas)"
4. Request increase for ACR agent pool vCPU quota

## Deployment Validation

### Verify Pool Status

```bash
az acr agentpool show \
    --registry myregistry \
    --name myagentpool \
    --output table
```

### Check Queue Status

```bash
az acr agentpool show \
    --registry myregistry \
    --name myagentpool \
    --queue-count
```

### Test with Quick Build

```bash
az acr build \
    --registry myregistry \
    --agent-pool myagentpool \
    --image testimage:v1 \
    --file Dockerfile \
    https://github.com/Azure-Samples/acr-build-helloworld-node.git#main
```

## Troubleshooting Deployment

### Pool Creation Fails

1. **Check registry tier**: Must be Premium
2. **Check region**: Must be in supported preview region
3. **Check quota**: May have exceeded vCPU quota

### VNet Pool Connection Issues

1. **Verify NSG rules**: Ensure all required outbound rules exist
2. **Check subnet**: Must have sufficient IP addresses
3. **Verify service endpoints**: Enable for required services

### Tasks Fail on Pool

1. **Check pool count**: Must be at least 1
2. **Verify pool status**: Pool must be provisioned
3. **Check network connectivity**: For VNet pools, verify firewall rules

## Clean Up

### Delete Agent Pool

```bash
az acr agentpool delete \
    --registry myregistry \
    --name myagentpool \
    --yes
```

### Delete Multiple Pools

```bash
for pool in pool-quick pool-standard pool-heavy pool-isolated; do
    az acr agentpool delete \
        --registry myregistry \
        --name $pool \
        --yes
done
```

## Source References

- Microsoft Learn: `tasks-agent-pools.md` - Lines 54-156
- ACR GitHub: `docs/tasks/agentpool/README.md` - Lines 26-77
- Network Bypass Policy: `manage-network-bypass-policy-for-tasks.md` - Lines 76-124

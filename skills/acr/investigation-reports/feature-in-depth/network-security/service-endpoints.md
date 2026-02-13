# ACR VNet Service Endpoints

## Overview

Azure Virtual Network (VNet) service endpoints provide secure access to Azure Container Registry by routing traffic over the Azure backbone network. This feature allows you to restrict registry access to specific VNet subnets, ensuring that only resources within those subnets can access the registry.

## Key Features

- **Secure Public IP**: Traffic from VNet uses optimal Azure backbone routing
- **Identity Transmission**: VNet and subnet identities are transmitted with each request
- **Network Isolation**: Registry accessible only from configured subnets
- **Preview Status**: Currently in preview with some limitations

## Prerequisites

- **SKU**: Premium tier required
- **Azure CLI**: Version 2.0.58 or later
- **Resource Provider**: `Microsoft.ContainerRegistry` must be registered
- **Maximum Rules**: 100 VNet rules per Premium registry

## Important Recommendation

> **Azure Private Link is Recommended**: Microsoft recommends using Private Link (private endpoints) instead of service endpoints for most network scenarios. Private endpoints are accessible from within the virtual network using private IP addresses and offer better security.

## Preview Limitations

1. **Portal Limitation**: Cannot configure service endpoints via Azure portal (CLI only)
2. **Supported Hosts**: Only AKS clusters and Azure VMs are supported
3. **Unsupported Services**: Azure Container Instances not supported
4. **Cloud Limitations**: Not available in Azure US Government or Azure China clouds
5. **Mutual Exclusivity**: Cannot use both Private Link and service endpoints

## Configuration Steps

### Step 1: Add Service Endpoint to Subnet

First, enable the `Microsoft.ContainerRegistry` service endpoint on your subnet:

```bash
az network vnet subnet update \
  --name myDockerVMSubnet \
  --vnet-name myDockerVMVNET \
  --resource-group myResourceGroup \
  --service-endpoints Microsoft.ContainerRegistry
```

### Step 2: Get Subnet Resource ID

Retrieve the subnet resource ID for use in network rules:

```bash
az network vnet subnet show \
  --name myDockerVMSubnet \
  --vnet-name myDockerVMVNET \
  --resource-group myResourceGroup \
  --query "id" \
  --output tsv
```

Output format:
```
/subscriptions/<subscription-id>/resourceGroups/<resource-group>/providers/Microsoft.Network/virtualNetworks/<vnet-name>/subnets/<subnet-name>
```

### Step 3: Change Default Network Access

Set the default action to deny access from all networks:

```bash
az acr update --name myContainerRegistry --default-action Deny
```

### Step 4: Add Network Rule for Subnet

Add a network rule to allow access from the configured subnet:

```bash
az acr network-rule add \
  --name mycontainerregistry \
  --subnet /subscriptions/<subscription-id>/resourceGroups/myResourceGroup/providers/Microsoft.Network/virtualNetworks/myDockerVMVNET/subnets/myDockerVMSubnet
```

Or using a variable:

```bash
SUBNET_ID=$(az network vnet subnet show \
  --name myDockerVMSubnet \
  --vnet-name myDockerVMVNET \
  --resource-group myResourceGroup \
  --query "id" --output tsv)

az acr network-rule add \
  --name mycontainerregistry \
  --subnet $SUBNET_ID
```

## Cross-Subscription Configuration

To configure service endpoints from a different Azure subscription:

1. Register the resource provider in the VNet's subscription:

```bash
az account set --subscription <Name or ID of VNet subscription>
az provider register --namespace Microsoft.ContainerRegistry
```

2. Add the network rule using the full subnet resource ID

## Verifying Access

### Test from VM in VNet

1. SSH to VM in the configured subnet
2. Log in to registry:

```bash
az acr login --name mycontainerregistry
```

3. Pull an image:

```bash
docker pull mycontainerregistry.azurecr.io/hello-world:v1
```

### Expected Results

- **From configured subnet**: Access succeeds
- **From outside subnet**: Access denied with 403 Forbidden error

```
Error response from daemon: login attempt to https://xxxxxxx.azurecr.io/v2/ failed with status: 403 Forbidden
```

## Managing Network Rules

### List Network Rules

```bash
az acr network-rule list --name mycontainerregistry
```

### Remove Network Rule

```bash
az acr network-rule remove \
  --name mycontainerregistry \
  --subnet /subscriptions/<subscription-id>/resourceGroups/myResourceGroup/providers/Microsoft.Network/virtualNetworks/myDockerVMVNET/subnets/myDockerVMSubnet
```

### Restore Default Access

After removing rules, restore default access:

```bash
az acr update --name myContainerRegistry --default-action Allow
```

## Interaction with Other Features

### Disabling Public Access

When public network access is disabled:
- Service endpoint access is also disabled
- Use Private Link instead for VNet-only access

```bash
# This disables both public and service endpoint access
az acr update --name myContainerRegistry --public-network-enabled false
```

### Cannot Combine with Private Link

If you configure Private Link for a registry:
1. Remove service endpoint network rules first
2. Service endpoints and Private Link are mutually exclusive

```bash
# List and remove network rules before enabling Private Link
az acr network-rule list --name mycontainerregistry
az acr network-rule remove --name mycontainerregistry --subnet <subnet-id>
```

### Trusted Services

The "Allow trusted services" setting does NOT apply to registries with service endpoints:
- Only works with Private Link or IP access rules
- Use agent pools for ACR Tasks with service endpoints

## Agent Pools for VNet-Enabled Registries

For ACR Tasks with VNet-restricted registries, use dedicated agent pools:

### Required Firewall Rules for Agent Pool Subnet

| Direction | Protocol | Source | Destination | Port | Service |
|-----------|----------|--------|-------------|------|---------|
| Outbound | TCP | VirtualNetwork | AzureKeyVault | 443 | Key Vault |
| Outbound | TCP | VirtualNetwork | Storage | 443 | Storage |
| Outbound | TCP | VirtualNetwork | EventHub | 443 | Event Hub |
| Outbound | TCP | VirtualNetwork | AzureActiveDirectory | 443 | AAD |
| Outbound | TCP | VirtualNetwork | AzureMonitor | 443, 12000 | Monitoring |

### Create Agent Pool in VNet

```bash
SUBNET_ID=$(az network vnet subnet show \
    --resource-group myresourcegroup \
    --vnet-name myvnetname \
    --name mysubnetname \
    --query id --output tsv)

az acr agentpool create \
    --name myagentpool \
    --registry myregistry \
    --tier S2 \
    --subnet-id $SUBNET_ID
```

### Enable Service Endpoints on Agent Pool Subnet

For advanced network configuration:

```bash
az network vnet subnet update \
  --name myagentpoolsubnet \
  --vnet-name myvnet \
  --resource-group myResourceGroup \
  --service-endpoints \
    Microsoft.AzureActiveDirectory \
    Microsoft.ContainerRegistry \
    Microsoft.EventHub \
    Microsoft.KeyVault \
    Microsoft.Storage
```

## Troubleshooting

### Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| 403 Forbidden | Subnet not in allowlist | Add network rule for subnet |
| Configuration fails | Service endpoint not enabled | Enable `Microsoft.ContainerRegistry` service endpoint |
| Cross-subscription fails | Resource provider not registered | Register `Microsoft.ContainerRegistry` in VNet subscription |
| Cannot configure via portal | Portal limitation | Use Azure CLI |

### Diagnostic Commands

Check registry health:

```bash
az acr check-health --name myregistry
```

Check VNet connectivity (from VM):

```bash
az acr check-health --name myregistry --vnet myvnetname
```

Verify service endpoint on subnet:

```bash
az network vnet subnet show \
  --name mysubnet \
  --vnet-name myvnet \
  --resource-group myResourceGroup \
  --query serviceEndpoints
```

## Comparison: Service Endpoints vs Private Link

| Aspect | Service Endpoints | Private Link |
|--------|-------------------|--------------|
| Configuration | CLI only (preview) | Portal and CLI |
| IP Address | Public IP (optimized routing) | Private IP |
| Supported hosts | VM, AKS only | Any VNet resource |
| On-premises access | No | Yes (via ExpressRoute/VPN) |
| DNS configuration | Not required | Required (Private DNS Zone) |
| Recommendation | Limited scenarios | Preferred approach |

## Source Files

- `/submodules/azure-management-docs/articles/container-registry/container-registry-vnet.md`
- `/submodules/azure-management-docs/articles/container-registry/container-registry-private-link.md`
- `/submodules/azure-management-docs/articles/container-registry/tasks-agent-pools.md`
- `/submodules/acr/docs/tasks/agentpool/README.md`

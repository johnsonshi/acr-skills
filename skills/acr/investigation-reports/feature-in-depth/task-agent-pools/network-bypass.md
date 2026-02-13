# ACR Task Agent Pools - Network Bypass Policy

## Overview

The `networkRuleBypassAllowedForTasks` setting is a policy that controls whether ACR Tasks using System-Assigned Managed Identity (SAMI) can bypass network restrictions to access a container registry. This document covers the relationship between this policy and agent pools as an alternative approach.

## The Network Bypass Challenge

### Background

When a container registry is network-isolated (via private endpoints or firewall rules), ACR Tasks need a way to authenticate and access the registry. Historically, ACR Tasks could bypass network controls as a "trusted service." This approach is changing.

### Policy Change Timeline

**Effective June 1, 2025**: The `networkRuleBypassAllowedForTasks` setting defaults to `false`. This means:
- Tasks using SAMI tokens will be denied network bypass by default
- Customers relying on implicit network bypass will see `403 forbidden errors`
- Explicit configuration is required to restore previous behavior

## Two Solutions for Network-Restricted Registries

### Solution 1: Enable Network Bypass Policy

Explicitly allow tasks to bypass network rules:

```bash
registry="myregistry"
resourceGroup="myresourcegroup"

az resource update \
    --namespace Microsoft.ContainerRegistry \
    --resource-type registries \
    --name $registry \
    --resource-group $resourceGroup \
    --api-version 2025-06-01-preview \
    --set properties.networkRuleBypassAllowedForTasks=true
```

### Solution 2: Use Agent Pools (Recommended)

Deploy agent pools within your VNet to access network-restricted registries:

```bash
agentpool="myagentpool"
registry="myregistry"
subnet="/subscriptions/<SubscriptionId>/resourceGroups/<ResourceGroupName>/providers/Microsoft.Network/virtualNetworks/<NetworkName>/subnets/<SubnetName>"

az acr agentpool create \
    --name $agentpool \
    --registry $registry \
    --subnet-id $subnet
```

## Solution Comparison

| Aspect | Network Bypass Policy | Agent Pool in VNet |
|--------|----------------------|-------------------|
| Setup Complexity | Simple (one command) | More complex (VNet config) |
| Network Control | Less control | Full network control |
| Cost | No additional cost | Pool allocation costs |
| Security | Relies on SAMI trust | VNet isolation |
| Scaling | N/A | Configurable instances |
| Use Case | Quick setup, trusted environments | Enterprise, compliance-focused |

## Understanding the Network Bypass Policy

### What It Controls

The `networkRuleBypassAllowedForTasks` setting specifically controls:
- Whether tasks using **System-Assigned Managed Identity (SAMI)** tokens can bypass network rules
- Does **NOT** affect tasks using **User-Assigned Identity** (UAI)

### Security Implications

When enabled, understand these risks:
- SAMI tokens become sensitive credentials
- If mishandled (written to logs), tokens could be intercepted and misused

### Best Practices for Token Security

1. **Never output tokens to logs**
2. **Implement strict logging hygiene**
3. **Monitor for accidental token leakage**
4. **Regularly audit task definitions and logs**

## Managing the Network Bypass Policy

### Enable the Policy

```bash
registry="myregistry"
resourceGroup="myresourcegroup"

az resource update \
    --namespace Microsoft.ContainerRegistry \
    --resource-type registries \
    --name $registry \
    --resource-group $resourceGroup \
    --api-version 2025-06-01-preview \
    --set properties.networkRuleBypassAllowedForTasks=true
```

### Disable the Policy

```bash
registry="myregistry"
resourceGroup="myresourcegroup"

az resource update \
    --namespace Microsoft.ContainerRegistry \
    --resource-type registries \
    --name $registry \
    --resource-group $resourceGroup \
    --api-version 2025-06-01-preview \
    --set properties.networkRuleBypassAllowedForTasks=false
```

### Check Policy Status

```bash
registry="myregistry"
resourceGroup="myresourcegroup"

az resource show \
    --namespace Microsoft.ContainerRegistry \
    --resource-type registries \
    --name $registry \
    --resource-group $resourceGroup \
    --api-version 2025-06-01-preview \
    --query properties.networkRuleBypassAllowedForTasks
```

## Agent Pool as Network Solution

### Why Agent Pools Solve the Problem

When deployed in a VNet, agent pools:
1. Run tasks within your controlled network
2. Can access private endpoints directly
3. Don't require network bypass permissions
4. Provide isolation from public internet

### Firewall Rules for VNet Agent Pools

Ensure these outbound rules are configured:

| Direction | Protocol | Source | Source Port | Destination | Dest Port |
|-----------|----------|--------|-------------|-------------|-----------|
| Outbound | TCP | VirtualNetwork | Any | AzureKeyVault | 443 |
| Outbound | TCP | VirtualNetwork | Any | Storage | 443 |
| Outbound | TCP | VirtualNetwork | Any | EventHub | 443 |
| Outbound | TCP | VirtualNetwork | Any | AzureActiveDirectory | 443 |
| Outbound | TCP | VirtualNetwork | Any | AzureMonitor | 443,12000 |

### Advanced Network Configuration

For more restrictive environments:

1. **Enable Service Endpoints**: Grant access via service endpoints instead of broad outbound rules
2. **Configure Route Tables**: Control traffic flow with user-defined routes

Required service endpoints:
- `Microsoft.AzureActiveDirectory`
- `Microsoft.ContainerRegistry` (if not using private link)
- `Microsoft.EventHub`
- `Microsoft.KeyVault`
- `Microsoft.Storage`

## Customer Scenarios

### Scenario 1: Enterprise with Strict Network Controls

**Recommendation**: Use Agent Pools in VNet

```bash
# Create VNet-connected agent pool
subnet=$(az network vnet subnet show \
    --resource-group myresourcegroup \
    --vnet-name myvnet \
    --name mysubnet \
    --query id -o tsv)

az acr agentpool create \
    --name myagentpool \
    --registry myregistry \
    --tier S2 \
    --subnet-id $subnet
    --count 2

# Run task on agent pool
az acr build \
    --registry myregistry \
    --agent-pool myagentpool \
    --image myimage:v1 \
    --file Dockerfile \
    https://github.com/myorg/myrepo.git#main
```

### Scenario 2: Quick Setup, Trusted Environment

**Recommendation**: Enable Network Bypass Policy

```bash
# Enable bypass
az resource update \
    --namespace Microsoft.ContainerRegistry \
    --resource-type registries \
    --name myregistry \
    --resource-group myresourcegroup \
    --api-version 2025-06-01-preview \
    --set properties.networkRuleBypassAllowedForTasks=true

# Run task normally (no agent pool)
az acr build \
    --registry myregistry \
    --image myimage:v1 \
    --file Dockerfile .
```

### Scenario 3: No Tasks Needed, Local Operations

**Recommendation**: Build locally or use self-hosted agents

- Use `docker build` and `docker push` from local machines
- Download ACR CLI from [GitHub](https://github.com/azure/acr-cli) for purge operations
- Operations occur within your trusted environment

### Scenario 4: User-Assigned Identity Already Configured

**No Action Required**: UAI is not affected by the network bypass policy change

Tasks using User-Assigned Managed Identity will continue to work without modification.

## Migration Path

### From Default Behavior to Explicit Configuration

If you currently rely on implicit network bypass:

1. **Audit your tasks**: Identify tasks using SAMI authentication
2. **Choose your approach**:
   - Option A: Enable `networkRuleBypassAllowedForTasks=true`
   - Option B: Deploy agent pools in VNet
   - Option C: Switch to User-Assigned Identity
3. **Test thoroughly**: Verify tasks work before June 1, 2025

### Testing Configuration

After making changes:

```bash
# Run a test task
az acr task run \
    --registry myregistry \
    --name mytask

# Check streamed logs for errors
az acr task logs --registry myregistry --run-id <run-id>
```

## Relationship to Trusted Services

### Historical Context

ACR Tasks previously operated as a "trusted service" that could automatically bypass network restrictions when the "Allow trusted Microsoft services" setting was enabled.

### Current Status

ACR Tasks have **transitioned from** the trusted services list. The `networkRuleBypassAllowedForTasks` setting provides explicit control:
- `true`: Tasks can bypass (like the old trusted service behavior)
- `false`: Tasks cannot bypass (explicit denial, the new default)

### Comparison with Other Trusted Services

| Service | Trusted Service Status | Bypass Mechanism |
|---------|----------------------|------------------|
| Azure Container Instances | Yes | Managed identity + RBAC |
| Microsoft Defender | Yes | Automatic |
| Azure Machine Learning | Yes | Managed identity + RBAC |
| ACR Import | Yes | Automatic |
| **ACR Tasks** | **Transitioning** | **Explicit policy setting** |

## Source Documentation

- Network Bypass Policy: `/articles/container-registry/manage-network-bypass-policy-for-tasks.md`
- Trusted Services: `/articles/container-registry/allow-access-trusted-services.md`
- Agent Pools: `/articles/container-registry/tasks-agent-pools.md`
- Private Link: `/articles/container-registry/container-registry-private-link.md`

# ACR Task Agent Pools Skill

This skill provides comprehensive knowledge about ACR Task agent pools.

## When to Use This Skill

Use this skill when answering questions about:
- Dedicated compute for ACR Tasks
- VNet-integrated builds
- Agent pool configuration
- Network bypass alternatives

## Overview

Agent pools provide dedicated compute for ACR Tasks with VNet support.

**Requirements:** Premium SKU only (Preview)

## Pool Tiers

| Tier | vCPUs | Memory | Use Case |
|------|-------|--------|----------|
| S1 | 2 | 3 GiB | Light builds |
| S2 | 4 | 8 GiB | Standard builds |
| S3 | 8 | 16 GiB | Heavy builds |
| I6 | 64 | 216 GiB | Isolated workloads |

## Quick Setup

```bash
# Create agent pool
az acr agentpool create \
  --registry myregistry \
  --name mypool \
  --tier S2 \
  --count 2

# Use agent pool for build
az acr build \
  --registry myregistry \
  --agent-pool mypool \
  --image myapp:v1 .

# Use agent pool for task
az acr task create \
  --name mytask \
  --registry myregistry \
  --agent-pool mypool \
  ...
```

## VNet Integration

```bash
# Create agent pool in VNet
az acr agentpool create \
  --registry myregistry \
  --name mypool \
  --tier S2 \
  --subnet-id /subscriptions/{sub}/resourceGroups/{rg}/providers/Microsoft.Network/virtualNetworks/{vnet}/subnets/{subnet}
```

### VNet Requirements
- Subnet must have service endpoints enabled
- NSG must allow outbound to Azure services
- Minimum /27 subnet recommended

## Management Commands

```bash
# List pools
az acr agentpool list --registry myregistry -o table

# Show pool details
az acr agentpool show --registry myregistry --name mypool

# Update pool count
az acr agentpool update \
  --registry myregistry \
  --name mypool \
  --count 4

# Delete pool
az acr agentpool delete --registry myregistry --name mypool
```

## Network Bypass Policy

> **June 2025 Change:** `networkRuleBypassAllowedForTasks` now defaults to `false`

Agent pools provide an alternative to network bypass:

| Approach | Security | Setup Complexity |
|----------|----------|------------------|
| Network Bypass | Lower | Simple |
| Agent Pools | Higher | More setup |

## Use Cases

1. **Private Registry Access**: Build from network-restricted registries
2. **Private Resource Access**: Access databases, APIs in VNet
3. **Compliance**: Isolated compute environment
4. **Performance**: Dedicated, consistent compute

## Quotas

| Limit | Default |
|-------|---------|
| Agent pools per registry | 10 |
| Instances per pool | 20 |

Request increases via Azure Support.

## Limitations

- Linux nodes only (no Windows)
- Premium SKU required
- Preview feature

## Reference Documentation

- **Investigation reports**: `agent-investigation-reports/acr-features/task-agent-pools/`
- **MS Learn docs**: `azure-management-docs/articles/container-registry/tasks-agent-pools.md`

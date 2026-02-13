# ACR Task Agent Pools - CLI Commands Reference

## Command Overview

All agent pool commands are under the `az acr agentpool` command group. This reference covers all commands, parameters, and usage examples.

## Prerequisites

```bash
# Required Azure CLI version
az --version  # Must be 2.3.1 or later

# Login to Azure
az login

# Set subscription (if multiple)
az account set --subscription "My Subscription"

# Optionally set default registry
az config set defaults.acr=myregistry
```

## Core Commands

### az acr agentpool create

Creates a new agent pool in the specified registry.

**Syntax:**
```bash
az acr agentpool create --name
                        --registry
                        [--count]
                        [--no-wait]
                        [--resource-group]
                        [--subnet-id]
                        [--tier]
```

**Parameters:**

| Parameter | Required | Description |
|-----------|----------|-------------|
| `--name`, `-n` | Yes | Name of the agent pool |
| `--registry`, `-r` | Yes | Name of the container registry (can be omitted if default set) |
| `--count`, `-c` | No | Number of instances (default: 1) |
| `--no-wait` | No | Don't wait for operation to complete |
| `--resource-group`, `-g` | No | Resource group (inferred from registry if not provided) |
| `--subnet-id` | No | Subnet ID for VNet integration |
| `--tier` | No | Pool tier: S1, S2, S3, or I6 (default: S1) |

**Examples:**

```bash
# Basic creation
az acr agentpool create \
    --name myagentpool \
    --registry myregistry

# With specific tier
az acr agentpool create \
    --registry myregistry \
    --name myagentpool \
    --tier S2

# With multiple instances
az acr agentpool create \
    --registry myregistry \
    --name myagentpool \
    --tier S2 \
    --count 3

# In VNet
az acr agentpool create \
    --registry myregistry \
    --name myagentpool \
    --tier S2 \
    --subnet-id "/subscriptions/<sub-id>/resourceGroups/<rg>/providers/Microsoft.Network/virtualNetworks/<vnet>/subnets/<subnet>"

# Non-blocking creation
az acr agentpool create \
    --registry myregistry \
    --name myagentpool \
    --tier S3 \
    --no-wait
```

---

### az acr agentpool update

Updates an existing agent pool configuration.

**Syntax:**
```bash
az acr agentpool update --name
                        --registry
                        [--count]
                        [--no-wait]
                        [--resource-group]
```

**Parameters:**

| Parameter | Required | Description |
|-----------|----------|-------------|
| `--name`, `-n` | Yes | Name of the agent pool |
| `--registry`, `-r` | Yes | Name of the container registry |
| `--count`, `-c` | No | New instance count |
| `--no-wait` | No | Don't wait for operation to complete |
| `--resource-group`, `-g` | No | Resource group |

**Examples:**

```bash
# Scale to 2 instances
az acr agentpool update \
    --registry myregistry \
    --name myagentpool \
    --count 2

# Scale to zero (cost savings)
az acr agentpool update \
    --registry myregistry \
    --name myagentpool \
    --count 0

# Scale up without waiting
az acr agentpool update \
    --registry myregistry \
    --name myagentpool \
    --count 5 \
    --no-wait
```

---

### az acr agentpool show

Displays details of a specific agent pool.

**Syntax:**
```bash
az acr agentpool show --name
                      --registry
                      [--query]
                      [--queue-count]
                      [--resource-group]
```

**Parameters:**

| Parameter | Required | Description |
|-----------|----------|-------------|
| `--name`, `-n` | Yes | Name of the agent pool |
| `--registry`, `-r` | Yes | Name of the container registry |
| `--queue-count` | No | Display only the number of queued runs |
| `--resource-group`, `-g` | No | Resource group |

**Examples:**

```bash
# Show full pool details
az acr agentpool show \
    --registry myregistry \
    --name myagentpool

# Show only queue count
az acr agentpool show \
    --registry myregistry \
    --name myagentpool \
    --queue-count

# Output as table
az acr agentpool show \
    --registry myregistry \
    --name myagentpool \
    --output table

# Get specific property
az acr agentpool show \
    --registry myregistry \
    --name myagentpool \
    --query tier
```

---

### az acr agentpool list

Lists all agent pools in a registry.

**Syntax:**
```bash
az acr agentpool list --registry
                      [--resource-group]
```

**Parameters:**

| Parameter | Required | Description |
|-----------|----------|-------------|
| `--registry`, `-r` | Yes | Name of the container registry |
| `--resource-group`, `-g` | No | Resource group |

**Examples:**

```bash
# List all pools
az acr agentpool list --registry myregistry

# Output as table
az acr agentpool list \
    --registry myregistry \
    --output table

# Get specific fields
az acr agentpool list \
    --registry myregistry \
    --query "[].{Name:name, Tier:tier, Count:count}"
```

---

### az acr agentpool delete

Deletes an agent pool from the registry.

**Syntax:**
```bash
az acr agentpool delete --name
                        --registry
                        [--no-wait]
                        [--resource-group]
                        [--yes]
```

**Parameters:**

| Parameter | Required | Description |
|-----------|----------|-------------|
| `--name`, `-n` | Yes | Name of the agent pool |
| `--registry`, `-r` | Yes | Name of the container registry |
| `--no-wait` | No | Don't wait for operation to complete |
| `--resource-group`, `-g` | No | Resource group |
| `--yes`, `-y` | No | Skip confirmation prompt |

**Examples:**

```bash
# Delete with confirmation
az acr agentpool delete \
    --registry myregistry \
    --name myagentpool

# Delete without confirmation
az acr agentpool delete \
    --registry myregistry \
    --name myagentpool \
    --yes

# Delete without waiting
az acr agentpool delete \
    --registry myregistry \
    --name myagentpool \
    --yes \
    --no-wait
```

---

## Using Agent Pools with Tasks

### az acr build (Quick Task)

Run a one-time build on an agent pool.

**Syntax:**
```bash
az acr build --registry
             --agent-pool
             --image
             --file
             <context>
```

**Examples:**

```bash
# Build from GitHub
az acr build \
    --registry myregistry \
    --agent-pool myagentpool \
    --image myimage:mytag \
    --file Dockerfile \
    https://github.com/Azure-Samples/acr-build-helloworld-node.git#main

# Build from local context
az acr build \
    --registry myregistry \
    --agent-pool myagentpool \
    --image myimage:v1 \
    --file ./Dockerfile \
    .

# Build with specific platform
az acr build \
    --registry myregistry \
    --agent-pool myagentpool \
    --image myimage:v1 \
    --platform linux/amd64 \
    .
```

---

### az acr task create

Create a persistent task definition that runs on an agent pool.

**Syntax:**
```bash
az acr task create --name
                   --registry
                   --agent-pool
                   --image
                   --file
                   --context
                   [--schedule]
                   [--commit-trigger-enabled]
```

**Examples:**

```bash
# Create scheduled task
az acr task create \
    --registry myregistry \
    --name mytask \
    --agent-pool myagentpool \
    --image myimage:mytag \
    --schedule "0 21 * * *" \
    --file Dockerfile \
    --context https://github.com/Azure-Samples/acr-build-helloworld-node.git#main \
    --commit-trigger-enabled false

# Create task with commit trigger
az acr task create \
    --registry myregistry \
    --name mytask \
    --agent-pool myagentpool \
    --image myimage:{{.Run.ID}} \
    --file Dockerfile \
    --context https://github.com/myorg/myrepo.git
```

---

### az acr task run

Manually trigger a task run.

**Syntax:**
```bash
az acr task run --name
                --registry
                [--no-logs]
```

**Examples:**

```bash
# Run task
az acr task run \
    --registry myregistry \
    --name mytask

# Run without streaming logs
az acr task run \
    --registry myregistry \
    --name mytask \
    --no-logs
```

---

## Network Bypass Policy Commands

### Enable Network Bypass

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

### Disable Network Bypass

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

### Check Network Bypass Status

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

---

## VNet Configuration Commands

### Get Subnet ID

```bash
subnet=$(az network vnet subnet show \
    --resource-group myresourcegroup \
    --vnet-name myvnet \
    --name mysubnet \
    --query id --output tsv)
```

### Enable Service Endpoints

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

---

## Scripting Examples

### Full Deployment Script

```bash
#!/bin/bash
set -e

# Variables
REGISTRY_NAME="myregistry"
RESOURCE_GROUP="myresourcegroup"
VNET_NAME="myvnet"
SUBNET_NAME="mysubnet"
POOL_NAME="myagentpool"
POOL_TIER="S2"
POOL_COUNT=2

# Check prerequisites
echo "Checking Azure CLI version..."
az --version | head -1

# Get subnet ID
echo "Getting subnet ID..."
SUBNET_ID=$(az network vnet subnet show \
    --resource-group $RESOURCE_GROUP \
    --vnet-name $VNET_NAME \
    --name $SUBNET_NAME \
    --query id --output tsv)

# Create agent pool
echo "Creating agent pool..."
az acr agentpool create \
    --registry $REGISTRY_NAME \
    --name $POOL_NAME \
    --tier $POOL_TIER \
    --subnet-id $SUBNET_ID \
    --count $POOL_COUNT

# Wait and verify
echo "Verifying pool creation..."
az acr agentpool show \
    --registry $REGISTRY_NAME \
    --name $POOL_NAME \
    --output table

echo "Agent pool $POOL_NAME created successfully!"
```

### Monitoring Script

```bash
#!/bin/bash

REGISTRY_NAME="myregistry"
POOL_NAME="myagentpool"

# Check pool status
echo "=== Pool Status ==="
az acr agentpool show \
    --registry $REGISTRY_NAME \
    --name $POOL_NAME \
    --output table

# Check queue
echo ""
echo "=== Queue Count ==="
az acr agentpool show \
    --registry $REGISTRY_NAME \
    --name $POOL_NAME \
    --queue-count
```

### Cleanup Script

```bash
#!/bin/bash

REGISTRY_NAME="myregistry"
POOLS=("pool-quick" "pool-standard" "pool-heavy")

for pool in "${POOLS[@]}"; do
    echo "Deleting $pool..."
    az acr agentpool delete \
        --registry $REGISTRY_NAME \
        --name $pool \
        --yes \
        --no-wait
done

echo "Cleanup initiated for all pools"
```

---

## Command Quick Reference

| Command | Purpose |
|---------|---------|
| `az acr agentpool create` | Create new agent pool |
| `az acr agentpool update` | Update pool configuration |
| `az acr agentpool show` | Display pool details |
| `az acr agentpool list` | List all pools |
| `az acr agentpool delete` | Delete agent pool |
| `az acr build --agent-pool` | Run quick build on pool |
| `az acr task create --agent-pool` | Create task targeting pool |
| `az acr task run` | Trigger task execution |

---

## Source Documentation

- Azure CLI Reference: `az acr agentpool` command group
- Microsoft Learn: `/articles/container-registry/tasks-agent-pools.md` - Lines 54-212
- ACR GitHub: `/docs/tasks/agentpool/README.md` - Lines 26-113
- Network Bypass: `/articles/container-registry/manage-network-bypass-policy-for-tasks.md` - Lines 26-139

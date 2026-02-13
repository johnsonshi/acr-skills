# ACR Network Security - CLI Commands Reference

## Overview

This document provides a comprehensive reference for all Azure CLI commands related to ACR network security features, including firewall rules, IP access control, VNet service endpoints, private endpoints, and trusted services.

## Prerequisites

- **Azure CLI**: Version 2.6.0 or later (for full feature support)
- **Subscription**: Appropriate permissions to manage ACR resources
- **SKU**: Premium tier required for most network security features

### Install/Upgrade Azure CLI

```bash
# Install
curl -sL https://aka.ms/InstallAzureCLIBuildScriptBashDev | sudo bash

# Upgrade
az upgrade

# Check version
az --version
```

## Registry Network Configuration

### View Current Network Settings

```bash
# Show all registry properties including network settings
az acr show --name myregistry --output table

# Show network rule set specifically
az acr show --name myregistry --query networkRuleSet

# Show public network access status
az acr show --name myregistry --query publicNetworkAccess

# Show all endpoints
az acr show-endpoints --name myregistry
```

### Public Network Access

```bash
# Disable public network access entirely
az acr update --name myregistry --public-network-enabled false

# Enable public network access
az acr update --name myregistry --public-network-enabled true
```

### Default Network Action

```bash
# Set default action to Deny (block all unless allowed)
az acr update --name myregistry --default-action Deny

# Set default action to Allow (permit all)
az acr update --name myregistry --default-action Allow
```

## IP Network Rules (Firewall)

### Add IP Rules

```bash
# Add single IP address
az acr network-rule add \
  --name myregistry \
  --ip-address 203.0.113.50

# Add CIDR range
az acr network-rule add \
  --name myregistry \
  --ip-address 10.0.0.0/24

# Add multiple IPs (run multiple commands)
az acr network-rule add --name myregistry --ip-address 10.1.0.0/24
az acr network-rule add --name myregistry --ip-address 10.2.0.0/24
```

### List IP Rules

```bash
# List all network rules
az acr network-rule list --name myregistry

# List only IP rules (filter output)
az acr network-rule list --name myregistry --query ipRules

# Format as table
az acr network-rule list --name myregistry --output table
```

### Remove IP Rules

```bash
# Remove specific IP address
az acr network-rule remove \
  --name myregistry \
  --ip-address 203.0.113.50

# Remove CIDR range
az acr network-rule remove \
  --name myregistry \
  --ip-address 10.0.0.0/24
```

## VNet Service Endpoint Rules

### Enable Service Endpoint on Subnet

```bash
# Enable Microsoft.ContainerRegistry service endpoint
az network vnet subnet update \
  --name mysubnet \
  --vnet-name myvnet \
  --resource-group myResourceGroup \
  --service-endpoints Microsoft.ContainerRegistry
```

### Get Subnet Resource ID

```bash
# Get subnet ID for use in network rules
az network vnet subnet show \
  --name mysubnet \
  --vnet-name myvnet \
  --resource-group myResourceGroup \
  --query id --output tsv
```

### Add VNet Rules

```bash
# Add VNet rule using subnet ID
az acr network-rule add \
  --name myregistry \
  --subnet /subscriptions/<sub-id>/resourceGroups/<rg>/providers/Microsoft.Network/virtualNetworks/<vnet>/subnets/<subnet>

# Using variable
SUBNET_ID=$(az network vnet subnet show \
  --name mysubnet \
  --vnet-name myvnet \
  --resource-group myResourceGroup \
  --query id --output tsv)

az acr network-rule add \
  --name myregistry \
  --subnet $SUBNET_ID
```

### List VNet Rules

```bash
# List VNet rules
az acr network-rule list --name myregistry --query virtualNetworkRules
```

### Remove VNet Rules

```bash
# Remove VNet rule
az acr network-rule remove \
  --name myregistry \
  --subnet $SUBNET_ID
```

## Private Endpoints (Private Link)

### Create Private DNS Zone

```bash
# Create private DNS zone for ACR
az network private-dns zone create \
  --resource-group myResourceGroup \
  --name "privatelink.azurecr.io"
```

### Link DNS Zone to VNet

```bash
# Create VNet link
az network private-dns link vnet create \
  --resource-group myResourceGroup \
  --zone-name "privatelink.azurecr.io" \
  --name myDNSLink \
  --virtual-network myvnet \
  --registration-enabled false
```

### Disable Network Policies on Subnet

```bash
# Disable private endpoint network policies
az network vnet subnet update \
  --name mysubnet \
  --vnet-name myvnet \
  --resource-group myResourceGroup \
  --disable-private-endpoint-network-policies
```

### Create Private Endpoint

```bash
# Get registry resource ID
REGISTRY_ID=$(az acr show --name myregistry --query 'id' --output tsv)

# Create private endpoint
az network private-endpoint create \
  --name myPrivateEndpoint \
  --resource-group myResourceGroup \
  --vnet-name myvnet \
  --subnet mysubnet \
  --private-connection-resource-id $REGISTRY_ID \
  --group-ids registry \
  --connection-name myConnection
```

### Get Private Endpoint IP Configuration

```bash
# Get network interface ID
NETWORK_INTERFACE_ID=$(az network private-endpoint show \
  --name myPrivateEndpoint \
  --resource-group myResourceGroup \
  --query 'networkInterfaces[0].id' --output tsv)

# Get registry private IP
REGISTRY_PRIVATE_IP=$(az network nic show \
  --ids $NETWORK_INTERFACE_ID \
  --query "ipConfigurations[?privateLinkConnectionProperties.requiredMemberName=='registry'].privateIPAddress" \
  --output tsv)

# Get data endpoint private IP (replace region)
DATA_ENDPOINT_PRIVATE_IP=$(az network nic show \
  --ids $NETWORK_INTERFACE_ID \
  --query "ipConfigurations[?privateLinkConnectionProperties.requiredMemberName=='registry_data_eastus'].privateIPAddress" \
  --output tsv)
```

### Create DNS Records

```bash
# Create A-record set for registry
az network private-dns record-set a create \
  --name myregistry \
  --zone-name privatelink.azurecr.io \
  --resource-group myResourceGroup

# Add A-record for registry
az network private-dns record-set a add-record \
  --record-set-name myregistry \
  --zone-name privatelink.azurecr.io \
  --resource-group myResourceGroup \
  --ipv4-address $REGISTRY_PRIVATE_IP

# Create and add A-record for data endpoint
az network private-dns record-set a create \
  --name myregistry.eastus.data \
  --zone-name privatelink.azurecr.io \
  --resource-group myResourceGroup

az network private-dns record-set a add-record \
  --record-set-name myregistry.eastus.data \
  --zone-name privatelink.azurecr.io \
  --resource-group myResourceGroup \
  --ipv4-address $DATA_ENDPOINT_PRIVATE_IP
```

### Manage Private Endpoint Connections

```bash
# List private endpoint connections
az acr private-endpoint-connection list --registry-name myregistry

# Approve a pending connection
az acr private-endpoint-connection approve \
  --registry-name myregistry \
  --name myConnection

# Reject a connection
az acr private-endpoint-connection reject \
  --registry-name myregistry \
  --name myConnection

# Delete a connection
az acr private-endpoint-connection delete \
  --registry-name myregistry \
  --name myConnection
```

## Dedicated Data Endpoints

### Enable Dedicated Data Endpoints

```bash
# Enable dedicated data endpoints
az acr update --name myregistry --data-endpoint-enabled

# Disable dedicated data endpoints
az acr update --name myregistry --data-endpoint-enabled false
```

### View Data Endpoints

```bash
# Show all endpoints including data endpoints
az acr show-endpoints --name myregistry

# Output format
az acr show-endpoints --name myregistry --output json
```

## Trusted Services

### Enable/Disable Trusted Services

```bash
# Enable trusted services bypass
az acr update --name myregistry --allow-trusted-services true

# Disable trusted services bypass
az acr update --name myregistry --allow-trusted-services false

# Check current setting
az acr show --name myregistry --query networkRuleBypassOptions
```

### Network Bypass for Tasks (Preview)

```bash
# Enable network bypass for tasks
az resource update \
  --namespace Microsoft.ContainerRegistry \
  --resource-type registries \
  --name myregistry \
  --resource-group myResourceGroup \
  --api-version 2025-06-01-preview \
  --set properties.networkRuleBypassAllowedForTasks=true

# Disable network bypass for tasks
az resource update \
  --namespace Microsoft.ContainerRegistry \
  --resource-type registries \
  --name myregistry \
  --resource-group myResourceGroup \
  --api-version 2025-06-01-preview \
  --set properties.networkRuleBypassAllowedForTasks=false

# Check current setting
az resource show \
  --namespace Microsoft.ContainerRegistry \
  --resource-type registries \
  --name myregistry \
  --resource-group myResourceGroup \
  --api-version 2025-06-01-preview \
  --query properties.networkRuleBypassAllowedForTasks
```

## Agent Pools (VNet-Enabled Tasks)

### Create Agent Pool

```bash
# Create agent pool without VNet
az acr agentpool create \
  --registry myregistry \
  --name myagentpool \
  --tier S2

# Create agent pool in VNet
SUBNET_ID=$(az network vnet subnet show \
  --resource-group myResourceGroup \
  --vnet-name myvnet \
  --name mysubnet \
  --query id --output tsv)

az acr agentpool create \
  --registry myregistry \
  --name myagentpool \
  --tier S2 \
  --subnet-id $SUBNET_ID
```

### Manage Agent Pools

```bash
# Scale agent pool
az acr agentpool update \
  --registry myregistry \
  --name myagentpool \
  --count 2

# Scale to zero
az acr agentpool update \
  --registry myregistry \
  --name myagentpool \
  --count 0

# Show agent pool status
az acr agentpool show \
  --registry myregistry \
  --name myagentpool

# Check queue status
az acr agentpool show \
  --registry myregistry \
  --name myagentpool \
  --queue-count

# Delete agent pool
az acr agentpool delete \
  --registry myregistry \
  --name myagentpool
```

### Run Tasks on Agent Pool

```bash
# Quick build on agent pool
az acr build \
  --registry myregistry \
  --agent-pool myagentpool \
  --image myimage:mytag \
  --file Dockerfile \
  https://github.com/example/repo.git

# Create scheduled task on agent pool
az acr task create \
  --registry myregistry \
  --name mytask \
  --agent-pool myagentpool \
  --image myimage:mytag \
  --schedule "0 21 * * *" \
  --file Dockerfile \
  --context https://github.com/example/repo.git \
  --commit-trigger-enabled false
```

## Health and Diagnostics

### Check Registry Health

```bash
# Basic health check
az acr check-health --name myregistry

# Detailed health check
az acr check-health --name myregistry --yes

# Check health including VNet connectivity
az acr check-health --name myregistry --vnet myvnetname
```

### Check AKS-ACR Connectivity

```bash
# Validate AKS can reach ACR
az aks check-acr \
  --resource-group myAKSResourceGroup \
  --name myAKSCluster \
  --acr myregistry.azurecr.io
```

## Export Policy (Data Loss Prevention)

### Manage Export Policy

```bash
# Disable export (requires public network access also disabled)
az resource update \
  --resource-group myResourceGroup \
  --name myregistry \
  --resource-type "Microsoft.ContainerRegistry/registries" \
  --api-version "2021-06-01-preview" \
  --set "properties.policies.exportPolicy.status=disabled" \
  --set "properties.publicNetworkAccess=disabled"

# Enable export
az resource update \
  --resource-group myResourceGroup \
  --name myregistry \
  --resource-type "Microsoft.ContainerRegistry/registries" \
  --api-version "2021-06-01-preview" \
  --set "properties.policies.exportPolicy.status=enabled"
```

## Cross-Subscription Configuration

### Register Resource Provider

```bash
# Switch to VNet subscription
az account set --subscription "<VNet-subscription-name-or-id>"

# Register ACR resource provider
az provider register --namespace Microsoft.ContainerRegistry

# Verify registration
az provider show --namespace Microsoft.ContainerRegistry --query registrationState
```

## Complete Network Lockdown Example

```bash
#!/bin/bash
# Complete network lockdown script

REGISTRY="myregistry"
RESOURCE_GROUP="myResourceGroup"
VNET="myvnet"
SUBNET="mysubnet"

# 1. Create private DNS zone
az network private-dns zone create \
  --resource-group $RESOURCE_GROUP \
  --name "privatelink.azurecr.io"

# 2. Link DNS zone to VNet
az network private-dns link vnet create \
  --resource-group $RESOURCE_GROUP \
  --zone-name "privatelink.azurecr.io" \
  --name myDNSLink \
  --virtual-network $VNET \
  --registration-enabled false

# 3. Disable network policies
az network vnet subnet update \
  --name $SUBNET \
  --vnet-name $VNET \
  --resource-group $RESOURCE_GROUP \
  --disable-private-endpoint-network-policies

# 4. Create private endpoint
REGISTRY_ID=$(az acr show --name $REGISTRY --query 'id' --output tsv)

az network private-endpoint create \
  --name "${REGISTRY}-pe" \
  --resource-group $RESOURCE_GROUP \
  --vnet-name $VNET \
  --subnet $SUBNET \
  --private-connection-resource-id $REGISTRY_ID \
  --group-ids registry \
  --connection-name "${REGISTRY}-connection"

# 5. Configure DNS records (automated if using private DNS zone integration)

# 6. Disable public access
az acr update --name $REGISTRY --public-network-enabled false

# 7. Enable dedicated data endpoints
az acr update --name $REGISTRY --data-endpoint-enabled

# 8. Enable trusted services (if needed)
az acr update --name $REGISTRY --allow-trusted-services true

echo "Registry $REGISTRY is now locked down to private access only"
```

## Source Files

- `/submodules/azure-management-docs/articles/container-registry/container-registry-access-selected-networks.md`
- `/submodules/azure-management-docs/articles/container-registry/container-registry-vnet.md`
- `/submodules/azure-management-docs/articles/container-registry/container-registry-private-link.md`
- `/submodules/azure-management-docs/articles/container-registry/allow-access-trusted-services.md`
- `/submodules/azure-management-docs/articles/container-registry/manage-network-bypass-policy-for-tasks.md`
- `/submodules/azure-management-docs/articles/container-registry/tasks-agent-pools.md`
- `/submodules/acr/docs/tasks/agentpool/README.md`

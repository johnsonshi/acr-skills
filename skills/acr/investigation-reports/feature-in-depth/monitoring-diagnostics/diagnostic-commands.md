# ACR Diagnostic Commands Reference

## Overview

This document provides a comprehensive reference of diagnostic commands for troubleshooting Azure Container Registry issues. Commands are organized by category and include examples for common scenarios.

## Health Check Commands

### Basic Health Check

```bash
# Check local environment only
az acr check-health

# Check environment and registry access
az acr check-health --name myregistry

# Check all items, continue on errors
az acr check-health --name myregistry --ignore-errors --yes

# Check access via specific virtual network
az acr check-health --name myregistry --vnet myvnet

# VNet in different subscription (use resource ID)
az acr check-health --name myregistry \
  --vnet /subscriptions/{sub}/resourceGroups/{rg}/providers/Microsoft.Network/virtualNetworks/{vnet}
```

### AKS Integration Check

```bash
# Verify AKS cluster can access ACR
az aks check-acr --name myakscluster --resource-group myRG --acr myregistry
```

## Registry Information Commands

### Basic Registry Info

```bash
# Show registry details
az acr show --name myregistry --output table

# Show network rule configuration
az acr show --query networkRuleSet --name myregistry

# Show registry policies
az acr show --query policies --name myregistry

# Show registry SKU
az acr show --query sku --name myregistry --output table

# List all registries in subscription
az acr list --output table
```

### Login Server and Credentials

```bash
# Get login server URL
az acr show --name myregistry --query loginServer --output tsv

# Show admin credentials (if enabled)
az acr credential show --name myregistry
```

## Metrics Commands

### List Available Metrics

```bash
az monitor metrics list-definitions \
  --resource /subscriptions/{sub}/resourceGroups/{rg}/providers/Microsoft.ContainerRegistry/registries/{registry}
```

### Retrieve Metric Values

```bash
# Get storage used
az monitor metrics list \
  --resource /subscriptions/{sub}/resourceGroups/{rg}/providers/Microsoft.ContainerRegistry/registries/{registry} \
  --metric "StorageUsed" \
  --interval PT1H

# Get push/pull counts
az monitor metrics list \
  --resource /subscriptions/{sub}/resourceGroups/{rg}/providers/Microsoft.ContainerRegistry/registries/{registry} \
  --metric "SuccessfulPullCount" \
  --interval PT1H \
  --start-time 2024-01-01T00:00:00Z \
  --end-time 2024-01-02T00:00:00Z
```

## Diagnostic Settings Commands

### List Diagnostic Settings

```bash
az monitor diagnostic-settings list \
  --resource /subscriptions/{sub}/resourceGroups/{rg}/providers/Microsoft.ContainerRegistry/registries/{registry}
```

### Create Diagnostic Setting

```bash
# Send logs to Log Analytics
az monitor diagnostic-settings create \
  --name "acr-diagnostics" \
  --resource /subscriptions/{sub}/resourceGroups/{rg}/providers/Microsoft.ContainerRegistry/registries/{registry} \
  --workspace /subscriptions/{sub}/resourceGroups/{rg}/providers/Microsoft.OperationalInsights/workspaces/{workspace} \
  --logs '[{"category":"ContainerRegistryLoginEvents","enabled":true},{"category":"ContainerRegistryRepositoryEvents","enabled":true}]'

# Send logs to Storage Account
az monitor diagnostic-settings create \
  --name "acr-archive" \
  --resource /subscriptions/{sub}/resourceGroups/{rg}/providers/Microsoft.ContainerRegistry/registries/{registry} \
  --storage-account /subscriptions/{sub}/resourceGroups/{rg}/providers/Microsoft.Storage/storageAccounts/{storage} \
  --logs '[{"category":"ContainerRegistryRepositoryEvents","enabled":true,"retentionPolicy":{"enabled":true,"days":90}}]'
```

### Delete Diagnostic Setting

```bash
az monitor diagnostic-settings delete \
  --name "acr-diagnostics" \
  --resource /subscriptions/{sub}/resourceGroups/{rg}/providers/Microsoft.ContainerRegistry/registries/{registry}
```

## Task Logs Commands

### View Task Run Logs

```bash
# View log for specific run ID
az acr task logs --registry myregistry --run-id cf4

# View most recent run for a task
az acr task logs --registry myregistry --task-name mytask

# Save log to local file
az acr task logs --registry myregistry --run-id cf4 > ~/tasklogs/cf4.log
```

### List Task Runs

```bash
# List all task runs
az acr task list-runs --registry myregistry --output table

# List runs for specific task
az acr task list-runs --registry myregistry --task-name mytask --output table

# Show run details
az acr task show-run --registry myregistry --run-id cf4
```

## Activity Log Commands

### View Activity Log

```bash
# View activity log for registry
az monitor activity-log list \
  --resource-id /subscriptions/{sub}/resourceGroups/{rg}/providers/Microsoft.ContainerRegistry/registries/{registry} \
  --start-time 2024-01-01T00:00:00Z \
  --output table

# Filter by operation type
az monitor activity-log list \
  --resource-id /subscriptions/{sub}/resourceGroups/{rg}/providers/Microsoft.ContainerRegistry/registries/{registry} \
  --start-time 2024-01-01T00:00:00Z \
  --filter "operationName eq 'Microsoft.ContainerRegistry/registries/delete'"
```

## Policy Compliance Commands

### List Policy Assignments

```bash
# List ACR-related policy assignments
az policy assignment list \
  --query "[?contains(displayName,'Container Registries')].{name:displayName, ID:id}" \
  --output table
```

### Check Policy Compliance

```bash
# Get compliance state for all resources under policy
az policy state list --resource <policyID>

# Get compliance for specific registry
az policy state list \
  --resource myregistry \
  --namespace Microsoft.ContainerRegistry \
  --resource-type registries \
  --resource-group myresourcegroup
```

## Network Diagnostic Commands

### DNS Lookup

```bash
# Linux/Mac
nslookup myregistry.azurecr.io
dig myregistry.azurecr.io

# Test private endpoint resolution
nslookup myregistry.privatelink.azurecr.io
```

### Connectivity Test

```bash
# Test HTTPS endpoint
curl -v https://myregistry.azurecr.io/v2/

# Test with authentication
curl -v https://myregistry.azurecr.io/v2/ \
  -H "Authorization: Bearer <token>"

# Test data endpoint
curl -v https://myregistry.eastus.data.azurecr.io/
```

### NSG and Firewall Rules

```bash
# List NSG rules
az network nsg rule list --nsg-name myNSG --resource-group myRG --output table

# Show ACR firewall rules
az acr show --name myregistry --query networkRuleSet --output json
```

## Role Assignment Commands

```bash
# List role assignments on registry
az role assignment list \
  --scope /subscriptions/{sub}/resourceGroups/{rg}/providers/Microsoft.ContainerRegistry/registries/{registry} \
  --output table

# Show current user/identity permissions
az role assignment list --assignee $(az ad signed-in-user show --query id -o tsv) --output table
```

## Repository Diagnostic Commands

### List Repositories and Images

```bash
# List repositories
az acr repository list --name myregistry --output table

# Show repository details
az acr repository show --name myregistry --repository myrepo

# List tags
az acr repository show-tags --name myregistry --repository myrepo --output table

# Show manifest
az acr repository show-manifests --name myregistry --repository myrepo --output table
```

### Image Information

```bash
# Get image digest
az acr repository show-manifests --name myregistry --repository myrepo \
  --query "[?tags[?contains(@,'latest')]].digest" --output tsv

# Show image metadata
az acr manifest show-metadata --registry myregistry --name myrepo:latest
```

## Docker Diagnostic Commands

### Docker Environment Check

```bash
# Check Docker daemon
docker info

# Check Docker version
docker --version

# Test Docker pull
docker pull mcr.microsoft.com/hello-world

# Check Docker credentials
docker login myregistry.azurecr.io

# View stored credentials
cat ~/.docker/config.json | jq '.auths'
```

### Docker Login Issues

```bash
# Login with Azure CLI token
az acr login --name myregistry

# Login with token exposed (no Docker daemon)
az acr login --name myregistry --expose-token

# Get access token manually
az acr login --name myregistry --expose-token --output tsv --query accessToken
```

## Log Analytics Queries (KQL)

Run these in Azure Portal Log Analytics:

### Recent Login Events
```kusto
ContainerRegistryLoginEvents
| where TimeGenerated > ago(24h)
| project TimeGenerated, Identity, CallerIpAddress, ResultDescription
| order by TimeGenerated desc
```

### Failed Operations
```kusto
ContainerRegistryRepositoryEvents
| where ResultDescription contains "40"
| project TimeGenerated, OperationName, Repository, ResultDescription
```

### Authentication Failures
```kusto
ContainerRegistryLoginEvents
| where ResultDescription != "200"
| project TimeGenerated, Identity, CallerIpAddress, ResultDescription
```

## Teleport Logs (AKS)

For artifact streaming/teleport diagnostics on AKS:

```bash
# SSH to node and check journald
journalctl -n all -u teleportd

# Create sidecar pod for log collection
kubectl apply -f teleport-logs-pod.yaml
kubectl logs teleport-logs > ./teleport-daemon.log

# Check Kubernetes events
kubectl get events
kubectl describe pod <pod-name>
```

## Troubleshooting Workflow

```bash
# 1. Start with health check
az acr check-health --name myregistry --ignore-errors --yes

# 2. Check registry exists and accessible
az acr show --name myregistry --output table

# 3. Verify network configuration
az acr show --query networkRuleSet --name myregistry

# 4. Check role assignments
az role assignment list --scope /subscriptions/{sub}/resourceGroups/{rg}/providers/Microsoft.ContainerRegistry/registries/{registry}

# 5. Test connectivity
curl -v https://myregistry.azurecr.io/v2/

# 6. Check logs in Log Analytics (if configured)
# Use KQL queries in Azure Portal

# 7. Review activity log for management operations
az monitor activity-log list --resource-id /subscriptions/{sub}/resourceGroups/{rg}/providers/Microsoft.ContainerRegistry/registries/{registry}
```

## Source References

- `/submodules/azure-management-docs/articles/container-registry/container-registry-check-health.md`
- `/submodules/azure-management-docs/articles/container-registry/container-registry-health-error-reference.md`
- `/submodules/azure-management-docs/articles/container-registry/monitor-container-registry.md`
- `/submodules/azure-management-docs/articles/container-registry/container-registry-troubleshoot-login-authn-authz.md`
- `/submodules/azure-management-docs/articles/container-registry/container-registry-troubleshoot-access.md`
- `/submodules/azure-management-docs/articles/container-registry/container-registry-troubleshoot-performance.md`
- `/submodules/acr/docs/Troubleshooting Guide.md`
- `/submodules/acr/docs/teleport/collecting-teleportd-logs-aks.md`

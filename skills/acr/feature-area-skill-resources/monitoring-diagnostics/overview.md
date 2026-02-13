# ACR Monitoring & Diagnostics Skill

This skill provides comprehensive knowledge about Azure Container Registry monitoring and diagnostics.

## When to Use This Skill

Use this skill when answering questions about:
- Azure Monitor integration
- Metrics and logs
- Health checks
- Azure Policy compliance

## Azure Monitor Integration

### Enable Diagnostic Logging
```bash
az monitor diagnostic-settings create \
  --name acr-diagnostics \
  --resource $(az acr show --name myregistry --query id -o tsv) \
  --logs '[{"category":"ContainerRegistryLoginEvents","enabled":true},{"category":"ContainerRegistryRepositoryEvents","enabled":true}]' \
  --workspace $(az monitor log-analytics workspace show --workspace-name myworkspace --resource-group myRG --query id -o tsv)
```

## Metrics

### Available Metrics
| Metric | Description |
|--------|-------------|
| `StorageUsed` | Storage used in bytes |
| `SuccessfulPullCount` | Successful pull operations |
| `SuccessfulPushCount` | Successful push operations |
| `TotalPullCount` | Total pull attempts |
| `TotalPushCount` | Total push attempts |
| `RunDuration` | ACR Task run duration |

### View Metrics
```bash
az monitor metrics list \
  --resource $(az acr show --name myregistry --query id -o tsv) \
  --metric StorageUsed \
  --interval PT1H
```

### Create Alert
```bash
az monitor metrics alert create \
  --name storage-alert \
  --resource-group myRG \
  --scopes $(az acr show --name myregistry --query id -o tsv) \
  --condition "avg StorageUsed > 400000000000" \
  --description "Registry storage above 400GB"
```

## Logs

### Log Categories
| Category | Description |
|----------|-------------|
| `ContainerRegistryLoginEvents` | Authentication events |
| `ContainerRegistryRepositoryEvents` | Push, pull, delete events |

### KQL Queries

**Authentication failures:**
```kusto
ContainerRegistryLoginEvents
| where ResultDescription != "200"
| project TimeGenerated, Identity, LoginServer, ResultDescription
| order by TimeGenerated desc
```

**Push/pull activity:**
```kusto
ContainerRegistryRepositoryEvents
| where OperationName in ("Push", "Pull")
| summarize count() by OperationName, Repository, bin(TimeGenerated, 1h)
```

**Top repositories by size:**
```kusto
ContainerRegistryRepositoryEvents
| where OperationName == "Push"
| summarize TotalSize = sum(MediaSize) by Repository
| top 10 by TotalSize desc
```

## Health Check

### Run Health Check
```bash
# Check local environment
az acr check-health

# Check environment and registry
az acr check-health --name myregistry

# Check with VNet
az acr check-health --name myregistry --vnet myVNet
```

### Common Error Codes
| Error | Cause | Solution |
|-------|-------|----------|
| `DOCKER_COMMAND_ERROR` | Docker not running | Start Docker |
| `CONNECTIVITY_DNS_ERROR` | DNS failure | Check network |
| `CONNECTIVITY_REFRESH_TOKEN_ERROR` | Auth expired | `az acr login` |
| `CONNECTIVITY_FORBIDDEN` | No permission | Check RBAC |
| `HELM_COMMAND_ERROR` | Helm not found | Install Helm |

## Azure Policy

### Built-in Policies
| Policy | Effect |
|--------|--------|
| Require CMK encryption | Deny |
| Disable public network | Deny |
| Require private endpoint | Audit |
| Disable admin account | Audit |

### Assign Policy
```bash
az policy assignment create \
  --name require-private-endpoint \
  --policy "container-registries-should-use-private-link" \
  --scope /subscriptions/{sub}/resourceGroups/{rg}
```

### Check Compliance
```bash
az policy state list \
  --resource $(az acr show --name myregistry --query id -o tsv)
```

## Task Logs

### View Task Run Logs
```bash
# List runs
az acr task list-runs --registry myregistry -o table

# View specific run
az acr task logs --registry myregistry --run-id abc123

# Stream live logs
az acr build --registry myregistry --image myapp:v1 .
# Logs stream automatically
```

### Task Log Retention
- Default: 30 days
- CMK registries: 24 hours

## Diagnostic Commands Summary

```bash
# Registry info
az acr show --name myregistry

# Usage statistics
az acr show-usage --name myregistry -o table

# Network config
az acr show --name myregistry --query networkRuleSet

# Encryption status
az acr show --name myregistry --query encryption

# Check endpoints
az acr show-endpoints --name myregistry
```

## Reference Documentation

- **Investigation reports**: `agent-investigation-reports/acr-features/monitoring-diagnostics/`
- **MS Learn docs**: `azure-management-docs/articles/container-registry/monitor-container-registry.md`

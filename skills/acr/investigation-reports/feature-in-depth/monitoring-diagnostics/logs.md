# ACR Logs Reference

## Overview

Azure Container Registry provides comprehensive logging capabilities through Azure Monitor resource logs and activity logs. These logs enable diagnostic evaluation, auditing, security analysis, and troubleshooting of registry operations.

## Log Categories

### Resource Logs

Resource logs are **NOT collected by default**. You must create a diagnostic setting to collect and route them.

#### ContainerRegistryRepositoryEvents

Records operations on images and artifacts in registry repositories.

| Field | Description |
|-------|-------------|
| TimeGenerated | Event timestamp |
| LoginServer | Registry login server URL |
| OperationName | Type of operation (Push, Pull, Delete, Untag) |
| Repository | Target repository name |
| Tag | Image tag (if applicable) |
| Digest | Content digest |
| Identity | User or service principal identity |
| CallerIpAddress | Source IP address |
| DurationMs | Operation duration in milliseconds |
| Region | Azure region |
| ResultType | Success or failure status |
| ResultDescription | HTTP status code or error |
| CorrelationId | Operation correlation ID |

**Logged Operations:**
- Push (image/manifest upload)
- Pull (image/manifest download)
- Untag (tag removal)
- Delete (repository or manifest deletion)
- Purge tag (retention policy triggered)
- Purge manifest (retention policy triggered)

**Note:** Purge events are only logged when a registry retention policy is configured.

#### ContainerRegistryLoginEvents

Records authentication events and status.

| Field | Description |
|-------|-------------|
| TimeGenerated | Event timestamp |
| LoginServer | Registry login server |
| Identity | Authenticating identity |
| CallerIpAddress | Client IP address |
| ResultType | Authentication result |
| ResultDescription | HTTP status or error code |
| UserAgent | Client user agent string |

### Activity Log

The Activity log captures subscription-level management events.

| Operation | Description |
|-----------|-------------|
| Create or Update Container Registry | Registry creation or property update |
| Delete Container Registry | Registry deletion |
| List Container Registry Login Credentials | Admin credential access |
| Import Image | Image import operations |
| Create Role Assignment | RBAC role assignment |
| Regenerate Login Credentials | Credential regeneration |
| Update Replication | Geo-replication changes |
| Delete Replication | Replica removal |

## Log Analytics Tables

| Table | Data Source |
|-------|-------------|
| AzureActivity | Azure Activity Log |
| AzureMetrics | Platform metrics |
| ContainerRegistryLoginEvents | Authentication logs |
| ContainerRegistryRepositoryEvents | Repository operation logs |

## Task Run Logs

ACR Tasks generates execution logs stored separately from diagnostic logs.

### Viewing Streamed Logs

When triggering tasks manually, logs stream to console:

```bash
az acr run --registry myregistry \
  --cmd '$Registry/samples/hello-world:v1' /dev/null
```

### Viewing Stored Logs

Logs stored for 30 days by default.

**Azure Portal:**
1. Navigate to registry
2. Select **Tasks** > **Runs**
3. Select a **Run Id** to view logs

**Azure CLI:**
```bash
# View log for specific run ID
az acr task logs --registry myregistry --run-id cf4

# View log for most recent run of a task
az acr task logs --registry myregistry --task-name mytask
```

### Alternative Log Storage

Export task logs to external storage:

```bash
# Save to local file
az acr task logs --registry myregistry \
  --run-id cf4 > ~/tasklogs/cf4.log

# Upload to Azure Storage (separate step)
az storage blob upload --account-name mystorageaccount \
  --container-name logs --file ~/tasklogs/cf4.log
```

## Configuring Diagnostic Settings

### Azure Portal

1. Navigate to registry
2. Select **Diagnostic settings** under **Monitoring**
3. Click **Add diagnostic setting**
4. Name the setting
5. Select log categories:
   - ContainerRegistryLoginEvents
   - ContainerRegistryRepositoryEvents
6. Choose destinations:
   - Log Analytics workspace
   - Storage account
   - Event hub
7. Save

### Azure CLI

```bash
# Create diagnostic setting for Log Analytics
az monitor diagnostic-settings create \
  --name "acr-diagnostics" \
  --resource "/subscriptions/{sub}/resourceGroups/{rg}/providers/Microsoft.ContainerRegistry/registries/{registry}" \
  --workspace "/subscriptions/{sub}/resourceGroups/{rg}/providers/Microsoft.OperationalInsights/workspaces/{workspace}" \
  --logs '[{"category":"ContainerRegistryLoginEvents","enabled":true},{"category":"ContainerRegistryRepositoryEvents","enabled":true}]'
```

### ARM Template

```json
{
  "type": "Microsoft.Insights/diagnosticSettings",
  "apiVersion": "2021-05-01-preview",
  "scope": "[resourceId('Microsoft.ContainerRegistry/registries', parameters('registryName'))]",
  "name": "acr-diagnostics",
  "properties": {
    "workspaceId": "[parameters('workspaceId')]",
    "logs": [
      {
        "category": "ContainerRegistryLoginEvents",
        "enabled": true
      },
      {
        "category": "ContainerRegistryRepositoryEvents",
        "enabled": true
      }
    ],
    "metrics": [
      {
        "category": "AllMetrics",
        "enabled": true
      }
    ]
  }
}
```

## Sample KQL Queries

### Recent Repository Events
```kusto
ContainerRegistryRepositoryEvents
| where TimeGenerated > ago(1d)
| project TimeGenerated, OperationName, Repository, Tag, Identity, ResultType
| order by TimeGenerated desc
```

### Authentication Failures
```kusto
ContainerRegistryLoginEvents
| where ResultDescription != "200"
| project TimeGenerated, Identity, CallerIpAddress, ResultDescription
| order by TimeGenerated desc
```

### Repository Deletion Audit
```kusto
ContainerRegistryRepositoryEvents
| where OperationName contains "Delete"
| project TimeGenerated, LoginServer, OperationName, Repository, Identity, CallerIpAddress
```

### Tag Removal Audit
```kusto
ContainerRegistryRepositoryEvents
| where OperationName contains "Untag"
| project TimeGenerated, LoginServer, OperationName, Repository, Tag, Identity, CallerIpAddress
```

### Failed Operations (4xx errors)
```kusto
ContainerRegistryRepositoryEvents
| where ResultDescription contains "40"
| project TimeGenerated, OperationName, Repository, Tag, ResultDescription
```

### Most Active Users
```kusto
ContainerRegistryRepositoryEvents
| where TimeGenerated > ago(7d)
| summarize OperationCount = count() by Identity
| order by OperationCount desc
| take 10
```

### Most Accessed Repositories
```kusto
ContainerRegistryRepositoryEvents
| where TimeGenerated > ago(7d)
| where OperationName == "Pull"
| summarize PullCount = count() by Repository
| order by PullCount desc
| take 10
```

### Operations by Region (Geo-replicated)
```kusto
ContainerRegistryRepositoryEvents
| where TimeGenerated > ago(1d)
| summarize count() by Region, OperationName
| order by Region, count_ desc
```

### Login Attempts by IP
```kusto
ContainerRegistryLoginEvents
| where TimeGenerated > ago(1d)
| summarize AttemptCount = count(), SuccessCount = countif(ResultDescription == "200") by CallerIpAddress
| extend FailureCount = AttemptCount - SuccessCount
| order by FailureCount desc
```

## Log Retention

| Log Type | Default Retention | Configurable |
|----------|-------------------|--------------|
| Task Run Logs | 30 days | No |
| Resource Logs | Depends on destination | Yes |
| Activity Log | 90 days | Via export |

### Configuring Retention

**Log Analytics:**
- Configure retention per table or workspace
- Default: 30-730 days

**Storage Account:**
- Configure lifecycle management policies
- Indefinite retention possible

## Troubleshooting with Logs

### Authentication Issues
Query `ContainerRegistryLoginEvents` for:
- Failed login attempts
- Unexpected IP addresses
- Identity mismatches

### Missing Images
Query `ContainerRegistryRepositoryEvents` for:
- Delete operations on repository
- Untag operations
- Purge events from retention policy

### Performance Issues
Query `ContainerRegistryRepositoryEvents` for:
- High DurationMs values
- Specific regions with latency
- Rate limiting (429 errors)

## Best Practices

1. **Enable diagnostic settings early** - Logs not collected by default
2. **Use Log Analytics** - Enables powerful KQL queries
3. **Set up log-based alerts** - Detect security issues
4. **Archive to storage** - Compliance and long-term retention
5. **Review regularly** - Audit access patterns
6. **Correlate with Activity Log** - Full picture of operations

## Source References

- `/submodules/azure-management-docs/articles/container-registry/monitor-container-registry.md`
- `/submodules/azure-management-docs/articles/container-registry/monitor-container-registry-reference.md`
- `/submodules/azure-management-docs/articles/container-registry/container-registry-tasks-logs.md`
- `/submodules/azure-management-docs/articles/container-registry/container-registry-troubleshoot-login-authn-authz.md`

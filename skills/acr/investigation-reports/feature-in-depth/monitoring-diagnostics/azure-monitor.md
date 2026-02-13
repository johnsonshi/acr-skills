# Azure Monitor Integration with ACR

## Overview

Azure Container Registry integrates with Azure Monitor to provide comprehensive monitoring capabilities including metrics collection, log analysis, alerting, and diagnostic insights. This integration enables DevOps engineers to analyze and respond to registry-related issues.

## Monitor Overview in Azure Portal

The **Overview** page in the Azure portal for each registry displays:
- Recent resource usage
- Push and pull operation counts
- Storage utilization trends
- Activity summary for the last 30 days

## Monitoring Data Collection

### Platform Metrics
- Collected and stored automatically
- Can be routed to other locations via diagnostic settings
- Available for analysis in Metrics Explorer

### Resource Logs
- **NOT collected by default** - requires diagnostic setting creation
- Must be explicitly routed to storage destinations
- Categories include authentication and repository events

### Activity Log
- Subscription-level events
- Management operations on registry resources
- Can be routed to Azure Monitor Logs for complex queries

## Collection and Routing

### Creating Diagnostic Settings

Navigate to your registry in the Azure portal:
1. Select **Diagnostic settings** under **Monitoring**
2. Click **Add diagnostic setting**
3. Select log categories to collect
4. Choose destinations (Log Analytics, Storage, Event Hubs)

### Routing Options

| Destination | Use Case |
|-------------|----------|
| Log Analytics Workspace | Query and analyze logs with KQL |
| Azure Storage Account | Long-term archival and compliance |
| Event Hubs | Stream to external systems (SIEM) |
| Partner Solutions | Third-party monitoring tools |

## Analyzing Metrics

### Metrics Explorer Access

Two ways to access:
1. **Azure Monitor menu** > **Metrics** > Select ACR resource
2. **Registry menu** > **Monitoring** > **Metrics**

### Available Metrics

| Metric | Description | Dimensions |
|--------|-------------|------------|
| Storage used | Total storage consumed | Geolocation |
| Successful Push Count | Number of successful pushes | None |
| Successful Pull Count | Number of successful pulls | None |
| Total Push Count | Total push attempts | None |
| Total Pull Count | Total pull attempts | None |
| Run Duration | ACR Task run duration | None |
| Agent Pool CPU Time | CPU time for agent pools | None |

### Azure CLI Commands

List metric definitions:
```bash
az monitor metrics list-definitions \
  --resource /subscriptions/{sub}/resourceGroups/{rg}/providers/Microsoft.ContainerRegistry/registries/{registry}
```

Retrieve metric values:
```bash
az monitor metrics list \
  --resource /subscriptions/{sub}/resourceGroups/{rg}/providers/Microsoft.ContainerRegistry/registries/{registry} \
  --metric "StorageUsed"
```

### REST API

- **List Definitions**: `/providers/microsoft.insights/metricDefinitions`
- **Get Metrics**: `/providers/microsoft.insights/metrics`

## Analyzing Logs

### Log Analytics Tables

| Table | Description |
|-------|-------------|
| AzureActivity | Subscription-level events |
| AzureMetrics | Metric data from registry |
| ContainerRegistryLoginEvents | Authentication events and status |
| ContainerRegistryRepositoryEvents | Repository operations (push, pull, delete) |

### Common Log Schema

All resource logs share common fields:
- TimeGenerated
- ResourceId
- Category
- OperationName
- ResultType
- CorrelationId

ACR-specific fields vary by event type.

## Sample Kusto Queries

### Recent Repository Events (24 hours)
```kusto
ContainerRegistryRepositoryEvents
| where TimeGenerated > ago(1d)
```

### Error Events (Last Hour)
```kusto
union Event, Syslog
| where TimeGenerated > ago(1h)
| where EventLevelName == "Error"
    or SeverityLevel== "err"
```

### 100 Most Recent Registry Events
```kusto
ContainerRegistryRepositoryEvents
| union ContainerRegistryLoginEvents
| top 100 by TimeGenerated
| project TimeGenerated, LoginServer, OperationName, Identity, Repository, DurationMs, Region, ResultType
```

### Identify Who Deleted Repository
```kusto
ContainerRegistryRepositoryEvents
| where OperationName contains "Delete"
| project LoginServer, OperationName, Repository, Identity, CallerIpAddress
```

### Identify Who Deleted Tag
```kusto
ContainerRegistryRepositoryEvents
| where OperationName contains "Untag"
| project LoginServer, OperationName, Repository, Tag, Identity, CallerIpAddress
```

### Repository-Level Failures
```kusto
ContainerRegistryRepositoryEvents
| where ResultDescription contains "40"
| project TimeGenerated, OperationName, Repository, Tag, ResultDescription
```

### Authentication Failures
```kusto
ContainerRegistryLoginEvents
| where ResultDescription != "200"
| project TimeGenerated, Identity, CallerIpAddress, ResultDescription
```

## Alerting

### Creating Alert Rules

1. Navigate to registry in Azure portal
2. Select **Metrics** under **Monitoring**
3. Select desired metric
4. Click **New alert rule**
5. Configure condition, action group, and severity

### Example: Storage Alert

Create alert when storage exceeds threshold:

1. **Signal**: Storage used
2. **Operator**: Greater than
3. **Aggregation**: Average
4. **Threshold**: 5 GB
5. **Action**: Email notification

### Suggested Alert Rules

| Alert Type | Condition | Description |
|------------|-----------|-------------|
| Metric | Storage used > 5 GB | Alert on high storage usage |
| Metric | Failed push count > 10 | Alert on push failures |
| Log | Authentication failures | Alert on login issues |
| Activity Log | Registry deleted | Alert on registry deletion |

## Data Storage

- Metrics are stored for 93 days by default
- Resource logs stored based on diagnostic setting configuration
- Activity logs retained for 90 days
- Task run logs retained for 30 days by default

## Azure Advisor Integration

Azure Advisor provides:
- Performance recommendations
- Cost optimization suggestions
- Security and reliability insights
- Best practice guidance

## Source References

- `/submodules/azure-management-docs/articles/container-registry/monitor-container-registry.md`
- `/submodules/azure-management-docs/articles/container-registry/monitor-container-registry-reference.md`

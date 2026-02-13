# ACR Metrics Reference

## Overview

Azure Container Registry exposes platform metrics through Azure Monitor that can be used to monitor registry health, usage patterns, and performance. These metrics are collected automatically and can be analyzed using Metrics Explorer or programmatic APIs.

## Supported Metrics for Microsoft.ContainerRegistry/registries

### Storage Metrics

| Metric Name | Display Name | Unit | Description |
|-------------|--------------|------|-------------|
| StorageUsed | Storage used | Bytes | Total storage consumed by the registry |

**Important Notes:**
- Due to layer sharing, registry **Storage used** may be less than the sum of storage for individual repositories
- When deleting a repository or tag, only storage for manifest files and unique layers is recovered
- Shared layers referenced by other images are not freed

### Operation Metrics

| Metric Name | Display Name | Unit | Description |
|-------------|--------------|------|-------------|
| SuccessfulPullCount | Successful Pull Count | Count | Number of successful image pulls |
| SuccessfulPushCount | Successful Push Count | Count | Number of successful image pushes |
| TotalPullCount | Total Pull Count | Count | Total number of image pull attempts |
| TotalPushCount | Total Push Count | Count | Total number of image push attempts |

### Task Metrics

| Metric Name | Display Name | Unit | Description |
|-------------|--------------|------|-------------|
| RunDuration | Run Duration | Milliseconds | Duration of ACR Task runs |
| AgentPoolCPUTime | Agent Pool CPU Time | Seconds | CPU time used by task agent pools |

## Metric Dimensions

### Geolocation Dimension
- Represents the Azure region for a registry or geo-replica
- Available for geo-replicated registries
- Allows filtering metrics by region

## Accessing Metrics

### Azure Portal - Metrics Explorer

1. Navigate to your registry
2. Select **Metrics** under **Monitoring**
3. Choose the metric to display
4. Configure aggregation and time range
5. Apply filters and splitting as needed

### Azure CLI

**List metric definitions:**
```bash
az monitor metrics list-definitions \
  --resource /subscriptions/<subscription-id>/resourceGroups/<resource-group>/providers/Microsoft.ContainerRegistry/registries/<registry-name>
```

**Retrieve specific metric values:**
```bash
az monitor metrics list \
  --resource /subscriptions/<subscription-id>/resourceGroups/<resource-group>/providers/Microsoft.ContainerRegistry/registries/<registry-name> \
  --metric "StorageUsed" \
  --interval PT1H \
  --start-time 2024-01-01T00:00:00Z \
  --end-time 2024-01-02T00:00:00Z
```

**Example - Get storage used:**
```bash
az monitor metrics list \
  --resource "/subscriptions/xxx/resourceGroups/myRG/providers/Microsoft.ContainerRegistry/registries/myregistry" \
  --metric "StorageUsed" \
  --aggregation Average \
  --interval PT1H
```

### REST API

**List metric definitions:**
```
GET https://management.azure.com/{resourceUri}/providers/microsoft.insights/metricDefinitions?api-version=2018-01-01
```

**Retrieve metric values:**
```
GET https://management.azure.com/{resourceUri}/providers/microsoft.insights/metrics?api-version=2018-01-01&metricnames=StorageUsed&aggregation=Average&interval=PT1H
```

## Metric Aggregation Types

| Aggregation | Description | Use Case |
|-------------|-------------|----------|
| Average | Mean value over time period | Storage trends |
| Total | Sum of all values | Operation counts |
| Count | Number of samples | Event frequency |
| Maximum | Highest value | Peak usage |
| Minimum | Lowest value | Baseline analysis |

## Metrics by SKU Tier

| Metric | Basic | Standard | Premium |
|--------|-------|----------|---------|
| Storage Used | Yes | Yes | Yes |
| Push/Pull Counts | Yes | Yes | Yes |
| Run Duration | Yes | Yes | Yes |
| Agent Pool CPU Time | No | No | Yes |
| Geo-replication metrics | No | No | Yes |

## Service Tier Limits

Metrics can help track against service tier limits:

| Tier | Max Storage | ReadOps/min | WriteOps/min |
|------|-------------|-------------|--------------|
| Basic | 10 GB | 1,000 | 100 |
| Standard | 100 GB | 3,000 | 500 |
| Premium | 500 GB+ | 10,000 | 2,000 |

## Creating Metric-Based Alerts

### Storage Alert Example

1. Go to registry > Metrics
2. Select **Storage used** metric
3. Click **New alert rule**
4. Configure:
   - Signal: Storage used
   - Operator: Greater than
   - Threshold: 5 GB (or appropriate value)
   - Aggregation: Average
   - Evaluation frequency: 5 minutes
5. Add action group for notifications
6. Create alert rule

### Push Failure Alert

```json
{
  "criteria": {
    "metric": "TotalPushCount - SuccessfulPushCount",
    "operator": "GreaterThan",
    "threshold": 10,
    "aggregation": "Total",
    "windowSize": "PT5M"
  }
}
```

## Exporting Metrics

### To Log Analytics

Configure diagnostic setting to send metrics to Log Analytics workspace:
- Enables long-term retention
- Allows KQL queries on metric data
- Correlate with resource logs

### To Storage Account

- Archive for compliance
- Long-term trend analysis
- Cost-effective retention

### To Event Hubs

- Stream to external systems
- Real-time monitoring integration
- SIEM ingestion

## Best Practices

1. **Set up baseline monitoring**: Establish normal patterns for your registry
2. **Configure storage alerts**: Prevent quota exhaustion
3. **Monitor operation rates**: Detect unusual activity
4. **Use geo-replication metrics**: For multi-region deployments
5. **Export to Log Analytics**: For advanced analysis and correlation
6. **Review metrics regularly**: Identify optimization opportunities

## Troubleshooting with Metrics

| Issue | Relevant Metrics | Action |
|-------|------------------|--------|
| Slow pulls | TotalPullCount, SuccessfulPullCount | Check network, upgrade tier |
| Push failures | TotalPushCount vs SuccessfulPushCount | Check authentication, quotas |
| Storage growing | StorageUsed | Implement retention policy |
| Task timeouts | RunDuration | Optimize tasks, increase resources |

## Source References

- `/submodules/azure-management-docs/articles/container-registry/monitor-container-registry-reference.md`
- `/submodules/azure-management-docs/articles/container-registry/monitor-container-registry.md`

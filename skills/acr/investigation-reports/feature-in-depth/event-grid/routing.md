# Azure Container Registry Event Grid - Routing and Filtering

## Overview

Event Grid provides powerful routing and filtering capabilities to direct ACR events to specific endpoints based on event properties. This document covers all filtering options, advanced filtering, and routing patterns.

## Event Routing Architecture

```
+------------------+     +------------------+     +-------------------+
| ACR Registry     | --> | Event Grid Topic | --> | Subscription 1    |
| Events:          |     |                  |     | (Filter: Push)    |
| - ImagePushed    |     | Routing Logic:   |     +-------------------+
| - ImageDeleted   |     | - Event Type     |
| - ChartPushed    |     | - Subject Filter | --> +-------------------+
| - ChartDeleted   |     | - Advanced Filter|     | Subscription 2    |
+------------------+     +------------------+     | (Filter: Delete)  |
                                                  +-------------------+

                                            --> +-------------------+
                                                | Subscription 3    |
                                                | (Filter: prod/*)  |
                                                +-------------------+
```

## Filtering Methods

### 1. Event Type Filtering

Filter events by their type (most common filtering method).

#### Subscribe to Single Event Type

```bash
# Only image push events
az eventgrid event-subscription create \
    --name push-only \
    --source-resource-id $ACR_REGISTRY_ID \
    --endpoint $ENDPOINT \
    --included-event-types Microsoft.ContainerRegistry.ImagePushed
```

#### Subscribe to Multiple Event Types

```bash
# All image events (push and delete)
az eventgrid event-subscription create \
    --name image-events \
    --source-resource-id $ACR_REGISTRY_ID \
    --endpoint $ENDPOINT \
    --included-event-types \
        Microsoft.ContainerRegistry.ImagePushed \
        Microsoft.ContainerRegistry.ImageDeleted
```

#### Subscribe to All Chart Events

```bash
az eventgrid event-subscription create \
    --name chart-events \
    --source-resource-id $ACR_REGISTRY_ID \
    --endpoint $ENDPOINT \
    --included-event-types \
        Microsoft.ContainerRegistry.ChartPushed \
        Microsoft.ContainerRegistry.ChartDeleted
```

### 2. Subject Filtering

Filter events based on the subject field (typically `repository:tag`).

#### Filter by Repository Prefix

Route events for images in specific namespaces:

```bash
# Events for images starting with "production/"
az eventgrid event-subscription create \
    --name prod-images \
    --source-resource-id $ACR_REGISTRY_ID \
    --endpoint $PROD_ENDPOINT \
    --subject-begins-with "production/"

# Events for images starting with "staging/"
az eventgrid event-subscription create \
    --name staging-images \
    --source-resource-id $ACR_REGISTRY_ID \
    --endpoint $STAGING_ENDPOINT \
    --subject-begins-with "staging/"
```

#### Filter by Tag Suffix

Route events for specific tags:

```bash
# Events for :latest tags only
az eventgrid event-subscription create \
    --name latest-only \
    --source-resource-id $ACR_REGISTRY_ID \
    --endpoint $ENDPOINT \
    --subject-ends-with ":latest"

# Events for version tags (e.g., :v1.0.0)
az eventgrid event-subscription create \
    --name version-tags \
    --source-resource-id $ACR_REGISTRY_ID \
    --endpoint $ENDPOINT \
    --subject-ends-with ":v*"
```

#### Combine Prefix and Suffix

```bash
# Production images with latest tag
az eventgrid event-subscription create \
    --name prod-latest \
    --source-resource-id $ACR_REGISTRY_ID \
    --endpoint $ENDPOINT \
    --subject-begins-with "production/" \
    --subject-ends-with ":latest"
```

### 3. Advanced Filtering

Use advanced filters to filter on event data properties.

#### Filter Operators

| Operator | Description | Example |
|----------|-------------|---------|
| `NumberIn` | Value is in list of numbers | `--advanced-filter data.target.size NumberIn 1000 2000` |
| `NumberNotIn` | Value is not in list | `--advanced-filter data.target.size NumberNotIn 0` |
| `NumberLessThan` | Value < number | `--advanced-filter data.target.size NumberLessThan 1000000` |
| `NumberGreaterThan` | Value > number | `--advanced-filter data.target.size NumberGreaterThan 1000` |
| `NumberLessThanOrEquals` | Value <= number | `--advanced-filter data.target.size NumberLessThanOrEquals 5000` |
| `NumberGreaterThanOrEquals` | Value >= number | `--advanced-filter data.target.size NumberGreaterThanOrEquals 100` |
| `BoolEquals` | Boolean comparison | `--advanced-filter data.isSuccess BoolEquals true` |
| `StringIn` | String is in list | `--advanced-filter data.target.tag StringIn latest stable` |
| `StringNotIn` | String not in list | `--advanced-filter data.target.tag StringNotIn dev test` |
| `StringBeginsWith` | String starts with | `--advanced-filter data.target.repository StringBeginsWith myapp` |
| `StringEndsWith` | String ends with | `--advanced-filter data.target.tag StringEndsWith release` |
| `StringContains` | String contains | `--advanced-filter data.target.repository StringContains api` |
| `IsNullOrUndefined` | Field is null/missing | `--advanced-filter data.connectedregistry IsNullOrUndefined` |
| `IsNotNull` | Field exists and not null | `--advanced-filter data.connectedregistry IsNotNull` |

#### Filter by Repository Name

```bash
# Events for specific repository
az eventgrid event-subscription create \
    --name myapp-events \
    --source-resource-id $ACR_REGISTRY_ID \
    --endpoint $ENDPOINT \
    --advanced-filter data.target.repository StringIn myapp myapp-api myapp-web
```

#### Filter by Image Size

```bash
# Events for large images (> 100MB)
az eventgrid event-subscription create \
    --name large-images \
    --source-resource-id $ACR_REGISTRY_ID \
    --endpoint $ENDPOINT \
    --advanced-filter data.target.size NumberGreaterThan 104857600
```

#### Filter by Tag

```bash
# Events for specific tags
az eventgrid event-subscription create \
    --name release-tags \
    --source-resource-id $ACR_REGISTRY_ID \
    --endpoint $ENDPOINT \
    --advanced-filter data.target.tag StringBeginsWith v
```

#### Filter by Media Type

```bash
# Events for Docker V2 manifests only
az eventgrid event-subscription create \
    --name docker-v2-only \
    --source-resource-id $ACR_REGISTRY_ID \
    --endpoint $ENDPOINT \
    --advanced-filter data.target.mediaType StringIn \
        "application/vnd.docker.distribution.manifest.v2+json"
```

### 4. Connected Registry Filtering

Special filtering for connected registry events.

#### Filter All Connected Registry Events

```bash
# Events from any connected registry
az eventgrid event-subscription create \
    --name all-connected \
    --source-resource-id $ACR_REGISTRY_ID \
    --endpoint $ENDPOINT \
    --advanced-filter data.connectedregistry IsNotNull
```

#### Filter by Specific Connected Registry

```bash
# Events from specific connected registry
az eventgrid event-subscription create \
    --name specific-connected \
    --source-resource-id $ACR_REGISTRY_ID \
    --endpoint $ENDPOINT \
    --advanced-filter data.connectedregistry IsNotNull \
    --advanced-filter data.connectedregistry.name StringIn myconnectedregistry
```

#### Exclude Connected Registry Events

```bash
# Events only from cloud registry (not connected)
az eventgrid event-subscription create \
    --name cloud-only \
    --source-resource-id $ACR_REGISTRY_ID \
    --endpoint $ENDPOINT \
    --advanced-filter data.connectedregistry IsNullOrUndefined
```

## Routing Patterns

### Pattern 1: Environment-Based Routing

Route events to different endpoints based on environment namespace:

```bash
# Development environment
az eventgrid event-subscription create \
    --name dev-routing \
    --source-resource-id $ACR_REGISTRY_ID \
    --endpoint $DEV_FUNCTION_URL \
    --subject-begins-with "dev/"

# Staging environment
az eventgrid event-subscription create \
    --name staging-routing \
    --source-resource-id $ACR_REGISTRY_ID \
    --endpoint $STAGING_FUNCTION_URL \
    --subject-begins-with "staging/"

# Production environment
az eventgrid event-subscription create \
    --name prod-routing \
    --source-resource-id $ACR_REGISTRY_ID \
    --endpoint $PROD_FUNCTION_URL \
    --subject-begins-with "prod/"
```

### Pattern 2: Event Type Separation

Route different event types to different handlers:

```bash
# Push events to deployment trigger
az eventgrid event-subscription create \
    --name deploy-trigger \
    --source-resource-id $ACR_REGISTRY_ID \
    --endpoint $DEPLOY_FUNCTION_URL \
    --included-event-types Microsoft.ContainerRegistry.ImagePushed

# Delete events to cleanup handler
az eventgrid event-subscription create \
    --name cleanup-trigger \
    --source-resource-id $ACR_REGISTRY_ID \
    --endpoint $CLEANUP_FUNCTION_URL \
    --included-event-types Microsoft.ContainerRegistry.ImageDeleted

# All events to audit log
az eventgrid event-subscription create \
    --name audit-log \
    --source-resource-id $ACR_REGISTRY_ID \
    --endpoint $AUDIT_EVENTHUB_URL
```

### Pattern 3: Team-Based Routing

Route events to team-specific endpoints:

```bash
# Team A repositories
az eventgrid event-subscription create \
    --name team-a \
    --source-resource-id $ACR_REGISTRY_ID \
    --endpoint $TEAM_A_ENDPOINT \
    --advanced-filter data.target.repository StringBeginsWith team-a/

# Team B repositories
az eventgrid event-subscription create \
    --name team-b \
    --source-resource-id $ACR_REGISTRY_ID \
    --endpoint $TEAM_B_ENDPOINT \
    --advanced-filter data.target.repository StringBeginsWith team-b/
```

### Pattern 4: Security Scanning Integration

Route push events to vulnerability scanner:

```bash
# Route all pushes to scanner
az eventgrid event-subscription create \
    --name vuln-scanner \
    --source-resource-id $ACR_REGISTRY_ID \
    --endpoint $SCANNER_WEBHOOK \
    --included-event-types Microsoft.ContainerRegistry.ImagePushed

# Route only production pushes to scanner
az eventgrid event-subscription create \
    --name prod-scanner \
    --source-resource-id $ACR_REGISTRY_ID \
    --endpoint $PROD_SCANNER_WEBHOOK \
    --included-event-types Microsoft.ContainerRegistry.ImagePushed \
    --subject-begins-with "production/"
```

### Pattern 5: Multi-Region Sync

Route events to sync handlers in different regions:

```bash
# Sync to West US 2
az eventgrid event-subscription create \
    --name sync-westus2 \
    --source-resource-id $ACR_REGISTRY_ID \
    --endpoint $WESTUS2_SYNC_FUNCTION \
    --included-event-types Microsoft.ContainerRegistry.ImagePushed

# Sync to East US
az eventgrid event-subscription create \
    --name sync-eastus \
    --source-resource-id $ACR_REGISTRY_ID \
    --endpoint $EASTUS_SYNC_FUNCTION \
    --included-event-types Microsoft.ContainerRegistry.ImagePushed
```

## Complex Filtering Examples

### Multiple Advanced Filters (AND Logic)

Multiple `--advanced-filter` flags use AND logic:

```bash
# Production images larger than 50MB with v* tags
az eventgrid event-subscription create \
    --name complex-filter \
    --source-resource-id $ACR_REGISTRY_ID \
    --endpoint $ENDPOINT \
    --subject-begins-with "production/" \
    --advanced-filter data.target.size NumberGreaterThan 52428800 \
    --advanced-filter data.target.tag StringBeginsWith v
```

### Combined Event Type and Subject Filtering

```bash
# Only push events for production latest images
az eventgrid event-subscription create \
    --name prod-latest-push \
    --source-resource-id $ACR_REGISTRY_ID \
    --endpoint $ENDPOINT \
    --included-event-types Microsoft.ContainerRegistry.ImagePushed \
    --subject-begins-with "production/" \
    --subject-ends-with ":latest"
```

## Limitations

1. **Maximum 25 advanced filters** per subscription
2. **String filters** limited to 512 characters per value
3. **Subject filters** support wildcards only with prefix/suffix (not inline)
4. **Array filtering** not directly supported on nested arrays
5. **Case sensitivity**: Subject filtering is case-sensitive by default

## Best Practices

### 1. Use Specific Filters
Start with broad subscriptions and add filters as needed to reduce unnecessary event delivery.

### 2. Combine Filter Types
Use event type + subject + advanced filters together for precise routing.

### 3. Test Filters
Test filter configurations with sample events before production deployment.

### 4. Monitor Filtered Events
Use Event Grid metrics to verify filters are working as expected.

### 5. Document Routing Rules
Maintain documentation of routing patterns for team reference.

## References

- Source: `/submodules/azure-management-docs/articles/container-registry/container-registry-event-grid-quickstart.md`
- Source: `/submodules/acr/docs/preview/connected-registry/quickstart-send-connected-registry-events-to-event-grid.md`

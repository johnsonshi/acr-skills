# Azure Container Registry Event Grid Integration - Feature Overview

## Summary

Azure Container Registry (ACR) integrates with Azure Event Grid to provide a fully managed event routing service for container registry events. This integration enables automated workflows when container images or Helm charts are pushed to or deleted from a registry.

## What is Azure Event Grid?

Azure Event Grid is a fully managed event routing service that uses a publish-subscribe model. It provides uniform event consumption across Azure services and allows you to:

- React to events in near real-time
- Filter events to route only specific events to endpoints
- Scale automatically based on event volume
- Connect Azure services with third-party applications

## Key Features

### 1. Native Azure Integration
- ACR publishes events directly to Azure Event Grid
- No additional configuration beyond creating event subscriptions
- Events are delivered reliably using Event Grid's built-in retry and dead-lettering

### 2. Supported Event Types
ACR emits the following event types to Event Grid:

| Event Type | Description |
|------------|-------------|
| `Microsoft.ContainerRegistry.ImagePushed` | Triggered when a container image is pushed |
| `Microsoft.ContainerRegistry.ImageDeleted` | Triggered when a container image is deleted |
| `Microsoft.ContainerRegistry.ChartPushed` | Triggered when a Helm chart is pushed |
| `Microsoft.ContainerRegistry.ChartDeleted` | Triggered when a Helm chart is deleted |

### 3. Event Destinations
Events can be routed to various Azure services:
- Azure Functions
- Azure Logic Apps
- Azure Automation
- Azure Event Hubs
- Azure Storage Queues
- Custom webhooks (HTTP/HTTPS endpoints)
- Azure Service Bus

### 4. Connected Registry Support
Connected registries (on-premises ACR deployments) can also send events to Event Grid:
- Events from connected registries are proxied through the parent cloud registry
- Events include a `connectedregistry` field identifying the source
- Supports filtering by connected registry name

## Event Grid vs. Webhooks

ACR supports two mechanisms for event notification:

| Feature | Event Grid | Webhooks |
|---------|------------|----------|
| Event Routing | Managed by Azure | Direct to endpoint |
| Retry Logic | Built-in with configurable dead-lettering | Basic retry |
| Event Filtering | Advanced filtering by event type, subject | Basic scope filtering |
| Authentication | Multiple auth options | Custom headers |
| Scalability | Auto-scales | Dependent on endpoint |
| Event Schema | CloudEvents or Event Grid schema | ACR-specific schema |
| Multiple Subscribers | Native support | Requires multiple webhooks |

## Prerequisites

### Resource Provider Registration
The Event Grid resource provider must be registered in your Azure subscription:

```bash
# Register the Event Grid resource provider
az provider register --namespace Microsoft.EventGrid

# Check registration status
az provider show --namespace Microsoft.EventGrid --query "registrationState"
```

### Supported SKUs
Event Grid integration is available on all ACR service tiers:
- Basic
- Standard
- Premium

### Permissions Required
To configure Event Grid subscriptions:
- `Microsoft.EventGrid/eventSubscriptions/write` permission on the registry
- The `Container Registry Contributor and Data Access Configuration Administrator` role includes Event Grid permissions

## Architecture

```
+----------------+     +------------------+     +------------------+
|  Azure ACR     | --> | Azure Event Grid | --> | Event Endpoints  |
|                |     |                  |     |                  |
| - Image Push   |     | - Routing        |     | - Azure Function |
| - Image Delete |     | - Filtering      |     | - Logic App      |
| - Chart Push   |     | - Retry Logic    |     | - Webhook        |
| - Chart Delete |     | - Dead-lettering |     | - Event Hub      |
+----------------+     +------------------+     +------------------+
        |
        v
+------------------+
| Connected        |
| Registries       |
| (On-premises)    |
+------------------+
```

## Use Cases

### 1. Automated CI/CD Pipeline Triggers
Trigger deployments when new images are pushed to the registry.

### 2. Security Scanning Workflows
Initiate vulnerability scans automatically when images are pushed.

### 3. Audit and Compliance
Log all registry events for compliance and auditing purposes.

### 4. Notification Systems
Send alerts when specific images are updated or deleted.

### 5. Cache Invalidation
Invalidate CDN or application caches when container images change.

### 6. Multi-Region Synchronization
Coordinate image deployments across multiple regions.

## Limitations

1. **Event Delivery Latency**: Events are typically delivered within seconds, but may experience delays during high load
2. **Event Size**: Maximum event size is 1 MB
3. **Subscription Limits**: Each Azure subscription has limits on the number of Event Grid subscriptions
4. **Connected Registry Delays**: Events from connected registries may take multiples of the sync interval time to appear

## Related Documentation

- [Events Schema Reference](./events.md)
- [Event Subscriptions](./subscriptions.md)
- [Event Routing and Filtering](./routing.md)
- [CLI Commands](./cli-commands.md)

## References

- Source: `/submodules/azure-management-docs/articles/container-registry/container-registry-event-grid-quickstart.md`
- Source: `/submodules/azure-management-docs/articles/container-registry/container-registry-webhook.md`
- Source: `/submodules/acr/docs/preview/connected-registry/quickstart-send-connected-registry-events-to-event-grid.md`

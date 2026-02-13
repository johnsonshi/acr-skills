# ACR Event Grid Skill

This skill provides comprehensive knowledge about Azure Container Registry Event Grid integration.

## When to Use This Skill

Use this skill when answering questions about:
- Event Grid integration
- Event subscriptions
- Event routing
- Azure Functions triggers

## Overview

Event Grid enables routing ACR events to various Azure services and external endpoints.

**Availability:** All SKUs

## Supported Events

| Event Type | Description |
|------------|-------------|
| `Microsoft.ContainerRegistry.ImagePushed` | Image pushed |
| `Microsoft.ContainerRegistry.ImageDeleted` | Image deleted |
| `Microsoft.ContainerRegistry.ChartPushed` | Helm chart pushed |
| `Microsoft.ContainerRegistry.ChartDeleted` | Helm chart deleted |

## Quick Setup

```bash
# Create Event Grid subscription
az eventgrid event-subscription create \
  --name mysubscription \
  --source-resource-id $(az acr show --name myregistry --query id -o tsv) \
  --endpoint https://myapp.example.com/events \
  --endpoint-type webhook
```

## Event Destinations

| Destination | Command |
|-------------|---------|
| Webhook | `--endpoint-type webhook` |
| Azure Function | `--endpoint-type azurefunction` |
| Event Hub | `--endpoint-type eventhub` |
| Storage Queue | `--endpoint-type storagequeue` |
| Service Bus Queue | `--endpoint-type servicebusqueue` |
| Service Bus Topic | `--endpoint-type servicebustopic` |

### Azure Function Example
```bash
az eventgrid event-subscription create \
  --name mysubscription \
  --source-resource-id $(az acr show --name myregistry --query id -o tsv) \
  --endpoint /subscriptions/{sub}/resourceGroups/{rg}/providers/Microsoft.Web/sites/{funcapp}/functions/{func} \
  --endpoint-type azurefunction
```

## Event Schema

```json
{
  "topic": "/subscriptions/.../providers/Microsoft.ContainerRegistry/registries/myregistry",
  "subject": "myapp:v1",
  "eventType": "Microsoft.ContainerRegistry.ImagePushed",
  "eventTime": "2024-01-15T10:00:00Z",
  "data": {
    "id": "event-id",
    "timestamp": "2024-01-15T10:00:00Z",
    "action": "push",
    "target": {
      "mediaType": "application/vnd.docker.distribution.manifest.v2+json",
      "size": 524,
      "digest": "sha256:abc123...",
      "repository": "myapp",
      "tag": "v1"
    }
  }
}
```

## Filtering

### By Event Type
```bash
az eventgrid event-subscription create \
  --name mysubscription \
  --source-resource-id $(az acr show --name myregistry --query id -o tsv) \
  --endpoint https://example.com/events \
  --included-event-types Microsoft.ContainerRegistry.ImagePushed
```

### By Subject (Repository)
```bash
az eventgrid event-subscription create \
  --name mysubscription \
  --source-resource-id $(az acr show --name myregistry --query id -o tsv) \
  --endpoint https://example.com/events \
  --subject-begins-with "production/"
```

### Advanced Filters
```bash
az eventgrid event-subscription create \
  --name mysubscription \
  --source-resource-id $(az acr show --name myregistry --query id -o tsv) \
  --endpoint https://example.com/events \
  --advanced-filter data.target.repository StringBeginsWith "prod-"
```

## Management

```bash
# List subscriptions
az eventgrid event-subscription list \
  --source-resource-id $(az acr show --name myregistry --query id -o tsv)

# Show subscription
az eventgrid event-subscription show \
  --name mysubscription \
  --source-resource-id $(az acr show --name myregistry --query id -o tsv)

# Update subscription
az eventgrid event-subscription update \
  --name mysubscription \
  --source-resource-id $(az acr show --name myregistry --query id -o tsv) \
  --endpoint https://newurl.example.com/events

# Delete subscription
az eventgrid event-subscription delete \
  --name mysubscription \
  --source-resource-id $(az acr show --name myregistry --query id -o tsv)
```

## Azure Function Handler

```csharp
[FunctionName("ACREventHandler")]
public static async Task Run(
    [EventGridTrigger] EventGridEvent eventGridEvent,
    ILogger log)
{
    var data = eventGridEvent.Data.ToObjectFromJson<ACREventData>();
    log.LogInformation($"Image pushed: {data.Target.Repository}:{data.Target.Tag}");

    // Trigger deployment, notification, etc.
}
```

## Connected Registry Events

For connected registries, events include `connectedregistry` field:
```json
{
  "data": {
    "connectedregistry": {
      "name": "myconnected"
    }
  }
}
```

Filter by connected registry:
```bash
--advanced-filter data.connectedregistry.name StringEquals "myconnected"
```

## Reference Documentation

- **Investigation reports**: `agent-investigation-reports/acr-features/event-grid/`
- **MS Learn docs**: `azure-management-docs/articles/container-registry/container-registry-event-grid-quickstart.md`

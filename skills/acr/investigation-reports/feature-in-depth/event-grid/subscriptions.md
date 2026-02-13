# Azure Container Registry Event Grid - Subscriptions

## Overview

Event Grid subscriptions define how events from ACR are routed to endpoints. This document covers subscription creation, configuration, filtering, and management.

## Creating Event Subscriptions

### Prerequisites

1. **Register the Event Grid resource provider** (if not already registered):
```bash
az provider register --namespace Microsoft.EventGrid
az provider show --namespace Microsoft.EventGrid --query "registrationState"
```

2. **Required permissions**:
   - `Microsoft.EventGrid/eventSubscriptions/write` on the registry resource
   - Or the `Container Registry Contributor and Data Access Configuration Administrator` role

### Basic Subscription Creation

#### Using Azure CLI

```bash
# Get registry resource ID
ACR_REGISTRY_ID=$(az acr show --name <registry-name> --query id --output tsv)

# Create endpoint URL (e.g., Azure Function, Logic App, or webhook)
APP_ENDPOINT="https://your-endpoint.azurewebsites.net/api/updates"

# Create event subscription
az eventgrid event-subscription create \
    --name <subscription-name> \
    --source-resource-id $ACR_REGISTRY_ID \
    --endpoint $APP_ENDPOINT
```

#### Using Azure Portal

1. Navigate to your container registry in the Azure portal
2. Select **Events** under Settings
3. Click **+ Event Subscription**
4. Configure:
   - **Name**: Unique subscription name
   - **Event Schema**: Event Grid Schema or CloudEvents v1.0
   - **Event Types**: Select which events to subscribe to
   - **Endpoint Type**: Select destination (Web Hook, Azure Function, etc.)
   - **Endpoint**: Configure the destination endpoint

### Subscription Output Example

```json
{
  "destination": {
    "endpointBaseUrl": "https://eventgridviewer.azurewebsites.net/api/updates",
    "endpointType": "WebHook",
    "endpointUrl": null
  },
  "filter": {
    "includedEventTypes": [
      "All"
    ],
    "isSubjectCaseSensitive": null,
    "subjectBeginsWith": "",
    "subjectEndsWith": ""
  },
  "id": "/subscriptions/<sub-id>/resourceGroups/<rg>/providers/Microsoft.ContainerRegistry/registries/<registry>/providers/Microsoft.EventGrid/eventSubscriptions/<name>",
  "labels": null,
  "name": "<subscription-name>",
  "provisioningState": "Succeeded",
  "resourceGroup": "<resource-group>",
  "topic": "/subscriptions/<sub-id>/resourceGroups/<rg>/providers/microsoft.containerregistry/registries/<registry>",
  "type": "Microsoft.EventGrid/eventSubscriptions"
}
```

## Subscription Endpoint Types

### 1. WebHook Endpoint

```bash
az eventgrid event-subscription create \
    --name acr-webhook-sub \
    --source-resource-id $ACR_REGISTRY_ID \
    --endpoint https://your-api.com/webhook \
    --endpoint-type webhook
```

**Validation**: Event Grid sends a validation event that the endpoint must respond to.

### 2. Azure Function

```bash
az eventgrid event-subscription create \
    --name acr-function-sub \
    --source-resource-id $ACR_REGISTRY_ID \
    --endpoint /subscriptions/<sub>/resourceGroups/<rg>/providers/Microsoft.Web/sites/<app>/functions/<function> \
    --endpoint-type azurefunction
```

### 3. Azure Event Hub

```bash
az eventgrid event-subscription create \
    --name acr-eventhub-sub \
    --source-resource-id $ACR_REGISTRY_ID \
    --endpoint /subscriptions/<sub>/resourceGroups/<rg>/providers/Microsoft.EventHub/namespaces/<namespace>/eventhubs/<hub> \
    --endpoint-type eventhub
```

### 4. Azure Storage Queue

```bash
az eventgrid event-subscription create \
    --name acr-queue-sub \
    --source-resource-id $ACR_REGISTRY_ID \
    --endpoint /subscriptions/<sub>/resourceGroups/<rg>/providers/Microsoft.Storage/storageAccounts/<account>/queueServices/default/queues/<queue> \
    --endpoint-type storagequeue
```

### 5. Azure Service Bus Queue

```bash
az eventgrid event-subscription create \
    --name acr-servicebus-sub \
    --source-resource-id $ACR_REGISTRY_ID \
    --endpoint /subscriptions/<sub>/resourceGroups/<rg>/providers/Microsoft.ServiceBus/namespaces/<namespace>/queues/<queue> \
    --endpoint-type servicebusqueue
```

### 6. Azure Service Bus Topic

```bash
az eventgrid event-subscription create \
    --name acr-servicebus-topic-sub \
    --source-resource-id $ACR_REGISTRY_ID \
    --endpoint /subscriptions/<sub>/resourceGroups/<rg>/providers/Microsoft.ServiceBus/namespaces/<namespace>/topics/<topic> \
    --endpoint-type servicebustopic
```

## Connected Registry Event Subscriptions

Connected registries require special filtering to receive only connected registry events.

### Filter Events from All Connected Registries

```bash
az eventgrid event-subscription create \
    --name event-sub-connected \
    --source-resource-id $ACR_REGISTRY_ID \
    --endpoint $APP_ENDPOINT \
    --advanced-filter data.connectedregistry IsNotNull
```

### Filter Events by Specific Connected Registry Name

```bash
ACR_CONNECTED_REGISTRY_NAME="myconnectedregistry"

az eventgrid event-subscription create \
    --name event-sub-specific-connected \
    --source-resource-id $ACR_REGISTRY_ID \
    --endpoint $APP_ENDPOINT \
    --advanced-filter data.connectedregistry IsNotNull \
    --advanced-filter data.connectedregistry.name StringIn $ACR_CONNECTED_REGISTRY_NAME
```

## Subscription Configuration Options

### Event Type Filtering

Subscribe only to specific event types:

```bash
az eventgrid event-subscription create \
    --name acr-push-only-sub \
    --source-resource-id $ACR_REGISTRY_ID \
    --endpoint $APP_ENDPOINT \
    --included-event-types Microsoft.ContainerRegistry.ImagePushed
```

Multiple event types:

```bash
az eventgrid event-subscription create \
    --name acr-image-events-sub \
    --source-resource-id $ACR_REGISTRY_ID \
    --endpoint $APP_ENDPOINT \
    --included-event-types \
        Microsoft.ContainerRegistry.ImagePushed \
        Microsoft.ContainerRegistry.ImageDeleted
```

### Subject Filtering

Filter by repository name prefix:

```bash
az eventgrid event-subscription create \
    --name acr-prod-images-sub \
    --source-resource-id $ACR_REGISTRY_ID \
    --endpoint $APP_ENDPOINT \
    --subject-begins-with "production/"
```

Filter by tag suffix:

```bash
az eventgrid event-subscription create \
    --name acr-latest-tag-sub \
    --source-resource-id $ACR_REGISTRY_ID \
    --endpoint $APP_ENDPOINT \
    --subject-ends-with ":latest"
```

### Dead-Letter Configuration

Configure dead-lettering for failed event delivery:

```bash
az eventgrid event-subscription create \
    --name acr-sub-with-deadletter \
    --source-resource-id $ACR_REGISTRY_ID \
    --endpoint $APP_ENDPOINT \
    --deadletter-endpoint /subscriptions/<sub>/resourceGroups/<rg>/providers/Microsoft.Storage/storageAccounts/<account>/blobServices/default/containers/<container>
```

### Retry Policy

Configure retry attempts and time-to-live:

```bash
az eventgrid event-subscription create \
    --name acr-sub-with-retry \
    --source-resource-id $ACR_REGISTRY_ID \
    --endpoint $APP_ENDPOINT \
    --max-delivery-attempts 30 \
    --event-ttl 1440  # 24 hours in minutes
```

### Labels

Add labels for organization:

```bash
az eventgrid event-subscription create \
    --name acr-labeled-sub \
    --source-resource-id $ACR_REGISTRY_ID \
    --endpoint $APP_ENDPOINT \
    --labels env=production team=devops
```

## Managing Subscriptions

### List Subscriptions

```bash
# List all subscriptions for a registry
az eventgrid event-subscription list \
    --source-resource-id $ACR_REGISTRY_ID

# List with details
az eventgrid event-subscription list \
    --source-resource-id $ACR_REGISTRY_ID \
    --output table
```

### Show Subscription Details

```bash
az eventgrid event-subscription show \
    --name <subscription-name> \
    --source-resource-id $ACR_REGISTRY_ID
```

### Update Subscription

```bash
az eventgrid event-subscription update \
    --name <subscription-name> \
    --source-resource-id $ACR_REGISTRY_ID \
    --endpoint https://new-endpoint.com/api/events
```

### Delete Subscription

```bash
az eventgrid event-subscription delete \
    --name <subscription-name> \
    --source-resource-id $ACR_REGISTRY_ID
```

## Subscription Validation

When creating a webhook subscription, Event Grid sends a validation request. The endpoint must:

1. Return HTTP 200 status
2. Echo back the `validationCode` from the request

### Validation Request Example

```json
{
  "validationCode": "512d38b6-c7b8-40c8-89fe-f46f9e9622b6",
  "validationUrl": "https://rp-eastus.eventgrid.azure.net:553/..."
}
```

### Validation Response

```json
{
  "validationResponse": "512d38b6-c7b8-40c8-89fe-f46f9e9622b6"
}
```

## Best Practices

### 1. Use Specific Event Type Filtering
Subscribe only to events you need to reduce unnecessary processing.

### 2. Implement Dead-Lettering
Always configure dead-lettering for production subscriptions to avoid losing events.

### 3. Use Appropriate Retry Settings
Configure retry policies based on your endpoint's availability characteristics.

### 4. Secure Webhook Endpoints
- Use HTTPS endpoints
- Implement authentication (shared secret, Azure AD)
- Validate the `aeg-subscription-name` header

### 5. Monitor Subscription Health
Use Azure Monitor to track:
- Delivery success/failure rates
- Latency metrics
- Dead-letter queue sizes

### 6. Handle Idempotency
Events may be delivered more than once. Design handlers to be idempotent using the event `id`.

## Troubleshooting

### Common Issues

1. **Subscription creation fails with 403**
   - Verify Event Grid resource provider is registered
   - Check permissions on the registry resource

2. **Events not being delivered**
   - Verify endpoint is accessible
   - Check Event Grid metrics for delivery failures
   - Review dead-letter storage for failed events

3. **Webhook validation fails**
   - Ensure endpoint returns 200 with validation code
   - Check endpoint is publicly accessible
   - Verify no firewall blocking Event Grid IP ranges

## References

- Source: `/submodules/azure-management-docs/articles/container-registry/container-registry-event-grid-quickstart.md`
- Source: `/submodules/acr/docs/preview/connected-registry/quickstart-send-connected-registry-events-to-event-grid.md`

# Azure Container Registry Event Grid - CLI Commands Reference

## Overview

This document provides a comprehensive reference for all Azure CLI commands related to ACR Event Grid integration, including event subscriptions, connected registry notifications, and provider management.

## Prerequisites

### Install Azure CLI
```bash
# Install Azure CLI (if not installed)
curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash

# Verify installation
az version
```

### Login and Set Subscription
```bash
# Login to Azure
az login

# Set subscription
az account set --subscription "<subscription-id>"
```

### Register Event Grid Provider
```bash
# Register the Event Grid resource provider
az provider register --namespace Microsoft.EventGrid

# Check registration status
az provider show --namespace Microsoft.EventGrid --query "registrationState"
# Expected output: "Registered"
```

## Environment Setup

### Set Common Variables
```bash
# Resource group
RESOURCE_GROUP_NAME="myResourceGroup"

# Container registry
ACR_NAME="myregistry"
ACR_REGISTRY_ID=$(az acr show --name $ACR_NAME --query id --output tsv)

# Event endpoint
APP_ENDPOINT="https://your-endpoint.azurewebsites.net/api/updates"

# Connected registry (if applicable)
ACR_CONNECTED_REGISTRY_NAME="myconnectedregistry"
```

## Event Grid Event Subscription Commands

### Create Event Subscription

#### Basic Creation
```bash
az eventgrid event-subscription create \
    --name <subscription-name> \
    --source-resource-id $ACR_REGISTRY_ID \
    --endpoint $APP_ENDPOINT
```

#### With Event Type Filter
```bash
az eventgrid event-subscription create \
    --name <subscription-name> \
    --source-resource-id $ACR_REGISTRY_ID \
    --endpoint $APP_ENDPOINT \
    --included-event-types \
        Microsoft.ContainerRegistry.ImagePushed \
        Microsoft.ContainerRegistry.ImageDeleted
```

#### With Subject Filter
```bash
az eventgrid event-subscription create \
    --name <subscription-name> \
    --source-resource-id $ACR_REGISTRY_ID \
    --endpoint $APP_ENDPOINT \
    --subject-begins-with "production/" \
    --subject-ends-with ":latest"
```

#### With Advanced Filter
```bash
az eventgrid event-subscription create \
    --name <subscription-name> \
    --source-resource-id $ACR_REGISTRY_ID \
    --endpoint $APP_ENDPOINT \
    --advanced-filter data.target.repository StringIn repo1 repo2
```

#### With Dead-Letter Configuration
```bash
az eventgrid event-subscription create \
    --name <subscription-name> \
    --source-resource-id $ACR_REGISTRY_ID \
    --endpoint $APP_ENDPOINT \
    --deadletter-endpoint "/subscriptions/<sub>/resourceGroups/<rg>/providers/Microsoft.Storage/storageAccounts/<account>/blobServices/default/containers/<container>"
```

#### With Retry Policy
```bash
az eventgrid event-subscription create \
    --name <subscription-name> \
    --source-resource-id $ACR_REGISTRY_ID \
    --endpoint $APP_ENDPOINT \
    --max-delivery-attempts 30 \
    --event-ttl 1440
```

#### To Azure Function Endpoint
```bash
az eventgrid event-subscription create \
    --name <subscription-name> \
    --source-resource-id $ACR_REGISTRY_ID \
    --endpoint /subscriptions/<sub>/resourceGroups/<rg>/providers/Microsoft.Web/sites/<app>/functions/<function> \
    --endpoint-type azurefunction
```

#### To Event Hub Endpoint
```bash
az eventgrid event-subscription create \
    --name <subscription-name> \
    --source-resource-id $ACR_REGISTRY_ID \
    --endpoint /subscriptions/<sub>/resourceGroups/<rg>/providers/Microsoft.EventHub/namespaces/<namespace>/eventhubs/<hub> \
    --endpoint-type eventhub
```

#### To Storage Queue Endpoint
```bash
az eventgrid event-subscription create \
    --name <subscription-name> \
    --source-resource-id $ACR_REGISTRY_ID \
    --endpoint /subscriptions/<sub>/resourceGroups/<rg>/providers/Microsoft.Storage/storageAccounts/<account>/queueServices/default/queues/<queue> \
    --endpoint-type storagequeue
```

#### To Service Bus Queue Endpoint
```bash
az eventgrid event-subscription create \
    --name <subscription-name> \
    --source-resource-id $ACR_REGISTRY_ID \
    --endpoint /subscriptions/<sub>/resourceGroups/<rg>/providers/Microsoft.ServiceBus/namespaces/<namespace>/queues/<queue> \
    --endpoint-type servicebusqueue
```

### List Event Subscriptions

```bash
# List all subscriptions for a registry
az eventgrid event-subscription list \
    --source-resource-id $ACR_REGISTRY_ID

# List with table output
az eventgrid event-subscription list \
    --source-resource-id $ACR_REGISTRY_ID \
    --output table
```

### Show Event Subscription Details

```bash
az eventgrid event-subscription show \
    --name <subscription-name> \
    --source-resource-id $ACR_REGISTRY_ID
```

### Update Event Subscription

```bash
# Update endpoint
az eventgrid event-subscription update \
    --name <subscription-name> \
    --source-resource-id $ACR_REGISTRY_ID \
    --endpoint https://new-endpoint.com/api/events

# Update filters
az eventgrid event-subscription update \
    --name <subscription-name> \
    --source-resource-id $ACR_REGISTRY_ID \
    --subject-begins-with "newprefix/"

# Update retry policy
az eventgrid event-subscription update \
    --name <subscription-name> \
    --source-resource-id $ACR_REGISTRY_ID \
    --max-delivery-attempts 10
```

### Delete Event Subscription

```bash
az eventgrid event-subscription delete \
    --name <subscription-name> \
    --source-resource-id $ACR_REGISTRY_ID
```

## Connected Registry Notification Commands

### Enable Notifications on Connected Registry

```bash
# Enable notifications for all artifacts with all actions
az acr connected-registry update \
    --registry $ACR_NAME \
    --name $ACR_CONNECTED_REGISTRY_NAME \
    --add-notifications "*:*"
```

### Configure Specific Notification Patterns

```bash
# Notify all actions on all tags for all artifacts
az acr connected-registry update \
    --registry $ACR_NAME \
    --name $ACR_CONNECTED_REGISTRY_NAME \
    --add-notifications "*:*:*"

# Notify all actions on all digests for all artifacts
az acr connected-registry update \
    --registry $ACR_NAME \
    --name $ACR_CONNECTED_REGISTRY_NAME \
    --add-notifications "*@*:*"

# Notify all actions on specific repository
az acr connected-registry update \
    --registry $ACR_NAME \
    --name $ACR_CONNECTED_REGISTRY_NAME \
    --add-notifications "path/to/repo:*"

# Notify all actions on specific tag
az acr connected-registry update \
    --registry $ACR_NAME \
    --name $ACR_CONNECTED_REGISTRY_NAME \
    --add-notifications "path/to/repo:myTag:*"

# Notify only delete action on specific digest
az acr connected-registry update \
    --registry $ACR_NAME \
    --name $ACR_CONNECTED_REGISTRY_NAME \
    --add-notifications "path/to/repo@myDigest:delete"

# Notify only push action on matching repositories
az acr connected-registry update \
    --registry $ACR_NAME \
    --name $ACR_CONNECTED_REGISTRY_NAME \
    --add-notifications "path/to/repo/*:push"
```

### Notification Pattern Syntax

Pattern format: `artifact:action`

Where:
- `artifact`: `name[:tag][@digest]`
- `action`: `push` or `delete` or `*` (wildcard)

Examples:
| Pattern | Description |
|---------|-------------|
| `*:*` | All actions on all artifacts |
| `*:*:*` | All actions on all tags of all artifacts |
| `*@*:*` | All actions on all digests of all artifacts |
| `myrepo:*` | All actions on all versions of myrepo |
| `myrepo:v1:*` | All actions on tag v1 of myrepo |
| `myrepo:v1:push` | Push actions only on tag v1 |
| `myrepo@sha256:abc:delete` | Delete actions on specific digest |

### Subscribe to Connected Registry Events

```bash
# Filter events from all connected registries
az eventgrid event-subscription create \
    --name event-sub-connected \
    --source-resource-id $ACR_REGISTRY_ID \
    --endpoint $APP_ENDPOINT \
    --advanced-filter data.connectedregistry IsNotNull

# Filter events from specific connected registry
az eventgrid event-subscription create \
    --name event-sub-specific-connected \
    --source-resource-id $ACR_REGISTRY_ID \
    --endpoint $APP_ENDPOINT \
    --advanced-filter data.connectedregistry IsNotNull \
    --advanced-filter data.connectedregistry.name StringIn $ACR_CONNECTED_REGISTRY_NAME
```

## ACR Registry Commands (Event-Related)

### Show Registry Details
```bash
az acr show --name $ACR_NAME --query id --output tsv
```

### Build and Push Image (Triggers ImagePushed Event)
```bash
az acr build \
    --registry $ACR_NAME \
    --image myimage:v1 \
    -f Dockerfile \
    https://github.com/Azure-Samples/acr-build-helloworld-node.git#main
```

### Delete Image (Triggers ImageDeleted Event)
```bash
# Delete specific image tag
az acr repository delete \
    --name $ACR_NAME \
    --image myimage:v1

# Delete entire repository
az acr repository delete \
    --name $ACR_NAME \
    --repository myimage
```

### Push Helm Chart (Triggers ChartPushed Event)
```bash
az acr helm push \
    mychart-1.0.0.tgz \
    --name $ACR_NAME
```

### Delete Helm Chart (Triggers ChartDeleted Event)
```bash
az acr helm delete \
    mychart \
    --version 1.0.0 \
    --name $ACR_NAME
```

## ACR Webhook Commands (Alternative to Event Grid)

### Create Webhook
```bash
az acr webhook create \
    --registry $ACR_NAME \
    --name mywebhook \
    --actions push delete \
    --uri http://webhookuri.com
```

### List Webhooks
```bash
az acr webhook list --registry $ACR_NAME
```

### Show Webhook
```bash
az acr webhook show \
    --registry $ACR_NAME \
    --name mywebhook
```

### Test Webhook (Ping)
```bash
az acr webhook ping \
    --registry $ACR_NAME \
    --name mywebhook
```

### List Webhook Events
```bash
az acr webhook list-events \
    --registry $ACR_NAME \
    --name mywebhook
```

### Update Webhook
```bash
az acr webhook update \
    --registry $ACR_NAME \
    --name mywebhook \
    --uri http://new-uri.com
```

### Delete Webhook
```bash
az acr webhook delete \
    --registry $ACR_NAME \
    --name mywebhook
```

## Complete Example: End-to-End Setup

```bash
#!/bin/bash

# Variables
RESOURCE_GROUP_NAME="acr-eventgrid-demo"
LOCATION="eastus"
ACR_NAME="myacreg$(date +%s)"
SITE_NAME="eventviewer$(date +%s)"

# Create resource group
az group create \
    --name $RESOURCE_GROUP_NAME \
    --location $LOCATION

# Create container registry
az acr create \
    --resource-group $RESOURCE_GROUP_NAME \
    --name $ACR_NAME \
    --sku Basic

# Deploy sample event viewer app
az deployment group create \
    --resource-group $RESOURCE_GROUP_NAME \
    --template-uri "https://raw.githubusercontent.com/Azure-Samples/azure-event-grid-viewer/master/azuredeploy.json" \
    --parameters siteName=$SITE_NAME hostingPlanName=$SITE_NAME-plan

# Register Event Grid provider
az provider register --namespace Microsoft.EventGrid

# Wait for registration
while [ $(az provider show --namespace Microsoft.EventGrid --query "registrationState" -o tsv) != "Registered" ]; do
    echo "Waiting for Event Grid provider registration..."
    sleep 5
done

# Get registry ID
ACR_REGISTRY_ID=$(az acr show --name $ACR_NAME --query id --output tsv)

# Create event subscription
APP_ENDPOINT="https://$SITE_NAME.azurewebsites.net/api/updates"

az eventgrid event-subscription create \
    --name event-sub-acr \
    --source-resource-id $ACR_REGISTRY_ID \
    --endpoint $APP_ENDPOINT

# Build and push an image to trigger ImagePushed event
az acr build \
    --registry $ACR_NAME \
    --image myimage:v1 \
    -f Dockerfile \
    https://github.com/Azure-Samples/acr-build-helloworld-node.git#main

# Delete the image to trigger ImageDeleted event
az acr repository delete \
    --name $ACR_NAME \
    --image myimage:v1 \
    --yes

echo "Event viewer URL: https://$SITE_NAME.azurewebsites.net"
echo "View events in the web browser"
```

## Troubleshooting Commands

### Check Event Grid Provider Status
```bash
az provider show --namespace Microsoft.EventGrid --query "registrationState"
```

### List All Event Subscriptions
```bash
az eventgrid event-subscription list --source-resource-id $ACR_REGISTRY_ID
```

### Get Event Subscription Delivery Status
```bash
az eventgrid event-subscription show \
    --name <subscription-name> \
    --source-resource-id $ACR_REGISTRY_ID \
    --query "deliveryWithResourceIdentity"
```

### Check ACR Registry Events
```bash
az acr repository show-tags --name $ACR_NAME --repository myimage
```

## References

- Source: `/submodules/azure-management-docs/articles/container-registry/container-registry-event-grid-quickstart.md`
- Source: `/submodules/azure-management-docs/articles/container-registry/container-registry-webhook.md`
- Source: `/submodules/acr/docs/preview/connected-registry/quickstart-send-connected-registry-events-to-event-grid.md`
- Azure CLI Reference: https://docs.microsoft.com/cli/azure/eventgrid/event-subscription
- Azure CLI Reference: https://docs.microsoft.com/cli/azure/acr/webhook

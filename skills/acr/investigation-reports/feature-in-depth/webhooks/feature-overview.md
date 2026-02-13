# Azure Container Registry Webhooks - Feature Overview

## What are ACR Webhooks?

Azure Container Registry webhooks are event-driven notification mechanisms that trigger HTTP POST requests to specified endpoints when certain actions occur in your registry. They enable automation of deployment workflows, CI/CD pipelines, and notification systems by responding to registry events in real-time.

## Core Capabilities

### Event-Driven Notifications
- Webhooks send HTTP POST requests to user-specified endpoints when registry actions occur
- Support for multiple event types including image push, image delete, Helm chart operations, and quarantine events
- Events can be scoped to the entire registry or specific repositories/tags

### Geo-Replication Support
- Each webhook can be configured for a specific regional replica in geo-replicated registries
- Allows tracking of push events as they complete across different geo-replicated regions
- Regional webhooks help manage workflows dependent on geo-replicated content

### Secure Communication
- Webhook endpoints must be publicly accessible from the registry
- Support for custom headers to pass authentication tokens (e.g., "Authorization: Bearer token")
- HTTPS endpoints are recommended for secure communication

## Use Cases

### 1. Automated Deployments
- Trigger container deployments when new images are pushed
- Azure App Service / Web Apps for Containers automatic deployments
- Kubernetes deployments via webhook-triggered CI/CD pipelines

### 2. CI/CD Pipeline Integration
- Notify build systems when base images are updated
- Trigger downstream builds when dependencies change
- Automate testing workflows when new images are available

### 3. Notification Systems
- Send alerts to Slack, Teams, or email when critical images are modified
- Track image deletion events for audit purposes
- Monitor quarantine state changes for security scanning workflows

### 4. Compliance and Auditing
- Log all push and delete events for compliance tracking
- Track who pushed what images and when
- Monitor for unauthorized deletions

### 5. Geo-Replication Workflows
- Track replication completion across regions
- Trigger regional deployments when images reach specific replicas
- Coordinate multi-region deployment strategies

## Service Tier Support

| Feature | Basic | Standard | Premium |
|---------|-------|----------|---------|
| Webhooks Supported | Yes | Yes | Yes |
| Webhook Quota | Varies by tier | Varies by tier | Highest quota |
| Geo-Replication Webhooks | No | No | Yes |

**Note**: All service tiers support webhooks, but the number of webhooks you can create varies by SKU. Premium tier supports regional webhooks for geo-replicated registries.

## Webhook vs Event Grid

ACR supports two notification mechanisms:

### Native Webhooks
- Built into ACR directly
- Simple setup through portal or CLI
- HTTP POST to custom endpoints
- Supports all webhook actions (push, delete, chart_push, chart_delete, quarantine)

### Event Grid Integration
- More advanced event routing
- Publish-subscribe model
- Better for complex event processing scenarios
- Integrates with other Azure services
- Supports filtering and multiple subscribers

## Key Concepts

### Webhook Scope
- **Registry-wide**: Respond to all events in the registry
- **Repository-scoped**: Target specific repositories using format `repository:*`
- **Tag-scoped**: Target specific tags using format `repository:tag`

### Webhook Actions
1. **push** - Container image pushed to a repository
2. **delete** - Container image or manifest deleted
3. **chart_push** - Helm chart pushed to a repository
4. **chart_delete** - Helm chart deleted
5. **quarantine** - Image enters quarantine state (preview feature)

### Webhook Status
- **Enabled**: Webhook is active and will fire on matching events
- **Disabled**: Webhook exists but will not fire

## Prerequisites

1. **Azure Container Registry**: Must have an existing ACR in any tier (Basic, Standard, or Premium)
2. **Publicly Accessible Endpoint**: The webhook endpoint must be reachable from Azure
3. **Docker CLI** (optional): For testing by pushing/pulling images

## Limitations

- Webhook endpoints must be publicly accessible from the registry
- Webhooks are scoped to a single registry (or replica in geo-replicated registries)
- Custom headers must be in "key: value" format
- Webhook names must be 5-50 alphanumeric characters
- Maximum number of webhooks depends on SKU tier

## Related Features

- **Event Grid**: For more advanced event routing and processing
- **Geo-Replication**: Regional webhooks for distributed deployments
- **ACR Tasks**: Automated builds triggered by various events
- **Quarantine (Preview)**: Security scanning workflow with quarantine webhooks

## Documentation References

- [Using Azure Container Registry webhooks](https://docs.microsoft.com/azure/container-registry/container-registry-webhook)
- [Azure Container Registry webhook schema reference](https://docs.microsoft.com/azure/container-registry/container-registry-webhook-reference)
- [Quickstart: Send container registry events to Event Grid](https://docs.microsoft.com/azure/container-registry/container-registry-event-grid-quickstart)

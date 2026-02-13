# ACR Webhooks Skill

This skill provides comprehensive knowledge about Azure Container Registry webhooks.

## When to Use This Skill

Use this skill when answering questions about:
- Push/delete notifications
- Webhook configuration
- Event schema
- CI/CD triggers

## Overview

Webhooks send HTTP POST notifications when registry events occur.

**Availability:** All SKUs (Basic, Standard, Premium)

## Supported Events

| Event | Trigger |
|-------|---------|
| `push` | Image pushed |
| `delete` | Image deleted |
| `chart_push` | Helm chart pushed |
| `chart_delete` | Helm chart deleted |
| `quarantine` | Quarantine state change (preview) |

## Quick Setup

```bash
# Create webhook
az acr webhook create \
  --registry myregistry \
  --name mywebhook \
  --uri https://myapp.example.com/webhook \
  --actions push delete

# List webhooks
az acr webhook list --registry myregistry -o table

# Test webhook (ping)
az acr webhook ping --registry myregistry --name mywebhook

# View recent deliveries
az acr webhook list-events --registry myregistry --name mywebhook
```

## Webhook Configuration

### Scope Filtering
```bash
# Scope to specific repository
az acr webhook create \
  --registry myregistry \
  --name mywebhook \
  --uri https://example.com/webhook \
  --actions push \
  --scope "myrepo:*"
```

### Custom Headers
```bash
az acr webhook create \
  --registry myregistry \
  --name mywebhook \
  --uri https://example.com/webhook \
  --actions push \
  --headers "Authorization=Bearer token123"
```

### Geo-Replicated Webhook
```bash
az acr webhook create \
  --registry myregistry \
  --name mywebhook \
  --uri https://example.com/webhook \
  --actions push \
  --location westeurope
```

## Event Schema

### Push Event
```json
{
  "id": "cb8c3971-9adc-488b-xxxx",
  "timestamp": "2024-01-15T10:00:00Z",
  "action": "push",
  "target": {
    "mediaType": "application/vnd.docker.distribution.manifest.v2+json",
    "size": 524,
    "digest": "sha256:abc123...",
    "repository": "myapp",
    "tag": "v1"
  },
  "request": {
    "id": "request-id",
    "host": "myregistry.azurecr.io",
    "method": "PUT"
  }
}
```

### Delete Event
```json
{
  "id": "cb8c3971-9adc-488b-xxxx",
  "timestamp": "2024-01-15T10:00:00Z",
  "action": "delete",
  "target": {
    "digest": "sha256:abc123...",
    "repository": "myapp"
  }
}
```

## Webhook Management

```bash
# Update webhook
az acr webhook update \
  --registry myregistry \
  --name mywebhook \
  --uri https://newurl.example.com/webhook

# Enable/disable webhook
az acr webhook update \
  --registry myregistry \
  --name mywebhook \
  --status disabled

# Delete webhook
az acr webhook delete \
  --registry myregistry \
  --name mywebhook
```

## Webhook Limits

| SKU | Max Webhooks |
|-----|-------------|
| Basic | 2 |
| Standard | 10 |
| Premium | 500 |

## Sample Handler (Python)

```python
from flask import Flask, request

app = Flask(__name__)

@app.route('/webhook', methods=['POST'])
def handle_webhook():
    event = request.json
    action = event.get('action')
    repo = event['target'].get('repository')
    tag = event['target'].get('tag', 'untagged')

    print(f"Event: {action} {repo}:{tag}")

    # Trigger deployment, notification, etc.

    return '', 200
```

## Webhooks vs Event Grid

| Feature | Webhooks | Event Grid |
|---------|----------|------------|
| Delivery | Direct HTTP | Azure Event Grid |
| Retry | Limited | Advanced |
| Filtering | Scope only | Rich filters |
| Destinations | HTTP endpoints | Many Azure services |
| Quarantine events | ✅ | ❌ |

## Reference Documentation

- **Investigation reports**: `agent-investigation-reports/acr-features/webhooks/`
- **MS Learn docs**: `azure-management-docs/articles/container-registry/container-registry-webhook.md`

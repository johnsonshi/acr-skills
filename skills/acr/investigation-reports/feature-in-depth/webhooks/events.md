# ACR Webhook Events - Supported Event Types

## Overview

Azure Container Registry webhooks support five distinct event types (actions) that can trigger webhook notifications. Each event type has its own payload schema and is triggered by specific registry operations.

## Supported Event Actions

| Action | Description | Triggered By |
|--------|-------------|--------------|
| `push` | Container image pushed to repository | `docker push`, `az acr build`, `az acr import` |
| `delete` | Container image or manifest deleted | `az acr repository delete`, `docker` manifest delete |
| `chart_push` | Helm chart pushed to repository | `az acr helm push`, Helm CLI |
| `chart_delete` | Helm chart or repository deleted | `az acr helm delete` |
| `quarantine` | Image enters quarantine state | Image push when quarantine policy enabled |

## Detailed Event Descriptions

### 1. Push Event (`push`)

**Action Value**: `push`

**Triggered When**:
- A container image is pushed to a repository via `docker push`
- An image is built and pushed via ACR Tasks (`az acr build`)
- An image is imported into the registry (`az acr import`)
- Any OCI-compliant artifact is pushed

**Example Triggers**:
```bash
# Docker push
docker push myregistry.azurecr.io/hello-world:v1

# ACR build (automatically pushes)
az acr build --registry myregistry --image myimage:v1 .

# Import from another registry
az acr import --name myregistry --source mcr.microsoft.com/hello-world:latest
```

**Key Characteristics**:
- Most commonly used webhook event
- Triggered for each manifest push (not individual layer uploads)
- Includes full target information (repository, tag, digest, size)

---

### 2. Delete Event (`delete`)

**Action Value**: `delete`

**Triggered When**:
- An image manifest is deleted from a repository
- A repository is deleted entirely
- An image is deleted by digest or tag

**NOT Triggered When**:
- Only a tag is deleted (tag deletion without manifest deletion)
- Individual layers are garbage collected

**Example Triggers**:
```bash
# Delete by image tag
az acr repository delete --name MyRegistry --image MyRepository:MyTag

# Delete entire repository
az acr repository delete --name MyRegistry --repository MyRepository

# Delete by digest
az acr repository delete --name MyRegistry --image MyRepository@sha256:abc123...
```

**Key Characteristics**:
- Tag-only deletions do NOT trigger this webhook
- Useful for audit and compliance tracking
- Payload includes digest but may not include tag information

---

### 3. Chart Push Event (`chart_push`)

**Action Value**: `chart_push`

**Triggered When**:
- A Helm chart is pushed to the registry

**Example Triggers**:
```bash
# Push Helm chart using Azure CLI
az acr helm push wordpress-5.4.0.tgz --name MyRegistry

# Push using Helm CLI with OCI support
helm push mychart-1.0.0.tgz oci://myregistry.azurecr.io/helm
```

**Key Characteristics**:
- Helm chart storage is supported in all ACR tiers
- Payload includes chart-specific fields (name, version)
- Media type is `application/vnd.acr.helm.chart`

---

### 4. Chart Delete Event (`chart_delete`)

**Action Value**: `chart_delete`

**Triggered When**:
- A Helm chart is deleted from a repository
- A Helm chart repository is deleted entirely

**Example Triggers**:
```bash
# Delete specific chart version
az acr helm delete wordpress --version 5.4.0 --name MyRegistry

# Delete all versions of a chart
az acr helm delete wordpress --name MyRegistry
```

**Key Characteristics**:
- Includes chart name and version in payload
- Useful for tracking chart lifecycle

---

### 5. Quarantine Event (`quarantine`)

**Action Value**: `quarantine`

**Triggered When**:
- An image is pushed to a registry with quarantine policy enabled
- Image enters quarantine state automatically upon push

**Quarantine Workflow**:
1. Image is pushed to registry
2. `quarantine` webhook fires (scanner subscribes to this)
3. Scanner pulls and analyzes the image
4. Scanner marks image as `Passed` or `Failed`
5. If `Passed`, `push` webhook fires for normal users

**Key Characteristics**:
- Preview feature requiring quarantine policy enablement
- Requires special roles (`AcrQuarantineReader`, `AcrQuarantineWriter`)
- Enables security scanning before images are available

**Example Payload**:
```json
{
  "id": "0d799b14-404b-4859-b2f6-50c5ee2a2c3a",
  "timestamp": "2018-02-28T00:42:54.4509516Z",
  "action": "quarantine",
  "target": {
    "size": 1791,
    "digest": "sha256:91ef6...",
    "length": 1791,
    "repository": "helloworld",
    "tag": "1"
  },
  "request": {
    "id": "978fc988-...",
    "host": "[registry].azurecr.io",
    "method": "PUT"
  }
}
```

---

## Multi-Action Webhooks

A single webhook can be configured to respond to multiple actions. When creating a webhook, you can specify one or more actions:

```bash
# Webhook for multiple actions
az acr webhook create \
  --registry myregistry \
  --name mywebhook \
  --actions push delete \
  --uri https://myendpoint.com/webhook
```

## Event Filtering by Scope

Events can be filtered by scope to target specific repositories or tags:

| Scope Format | Description | Example |
|--------------|-------------|---------|
| (empty/unset) | All events in registry | Receives all push/delete events |
| `repository:*` | All tags in a repository | `myrepo:*` - any tag in myrepo |
| `repository:tag` | Specific tag only | `myrepo:v1` - only v1 tag |

## Event Timing

- **Push events**: Triggered after the manifest is successfully stored
- **Delete events**: Triggered after the manifest is successfully deleted
- **Quarantine events**: Triggered immediately when image enters quarantine
- **Geo-replicated registries**: Events fire per-replica as replication completes

## Event Grid vs Webhook Events

| Feature | Native Webhooks | Event Grid |
|---------|-----------------|------------|
| Push event | `push` action | `Microsoft.ContainerRegistry.ImagePushed` |
| Delete event | `delete` action | `Microsoft.ContainerRegistry.ImageDeleted` |
| Chart events | Supported | Not directly supported |
| Quarantine | Supported | Not supported |

## Best Practices

1. **Use specific scopes** when possible to reduce noise
2. **Handle retries** - implement idempotent webhook handlers
3. **Validate signatures** - use custom headers for authentication
4. **Monitor webhook health** - check webhook event history regularly
5. **Consider Event Grid** for complex routing scenarios

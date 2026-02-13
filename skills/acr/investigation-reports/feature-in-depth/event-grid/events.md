# Azure Container Registry Event Grid - Events Reference

## Overview

This document provides a comprehensive reference for all event types emitted by Azure Container Registry to Azure Event Grid, including their schemas, payloads, and triggering actions.

## Event Types

### 1. Microsoft.ContainerRegistry.ImagePushed

Triggered when a container image manifest is pushed to a repository.

#### Schema

```json
{
  "id": "831e1650-001e-001b-66ab-eeb76e069631",
  "topic": "/subscriptions/<subscription-id>/resourceGroups/<resource-group>/providers/Microsoft.ContainerRegistry/registries/<registry-name>",
  "subject": "<repository>:<tag>",
  "eventType": "Microsoft.ContainerRegistry.ImagePushed",
  "eventTime": "2023-04-25T21:39:47.6549614Z",
  "data": {
    "id": "31c51664-e5bd-416a-a5df-e5206bc47ed0",
    "timestamp": "2023-04-25T21:39:47.276585742Z",
    "action": "push",
    "target": {
      "mediaType": "application/vnd.docker.distribution.manifest.v2+json",
      "size": 3023,
      "digest": "sha256:213bbc182920ab41e18edc2001e06abcca6735d87782d9cef68abd83941cf0e5",
      "repository": "<repository-name>",
      "tag": "<tag-name>",
      "length": 3023
    },
    "request": {
      "id": "978fc988-zzz-yyyy-xxxx-4f6e331d1591",
      "host": "<registry-name>.azurecr.io",
      "method": "PUT",
      "useragent": "docker/17.09.0-ce go/go1.8.3 ..."
    }
  },
  "dataVersion": "1.0",
  "metadataVersion": "1"
}
```

#### Triggering Actions
- `docker push <registry>.azurecr.io/<image>:<tag>`
- `az acr build --registry <registry> --image <image>:<tag> .`
- `az acr import --name <registry> --source <source-image>`

### 2. Microsoft.ContainerRegistry.ImageDeleted

Triggered when an image repository or manifest is deleted. **Note**: Not triggered when only a tag is deleted.

#### Schema

```json
{
  "id": "831e1650-001e-001b-66ab-eeb76e069631",
  "topic": "/subscriptions/<subscription-id>/resourceGroups/<resource-group>/providers/Microsoft.ContainerRegistry/registries/<registry-name>",
  "subject": "<repository>",
  "eventType": "Microsoft.ContainerRegistry.ImageDeleted",
  "eventTime": "2023-04-25T21:39:47.6549614Z",
  "data": {
    "id": "31c51664-e5bd-416a-a5df-e5206bc47ed0",
    "timestamp": "2023-04-25T21:39:47.276585742Z",
    "action": "delete",
    "target": {
      "mediaType": "application/vnd.docker.distribution.manifest.v2+json",
      "digest": "sha256:213bbc182920ab41e18edc2001e06abcca6735d87782d9cef68abd83941cf0e5",
      "repository": "<repository-name>"
    },
    "request": {
      "id": "978fc988-zzz-yyyy-xxxx-4f6e331d1591",
      "host": "<registry-name>.azurecr.io",
      "method": "DELETE",
      "useragent": "python-requests/2.18.4"
    }
  },
  "dataVersion": "1.0",
  "metadataVersion": "1"
}
```

#### Triggering Actions
- `az acr repository delete --name <registry> --repository <repository>`
- `az acr repository delete --name <registry> --image <repository>:<tag>`
- Manifest deletion via API

### 3. Microsoft.ContainerRegistry.ChartPushed

Triggered when a Helm chart is pushed to a repository.

#### Schema

```json
{
  "id": "6356e9e0-627f-4fed-xxxx-d9059b5143ac",
  "topic": "/subscriptions/<subscription-id>/resourceGroups/<resource-group>/providers/Microsoft.ContainerRegistry/registries/<registry-name>",
  "subject": "<repository>:<chart-version>",
  "eventType": "Microsoft.ContainerRegistry.ChartPushed",
  "eventTime": "2023-03-05T23:45:31.2614267Z",
  "data": {
    "id": "6356e9e0-627f-4fed-xxxx-d9059b5143ac",
    "timestamp": "2023-03-05T23:45:31.2614267Z",
    "action": "chart_push",
    "target": {
      "mediaType": "application/vnd.acr.helm.chart",
      "size": 25265,
      "digest": "sha256:xxxx8075264b5ba7c14c23672xxxx52ae6a3ebac1c47916e4efe19cd624dxxxx",
      "repository": "<repository-name>",
      "tag": "<chart-name>-<version>.tgz",
      "name": "<chart-name>",
      "version": "<version>"
    }
  },
  "dataVersion": "1.0",
  "metadataVersion": "1"
}
```

#### Triggering Actions
- `az acr helm push <chart>.tgz --name <registry>`
- `helm push <chart>.tgz oci://<registry>.azurecr.io/helm`

### 4. Microsoft.ContainerRegistry.ChartDeleted

Triggered when a Helm chart or chart repository is deleted.

#### Schema

```json
{
  "id": "338a3ef7-ad68-4128-xxxx-fdd3af8e8f67",
  "topic": "/subscriptions/<subscription-id>/resourceGroups/<resource-group>/providers/Microsoft.ContainerRegistry/registries/<registry-name>",
  "subject": "<repository>:<chart-version>",
  "eventType": "Microsoft.ContainerRegistry.ChartDeleted",
  "eventTime": "2023-03-06T00:10:48.1270754Z",
  "data": {
    "id": "338a3ef7-ad68-4128-xxxx-fdd3af8e8f67",
    "timestamp": "2023-03-06T00:10:48.1270754Z",
    "action": "chart_delete",
    "target": {
      "mediaType": "application/vnd.acr.helm.chart",
      "size": 25265,
      "digest": "sha256:xxxx8075264b5ba7c14c23672xxxx52ae6a3ebac1c47916e4efe19cd624dxxxx",
      "repository": "<repository-name>",
      "tag": "<chart-name>-<version>.tgz",
      "name": "<chart-name>",
      "version": "<version>"
    }
  },
  "dataVersion": "1.0",
  "metadataVersion": "1"
}
```

#### Triggering Actions
- `az acr helm delete <chart-name> --version <version> --name <registry>`

## Connected Registry Events

Connected registries (on-premises ACR) emit the same event types with an additional `connectedregistry` field in the data payload.

### Connected Registry ImagePushed Event

```json
{
  "id": "831e1650-001e-001b-66ab-eeb76e069631",
  "topic": "/subscriptions/<subscription-id>/resourceGroups/<resource-group>/providers/Microsoft.ContainerRegistry/registries/<registry-name>/connectedRegistries/<connected-registry-name>",
  "subject": "aci-helloworld:v1",
  "eventType": "Microsoft.ContainerRegistry.ImagePushed",
  "eventTime": "2018-04-25T21:39:47.6549614Z",
  "data": {
    "id": "31c51664-e5bd-416a-a5df-e5206bc47ed0",
    "timestamp": "2018-04-25T21:39:47.276585742Z",
    "action": "push",
    "target": {
      "mediaType": "application/vnd.docker.distribution.manifest.v2+json",
      "size": 3023,
      "digest": "sha256:213bbc182920ab41e18edc2001e06abcca6735d87782d9cef68abd83941cf0e5",
      "repository": "aci-helloworld",
      "tag": "v1",
      "length": 3023
    },
    "connectedregistry": {
      "name": "<connected-registry-name>"
    }
  },
  "dataVersion": "2.0",
  "metadataVersion": "1"
}
```

### Connected Registry ImageDeleted Event

```json
{
  "id": "831e1650-001e-001b-66ab-eeb76e069631",
  "topic": "/subscriptions/<subscription-id>/resourceGroups/<resource-group>/providers/Microsoft.ContainerRegistry/registries/<registry-name>/connectedRegistries/<connected-registry-name>",
  "subject": "aci-helloworld",
  "eventType": "Microsoft.ContainerRegistry.ImageDeleted",
  "eventTime": "2018-04-25T21:39:47.6549614Z",
  "data": {
    "id": "31c51664-e5bd-416a-a5df-e5206bc47ed0",
    "timestamp": "2018-04-25T21:39:47.276585742Z",
    "action": "delete",
    "target": {
      "mediaType": "application/vnd.docker.distribution.manifest.v2+json",
      "digest": "sha256:213bbc182920ab41e18edc2001e06abcca6735d87782d9cef68abd83941cf0e5",
      "repository": "aci-helloworld"
    },
    "connectedregistry": {
      "name": "<connected-registry-name>"
    }
  },
  "dataVersion": "2.0",
  "metadataVersion": "1"
}
```

## Event Data Schema Reference

### Common Event Envelope Fields

| Field | Type | Description |
|-------|------|-------------|
| `id` | String | Unique identifier for the event |
| `topic` | String | Full resource path to the event source |
| `subject` | String | Publisher-defined path to the event subject (repository:tag) |
| `eventType` | String | Registered event type (e.g., `Microsoft.ContainerRegistry.ImagePushed`) |
| `eventTime` | DateTime | Time the event was generated (UTC) |
| `data` | Object | Event-specific data payload |
| `dataVersion` | String | Schema version of the data object |
| `metadataVersion` | String | Schema version of the event metadata |

### Target Object Fields

| Field | Type | Description |
|-------|------|-------------|
| `mediaType` | String | MIME type of the artifact |
| `size` | Int32 | Size of content in bytes |
| `digest` | String | Content digest (SHA256) |
| `repository` | String | Repository name |
| `tag` | String | Image/chart tag name |
| `length` | Int32 | Size in bytes (same as size) |
| `name` | String | Chart name (Helm charts only) |
| `version` | String | Chart version (Helm charts only) |

### Request Object Fields

| Field | Type | Description |
|-------|------|-------------|
| `id` | String | Request ID that initiated the event |
| `host` | String | Registry hostname |
| `method` | String | HTTP method (PUT, DELETE) |
| `useragent` | String | User agent of the request |

### Connected Registry Object Fields

| Field | Type | Description |
|-------|------|-------------|
| `name` | String | Name of the connected registry |

## Media Types

| Media Type | Description |
|------------|-------------|
| `application/vnd.docker.distribution.manifest.v2+json` | Docker V2 manifest |
| `application/vnd.oci.image.manifest.v1+json` | OCI image manifest |
| `application/vnd.acr.helm.chart` | Helm chart |

## Webhook Events (Comparison)

ACR webhooks use a similar but distinct schema. Key differences:

| Aspect | Event Grid | Webhooks |
|--------|------------|----------|
| Envelope | CloudEvents/Event Grid schema | ACR-specific schema |
| Event Types | `Microsoft.ContainerRegistry.*` | `push`, `delete`, `chart_push`, `chart_delete`, `quarantine` |
| Additional Events | N/A | `quarantine` (for quarantine workflow) |

### Webhook-Specific: Quarantine Event

Webhooks support a `quarantine` action not available in Event Grid:

```json
{
  "id": "0d799b14-404b-4859-b2f6-50c5ee2a2c3a",
  "timestamp": "2018-02-28T00:42:54.4509516Z",
  "action": "quarantine",
  "target": {
    "size": 1791,
    "digest": "sha256:91ef6",
    "length": 1791,
    "repository": "helloworld",
    "tag": "1"
  },
  "request": {
    "id": "978fc988-zzz-yyyy-xxxx-4f6e331d1591",
    "host": "[registry].azurecr.io",
    "method": "PUT"
  }
}
```

## References

- Source: `/submodules/azure-management-docs/articles/container-registry/container-registry-event-grid-quickstart.md`
- Source: `/submodules/azure-management-docs/articles/container-registry/container-registry-webhook-reference.md`
- Source: `/submodules/acr/docs/preview/connected-registry/quickstart-send-connected-registry-events-to-event-grid.md`
- Source: `/submodules/acr/docs/preview/quarantine/readme.md`

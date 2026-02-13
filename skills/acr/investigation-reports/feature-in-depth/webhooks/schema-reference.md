# ACR Webhook Schema Reference

## Overview

This document provides complete schema reference for all webhook event payloads sent by Azure Container Registry. When a webhook is triggered, ACR sends an HTTP POST request to your endpoint with a JSON payload containing event information.

## HTTP Request Format

### Headers

| Header | Value | Notes |
|--------|-------|-------|
| Content-Type | `application/json` | Default unless overridden |
| Custom Headers | User-defined | Specified during webhook creation |

### Method

All webhook requests use HTTP `POST`.

---

## Common Schema Elements

### Base Event Structure

All webhook events share this base structure:

```json
{
  "id": "string",
  "timestamp": "datetime",
  "action": "string",
  "target": { ... },
  "request": { ... }
}
```

| Field | Type | Description |
|-------|------|-------------|
| `id` | String | Unique identifier for the webhook event |
| `timestamp` | DateTime | ISO 8601 timestamp when the event was triggered |
| `action` | String | The action type: `push`, `delete`, `chart_push`, `chart_delete`, `quarantine` |
| `target` | Object | Information about the artifact that triggered the event |
| `request` | Object | Information about the request that generated the event |

---

## Push Event Schema

### Action
`push`

### Complete Payload Structure

```json
{
  "id": "string",
  "timestamp": "datetime",
  "action": "push",
  "target": {
    "mediaType": "string",
    "size": "integer",
    "digest": "string",
    "length": "integer",
    "repository": "string",
    "tag": "string"
  },
  "request": {
    "id": "string",
    "host": "string",
    "method": "string",
    "useragent": "string"
  }
}
```

### Target Object (Push)

| Field | Type | Description |
|-------|------|-------------|
| `mediaType` | String | MIME type of the manifest (e.g., `application/vnd.docker.distribution.manifest.v2+json`) |
| `size` | Int32 | Number of bytes of the content |
| `digest` | String | Digest of the content (sha256 hash) |
| `length` | Int32 | Number of bytes of the content (same as size) |
| `repository` | String | Repository name |
| `tag` | String | Image tag name |

### Request Object (Push)

| Field | Type | Description |
|-------|------|-------------|
| `id` | String | Unique identifier for the request that initiated the event |
| `host` | String | Registry hostname (e.g., `myregistry.azurecr.io`) |
| `method` | String | HTTP method that generated the event (typically `PUT`) |
| `useragent` | String | User agent string of the client |

### Example Push Payload

```json
{
  "id": "cb8c3971-9adc-488b-xxxx-43cbb4974ff5",
  "timestamp": "2023-11-17T16:52:01.343145347Z",
  "action": "push",
  "target": {
    "mediaType": "application/vnd.docker.distribution.manifest.v2+json",
    "size": 524,
    "digest": "sha256:xxxxd5c8786bb9e621a45ece0dbxxxx1cdc624ad20da9fe62e9d25490f33xxxx",
    "length": 524,
    "repository": "hello-world",
    "tag": "v1"
  },
  "request": {
    "id": "3cbb6949-7549-4fa1-xxxx-a6d5451dffc7",
    "host": "myregistry.azurecr.io",
    "method": "PUT",
    "useragent": "docker/17.09.0-ce go/go1.8.3 git-commit/afdb6d4 kernel/4.10.0-27-generic os/linux arch/amd64 UpstreamClient(Docker-Client/17.09.0-ce \\(linux\\))"
  }
}
```

---

## Delete Event Schema

### Action
`delete`

### Complete Payload Structure

```json
{
  "id": "string",
  "timestamp": "datetime",
  "action": "delete",
  "target": {
    "mediaType": "string",
    "digest": "string",
    "repository": "string"
  },
  "request": {
    "id": "string",
    "host": "string",
    "method": "string",
    "useragent": "string"
  }
}
```

### Target Object (Delete)

| Field | Type | Description |
|-------|------|-------------|
| `mediaType` | String | MIME type of the deleted manifest |
| `digest` | String | Digest of the deleted content |
| `repository` | String | Repository name |

**Note**: The delete target does NOT include `tag`, `size`, or `length` fields.

### Request Object (Delete)

| Field | Type | Description |
|-------|------|-------------|
| `id` | String | Unique identifier for the request that initiated the event |
| `host` | String | Registry hostname |
| `method` | String | HTTP method (typically `DELETE`) |
| `useragent` | String | User agent string of the client |

### Example Delete Payload

```json
{
  "id": "afc359ce-df7f-4e32-xxxx-1ff8aa80927b",
  "timestamp": "2023-11-17T16:54:53.657764628Z",
  "action": "delete",
  "target": {
    "mediaType": "application/vnd.docker.distribution.manifest.v2+json",
    "digest": "sha256:xxxxd5c8786bb9e621a45ece0dbxxxx1cdc624ad20da9fe62e9d25490f33xxxx",
    "repository": "hello-world"
  },
  "request": {
    "id": "3d78b540-ab61-4f75-xxxx-7ca9ecf559b3",
    "host": "myregistry.azurecr.io",
    "method": "DELETE",
    "useragent": "python-requests/2.18.4"
  }
}
```

---

## Chart Push Event Schema

### Action
`chart_push`

### Complete Payload Structure

```json
{
  "id": "string",
  "timestamp": "datetime",
  "action": "chart_push",
  "target": {
    "mediaType": "string",
    "size": "integer",
    "digest": "string",
    "repository": "string",
    "tag": "string",
    "name": "string",
    "version": "string"
  }
}
```

### Target Object (Chart Push)

| Field | Type | Description |
|-------|------|-------------|
| `mediaType` | String | MIME type (typically `application/vnd.acr.helm.chart`) |
| `size` | Int32 | Number of bytes of the chart |
| `digest` | String | Digest of the chart content |
| `repository` | String | Repository name |
| `tag` | String | Chart tag name (usually the tgz filename) |
| `name` | String | Helm chart name |
| `version` | String | Helm chart version |

**Note**: Chart push events do NOT include a `request` object.

### Example Chart Push Payload

```json
{
  "id": "6356e9e0-627f-4fed-xxxx-d9059b5143ac",
  "timestamp": "2023-03-05T23:45:31.2614267Z",
  "action": "chart_push",
  "target": {
    "mediaType": "application/vnd.acr.helm.chart",
    "size": 25265,
    "digest": "sha256:xxxx8075264b5ba7c14c23672xxxx52ae6a3ebac1c47916e4efe19cd624dxxxx",
    "repository": "repo",
    "tag": "wordpress-5.4.0.tgz",
    "name": "wordpress",
    "version": "5.4.0.tgz"
  }
}
```

---

## Chart Delete Event Schema

### Action
`chart_delete`

### Complete Payload Structure

```json
{
  "id": "string",
  "timestamp": "datetime",
  "action": "chart_delete",
  "target": {
    "mediaType": "string",
    "size": "integer",
    "digest": "string",
    "repository": "string",
    "tag": "string",
    "name": "string",
    "version": "string"
  }
}
```

### Target Object (Chart Delete)

| Field | Type | Description |
|-------|------|-------------|
| `mediaType` | String | MIME type (`application/vnd.acr.helm.chart`) |
| `size` | Int32 | Number of bytes of the deleted chart |
| `digest` | String | Digest of the deleted chart content |
| `repository` | String | Repository name |
| `tag` | String | Chart tag name |
| `name` | String | Helm chart name |
| `version` | String | Helm chart version |

### Example Chart Delete Payload

```json
{
  "id": "338a3ef7-ad68-4128-xxxx-fdd3af8e8f67",
  "timestamp": "2023-03-06T00:10:48.1270754Z",
  "action": "chart_delete",
  "target": {
    "mediaType": "application/vnd.acr.helm.chart",
    "size": 25265,
    "digest": "sha256:xxxx8075264b5ba7c14c23672xxxx52ae6a3ebac1c47916e4efe19cd624dxxxx",
    "repository": "repo",
    "tag": "wordpress-5.4.0.tgz",
    "name": "wordpress",
    "version": "5.4.0.tgz"
  }
}
```

---

## Quarantine Event Schema

### Action
`quarantine`

### Complete Payload Structure

```json
{
  "id": "string",
  "timestamp": "datetime",
  "action": "quarantine",
  "target": {
    "size": "integer",
    "digest": "string",
    "length": "integer",
    "repository": "string",
    "tag": "string"
  },
  "request": {
    "id": "string",
    "host": "string",
    "method": "string"
  }
}
```

### Target Object (Quarantine)

| Field | Type | Description |
|-------|------|-------------|
| `size` | Int32 | Number of bytes of the content |
| `digest` | String | Digest of the content |
| `length` | Int32 | Number of bytes of the content |
| `repository` | String | Repository name |
| `tag` | String | Image tag name |

### Request Object (Quarantine)

| Field | Type | Description |
|-------|------|-------------|
| `id` | String | Unique identifier for the request |
| `host` | String | Registry hostname |
| `method` | String | HTTP method (typically `PUT`) |

### Example Quarantine Payload

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
    "id": "978fc988-zzz-yyyy-xxxx-4f6e331d1591",
    "host": "[registry].azurecr.io",
    "method": "PUT"
  }
}
```

---

## Media Types Reference

| Media Type | Description |
|------------|-------------|
| `application/vnd.docker.distribution.manifest.v2+json` | Docker V2 manifest |
| `application/vnd.docker.distribution.manifest.list.v2+json` | Docker manifest list (multi-arch) |
| `application/vnd.oci.image.manifest.v1+json` | OCI image manifest |
| `application/vnd.oci.image.index.v1+json` | OCI image index |
| `application/vnd.acr.helm.chart` | Helm chart |

---

## Ping Event Schema

When you use the ping feature to test a webhook, ACR sends a generic POST request:

```json
{
  "id": "ping-test-id",
  "timestamp": "2023-11-17T12:00:00Z",
  "action": "ping"
}
```

The ping does not include target or request objects.

---

## Handling Webhook Payloads

### Sample Webhook Handler (Python)

```python
from flask import Flask, request, jsonify

app = Flask(__name__)

@app.route('/webhook', methods=['POST'])
def handle_webhook():
    event = request.get_json()

    action = event.get('action')

    if action == 'push':
        repository = event['target']['repository']
        tag = event['target']['tag']
        digest = event['target']['digest']
        print(f"Image pushed: {repository}:{tag} ({digest})")

    elif action == 'delete':
        repository = event['target']['repository']
        digest = event['target']['digest']
        print(f"Image deleted: {repository} ({digest})")

    elif action == 'chart_push':
        name = event['target']['name']
        version = event['target']['version']
        print(f"Chart pushed: {name}:{version}")

    elif action == 'chart_delete':
        name = event['target']['name']
        version = event['target']['version']
        print(f"Chart deleted: {name}:{version}")

    elif action == 'quarantine':
        repository = event['target']['repository']
        tag = event['target']['tag']
        print(f"Image quarantined: {repository}:{tag}")

    return jsonify({'status': 'ok'}), 200

if __name__ == '__main__':
    app.run(port=5000)
```

### Sample Webhook Handler (Node.js)

```javascript
const express = require('express');
const app = express();

app.use(express.json());

app.post('/webhook', (req, res) => {
    const event = req.body;

    switch(event.action) {
        case 'push':
            console.log(`Image pushed: ${event.target.repository}:${event.target.tag}`);
            break;
        case 'delete':
            console.log(`Image deleted: ${event.target.repository}`);
            break;
        case 'chart_push':
            console.log(`Chart pushed: ${event.target.name}:${event.target.version}`);
            break;
        case 'chart_delete':
            console.log(`Chart deleted: ${event.target.name}:${event.target.version}`);
            break;
        case 'quarantine':
            console.log(`Image quarantined: ${event.target.repository}:${event.target.tag}`);
            break;
    }

    res.status(200).json({ status: 'ok' });
});

app.listen(5000, () => console.log('Webhook handler listening on port 5000'));
```

---

## Schema Validation

### JSON Schema for Push Event

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "required": ["id", "timestamp", "action", "target", "request"],
  "properties": {
    "id": { "type": "string" },
    "timestamp": { "type": "string", "format": "date-time" },
    "action": { "const": "push" },
    "target": {
      "type": "object",
      "required": ["mediaType", "digest", "repository"],
      "properties": {
        "mediaType": { "type": "string" },
        "size": { "type": "integer" },
        "digest": { "type": "string" },
        "length": { "type": "integer" },
        "repository": { "type": "string" },
        "tag": { "type": "string" }
      }
    },
    "request": {
      "type": "object",
      "required": ["id", "host", "method"],
      "properties": {
        "id": { "type": "string" },
        "host": { "type": "string" },
        "method": { "type": "string" },
        "useragent": { "type": "string" }
      }
    }
  }
}
```

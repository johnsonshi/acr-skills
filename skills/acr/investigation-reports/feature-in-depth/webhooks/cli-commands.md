# ACR Webhook CLI Commands Reference

## Overview

This document provides a comprehensive reference for all Azure CLI commands related to ACR webhooks. All commands are part of the `az acr webhook` command group.

## Command Summary

| Command | Description |
|---------|-------------|
| `az acr webhook create` | Create a new webhook |
| `az acr webhook delete` | Delete a webhook |
| `az acr webhook get-config` | Get webhook service URI and headers |
| `az acr webhook list` | List webhooks for a registry |
| `az acr webhook list-events` | List recent events for a webhook |
| `az acr webhook ping` | Send a test ping to a webhook |
| `az acr webhook show` | Get details of a webhook |
| `az acr webhook update` | Update a webhook |

---

## az acr webhook create

Create a new webhook for a container registry.

### Syntax

```bash
az acr webhook create --name
                      --registry
                      --uri
                      --actions {chart_delete, chart_push, delete, push, quarantine}
                      [--headers]
                      [--location]
                      [--resource-group]
                      [--scope]
                      [--status {disabled, enabled}]
                      [--tags]
```

### Required Parameters

| Parameter | Description |
|-----------|-------------|
| `--name` / `-n` | Name of the webhook (5-50 alphanumeric characters) |
| `--registry` / `-r` | Name of the container registry |
| `--uri` | Service URI for the webhook to POST notifications |
| `--actions` | Space-separated list of actions that trigger the webhook |

### Optional Parameters

| Parameter | Description | Default |
|-----------|-------------|---------|
| `--headers` | Space-separated custom headers in `key=value` format | None |
| `--location` / `-l` | Location for geo-replicated registries | Registry location |
| `--resource-group` / `-g` | Resource group name | Registry's resource group |
| `--scope` | Repository or tag scope (e.g., `repo:tag` or `repo:*`) | All repositories |
| `--status` | Enable or disable the webhook | `enabled` |
| `--tags` | Space-separated tags in `key=value` format | None |

### Examples

**Basic webhook for push events**:
```bash
az acr webhook create \
  --name mypushwebhook \
  --registry myregistry \
  --actions push \
  --uri https://myservice.example.com/webhook
```

**Webhook with multiple actions**:
```bash
az acr webhook create \
  --name mywebhook \
  --registry myregistry \
  --actions push delete chart_push chart_delete \
  --uri https://myservice.example.com/webhook
```

**Webhook with custom headers for authentication**:
```bash
az acr webhook create \
  --name myauthwebhook \
  --registry myregistry \
  --actions push \
  --uri https://myservice.example.com/webhook \
  --headers "Authorization=Bearer mytoken123" "X-Custom-Header=value"
```

**Webhook with repository scope**:
```bash
az acr webhook create \
  --name myscoped webhook \
  --registry myregistry \
  --actions push \
  --uri https://myservice.example.com/webhook \
  --scope myrepository:*
```

**Webhook for specific tag**:
```bash
az acr webhook create \
  --name mytagwebhook \
  --registry myregistry \
  --actions push \
  --uri https://myservice.example.com/webhook \
  --scope production/myapp:latest
```

**Webhook for geo-replicated registry (specific location)**:
```bash
az acr webhook create \
  --name myregionalwebhook \
  --registry myregistry \
  --actions push \
  --uri https://myservice.example.com/webhook \
  --location eastus
```

**Create webhook in disabled state**:
```bash
az acr webhook create \
  --name mydisabledwebhook \
  --registry myregistry \
  --actions push \
  --uri https://myservice.example.com/webhook \
  --status disabled
```

**Webhook for quarantine events**:
```bash
az acr webhook create \
  --name myquarantinewebhook \
  --registry myregistry \
  --actions quarantine \
  --uri https://scanner.example.com/webhook
```

---

## az acr webhook delete

Delete a webhook from a container registry.

### Syntax

```bash
az acr webhook delete --name
                      --registry
                      [--resource-group]
```

### Parameters

| Parameter | Required | Description |
|-----------|----------|-------------|
| `--name` / `-n` | Yes | Name of the webhook to delete |
| `--registry` / `-r` | Yes | Name of the container registry |
| `--resource-group` / `-g` | No | Resource group name |

### Examples

```bash
# Delete a webhook
az acr webhook delete --registry myregistry --name mywebhook

# Delete with explicit resource group
az acr webhook delete \
  --registry myregistry \
  --name mywebhook \
  --resource-group myresourcegroup
```

---

## az acr webhook get-config

Get the service URI and custom headers for a webhook.

### Syntax

```bash
az acr webhook get-config --name
                          --registry
                          [--resource-group]
```

### Parameters

| Parameter | Required | Description |
|-----------|----------|-------------|
| `--name` / `-n` | Yes | Name of the webhook |
| `--registry` / `-r` | Yes | Name of the container registry |
| `--resource-group` / `-g` | No | Resource group name |

### Examples

```bash
# Get webhook configuration
az acr webhook get-config --registry myregistry --name mywebhook
```

### Sample Output

```json
{
  "customHeaders": {
    "Authorization": "Bearer mytoken123"
  },
  "serviceUri": "https://myservice.example.com/webhook"
}
```

---

## az acr webhook list

List all webhooks for a container registry.

### Syntax

```bash
az acr webhook list --registry
                    [--resource-group]
```

### Parameters

| Parameter | Required | Description |
|-----------|----------|-------------|
| `--registry` / `-r` | Yes | Name of the container registry |
| `--resource-group` / `-g` | No | Resource group name |

### Examples

```bash
# List all webhooks
az acr webhook list --registry myregistry

# List webhooks in table format
az acr webhook list --registry myregistry --output table

# List webhooks with specific resource group
az acr webhook list --registry myregistry --resource-group myresourcegroup
```

### Sample Output (Table)

```
Name               Location    Status    Scope         Actions
-----------------  ----------  --------  ------------  -----------
mypushwebhook      westus      enabled   myrepo:*      push
mydeletewebhook    westus      enabled                 delete
mychartwebhook     eastus      disabled  charts/*      chart_push
```

---

## az acr webhook list-events

List recent events for a webhook.

### Syntax

```bash
az acr webhook list-events --name
                           --registry
                           [--resource-group]
```

### Parameters

| Parameter | Required | Description |
|-----------|----------|-------------|
| `--name` / `-n` | Yes | Name of the webhook |
| `--registry` / `-r` | Yes | Name of the container registry |
| `--resource-group` / `-g` | No | Resource group name |

### Examples

```bash
# List events for a webhook
az acr webhook list-events --registry myregistry --name mywebhook

# List events in table format
az acr webhook list-events --registry myregistry --name mywebhook --output table
```

### Sample Output

```json
[
  {
    "eventRequestMessage": {
      "content": {
        "action": "push",
        "id": "cb8c3971-9adc-488b-xxxx-43cbb4974ff5",
        "target": {
          "digest": "sha256:abc123...",
          "repository": "hello-world",
          "tag": "v1"
        },
        "timestamp": "2023-11-17T16:52:01.343145347Z"
      },
      "headers": {
        "Content-Type": "application/json"
      },
      "method": "POST",
      "requestUri": "https://myservice.example.com/webhook",
      "version": "1.1"
    },
    "eventResponseMessage": {
      "content": "",
      "headers": {
        "Content-Length": "0",
        "Date": "Fri, 17 Nov 2023 16:52:02 GMT"
      },
      "reasonPhrase": "OK",
      "statusCode": "200",
      "version": "1.1"
    },
    "id": "event-id-12345"
  }
]
```

---

## az acr webhook ping

Trigger a ping event for a webhook (for testing).

### Syntax

```bash
az acr webhook ping --name
                    --registry
                    [--resource-group]
```

### Parameters

| Parameter | Required | Description |
|-----------|----------|-------------|
| `--name` / `-n` | Yes | Name of the webhook |
| `--registry` / `-r` | Yes | Name of the container registry |
| `--resource-group` / `-g` | No | Resource group name |

### Examples

```bash
# Ping a webhook to test connectivity
az acr webhook ping --registry myregistry --name mywebhook
```

### Sample Output

```json
{
  "id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890"
}
```

### Verification

After pinging, use `list-events` to see the response:
```bash
az acr webhook list-events --registry myregistry --name mywebhook
```

---

## az acr webhook show

Get details of a specific webhook.

### Syntax

```bash
az acr webhook show --name
                    --registry
                    [--resource-group]
```

### Parameters

| Parameter | Required | Description |
|-----------|----------|-------------|
| `--name` / `-n` | Yes | Name of the webhook |
| `--registry` / `-r` | Yes | Name of the container registry |
| `--resource-group` / `-g` | No | Resource group name |

### Examples

```bash
# Show webhook details
az acr webhook show --registry myregistry --name mywebhook

# Show specific property
az acr webhook show --registry myregistry --name mywebhook --query status

# Show in YAML format
az acr webhook show --registry myregistry --name mywebhook --output yaml
```

### Sample Output

```json
{
  "actions": [
    "push"
  ],
  "id": "/subscriptions/.../providers/Microsoft.ContainerRegistry/registries/myregistry/webhooks/mywebhook",
  "location": "westus",
  "name": "mywebhook",
  "provisioningState": "Succeeded",
  "resourceGroup": "myresourcegroup",
  "scope": "myrepo:*",
  "status": "enabled",
  "tags": {},
  "type": "Microsoft.ContainerRegistry/registries/webhooks"
}
```

---

## az acr webhook update

Update an existing webhook.

### Syntax

```bash
az acr webhook update --name
                      --registry
                      [--actions {chart_delete, chart_push, delete, push, quarantine}]
                      [--add]
                      [--force-string]
                      [--headers]
                      [--remove]
                      [--resource-group]
                      [--scope]
                      [--set]
                      [--status {disabled, enabled}]
                      [--tags]
                      [--uri]
```

### Parameters

| Parameter | Required | Description |
|-----------|----------|-------------|
| `--name` / `-n` | Yes | Name of the webhook |
| `--registry` / `-r` | Yes | Name of the container registry |
| `--actions` | No | Updated list of actions |
| `--headers` | No | Updated custom headers |
| `--scope` | No | Updated scope |
| `--status` | No | Enable or disable webhook |
| `--uri` | No | Updated service URI |
| `--tags` | No | Updated tags |
| `--resource-group` / `-g` | No | Resource group name |

### Examples

**Update webhook URI**:
```bash
az acr webhook update \
  --registry myregistry \
  --name mywebhook \
  --uri https://newservice.example.com/webhook
```

**Update actions**:
```bash
az acr webhook update \
  --registry myregistry \
  --name mywebhook \
  --actions push delete chart_push
```

**Enable a disabled webhook**:
```bash
az acr webhook update \
  --registry myregistry \
  --name mywebhook \
  --status enabled
```

**Disable a webhook**:
```bash
az acr webhook update \
  --registry myregistry \
  --name mywebhook \
  --status disabled
```

**Update custom headers**:
```bash
az acr webhook update \
  --registry myregistry \
  --name mywebhook \
  --headers "Authorization=Bearer newtoken" "X-New-Header=value"
```

**Update scope**:
```bash
az acr webhook update \
  --registry myregistry \
  --name mywebhook \
  --scope newrepository:*
```

**Update multiple properties**:
```bash
az acr webhook update \
  --registry myregistry \
  --name mywebhook \
  --uri https://newservice.example.com/webhook \
  --actions push delete \
  --scope myrepo:* \
  --status enabled
```

---

## Common Workflows

### Create and Test a Webhook

```bash
# Create the webhook
az acr webhook create \
  --registry myregistry \
  --name mywebhook \
  --actions push \
  --uri https://myservice.example.com/webhook

# Test the webhook
az acr webhook ping --registry myregistry --name mywebhook

# Check the test results
az acr webhook list-events --registry myregistry --name mywebhook
```

### Set Up Webhooks for CI/CD

```bash
# Create webhook for build pipeline
az acr webhook create \
  --registry myregistry \
  --name build-trigger \
  --actions push \
  --uri https://dev.azure.com/myorg/myproject/_apis/webhooks/receivers/genericweb \
  --scope myapp:* \
  --headers "Authorization=Bearer $(PIPELINE_TOKEN)"
```

### Set Up Regional Webhooks for Geo-Replicated Registry

```bash
# Create webhook for West US replica
az acr webhook create \
  --registry myregistry \
  --name westus-webhook \
  --actions push \
  --uri https://westus.myservice.example.com/webhook \
  --location westus

# Create webhook for East US replica
az acr webhook create \
  --registry myregistry \
  --name eastus-webhook \
  --actions push \
  --uri https://eastus.myservice.example.com/webhook \
  --location eastus
```

### Monitor Webhook Activity

```bash
# List all webhooks
az acr webhook list --registry myregistry --output table

# Check events for each webhook
for webhook in $(az acr webhook list --registry myregistry --query "[].name" -o tsv); do
  echo "=== Events for $webhook ==="
  az acr webhook list-events --registry myregistry --name $webhook --output table
done
```

---

## Error Handling

### Common Errors

| Error | Cause | Solution |
|-------|-------|----------|
| `WebhookNameInvalid` | Name not 5-50 alphanumeric chars | Use valid name format |
| `WebhookAlreadyExists` | Webhook name already in use | Choose different name |
| `WebhookNotFound` | Webhook doesn't exist | Check webhook name |
| `InvalidActions` | Invalid action specified | Use valid actions |

### Debugging Failed Webhooks

```bash
# Check webhook configuration
az acr webhook show --registry myregistry --name mywebhook

# View recent event responses
az acr webhook list-events --registry myregistry --name mywebhook

# Test connectivity
az acr webhook ping --registry myregistry --name mywebhook

# Get full configuration including URI
az acr webhook get-config --registry myregistry --name mywebhook
```

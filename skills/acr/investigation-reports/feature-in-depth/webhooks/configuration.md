# ACR Webhook Configuration Guide

## Overview

This guide covers all aspects of configuring webhooks in Azure Container Registry, including creation, testing, updating, and deletion using both the Azure Portal and Azure CLI.

## Configuration Options

### Webhook Properties

| Property | Required | Description | Constraints |
|----------|----------|-------------|-------------|
| Webhook name | Yes | Unique identifier for the webhook | 5-50 alphanumeric characters |
| Service URI | Yes | HTTPS/HTTP endpoint to receive notifications | Must be publicly accessible |
| Location | Conditional | Azure region for geo-replicated registries | Required for geo-replicated registries |
| Custom headers | No | Additional HTTP headers sent with requests | Format: `key: value` |
| Trigger actions | Yes | Events that trigger the webhook | `push`, `delete`, `chart_push`, `chart_delete`, `quarantine` |
| Status | No | Enable/disable the webhook | Default: enabled |
| Scope | No | Filter events by repository/tag | Format: `repository:tag` or `repository:*` |

---

## Configuration via Azure Portal

### Creating a Webhook

1. Sign in to the [Azure portal](https://portal.azure.com)
2. Navigate to your container registry
3. Under **Services**, select **Webhooks**
4. Click **Add** in the webhook toolbar
5. Complete the webhook form:

| Field | Value |
|-------|-------|
| **Webhook name** | Enter a unique name (5-50 alphanumeric characters) |
| **Location** | For geo-replicated registries, select the regional replica |
| **Service URI** | Enter the endpoint URL (e.g., `https://myservice.com/webhook`) |
| **Custom headers** | Optional: Add headers like `Authorization: Bearer <token>` |
| **Trigger actions** | Select one or more: push, delete, chart_push, chart_delete, quarantine |
| **Status** | Toggle enabled/disabled |
| **Scope** | Optional: Specify repository/tag filter (e.g., `myrepo:*`) |

6. Click **Create**

### Viewing Webhooks

1. Navigate to your container registry
2. Under **Services**, select **Webhooks**
3. View the list of configured webhooks with their status and location

### Testing a Webhook (Ping)

1. Select the webhook you want to test
2. In the top toolbar, click **Ping**
3. Check the **HTTP STATUS** column for the response
4. Review the response details to verify correct configuration

### Viewing Webhook Event History

1. Select a webhook from the list
2. View the **Event history** showing:
   - Timestamp of each event
   - Event type (push, delete, etc.)
   - HTTP status code of the response
   - Response details

### Updating a Webhook

1. Select the webhook to modify
2. Update the desired properties
3. Click **Save**

### Deleting a Webhook

1. Select the webhook to delete
2. Click the **Delete** button
3. Confirm the deletion

---

## Configuration via Azure CLI

### Creating a Webhook

**Basic Webhook Creation**:
```bash
az acr webhook create \
  --registry mycontainerregistry \
  --name myacrwebhook01 \
  --actions delete \
  --uri http://webhookuri.com
```

**Webhook with Multiple Actions**:
```bash
az acr webhook create \
  --registry mycontainerregistry \
  --name myacrwebhook02 \
  --actions push delete \
  --uri https://myservice.com/webhook
```

**Webhook with Custom Headers**:
```bash
az acr webhook create \
  --registry mycontainerregistry \
  --name myacrwebhook03 \
  --actions push \
  --uri https://myservice.com/webhook \
  --headers "Authorization=Bearer mytoken" "X-Custom-Header=value"
```

**Webhook with Scope**:
```bash
az acr webhook create \
  --registry mycontainerregistry \
  --name myacrwebhook04 \
  --actions push \
  --uri https://myservice.com/webhook \
  --scope myrepository:*
```

**Webhook for Geo-Replicated Registry (specific region)**:
```bash
az acr webhook create \
  --registry mycontainerregistry \
  --name myacrwebhook05 \
  --actions push \
  --uri https://myservice.com/webhook \
  --location eastus
```

**Webhook Created Disabled**:
```bash
az acr webhook create \
  --registry mycontainerregistry \
  --name myacrwebhook06 \
  --actions push \
  --uri https://myservice.com/webhook \
  --status disabled
```

### Listing Webhooks

```bash
az acr webhook list --registry mycontainerregistry
```

**Output in Table Format**:
```bash
az acr webhook list --registry mycontainerregistry --output table
```

### Showing Webhook Details

```bash
az acr webhook show --registry mycontainerregistry --name myacrwebhook01
```

### Testing a Webhook (Ping)

```bash
az acr webhook ping --registry mycontainerregistry --name myacrwebhook01
```

**Example Output**:
```json
{
  "id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890"
}
```

### Viewing Webhook Events

```bash
az acr webhook list-events --registry mycontainerregistry --name myacrwebhook01
```

**Example Output**:
```json
[
  {
    "eventRequestMessage": {
      "content": {...},
      "headers": {...},
      "method": "POST",
      "requestUri": "https://myservice.com/webhook",
      "version": "1.1"
    },
    "eventResponseMessage": {
      "content": "",
      "headers": {...},
      "reasonPhrase": "OK",
      "statusCode": "200",
      "version": "1.1"
    },
    "id": "event-id-123"
  }
]
```

### Updating a Webhook

**Update URI**:
```bash
az acr webhook update \
  --registry mycontainerregistry \
  --name myacrwebhook01 \
  --uri https://newuri.com/webhook
```

**Update Actions**:
```bash
az acr webhook update \
  --registry mycontainerregistry \
  --name myacrwebhook01 \
  --actions push delete chart_push
```

**Enable/Disable Webhook**:
```bash
# Disable
az acr webhook update \
  --registry mycontainerregistry \
  --name myacrwebhook01 \
  --status disabled

# Enable
az acr webhook update \
  --registry mycontainerregistry \
  --name myacrwebhook01 \
  --status enabled
```

**Update Headers**:
```bash
az acr webhook update \
  --registry mycontainerregistry \
  --name myacrwebhook01 \
  --headers "Authorization=Bearer newtoken"
```

**Update Scope**:
```bash
az acr webhook update \
  --registry mycontainerregistry \
  --name myacrwebhook01 \
  --scope newrepository:v1
```

### Deleting a Webhook

```bash
az acr webhook delete --registry mycontainerregistry --name myacrwebhook01
```

### Getting Webhook Callback URL

```bash
az acr webhook get-config --registry mycontainerregistry --name myacrwebhook01
```

---

## Configuration via REST API

### Create Webhook

```http
PUT https://management.azure.com/subscriptions/{subscriptionId}/resourceGroups/{resourceGroupName}/providers/Microsoft.ContainerRegistry/registries/{registryName}/webhooks/{webhookName}?api-version=2023-07-01

{
  "location": "westus",
  "properties": {
    "serviceUri": "https://myservice.com/webhook",
    "actions": ["push", "delete"],
    "status": "enabled",
    "scope": "myrepository:*",
    "customHeaders": {
      "Authorization": "Bearer mytoken"
    }
  }
}
```

### List Webhooks

```http
GET https://management.azure.com/subscriptions/{subscriptionId}/resourceGroups/{resourceGroupName}/providers/Microsoft.ContainerRegistry/registries/{registryName}/webhooks?api-version=2023-07-01
```

### Delete Webhook

```http
DELETE https://management.azure.com/subscriptions/{subscriptionId}/resourceGroups/{resourceGroupName}/providers/Microsoft.ContainerRegistry/registries/{registryName}/webhooks/{webhookName}?api-version=2023-07-01
```

---

## Configuration for Quarantine Webhooks

Quarantine webhooks are used for vulnerability scanning workflows:

### Enable Quarantine Policy

```bash
id=$(az acr show --name myregistry --query id -o tsv)
az resource update --ids $id --set properties.policies.quarantinePolicy.status=enabled
```

### Create Quarantine Webhook

Using the REST API:
```http
PUT https://management.azure.com/subscriptions/{subscriptionId}/resourceGroups/{resourceGroupName}/providers/Microsoft.ContainerRegistry/registries/{registryName}/webhooks/{webhookName}?api-version=2023-07-01

{
  "location": "westus",
  "properties": {
    "serviceUri": "https://scanner.com/webhook",
    "actions": ["quarantine"],
    "status": "enabled"
  }
}
```

---

## Best Practices

### Security
1. **Use HTTPS endpoints** for webhook URIs
2. **Implement authentication** via custom headers
3. **Validate webhook payloads** using signature verification

### Reliability
1. **Test webhooks** using ping before relying on them
2. **Monitor webhook events** regularly
3. **Implement retry logic** in your webhook handler

### Scope Management
1. **Use specific scopes** to reduce noise
2. **Create separate webhooks** for different workflows
3. **Use repository-level scopes** for CI/CD pipelines

### Geo-Replication
1. **Create regional webhooks** to track replication
2. **Monitor events per region** for deployment coordination
3. **Consider latency** between regions for event timing

---

## Troubleshooting Configuration Issues

### Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| Webhook not firing | Endpoint not accessible | Verify endpoint is publicly accessible |
| 401/403 responses | Authentication failure | Check custom headers and credentials |
| Webhook not created | Name constraints | Ensure 5-50 alphanumeric characters |
| Events not matching | Scope too restrictive | Verify scope matches expected repository/tag |

### Verification Steps

1. **Test connectivity**:
   ```bash
   az acr webhook ping --registry myregistry --name mywebhook
   ```

2. **Check event history**:
   ```bash
   az acr webhook list-events --registry myregistry --name mywebhook
   ```

3. **Verify configuration**:
   ```bash
   az acr webhook show --registry myregistry --name mywebhook
   ```

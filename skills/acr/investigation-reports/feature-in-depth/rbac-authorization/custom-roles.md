# Azure Container Registry Custom Roles

## Overview

Azure Container Registry supports creating custom roles when built-in roles don't meet specific requirements. Custom roles allow fine-grained permissions tailored to unique scenarios.

## Custom Role Permissions

A custom role is defined by a set of permissions consisting of:
- **Actions** - Control plane operations on registries
- **Data Actions** - Data plane operations on images/artifacts
- **NotActions** - Excluded control plane operations
- **NotDataActions** - Excluded data plane operations

## Finding Available Permissions

### List All ACR Permissions via Azure CLI

```bash
az provider operation show --namespace Microsoft.ContainerRegistry
```

### List All ACR Permissions via Azure PowerShell

```powershell
Get-AzProviderOperation -OperationSearchString Microsoft.ContainerRegistry/*
```

### Permission References

1. **Azure built-in roles for Containers**: Review JSON definitions at `/azure/role-based-access-control/built-in-roles/containers`
2. **Complete permission list**: `/azure/role-based-access-control/permissions/containers#microsoftcontainerregistry`

## Common Control Plane Actions (Microsoft.ContainerRegistry)

| Action | Description |
|--------|-------------|
| `*/read` | Read any registry resource |
| `registries/read` | List registries or get registry properties |
| `registries/write` | Create or update a registry |
| `registries/delete` | Delete a registry |
| `registries/webhooks/read` | List webhooks or get webhook properties |
| `registries/webhooks/write` | Create or update a webhook |
| `registries/webhooks/delete` | Delete a webhook |
| `registries/webhooks/getCallbackConfig/action` | Get webhook callback config |
| `registries/webhooks/ping/action` | Ping a webhook |
| `registries/webhooks/listEvents/action` | List webhook events |
| `registries/replications/read` | List replications or get replication properties |
| `registries/replications/write` | Create or update a replication |
| `registries/replications/delete` | Delete a replication |
| `registries/tasks/read` | List tasks or get task properties |
| `registries/tasks/write` | Create or update a task |
| `registries/tasks/delete` | Delete a task |
| `registries/runs/read` | List runs or get run properties |
| `registries/runs/write` | Create or update a run |
| `registries/runs/listLogSasUrl/action` | Get run log SAS URL |
| `registries/runs/cancel/action` | Cancel a run |
| `registries/agentPools/read` | List agent pools or get properties |
| `registries/agentPools/write` | Create or update an agent pool |
| `registries/agentPools/delete` | Delete an agent pool |
| `registries/listCredentials/action` | List registry login credentials |
| `registries/regenerateCredential/action` | Regenerate registry credential |
| `registries/listUsages/read` | List registry usage |
| `registries/privateEndpointConnections/read` | List private endpoint connections |
| `registries/privateEndpointConnections/write` | Approve/reject private endpoint |
| `registries/privateEndpointConnections/delete` | Delete private endpoint connection |
| `registries/importImage/action` | Import image to registry |
| `registries/tokens/read` | List tokens or get token properties |
| `registries/tokens/write` | Create or update a token |
| `registries/tokens/delete` | Delete a token |
| `registries/scopeMaps/read` | List scope maps or get properties |
| `registries/scopeMaps/write` | Create or update a scope map |
| `registries/scopeMaps/delete` | Delete a scope map |
| `registries/connectedRegistries/read` | List connected registries |
| `registries/connectedRegistries/write` | Create or update connected registry |
| `registries/connectedRegistries/delete` | Delete connected registry |
| `registries/exportPipelines/read` | List export pipelines |
| `registries/exportPipelines/write` | Create or update export pipeline |
| `registries/exportPipelines/delete` | Delete export pipeline |
| `registries/importPipelines/read` | List import pipelines |
| `registries/importPipelines/write` | Create or update import pipeline |
| `registries/importPipelines/delete` | Delete import pipeline |
| `registries/pipelineRuns/read` | List pipeline runs |
| `registries/pipelineRuns/write` | Create or update pipeline run |
| `registries/pipelineRuns/delete` | Delete pipeline run |

## Common Data Actions (Microsoft.ContainerRegistry)

| Data Action | Description |
|-------------|-------------|
| `registries/repositories/content/read` | Pull/read artifacts from a repository |
| `registries/repositories/content/write` | Push/write artifacts to a repository |
| `registries/repositories/content/delete` | Delete artifacts from a repository |
| `registries/repositories/metadata/read` | Read repository metadata (tags, manifests) |
| `registries/repositories/metadata/write` | Write repository metadata |
| `registries/catalog/read` | List all repositories in the registry |
| `registries/quarantine/read` | Read quarantined artifacts |
| `registries/quarantine/write` | Modify quarantine state |
| `registries/sign/write` | Sign content with content trust |

## Example: Custom Role for Webhook Management

This custom role allows managing webhooks only:

```json
{
   "assignableScopes": [
     "/subscriptions/<subscription-id>"
   ],
   "description": "Manage Azure Container Registry webhooks.",
   "Name": "Container Registry Webhook Contributor",
   "permissions": [
     {
       "actions": [
         "Microsoft.ContainerRegistry/registries/webhooks/read",
         "Microsoft.ContainerRegistry/registries/webhooks/write",
         "Microsoft.ContainerRegistry/registries/webhooks/delete"
       ],
       "dataActions": [],
       "notActions": [],
       "notDataActions": []
     }
   ],
   "roleType": "CustomRole"
}
```

## Example: Custom Role for Read-Only Task Access

```json
{
   "assignableScopes": [
     "/subscriptions/<subscription-id>"
   ],
   "description": "Read-only access to ACR Tasks.",
   "Name": "Container Registry Tasks Reader",
   "permissions": [
     {
       "actions": [
         "Microsoft.ContainerRegistry/registries/tasks/read",
         "Microsoft.ContainerRegistry/registries/runs/read",
         "Microsoft.ContainerRegistry/registries/runs/listLogSasUrl/action"
       ],
       "dataActions": [],
       "notActions": [],
       "notDataActions": []
     }
   ],
   "roleType": "CustomRole"
}
```

## Example: Custom Role for Repository Content Management

```json
{
   "assignableScopes": [
     "/subscriptions/<subscription-id>"
   ],
   "description": "Push, pull, and delete repository content.",
   "Name": "Container Registry Content Manager",
   "permissions": [
     {
       "actions": [],
       "dataActions": [
         "Microsoft.ContainerRegistry/registries/repositories/content/read",
         "Microsoft.ContainerRegistry/registries/repositories/content/write",
         "Microsoft.ContainerRegistry/registries/repositories/content/delete",
         "Microsoft.ContainerRegistry/registries/repositories/metadata/read",
         "Microsoft.ContainerRegistry/registries/repositories/metadata/write"
       ],
       "notActions": [],
       "notDataActions": []
     }
   ],
   "roleType": "CustomRole"
}
```

## Creating Custom Roles

### Using Azure CLI

```bash
# Create from JSON file
az role definition create --role-definition custom-role.json

# Update existing custom role
az role definition update --role-definition custom-role.json

# Delete custom role
az role definition delete --name "Container Registry Webhook Contributor"
```

### Using Azure PowerShell

```powershell
# Create custom role
New-AzRoleDefinition -InputFile custom-role.json

# Update custom role
Set-AzRoleDefinition -InputFile custom-role.json

# Delete custom role
Remove-AzRoleDefinition -Name "Container Registry Webhook Contributor"
```

### Using Azure Resource Manager Template

Custom roles can also be created via ARM templates. See the Azure documentation for template schema.

## Wildcard Actions

In tenants configured with **Azure Resource Manager private link**, ACR supports wildcard actions in custom roles:

```json
{
  "actions": [
    "Microsoft.ContainerRegistry/*/read",
    "Microsoft.ContainerRegistry/registries/*/write"
  ]
}
```

> **Important**: In tenants **without** ARM private link, don't use wildcards. Specify all required registry actions individually.

## Assigning Custom Roles

Custom roles are assigned the same way as built-in roles:

### Azure CLI

```bash
az role assignment create \
  --role "Container Registry Webhook Contributor" \
  --assignee user@example.com \
  --scope /subscriptions/<subscription-id>/resourceGroups/<rg>/providers/Microsoft.ContainerRegistry/registries/<registry-name>
```

### Azure PowerShell

```powershell
New-AzRoleAssignment `
  -RoleDefinitionName "Container Registry Webhook Contributor" `
  -SignInName "user@example.com" `
  -Scope "/subscriptions/<subscription-id>/resourceGroups/<rg>/providers/Microsoft.ContainerRegistry/registries/<registry-name>"
```

## Best Practices

1. **Start with built-in roles** - Only create custom roles when built-in roles don't meet requirements
2. **Follow least privilege** - Include only the minimum permissions needed
3. **Document custom roles** - Use clear names and descriptions
4. **Limit assignable scopes** - Restrict where the role can be assigned
5. **Test before deployment** - Verify permissions work as expected
6. **Review periodically** - Audit custom roles for continued necessity

## Source Documentation

- MS Learn: `/submodules/azure-management-docs/articles/container-registry/container-registry-rbac-custom-roles.md`
- Azure RBAC: `/azure/role-based-access-control/custom-roles`
- Azure RBAC: `/azure/role-based-access-control/custom-roles-cli`
- Azure RBAC: `/azure/role-based-access-control/custom-roles-powershell`
- Azure RBAC: `/azure/role-based-access-control/custom-roles-template`

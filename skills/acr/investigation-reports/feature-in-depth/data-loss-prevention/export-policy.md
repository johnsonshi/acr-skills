# Azure Container Registry Export Policy

## Overview

The **export policy** is a registry-level property that controls whether artifacts can be exported from a container registry. When disabled, it prevents data exfiltration by blocking attempts to copy artifacts to external locations.

## Technical Details

### API Information

| Property | Value |
|----------|-------|
| API Version | `2021-06-01-preview` |
| Property Path | `properties.policies.exportPolicy.status` |
| Resource Type | `Microsoft.ContainerRegistry/registries` |
| Values | `enabled` (default) or `disabled` |

### Property Structure

```json
{
  "properties": {
    "policies": {
      "exportPolicy": {
        "status": "enabled|disabled"
      }
    }
  }
}
```

## What Export Policy Controls

### Operations Blocked When Disabled

1. **Image Import TO Another Registry**
   - Prevents using `az acr import` with this registry as the source
   - Blocks the Data Importer role's ability to copy images out
   - Example blocked operation:
     ```bash
     # This will fail if source-registry has exportPolicy disabled
     az acr import \
       --name target-registry \
       --source source-registry.azurecr.io/myimage:tag \
       --image myimage:tag
     ```

2. **Export Pipeline Creation**
   - Prevents creating `ExportPipeline` resources via ACR Transfer
   - Blocks ARM template deployments that create export pipelines
   - Example blocked operation:
     ```bash
     # This will fail if the registry has exportPolicy disabled
     az acr export-pipeline create \
       --resource-group myRG \
       --registry protected-registry \
       --name myExportPipeline \
       --secret-uri https://mykv.vault.azure.net/secrets/sas \
       --storage-container-uri https://mystorage.blob.core.windows.net/transfer
     ```

### Operations Still Allowed

| Operation | Allowed? | Notes |
|-----------|----------|-------|
| `docker pull` | Yes | Within authorized network only |
| `docker push` | Yes | Standard push operations |
| `az acr import` (INTO registry) | Yes | Importing from external sources |
| `az acr repository list` | Yes | Metadata operations |
| `az acr manifest list-metadata` | Yes | Manifest inspection |
| Webhook triggers | Yes | Event notifications |

## Prerequisites for Disabling Exports

### 1. Premium SKU Required

Export policy is only available on Premium container registries.

```bash
# Check registry SKU
az acr show --name myregistry --query sku.name

# Upgrade to Premium if needed
az acr update --name myregistry --sku Premium
```

### 2. Private Endpoint Must Be Configured

The registry must have at least one private endpoint connection.

```bash
# List private endpoints
az acr private-endpoint-connection list --registry-name myregistry
```

### 3. Public Network Access Must Be Disabled

The `publicNetworkAccess` property must be set to `disabled` before or simultaneously with disabling export policy.

```bash
# Check public network access status
az acr show --name myregistry --query publicNetworkAccess
```

### 4. No Existing Export Pipelines

All export pipelines must be deleted before disabling export policy.

```bash
# List existing export pipelines
az acr export-pipeline list --registry myregistry --resource-group myRG

# Delete if any exist
az acr export-pipeline delete \
  --resource-group myRG \
  --registry myregistry \
  --name existing-pipeline
```

## API Behavior

### Enabling Export Policy (Default)

When `exportPolicy.status = "enabled"`:
- Artifacts can be imported to other registries
- Export pipelines can be created
- ACR Transfer operations work normally

### Disabling Export Policy

When `exportPolicy.status = "disabled"`:
- Import operations from this registry fail with authorization error
- Export pipeline creation attempts fail
- Existing import pipelines (importing INTO this registry) continue to work

## Error Messages

### Attempting Import from Protected Registry

```
(Forbidden) Import from source registry with export policy disabled is not allowed.
```

### Attempting to Create Export Pipeline

```
(BadRequest) Cannot create export pipeline. Export policy is disabled for registry 'myregistry'.
```

### Attempting to Disable Export with Existing Pipelines

```
(BadRequest) Cannot disable export policy. Registry has existing export pipelines. Delete all export pipelines before disabling export policy.
```

### Attempting to Disable Export Without Private Endpoint

```
(BadRequest) Export policy can only be disabled for registries with private endpoints configured.
```

## Interaction with Other Policies

The export policy works alongside other registry policies:

```json
{
  "properties": {
    "policies": {
      "exportPolicy": {
        "status": "disabled"
      },
      "quarantinePolicy": {
        "status": "disabled"
      },
      "retentionPolicy": {
        "days": 7,
        "status": "disabled"
      },
      "trustPolicy": {
        "status": "disabled",
        "type": "Notary"
      }
    },
    "publicNetworkAccess": "Disabled",
    "zoneRedundancy": "Disabled"
  }
}
```

## RBAC Considerations

### Roles that Can Modify Export Policy

- **Owner** - Full control over registry
- **Contributor** - Can modify registry settings

### Roles Affected by Export Policy

- **Container Registry Data Importer** - Import operations blocked when source registry has exports disabled
- **Container Registry Transfer Pipeline Contributor** - Export pipeline creation blocked

## Best Practices

1. **Plan Network Architecture First**
   - Ensure private endpoints are configured before enabling DLP
   - Document allowed access paths

2. **Audit Before Enabling**
   - Review existing import operations
   - Identify any dependencies on artifact export

3. **Use Diagnostic Logging**
   - Enable diagnostic settings before disabling exports
   - Monitor for failed access attempts

4. **Document Exception Process**
   - Establish procedure for temporarily re-enabling exports if needed
   - Require multiple approvals for export policy changes

## Related Properties

| Property | Purpose |
|----------|---------|
| `publicNetworkAccess` | Controls public endpoint access |
| `networkRuleSet` | IP and subnet firewall rules |
| `privateEndpointConnections` | List of private endpoint connections |
| `dataEndpointEnabled` | Enables dedicated data endpoints |

## Source References

- Primary source: `/Users/johnsonshi/repos/acr-skills/submodules/azure-management-docs/articles/container-registry/data-loss-prevention.md`
- RBAC roles: `/Users/johnsonshi/repos/acr-skills/submodules/azure-management-docs/articles/container-registry/container-registry-rbac-built-in-roles-overview.md`

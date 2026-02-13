# ACR Dedicated Data Endpoints - Configuration Guide

## Prerequisites

1. **Premium SKU Registry**: Dedicated data endpoints are only available in the Premium service tier
2. **Azure CLI**: Version 2.4.0 or later
3. **Client Firewall Access**: Ability to configure firewall rules on clients

## Enabling Dedicated Data Endpoints

### Method 1: Azure Portal

1. Navigate to your container registry in the Azure portal
2. Select **Networking** > **Public access**
3. Select the **Use dedicated data endpoint** checkbox
4. Select **Save**

After enabling, the data endpoints are displayed in the portal.

### Method 2: Azure CLI

#### Enable Dedicated Data Endpoints

```bash
az acr update --name <registry-name> --data-endpoint-enabled
```

Example:
```bash
az acr update --name contoso --data-endpoint-enabled
```

#### View Data Endpoints

```bash
az acr show-endpoints --name <registry-name>
```

Example output:
```json
{
  "loginServer": "contoso.azurecr.io",
  "dataEndpoints": [
    {
      "region": "eastus",
      "endpoint": "contoso.eastus.data.azurecr.io"
    },
    {
      "region": "westus",
      "endpoint": "contoso.westus.data.azurecr.io"
    }
  ]
}
```

## Configuration for Different Scenarios

### Scenario 1: Single Region Registry

```bash
# Enable dedicated data endpoints
az acr update --name myregistry --data-endpoint-enabled

# View endpoints
az acr show-endpoints --name myregistry
```

Output:
```json
{
  "loginServer": "myregistry.azurecr.io",
  "dataEndpoints": [
    {
      "region": "eastus",
      "endpoint": "myregistry.eastus.data.azurecr.io"
    }
  ]
}
```

### Scenario 2: Geo-Replicated Registry

When dedicated data endpoints are enabled on a geo-replicated registry, endpoints are automatically created for all replica regions.

```bash
# Enable dedicated data endpoints (applies to all replicas)
az acr update --name myregistry --data-endpoint-enabled

# View all regional endpoints
az acr show-endpoints --name myregistry
```

Output:
```json
{
  "loginServer": "myregistry.azurecr.io",
  "dataEndpoints": [
    {
      "region": "eastus",
      "endpoint": "myregistry.eastus.data.azurecr.io"
    },
    {
      "region": "westus",
      "endpoint": "myregistry.westus.data.azurecr.io"
    },
    {
      "region": "westeurope",
      "endpoint": "myregistry.westeurope.data.azurecr.io"
    }
  ]
}
```

### Scenario 3: Private Link Configuration

When configuring a private endpoint, dedicated data endpoints are enabled automatically.

```bash
# Create private endpoint (data endpoints auto-enabled)
az network private-endpoint create \
    --name myPrivateEndpoint \
    --resource-group myResourceGroup \
    --vnet-name myVNet \
    --subnet mySubnet \
    --private-connection-resource-id $REGISTRY_ID \
    --group-ids registry \
    --connection-name myConnection
```

Note: Private endpoints include IPs for both:
- Registry endpoint (`myregistry.azurecr.io`)
- Data endpoint(s) (`myregistry.<region>.data.azurecr.io`)

### Scenario 4: Connected Registry Preparation

Connected registries require dedicated data endpoints on the parent cloud registry.

```bash
# Enable dedicated data endpoint for cloud registry
az acr update -n mycontainerregistry001 --data-endpoint-enabled

# Create connected registry (validates data endpoint is enabled)
az acr connected-registry create \
  --registry mycontainerregistry001 \
  --name myconnectedregistry \
  --repository "hello-world"
```

## DNS Configuration

### For Private Link Scenarios

Create DNS records in your private DNS zone for both registry and data endpoints.

#### Create Private DNS Zone

```bash
az network private-dns zone create \
  --resource-group myResourceGroup \
  --name "privatelink.azurecr.io"
```

#### Create DNS Records

```bash
# Registry endpoint
az network private-dns record-set a create \
  --name myregistry \
  --zone-name privatelink.azurecr.io \
  --resource-group myResourceGroup

az network private-dns record-set a add-record \
  --record-set-name myregistry \
  --zone-name privatelink.azurecr.io \
  --resource-group myResourceGroup \
  --ipv4-address $REGISTRY_PRIVATE_IP

# Data endpoint (region-specific)
az network private-dns record-set a create \
  --name myregistry.eastus.data \
  --zone-name privatelink.azurecr.io \
  --resource-group myResourceGroup

az network private-dns record-set a add-record \
  --record-set-name myregistry.eastus.data \
  --zone-name privatelink.azurecr.io \
  --resource-group myResourceGroup \
  --ipv4-address $DATA_ENDPOINT_PRIVATE_IP
```

## Migration Considerations

### Before Enabling Dedicated Data Endpoints

**Important**: Switching to dedicated data endpoints will impact clients configured with firewall access to `*.blob.core.windows.net` endpoints, causing pull failures.

**Migration Steps**:

1. **Identify Current Firewall Rules**
   - Document existing `*.blob.core.windows.net` rules
   - Identify all clients accessing the registry

2. **Add New Data Endpoint Rules**
   ```bash
   # Get the data endpoints first
   az acr show-endpoints --name myregistry
   ```

   Add firewall rules for:
   - `<registry>.azurecr.io` (REST endpoint)
   - `<registry>.<region>.data.azurecr.io` (data endpoints)

3. **Enable Dedicated Data Endpoints**
   ```bash
   az acr update --name myregistry --data-endpoint-enabled
   ```

4. **Test Client Connectivity**
   ```bash
   docker pull myregistry.azurecr.io/myimage:tag
   ```

5. **Remove Old Wildcard Rules** (optional, after verification)
   - Remove `*.blob.core.windows.net` rules once all clients are migrated

## Disabling Dedicated Data Endpoints

**Note**: Generally not recommended once enabled, but can be done:

```bash
az acr update --name myregistry --data-endpoint-enabled false
```

**Caution**:
- Clients with specific data endpoint rules will fail
- You'll need to re-add `*.blob.core.windows.net` rules
- Connected registries will lose sync capability

## Verification

### Check Data Endpoint Status

```bash
az acr show --name myregistry --query "dataEndpointEnabled"
```

### List All Endpoints

```bash
az acr show-endpoints --name myregistry
```

### Test Connectivity

```bash
# Test registry endpoint
curl -v https://myregistry.azurecr.io/v2/

# Test data endpoint (should get authentication required)
curl -v https://myregistry.eastus.data.azurecr.io/
```

### Validate DNS Resolution (for Private Link)

```bash
# From within the VNet
dig myregistry.azurecr.io
dig myregistry.eastus.data.azurecr.io

# Should resolve to private IPs
```

## Configuration Summary Table

| Configuration | CLI Flag/Parameter | Portal Location |
|--------------|-------------------|-----------------|
| Enable data endpoints | `--data-endpoint-enabled` | Networking > Public access |
| View endpoints | `az acr show-endpoints` | Networking > Public access |
| Private DNS | N/A (separate commands) | Private DNS zones |

## Sources

- MS Learn: `/articles/container-registry/container-registry-dedicated-data-endpoints.md`
- MS Learn: `/articles/container-registry/container-registry-firewall-access-rules.md`
- MS Learn: `/articles/container-registry/container-registry-private-link.md`
- ACR Repo: `/docs/preview/connected-registry/quickstart-connected-registry-cli.md`
- ACR Repo: `/docs/custom-domain/README.md`

# ACR Firewall Rules - IP-Based Access Control

## Overview

Azure Container Registry firewall rules provide IP-based access control to restrict which public IP addresses or CIDR ranges can access your registry. This feature is essential for securing registries accessible from the internet while allowing access only from known, trusted sources.

## Prerequisites

- **SKU**: Premium tier required
- **Azure CLI**: Version 2.6.0 or later for `--public-network-enabled` argument
- **Maximum Rules**: 200 IP access rules per registry

## Key Concepts

### Registry Endpoints

When configuring firewall access, understand that ACR has two endpoint types:

1. **REST API Endpoint** (`<registry>.azurecr.io`)
   - Handles authentication and registry management
   - Uses wildcard SSL certificate (requires `azurecr.io` accessibility)

2. **Data Endpoint** (Storage)
   - Default: `*.blob.core.windows.net`
   - Dedicated: `<registry>.<region>.data.azurecr.io`
   - Handles image layer blob transfers

### IP Address Sources

Download IP ranges from Microsoft:
- **Download URL**: https://www.microsoft.com/download/details.aspx?id=56519
- **File**: Azure IP Ranges and Service Tags - Public Cloud (JSON)
- **Update Frequency**: Weekly

Search for:
- `AzureContainerRegistry` - All ACR REST endpoint IPs
- `AzureContainerRegistry.<region>` - Region-specific IPs
- `Storage` - All Azure Storage IPs (for blob endpoints)
- `Storage.<region>` - Region-specific storage IPs

## Configuration Methods

### Azure CLI

#### Change Default Network Access

Deny all access by default (recommended before adding rules):

```bash
az acr update --name myContainerRegistry --default-action Deny
```

Allow all access (restore default):

```bash
az acr update --name myContainerRegistry --default-action Allow
```

#### Add IP Network Rules

Add single IP address:

```bash
az acr network-rule add \
  --name mycontainerregistry \
  --ip-address 203.0.113.50
```

Add CIDR range:

```bash
az acr network-rule add \
  --name mycontainerregistry \
  --ip-address 203.0.113.0/24
```

#### List Network Rules

```bash
az acr network-rule list --name mycontainerregistry
```

#### Remove Network Rules

```bash
az acr network-rule remove \
  --name mycontainerregistry \
  --ip-address 203.0.113.50
```

#### Disable Public Network Access Entirely

```bash
az acr update --name myContainerRegistry --public-network-enabled false
```

#### Re-enable Public Network Access

```bash
az acr update --name myContainerRegistry --public-network-enabled true
```

### Azure Portal

1. Navigate to your container registry
2. Under **Settings**, select **Networking**
3. Select the **Public access** tab
4. Choose access level:
   - **All networks** - Allow from anywhere
   - **Selected networks** - Allow from specified IPs only
   - **Disabled** - Block all public access

5. Under **Firewall**, enter IP addresses or CIDR ranges
6. Click **Save**

**Note**: Changes take a few minutes to propagate.

## Dedicated Data Endpoints

### Why Use Dedicated Data Endpoints?

- Eliminates need for wildcard `*.blob.core.windows.net` firewall rules
- Mitigates data exfiltration risks
- Provides tightly scoped firewall rules
- Format: `<registry>.<region>.data.azurecr.io`

### Enable Dedicated Data Endpoints

**Azure CLI**:

```bash
az acr update --name myregistry --data-endpoint-enabled
```

**View Data Endpoints**:

```bash
az acr show-endpoints --name myregistry
```

**Sample Output**:

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
        }
    ]
}
```

### Azure Portal

1. Navigate to your container registry
2. Select **Networking** > **Public access**
3. Check **Use dedicated data endpoint**
4. Click **Save**

## Client Firewall Configuration

When clients need to access ACR from behind a firewall:

### Required Endpoints

| Endpoint Type | Pattern | Purpose |
|--------------|---------|---------|
| REST Endpoint | `<registry>.azurecr.io` | Authentication, API |
| SSL Certificate | `azurecr.io` | TLS handshake |
| Data (default) | `*.blob.core.windows.net` | Image layers |
| Data (dedicated) | `<registry>.<region>.data.azurecr.io` | Image layers |

### Geo-Replicated Registries

For geo-replicated registries, configure access to data endpoints in all replica regions:

```
myregistry.eastus.data.azurecr.io
myregistry.westus.data.azurecr.io
myregistry.westeurope.data.azurecr.io
```

## Common Scenarios

### Azure Pipelines

Azure Pipelines using Microsoft-hosted agents have changing IP addresses.

**Solutions**:
1. Use self-hosted agents with fixed IP addresses
2. Add the self-hosted agent's IP to registry firewall rules

### Azure Kubernetes Service (AKS)

AKS egress IPs are dynamically assigned by default.

**Solutions**:
1. **Basic Load Balancer**: Configure static egress IP
2. **Standard Load Balancer**: Control egress traffic configuration
3. Use Private Link for private connectivity

### HTTPS Proxy

When accessing behind an HTTPS proxy:

1. Configure Docker client for proxy:
   ```bash
   export HTTP_PROXY=http://proxy.example.com:8080
   export HTTPS_PROXY=http://proxy.example.com:8080
   ```

2. Configure Docker daemon for proxy (systemd):
   ```ini
   [Service]
   Environment="HTTP_PROXY=http://proxy.example.com:8080"
   Environment="HTTPS_PROXY=http://proxy.example.com:8080"
   ```

3. Add proxy IP to registry firewall rules

## Troubleshooting

### Common Errors

| Error | Cause | Solution |
|-------|-------|----------|
| `403 Forbidden` | IP not in allowlist | Add client IP to firewall rules |
| `login attempt failed with status: 403` | Public access disabled or IP blocked | Verify IP rules or enable public access |
| `Looks like you don't have access to registry` | Network restrictions | Check firewall configuration |

### Diagnostic Commands

Check registry health:

```bash
az acr check-health --name myregistry
```

View registry network settings:

```bash
az acr show --name myregistry --query networkRuleSet
```

### Finding Your Public IP

Search "what is my IP address" in browser, or use:

```bash
curl -s ifconfig.me
```

The Azure portal also shows your current IP when configuring firewall settings.

## Security Considerations

### Best Practices

1. **Start with Deny**: Set default action to Deny before adding rules
2. **Minimal Access**: Only add IPs that truly need access
3. **Use CIDR Ranges**: For large deployments, use CIDR notation
4. **Enable Dedicated Endpoints**: Reduce data exfiltration risk
5. **Regular Audits**: Review IP rules periodically
6. **Monitor Access**: Enable diagnostic logging

### Combining with Other Features

- **Private Link**: IP rules don't apply to private endpoints
- **Service Endpoints**: Can't use both Private Link and Service Endpoints
- **Trusted Services**: Works alongside firewall rules

## Interaction with Other Features

| Feature | Behavior with Firewall Rules |
|---------|------------------------------|
| Private Link | IP rules don't apply to private endpoints |
| Service Endpoints | Disabling public access disables service endpoint access |
| Trusted Services | Can bypass firewall rules |
| Geo-replication | Configure data endpoints for all regions |

## Source Files

- `/submodules/azure-management-docs/articles/container-registry/container-registry-access-selected-networks.md`
- `/submodules/azure-management-docs/articles/container-registry/container-registry-firewall-access-rules.md`
- `/submodules/azure-management-docs/articles/container-registry/container-registry-dedicated-data-endpoints.md`
- `/submodules/azure-management-docs/articles/container-registry/container-registry-troubleshoot-access.md`

# ACR Dedicated Data Endpoints - Firewall Rules Configuration

## Overview

This document covers firewall rule configuration for accessing Azure Container Registry with dedicated data endpoints enabled. Proper firewall configuration is essential for clients behind firewalls, proxy servers, or in on-premises environments.

## Understanding Endpoints for Firewall Rules

### Required Endpoints

To pull or push images, clients need access to two types of endpoints over **HTTPS (port 443)**:

| Endpoint Type | Purpose | Format | Example |
|--------------|---------|--------|---------|
| Registry REST API | Authentication, manifest retrieval | `<registry>.azurecr.io` | `contoso.azurecr.io` |
| SSL Certificate | TLS handshake | `azurecr.io` | `azurecr.io` |
| Data Endpoint | Layer download/upload | `<registry>.<region>.data.azurecr.io` | `contoso.eastus.data.azurecr.io` |

### Without Dedicated Data Endpoints

```
Required firewall rules:
- <registry>.azurecr.io:443
- *.blob.core.windows.net:443  <-- SECURITY RISK: Wildcard
```

### With Dedicated Data Endpoints

```
Required firewall rules:
- <registry>.azurecr.io:443
- <registry>.<region>.data.azurecr.io:443  <-- Specific to your registry
```

## Client Firewall Configuration

### Single Region Registry

For a registry named `myregistry` in East US:

```
Allow outbound HTTPS (port 443):
- myregistry.azurecr.io
- myregistry.eastus.data.azurecr.io
- azurecr.io (for SSL certificates)
```

### Geo-Replicated Registry

For a geo-replicated registry with replicas in multiple regions:

```
Allow outbound HTTPS (port 443):
- myregistry.azurecr.io
- myregistry.eastus.data.azurecr.io
- myregistry.westus.data.azurecr.io
- myregistry.westeurope.data.azurecr.io
- azurecr.io (for SSL certificates)
```

### Getting Your Data Endpoints

```bash
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
    }
  ]
}
```

## Firewall Rules by Platform

### Azure Firewall

```bash
# Create application rule collection
az network firewall application-rule create \
    --collection-name ACR-Rules \
    --firewall-name myFirewall \
    --name AllowACR \
    --resource-group myResourceGroup \
    --protocols https=443 \
    --source-addresses "*" \
    --target-fqdns \
        "myregistry.azurecr.io" \
        "myregistry.eastus.data.azurecr.io" \
        "azurecr.io"
```

### Network Security Group (NSG)

For NSG rules, use the **AzureContainerRegistry** service tag:

```bash
# Outbound rule for ACR
az network nsg rule create \
    --resource-group myResourceGroup \
    --nsg-name myNSG \
    --name AllowACR \
    --priority 100 \
    --direction Outbound \
    --access Allow \
    --protocol Tcp \
    --destination-port-ranges 443 \
    --destination-address-prefixes AzureContainerRegistry
```

Regional service tags:
```
AzureContainerRegistry.EastUS
AzureContainerRegistry.WestUS
AzureContainerRegistry.WestEurope
```

### Windows Firewall

```powershell
# Allow registry endpoint
New-NetFirewallRule -DisplayName "Allow ACR Registry" `
    -Direction Outbound `
    -Action Allow `
    -Protocol TCP `
    -RemotePort 443 `
    -RemoteAddress "myregistry.azurecr.io"

# Allow data endpoint
New-NetFirewallRule -DisplayName "Allow ACR Data" `
    -Direction Outbound `
    -Action Allow `
    -Protocol TCP `
    -RemotePort 443 `
    -RemoteAddress "myregistry.eastus.data.azurecr.io"
```

### Linux iptables

```bash
# Allow registry endpoint
iptables -A OUTPUT -p tcp --dport 443 -d myregistry.azurecr.io -j ACCEPT

# Allow data endpoint
iptables -A OUTPUT -p tcp --dport 443 -d myregistry.eastus.data.azurecr.io -j ACCEPT
```

## IP Address-Based Rules

If your firewall requires IP addresses instead of FQDNs:

### Download IP Ranges

Download the Azure IP ranges JSON file:
- URL: https://www.microsoft.com/download/details.aspx?id=56519

### Find ACR IP Ranges

Search for `AzureContainerRegistry` in the JSON:

```json
{
  "name": "AzureContainerRegistry",
  "id": "AzureContainerRegistry",
  "properties": {
    "changeNumber": 10,
    "region": "",
    "platform": "Azure",
    "systemService": "AzureContainerRegistry",
    "addressPrefixes": [
      "13.66.140.72/29",
      "13.67.9.224/29",
      ...
    ]
  }
}
```

### Region-Specific IP Ranges

```json
{
  "name": "AzureContainerRegistry.EastUS",
  "id": "AzureContainerRegistry.EastUS",
  "properties": {
    "changeNumber": 1,
    "region": "eastus",
    "platform": "Azure",
    "systemService": "AzureContainerRegistry",
    "addressPrefixes": [
      "13.70.72.136/29",
      ...
    ]
  }
}
```

**Important**: IP ranges change weekly. Download updates regularly and automate rule updates.

## Proxy Server Configuration

### HTTPS Proxy Setup

For clients behind an HTTPS proxy:

```bash
# Docker daemon configuration
export HTTPS_PROXY=http://proxy.example.com:8080
export NO_PROXY=localhost,127.0.0.1

# Or in /etc/docker/daemon.json
{
  "proxies": {
    "https-proxy": "http://proxy.example.com:8080",
    "no-proxy": "localhost,127.0.0.1"
  }
}
```

### Proxy Allow List

Configure your proxy to allow:
```
myregistry.azurecr.io:443
myregistry.eastus.data.azurecr.io:443
azurecr.io:443
```

## AKS HTTP Proxy Configuration

For AKS clusters behind a proxy:

```yaml
# In AKS HTTP proxy configuration
noProxy:
  - myregistry.azurecr.io
  - myregistry.eastus.data.azurecr.io
```

Or add endpoints to the proxy's allowed list for central traffic control.

## IoT Edge Device Configuration

For IoT Edge devices pulling from ACR:

```json
// config.toml or deployment manifest
{
  "registryCredentials": {
    "myregistry": {
      "username": "<token-name>",
      "password": "<token-password>",
      "address": "myregistry.azurecr.io"
    }
  }
}
```

Ensure firewall allows:
- `myregistry.azurecr.io:443`
- `myregistry.<region>.data.azurecr.io:443`

## Troubleshooting Firewall Issues

### Common Error Messages

| Error | Likely Cause | Solution |
|-------|--------------|----------|
| `dial tcp: lookup myregistry.azurecr.io` | DNS resolution blocked | Allow DNS or add hosts entry |
| `Client.Timeout exceeded` | Firewall blocking traffic | Check firewall rules |
| `Could not connect to registry login server` | Registry endpoint blocked | Allow `<registry>.azurecr.io:443` |
| Pull timeout on large images | Data endpoint blocked | Allow `<registry>.<region>.data.azurecr.io:443` |

### Diagnostic Commands

```bash
# Test registry endpoint
curl -v https://myregistry.azurecr.io/v2/

# Test data endpoint
curl -v https://myregistry.eastus.data.azurecr.io/

# DNS resolution test
nslookup myregistry.azurecr.io
nslookup myregistry.eastus.data.azurecr.io

# ACR health check
az acr check-health --name myregistry --yes
```

### Check Resource Logs

Enable diagnostic logs to identify blocked connections:

```bash
az monitor diagnostic-settings create \
    --name ACRDiagnostics \
    --resource $(az acr show --name myregistry --query id -o tsv) \
    --logs '[{"category": "ContainerRegistryLoginEvents", "enabled": true}]'
```

## Migration from Wildcard Rules

### Step-by-Step Migration

1. **Document Current State**
   ```bash
   # List current endpoints
   az acr show-endpoints --name myregistry
   ```

2. **Add Specific Rules** (before enabling dedicated endpoints)
   ```
   Add rules for:
   - myregistry.azurecr.io:443
   - myregistry.eastus.data.azurecr.io:443
   ```

3. **Enable Dedicated Data Endpoints**
   ```bash
   az acr update --name myregistry --data-endpoint-enabled
   ```

4. **Test Connectivity**
   ```bash
   docker pull myregistry.azurecr.io/testimage:latest
   ```

5. **Remove Wildcard Rules** (after verification)
   ```
   Remove:
   - *.blob.core.windows.net:443
   ```

## Data Exfiltration Prevention

### Why Remove Wildcard Rules?

With `*.blob.core.windows.net` allowed:
- Any Azure storage account is accessible
- Malicious code could write to unauthorized storage
- Data exfiltration is possible

With dedicated data endpoints:
- Only your registry's storage is accessible
- Bad actors cannot write to other storage accounts
- Data exfiltration risk is minimized

### Security Best Practice

```
REMOVE: *.blob.core.windows.net:443
ADD:    myregistry.eastus.data.azurecr.io:443
```

## Quick Reference: Required FQDNs

| Scenario | FQDNs to Allow |
|----------|---------------|
| Basic access | `<registry>.azurecr.io`, `<registry>.<region>.data.azurecr.io`, `azurecr.io` |
| Geo-replicated | Add `<registry>.<each-region>.data.azurecr.io` |
| Private Link | Same, but resolved to private IPs |
| Connected Registry | `<registry>.azurecr.io`, `<registry>.<region>.data.azurecr.io` (parent registry) |

## Sources

- MS Learn: `/articles/container-registry/container-registry-firewall-access-rules.md`
- MS Learn: `/articles/container-registry/container-registry-dedicated-data-endpoints.md`
- MS Learn: `/articles/container-registry/container-registry-troubleshoot-access.md`
- ACR Repo: `/docs/preview/connected-registry/overview-connected-registry-access.md`

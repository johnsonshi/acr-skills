# ACR Health Check Reference

## Overview

The `az acr check-health` command provides quick diagnostics for Azure Container Registry environments. It helps identify common problems with Docker configuration, network connectivity, and registry access. This command is available in Azure CLI version 2.0.67 and later.

## Command Syntax

```bash
az acr check-health [--name <registry-name>] [--vnet <vnet-name-or-id>] [--ignore-errors] [--yes]
```

### Parameters

| Parameter | Description |
|-----------|-------------|
| `--name`, `-n` | Name of the target registry to check |
| `--vnet` | Virtual network name or resource ID for DNS routing verification |
| `--ignore-errors` | Continue checking all items even if errors are found |
| `--yes`, `-y` | Skip confirmation prompts (e.g., for image pull) |

## Usage Scenarios

### Check Environment Only

Check local Docker daemon, CLI version, and Helm client:

```bash
az acr check-health
```

**Checks performed:**
- Docker daemon status
- Docker version
- Docker pull capability
- ACR CLI version
- Helm version (if installed)

### Check Environment and Registry Access

Verify both local environment and registry connectivity:

```bash
az acr check-health --name myregistry
```

**Additional checks:**
- DNS lookup to registry
- Challenge endpoint accessibility
- Refresh token retrieval
- Access token retrieval

### Check Access in Virtual Network

Verify DNS routing to private endpoint:

```bash
az acr check-health --name myregistry --vnet myvnet
```

**Note:** Use resource ID if VNet is in different subscription or resource group:
```bash
az acr check-health --name myregistry \
  --vnet /subscriptions/{sub}/resourceGroups/{rg}/providers/Microsoft.Network/virtualNetworks/{vnet}
```

### Check All Items (Ignore Errors)

Continue checking all items even when errors occur:

```bash
az acr check-health --ignore-errors
az acr check-health --name myregistry --ignore-errors --yes
```

## Sample Output

### Successful Check

```
Docker daemon status: available
Docker version: Docker version 18.09.2, build 6247962
Docker pull of 'mcr.microsoft.com/mcr/hello-world:latest' : OK
ACR CLI version: 2.2.9
Helm version:
Client: &version.Version{SemVer:"v2.14.1", GitCommit:"5270352a09c7e8b6e8c9593002a73535276507c0", GitTreeState:"clean"}
DNS lookup to myregistry.azurecr.io at IP 40.xxx.xxx.162 : OK
Challenge endpoint https://myregistry.azurecr.io/v2/ : OK
Fetch refresh token for registry 'myregistry.azurecr.io' : OK
Fetch access token for registry 'myregistry.azurecr.io' : OK
```

## Error Codes Reference

### Docker Errors

| Error Code | Description | Solution |
|------------|-------------|----------|
| DOCKER_COMMAND_ERROR | Docker client not found | Install Docker, add to PATH |
| DOCKER_DAEMON_ERROR | Docker daemon unavailable | Restart Docker daemon |
| DOCKER_VERSION_ERROR | Cannot get Docker version | Update Docker/CLI |
| DOCKER_PULL_ERROR | Cannot pull sample image | Check permissions, network |

### Helm Errors

| Error Code | Description | Solution |
|------------|-------------|----------|
| HELM_COMMAND_ERROR | Helm client not found | Install Helm, add to PATH |
| HELM_VERSION_ERROR | Cannot determine Helm version | Update Azure CLI or Helm |

### Connectivity Errors

| Error Code | Description | Solution |
|------------|-------------|----------|
| CONNECTIVITY_DNS_ERROR | DNS lookup failed | Check connectivity, verify registry exists |
| CONNECTIVITY_FORBIDDEN_ERROR | 403 Forbidden response | Check firewall rules, VNet config |
| CONNECTIVITY_CHALLENGE_ERROR | Challenge endpoint failed | Retry or open support ticket |
| CONNECTIVITY_AAD_LOGIN_ERROR | AAD authentication not supported | Use different auth method |
| CONNECTIVITY_REFRESH_TOKEN_ERROR | Refresh token denied | Check permissions, re-login |
| CONNECTIVITY_ACCESS_TOKEN_ERROR | Access token denied | Check permissions, re-login |
| CONNECTIVITY_SSL_ERROR | SSL connection failed | Check proxy configuration |
| CONNECTIVITY_TOOMANYREQUESTS_ERROR | Rate limited (429) | Wait and retry |

### Registry Errors

| Error Code | Description | Solution |
|------------|-------------|----------|
| LOGIN_SERVER_ERROR | Cannot find login server | Verify registry name and permissions |
| CMK_ERROR | Cannot access managed identity for CMK | Check identity configuration |

### Notary Errors

| Error Code | Description | Solution |
|------------|-------------|----------|
| NOTARY_VERSION_ERROR | Notary version incompatible | Downgrade Notary to < 0.6.0 |
| NOTARY_COMMAND_ERROR | Notary client not found | Install Notary |

## Error Solutions in Detail

### DOCKER_COMMAND_ERROR
```
Potential solutions:
- Install Docker Desktop
- Add Docker to system PATH
- Verify Docker installation: `docker --version`
```

### DOCKER_DAEMON_ERROR
```
Potential solutions:
- Start Docker Desktop
- Linux: `sudo systemctl start docker`
- Check Docker status: `docker info`
```

### CONNECTIVITY_DNS_ERROR
```
Potential solutions:
- Verify internet connectivity
- Check registry name spelling
- Verify registry exists: `az acr show --name <registry>`
- Check if using correct Azure cloud
```

### CONNECTIVITY_FORBIDDEN_ERROR
```
Potential solutions:
- Check firewall rules: `az acr show --query networkRuleSet --name <registry>`
- Add client IP to allowed list
- Remove VNet restrictions for testing
```

### CONNECTIVITY_REFRESH_TOKEN_ERROR
```
Potential solutions:
- Verify permissions: `az role assignment list`
- Re-authenticate: `az login`
- Check repository-level permissions (ABAC)
```

## Azure Cloud Shell Note

When running in Azure Cloud Shell:
- Local environment checks are skipped (Docker not available)
- Registry access checks still work
- Use `--name` parameter to check specific registry

## Quarantine Check

If registry has quarantine enabled, pull attempts will fail:

```
Any attempts to access or pull quarantined images will fail with an error.
```

To access quarantined images, use the quarantine-enabled pull workflow.

## Related Commands

### AKS Integration Check

For AKS clusters, use dedicated command:
```bash
az aks check-acr --name <aks-cluster> --resource-group <rg> --acr <registry>
```

### Network Diagnostics

For network-specific issues:
```bash
# Check NSG rules
az network nsg rule list --nsg-name <nsg> --resource-group <rg>

# DNS lookup
nslookup myregistry.azurecr.io

# Test connectivity
curl -v https://myregistry.azurecr.io/v2/
```

## Best Practices

1. **Run health check first** when troubleshooting any ACR issue
2. **Use --ignore-errors** to get complete diagnostic picture
3. **Check both environment and registry** with `--name` parameter
4. **Verify VNet routing** with `--vnet` for private endpoint setups
5. **Review error reference** for detailed solutions
6. **Update Azure CLI** regularly for latest health checks

## Troubleshooting Workflow

```
1. Run: az acr check-health --name <registry> --ignore-errors

2. If Docker errors:
   - Verify Docker installation
   - Restart Docker daemon

3. If connectivity errors:
   - Check network/firewall settings
   - Verify registry exists
   - Check permissions

4. If authentication errors:
   - Re-authenticate with az login
   - Verify role assignments
   - Check credential expiration

5. If still failing:
   - Check resource logs in Log Analytics
   - Open support ticket with error codes
```

## Source References

- `/submodules/azure-management-docs/articles/container-registry/container-registry-check-health.md`
- `/submodules/azure-management-docs/articles/container-registry/container-registry-health-error-reference.md`
- `/submodules/azure-management-docs/articles/container-registry/container-registry-troubleshoot-login-authn-authz.md`
- `/submodules/azure-management-docs/articles/container-registry/container-registry-troubleshoot-access.md`
- `/submodules/azure-management-docs/articles/container-registry/container-registry-troubleshoot-performance.md`

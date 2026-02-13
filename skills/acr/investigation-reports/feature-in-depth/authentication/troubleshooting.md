# Azure Container Registry Authentication - Troubleshooting Guide

## Diagnostic Tools

### az acr check-health

The primary diagnostic command for ACR authentication issues.

```bash
# Check local environment
az acr check-health

# Check environment and registry access
az acr check-health --name <registry>

# Check with virtual network
az acr check-health --name <registry> --vnet <vnet-name-or-id>

# Continue despite errors
az acr check-health --name <registry> --ignore-errors --yes
```

**Sample Output**:
```
Docker daemon status: available
Docker version: Docker version 20.10.17, build 100c701
Docker pull of 'mcr.microsoft.com/mcr/hello-world:latest' : OK
ACR CLI version: 2.45.0
DNS lookup to myregistry.azurecr.io at IP 40.xxx.xxx.162 : OK
Challenge endpoint https://myregistry.azurecr.io/v2/ : OK
Fetch refresh token for registry 'myregistry.azurecr.io' : OK
Fetch access token for registry 'myregistry.azurecr.io' : OK
```

---

## Common Authentication Errors

### DOCKER_COMMAND_ERROR

**Symptom**: Docker client not found

**Causes**:
- Docker not installed
- Docker not in system PATH
- Docker daemon not running

**Solutions**:
1. Install Docker Desktop or Docker Engine
2. Add Docker to system PATH
3. Verify Docker is running: `docker version`

---

### DOCKER_DAEMON_ERROR

**Symptom**: Docker daemon unavailable or unreachable

**Causes**:
- Docker daemon not started
- Docker service crashed
- Permissions issue

**Solutions**:
1. Start Docker daemon:
   - Linux: `sudo systemctl start docker`
   - macOS/Windows: Start Docker Desktop
2. Check daemon status: `docker info`
3. Verify user permissions (add to docker group on Linux)

---

### CONNECTIVITY_DNS_ERROR

**Symptom**: Cannot resolve registry login server

**Causes**:
- Registry doesn't exist
- Network connectivity issues
- DNS configuration problems
- Wrong cloud environment in CLI

**Solutions**:
1. Verify registry exists: `az acr show --name <registry>`
2. Check DNS resolution: `nslookup <registry>.azurecr.io`
3. Verify Azure CLI cloud: `az cloud show`
4. Check network connectivity: `curl https://<registry>.azurecr.io/v2/`

---

### CONNECTIVITY_FORBIDDEN_ERROR (403)

**Symptom**: Challenge endpoint returns 403 Forbidden

**Causes**:
- Virtual network rules blocking access
- IP firewall rules blocking access
- Public network access disabled

**Solutions**:
1. Check firewall rules:
   ```bash
   az acr show --name <registry> --query networkRuleSet
   ```
2. Add client IP to allowed list:
   ```bash
   az acr network-rule add --name <registry> --ip-address <your-ip>
   ```
3. If using private endpoint, ensure proper DNS resolution
4. Check if public access is disabled:
   ```bash
   az acr show --name <registry> --query publicNetworkAccess
   ```

---

### CONNECTIVITY_REFRESH_TOKEN_ERROR

**Symptom**: Registry denied access, cannot obtain refresh token

**Error Message**: `CONNECTIVITY_REFRESH_TOKEN_ERROR. Access to registry was denied.`

**Causes**:
- User lacks permissions on registry
- Stale Azure CLI credentials
- ABAC-enabled registry without repository permissions
- Token/credential expired

**Solutions**:
1. Re-authenticate to Azure:
   ```bash
   az logout
   az login
   ```
2. Verify role assignment:
   ```bash
   az role assignment list --scope <registry-resource-id>
   ```
3. For ABAC-enabled registries, verify repository-level permissions
4. Check if using correct role for registry type:
   - ABAC: `Container Registry Repository Reader/Writer`
   - Non-ABAC: `AcrPull/AcrPush`

---

### CONNECTIVITY_ACCESS_TOKEN_ERROR

**Symptom**: Cannot obtain access token for registry operations

**Causes**:
- Insufficient permissions for requested operation
- Scope map doesn't include required repository
- Token credentials expired

**Solutions**:
1. Refresh credentials: `az login`
2. Check scope map permissions for repository tokens
3. Verify role assignment includes required data actions
4. For ABAC registries, check ABAC conditions on role assignment

---

### CONNECTIVITY_AAD_LOGIN_ERROR

**Symptom**: Registry doesn't support Microsoft Entra authentication

**Solutions**:
1. Use alternative authentication method (admin credentials, token)
2. Verify registry supports AAD authentication
3. Check registry SKU (Basic tier has full AAD support)

---

### CONNECTIVITY_SSL_ERROR

**Symptom**: Cannot establish secure connection

**Causes**:
- Proxy server issues
- SSL/TLS certificate problems
- Network inspection/interception

**Solutions**:
1. Configure proxy for Docker daemon
2. Check SSL certificate chain
3. Verify corporate firewall isn't intercepting traffic

---

### CONNECTIVITY_TOOMANYREQUESTS_ERROR

**Symptom**: Too many authentication requests

**Causes**:
- Hit rate limit on authentication system
- High-frequency automated requests

**Solutions**:
1. Wait before retrying (typically 1-5 minutes)
2. Implement retry logic with exponential backoff
3. Consider caching tokens in automated workflows

---

## Login-Specific Issues

### "unauthorized: authentication required"

**Causes**:
- Not logged in to registry
- Expired credentials
- Cached credentials from previous session

**Solutions**:
```bash
# Re-authenticate
az acr login --name <registry>

# Or with Docker directly
docker logout <registry>.azurecr.io
docker login <registry>.azurecr.io
```

---

### "unauthorized: Application not registered with AAD"

**Causes**:
- Service principal not properly configured
- Multitenant app not provisioned in target tenant

**Solutions**:
1. Verify service principal exists in correct tenant
2. For cross-tenant scenarios, ensure app is provisioned in target tenant
3. Check tenant ID matches expected value

---

### "pull access denied" with Anonymous Pull Enabled

**Symptom**: Cannot pull even though anonymous pull is enabled

**Cause**: Cached credentials from previous authenticated session

**Solution**:
```bash
# Clear Docker credentials
docker logout <registry>.azurecr.io

# Then attempt anonymous pull
docker pull <registry>.azurecr.io/myimage:latest
```

---

### "Could not connect to the registry login server"

**Causes**:
- Network connectivity issues
- DNS resolution failure
- Firewall blocking outbound traffic

**Solutions**:
1. Check network connectivity
2. Verify firewall allows traffic to `*.azurecr.io`
3. For private endpoints, verify DNS configuration
4. Check if registry exists and name is correct

---

## Permission Issues

### RBAC Role Not Working

**Symptom**: Role assigned but operations fail

**Common Issues**:

1. **ABAC vs Non-ABAC mismatch**:
   - ABAC registries: Use `Container Registry Repository Reader/Writer/Contributor`
   - Non-ABAC registries: Use `AcrPull/AcrPush/AcrDelete`

2. **Missing ABAC conditions**:
   - Verify ABAC condition includes target repository
   - Check condition syntax for wildcards

3. **Propagation delay**:
   - Role assignments can take several minutes to propagate
   - Wait and retry

**Diagnostic**:
```bash
# Check registry mode
az acr show --name <registry> --query roleAssignmentMode

# List role assignments
az role assignment list --scope <registry-resource-id> --assignee <principal-id>
```

---

### Service Principal Cannot Push/Pull

**Causes**:
- Missing role assignment
- Expired client secret
- Wrong credentials used

**Solutions**:
1. Verify role assignment:
   ```bash
   az role assignment list --assignee <sp-app-id> --all
   ```
2. Reset credentials:
   ```bash
   az ad sp credential reset --name <sp-name>
   ```
3. Verify credentials in use match service principal

---

### Managed Identity Cannot Access Registry

**Causes**:
- Role not assigned to identity
- Using wrong identity (system vs user-assigned)
- Identity not enabled on resource

**Solutions**:
1. Verify identity is enabled:
   ```bash
   # For VMs
   az vm show --name <vm> --resource-group <rg> --query identity

   # For App Service
   az webapp identity show --name <app> --resource-group <rg>
   ```
2. Assign role to identity:
   ```bash
   az role assignment create --assignee <principal-id> --scope <acr-id> --role AcrPull
   ```
3. For user-assigned identities, verify correct identity is used for authentication

---

## Token-Based Authentication Issues

### Token Password Not Working

**Causes**:
- Token disabled
- Password expired
- Scope map doesn't include repository

**Solutions**:
1. Check token status:
   ```bash
   az acr token show --name <token> --registry <registry>
   ```
2. Enable token if disabled:
   ```bash
   az acr token update --name <token> --registry <registry> --status enabled
   ```
3. Regenerate password:
   ```bash
   az acr token credential generate --name <token> --registry <registry> --password1
   ```
4. Check scope map includes target repository

---

### Scope Map Wildcard Not Working

**Causes**:
- Invalid wildcard pattern
- Missing trailing slash

**Valid patterns**:
- `samples/*` - matches `samples/hello`, `samples/world`
- `*` - matches all repositories

**Invalid patterns**:
- `sample/*/teamA` - wildcard in middle
- `sample/teamA*` - missing trailing `/*`
- `sample/*/project/*` - multiple wildcards

---

## Kubernetes/AKS Issues

### AKS Cannot Pull Images

**Diagnostic**:
```bash
# Validate AKS can reach ACR
az aks check-acr --name <aks-cluster> --resource-group <rg> --acr <registry>
```

**Common Causes**:

1. **Managed identity not attached**:
   ```bash
   az aks update --name <aks> --resource-group <rg> --attach-acr <registry>
   ```

2. **Cross-tenant without proper configuration**:
   - Cannot use managed identity for cross-tenant
   - Must use service principal

3. **Network restrictions blocking AKS**:
   - Add AKS outbound IPs to ACR firewall
   - Or use private endpoint

---

### Image Pull Secret Not Working

**Causes**:
- Secret in wrong namespace
- Secret credentials expired
- Service principal password changed

**Solutions**:
1. Verify secret exists in correct namespace:
   ```bash
   kubectl get secret <secret-name> -n <namespace>
   ```
2. Recreate secret with new credentials:
   ```bash
   kubectl delete secret <secret-name> -n <namespace>
   kubectl create secret docker-registry <secret-name> -n <namespace> \
       --docker-server=<registry>.azurecr.io \
       --docker-username=<sp-id> \
       --docker-password=<new-password>
   ```
3. Verify Pod references the secret correctly

---

## Network-Related Issues

### Private Endpoint DNS Issues

**Symptom**: Cannot resolve private endpoint address

**Diagnostic**:
```bash
# Check DNS resolution
nslookup <registry>.azurecr.io
dig <registry>.azurecr.io

# Expected: Should resolve to private IP (10.x.x.x or similar)
```

**Solutions**:
1. Verify Private DNS zone exists and is linked to VNet
2. Check DNS zone has correct A records:
   - `<registry>.azurecr.io` -> private IP
   - `<registry>.<region>.data.azurecr.io` -> private IP (for data endpoint)
3. Use `az acr check-health --vnet <vnet>` to diagnose

---

### VNet Service Endpoint Issues

**Symptom**: Cannot access from within VNet

**Solutions**:
1. Verify service endpoint enabled on subnet:
   ```bash
   az network vnet subnet show --vnet-name <vnet> --name <subnet> --resource-group <rg> --query serviceEndpoints
   ```
2. Add network rule for subnet:
   ```bash
   az acr network-rule add --name <registry> --subnet <subnet-resource-id>
   ```

---

## Advanced Troubleshooting

### Enable Resource Logging

Configure diagnostic logs to capture authentication events:

```bash
# Create log analytics workspace (if needed)
az monitor log-analytics workspace create --name <workspace> --resource-group <rg>

# Enable diagnostic settings
az monitor diagnostic-settings create \
    --name <setting-name> \
    --resource <registry-resource-id> \
    --workspace <workspace-id> \
    --logs '[{"category":"ContainerRegistryLoginEvents","enabled":true}]'
```

**Query Authentication Failures**:
```kusto
ContainerRegistryLoginEvents
| where ResultDescription != "OK"
| project TimeGenerated, Identity, CallerIpAddress, OperationName, ResultDescription
| order by TimeGenerated desc
```

---

### Common Error Codes from Logs

| Code | Description | Resolution |
|------|-------------|------------|
| `UNAUTHORIZED` | Authentication required | Login to registry |
| `DENIED` | Permission denied | Check role assignments |
| `NAME_UNKNOWN` | Repository not found | Verify repository name |
| `MANIFEST_UNKNOWN` | Image tag not found | Verify image tag |
| `TOOMANYREQUESTS` | Rate limited | Wait and retry |

---

## Checklist for Troubleshooting

1. **Verify Docker is running**: `docker version`
2. **Check Azure CLI authentication**: `az account show`
3. **Verify registry exists**: `az acr show --name <registry>`
4. **Run health check**: `az acr check-health --name <registry>`
5. **Check role assignments**: `az role assignment list --scope <registry-id>`
6. **Verify network access**: `curl https://<registry>.azurecr.io/v2/`
7. **Clear cached credentials**: `docker logout <registry>.azurecr.io`
8. **Re-authenticate**: `az acr login --name <registry>`

---

## Sources

- `/submodules/azure-management-docs/articles/container-registry/container-registry-troubleshoot-login-authn-authz.md`
- `/submodules/azure-management-docs/articles/container-registry/container-registry-troubleshoot-access.md`
- `/submodules/azure-management-docs/articles/container-registry/container-registry-health-error-reference.md`
- `/submodules/azure-management-docs/articles/container-registry/container-registry-check-health.md`
- `/submodules/acr/docs/Troubleshooting Guide.md`

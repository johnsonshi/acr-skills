# ACR Trusted Services - Allowing Azure Services Access

## Overview

Azure Container Registry's "Allow trusted services" feature enables select Azure services to securely bypass network restrictions on a container registry. This allows trusted service instances to perform operations like pulling or pushing images even when the registry has IP firewall rules or private endpoints configured.

## How It Works

When you deploy a registry with network restrictions (Private Link or firewall rules), it denies access to users or services from outside those sources. However, several multitenant Azure services operate from networks that cannot be included in registry network settings.

By enabling the "allow trusted services" setting, the registry permits specific Azure service instances to bypass network rules and access the registry.

## Trusted Services List

The following Azure services can access a network-restricted registry when trusted services are enabled:

| Trusted Service | Usage Scenario | Requires Managed Identity with RBAC |
|-----------------|----------------|-------------------------------------|
| **Azure Container Instances** | Deploy to ACI from ACR using managed identity | Yes (system or user-assigned) |
| **Microsoft Defender for Cloud** | Vulnerability scanning of container images | No |
| **Azure Machine Learning** | Deploy or train models with custom Docker images | Yes |
| **Azure Container Registry** | Import images to/from network-restricted registries | No |

> **Note**: App Service is NOT currently supported as a trusted service.

## Limitations

1. **Service Endpoints Not Supported**: Trusted services bypass does NOT work with registries using VNet service endpoints. Only works with:
   - Private Link (private endpoints)
   - Public IP access rules (firewall)

2. **Managed Identity Required**: Some services require managed identity configuration:
   - System-assigned managed identity (SAMI) is required for most scenarios
   - User-assigned managed identity supported only where noted

## Default Setting

- **New Registries**: Enabled by default
- **Existing Registries**: May need to be enabled explicitly

## Configuration

### Azure CLI

#### Check Current Setting

```bash
az acr show --name myregistry --query networkRuleBypassOptions
```

#### Enable Trusted Services

```bash
az acr update --name myregistry --allow-trusted-services true
```

#### Disable Trusted Services

```bash
az acr update --name myregistry --allow-trusted-services false
```

### Azure Portal

1. Navigate to your container registry
2. Under **Settings**, select **Networking**
3. In **Allow public network access**, select **Selected networks** or **Disabled**
4. Under **Firewall exception**:
   - **Check**: "Allow trusted Microsoft services to access this container registry" (to enable)
   - **Uncheck**: To disable
5. Click **Save**

## Trusted Services Workflow

When using a trusted service to access a network-restricted registry:

### Step 1: Enable Managed Identity

Enable a managed identity on the service instance:

```bash
# Example: Enable system-assigned identity on ACI
az container create \
  --resource-group myResourceGroup \
  --name mycontainer \
  --assign-identity
```

### Step 2: Assign RBAC Role

Assign an appropriate Azure role to the managed identity:

```bash
# Get the principal ID of the managed identity
PRINCIPAL_ID=$(az container show \
  --resource-group myResourceGroup \
  --name mycontainer \
  --query identity.principalId --output tsv)

# Get the registry resource ID
REGISTRY_ID=$(az acr show --name myregistry --query id --output tsv)

# Assign AcrPull role (for non-ABAC registries)
az role assignment create \
  --assignee $PRINCIPAL_ID \
  --role AcrPull \
  --scope $REGISTRY_ID

# Or for ABAC-enabled registries, use:
# --role "Container Registry Repository Reader"
```

### Step 3: Enable Trusted Services on Registry

```bash
az acr update --name myregistry --allow-trusted-services true
```

### Step 4: Authenticate and Access

The service uses its managed identity to authenticate:

```bash
# The service authenticates automatically using managed identity
# No explicit login required - the identity tokens are used
```

## ACR Tasks Network Bypass Policy

### Background

ACR Tasks have a separate network bypass policy (`networkRuleBypassAllowedForTasks`) that controls whether tasks using System Assigned Managed Identity (SAMI) can bypass network restrictions.

### Important Change (June 2025)

As of **June 1, 2025**, if `networkRuleBypassAllowedForTasks` is set to `false` (or not set), network bypass for tasks using SAMI tokens will be **denied by default**.

### Enable Network Bypass for Tasks

```bash
registry="myregistry"
resourceGroup="myresourcegroup"

az resource update \
  --namespace Microsoft.ContainerRegistry \
  --resource-type registries \
  --name $registry \
  --resource-group $resourceGroup \
  --api-version 2025-06-01-preview \
  --set properties.networkRuleBypassAllowedForTasks=true
```

### Disable Network Bypass for Tasks

```bash
az resource update \
  --namespace Microsoft.ContainerRegistry \
  --resource-type registries \
  --name $registry \
  --resource-group $resourceGroup \
  --api-version 2025-06-01-preview \
  --set properties.networkRuleBypassAllowedForTasks=false
```

### Check Current Setting

```bash
az resource show \
  --namespace Microsoft.ContainerRegistry \
  --resource-type registries \
  --name $registry \
  --resource-group $resourceGroup \
  --api-version 2025-06-01-preview \
  --query properties.networkRuleBypassAllowedForTasks
```

### Security Considerations for Tasks

When enabling network bypass for tasks:

1. **Token Security**: SAMI tokens are sensitive credentials
2. **Best Practices**:
   - Never output tokens to logs
   - Implement strict logging hygiene
   - Monitor for accidental token leakage
   - Regularly audit task definitions and logs

### Alternative: Use Agent Pools

Instead of enabling network bypass, use dedicated agent pools:

```bash
# Create agent pool in VNet
az acr agentpool create \
  --name myagentpool \
  --registry myregistry \
  --subnet-id $SUBNET_ID

# Run tasks on agent pool
az acr build \
  --registry myregistry \
  --agent-pool myagentpool \
  --image myimage:mytag \
  --file Dockerfile \
  https://github.com/example/repo.git
```

## Service-Specific Configurations

### Azure Container Instances

```bash
# Create ACI with managed identity that pulls from ACR
az container create \
  --resource-group myResourceGroup \
  --name mycontainer \
  --image myregistry.azurecr.io/myimage:latest \
  --assign-identity \
  --acr-identity [system]
```

### Microsoft Defender for Cloud

No additional configuration required:
- Defender automatically uses trusted services bypass
- Scans images in network-restricted registries

### Azure Machine Learning

```bash
# Configure ML workspace to use ACR with managed identity
az ml workspace update \
  --name myworkspace \
  --resource-group myResourceGroup \
  --container-registry /subscriptions/.../registries/myregistry
```

## Troubleshooting

### 403 Forbidden Errors

If trusted service access fails:

1. **Verify trusted services is enabled**:
   ```bash
   az acr show --name myregistry --query networkRuleBypassOptions
   ```

2. **Check managed identity assignment**:
   ```bash
   az role assignment list --assignee <principal-id> --scope <registry-id>
   ```

3. **Verify network configuration**:
   - Ensure NOT using service endpoints (trusted services don't work with service endpoints)
   - Confirm using Private Link or IP firewall rules

### Service Endpoints vs Private Link

| Network Configuration | Trusted Services Support |
|----------------------|-------------------------|
| Service Endpoints | **NOT SUPPORTED** |
| Private Link (Private Endpoints) | Supported |
| IP Firewall Rules | Supported |
| Both Private Link and IP Rules | Supported |

### ACR Tasks Failures (403 Errors)

After June 2025, if tasks fail with 403:

1. **Option 1**: Enable network bypass for tasks:
   ```bash
   az resource update ... --set properties.networkRuleBypassAllowedForTasks=true
   ```

2. **Option 2**: Use agent pools in VNet

3. **Option 3**: Use User-Assigned Managed Identity (not affected by policy)

## Best Practices

1. **Minimal Access**: Only enable trusted services when necessary
2. **Use Private Link**: Preferred over IP firewall rules for security
3. **RBAC Least Privilege**: Assign minimum required roles to managed identities
4. **Monitor Access**: Enable diagnostic logging to track service access
5. **Regular Audits**: Review which services are accessing the registry
6. **Agent Pools for Tasks**: Consider agent pools over network bypass for better control

## Source Files

- `/submodules/azure-management-docs/articles/container-registry/allow-access-trusted-services.md`
- `/submodules/azure-management-docs/articles/container-registry/manage-network-bypass-policy-for-tasks.md`
- `/submodules/azure-management-docs/articles/container-registry/tasks-agent-pools.md`
- `/submodules/azure-management-docs/articles/container-registry/container-registry-private-link.md`

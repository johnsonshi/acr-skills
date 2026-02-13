# ACR Tasks - Authentication

## Overview

ACR Tasks supports multiple authentication methods to access Azure resources, external registries, and secrets during task execution.

## Managed Identities

### Types of Managed Identities

| Type | Description | Best For |
|------|-------------|----------|
| System-assigned | Auto-created, tied to task lifecycle | Single-task scenarios |
| User-assigned | Persistent, can be shared | Multi-task, cross-resource scenarios |

### System-Assigned Identity

Automatically created when the task is created, deleted when task is deleted.

**Create Task with System-Assigned Identity:**
```bash
az acr task create \
    --registry myregistry \
    --name my-task \
    --image myapp:{{.Run.ID}} \
    --context https://github.com/user/repo.git#main \
    --file Dockerfile \
    --assign-identity
```

**Get Identity Properties:**
```bash
az acr task show \
    --registry myregistry \
    --name my-task \
    --query identity
```

### User-Assigned Identity

Persistent identity that can be assigned to multiple resources.

**Create User-Assigned Identity:**
```bash
az identity create \
    --resource-group myResourceGroup \
    --name myTaskIdentity
```

**Get Identity Resource ID:**
```bash
resourceID=$(az identity show \
    --resource-group myResourceGroup \
    --name myTaskIdentity \
    --query id --output tsv)
```

**Create Task with User-Assigned Identity:**
```bash
az acr task create \
    --registry myregistry \
    --name my-task \
    --image myapp:{{.Run.ID}} \
    --context https://github.com/user/repo.git#main \
    --file Dockerfile \
    --assign-identity $resourceID
```

### Grant Identity Permissions

**Get Principal ID (System-Assigned):**
```bash
principalID=$(az acr task show \
    --registry myregistry \
    --name my-task \
    --query identity.principalId --output tsv)
```

**Get Principal ID (User-Assigned):**
```bash
principalID=$(az identity show \
    --resource-group myResourceGroup \
    --name myTaskIdentity \
    --query principalId --output tsv)
```

**Assign Role:**
```bash
# For ABAC-enabled registries
ROLE="Container Registry Repository Reader"

# For non-ABAC registries
# ROLE="AcrPull"

az role assignment create \
    --assignee $principalID \
    --scope <resource-id> \
    --role "$ROLE"
```

## Cross-Registry Authentication

### Scenario
Pull base images from one registry while building in another.

### Setup Steps

1. **Create Task with Identity:**
```bash
az acr task create \
    --registry myregistry \
    --name cross-reg-task \
    --context /dev/null \
    --file helloworldtask.yaml \
    --assign-identity
```

2. **Grant Pull Access to Base Registry:**
```bash
baseregID=$(az acr show --name mybaseregistry --query id --output tsv)

# For ABAC-enabled registries
ROLE="Container Registry Repository Reader"

az role assignment create \
    --assignee $principalID \
    --scope $baseregID \
    --role "$ROLE"
```

3. **Add Credentials to Task:**
```bash
# System-assigned identity
az acr task credential add \
    --name cross-reg-task \
    --registry myregistry \
    --login-server mybaseregistry.azurecr.io \
    --use-identity [system]

# User-assigned identity
az acr task credential add \
    --name cross-reg-task \
    --registry myregistry \
    --login-server mybaseregistry.azurecr.io \
    --use-identity $clientID
```

### Example YAML for Cross-Registry

```yaml
version: v1.1.0
steps:
  - build: -t $Registry/myapp:$ID https://github.com/user/repo.git#main -f Dockerfile --build-arg REGISTRY_NAME=mybaseregistry.azurecr.io
  - push: ["$Registry/myapp:$ID"]
```

## Azure Key Vault Integration

### Overview
Store sensitive credentials in Key Vault and access them during task execution.

### Setup Steps

1. **Create Key Vault:**
```bash
az keyvault create \
    --name mykeyvault \
    --resource-group myResourceGroup \
    --location eastus
```

2. **Store Secrets:**
```bash
az keyvault secret set \
    --vault-name mykeyvault \
    --name UserName \
    --value "$USERNAME"

az keyvault secret set \
    --vault-name mykeyvault \
    --name Password \
    --value "$PASSWORD"
```

3. **Grant Identity Access:**
```bash
az keyvault set-policy \
    --name mykeyvault \
    --resource-group myResourceGroup \
    --object-id $principalID \
    --secret-permissions get
```

### Using Key Vault Secrets in YAML

```yaml
version: v1.1.0
secrets:
  - id: username
    keyvault: https://mykeyvault.vault.azure.net/secrets/UserName
  - id: password
    keyvault: https://mykeyvault.vault.azure.net/secrets/Password
steps:
  - cmd: bash echo '{{.Secrets.password}}' | docker login --username '{{.Secrets.username}}' --password-stdin registry.example.com
  - build: -t $Registry/myapp:$ID https://github.com/user/repo.git
  - push: ["$Registry/myapp:$ID"]
```

### Key Vault with User-Assigned Identity

```yaml
version: v1.1.0
secrets:
  - id: mysecret
    keyvault: https://mykeyvault.vault.azure.net/secrets/MySecret
    clientID: <user-assigned-identity-client-id>
steps:
  - cmd: myapp --secret {{.Secrets.mysecret}}
```

## External Registry Authentication

### Docker Hub

**Store Credentials:**
```yaml
version: v1.1.0
secrets:
  - id: dockeruser
    keyvault: https://mykeyvault.vault.azure.net/secrets/DockerUsername
  - id: dockerpass
    keyvault: https://mykeyvault.vault.azure.net/secrets/DockerPassword
steps:
  - cmd: bash echo '{{.Secrets.dockerpass}}' | docker login --username '{{.Secrets.dockeruser}}' --password-stdin
  - build: -t hubuser/myrepo:$ID .
  - push: ["hubuser/myrepo:$ID"]
```

### Service Principal Credentials

**Add Credentials to Task:**
```bash
az acr task credential add \
    --name my-task \
    --registry myregistry \
    --login-server externalregistry.example.com \
    --username $SP_APP_ID \
    --password $SP_PASSWORD
```

### Update Credentials

```bash
az acr task credential update \
    --name my-task \
    --registry myregistry \
    --login-server externalregistry.example.com \
    --username $NEW_USERNAME \
    --password $NEW_PASSWORD
```

### Remove Credentials

```bash
az acr task credential remove \
    --name my-task \
    --registry myregistry \
    --login-server externalregistry.example.com
```

## ABAC-Enabled Registries

For registries with Microsoft Entra Attribute-Based Access Control:

### Quick Builds

```bash
az acr build \
    --registry myregistry \
    --image myapp:v1 \
    --source-acr-auth-id [caller] \
    .
```

### Quick Runs

```bash
az acr run \
    --registry myregistry \
    --source-acr-auth-id [caller] \
    --cmd '$Registry/myimage:tag' \
    /dev/null
```

### Tasks with ABAC

```bash
az acr task create \
    --registry myregistry \
    --name my-task \
    --image myapp:{{.Run.ID}} \
    --context https://github.com/user/repo.git#main \
    --file Dockerfile \
    --assign-identity $resourceID \
    --source-acr-auth-id $resourceID
```

## Azure Roles for ACR Tasks

### Non-ABAC Registries

| Role | Permissions |
|------|-------------|
| AcrPull | Pull images |
| AcrPush | Push and pull images |
| AcrDelete | Delete images |
| Contributor | Full access |

### ABAC-Enabled Registries

| Role | Permissions |
|------|-------------|
| Container Registry Repository Reader | Read repository content |
| Container Registry Repository Writer | Read and write repository content |
| Container Registry Repository Contributor | Full repository access |

## Authentication Best Practices

1. **Use Managed Identities**: Avoid storing credentials where possible
2. **Principle of Least Privilege**: Grant minimum required permissions
3. **Key Vault for Secrets**: Store external credentials in Key Vault
4. **Rotate Credentials**: Regularly update service principal passwords
5. **Monitor Access**: Use Azure Monitor to track authentication events
6. **Separate Identities**: Use different identities for different environments

## Troubleshooting Authentication

### Common Issues

**1. Identity Not Authorized**
```
Error: unauthorized: identity not authorized
```
- Check role assignments
- Verify identity is assigned to task
- Wait for role assignment propagation (can take several minutes)

**2. Key Vault Access Denied**
```
Error: Access denied to Key Vault
```
- Check Key Vault access policy
- Verify identity's object ID
- Ensure secret permissions include "get"

**3. Cross-Registry Authentication Failed**
- Verify credentials are added to task
- Check network connectivity
- Verify registry login server name

### Debug Commands

```bash
# Check task identity
az acr task show --registry myregistry --name my-task --query identity

# List task credentials
az acr task credential list --registry myregistry --name my-task

# Check role assignments
az role assignment list --assignee $principalID

# Test Key Vault access
az keyvault secret show --vault-name mykeyvault --name MySecret
```

## Related Topics

- [Triggers](./triggers.md)
- [YAML Reference](./yaml-reference.md)
- [CLI Commands](./cli-commands.md)

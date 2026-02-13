# ACR Artifact Cache - Setup with Azure CLI

## Prerequisites

Before configuring artifact cache with Azure CLI, ensure you have:

1. **Azure CLI** version 2.46.0 or later
   ```bash
   # Check version
   az --version

   # Install or upgrade
   # See: https://docs.microsoft.com/cli/azure/install-azure-cli
   ```

2. **Azure Subscription** with an active account
   ```bash
   # Login to Azure
   az login

   # Set subscription (if needed)
   az account set --subscription "Your-Subscription-Name-or-ID"
   ```

3. **Existing ACR Instance**
   ```bash
   # Create if needed
   az acr create --resource-group MyResourceGroup --name MyRegistry --sku Standard
   ```

4. **Azure Key Vault** (for authenticated pulls)
   ```bash
   # Create if needed
   az keyvault create --name MyKeyVault --resource-group MyResourceGroup --location eastus
   ```

5. **Permissions** to set and retrieve secrets from Key Vault

## Step 1: Create Credentials (For Authenticated Registries)

Credentials are required for:
- Docker Hub (always required)
- Google Artifact Registry
- Private upstream registries

### 1.1 Store Secrets in Key Vault

First, add your upstream registry credentials as secrets in Azure Key Vault:

```bash
# Store username secret
az keyvault secret set \
  --vault-name MyKeyVault \
  --name dockerhub-username \
  --value "your-dockerhub-username"

# Store password/token secret
az keyvault secret set \
  --vault-name MyKeyVault \
  --name dockerhub-password \
  --value "your-dockerhub-access-token"
```

### 1.2 Create Credential Set

Create a credential set that references your Key Vault secrets:

```bash
az acr credential-set create \
  --registry MyRegistry \
  --name MyDockerHubCredSet \
  --login-server docker.io \
  --username-id https://MyKeyVault.vault.azure.net/secrets/dockerhub-username \
  --password-id https://MyKeyVault.vault.azure.net/secrets/dockerhub-password
```

**Parameters:**
| Parameter | Description |
|-----------|-------------|
| `-r, --registry` | Name of your ACR |
| `-n, --name` | Name for the credential set |
| `-l, --login-server` | Upstream registry login server |
| `-u, --username-id` | Key Vault secret URI for username |
| `-p, --password-id` | Key Vault secret URI for password |

### 1.3 View Credential Set

```bash
az acr credential-set show \
  --registry MyRegistry \
  --name MyDockerHubCredSet
```

**Example Output:**
```json
{
  "authCredentials": [
    {
      "name": "Credential1",
      "passwordSecretIdentifier": "https://MyKeyVault.vault.azure.net/secrets/dockerhub-password",
      "usernameSecretIdentifier": "https://MyKeyVault.vault.azure.net/secrets/dockerhub-username"
    }
  ],
  "creationDate": "2025-01-15T10:30:00.000000+00:00",
  "identity": {
    "principalId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
    "tenantId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
    "type": "SystemAssigned"
  },
  "loginServer": "docker.io",
  "name": "MyDockerHubCredSet",
  "resourceGroup": "MyResourceGroup",
  "type": "Microsoft.ContainerRegistry/registries/credentialSets"
}
```

### 1.4 Update Credential Set

Update username or password secret references:

```bash
az acr credential-set update \
  --registry MyRegistry \
  --name MyDockerHubCredSet \
  --password-id https://MyKeyVault.vault.azure.net/secrets/new-password-secret
```

## Step 2: Assign Key Vault Permissions

The credential set's system identity needs permission to access Key Vault secrets.

### Option A: Using Azure RBAC

#### Azure CLI Method

```bash
# Get the principal ID of the credential set
az acr credential-set show \
  --name MyDockerHubCredSet \
  --registry MyRegistry \
  --query 'identity.principalId' \
  --output tsv

# Get Key Vault resource ID
az keyvault show \
  --name MyKeyVault \
  --resource-group MyResourceGroup \
  --query 'id' \
  --output tsv

# Assign Key Vault Secrets User role
az role assignment create \
  --role "Key Vault Secrets User" \
  --assignee <PRINCIPAL_ID> \
  --scope <KEYVAULT_RESOURCE_ID>
```

#### Bash Script Method (Variables)

```bash
# Get principal ID
PRINCIPAL_ID=$(az acr credential-set show \
  --name MyDockerHubCredSet \
  --registry MyRegistry \
  --query 'identity.principalId' \
  --output tsv)

# Get Key Vault resource ID
KEYVAULT_ID=$(az keyvault show \
  --name MyKeyVault \
  --resource-group MyResourceGroup \
  --query 'id' \
  --output tsv)

# Assign role
az role assignment create \
  --role "Key Vault Secrets User" \
  --assignee $PRINCIPAL_ID \
  --scope $KEYVAULT_ID
```

### Option B: Using Access Policies

If your Key Vault uses access policies instead of RBAC:

```bash
# Get principal ID
PRINCIPAL_ID=$(az acr credential-set show \
  --name MyDockerHubCredSet \
  --registry MyRegistry \
  --query 'identity.principalId' \
  --output tsv)

# Set access policy
az keyvault set-policy \
  --name MyKeyVault \
  --object-id $PRINCIPAL_ID \
  --secret-permissions get
```

### Option C: Grant Access to Specific Secrets Only

For more restrictive access:

```bash
# Get username secret ID
USERNAME_SECRET_ID=$(az keyvault secret show \
  --vault-name MyKeyVault \
  --name dockerhub-username \
  --query 'id' \
  --output tsv)

# Get password secret ID
PASSWORD_SECRET_ID=$(az keyvault secret show \
  --vault-name MyKeyVault \
  --name dockerhub-password \
  --query 'id' \
  --output tsv)

# Assign role to username secret
az role assignment create \
  --role "Key Vault Secrets User" \
  --assignee $PRINCIPAL_ID \
  --scope $USERNAME_SECRET_ID

# Assign role to password secret
az role assignment create \
  --role "Key Vault Secrets User" \
  --assignee $PRINCIPAL_ID \
  --scope $PASSWORD_SECRET_ID
```

## Step 3: Create Cache Rule

### With Credentials (Authenticated Pull)

```bash
az acr cache create \
  --registry MyRegistry \
  --name MyUbuntuCache \
  --source-repo docker.io/library/ubuntu \
  --target-repo ubuntu \
  --cred-set MyDockerHubCredSet
```

### Without Credentials (Unauthenticated Pull)

For registries that support unauthenticated pulls (MCR, GitHub, etc.):

```bash
az acr cache create \
  --registry MyRegistry \
  --name MyMCRCache \
  --source-repo mcr.microsoft.com/dotnet/runtime \
  --target-repo dotnet/runtime
```

**Parameters:**
| Parameter | Description |
|-----------|-------------|
| `-r, --registry` | Name of your ACR |
| `-n, --name` | Unique name for the cache rule |
| `-s, --source-repo` | Full path to upstream repository |
| `-t, --target-repo` | Local ACR repository namespace |
| `-c, --cred-set` | Name of credential set (optional) |

## Step 4: View Cache Rules

### Show Specific Cache Rule

```bash
az acr cache show \
  --registry MyRegistry \
  --name MyUbuntuCache
```

### List All Cache Rules

```bash
az acr cache list \
  --registry MyRegistry
```

**Example Output:**
```json
{
  "creationDate": "2025-01-15T11:00:00.000000+00:00",
  "credentialSetResourceId": "/subscriptions/.../credentialSets/MyDockerHubCredSet",
  "name": "MyUbuntuCache",
  "resourceGroup": "MyResourceGroup",
  "sourceRepository": "docker.io/library/ubuntu",
  "targetRepository": "ubuntu",
  "type": "Microsoft.ContainerRegistry/registries/cacheRules"
}
```

## Step 5: Update Cache Rule

Update the credential set associated with a cache rule:

```bash
az acr cache update \
  --registry MyRegistry \
  --name MyUbuntuCache \
  --cred-set NewCredSet
```

Remove credentials from a cache rule:

```bash
az acr cache update \
  --registry MyRegistry \
  --name MyUbuntuCache \
  --remove-cred-set
```

## Step 6: Pull Your Image

After creating the cache rule, pull images using your ACR as the source:

```bash
# Login to ACR
az acr login --name MyRegistry

# Pull image (triggers caching on first pull)
docker pull myregistry.azurecr.io/ubuntu:latest
```

## Complete Example: Docker Hub Image Caching

Here's a complete workflow for caching Docker Hub images:

```bash
#!/bin/bash

# Variables
RESOURCE_GROUP="MyResourceGroup"
ACR_NAME="myregistry"
KEYVAULT_NAME="mykeyvault"
LOCATION="eastus"

# Create Resource Group (if needed)
az group create --name $RESOURCE_GROUP --location $LOCATION

# Create Key Vault (if needed)
az keyvault create \
  --name $KEYVAULT_NAME \
  --resource-group $RESOURCE_GROUP \
  --location $LOCATION

# Store Docker Hub credentials
az keyvault secret set \
  --vault-name $KEYVAULT_NAME \
  --name dockerhub-username \
  --value "YOUR_DOCKERHUB_USERNAME"

az keyvault secret set \
  --vault-name $KEYVAULT_NAME \
  --name dockerhub-token \
  --value "YOUR_DOCKERHUB_ACCESS_TOKEN"

# Create ACR (if needed)
az acr create \
  --resource-group $RESOURCE_GROUP \
  --name $ACR_NAME \
  --sku Standard

# Create credential set
az acr credential-set create \
  --registry $ACR_NAME \
  --name DockerHubCreds \
  --login-server docker.io \
  --username-id "https://${KEYVAULT_NAME}.vault.azure.net/secrets/dockerhub-username" \
  --password-id "https://${KEYVAULT_NAME}.vault.azure.net/secrets/dockerhub-token"

# Get credential set principal ID
PRINCIPAL_ID=$(az acr credential-set show \
  --registry $ACR_NAME \
  --name DockerHubCreds \
  --query 'identity.principalId' \
  --output tsv)

# Get Key Vault resource ID
KEYVAULT_ID=$(az keyvault show \
  --name $KEYVAULT_NAME \
  --resource-group $RESOURCE_GROUP \
  --query 'id' \
  --output tsv)

# Assign Key Vault access
az role assignment create \
  --role "Key Vault Secrets User" \
  --assignee $PRINCIPAL_ID \
  --scope $KEYVAULT_ID

# Create cache rule for nginx
az acr cache create \
  --registry $ACR_NAME \
  --name nginx-cache \
  --source-repo docker.io/library/nginx \
  --target-repo nginx \
  --cred-set DockerHubCreds

# Login and pull
az acr login --name $ACR_NAME
docker pull ${ACR_NAME}.azurecr.io/nginx:latest
```

## Complete Example: MCR Image Caching (Unauthenticated)

```bash
#!/bin/bash

ACR_NAME="myregistry"

# Create cache rule for .NET Runtime (no credentials needed)
az acr cache create \
  --registry $ACR_NAME \
  --name dotnet-runtime-cache \
  --source-repo mcr.microsoft.com/dotnet/runtime \
  --target-repo dotnet/runtime

# Create cache rule for Azure CLI
az acr cache create \
  --registry $ACR_NAME \
  --name azure-cli-cache \
  --source-repo mcr.microsoft.com/azure-cli \
  --target-repo azure-cli

# Pull images
az acr login --name $ACR_NAME
docker pull ${ACR_NAME}.azurecr.io/dotnet/runtime:8.0
docker pull ${ACR_NAME}.azurecr.io/azure-cli:latest
```

## Clean Up Resources

### Delete Cache Rule

```bash
az acr cache delete \
  --registry MyRegistry \
  --name MyUbuntuCache
```

### Delete Credential Set

```bash
az acr credential-set delete \
  --registry MyRegistry \
  --name MyDockerHubCredSet
```

### Delete Key Vault Secrets

```bash
az keyvault secret delete \
  --vault-name MyKeyVault \
  --name dockerhub-username

az keyvault secret delete \
  --vault-name MyKeyVault \
  --name dockerhub-password
```

## CLI Command Reference

| Command | Description |
|---------|-------------|
| `az acr credential-set create` | Create a credential set |
| `az acr credential-set show` | Show credential set details |
| `az acr credential-set update` | Update credential set |
| `az acr credential-set delete` | Delete credential set |
| `az acr credential-set list` | List all credential sets |
| `az acr cache create` | Create a cache rule |
| `az acr cache show` | Show cache rule details |
| `az acr cache update` | Update cache rule |
| `az acr cache delete` | Delete cache rule |
| `az acr cache list` | List all cache rules |

## References

- [Azure CLI ACR Commands](https://docs.microsoft.com/cli/azure/acr)
- [Azure CLI ACR Cache Commands](https://docs.microsoft.com/cli/azure/acr/cache)
- [Azure CLI ACR Credential Set Commands](https://docs.microsoft.com/cli/azure/acr/credential-set)
- [Azure Key Vault CLI](https://docs.microsoft.com/cli/azure/keyvault)

# Azure Container Registry Authentication - CLI Commands Reference

## Login and Authentication Commands

### az acr login
Authenticates to an Azure Container Registry.

```bash
# Basic login (requires Docker daemon)
az acr login --name <registry-name>

# Login with exposed token (no Docker required)
az acr login --name <registry-name> --expose-token

# Login to specific resource group registry
az acr login --name <registry-name> --resource-group <resource-group>

# Login with suffix (for sovereign clouds)
az acr login --name <registry-name> --suffix <suffix>
```

**Parameters:**
| Parameter | Description | Required |
|-----------|-------------|----------|
| `--name`, `-n` | Registry name | Yes |
| `--expose-token` | Return access token instead of Docker login | No |
| `--resource-group`, `-g` | Resource group name | No |
| `--suffix` | Login server suffix | No |
| `--username`, `-u` | Username for login | No |
| `--password`, `-p` | Password for login | No |

### az login (Azure CLI)
Azure authentication commands relevant to ACR.

```bash
# Standard login (ARM-scoped token)
az login

# Login with ACR-scoped token (recommended for ACR)
az login --scope https://containerregistry.azure.net/.default

# Login with service principal
az login --service-principal --username <app-id> --password <password> --tenant <tenant-id>

# Login with service principal certificate
az login --service-principal --username <app-id> --tenant <tenant-id> --password /path/to/cert.pem

# Login with managed identity
az login --identity

# Login with specific user-assigned managed identity
az login --identity --username <client-id-of-user-assigned-identity>
```

### PowerShell Authentication Commands

```powershell
# Azure authentication
Connect-AzAccount

# ACR authentication
Connect-AzContainerRegistry -Name <registry-name>
```

## Admin Account Management

### az acr update (Admin User)
Enable or disable admin account.

```bash
# Enable admin user
az acr update --name <registry-name> --admin-enabled true

# Disable admin user
az acr update --name <registry-name> --admin-enabled false
```

### az acr credential
Manage admin account credentials.

```bash
# Show admin credentials
az acr credential show --name <registry-name>

# Renew password1
az acr credential renew --name <registry-name> --password-name password1

# Renew password2
az acr credential renew --name <registry-name> --password-name password2
```

## Anonymous Pull Access

### az acr update (Anonymous Pull)
Configure anonymous pull access.

```bash
# Enable anonymous pull
az acr update --name <registry-name> --anonymous-pull-enabled

# Disable anonymous pull
az acr update --name <registry-name> --anonymous-pull-enabled false
```

### az acr show (Query Anonymous Pull Status)
```bash
# Check if anonymous pull is enabled
az acr show --name <registry-name> --query anonymousPullEnabled
```

## Authentication Scope Configuration

### az acr config authentication-as-arm
Configure registry acceptance of Microsoft Entra authentication scopes.

```bash
# Show current configuration
az acr config authentication-as-arm show --registry <registry-name>

# Disable ARM-scoped authentication (accept only ACR-scoped)
az acr config authentication-as-arm update --registry <registry-name> --status disabled

# Enable ARM-scoped authentication (accept both)
az acr config authentication-as-arm update --registry <registry-name> --status enabled
```

## Trusted Services Configuration

### az acr update (Trusted Services)
Configure trusted Azure services bypass.

```bash
# Enable trusted services
az acr update --name <registry-name> --allow-trusted-services true

# Disable trusted services
az acr update --name <registry-name> --allow-trusted-services false
```

## Service Principal Management

### az ad sp create-for-rbac
Create a service principal for ACR access.

```bash
# Create service principal with AcrPull role
az ad sp create-for-rbac \
    --name <service-principal-name> \
    --scopes /subscriptions/<subscription-id>/resourceGroups/<resource-group>/providers/Microsoft.ContainerRegistry/registries/<registry-name> \
    --role acrpull

# Create service principal with AcrPush role
az ad sp create-for-rbac \
    --name <service-principal-name> \
    --scopes /subscriptions/<subscription-id>/resourceGroups/<resource-group>/providers/Microsoft.ContainerRegistry/registries/<registry-name> \
    --role acrpush
```

### az ad sp credential reset
Reset service principal credentials.

```bash
# Reset password
az ad sp credential reset --name <service-principal-name>

# Query new password
az ad sp credential reset --name <service-principal-name> --query password --output tsv
```

## Role Assignment Commands

### az role assignment create
Assign roles to identities for ACR access.

```bash
# Assign AcrPull role (non-ABAC registries)
az role assignment create \
    --assignee <principal-id> \
    --scope <registry-resource-id> \
    --role AcrPull

# Assign AcrPush role (non-ABAC registries)
az role assignment create \
    --assignee <principal-id> \
    --scope <registry-resource-id> \
    --role AcrPush

# Assign Container Registry Repository Reader (ABAC registries)
az role assignment create \
    --assignee <principal-id> \
    --scope <registry-resource-id> \
    --role "Container Registry Repository Reader"

# Assign Container Registry Repository Writer (ABAC registries)
az role assignment create \
    --assignee <principal-id> \
    --scope <registry-resource-id> \
    --role "Container Registry Repository Writer"
```

### az role assignment list
List role assignments for a registry.

```bash
# List all role assignments on a registry
az role assignment list \
    --scope /subscriptions/<subscription-id>/resourceGroups/<resource-group>/providers/Microsoft.ContainerRegistry/registries/<registry-name>
```

## Managed Identity Commands

### az identity create
Create a user-assigned managed identity.

```bash
az identity create --resource-group <resource-group> --name <identity-name>
```

### az identity show
Get details of a managed identity.

```bash
# Get resource ID
az identity show --resource-group <resource-group> --name <identity-name> --query id --output tsv

# Get principal ID
az identity show --resource-group <resource-group> --name <identity-name> --query principalId --output tsv

# Get client ID
az identity show --resource-group <resource-group> --name <identity-name> --query clientId --output tsv
```

### az vm identity assign
Assign managed identity to a VM.

```bash
# Assign system-assigned identity
az vm identity assign --resource-group <resource-group> --name <vm-name>

# Assign user-assigned identity
az vm identity assign --resource-group <resource-group> --name <vm-name> --identities <identity-resource-id>
```

## Token and Scope Map Management

### az acr token create
Create repository-scoped tokens.

```bash
# Create token with inline repository permissions
az acr token create \
    --name <token-name> \
    --registry <registry-name> \
    --repository <repo-name> content/write content/read

# Create token with scope map
az acr token create \
    --name <token-name> \
    --registry <registry-name> \
    --scope-map <scope-map-name>

# Create token with wildcard permissions
az acr token create \
    --name <token-name> \
    --registry <registry-name> \
    --repository "samples/*" content/write content/read
```

### az acr token list
List tokens in a registry.

```bash
az acr token list --registry <registry-name> --output table
```

### az acr token show
Show token details.

```bash
az acr token show --name <token-name> --registry <registry-name>
```

### az acr token update
Update token settings.

```bash
# Disable token
az acr token update --name <token-name> --registry <registry-name> --status disabled

# Enable token
az acr token update --name <token-name> --registry <registry-name> --status enabled

# Update scope map
az acr token update --name <token-name> --registry <registry-name> --scope-map <new-scope-map>
```

### az acr token delete
Delete a token.

```bash
az acr token delete --name <token-name> --registry <registry-name>
```

### az acr token credential generate
Generate or regenerate token passwords.

```bash
# Generate password1 with 30-day expiration
az acr token credential generate \
    --name <token-name> \
    --registry <registry-name> \
    --expiration-in-days 30 \
    --password1

# Generate password2
az acr token credential generate \
    --name <token-name> \
    --registry <registry-name> \
    --password2
```

### az acr scope-map create
Create a scope map.

```bash
az acr scope-map create \
    --name <scope-map-name> \
    --registry <registry-name> \
    --repository <repo-name> content/write content/read \
    --description "Description of scope map"
```

### az acr scope-map list
List scope maps.

```bash
az acr scope-map list --registry <registry-name> --output table
```

### az acr scope-map show
Show scope map details.

```bash
az acr scope-map show --name <scope-map-name> --registry <registry-name>
```

### az acr scope-map update
Update scope map permissions.

```bash
# Add repository permissions
az acr scope-map update \
    --name <scope-map-name> \
    --registry <registry-name> \
    --add-repository <repo-name> content/write content/read

# Remove repository permissions
az acr scope-map update \
    --name <scope-map-name> \
    --registry <registry-name> \
    --remove-repository <repo-name> content/write
```

## Health Check Commands

### az acr check-health
Diagnose authentication and connectivity issues.

```bash
# Check local environment only
az acr check-health

# Check environment and registry access
az acr check-health --name <registry-name>

# Check with virtual network
az acr check-health --name <registry-name> --vnet <vnet-name-or-id>

# Ignore errors and continue
az acr check-health --name <registry-name> --ignore-errors --yes
```

## Docker Commands for ACR

### docker login
Authenticate directly with Docker.

```bash
# Login with service principal
docker login <registry-name>.azurecr.io --username <app-id> --password <password>

# Login with admin credentials
docker login <registry-name>.azurecr.io --username <admin-username> --password <admin-password>

# Login with ACR refresh token
docker login <registry-name>.azurecr.io --username 00000000-0000-0000-0000-000000000000 --password <acr-refresh-token>

# Login with stdin password
echo $PASSWORD | docker login <registry-name>.azurecr.io --username <username> --password-stdin
```

### docker logout
Clear cached credentials.

```bash
docker logout <registry-name>.azurecr.io
```

## Helm Commands for ACR

### helm registry login
Authenticate Helm with ACR.

```bash
# Login with token from az acr login --expose-token
echo $TOKEN | helm registry login <registry-name>.azurecr.io --username 00000000-0000-0000-0000-000000000000 --password-stdin
```

## Kubernetes Commands

### kubectl create secret
Create image pull secrets.

```bash
kubectl create secret docker-registry <secret-name> \
    --namespace <namespace> \
    --docker-server=<registry-name>.azurecr.io \
    --docker-username=<service-principal-id> \
    --docker-password=<service-principal-password>
```

## ACR Task Identity Commands

### az acr task credential add
Add credentials to an ACR task.

```bash
# Add system-assigned identity credentials
az acr task credential add \
    --name <task-name> \
    --registry <registry-name> \
    --login-server <target-registry>.azurecr.io \
    --use-identity [system]

# Add user-assigned identity credentials
az acr task credential add \
    --name <task-name> \
    --registry <registry-name> \
    --login-server <target-registry>.azurecr.io \
    --use-identity <client-id>
```

## Useful Queries

### Get Registry Resource ID
```bash
az acr show --name <registry-name> --query id --output tsv
```

### Get Registry Login Server
```bash
az acr show --name <registry-name> --query loginServer --output tsv
```

### Get Access Token Programmatically
```bash
# Get AAD access token for ARM
az account get-access-token --query accessToken --output tsv

# Get AAD access token for ACR
az account get-access-token --resource=https://containerregistry.azure.net --query accessToken --output tsv
```

## Sources

- `/Users/johnsonshi/repos/acr-skills/submodules/azure-management-docs/articles/container-registry/container-registry-authentication.md`
- `/Users/johnsonshi/repos/acr-skills/submodules/azure-management-docs/articles/container-registry/container-registry-auth-service-principal.md`
- `/Users/johnsonshi/repos/acr-skills/submodules/azure-management-docs/articles/container-registry/container-registry-token-based-repository-permissions.md`
- `/Users/johnsonshi/repos/acr-skills/submodules/azure-management-docs/articles/container-registry/container-registry-check-health.md`
- `/Users/johnsonshi/repos/acr-skills/submodules/acr/docs/AAD-OAuth.md`

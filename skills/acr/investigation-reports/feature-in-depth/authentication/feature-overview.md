# Azure Container Registry Authentication - Feature Overview

## Introduction

Azure Container Registry (ACR) provides multiple authentication methods to securely access and manage container images and artifacts. Authentication is required for most registry operations including pushing, pulling, and managing images.

## Authentication Methods Overview

| Method | Use Case | Microsoft Entra RBAC | Limitations |
|--------|----------|---------------------|-------------|
| Individual Microsoft Entra Identity | Interactive push/pull by developers | Yes | Token valid for 3 hours |
| Service Principal | CI/CD pipelines, headless automation | Yes | Password default expiry 1 year |
| Managed Identity | Azure service to ACR authentication | Yes | Only select Azure services |
| Admin Account | Testing, portal deployments | No (full access) | Single account, not for production |
| Non-Microsoft Entra Tokens | IoT devices, external systems | No | Not integrated with Microsoft Entra |
| Anonymous Pull | Public content distribution | N/A | Read-only, all repositories |

## 1. Microsoft Entra Individual Identity Authentication

### Overview
The most common authentication method for developers working directly with a registry. Uses the Azure CLI or PowerShell to authenticate.

### How It Works
1. User authenticates with Azure (`az login` or `Connect-AzAccount`)
2. User authenticates to registry (`az acr login` or `Connect-AzContainerRegistry`)
3. CLI obtains a Microsoft Entra token and stores it in Docker's credential store
4. Subsequent Docker commands use cached credentials

### Token Details
- **Validity**: 3 hours
- **Renewal**: Run `az acr login` again to refresh
- **Storage**: Token stored in `docker.config` file

### Authentication Without Docker Daemon
Use `--expose-token` flag to get an access token for environments without Docker:
```bash
az acr login --name <registry> --expose-token
```
Then use the token with `docker login`:
```bash
docker login myregistry.azurecr.io --username 00000000-0000-0000-0000-000000000000 --password-stdin <<< $TOKEN
```

## 2. Service Principal Authentication

### Overview
Service principals provide headless/unattended authentication for applications, CI/CD pipelines, and automated services.

### When to Use
- CI/CD pipelines (Azure DevOps, Jenkins, GitHub Actions)
- Container orchestrators (Kubernetes, Docker Swarm)
- Automated deployment systems
- Cross-tenant scenarios

### Authentication Flow
Service principals authenticate using:
- **Username**: Application (client) ID
- **Password**: Client secret

### Certificate-Based Authentication
For enhanced security, service principals can use certificates instead of passwords:
```bash
az login --service-principal --username $SP_APP_ID --tenant $SP_TENANT_ID --password /path/to/cert.pem
az acr login --name myregistry
```

### Cross-Tenant Scenarios
Service principals support cross-tenant authentication where:
1. Create a multitenant app in Tenant A
2. Provision the app in Tenant B
3. Grant permissions to the registry in Tenant B
4. Authenticate from services in Tenant A

### Password Management
- Default expiry: 1 year
- Regenerate: `az ad sp credential reset --name <sp-name>`

## 3. Managed Identity Authentication

### Overview
Managed identities allow Azure resources to authenticate to ACR without managing credentials.

### Identity Types

#### System-Assigned Identity
- Tied to a specific Azure resource
- Automatically managed lifecycle
- Deleted when resource is deleted

#### User-Assigned Identity
- Standalone Azure resource
- Can be assigned to multiple resources
- Persists independently of assigned resources

### Supported Azure Services
- Azure Virtual Machines
- Azure Kubernetes Service (AKS)
- Azure Container Instances (ACI)
- Azure Container Apps (ACA)
- Azure App Service
- Azure Functions
- Azure Machine Learning
- Azure Batch
- Azure DevOps Pipelines

### Role Requirements
For ABAC-enabled registries:
- `Container Registry Repository Reader` - Pull permissions
- `Container Registry Repository Writer` - Push and pull permissions

For non-ABAC registries:
- `AcrPull` - Pull permissions
- `AcrPush` - Push and pull permissions

## 4. Admin Account

### Overview
A built-in admin account with full registry access, disabled by default.

### Characteristics
- Two regeneratable passwords
- Full push and pull access
- Single account per registry
- No RBAC support

### Use Cases
- Testing purposes
- Portal deployments to Azure App Service
- Portal deployments to Azure Container Instances

### Security Warning
**NOT recommended for production use** because:
- Provides full registry access
- Cannot limit permissions
- Single shared account
- No audit trail per user

## 5. Non-Microsoft Entra Token-Based Repository Permissions

### Overview
Tokens with scope maps provide fine-grained repository access without Microsoft Entra integration.

### Concepts
- **Token**: Credential with generated passwords
- **Scope Map**: Defines repository permissions associated with tokens

### Available Actions
| Action | Description |
|--------|-------------|
| `content/read` | Pull artifacts |
| `content/write` | Push artifacts |
| `content/delete` | Delete artifacts |
| `metadata/read` | List tags/manifests |
| `metadata/write` | Enable/disable operations |

### Use Cases
- IoT devices with individual tokens
- External organizations with limited access
- User groups with different permission levels

### Wildcard Support
Scope maps support wildcards for pattern-based permissions:
- `samples/*` - All repositories under samples/
- `*` - All repositories (root-level wildcard)

## 6. Anonymous Pull Access

### Overview
Allows unauthenticated read access to all registry content.

### Availability
- Standard and Premium service tiers only
- Applies to all repositories in the registry

### Configuration
```bash
# Enable anonymous pull
az acr update --name myregistry --anonymous-pull-enabled

# Disable anonymous pull
az acr update --name myregistry --anonymous-pull-enabled false

# Query status
az acr show -n myregistry --query anonymousPullEnabled
```

### Important Notes
- Only data-plane read operations available
- High request rates may be throttled
- Clear existing credentials before anonymous pull

## 7. Kubernetes Authentication

### Authentication Methods by Cluster Type

| Cluster Type | Recommended Method |
|--------------|-------------------|
| AKS (same tenant) | AKS managed identity |
| AKS (cross-tenant) | Service principal |
| Non-AKS Kubernetes | Image pull secrets |

### AKS Managed Identity Integration
AKS can use its kubelet managed identity to pull images:
- Registry and cluster must be in same tenant
- Can be different subscriptions
- Automatic credential management

### Image Pull Secrets
For non-AKS clusters:
```bash
kubectl create secret docker-registry <secret-name> \
    --namespace <namespace> \
    --docker-server=<registry>.azurecr.io \
    --docker-username=<service-principal-ID> \
    --docker-password=<service-principal-password>
```

## 8. Azure Container Instances (ACI) Authentication

### Service Principal Authentication
```bash
az container create \
    --resource-group myResourceGroup \
    --name mycontainer \
    --image myregistry.azurecr.io/myimage:v1 \
    --registry-login-server myregistry.azurecr.io \
    --registry-username <service-principal-ID> \
    --registry-password <service-principal-password>
```

### Managed Identity Authentication
ACI supports managed identity for pulling from ACR:
- System-assigned or user-assigned identity
- Configure identity with appropriate role assignment

## 9. Conditional Access Policy

### Overview
Conditional Access policies enforce strong authentication and access controls for ACR.

### Prerequisites
- Disable `authentication-as-arm` for registries in the tenant
- Applies to Microsoft Entra users only (not service principals)

### Capabilities
- Enforce multi-factor authentication (MFA)
- Control access by location, device, risk level
- Session-level experience controls

## 10. Authentication Scope Configuration

### Two-Hop Authentication Flow
1. **First hop**: Authenticate with Microsoft Entra ID (`az login`)
2. **Second hop**: Authenticate with ACR (`az acr login`)

### Microsoft Entra Authentication Scopes

#### ARM-Scoped (Default)
- Broad Azure Resource Manager access
- Default when running `az login`

#### ACR-Scoped (Recommended)
- Narrow registry-specific access
- Requires: `az login --scope https://containerregistry.azure.net/.default`

### Registry Configuration
```bash
# Check current configuration
az acr config authentication-as-arm show -r <registry>

# Accept only ACR-scoped tokens (recommended)
az acr config authentication-as-arm update -r <registry> --status disabled

# Accept both ARM and ACR-scoped tokens (default)
az acr config authentication-as-arm update -r <registry> --status enabled
```

## Trusted Services

### Overview
Allows select Azure services to bypass network restrictions.

### Supported Trusted Services
| Service | Configuration Required |
|---------|----------------------|
| Azure Container Instances | Managed identity + RBAC |
| Microsoft Defender for Cloud | No |
| Azure Machine Learning | Managed identity + RBAC |
| Azure Container Registry (import) | No |

### Configuration
```bash
# Enable trusted services
az acr update --name myregistry --allow-trusted-services true

# Disable trusted services
az acr update --name myregistry --allow-trusted-services false
```

## OAuth2 Protocol Details

### ACR Token Types
1. **Refresh Token**: Long-lived, used to obtain access tokens
2. **Access Token**: Short-lived, used for specific operations

### Token Exchange Flow
1. Call `POST /oauth2/exchange` with Microsoft Entra token to get ACR refresh token
2. Call `POST /oauth2/token` with ACR refresh token to get ACR access token
3. Use access token for registry API calls

### JWT Token Claims
Refresh token claims include:
- `sub`: Subject (user email or service principal)
- `aud`: Audience (registry login server)
- `permissions`: Actions and notActions
- `tenant`: Microsoft Entra tenant ID

Access token claims include:
- `access`: Array of type, name, and actions
- `grant_type`: "access_token"

## Alternative Container Tools

### Using Podman Instead of Docker
```bash
DOCKER_COMMAND=podman az acr login --name <registry>
```

## Sources

- `/Users/johnsonshi/repos/acr-skills/submodules/azure-management-docs/articles/container-registry/container-registry-authentication.md`
- `/Users/johnsonshi/repos/acr-skills/submodules/azure-management-docs/articles/container-registry/container-registry-authentication-managed-identity.md`
- `/Users/johnsonshi/repos/acr-skills/submodules/azure-management-docs/articles/container-registry/container-registry-auth-service-principal.md`
- `/Users/johnsonshi/repos/acr-skills/submodules/azure-management-docs/articles/container-registry/authenticate-kubernetes-options.md`
- `/Users/johnsonshi/repos/acr-skills/submodules/azure-management-docs/articles/container-registry/anonymous-pull-access.md`
- `/Users/johnsonshi/repos/acr-skills/submodules/azure-management-docs/articles/container-registry/container-registry-configure-conditional-access.md`
- `/Users/johnsonshi/repos/acr-skills/submodules/azure-management-docs/articles/container-registry/container-registry-disable-authentication-as-arm.md`
- `/Users/johnsonshi/repos/acr-skills/submodules/acr/docs/AAD-OAuth.md`
- `/Users/johnsonshi/repos/acr-skills/submodules/acr/docs/Token-BasicAuth.md`

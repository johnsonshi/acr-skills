# ACR Authentication Skill

This skill provides comprehensive knowledge about Azure Container Registry authentication methods.

## When to Use This Skill

Use this skill when answering questions about:
- Logging into ACR
- Service principal authentication
- Managed identity integration
- Admin account management
- Repository-scoped tokens
- Anonymous pull access
- Kubernetes/AKS authentication
- Cross-tenant scenarios

## Authentication Methods Overview

| Method | Use Case | Microsoft Entra RBAC | Recommended |
|--------|----------|---------------------|-------------|
| Microsoft Entra Identity | Interactive dev push/pull | Yes | ✅ Production |
| Service Principal | CI/CD pipelines, automation | Yes | ✅ Production |
| Managed Identity | Azure service to ACR | Yes | ✅ Production |
| Repository Tokens | IoT devices, external systems | No | ✅ Specific scenarios |
| Admin Account | Testing, portal deployments | No | ⚠️ Testing only |
| Anonymous Pull | Public content distribution | N/A | ✅ Public registries |

## Quick Reference Commands

### Developer Login
```bash
# Standard login (requires Docker)
az acr login --name myregistry

# Login without Docker (get token)
az acr login --name myregistry --expose-token

# ACR-scoped login (recommended)
az login --scope https://containerregistry.azure.net/.default
az acr login --name myregistry
```

### Service Principal
```bash
# Create service principal with AcrPush role
az ad sp create-for-rbac \
  --name myacr-sp \
  --role AcrPush \
  --scopes /subscriptions/{sub}/resourceGroups/{rg}/providers/Microsoft.ContainerRegistry/registries/{acr}

# Use in Docker
docker login myregistry.azurecr.io -u <appId> -p <password>
```

### Managed Identity
```bash
# AKS integration (same tenant)
az aks update -n myaks -g mygroup --attach-acr myregistry

# Assign role to managed identity
az role assignment create \
  --assignee <principal-id> \
  --scope <registry-resource-id> \
  --role AcrPull
```

### Repository Tokens
```bash
# Create token with repository permissions
az acr token create \
  --name mytoken \
  --registry myregistry \
  --repository myrepo content/read content/write

# Generate password
az acr token credential generate \
  --name mytoken \
  --registry myregistry \
  --expiration-in-days 30 \
  --password1
```

### Admin Account (Testing Only)
```bash
# Enable admin
az acr update --name myregistry --admin-enabled true

# Get credentials
az acr credential show --name myregistry
```

### Anonymous Pull
```bash
# Enable anonymous pull (Standard/Premium only)
az acr update --name myregistry --anonymous-pull-enabled true
```

## RBAC Roles

### For ABAC-Enabled Registries
| Role | Capabilities |
|------|-------------|
| Container Registry Repository Reader | Pull images |
| Container Registry Repository Writer | Push and pull images |
| Container Registry Repository Contributor | Full read/write/delete |

### For Non-ABAC Registries
| Role | Capabilities |
|------|-------------|
| AcrPull | Pull images |
| AcrPush | Push and pull images |
| AcrDelete | Delete images |

## Kubernetes Authentication

### AKS (Same Tenant)
```bash
# Attach ACR to AKS
az aks update -n myaks -g mygroup --attach-acr myregistry
```

### AKS (Cross-Tenant)
Requires service principal provisioned in both tenants.

### Non-AKS Kubernetes
```bash
# Create image pull secret
kubectl create secret docker-registry acr-secret \
  --docker-server=myregistry.azurecr.io \
  --docker-username=<sp-id> \
  --docker-password=<sp-password>
```

```yaml
# Use in pod spec
spec:
  imagePullSecrets:
    - name: acr-secret
  containers:
    - name: myapp
      image: myregistry.azurecr.io/myapp:v1
```

## Token Details

| Token Type | Validity | Renewal |
|------------|----------|---------|
| `az acr login` token | 3 hours | Run `az acr login` again |
| Service principal secret | 1 year (default) | `az ad sp credential reset` |
| Repository token password | Configurable | `az acr token credential generate` |

## Authentication Scope Configuration

```bash
# Check current auth scope setting
az acr config authentication-as-arm show -r myregistry

# Disable ARM-scoped auth (recommended for security)
az acr config authentication-as-arm update -r myregistry --status disabled
```

## Trusted Services

Allow Azure services to bypass network restrictions:
```bash
az acr update --name myregistry --allow-trusted-services true
```

Supported services:
- Azure Container Instances (with managed identity)
- Microsoft Defender for Cloud
- Azure Machine Learning
- ACR (for import operations)

## Troubleshooting

```bash
# Check health and authentication
az acr check-health --name myregistry

# Check with VNet
az acr check-health --name myregistry --vnet myvnet
```

### Common Issues
| Error | Cause | Solution |
|-------|-------|----------|
| CONNECTIVITY_REFRESH_TOKEN_ERROR | Token expired | Run `az acr login` again |
| DOCKER_COMMAND_ERROR | Docker not running | Start Docker daemon |
| CONNECTIVITY_DNS_ERROR | DNS resolution failed | Check network config |
| 403 Forbidden | Missing permissions | Check RBAC role assignments |

## Reference Documentation

- **Investigation reports**: `agent-investigation-reports/acr-features/authentication/`
- **MS Learn docs**: `azure-management-docs/articles/container-registry/container-registry-authentication.md`

# ACR Image Import - Comprehensive Guide

## Overview

Azure Container Registry allows you to import (copy) container images without using Docker commands. This is particularly useful for copying images from public registries, between ACR instances, or from private non-Azure registries.

## Key Benefits

1. **No Docker Installation Required** - Import any container image regardless of client environment
2. **Multi-Architecture Support** - Automatically imports all architectures and platforms from manifest lists
3. **Network-Restricted Support** - Works with registries that don't expose public endpoints
4. **Cross-Tenant Support** - Import images between different Azure AD tenants

## Prerequisites

### Required Permissions
- `Container Registry Data Importer` role on the target registry
- `Container Registry Data Reader` role for reading from source

### CLI Requirements
- **Azure CLI**: Version 2.0.55 or later
- **Azure PowerShell**: Version 5.9.0 or later

## Import Scenarios

### 1. Import from Public Registries

#### Docker Hub
```bash
# Import official image
az acr import \
  --name myregistry \
  --source docker.io/library/hello-world:latest \
  --image hello-world:latest

# Import with credentials (recommended for rate limits)
az acr import \
  --name myregistry \
  --source docker.io/tensorflow/tensorflow:latest-gpu \
  --image tensorflow:latest-gpu \
  --username <Docker Hub user name> \
  --password <Docker Hub token>
```

#### Microsoft Container Registry (MCR)
```bash
az acr import \
  --name myregistry \
  --source mcr.microsoft.com/windows/servercore:ltsc2022 \
  --image servercore:ltsc2022
```

### 2. Import from Another Azure Container Registry

#### Same AD Tenant (Same or Different Subscription)
```bash
# Using Entra identity authentication
az acr import \
  --name myregistry \
  --source aci-helloworld:latest \
  --image aci-helloworld:latest \
  --registry /subscriptions/<subscription-id>/resourceGroups/<resource-group>/providers/Microsoft.ContainerRegistry/registries/mysourceregistry
```

#### Different AD Tenant
```bash
# Using service principal or token
az acr import \
  --name myregistry \
  --source sourceregistry.azurecr.io/sourcerepo:tag \
  --image targetimage:tag \
  --username <SP_App_ID> \
  --password <SP_Passwd>

# Using access token
az acr import \
  --name myregistry \
  --source sourceregistry.azurecr.io/sourcerepo:tag \
  --image targetimage:tag \
  --password <access-token>
```

### 3. Import from Non-Azure Private Registry
```bash
az acr import \
  --name myregistry \
  --source docker.io/privaterepo/privateimage:tag \
  --image privateimage:tag \
  --username <username> \
  --password <password>
```

### 4. Import by Digest (Without Tag)
```bash
az acr import \
   --name myregistry \
   --source docker.io/library/hello-world@sha256:abc123 \
   --repository hello-world
```

## PowerShell Equivalents

```powershell
# Import from Docker Hub
Import-AzContainerRegistryImage `
  -RegistryName myregistry `
  -ResourceGroupName myResourceGroup `
  -SourceRegistryUri docker.io `
  -SourceImage library/hello-world:latest

# Import from another ACR (same tenant)
Import-AzContainerRegistryImage `
  -RegistryName myregistry `
  -ResourceGroupName myResourceGroup `
  -SourceRegistryResourceId '/subscriptions/<sub-id>/resourceGroups/<rg>/providers/Microsoft.ContainerRegistry/registries/mysourceregistry' `
  -SourceImage aci-helloworld:latest

# Import with credentials
Import-AzContainerRegistryImage `
  -RegistryName myregistry `
  -ResourceGroupName myResourceGroup `
  -SourceRegistryUri sourceregistry.azurecr.io `
  -SourceImage sourcerepo:tag `
  -Username <SP_App_ID> `
  -Password <SP_Passwd>
```

## Network-Restricted Registries

For registries with private endpoints or firewall rules:

1. **Enable Trusted Services**: The target registry must allow access by trusted services
2. **Specify by Resource ID**: When public access is disabled, use resource ID instead of login server name

```bash
# Import to/from network-restricted registry
az acr import \
  --name myregistry \
  --source aci-helloworld:latest \
  --image aci-helloworld:latest \
  --registry /subscriptions/.../registries/mysourceregistry
```

**Important**: By default, trusted services access is enabled. If disabled, import will fail.

## Limitations

| Limitation | Value |
|------------|-------|
| Maximum manifests per imported image | 50 |
| Maximum layer size (ACR Transfer) | 8 GB |
| Cross-tenant with public access disabled | Not supported |

## Verifying Import

After import, verify with:
```bash
# List manifests
az acr manifest list-metadata \
  --name hello-world \
  --registry myregistry

# Show specific manifest
az acr manifest show-metadata \
  --name hello-world:latest \
  --registry myregistry
```

## Troubleshooting

### Common Errors

| Error | Cause | Solution |
|-------|-------|----------|
| `The remote server may not be RFC 7233 compliant` | Source registry doesn't support range headers | Use a registry that supports RFC 7233 |
| `Unexpected response status code` | Network or permission issues | Check credentials and network access |
| `Unexpected length of body in response` | Content length mismatch | Retry import; check source registry health |

### Best Practices

1. **Use Credentials for Docker Hub** - Avoid rate limiting with authenticated requests
2. **Import by Digest** - Guarantees specific image version
3. **Verify After Import** - Always confirm manifests match expectations
4. **Enable Trusted Services** - Required for network-restricted registries

## Related Features

- **ACR Transfer** - For bulk transfers between air-gapped environments
- **Geo-replication** - Automatically replicate imported images
- **Artifact Cache** - Cache images from upstream registries

## Source Documentation

- `/submodules/azure-management-docs/articles/container-registry/container-registry-import-images.md`
- `/submodules/azure-management-docs/articles/container-registry/container-registry-transfer-images.md`
- `/submodules/azure-management-docs/articles/container-registry/allow-access-trusted-services.md`

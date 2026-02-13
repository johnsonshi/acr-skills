# ACR Artifact Cache - Troubleshooting Guide

## Overview

This guide helps diagnose and resolve common issues with ACR Artifact Cache, including credential problems, cache rule conflicts, and pull failures.

## Common Issues and Solutions

### 1. Unhealthy Credentials

**Symptoms:**
- Credential set shows "Unhealthy" status in portal
- Pulls fail with authentication errors
- Error messages about invalid credentials

**Causes:**
- Key Vault secrets expired or invalid
- Key Vault access not granted to credential set identity
- Incorrect secret URIs
- Upstream registry credentials changed

**Solutions:**

#### Verify Key Vault Secrets
```bash
# Check if secrets exist and are valid
az keyvault secret show --vault-name MyKeyVault --name username-secret
az keyvault secret show --vault-name MyKeyVault --name password-secret

# Update secrets if needed
az keyvault secret set --vault-name MyKeyVault --name password-secret --value "new-token"
```

#### Verify Key Vault Access
```bash
# Get credential set principal ID
PRINCIPAL_ID=$(az acr credential-set show \
  --registry MyRegistry \
  --name MyCreds \
  --query 'identity.principalId' \
  --output tsv)

# Check role assignments
az role assignment list --assignee $PRINCIPAL_ID --all

# Grant access if missing (RBAC)
KEYVAULT_ID=$(az keyvault show --name MyKeyVault --query 'id' --output tsv)
az role assignment create \
  --role "Key Vault Secrets User" \
  --assignee $PRINCIPAL_ID \
  --scope $KEYVAULT_ID

# OR using access policies
az keyvault set-policy \
  --name MyKeyVault \
  --object-id $PRINCIPAL_ID \
  --secret-permissions get
```

#### Update Credential Set
```bash
# Update to correct secret URIs
az acr credential-set update \
  --registry MyRegistry \
  --name MyCreds \
  --username-id https://MyKeyVault.vault.azure.net/secrets/correct-username \
  --password-id https://MyKeyVault.vault.azure.net/secrets/correct-password
```

### 2. Unable to Create Cache Rule

**Symptoms:**
- Cache rule creation fails
- Error about conflicting rules
- Error about target repository

**Cause: Cache Rule Limit Reached**

```bash
# Check current rule count
az acr cache list --registry MyRegistry --query "length(@)"
```

**Solution:** Delete unused cache rules
```bash
az acr cache delete --registry MyRegistry --name unused-rule
```

**Cause: Overlapping Wildcard Rule**

Error: "There is already a cache rule with wildcard for the specified target repository"

**Solution:**
1. Identify the conflicting rule
```bash
az acr cache list --registry MyRegistry --output table
```

2. Delete conflicting rule or use static override
```bash
# Option A: Delete conflicting wildcard
az acr cache delete --registry MyRegistry --name conflicting-wildcard

# Option B: Create static rule (overrides wildcard)
az acr cache create --registry MyRegistry --name static-override \
  --source-repo docker.io/library/nginx \
  --target-repo nginx
```

**Cause: Target Repository Already Exists**

Error: "Target repository already exists"

**Solution:** Choose a different target namespace
```bash
# Use different namespace
az acr cache create --registry MyRegistry --name nginx-cache \
  --source-repo docker.io/library/nginx \
  --target-repo cached/nginx  # Different from existing "nginx" repo
```

### 3. Pull Failures After Rule Creation

**Symptoms:**
- `docker pull` fails after creating cache rule
- Authentication errors during pull
- Timeout errors

**Diagnostic Steps:**

```bash
# 1. Verify cache rule exists
az acr cache show --registry MyRegistry --name MyRule

# 2. Check credential set health
az acr credential-set show --registry MyRegistry --name MyCreds

# 3. Test ACR login
az acr login --name MyRegistry

# 4. Test direct pull
docker pull myregistry.azurecr.io/target-repo:tag
```

**Cause: Key Vault Access Not Propagated**

RBAC changes can take up to 10 minutes to propagate.

**Solution:** Wait and retry
```bash
# Wait a few minutes, then retry
sleep 300
docker pull myregistry.azurecr.io/target-repo:tag
```

**Cause: Upstream Registry Rate Limiting**

Docker Hub limits unauthenticated pulls.

**Solution:** Ensure credentials are configured
```bash
# Verify credentials are attached to rule
az acr cache show --registry MyRegistry --name MyRule \
  --query 'credentialSetResourceId'

# If empty, update with credentials
az acr cache update --registry MyRegistry --name MyRule \
  --cred-set MyCreds
```

**Cause: Network Issues**

Firewall blocking access to upstream registry.

**Solution:** Verify network connectivity
```bash
# Test from Azure Cloud Shell or same network
curl -I https://registry-1.docker.io/v2/
curl -I https://mcr.microsoft.com/v2/
```

### 4. Cached Images Not Updating

**Symptoms:**
- Old image version returned despite upstream updates
- `latest` tag doesn't reflect current upstream state

**Explanation:**

Artifact cache does **NOT** automatically sync when upstream changes. This is by design.

**Solutions:**

#### Force Re-pull Specific Tag
```bash
# Delete local cache and re-pull
az acr repository delete --name MyRegistry --repository my-image --tag latest --yes
docker pull myregistry.azurecr.io/my-image:latest
```

#### Use Specific Version Tags
Instead of `:latest`, use explicit version tags for reproducibility:
```bash
docker pull myregistry.azurecr.io/nginx:1.25.3
```

#### Implement Scheduled Refresh
Create an automation to periodically pull images:
```bash
#!/bin/bash
# Scheduled script to refresh cached images
IMAGES=("nginx:latest" "redis:latest" "node:20")
for img in "${IMAGES[@]}"; do
  docker pull myregistry.azurecr.io/$img
done
```

### 5. Wildcard Rule Issues

**Symptoms:**
- Wildcard rule not matching expected paths
- Wrong images being pulled
- Unable to create wildcard rules

**Diagnostic Steps:**

```bash
# List all cache rules to understand current mappings
az acr cache list --registry MyRegistry \
  --query "[].{name:name,source:sourceRepository,target:targetRepository}" \
  --output table
```

**Issue: Wildcard Not Matching**

Verify the wildcard pattern matches the upstream structure:

| Pattern | Matches | Does NOT Match |
|---------|---------|----------------|
| `mcr.microsoft.com/*` | `mcr.microsoft.com/dotnet/sdk` | - |
| `mcr.microsoft.com/dotnet/*` | `mcr.microsoft.com/dotnet/sdk` | `mcr.microsoft.com/azure-cli` |
| `docker.io/library/*` | `docker.io/library/nginx` | `docker.io/bitnami/nginx` |

**Issue: Static Rule Override Not Working**

Ensure static rule is more specific than wildcard:
```bash
# This works - static overrides wildcard
Wildcard: contoso.azurecr.io/* => mcr.microsoft.com/*
Static:   contoso.azurecr.io/nginx => docker.io/library/nginx

# Static rule will be used for nginx, wildcard for everything else
```

### 6. Supported Upstream Registry Issues

**Symptoms:**
- Error about unsupported upstream
- Feature not available in portal

**Supported Upstreams Reference:**

| Registry | Portal | CLI | Auth Required |
|----------|--------|-----|---------------|
| Docker Hub | Yes | Yes | Yes (always) |
| MCR | Yes | Yes | No |
| GitHub (ghcr.io) | Yes | Yes | Optional |
| Quay | Yes | Yes | Optional |
| AWS ECR Public | Yes | Yes | No |
| registry.k8s.io | No | Yes | Optional |
| Google Artifact Registry | No | Yes | Yes |
| Legacy GCR | No | Yes | Optional |

**Solution for CLI-only registries:**
```bash
# Use Azure CLI instead of portal
az acr cache create \
  --registry MyRegistry \
  --name k8s-pause \
  --source-repo registry.k8s.io/pause \
  --target-repo k8s/pause
```

## Error Messages Reference

### Authentication Errors

| Error Message | Cause | Solution |
|---------------|-------|----------|
| "unauthorized: authentication required" | Missing or invalid credentials | Configure credential set |
| "Key vault secret not found" | Invalid secret URI | Verify Key Vault secret names |
| "Access denied to Key Vault" | Missing RBAC/policy | Grant Key Vault Secrets User role |

### Cache Rule Errors

| Error Message | Cause | Solution |
|---------------|-------|----------|
| "Cache rule limit exceeded" | >1000 rules | Delete unused rules |
| "Target repository already exists" | Repo pre-exists in ACR | Use different namespace |
| "Conflicting wildcard rule" | Overlapping wildcard | Delete conflict or use static |

### Pull Errors

| Error Message | Cause | Solution |
|---------------|-------|----------|
| "manifest unknown" | Image not in upstream | Verify source repository path |
| "rate limit exceeded" | Docker Hub throttling | Add credentials |
| "dial tcp: lookup failed" | Network/DNS issue | Check network connectivity |

## Diagnostic Commands

### Check Overall Cache Status
```bash
# List all cache rules and their status
az acr cache list --registry MyRegistry --output table

# List all credential sets
az acr credential-set list --registry MyRegistry --output table
```

### Check Specific Rule
```bash
az acr cache show --registry MyRegistry --name MyRule
```

### Check Credential Set
```bash
az acr credential-set show --registry MyRegistry --name MyCreds
```

### Check Cached Repositories
```bash
az acr repository list --name MyRegistry --output table
```

### Check Specific Cached Image Tags
```bash
az acr repository show-tags --name MyRegistry --repository my-cached-image
```

### View ACR Activity Logs
```bash
# Check recent operations
az monitor activity-log list \
  --resource-group MyResourceGroup \
  --resource-type Microsoft.ContainerRegistry/registries \
  --query "[?resourceId contains 'MyRegistry'].{time:eventTimestamp,operation:operationName.localizedValue,status:status.localizedValue}" \
  --output table
```

## Best Practices to Avoid Issues

1. **Always test credentials** before creating cache rules
2. **Use access tokens** instead of passwords when available
3. **Grant minimum required permissions** to Key Vault
4. **Document your cache rule strategy** for team reference
5. **Monitor credential expiration** and rotate proactively
6. **Use specific image tags** for production workloads
7. **Implement retry logic** in CI/CD pipelines
8. **Plan namespace structure** before creating rules
9. **Test wildcard patterns** with dry-run pulls

## Getting Help

If issues persist after following this guide:

1. Check [Azure Service Health](https://status.azure.com/) for outages
2. Review [ACR documentation](https://learn.microsoft.com/azure/container-registry/)
3. Search [Microsoft Q&A](https://learn.microsoft.com/answers/tags/133/azure-container-registry)
4. Open [Azure Support ticket](https://azure.microsoft.com/support/create-ticket/)

## References

- [Troubleshoot Artifact Cache](https://learn.microsoft.com/azure/container-registry/troubleshoot-artifact-cache)
- [Artifact Cache Overview](https://learn.microsoft.com/azure/container-registry/artifact-cache-overview)
- [Wildcard Support](https://learn.microsoft.com/azure/container-registry/wildcards-artifact-cache)
- [Azure Key Vault Troubleshooting](https://learn.microsoft.com/azure/key-vault/general/troubleshooting-access-issues)

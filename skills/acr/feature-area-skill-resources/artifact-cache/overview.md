# ACR Artifact Cache Skill

This skill provides comprehensive knowledge about Azure Container Registry artifact cache (pull-through cache).

## When to Use This Skill

Use this skill when answering questions about:
- Caching upstream registry images
- Cache rules configuration
- Docker Hub rate limits
- Pull-through cache

## Overview

Artifact cache provides pull-through caching for upstream registries, reducing rate limits and improving performance.

**Availability:** All SKUs (Basic, Standard, Premium)

## Supported Upstream Registries

| Registry | Auth Required |
|----------|--------------|
| Docker Hub | ✅ Always |
| MCR | ❌ |
| GitHub Container Registry | ✅ (private) |
| Amazon ECR Public | ❌ |
| Quay.io | ✅ (private) |
| Google Artifact Registry | ✅ |
| K8s Registry | ❌ |

## Quick Setup

### Without Credentials (MCR)
```bash
az acr cache create \
  --registry myregistry \
  --name mcr-cache \
  --source-repo mcr.microsoft.com/dotnet/runtime \
  --target-repo dotnet/runtime
```

### With Credentials (Docker Hub)
```bash
# Create credential
az acr credential-set create \
  --registry myregistry \
  --name dockerhub-creds \
  --login-server docker.io \
  --username-secret-identifier https://myvault.vault.azure.net/secrets/dockerhub-user \
  --password-secret-identifier https://myvault.vault.azure.net/secrets/dockerhub-pass

# Create cache rule
az acr cache create \
  --registry myregistry \
  --name nginx-cache \
  --source-repo docker.io/library/nginx \
  --target-repo nginx \
  --cred-set dockerhub-creds
```

## Using Cached Images

```bash
# Pull through cache
docker pull myregistry.azurecr.io/nginx:latest
# Fetches from Docker Hub on first pull, cached after

# Subsequent pulls
docker pull myregistry.azurecr.io/nginx:latest
# Served from ACR cache
```

## Wildcard Rules

```bash
# Cache all Docker Hub official images
az acr cache create \
  --registry myregistry \
  --name dockerhub-official \
  --source-repo docker.io/library/* \
  --target-repo library/*

# Cache all from an org
az acr cache create \
  --registry myregistry \
  --name myorg-cache \
  --source-repo docker.io/myorg/* \
  --target-repo myorg/*
```

### Wildcard Rules
- `docker.io/library/*` → Matches all official images
- `docker.io/myorg/*` → Matches all under org
- Cannot use `*` at registry level

## Credential Management

### Create Key Vault Secrets
```bash
# Store Docker Hub credentials
az keyvault secret set \
  --vault-name myvault \
  --name dockerhub-user \
  --value "myusername"

az keyvault secret set \
  --vault-name myvault \
  --name dockerhub-pass \
  --value "mypassword"
```

### Grant ACR Access to Key Vault
```bash
# RBAC method
az role assignment create \
  --assignee $(az acr show --name myregistry --query identity.principalId -o tsv) \
  --scope $(az keyvault show --name myvault --query id -o tsv) \
  --role "Key Vault Secrets User"
```

## Cache Rule Management

```bash
# List cache rules
az acr cache list --registry myregistry -o table

# Show cache rule
az acr cache show --registry myregistry --name nginx-cache

# Update cache rule
az acr cache update \
  --registry myregistry \
  --name nginx-cache \
  --cred-set new-creds

# Delete cache rule
az acr cache delete --registry myregistry --name nginx-cache
```

## Credential Set Management

```bash
# List credential sets
az acr credential-set list --registry myregistry

# Check credential health
az acr credential-set show \
  --registry myregistry \
  --name dockerhub-creds \
  --query "{status:authStatus,login:loginServer}"
```

## Behavior

- **Pull-through only**: Images cached on first pull
- **No auto-sync**: New upstream tags not auto-pulled
- **TTL**: Cached images remain until deleted

## Limits

| Limit | Value |
|-------|-------|
| Cache rules per registry | 1,000 |
| Credential sets per registry | 100 |

## Troubleshooting

### Unhealthy Credentials
```bash
# Check status
az acr credential-set show --registry myregistry --name creds

# Rotate credentials
# Update secrets in Key Vault, then:
az acr credential-set update --registry myregistry --name creds
```

### Cache Miss
- First pull always goes to upstream
- Ensure cache rule matches source path

## Reference Documentation

- **Investigation reports**: `agent-investigation-reports/acr-features/artifact-cache/`
- **MS Learn docs**: `azure-management-docs/articles/container-registry/artifact-cache-overview.md`

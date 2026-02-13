# ACR CLI Commands Reference - Image Management

## Overview

This document provides a comprehensive reference for Azure CLI commands related to ACR image management operations.

## Authentication Commands

### Login to ACR
```bash
# Login using Azure identity
az acr login --name myregistry

# Login with explicit credentials
az acr login --name myregistry --username <username> --password <password>

# Get login server name
az acr show --name myregistry --query loginServer -o tsv
```

## Repository Commands

### List Repositories
```bash
# List all repositories
az acr repository list --name myregistry

# Output as table
az acr repository list --name myregistry -o table
```

### Show Repository Details
```bash
# Show repository attributes
az acr repository show \
    --name myregistry \
    --repository myrepo

# Show with JSON output
az acr repository show \
    --name myregistry \
    --repository myrepo \
    --output jsonc
```

### Delete Repository
```bash
# Delete entire repository (all tags and manifests)
az acr repository delete \
    --name myregistry \
    --repository myrepo

# Delete without confirmation
az acr repository delete \
    --name myregistry \
    --repository myrepo \
    --yes
```

### Update Repository Attributes
```bash
# Lock repository (prevent writes)
az acr repository update \
    --name myregistry \
    --repository myrepo \
    --write-enabled false

# Protect from deletion
az acr repository update \
    --name myregistry \
    --repository myrepo \
    --delete-enabled false

# Full lock (immutable)
az acr repository update \
    --name myregistry \
    --repository myrepo \
    --write-enabled false \
    --delete-enabled false

# Unlock
az acr repository update \
    --name myregistry \
    --repository myrepo \
    --write-enabled true \
    --delete-enabled true
```

## Image Commands

### Delete Image
```bash
# Delete by tag
az acr repository delete \
    --name myregistry \
    --image myrepo:tag

# Delete by digest
az acr repository delete \
    --name myregistry \
    --image myrepo@sha256:abc123...

# Force delete without confirmation
az acr repository delete \
    --name myregistry \
    --image myrepo:tag \
    --yes
```

### Show Image Details
```bash
# Show image attributes
az acr repository show \
    --name myregistry \
    --image myrepo:tag

# Show with detailed output
az acr repository show \
    --name myregistry \
    --image myrepo:tag \
    --output jsonc
```

### Update Image Attributes
```bash
# Lock image
az acr repository update \
    --name myregistry \
    --image myrepo:tag \
    --write-enabled false

# Protect from deletion
az acr repository update \
    --name myregistry \
    --image myrepo:tag \
    --delete-enabled false

# Quarantine (block pulls)
az acr repository update \
    --name myregistry \
    --image myrepo:tag \
    --read-enabled false
```

### Untag Image
```bash
# Remove tag (keeps manifest)
az acr repository untag \
    --name myregistry \
    --image myrepo:tag
```

## Manifest Commands

### List Manifests
```bash
# List all manifests in repository
az acr manifest list-metadata \
    --name myrepo \
    --registry myregistry

# Order by time (ascending)
az acr manifest list-metadata \
    --name myrepo \
    --registry myregistry \
    --orderby time_asc

# Order by time (descending)
az acr manifest list-metadata \
    --name myrepo \
    --registry myregistry \
    --orderby time_desc

# Output as table
az acr manifest list-metadata \
    --name myrepo \
    --registry myregistry \
    -o table
```

### Show Manifest Details
```bash
# Show manifest by tag
az acr manifest show-metadata \
    --registry myregistry \
    --name myrepo:tag

# Show manifest by digest
az acr manifest show-metadata \
    --registry myregistry \
    --name myrepo@sha256:abc123...
```

### Query Manifests
```bash
# Get manifests older than date
az acr manifest list-metadata \
    --name myrepo \
    --registry myregistry \
    --orderby time_asc \
    --query "[?lastUpdateTime < '2023-04-05'].[digest, lastUpdateTime]" \
    -o tsv

# Get untagged manifests
az acr manifest list-metadata \
    --name myrepo \
    --registry myregistry \
    --query "[?tags==null].digest" \
    -o tsv

# Get specific tag's digest
az acr manifest show-metadata \
    --registry myregistry \
    --name myrepo:tag \
    --query digest \
    -o tsv
```

## Import Commands

### Import from Docker Hub
```bash
# Import official image
az acr import \
    --name myregistry \
    --source docker.io/library/nginx:latest \
    --image nginx:latest

# Import with credentials
az acr import \
    --name myregistry \
    --source docker.io/myorg/myimage:tag \
    --image myimage:tag \
    --username <docker-hub-user> \
    --password <docker-hub-token>
```

### Import from MCR
```bash
az acr import \
    --name myregistry \
    --source mcr.microsoft.com/dotnet/sdk:6.0 \
    --image dotnet/sdk:6.0
```

### Import from Another ACR
```bash
# Same tenant (using resource ID for private registries)
az acr import \
    --name myregistry \
    --source myimage:tag \
    --image myimage:tag \
    --registry /subscriptions/<sub-id>/resourceGroups/<rg>/providers/Microsoft.ContainerRegistry/registries/sourceregistry

# Cross-tenant with credentials
az acr import \
    --name myregistry \
    --source sourceregistry.azurecr.io/myimage:tag \
    --image myimage:tag \
    --username <sp-app-id> \
    --password <sp-password>
```

### Import by Digest
```bash
az acr import \
    --name myregistry \
    --source docker.io/library/nginx@sha256:abc123 \
    --repository nginx
```

## Retention Policy Commands

### Configure Retention Policy
```bash
# Enable retention (Premium only)
az acr config retention update \
    --registry myregistry \
    --status enabled \
    --days 30 \
    --type UntaggedManifests

# Show current policy
az acr config retention show \
    --registry myregistry

# Disable retention
az acr config retention update \
    --registry myregistry \
    --status disabled \
    --type UntaggedManifests
```

## Soft Delete Commands

### Configure Soft Delete
```bash
# Enable soft delete
az acr config soft-delete update \
    --registry myregistry \
    --days 7 \
    --status enabled

# Show current policy
az acr config soft-delete show \
    --registry myregistry

# Disable soft delete
az acr config soft-delete update \
    --registry myregistry \
    --status disabled
```

### Manage Soft-Deleted Artifacts
```bash
# List deleted repositories
az acr repository list-deleted \
    --name myregistry

# List deleted manifests
az acr manifest list-deleted \
    --registry myregistry \
    --name myrepo

# List deleted tags
az acr manifest list-deleted-tags \
    --registry myregistry \
    --name myrepo

# Restore deleted manifest
az acr manifest restore \
    --registry myregistry \
    --name myrepo:tag \
    --digest sha256:abc123

# Force restore (overwrites existing)
az acr manifest restore \
    --registry myregistry \
    --name myrepo:tag \
    --digest sha256:abc123 \
    --force
```

## ACR Purge Commands

### Run Purge On-Demand
```bash
# Purge old images
az acr run \
    --cmd "acr purge --filter 'myrepo:.*' --ago 7d" \
    --registry myregistry \
    /dev/null

# Purge with untagged manifests
az acr run \
    --cmd "acr purge --filter 'myrepo:.*' --ago 7d --untagged" \
    --registry myregistry \
    /dev/null

# Dry run (preview only)
az acr run \
    --cmd "acr purge --filter 'myrepo:.*' --ago 7d --dry-run" \
    --registry myregistry \
    /dev/null

# Keep N most recent
az acr run \
    --cmd "acr purge --filter 'myrepo:.*' --ago 0d --keep 5" \
    --registry myregistry \
    /dev/null
```

### Create Scheduled Purge Task
```bash
# Daily purge at midnight UTC
az acr task create \
    --name dailyPurge \
    --registry myregistry \
    --cmd "acr purge --filter 'myrepo:.*' --ago 7d --untagged" \
    --schedule "0 0 * * *" \
    --context /dev/null

# Weekly purge on Sunday
az acr task create \
    --name weeklyPurge \
    --registry myregistry \
    --cmd "acr purge --filter '.*:.*' --ago 30d" \
    --schedule "0 1 * * Sun" \
    --context /dev/null

# Show task details
az acr task show \
    --name dailyPurge \
    --registry myregistry
```

## Registry Usage Commands

### Show Storage Usage
```bash
# Show usage
az acr show-usage \
    --name myregistry \
    --resource-group myResourceGroup

# Table output
az acr show-usage \
    --name myregistry \
    --resource-group myResourceGroup \
    --output table
```

## PowerShell Equivalents

### Import Images
```powershell
# Import from Docker Hub
Import-AzContainerRegistryImage `
    -RegistryName myregistry `
    -ResourceGroupName myResourceGroup `
    -SourceRegistryUri docker.io `
    -SourceImage library/nginx:latest

# Import from another ACR
Import-AzContainerRegistryImage `
    -RegistryName myregistry `
    -ResourceGroupName myResourceGroup `
    -SourceRegistryResourceId '/subscriptions/.../registries/sourceregistry' `
    -SourceImage myimage:tag
```

### List Manifests
```powershell
Get-AzContainerRegistryManifest `
    -RepositoryName myrepo `
    -RegistryName myregistry
```

## Common Command Patterns

### Delete All Untagged Manifests
```bash
az acr manifest list-metadata \
    --name myrepo \
    --registry myregistry \
    --query "[?tags==null].digest" -o tsv \
| xargs -I% az acr repository delete \
    --name myregistry \
    --image myrepo@% --yes
```

### Lock All Images in Repository
```bash
for tag in $(az acr repository show-tags --name myregistry --repository myrepo -o tsv); do
    az acr repository update \
        --name myregistry \
        --image myrepo:$tag \
        --write-enabled false
done
```

### List All Locked Images
```bash
az acr manifest list-metadata \
    --name myrepo \
    --registry myregistry \
    --query "[?changeableAttributes.writeEnabled==\`false\`]" \
    -o table
```

## Source Documentation

- `/submodules/azure-management-docs/articles/container-registry/container-registry-delete.md`
- `/submodules/azure-management-docs/articles/container-registry/container-registry-import-images.md`
- `/submodules/azure-management-docs/articles/container-registry/container-registry-image-lock.md`
- `/submodules/azure-management-docs/articles/container-registry/container-registry-auto-purge.md`
- `/submodules/azure-management-docs/articles/container-registry/container-registry-retention-policy.md`
- `/submodules/azure-management-docs/articles/container-registry/container-registry-soft-delete-policy.md`

# ACR Image Management Skill

This skill provides comprehensive knowledge about managing images in Azure Container Registry.

## When to Use This Skill

Use this skill when answering questions about:
- Importing images
- Deleting images/repositories
- Image locking (immutability)
- Tagging best practices
- Multi-architecture images

## Image Import

Import images without Docker:

```bash
# Import from Docker Hub
az acr import \
  --name myregistry \
  --source docker.io/library/nginx:latest \
  --image nginx:latest

# Import from another ACR
az acr import \
  --name myregistry \
  --source sourceregistry.azurecr.io/myapp:v1 \
  --image myapp:v1

# Import from MCR
az acr import \
  --name myregistry \
  --source mcr.microsoft.com/dotnet/runtime:6.0 \
  --image dotnet/runtime:6.0

# Import with credentials (Docker Hub)
az acr import \
  --name myregistry \
  --source docker.io/myorg/myimage:v1 \
  --image myimage:v1 \
  --username <docker-hub-user> \
  --password <docker-hub-password>
```

## Image Delete

```bash
# Delete by tag
az acr repository delete \
  --name myregistry \
  --image myapp:v1

# Delete by digest
az acr repository delete \
  --name myregistry \
  --image myapp@sha256:abc123...

# Delete entire repository
az acr repository delete \
  --name myregistry \
  --repository myapp

# Untag only (keep manifest)
az acr repository untag \
  --name myregistry \
  --image myapp:v1
```

## Image Locking

Prevent modifications to images:

```bash
# Lock by tag
az acr repository update \
  --name myregistry \
  --image myapp:v1 \
  --write-enabled false \
  --delete-enabled false

# Lock entire repository
az acr repository update \
  --name myregistry \
  --repository myapp \
  --write-enabled false \
  --delete-enabled false

# Unlock
az acr repository update \
  --name myregistry \
  --image myapp:v1 \
  --write-enabled true \
  --delete-enabled true
```

### Lock Attributes
| Attribute | Effect |
|-----------|--------|
| `write-enabled: false` | Prevent push/update |
| `delete-enabled: false` | Prevent deletion |
| `read-enabled: false` | Prevent pull |
| `list-enabled: false` | Hide from catalog |

## Tagging Best Practices

### Stable Tags (for humans)
```
myapp:v1
myapp:latest
myapp:production
```

### Unique Tags (for automation)
```
myapp:v1.2.3-abc123
myapp:20240115-sha-abc123
myapp:build-1234
```

### Semantic Versioning
```
myapp:1.0.0
myapp:1.0
myapp:1
```

## Multi-Architecture Images

### Import Multi-Arch
```bash
az acr import \
  --name myregistry \
  --source mcr.microsoft.com/dotnet/runtime:6.0 \
  --image dotnet/runtime:6.0
# Imports all platforms
```

### Build Multi-Arch with Buildx
```bash
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  --push \
  -t myregistry.azurecr.io/myapp:v1 .
```

### Create Manifest List
```bash
docker manifest create myregistry.azurecr.io/myapp:v1 \
  myregistry.azurecr.io/myapp:v1-amd64 \
  myregistry.azurecr.io/myapp:v1-arm64

docker manifest push myregistry.azurecr.io/myapp:v1
```

## ACR Purge (Automated Cleanup)

```bash
# Purge images older than 7 days
az acr run \
  --cmd "acr purge --filter 'myapp:.*' --ago 7d --untagged" \
  --registry myregistry \
  /dev/null

# Schedule purge
az acr task create \
  --name purge-old-images \
  --registry myregistry \
  --cmd "acr purge --filter 'myapp:.*' --ago 30d" \
  --context /dev/null \
  --schedule "0 0 * * *"
```

## Repository Listing

```bash
# List repositories
az acr repository list --name myregistry -o table

# List tags
az acr repository show-tags --name myregistry --repository myapp -o table

# Show manifest details
az acr manifest list-metadata \
  --registry myregistry \
  --name myapp
```

## Reference Documentation

- **Investigation reports**: `agent-investigation-reports/acr-features/image-management/`
- **MS Learn docs**: `azure-management-docs/articles/container-registry/container-registry-import-images.md`

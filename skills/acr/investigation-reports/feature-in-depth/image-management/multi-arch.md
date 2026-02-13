# ACR Multi-Architecture Images - Comprehensive Guide

## Overview

Multi-architecture (multi-arch) images allow you to store container images for different CPU architectures and operating systems under a single tag. When a client pulls the image, Docker automatically selects the appropriate variant.

## Key Concepts

### Manifest vs Manifest List

| Type | Description |
|------|-------------|
| **Manifest** | JSON file describing a single-architecture image's layers |
| **Manifest List** | Collection (index) of manifests for multiple architectures |

### How Multi-Arch Works

```
myimage:latest (manifest list)
├── linux/amd64 (manifest A)
├── linux/arm64 (manifest B)
└── windows/amd64 (manifest C)
```

When you `docker pull myimage:latest`:
- Docker detects your platform
- Fetches the manifest list
- Selects the appropriate architecture manifest
- Pulls only the layers for your platform

## Supported Platforms

Common platform combinations:
- `linux/amd64` - Standard x64 Linux
- `linux/arm64` - ARM64 Linux (e.g., AWS Graviton, Apple M1)
- `linux/arm/v7` - ARMv7 Linux (e.g., Raspberry Pi)
- `windows/amd64` - Windows x64

## Working with Multi-Arch Images

### View a Manifest List

```bash
docker manifest inspect mcr.microsoft.com/mcr/hello-world:latest
```

Example output:
```json
{
  "schemaVersion": 2,
  "mediaType": "application/vnd.docker.distribution.manifest.list.v2+json",
  "manifests": [
    {
      "mediaType": "application/vnd.docker.distribution.manifest.v2+json",
      "size": 524,
      "digest": "sha256:83c7f9c92844...",
      "platform": {
        "architecture": "amd64",
        "os": "linux"
      }
    },
    {
      "mediaType": "application/vnd.docker.distribution.manifest.v2+json",
      "size": 525,
      "digest": "sha256:873612c5503f...",
      "platform": {
        "architecture": "arm64",
        "os": "linux"
      }
    },
    {
      "mediaType": "application/vnd.docker.distribution.manifest.v2+json",
      "size": 1124,
      "digest": "sha256:b791ad98d505...",
      "platform": {
        "architecture": "amd64",
        "os": "windows",
        "os.version": "10.0.17763.1697"
      }
    }
  ]
}
```

### View in ACR
```bash
az acr manifest list-metadata \
  --name hello-world \
  --registry myregistry
```

## Importing Multi-Arch Images

Import from public registries automatically includes all architectures:

```bash
# Import multi-arch image from Docker Hub
az acr import \
  --name myregistry \
  --source docker.io/library/hello-world:latest \
  --image hello-world:latest
```

**ACR automatically imports all architecture variants** specified in the manifest list.

## Creating Multi-Arch Images

### Method 1: Docker Manifest (Manual)

#### Step 1: Build and Push Architecture-Specific Images
```bash
# Build for amd64
docker build -t myregistry.azurecr.io/myimage:amd64 .
docker push myregistry.azurecr.io/myimage:amd64

# Build for arm64 (on ARM machine or with cross-compilation)
docker build -t myregistry.azurecr.io/myimage:arm64 .
docker push myregistry.azurecr.io/myimage:arm64
```

#### Step 2: Create Manifest List
```bash
docker manifest create myregistry.azurecr.io/myimage:latest \
  myregistry.azurecr.io/myimage:amd64 \
  myregistry.azurecr.io/myimage:arm64
```

#### Step 3: Push Manifest List
```bash
docker manifest push myregistry.azurecr.io/myimage:latest
```

#### Step 4: Verify
```bash
docker manifest inspect myregistry.azurecr.io/myimage:latest
```

### Method 2: Docker Buildx (Cross-Platform Builds)

Buildx enables building for multiple platforms from a single machine.

```bash
# Create builder instance
docker buildx create --name mybuilder --use

# Build and push multi-arch
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  -t myregistry.azurecr.io/myimage:latest \
  --push \
  .
```

### Method 3: ACR Tasks (Multi-Step Task)

Use ACR Tasks for automated multi-arch builds:

```yaml
# acr-task.yaml
version: v1.1.0

steps:
# Build for amd64
- build: -t {{.Run.Registry}}/myimage:{{.Run.ID}}-amd64 -f Dockerfile.amd64 .

# Build for arm64
- build: -t {{.Run.Registry}}/myimage:{{.Run.ID}}-arm64 -f Dockerfile.arm64 .

# Push architecture-specific images
- push:
    - {{.Run.Registry}}/myimage:{{.Run.ID}}-amd64
    - {{.Run.Registry}}/myimage:{{.Run.ID}}-arm64

# Create manifest list
- cmd: >
    docker manifest create
    {{.Run.Registry}}/myimage:latest
    {{.Run.Registry}}/myimage:{{.Run.ID}}-amd64
    {{.Run.Registry}}/myimage:{{.Run.ID}}-arm64

# Push manifest list
- cmd: docker manifest push --purge {{.Run.Registry}}/myimage:latest

# Verify
- cmd: docker manifest inspect {{.Run.Registry}}/myimage:latest
```

Run the task:
```bash
az acr run \
  --registry myregistry \
  --file acr-task.yaml \
  .
```

## Using Buildx with ACR Tasks Agent Pool

For cross-platform builds using ACR's infrastructure:

```yaml
version: v1.1.0

steps:
- cmd: |
    docker buildx create --name multiarch --use
    docker buildx build \
      --platform linux/amd64,linux/arm64 \
      -t {{.Run.Registry}}/myimage:{{.Run.ID}} \
      --push \
      .
```

## Working with Multi-Arch in Kubernetes

Kubernetes automatically selects the correct architecture:

```yaml
apiVersion: apps/v1
kind: Deployment
spec:
  template:
    spec:
      containers:
      - name: myapp
        image: myregistry.azurecr.io/myimage:latest  # Multi-arch tag
        # Kubernetes selects correct architecture per node
```

## Best Practices

### 1. Use Consistent Tagging
```
myimage:latest        # Multi-arch manifest list
myimage:v1.0-amd64    # Architecture-specific (optional)
myimage:v1.0-arm64    # Architecture-specific (optional)
```

### 2. Include OS Version for Windows
Windows images should include OS version:
```json
{
  "platform": {
    "architecture": "amd64",
    "os": "windows",
    "os.version": "10.0.17763.1697"
  }
}
```

### 3. Test on Target Platforms
Always verify images work on each target platform before release.

### 4. Use Buildx for Development
Buildx simplifies cross-platform development:
```bash
# Quick multi-arch build and push
docker buildx build --platform linux/amd64,linux/arm64 -t myimage:latest --push .
```

### 5. Consider CI/CD Pipeline
Automate multi-arch builds in your pipeline:

```yaml
# Azure DevOps example
- task: DockerInstaller@0
  inputs:
    dockerVersion: '20.10.0'

- script: |
    docker buildx create --use
    docker buildx build \
      --platform linux/amd64,linux/arm64 \
      -t $(ACR_NAME).azurecr.io/$(IMAGE_NAME):$(Build.BuildId) \
      --push \
      .
  displayName: 'Build multi-arch image'
```

## Troubleshooting

### Image Not Found for Platform
```
no matching manifest for linux/arm64 in the manifest list
```
**Solution**: Ensure the manifest list includes the required platform.

### Build Failures on Different Architectures
**Solution**: Use separate Dockerfiles or multi-stage builds with platform-specific base images.

### Buildx Not Available
```bash
# Enable experimental features
export DOCKER_CLI_EXPERIMENTAL=enabled

# Or create config
echo '{"experimental": "enabled"}' > ~/.docker/config.json
```

## Source Documentation

- `/submodules/azure-management-docs/articles/container-registry/push-multi-architecture-images.md`
- `/submodules/azure-management-docs/articles/container-registry/container-registry-import-images.md`
- `/submodules/azure-management-docs/articles/container-registry/container-registry-tasks-multi-step.md`
- `/submodules/acr/docs/tasks/buildx/README.md`

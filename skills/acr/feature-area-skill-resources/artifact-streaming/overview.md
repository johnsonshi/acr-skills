# ACR Artifact Streaming Skill

This skill provides comprehensive knowledge about Azure Container Registry artifact streaming.

## When to Use This Skill

Use this skill when answering questions about:
- Fast container startup
- Streaming images to AKS
- Large image optimization
- Lazy loading layers

## Overview

Artifact streaming reduces container startup time by streaming image layers on-demand instead of pulling the entire image upfront.

**Requirements:** Premium SKU (Preview)

## Benefits

| Metric | Improvement |
|--------|-------------|
| Pull time reduction | 15%+ for large images |
| Optimal for | Images > 30GB |
| Time to first byte | Dramatically reduced |

## Quick Setup

```bash
# Enable artifact streaming for repository
az acr artifact-streaming create \
  --registry myregistry \
  --repository myapp

# Check status
az acr artifact-streaming show \
  --registry myregistry \
  --repository myapp
```

## How It Works

1. **Image Push**: Standard push, streaming artifact generated
2. **AKS Pull**: Minimal data fetched for container start
3. **On-Demand**: Layers streamed as accessed
4. **Background**: Full image downloaded asynchronously

```
Traditional: ████████████████████ (full download) → Start
Streaming:   ██ (metadata) → Start → ░░░░░░░░░░░ (stream)
```

## AKS Integration

### Prerequisites
- Kubernetes 1.26+
- Ubuntu 20.04+ nodes
- Containerd runtime

### Enable on AKS
```bash
# Create AKS with artifact streaming
az aks create \
  --name myaks \
  --resource-group myRG \
  --enable-artifact-streaming
```

### Deploy Workload
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp
spec:
  containers:
  - name: myapp
    image: myregistry.azurecr.io/myapp:v1
```

## Repository Management

```bash
# List streaming-enabled repos
az acr artifact-streaming list --registry myregistry

# Update streaming config
az acr artifact-streaming update \
  --registry myregistry \
  --repository myapp \
  --enabled true

# Disable streaming
az acr artifact-streaming update \
  --registry myregistry \
  --repository myapp \
  --enabled false
```

## Compatibility

| Feature | Compatible |
|---------|------------|
| Geo-replication | ✅ |
| Private endpoints | ✅ |
| Cross-region | ✅ |
| CMK encryption | ❌ |
| Kubernetes regcred | ❌ |

## Supported Platforms

| Platform | Support |
|----------|---------|
| Linux AMD64 | ✅ |
| Linux ARM64 | ❌ |
| Windows | ❌ |

## Incompatible Configurations

- OS Guard enabled
- Pod Sandboxing enabled
- Confidential VMs

## Pricing

- No additional feature cost
- Premium SKU includes 500 GiB streaming storage
- Additional storage billed separately

## Troubleshooting

### Check Pod Events
```bash
kubectl describe pod myapp
```

Look for:
- `UpgradeIfStreamableDisabled` condition
- Image pull events

### Verify Streaming Artifact
```bash
az acr artifact-streaming show \
  --registry myregistry \
  --repository myapp
```

## Reference Documentation

- **Investigation reports**: `agent-investigation-reports/acr-features/artifact-streaming/`
- **MS Learn docs**: `azure-management-docs/articles/container-registry/container-registry-artifact-streaming.md`

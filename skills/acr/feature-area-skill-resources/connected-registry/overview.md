# ACR Connected Registry Skill

This skill provides comprehensive knowledge about Azure Container Registry connected registries for edge and on-premises scenarios.

## When to Use This Skill

Use this skill when answering questions about:
- Edge deployments
- On-premises registry sync
- Azure Arc integration
- IoT Edge scenarios
- Nested registries

## Overview

Connected registry enables on-premises or edge registry instances that sync with a cloud ACR.

**Requirements:** Premium SKU

## Operating Modes

| Mode | Pull | Push | Use Case |
|------|------|------|----------|
| **ReadOnly** | ✅ | ❌ | Production edge deployments |
| **ReadWrite** | ✅ | ✅ | Local development, testing |

## Quick Setup

### Create Connected Registry
```bash
# Create sync token
az acr token create \
  --name mysynctoken \
  --registry myregistry \
  --scope-map _repositories_pull

# Create connected registry
az acr connected-registry create \
  --registry myregistry \
  --name myconnected \
  --mode ReadOnly \
  --sync-token-name mysynctoken \
  --repository myapp
```

### Deploy on Arc-enabled Kubernetes
```bash
# Get connection string
CONNECTION=$(az acr connected-registry get-settings \
  --registry myregistry \
  --name myconnected \
  --parent-protocol https \
  --generate-password 1 \
  --query ACR_REGISTRY_CONNECTION_STRING -o tsv)

# Install Arc extension
az k8s-extension create \
  --cluster-name myarccluster \
  --resource-group myRG \
  --cluster-type connectedClusters \
  --extension-type Microsoft.ContainerRegistry.ConnectedRegistry \
  --name connectedregistry \
  --config connectionString=$CONNECTION
```

## Architecture

```
┌─────────────────────────────────────┐
│     Cloud ACR (myregistry)          │
│     Premium SKU                      │
└──────────────┬──────────────────────┘
               │ Sync (HTTPS)
               ▼
┌─────────────────────────────────────┐
│   Connected Registry (myconnected)  │
│   On-premises / Edge                │
└──────────────┬──────────────────────┘
               │ Pull
               ▼
┌─────────────────────────────────────┐
│   Container Runtime / IoT Edge      │
└─────────────────────────────────────┘
```

## Sync Configuration

```bash
# Configure repositories to sync
az acr connected-registry create \
  --registry myregistry \
  --name myconnected \
  --repository "myapp" "mybase/*" \
  --sync-schedule "0 */6 * * *" \
  --sync-window PT4H
```

### Sync Parameters
| Parameter | Description |
|-----------|-------------|
| `--sync-schedule` | Cron expression for sync |
| `--sync-window` | Maximum sync duration |
| `--repository` | Repos to sync (wildcards supported) |

## Client Access

### Create Client Token
```bash
az acr token create \
  --name myclienttoken \
  --registry myregistry \
  --scope-map _repositories_pull \
  --connected-registry myconnected
```

### Pull from Connected Registry
```bash
# Login
docker login myconnected:8080 -u myclienttoken

# Pull
docker pull myconnected:8080/myapp:v1
```

## Nested Hierarchies

Support for ISA-95 architectures:

```
Cloud ACR
    └── Level 4 Connected Registry
            └── Level 3 Connected Registry
                    └── Level 2 Connected Registry
```

```bash
# Create nested registry
az acr connected-registry create \
  --registry myregistry \
  --name level3 \
  --parent level4 \
  --mode ReadOnly
```

## Deployment Options

| Platform | Method |
|----------|--------|
| Azure Arc Kubernetes | Arc Extension |
| IoT Edge | IoT Edge Module |
| Generic Kubernetes | Helm Chart |
| Docker | Docker Compose |

## Management

```bash
# List connected registries
az acr connected-registry list --registry myregistry -o table

# Show status
az acr connected-registry show \
  --registry myregistry \
  --name myconnected

# Update configuration
az acr connected-registry update \
  --registry myregistry \
  --name myconnected \
  --sync-schedule "0 0 * * *"

# Delete
az acr connected-registry delete \
  --registry myregistry \
  --name myconnected
```

## Limits

| Limit | Value |
|-------|-------|
| Connected registries per cloud ACR | 50 |
| Clients per connected registry | 50 |
| Nesting depth | 5 levels |

## Billing

Starting August 1, 2025:
- Per connected registry charges apply
- Check current pricing documentation

## Reference Documentation

- **Investigation reports**: `agent-investigation-reports/acr-features/connected-registry/`
- **MS Learn docs**: `azure-management-docs/articles/container-registry/intro-connected-registry.md`

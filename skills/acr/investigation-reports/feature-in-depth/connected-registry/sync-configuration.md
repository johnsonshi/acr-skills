# ACR Connected Registry - Sync Configuration

## Overview

Synchronization is the core mechanism by which connected registries maintain consistency with their parent registries. This document covers how to configure and manage synchronization settings.

## Sync Mechanism

### How Synchronization Works

1. **Parent-to-Child Sync**: Container images and OCI artifacts sync from the parent registry to the connected registry
2. **Child-to-Parent Sync** (ReadWrite mode only): Pushed images sync from connected registry back to parent
3. **Token-Based Authentication**: Sync token authenticates the connected registry with its parent
4. **Message-Based Protocol**: Uses gateway messages for sync scheduling and updates

### Sync Token

Each connected registry has an automatically generated **sync token** that:
- Authenticates with the parent registry
- Has permissions to synchronize selected repositories
- Can read/write sync messages on the gateway
- Accesses two endpoints:
  - Login server: `<registry>.azurecr.io` (authentication)
  - Data endpoint: `<registry>.<region>.data.azurecr.io` (sync messages)

## Configuring Sync Schedule

### Default Behavior (Continuous Sync)

By default, connected registries sync **continuously** without a defined schedule:

```azurecli
# Create with default continuous sync
az acr connected-registry create \
  --registry myacrregistry \
  --name myconnectedregistry \
  --repository "hello-world"
```

### Scheduled Sync

For occasionally connected scenarios, configure a specific sync schedule and window:

```azurecli
az acr connected-registry update \
  --registry myacrregistry \
  --name myconnectedregistry \
  --sync-schedule "0 12 * * *" \
  --sync-window PT4H
```

### Schedule Parameters

| Parameter | Description | Format |
|-----------|-------------|--------|
| `--sync-schedule` | CRON expression defining when sync starts | Cron format |
| `--sync-window` | Duration of sync window | ISO 8601 duration |

### CRON Expression Examples

| Expression | Description |
|------------|-------------|
| `* * * * *` | Every minute |
| `0 * * * *` | Every hour |
| `0 12 * * *` | Daily at 12:00 PM UTC |
| `0 0 * * 0` | Weekly on Sunday at midnight |
| `0 6,18 * * *` | Twice daily at 6:00 AM and 6:00 PM |

### ISO 8601 Duration Examples

| Duration | Description |
|----------|-------------|
| `PT1H` | 1 hour |
| `PT4H` | 4 hours |
| `PT30M` | 30 minutes |
| `P1D` | 1 day |
| `PT12H` | 12 hours |

## Sync Limitations

### Continuous Sync Limits

| Parameter | Minimum | Maximum |
|-----------|---------|---------|
| `minMessageTtl` | 1 day | - |
| `maxMessageTtl` | - | 90 days |

### Scheduled Sync Limits

| Parameter | Minimum | Maximum |
|-----------|---------|---------|
| `minSyncWindow` | 1 hour | - |
| `maxSyncWindow` | - | 7 days |

## Repository Configuration

### Selecting Repositories to Sync

When creating a connected registry, specify which repositories to synchronize:

```azurecli
az acr connected-registry create \
  --registry myacrregistry \
  --name myconnectedregistry \
  --repository "hello-world" "app/frontend" "app/backend"
```

### Adding Repositories After Creation

```azurecli
az acr connected-registry repo \
  --registry myacrregistry \
  --name myconnectedregistry \
  --add "new-repo"
```

### Removing Repositories

```azurecli
az acr connected-registry repo \
  --registry myacrregistry \
  --name myconnectedregistry \
  --remove "old-repo"
```

### Important: acr/connected-registry Repository

For **nested scenarios** where lower layers have no Internet access, you must always include:

```azurecli
--repository "acr/connected-registry"
```

This repository contains the connected registry runtime image needed for deploying child registries.

## Sync Configuration Examples

### Example 1: Continuous Sync (Default)

Best for: Environments with always-on connectivity

```azurecli
az acr connected-registry create \
  --registry myacrregistry \
  --name myconnectedregistry \
  --repository "hello-world" "acr/connected-registry"
# No sync schedule specified = continuous sync
```

### Example 2: Daily Sync Window

Best for: Environments with limited bandwidth, sync during off-hours

```azurecli
# Create connected registry
az acr connected-registry create \
  --registry myacrregistry \
  --name myconnectedregistry \
  --repository "hello-world"

# Configure daily sync at midnight with 4-hour window
az acr connected-registry update \
  --registry myacrregistry \
  --name myconnectedregistry \
  --sync-schedule "0 0 * * *" \
  --sync-window PT4H
```

### Example 3: Hourly Sync

Best for: Environments needing frequent updates

```azurecli
az acr connected-registry update \
  --registry myacrregistry \
  --name myconnectedregistry \
  --sync-schedule "0 * * * *"
```

### Example 4: Every Minute Sync

Best for: Development/testing environments

```azurecli
az acr connected-registry update \
  --registry myacrregistry \
  --name myconnectedregistry \
  --sync-schedule "* * * * *"
```

### Example 5: Weekend Sync Only

Best for: Environments with weekend maintenance windows

```azurecli
az acr connected-registry update \
  --registry myacrregistry \
  --name myconnectedregistry \
  --sync-schedule "0 2 * * 0,6" \
  --sync-window PT12H
```

## Viewing Sync Configuration

### Portal Method

1. Navigate to your container registry in Azure Portal
2. Select **Connected registries** under Services
3. Click on your connected registry
4. View **Sync properties** section

### CLI Method

```azurecli
az acr connected-registry show \
  --registry myacrregistry \
  --name myconnectedregistry \
  --output table
```

Output shows:

```
NAME                 MODE       CONNECTION STATE  PARENT         LAST SYNC(UTC)       SYNC SCHEDULE  SYNC WINDOW
-------------------  ---------  ----------------  -------------  -------------------  -------------  -----------
myconnectedregistry  ReadWrite  online            myacrregistry  2026-01-09 12:00:00  0 0 * * *      00:00:00-23:59:59
```

## Sync Token Management

### Regenerating Sync Token Passwords

```azurecli
az acr connected-registry get-settings \
  --registry myacrregistry \
  --name myconnectedregistry \
  --generate-password 1
```

**Important**:
- Passwords are one-time and cannot be retrieved after generation
- New passwords must be deployed to the connected registry
- Follow standard practices for rotating passwords

### Sync Token Scope Map

The sync token's scope map determines which repositories can be synced:

```azurecli
# View sync token scope map
az acr token show \
  --registry myacrregistry \
  --name myconnectedregistry-sync-token \
  --query scopeMapId
```

### Disabling a Connected Registry

To temporarily disable sync, set the sync token to disabled:

```azurecli
# Note: You cannot delete a sync token while the connected registry exists
# Instead, set its status to disabled
az acr token update \
  --registry myacrregistry \
  --name myconnectedregistry-sync-token \
  --status disabled
```

## Connection String Format

The connection string contains sync configuration:

```
ConnectedRegistryName=myconnectedregistry;
SyncTokenName=myconnectedregistry-sync-token;
SyncTokenPassword=xxxxxxxxxxxxxxxx;
ParentGatewayEndpoint=myacrregistry.westus2.data.azurecr.io;
ParentEndpointProtocol=https
```

### Components

| Component | Description |
|-----------|-------------|
| `ConnectedRegistryName` | Name of the connected registry |
| `SyncTokenName` | Name of the sync token |
| `SyncTokenPassword` | Password for authentication |
| `ParentGatewayEndpoint` | Data endpoint for sync messages |
| `ParentEndpointProtocol` | Protocol (https or http) |

## Monitoring Sync Status

### Check Last Sync Time

```azurecli
az acr connected-registry show \
  --registry myacrregistry \
  --name myconnectedregistry \
  --query lastSync
```

### Check Connection State

Connection states indicate sync health:

| State | Description |
|-------|-------------|
| `Online` | Connected and syncing normally |
| `Offline` | Not connected to cloud |
| `Unhealthy` | Connected but reporting errors |

```azurecli
az acr connected-registry show \
  --registry myacrregistry \
  --name myconnectedregistry \
  --query connectionState
```

## Troubleshooting Sync Issues

### Repository Not Syncing

**Problem**: Images not appearing in connected registry

**Solution**: Ensure repository is configured for sync

```azurecli
az acr connected-registry repo \
  --registry myacrregistry \
  --name myconnectedregistry \
  --add "missing-repo"
```

### Sync Token Expired/Invalid

**Problem**: Sync failing with authentication errors

**Solution**: Regenerate sync token credentials and update deployment

```azurecli
az acr connected-registry get-settings \
  --registry myacrregistry \
  --name myconnectedregistry \
  --generate-password 1 \
  --yes
```

### Data Endpoint Not Enabled

**Problem**: Cannot establish sync connection

**Solution**: Enable dedicated data endpoint

```azurecli
az acr update --name myacrregistry --data-endpoint-enabled
```

## Sources

- `/Users/johnsonshi/repos/acr-skills/submodules/azure-management-docs/articles/container-registry/quickstart-create-connected-registry.md`
- `/Users/johnsonshi/repos/acr-skills/submodules/azure-management-docs/articles/container-registry/intro-connected-registry.md`
- `/Users/johnsonshi/repos/acr-skills/submodules/acr/docs/preview/connected-registry/quickstart-connected-registry-cli.md`
- `/Users/johnsonshi/repos/acr-skills/submodules/acr/docs/preview/connected-registry/overview-connected-registry-access.md`

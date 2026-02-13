# ACR Connected Registry - Troubleshooting Guide

## Overview

This guide covers common issues encountered when setting up, configuring, and operating connected registries, along with their solutions.

## Connection States

Before troubleshooting, understand the connection states:

| State | Description |
|-------|-------------|
| **Online** | Connected registry is connected and healthy |
| **Offline** | Connected registry is disconnected from cloud |
| **Unhealthy** | Connected but reporting critical errors |

Check connection state:

```azurecli
az acr connected-registry show \
  --registry MyRegistry \
  --name MyConnectedRegistry \
  --output table
```

## Checking Status Details

When connection state is `Unhealthy`, view detailed errors:

```azurecli
az acr connected-registry show \
  --registry MyRegistry \
  --name MyConnectedRegistry \
  --query statusDetails
```

Status details format:

```json
{
  "code": "Error code",
  "correlationId": "CorrelationId for debugging",
  "description": "Detailed error description",
  "timestamp": "When error occurred",
  "type": "Component reporting error"
}
```

## Common Issues and Solutions

### 1. Already Activated Error

**Error Message:**
```
Failed to activate the connected registry as it is already activated by another instance. Only one instance is supported at any time.
```

**Cause:** Another instance of the connected registry is already deployed.

**Solutions:**

**Option A: Deactivate Existing Instance**
```azurecli
az acr connected-registry deactivate \
  --registry myacrregistry \
  --name myconnectedregistry
```
Then redeploy the connected registry.

**Option B: Create New Connected Registry**

Create a new connected registry with a different name:
```azurecli
az acr connected-registry create \
  --registry myacrregistry \
  --name mynewconnectedregistry \
  --repository "hello-world"
```

---

### 2. HTTP/HTTPS Mismatch Error

**Error Message:**
```
Error response from daemon: Get https://<connected-registry>/v2/: http: server gave HTTP response to HTTPS client
```

**Cause:** Client is using HTTPS but connected registry is configured for HTTP only.

**Solution:** Configure Docker daemon for insecure registry (testing only):

Edit `/etc/docker/daemon.json`:
```json
{
  "insecure-registries": ["<IP_address>:<port>"]
}
```

Restart Docker:
```bash
sudo systemctl restart docker
```

**Note:** For production, configure the connected registry with TLS certificates instead.

---

### 3. Manifest Unknown Error

**Error Message:**
```
manifest for <connected-registry>/<repository>:<tag> not found: manifest unknown
```

**Cause:** The repository is not configured for synchronization.

**Solution:** Add the repository to the connected registry:

```azurecli
az acr connected-registry repo \
  --registry myacrregistry \
  --name myconnectedregistry \
  --add <repository-name>
```

Wait a few minutes for synchronization to complete.

---

### 4. Insufficient Scopes Error

**Error Message:**
```
pull access denied for <repository>, repository does not exist or may require 'docker login': denied: Insufficient scopes to perform the operation
```

**Cause:** Client token lacks required permissions.

**Solution:** Update the scope map for the client token:

1. Find the scope map:
```azurecli
az acr token show \
  --registry myacrregistry \
  --name <client-token-name> \
  --output tsv --query scopeMapId
```

2. Add permissions:
```azurecli
# For pull access
az acr scope-map update \
  --name <scope-map-name> \
  --registry myacrregistry \
  --add-repository <repository-name> content/read

# For push access (ReadWrite mode only)
az acr scope-map update \
  --name <scope-map-name> \
  --registry myacrregistry \
  --add-repository <repository-name> content/read content/write
```

---

### 5. Push Not Allowed Error

**Error Message:**
```
denied: This operation is not allowed on this registry.
```

**Cause:** Attempting to push to a ReadOnly mode connected registry.

**Solution:** Create a new connected registry in ReadWrite mode:

```azurecli
az acr connected-registry create \
  --registry myacrregistry \
  --name mynewregistry \
  --repository "app/hello-world" \
  --mode ReadWrite \
  --client-tokens <client-token-name>
```

**Note:** Mode cannot be changed after creation.

---

### 6. Disk Errors

#### Insufficient Permissions

**Status Detail:**
```json
{
  "code": "DiskError",
  "description": "Access to the path '/var/acr/data/registry/dummy.txt' is denied."
}
```

**Solution:** Update permissions on host storage directory:

```bash
# Ensure container user has read/write/execute access
chmod -R 755 /var/acr/data
chown -R <container-user> /var/acr/data
```

#### No Disk Space

**Status Detail:**
```json
{
  "code": "DiskError",
  "description": "No space left on device."
}
```

**Solutions:**

**Option A: Configure global log limits** (`/etc/docker/daemon.json`):
```json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  }
}
```

**Option B: Container-specific log limits** (add to docker run):
```bash
--log-driver json-file --log-opt max-size=10m --log-opt max-file=3
```

**Option C: Reduce log verbosity:**
```azurecli
az acr connected-registry update \
  --registry mycloudregistry \
  --name myconnectedregistry \
  --log-level Error
```

---

## Arc Extension Troubleshooting

### Check Extension Status

```bash
# Get extension status
az k8s-extension show \
  --name <extension-name> \
  --cluster-name <cluster-name> \
  --resource-group <resource-group> \
  --cluster-type connectedClusters

# Check pod status
kubectl get pod -n connected-registry

# Get events
kubectl get events -n connected-registry --sort-by='.lastTimestamp'
```

### Extension Installation Errors

#### "Name still in use"

**Solution:** Use a different extension name, or delete the existing extension first.

#### "Namespace being terminated"

**Solution:** Wait for previous uninstallation to complete:
```bash
az k8s-extension show --name <extension-name> # Check status
```

#### "Chart path not found"

**Solution:** Use a valid version or omit `--version` for latest:
```bash
az k8s-extension create ... # Without --version flag
```

### Extension Stuck in Running State

#### PVC Issues

Check PVC status:
```bash
kubectl get pvc -n connected-registry -o yaml connected-registry-pvc
```

If stuck in `pending`, check storage class:
```bash
kubectl get storageclass --all-namespaces
```

Recreate with correct storage class:
```bash
--config pvc.storageClassName="standard"
```

Or increase storage:
```bash
--config pvc.storageRequest="250Gi"
```

#### Bad Connection String

Check pod logs:
```bash
kubectl get pod -n connected-registry
kubectl logs -n connected-registry connected-registry-xxxxxxxxx-xxxxx
```

If you see `UNAUTHORIZED` error, regenerate connection string:
```bash
cat << EOF > protected-settings-extension.json
{
  "connectionString": "$(az acr connected-registry get-settings \
    --name myconnectedregistry \
    --registry myacrregistry \
    --parent-protocol https \
    --generate-password 1 \
    --query ACR_REGISTRY_CONNECTION_STRING --output tsv --yes)"
}
EOF

az k8s-extension update \
  --cluster-name <myarck8scluster> \
  --cluster-type connectedClusters \
  --name <myconnectedregistry> \
  -g <myresourcegroup> \
  --config-protected-file protected-settings-extension.json
```

### Extension Not Coming Online

Check for "ALREADY_ACTIVATED" error in logs and deactivate:

```azurecli
az acr connected-registry deactivate \
  --registry myacrregistry \
  --name myconnectedregistry
```

---

## Logging Configuration

### Enable Debug Logging

```azurecli
az acr connected-registry update \
  --registry mycloudregistry \
  --name myconnectedregistry \
  --log-level debug
```

### Log Levels

| Level | Description |
|-------|-------------|
| `Debug` | Detailed debugging information |
| `Information` | General operational info (default) |
| `Warning` | Potential issues |
| `Error` | Errors preventing operations |
| `None` | Disable logging |

### Azure CLI Verbosity

```bash
# More detail
az acr connected-registry show ... --verbose

# Full debug
az acr connected-registry show ... --debug
```

---

## Network Troubleshooting

### Verify Endpoint Connectivity

Connected registry needs access to:
1. Login server: `<registry>.azurecr.io`
2. Data endpoint: `<registry>.<region>.data.azurecr.io`

Test connectivity:
```bash
# Test login server
curl -v https://<registry>.azurecr.io/v2/

# Test data endpoint
curl -v https://<registry>.<region>.data.azurecr.io/
```

### Enable Dedicated Data Endpoint

```azurecli
az acr update --name myacrregistry --data-endpoint-enabled
```

---

## Token Troubleshooting

### Token Password Lost

Passwords cannot be retrieved. Generate new ones:

```azurecli
# For client tokens
az acr token credential generate \
  --name <token-name> \
  --registry myacrregistry \
  --password1

# For sync tokens (regenerates connection string)
az acr connected-registry get-settings \
  --registry myacrregistry \
  --name myconnectedregistry \
  --generate-password 1
```

### Token Disabled/Deleted

**Warning:** Do not delete sync tokens as this breaks the connected registry permanently.

To disable temporarily:
```azurecli
az acr token update \
  --registry myacrregistry \
  --name myconnectedregistry-sync-token \
  --status disabled
```

---

## Cleanup and Reset

### Full Cleanup Procedure

1. Delete the extension:
```azurecli
az k8s-extension delete \
  --name myconnectedregistry \
  --cluster-name myarccluster \
  --resource-group myresourcegroup \
  --cluster-type connectedClusters
```

2. Delete the connected registry resource:
```azurecli
az acr connected-registry delete \
  --registry myacrregistry \
  --name myconnectedregistry
```

3. Optionally clean up tokens and scope maps:
```azurecli
az acr token delete --registry myacrregistry --name <token-name>
az acr scope-map delete --registry myacrregistry --name <scope-map-name>
```

---

## Getting Help

### View Release Notes

Check for known issues in release notes for your version.

### Create Support Issue

For Azure support:
1. Azure Portal > Help + Support > New Support Request
2. Include connection state, status details, and logs

### Community Support

GitHub Issues: https://github.com/Azure/acr/issues

Include:
- Error messages
- CLI commands run
- Connected registry mode and configuration
- Deployment type (Arc, IoT Edge, etc.)

## Sources

- `/Users/johnsonshi/repos/acr-skills/submodules/azure-management-docs/articles/container-registry/troubleshoot-connected-registry-arc.md`
- `/Users/johnsonshi/repos/acr-skills/submodules/acr/docs/preview/connected-registry/troubleshooting.md`
- `/Users/johnsonshi/repos/acr-skills/submodules/acr/docs/preview/connected-registry/connected-registry-error-codes.md`
- `/Users/johnsonshi/repos/acr-skills/submodules/acr/docs/preview/connected-registry/README.md`

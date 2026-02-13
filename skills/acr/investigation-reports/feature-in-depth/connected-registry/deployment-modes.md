# ACR Connected Registry - Deployment Modes

## Overview

Connected Registry supports two primary deployment modes that determine the functionality available to clients:

1. **ReadOnly Mode** (formerly called "Mirror Mode") - Pull-only access
2. **ReadWrite Mode** (formerly called "Registry Mode") - Full pull and push access

## Deployment Mode Comparison

| Feature | ReadOnly Mode | ReadWrite Mode |
|---------|--------------|----------------|
| Image Pull | Yes | Yes |
| Image Push | No | Yes |
| Push to Cloud Sync | N/A | Yes (auto-synced) |
| Recommended For | Production, edge devices | Development, local workflows |
| Default Mode | Yes (secure-by-default) | No |
| Repository Delete | Not supported | Supported |

## ReadOnly Mode (Default)

### Description

ReadOnly mode is the **default and secure-by-default** configuration for connected registries. In this mode, clients can only **pull (read) artifacts** from the connected registry.

### Use Cases

- **Production environments** - Ensure images cannot be accidentally modified
- **Edge/IoT devices** - Devices only need to pull container images to operate
- **Lower layers in hierarchy** - When parent is ReadOnly, children must also be ReadOnly
- **Read-only access requirements** - When strict control over image sources is needed

### Configuration

```azurecli
# Create a connected registry in ReadOnly mode (explicit)
az acr connected-registry create \
  --registry myacrregistry \
  --name myconnectedregistry \
  --repository "hello-world" \
  --mode ReadOnly
```

### Behavior

- Clients can **pull** images using `docker pull`
- **Push operations are blocked** - Returns error "This operation is not allowed on this registry"
- Images **sync from parent to connected registry** (one-way)
- Client tokens can only have **content/read** permissions

## ReadWrite Mode

### Description

ReadWrite mode allows clients to **both pull and push artifacts** to the connected registry. Artifacts pushed to the connected registry are automatically synchronized back to the cloud registry.

### Use Cases

- **Local development environments** - Push images locally, sync to cloud
- **Build pipelines** - Build images on-premises and sync to central registry
- **Bidirectional sync requirements** - When images need to flow both ways
- **Parent registries in hierarchy** - Can serve ReadWrite or ReadOnly children

### Configuration

```azurecli
# Create a connected registry in ReadWrite mode
az acr connected-registry create \
  --registry myacrregistry \
  --name myconnectedregistry \
  --repository "hello-world" "acr/connected-registry" \
  --mode ReadWrite
```

### Behavior

- Clients can **pull and push** images
- Pushed images are **synchronized to parent registry**
- Client tokens can have **content/read and content/write** permissions
- Repository delete operations are supported

## Mode Hierarchy Rules

Connected registries form a hierarchy where mode compatibility must be maintained:

### Rule 1: ReadWrite Parent

A connected registry in **ReadWrite mode** can have children in either mode:

```
Cloud Registry
    │
    └── Connected Registry (ReadWrite)
            │
            ├── Child Registry (ReadWrite) ✓
            │
            └── Child Registry (ReadOnly) ✓
```

### Rule 2: ReadOnly Parent

A connected registry in **ReadOnly mode** can **only** have children in ReadOnly mode:

```
Cloud Registry
    │
    └── Connected Registry (ReadOnly)
            │
            ├── Child Registry (ReadWrite) ✗ NOT ALLOWED
            │
            └── Child Registry (ReadOnly) ✓
```

### Mode Compatibility Matrix

| Parent Mode | Child Mode | Allowed |
|-------------|------------|---------|
| Cloud Registry | ReadWrite | Yes |
| Cloud Registry | ReadOnly | Yes |
| ReadWrite | ReadWrite | Yes |
| ReadWrite | ReadOnly | Yes |
| ReadOnly | ReadWrite | **No** |
| ReadOnly | ReadOnly | Yes |

## Mode Cannot Be Changed After Creation

**Important**: Once a connected registry is created, its mode cannot be changed. If you need to switch from ReadOnly to ReadWrite:

1. **Create a new connected registry** in the desired mode
2. **Deploy the new connected registry** with appropriate configuration
3. **Update client tokens** to point to the new registry
4. **Optionally delete** the old connected registry

## Mode Selection Guidelines

### Choose ReadOnly When:

- You want **secure-by-default** configuration
- Clients only need to **consume (pull) images**
- You're deploying to **production or edge environments**
- You need to **prevent accidental image modifications**
- The connected registry is a **child of a ReadOnly parent**

### Choose ReadWrite When:

- You need **local development** capabilities
- Images must be **pushed locally and synced to cloud**
- You're implementing a **build pipeline on-premises**
- You need **bidirectional content flow**

## Examples

### Example 1: Production Deployment (ReadOnly)

```azurecli
# Create ReadOnly connected registry for production
az acr connected-registry create \
  --registry myacrregistry \
  --name production-edge \
  --repository "app/frontend" "app/backend" \
  --mode ReadOnly
```

### Example 2: Development Environment (ReadWrite)

```azurecli
# Create ReadWrite connected registry for development
az acr connected-registry create \
  --registry myacrregistry \
  --name dev-local \
  --repository "app/frontend" "app/backend" "dev/test-images" \
  --mode ReadWrite
```

### Example 3: Hierarchical Deployment

```azurecli
# Parent: ReadWrite for main site
az acr connected-registry create \
  --registry myacrregistry \
  --name site-main \
  --repository "hello-world" "acr/connected-registry" \
  --mode ReadWrite

# Child: ReadOnly for edge devices
az acr connected-registry create \
  --registry myacrregistry \
  --parent site-main \
  --name site-edge \
  --repository "hello-world" \
  --mode ReadOnly
```

## Verifying Mode

To check the mode of an existing connected registry:

```azurecli
az acr connected-registry show \
  --registry myacrregistry \
  --name myconnectedregistry \
  --output table
```

Output shows the MODE column:

```
NAME                  MODE       CONNECTION STATE    PARENT        LOGIN SERVER
--------------------  ---------  ------------------  ------------  --------------
myconnectedregistry   ReadOnly   Online              myacrregistry ...
```

## Error Messages Related to Mode

### Push to ReadOnly Registry

Error when attempting to push to a ReadOnly connected registry:

```
denied: This operation is not allowed on this registry.
```

**Solution**: Create a new connected registry in ReadWrite mode if push operations are required.

### Invalid Child Mode

Error when trying to create a ReadWrite child under a ReadOnly parent:

```
InvalidConnectedRegistryMode: The connected registry mode is not compatible with its parent.
```

**Solution**: Use ReadOnly mode for the child registry, or change the parent to ReadWrite mode (requires recreation).

## Sources

- `/Users/johnsonshi/repos/acr-skills/submodules/azure-management-docs/articles/container-registry/intro-connected-registry.md`
- `/Users/johnsonshi/repos/acr-skills/submodules/azure-management-docs/articles/container-registry/quickstart-create-connected-registry.md`
- `/Users/johnsonshi/repos/acr-skills/submodules/acr/docs/preview/connected-registry/troubleshooting.md`
- `/Users/johnsonshi/repos/acr-skills/submodules/acr/docs/preview/connected-registry/quickstart-connected-registry-cli.md`

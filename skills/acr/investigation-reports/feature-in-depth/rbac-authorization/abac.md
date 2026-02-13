# Azure Container Registry ABAC (Attribute-Based Access Control)

## Overview

Azure Attribute-Based Access Control (ABAC) extends Azure RBAC by allowing repository-specific conditions in role assignments. This enables granular permissions management at the repository level within Azure Container Registry.

## Key Concepts

### What is ABAC?

ABAC builds upon Azure RBAC by introducing **conditions** that evaluate attributes during access decisions:
- **Request attributes** - Properties of the access request (e.g., repository name)
- **Resource attributes** - Properties of the resource being accessed
- **Principal attributes** - Properties of the requesting identity

### Why Use ABAC?

1. **Repository-level permissions** - Scope access to specific repositories
2. **Wildcard/prefix matching** - Grant access to multiple repositories with common naming patterns
3. **Enhanced security** - Apply least privilege at the repository level
4. **Simplified management** - Fewer role assignments needed compared to using tokens for each repository

## Role Assignment Permissions Mode

To use ABAC, configure the registry's **Role assignment permissions mode**:

| Mode | ABAC Support | Applicable Roles |
|------|--------------|------------------|
| `RBAC Registry Permissions` | No | AcrPull, AcrPush, AcrDelete, AcrImageSigner |
| `RBAC Registry + ABAC Repository Permissions` | Yes | Container Registry Repository Reader/Writer/Contributor |

### Configure via Azure Portal

1. Navigate to registry in Azure portal
2. Go to **Settings** > **Properties**
3. Set **Role assignment permissions mode** to `RBAC Registry + ABAC Repository Permissions`
4. Click **Save**

### Configure via Azure CLI

```bash
# Create new registry with ABAC enabled
az acr create \
  --name <registry-name> \
  --resource-group <resource-group> \
  --sku <sku> \
  --role-assignment-mode 'rbac-abac' \
  --location <location>

# Update existing registry to enable ABAC
az acr update \
  --name <registry-name> \
  --resource-group <resource-group> \
  --role-assignment-mode 'rbac-abac'

# Check current mode
az acr show \
  --name <registry-name> \
  --resource-group <resource-group> \
  --query "roleAssignmentMode"
```

## Effects of Enabling ABAC

### On Existing Role Assignments

When enabling ABAC on an existing registry:

| Role | Effect in ABAC-enabled Registry |
|------|--------------------------------|
| `AcrPull` | **Not honored** - Use `Container Registry Repository Reader` instead |
| `AcrPush` | **Not honored** - Use `Container Registry Repository Writer` instead |
| `AcrDelete` | **Not honored** - Use `Container Registry Repository Contributor` instead |
| `Owner` | Control plane only - **No data plane permissions** |
| `Contributor` | Control plane only - **No data plane permissions** |
| `Reader` | Control plane only - **No data plane permissions** |

### On ACR Tasks, Quick Tasks, Quick Builds, Quick Runs

ACR Tasks lose default data plane access in ABAC-enabled registries:

**For ACR Tasks:**
- Use `--source-acr-auth-id [system]` to attach system-assigned identity
- Use `--source-acr-auth-id $resourceId` to attach user-assigned identity
- Grant the identity `Container Registry Repository Reader/Writer/Contributor` role
- Use `--source-acr-auth-id none` to disable source registry access

```bash
# Create task with identity for source registry access
az acr task create \
  --name <task-name> \
  --registry <registry-name> \
  --source-acr-auth-id [system] \
  ...

# Update existing task
az acr task update \
  --name <task-name> \
  --registry <registry-name> \
  --source-acr-auth-id <identity-resource-id>
```

**For Quick Tasks (az acr build/run):**
- Use `--source-acr-auth-id [caller]` to use caller's identity
- Caller must have appropriate ACR role assignment
- Use `--source-acr-auth-id none` to disable source registry access

```bash
# Quick build with caller identity
az acr build \
  --source-acr-auth-id [caller] \
  ...
```

> **Note**: The `--auth-mode` flag is deprecated for ABAC-enabled registries.

## ABAC-Enabled Built-in Roles

### Container Registry Repository Reader

- **Permissions**: Pull images, view tags, read metadata, view OCI referrers
- **ABAC Support**: Yes
- **Does NOT include**: Repository catalog listing

### Container Registry Repository Writer

- **Permissions**: Push/pull images, manage tags, manage OCI referrers, enable artifact streaming
- **ABAC Support**: Yes
- **Does NOT include**: Repository catalog listing, deletion

### Container Registry Repository Contributor

- **Permissions**: Push/pull/delete images, manage tags, delete OCI referrers, full artifact streaming
- **ABAC Support**: Yes
- **Does NOT include**: Repository catalog listing

### Container Registry Repository Catalog Lister

- **Permissions**: List all repositories via `_catalog` endpoint
- **ABAC Support**: **No** - Always grants full catalog listing
- **Does NOT include**: Any other permissions

## Configuring ABAC Conditions

### Condition Expression Syntax

ABAC conditions use a specific syntax:

```
(
 (
  !(ActionMatches{'<action-1>'})
  AND
  !(ActionMatches{'<action-2>'})
 )
 OR
 (
  @Request[Microsoft.ContainerRegistry/registries/repositories:name] <operator> '<value>'
 )
)
```

### Available Operators

| Operator | Description | Example |
|----------|-------------|---------|
| `StringEqualsIgnoreCase` | Exact match (case-insensitive) | `hello-world` |
| `StringStartsWithIgnoreCase` | Prefix/wildcard match | `backend/` |

### Example 1: Scope to Specific Repository

Grant access only to `hello-world` repository:

```
(
 (
  !(ActionMatches{'Microsoft.ContainerRegistry/registries/repositories/content/read'})
  AND
  !(ActionMatches{'Microsoft.ContainerRegistry/registries/repositories/metadata/read'})
 )
 OR
 (
  @Request[Microsoft.ContainerRegistry/registries/repositories:name] StringEqualsIgnoreCase 'hello-world'
 )
)
```

### Example 2: Scope to Repository Prefix (Wildcard)

Grant access to all repositories starting with `backend/`:

```
(
 (
  !(ActionMatches{'Microsoft.ContainerRegistry/registries/repositories/content/read'})
  AND
  !(ActionMatches{'Microsoft.ContainerRegistry/registries/repositories/metadata/read'})
 )
 OR
 (
  @Request[Microsoft.ContainerRegistry/registries/repositories:name] StringStartsWithIgnoreCase 'backend/'
 )
)
```

> **Important**: Include trailing slash `/` to avoid unintended matches. Without it, `backend` would match both `backend/nginx` and `backend-infra/k8s`.

### Example 3: Multiple Repository Prefixes

Grant access to repositories starting with `backend/` OR `frontend/js/`:

```
(
 (
  !(ActionMatches{'Microsoft.ContainerRegistry/registries/repositories/content/read'})
  AND
  !(ActionMatches{'Microsoft.ContainerRegistry/registries/repositories/metadata/read'})
 )
 OR
 (
  @Request[Microsoft.ContainerRegistry/registries/repositories:name] StringStartsWithIgnoreCase 'backend/'
  OR
  @Request[Microsoft.ContainerRegistry/registries/repositories:name] StringStartsWithIgnoreCase 'frontend/js/'
 )
)
```

## Configuring ABAC via Azure Portal

1. Navigate to registry > **Access control (IAM)**
2. Click **Add** > **Add role assignment**
3. Select an ABAC-enabled role (e.g., `Container Registry Repository Reader`)
4. In **Members** tab, select the identity
5. In **Conditions** tab:
   - Click **Add condition**
   - Select **Visual** editor type
   - Under **Condition #1**, select actions to include
   - Under **Build expression**, click **Add expression**:
     - **Attribute source**: `Request`
     - **Attribute**: `Repository name`
     - **Operator**: Choose `StringEqualsIgnoreCase` or `StringStartsWithIgnoreCase`
     - **Value**: Enter repository name or prefix
6. Click **Save** to save condition
7. Click **Review + assign** to complete

## Configuring ABAC via Azure CLI

```bash
# Store condition in variable
condition=$(cat <<'EOF' | tr -d '\n'
(
 (
  !(ActionMatches{'Microsoft.ContainerRegistry/registries/repositories/content/read'})
  AND
  !(ActionMatches{'Microsoft.ContainerRegistry/registries/repositories/metadata/read'})
 )
 OR
 (
  @Request[Microsoft.ContainerRegistry/registries/repositories:name] StringEqualsIgnoreCase 'hello-world'
 )
)
EOF
)

# Get registry scope
scope=$(az acr show --name <registry-name> --resource-group <rg> --query "id" -o tsv)

# Create role assignment with condition
az role assignment create \
  --role "Container Registry Repository Reader" \
  --scope "$scope" \
  --assignee "user@contoso.com" \
  --description "Read access to hello-world repository" \
  --condition "$condition" \
  --condition-version "2.0"
```

## Important Considerations

### ABAC-Enabled Roles Without Conditions

> **Warning**: If you assign an ABAC-enabled role **without** ABAC conditions, the role assignment grants permissions to **all repositories** in the registry. Always include conditions when you want repository-specific access.

### Catalog Listing

The `Container Registry Repository Catalog Lister` role does **not** support ABAC conditions. It always grants permissions to list all repositories. Assign this role only when full catalog access is required.

### Docker Content Trust

Signing images with Docker Content Trust (DCT) is **not supported** in ABAC-enabled registries. Use OCI referrers (Notary Project) instead.

## Best Practices

1. **Plan repository naming** - Use consistent prefixes for easier ABAC management
2. **Use prefixes with trailing slash** - Prevent unintended matches
3. **Combine roles as needed** - Add `Catalog Lister` only when repository listing is required
4. **Test conditions** - Verify access works as expected before production use
5. **Document role assignments** - Use descriptions to explain the purpose
6. **Review periodically** - Audit ABAC conditions for continued appropriateness

## Source Documentation

- MS Learn: `/submodules/azure-management-docs/articles/container-registry/container-registry-rbac-abac-repository-permissions.md`
- ACR Repo: `/submodules/acr/docs/preview/abac-repo-permissions/README.md`
- ACR Repo: `/submodules/acr/docs/blog/abac-repo-permissions.md`

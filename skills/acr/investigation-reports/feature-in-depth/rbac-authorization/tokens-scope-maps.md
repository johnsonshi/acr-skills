# Azure Container Registry Tokens and Scope Maps

## Overview

Azure Container Registry supports **non-Microsoft Entra token-based authentication** for managing repository-scoped permissions. This approach uses **tokens** and **scope maps** to provide fine-grained access control without requiring Microsoft Entra identities.

## Use Cases

- **IoT devices** - Individual tokens for each device to pull images
- **External organizations** - Grant repository access to external parties
- **User group isolation** - Different access levels for different teams
- **Connected registries** - Client and sync token authentication
- **Air-gapped environments** - Authentication without Microsoft Entra connectivity

## Key Concepts

### Tokens

A **token** is an authentication credential with:
- Token name (used as username)
- One or two passwords (with optional expiration)
- Associated scope map defining permissions
- Status (enabled or disabled)

### Scope Maps

A **scope map** defines:
- One or more repositories
- Actions allowed on each repository
- Can be reused across multiple tokens

### Actions

| Action | Description | Example Operations |
|--------|-------------|-------------------|
| `content/read` | Read data from repository | Pull artifacts |
| `content/write` | Write data to repository | Push artifacts (requires `content/read`) |
| `content/delete` | Delete data from repository | Delete manifests, repositories |
| `metadata/read` | Read metadata | List tags, list manifests |
| `metadata/write` | Write metadata | Enable/disable operations |

### Relationship Diagram

```
+------------------+       +-------------------+
|     Token        |------>|    Scope Map      |
|------------------|       |-------------------|
| - name           |       | - name            |
| - password1      |       | - description     |
| - password2      |       | - repositories    |
| - status         |       |   - repo1: actions|
| - expiry         |       |   - repo2: actions|
+------------------+       +-------------------+
```

## System-Defined Scope Maps

ACR provides three built-in scope maps:

| Scope Map | Description | Permissions |
|-----------|-------------|-------------|
| `_repositories_admin` | Full read, write, delete on all repos | All actions on `*` |
| `_repositories_pull` | Pull from any repository | `content/read`, `metadata/read` on `*` |
| `_repositories_push` | Push to any repository | `content/read`, `content/write`, `metadata/read`, `metadata/write` on `*` |

## Creating Tokens via CLI

### Create Token with Inline Repository Permissions

```bash
az acr token create \
  --name MyToken \
  --registry myregistry \
  --repository samples/hello-world content/write content/read \
  --output json
```

**Output includes:**
- Token credentials (password1, password2)
- Auto-created scope map (`MyToken-scope-map`)
- Token ID and status

### Create Token with Existing Scope Map

```bash
# First create scope map
az acr scope-map create \
  --name MyScopeMap \
  --registry myregistry \
  --repository samples/hello-world content/write content/read \
  --description "Sample scope map"

# Then create token using scope map
az acr token create \
  --name MyToken \
  --registry myregistry \
  --scope-map MyScopeMap
```

## Wildcards in Scope Maps

### Repository Prefix Wildcards

Grant access to multiple repositories with common prefix:

```bash
# Create scope map with wildcard
az acr scope-map create \
  --name MyScopeMapWildcard \
  --registry myregistry \
  --repository samples/* content/write content/read \
  --description "Sample scope map with wildcards"

# Create token with wildcard
az acr token create \
  --name MyTokenWildcard \
  --registry myregistry \
  --repository samples/* content/write content/read
```

### Wildcard Rules

1. Wildcards must end with `/*` suffix
2. Only one wildcard per repository name
3. Wildcards are **additive** - permissions from all matching rules apply

**Invalid Wildcards:**
- `sample/*/teamA` - wildcard in middle
- `sample/teamA*` - doesn't end with `/*`
- `sample/teamA/*/projectB/*` - multiple wildcards

### Root Level Wildcards

Grant permissions registry-wide:

```bash
az acr token create \
  --name MyRootToken \
  --registry myregistry \
  --repository * content/write content/read
```

> **Note**: Root wildcards apply to all repositories, including those that don't exist yet.

### Wildcard Permission Inheritance

Example scope map with hierarchical wildcards:

| Repository Pattern | Permission |
|--------------------|------------|
| `sample/*` | `content/read` |
| `sample/teamA/*` | `content/write` |
| `sample/teamA/projectB` | `content/delete` |

**Effective permissions:**
- `sample/teamA/projectB`: `content/read`, `content/write`, `content/delete`
- `sample/teamA/projectC`: `content/read`, `content/write`
- `sample/teamB/foo`: `content/read`

## Creating Tokens via Portal

1. Navigate to registry > **Repository permissions** > **Tokens**
2. Click **+Add**
3. Enter token name
4. Under **Scope map**:
   - Select existing scope map, OR
   - Select **Create new** and configure repositories/permissions
5. Set **Status** to **Enabled**
6. Click **Create**
7. **Generate passwords** after token creation:
   - Navigate to token > click on `password1` or `password2`
   - Click **Generate** icon
   - Set optional expiration date
   - Copy and save password securely

## Authenticating with Tokens

### Docker Login

```bash
TOKEN_NAME=MyToken
TOKEN_PWD=<token-password>

echo $TOKEN_PWD | docker login --username $TOKEN_NAME --password-stdin myregistry.azurecr.io
```

### Azure CLI

```bash
az acr login \
  --name myregistry \
  --username MyToken \
  --password <token-password>
```

### Repository Operations by Action

| Action | CLI Commands |
|--------|--------------|
| `content/read` | `docker login`, `docker pull`, `az acr login` |
| `content/write` | `docker push` |
| `content/delete` | `az acr repository delete` |
| `metadata/read` | `az acr repository show`, `az acr repository show-tags`, `az acr manifest list-metadata` |
| `metadata/write` | `az acr repository untag`, `az acr repository update` |

## Managing Tokens and Scope Maps

### List Scope Maps

```bash
az acr scope-map list --registry myregistry --output table
```

### View Scope Map Details

```bash
az acr scope-map show --name MyScopeMap --registry myregistry
```

### Update Scope Map Permissions

```bash
# Add permissions
az acr scope-map update \
  --name MyScopeMap \
  --registry myregistry \
  --add-repository samples/nginx content/write content/read

# Remove permissions
az acr scope-map update \
  --name MyScopeMap \
  --registry myregistry \
  --remove-repository samples/hello-world content/write
```

### List Tokens

```bash
az acr token list --registry myregistry --output table
```

### View Token Details

```bash
az acr token show --name MyToken --registry myregistry
```

### Regenerate Token Passwords

```bash
# Regenerate password1 with 30-day expiration
TOKEN_PWD=$(az acr token credential generate \
  --name MyToken \
  --registry myregistry \
  --expiration-in-days 30 \
  --password1 \
  --query 'passwords[0].value' \
  --output tsv)
```

> **Note**: Regenerated passwords take up to 60 seconds to replicate.

### Update Token's Scope Map

```bash
az acr token update \
  --name MyToken \
  --registry myregistry \
  --scope-map MyNewScopeMap
```

### Disable Token

```bash
az acr token update \
  --name MyToken \
  --registry myregistry \
  --status disabled
```

### Delete Token

```bash
az acr token delete \
  --name MyToken \
  --registry myregistry
```

## Connected Registry Tokens

Connected registries use two types of tokens:

### Client Tokens

- Used by on-premises clients to authenticate with connected registry
- Scoped for repository pull/push operations
- Must be configured on connected registry via `az acr connected-registry update`

```bash
# Create client token for connected registry
az acr token create \
  --name ClientToken \
  --registry myregistry \
  --repository myrepo content/read content/write

# Associate with connected registry
az acr connected-registry update \
  --name myconnectedregistry \
  --registry myregistry \
  --add-client-token ClientToken
```

### Sync Tokens

- Auto-generated when creating connected registry
- Used for synchronization between connected and parent registry
- Cannot be deleted until connected registry is deleted

```bash
# Get sync token credentials
az acr connected-registry get-settings \
  --name myconnectedregistry \
  --registry myregistry
```

## Limitations

| Limit | Value |
|-------|-------|
| Maximum tokens per registry | 20,000 |
| Maximum scope maps per registry | 20,000 |
| Maximum repository permissions per scope map | 500 |
| Maximum clients per connected registry | 50 |

## Tokens vs ABAC Comparison

| Feature | Tokens | ABAC |
|---------|--------|------|
| Requires Entra identity | No | Yes |
| Repository-level permissions | Yes | Yes |
| Supports wildcards | Yes | Yes |
| Password management | Manual | N/A |
| Expiration support | Yes | N/A |
| Connected registry support | Yes | No |
| Audit via Entra logs | Limited | Full |
| Catalog listing | No | Separate role |

## Best Practices

1. **Use strong passwords** - Let Azure generate passwords
2. **Set expiration dates** - Rotate credentials regularly
3. **Scope narrowly** - Grant minimum required permissions
4. **Monitor token usage** - Review and disable unused tokens
5. **Document tokens** - Track which tokens are for which purpose
6. **Prefer ABAC for Entra identities** - Use tokens only when Entra is not feasible

## Source Documentation

- MS Learn: `/submodules/azure-management-docs/articles/container-registry/container-registry-token-based-repository-permissions.md`
- MS Learn: `/submodules/azure-management-docs/articles/container-registry/intro-connected-registry.md`
- ACR Repo: `/submodules/acr/docs/preview/connected-registry/overview-connected-registry-access.md`

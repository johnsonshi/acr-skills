# ACR RBAC & Authorization Skill

This skill provides comprehensive knowledge about Azure Container Registry role-based and attribute-based access control.

## When to Use This Skill

Use this skill when answering questions about:
- ACR built-in roles (AcrPull, AcrPush, etc.)
- ABAC repository permissions
- Custom roles
- Repository-scoped tokens
- Scope maps

## Registry Permission Modes

| Mode | Description |
|------|-------------|
| **RBAC Registry Permissions** | Traditional roles apply to entire registry |
| **RBAC + ABAC Repository Permissions** | Fine-grained per-repository permissions |

## Built-in Roles

### Data Plane Roles (Non-ABAC Registries)
| Role | Pull | Push | Delete |
|------|------|------|--------|
| AcrPull | ✅ | ❌ | ❌ |
| AcrPush | ✅ | ✅ | ❌ |
| AcrDelete | ❌ | ❌ | ✅ |
| AcrImageSigner | ✅ | ✅ | ❌ (+ sign) |

### ABAC-Enabled Roles
| Role | Pull | Push | Delete |
|------|------|------|--------|
| Container Registry Repository Reader | ✅ | ❌ | ❌ |
| Container Registry Repository Writer | ✅ | ✅ | ❌ |
| Container Registry Repository Contributor | ✅ | ✅ | ✅ |
| Container Registry Repository Catalog Lister | List repos | ❌ | ❌ |

### Control Plane Roles
| Role | Purpose |
|------|---------|
| Container Registry Contributor | Full management (no data) |
| Container Registry Configuration Reader | Read config |
| Container Registry Tasks Contributor | Manage tasks |

## ABAC Repository Permissions

### Enable ABAC Mode
```bash
az acr update --name myregistry --role-assignment-mode AbacRepositoryPermissions
```

### Assign with Conditions
```bash
az role assignment create \
  --assignee <principal-id> \
  --role "Container Registry Repository Reader" \
  --scope <registry-id> \
  --condition "
    @Resource[Microsoft.ContainerRegistry/registries/repositories:name] StringStartsWithIgnoreCase 'team-a/'
  " \
  --condition-version "2.0"
```

### Condition Operators
| Operator | Example |
|----------|---------|
| `StringEqualsIgnoreCase` | Exact match |
| `StringStartsWithIgnoreCase` | Prefix match (e.g., `team-a/*`) |

> **Note:** In ABAC mode, Owner/Contributor/Reader have NO data plane permissions!

## Repository-Scoped Tokens

For non-Entra authentication (IoT, external systems):

### Create Token
```bash
az acr token create \
  --name mytoken \
  --registry myregistry \
  --repository myrepo content/read content/write
```

### Token Actions
| Action | Description |
|--------|-------------|
| `content/read` | Pull artifacts |
| `content/write` | Push artifacts |
| `content/delete` | Delete artifacts |
| `metadata/read` | List tags/manifests |
| `metadata/write` | Enable/disable operations |

### Generate Password
```bash
az acr token credential generate \
  --name mytoken \
  --registry myregistry \
  --expiration-in-days 30 \
  --password1
```

## Scope Maps

Define reusable permission sets:

```bash
# Create scope map
az acr scope-map create \
  --name team-a-access \
  --registry myregistry \
  --repository team-a/app1 content/read content/write \
  --repository team-a/app2 content/read

# Create token with scope map
az acr token create \
  --name team-a-token \
  --registry myregistry \
  --scope-map team-a-access
```

### Wildcard Support
```bash
# All repos under prefix
--repository "team-a/*" content/read

# All repos (root wildcard)
--repository "*" content/read
```

## Custom Roles

```bash
az role definition create --role-definition '{
  "Name": "ACR Webhook Manager",
  "Actions": [
    "Microsoft.ContainerRegistry/registries/webhooks/*"
  ],
  "AssignableScopes": ["/subscriptions/{sub-id}"]
}'
```

### Available Actions
- `Microsoft.ContainerRegistry/registries/read`
- `Microsoft.ContainerRegistry/registries/push/write`
- `Microsoft.ContainerRegistry/registries/pull/read`
- `Microsoft.ContainerRegistry/registries/delete`
- `Microsoft.ContainerRegistry/registries/webhooks/*`
- `Microsoft.ContainerRegistry/registries/tasks/*`

## Key Considerations

1. **ABAC vs Non-ABAC**: Choose mode based on whether you need per-repo permissions
2. **DCT Not Supported**: Docker Content Trust doesn't work with ABAC-enabled registries
3. **Connected Registry**: Only supports token-based auth, not Entra RBAC
4. **Limits**: 20,000 tokens/scope maps, 500 repo permissions per scope map

## Reference Documentation

- **Investigation reports**: `agent-investigation-reports/acr-features/rbac-authorization/`
- **MS Learn docs**: `azure-management-docs/articles/container-registry/container-registry-rbac-built-in-roles-overview.md`

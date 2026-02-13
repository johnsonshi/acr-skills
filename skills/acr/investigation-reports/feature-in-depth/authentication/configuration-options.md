# Azure Container Registry Authentication - Configuration Options

## Registry-Level Authentication Settings

### Admin Account Configuration

| Setting | Values | Default | Description |
|---------|--------|---------|-------------|
| `adminUserEnabled` | `true`, `false` | `false` | Enable/disable admin account |

**CLI Configuration:**
```bash
az acr update --name <registry> --admin-enabled true|false
```

**Portal Location:** Registry > Settings > Access keys

### Anonymous Pull Access

| Setting | Values | Default | Description |
|---------|--------|---------|-------------|
| `anonymousPullEnabled` | `true`, `false` | `false` | Enable unauthenticated read access |

**CLI Configuration:**
```bash
az acr update --name <registry> --anonymous-pull-enabled true|false
```

**Requirements:**
- Standard or Premium SKU only
- Applies to all repositories in registry

### Authentication-as-ARM Policy

| Setting | Values | Default | Description |
|---------|--------|---------|-------------|
| `azureADAuthenticationAsArmPolicy.status` | `enabled`, `disabled` | `enabled` | Control accepted Microsoft Entra token scopes |

**Configuration Options:**
- `enabled`: Accept both ARM-scoped and ACR-scoped Microsoft Entra tokens
- `disabled`: Accept only ACR-scoped Microsoft Entra tokens (recommended)

**CLI Configuration:**
```bash
az acr config authentication-as-arm show --registry <registry>
az acr config authentication-as-arm update --registry <registry> --status enabled|disabled
```

### Trusted Services Bypass

| Setting | Values | Default | Description |
|---------|--------|---------|-------------|
| `networkRuleBypassOptions` | `AzureServices`, `None` | `AzureServices` | Allow trusted Azure services to bypass network rules |

**CLI Configuration:**
```bash
az acr update --name <registry> --allow-trusted-services true|false
```

**Portal Location:** Registry > Settings > Networking > Firewall exception

## Token Configuration Options

### Token Properties

| Property | Type | Description |
|----------|------|-------------|
| `name` | string | Token name (unique within registry) |
| `status` | string | `enabled` or `disabled` |
| `scopeMapId` | string | Associated scope map resource ID |
| `credentials.passwords` | array | Generated passwords |

### Token Password Properties

| Property | Type | Description |
|----------|------|-------------|
| `name` | string | `password1` or `password2` |
| `value` | string | Generated password value |
| `creationTime` | datetime | Password creation timestamp |
| `expiry` | datetime | Password expiration (null = no expiry) |

### Scope Map Properties

| Property | Type | Description |
|----------|------|-------------|
| `name` | string | Scope map name |
| `type` | string | `SystemDefined` or `UserDefined` |
| `description` | string | Human-readable description |
| `actions` | array | Repository action permissions |

### Scope Map Actions

| Action | Scope | Description |
|--------|-------|-------------|
| `content/read` | Repository | Pull artifacts |
| `content/write` | Repository | Push artifacts |
| `content/delete` | Repository | Delete artifacts, manifests, tags |
| `metadata/read` | Repository | List tags, manifests |
| `metadata/write` | Repository | Update metadata |

### System-Defined Scope Maps

| Scope Map Name | Permissions |
|----------------|-------------|
| `_repositories_admin` | All read, write, delete operations |
| `_repositories_pull` | Pull from any repository |
| `_repositories_push` | Push to any repository |

### Wildcard Patterns in Scope Maps

| Pattern | Description |
|---------|-------------|
| `samples/*` | All repositories under samples/ |
| `*` | All repositories (root-level) |

**Wildcard Rules:**
- Must end with `/*` suffix
- Only single wildcard allowed per repository path
- Invalid: `sample/*/teamA`, `sample/teamA*`, `sample/*/projectB/*`

## Service Principal Configuration

### Service Principal Properties

| Property | Description | Example Format |
|----------|-------------|----------------|
| Application (client) ID | Username for authentication | `xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx` |
| Client secret | Password for authentication | Generated string |
| Tenant ID | Microsoft Entra tenant | `xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx` |

### Service Principal Expiration

| Setting | Default | Maximum | Notes |
|---------|---------|---------|-------|
| Client secret validity | 1 year | Configurable | Use `az ad sp credential reset` to extend |

### Certificate Authentication Properties

| Property | Description |
|----------|-------------|
| Certificate format | PEM with private key |
| Authentication | `az login --service-principal --password /path/to/cert.pem` |

## Managed Identity Configuration

### System-Assigned Identity Properties

| Property | Description |
|----------|-------------|
| `principalId` | Object ID for role assignments |
| `tenantId` | Microsoft Entra tenant |

### User-Assigned Identity Properties

| Property | Description |
|----------|-------------|
| `id` | Full resource ID |
| `principalId` | Object ID for role assignments |
| `clientId` | Client ID for authentication |
| `tenantId` | Microsoft Entra tenant |

## Role Assignment Configuration

### Built-in ACR Roles (Non-ABAC Registries)

| Role | Permissions | Data Actions |
|------|-------------|--------------|
| `AcrPull` | Pull images | `content/read` |
| `AcrPush` | Push and pull images | `content/read`, `content/write` |
| `AcrDelete` | Delete images | `content/delete` |
| `AcrImageSigner` | Sign images (DCT) | Trust data operations |

### Built-in ACR Roles (ABAC-Enabled Registries)

| Role | Permissions | ABAC Support |
|------|-------------|--------------|
| `Container Registry Repository Reader` | Pull images | Yes |
| `Container Registry Repository Writer` | Push and pull images | Yes |
| `Container Registry Repository Contributor` | Full data plane access | Yes |
| `Container Registry Repository Catalog Lister` | List repositories | No |

### Control Plane Roles

| Role | Permissions |
|------|-------------|
| `Container Registry Contributor and Data Access Configuration Administrator` | Create, configure, delete registries |
| `Container Registry Configuration Reader and Data Access Configuration Reader` | Read registry configurations |
| `Container Registry Tasks Contributor` | Manage ACR Tasks |
| `Container Registry Data Importer and Data Reader` | Import images |
| `Container Registry Transfer Pipeline Contributor` | Manage transfer pipelines |

## OAuth2 Token Configuration

### Token Endpoint URLs

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/oauth2/exchange` | POST | Exchange Microsoft Entra token for ACR refresh token |
| `/oauth2/token` | GET/POST | Get ACR access token |

### Token Exchange Request Parameters

| Parameter | Required | Description |
|-----------|----------|-------------|
| `grant_type` | Yes | `access_token`, `refresh_token` |
| `service` | Yes | Registry login server |
| `tenant` | Yes | Microsoft Entra tenant ID |
| `access_token` | Conditional | Microsoft Entra access token |
| `scope` | For access token | Operation scope (e.g., `repository:myrepo:pull`) |

### Token Response Properties

| Property | Description |
|----------|-------------|
| `access_token` | JWT for API calls |
| `refresh_token` | JWT for obtaining new access tokens |

### Token Lifetimes

| Token Type | Validity | Renewal |
|------------|----------|---------|
| Microsoft Entra token (az acr login) | 3 hours | Re-run `az acr login` |
| ACR refresh token | Configurable | Via token API |
| ACR access token | Short-lived | Via `/oauth2/token` |

## Conditional Access Configuration

### Prerequisites

| Requirement | Description |
|-------------|-------------|
| Disable authentication-as-arm | Must be disabled for all registries in tenant |
| User scope | Applies to Microsoft Entra users only (not service principals) |

### Policy Components

| Component | Options |
|-----------|---------|
| Users | Select users, groups, or all users |
| Cloud apps | Azure Container Registry |
| Conditions | User risk, sign-in risk, device, location, client apps |
| Grant controls | Require MFA, compliant device, etc. |
| Session controls | Sign-in frequency, persistent browser |

## AKS Integration Configuration

### Same-Tenant AKS Integration

| Configuration | Value |
|---------------|-------|
| Authentication method | Managed identity |
| Required role | `AcrPull` or `Container Registry Repository Reader` |

### Cross-Tenant AKS Integration

| Configuration | Value |
|---------------|-------|
| App registration | Multitenant |
| Supported account types | Accounts in any organizational directory |
| Redirect URI | `https://www.microsoft.com` (Web platform) |

## Environment Variables

### Docker Configuration

| Variable | Purpose |
|----------|---------|
| `DOCKER_COMMAND` | Alternative container tool (e.g., `podman`) |

### Azure CLI Configuration

| Variable | Purpose |
|----------|---------|
| `AZURE_CONFIG_DIR` | Azure CLI configuration directory |

## ACR Task Identity Configuration

### Task Identity Assignment

| Option | Value | Description |
|--------|-------|-------------|
| `--assign-identity` | (no value) | System-assigned identity |
| `--assign-identity` | Resource ID | User-assigned identity |

### Task Credential Configuration

| Parameter | Value | Description |
|-----------|-------|-------------|
| `--use-identity` | `[system]` | Use system-assigned identity |
| `--use-identity` | Client ID | Use user-assigned identity |

## Sources

- `/Users/johnsonshi/repos/acr-skills/submodules/azure-management-docs/articles/container-registry/container-registry-authentication.md`
- `/Users/johnsonshi/repos/acr-skills/submodules/azure-management-docs/articles/container-registry/container-registry-token-based-repository-permissions.md`
- `/Users/johnsonshi/repos/acr-skills/submodules/azure-management-docs/articles/container-registry/container-registry-disable-authentication-as-arm.md`
- `/Users/johnsonshi/repos/acr-skills/submodules/azure-management-docs/articles/container-registry/container-registry-configure-conditional-access.md`
- `/Users/johnsonshi/repos/acr-skills/submodules/azure-management-docs/articles/container-registry/container-registry-rbac-built-in-roles-overview.md`
- `/Users/johnsonshi/repos/acr-skills/submodules/acr/docs/AAD-OAuth.md`

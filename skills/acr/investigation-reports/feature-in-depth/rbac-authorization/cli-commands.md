# Azure Container Registry RBAC & Authorization CLI Commands

## Overview

This document provides a comprehensive reference of Azure CLI commands for managing ACR RBAC, ABAC, tokens, and scope maps.

---

## Role Assignment Commands

### View Current Role Assignments

```bash
# List all role assignments for a registry
az role assignment list \
  --scope /subscriptions/<sub>/resourceGroups/<rg>/providers/Microsoft.ContainerRegistry/registries/<registry> \
  --output table

# List role assignments for specific identity
az role assignment list \
  --assignee user@example.com \
  --scope /subscriptions/<sub>/resourceGroups/<rg>/providers/Microsoft.ContainerRegistry/registries/<registry>
```

### Create Role Assignments

```bash
# Assign built-in role (non-ABAC registry)
az role assignment create \
  --role "AcrPull" \
  --assignee user@example.com \
  --scope /subscriptions/<sub>/resourceGroups/<rg>/providers/Microsoft.ContainerRegistry/registries/<registry>

# Assign built-in role (ABAC-enabled registry)
az role assignment create \
  --role "Container Registry Repository Reader" \
  --assignee user@example.com \
  --scope /subscriptions/<sub>/resourceGroups/<rg>/providers/Microsoft.ContainerRegistry/registries/<registry>

# Assign role to managed identity
az role assignment create \
  --role "AcrPull" \
  --assignee-object-id <managed-identity-principal-id> \
  --assignee-principal-type ServicePrincipal \
  --scope /subscriptions/<sub>/resourceGroups/<rg>/providers/Microsoft.ContainerRegistry/registries/<registry>

# Assign role to service principal
az role assignment create \
  --role "AcrPush" \
  --assignee <service-principal-app-id> \
  --scope /subscriptions/<sub>/resourceGroups/<rg>/providers/Microsoft.ContainerRegistry/registries/<registry>
```

### Delete Role Assignments

```bash
az role assignment delete \
  --role "AcrPull" \
  --assignee user@example.com \
  --scope /subscriptions/<sub>/resourceGroups/<rg>/providers/Microsoft.ContainerRegistry/registries/<registry>
```

---

## ABAC Configuration Commands

### Configure Registry Role Assignment Mode

```bash
# Create registry with ABAC enabled
az acr create \
  --name <registry-name> \
  --resource-group <resource-group> \
  --sku Premium \
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
  --query "roleAssignmentMode" \
  --output tsv
```

### Create Role Assignment with ABAC Condition

```bash
# Define condition for specific repository
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
scope=$(az acr show --name <registry> --resource-group <rg> --query "id" -o tsv)

# Create role assignment with condition
az role assignment create \
  --role "Container Registry Repository Reader" \
  --scope "$scope" \
  --assignee "user@example.com" \
  --description "Read access to hello-world repository" \
  --condition "$condition" \
  --condition-version "2.0"
```

### ABAC Condition for Repository Prefix (Wildcard)

```bash
condition=$(cat <<'EOF' | tr -d '\n'
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
EOF
)

az role assignment create \
  --role "Container Registry Repository Reader" \
  --scope "$scope" \
  --assignee "user@example.com" \
  --description "Read access to backend/* repositories" \
  --condition "$condition" \
  --condition-version "2.0"
```

### ABAC Condition for Multiple Prefixes

```bash
condition=$(cat <<'EOF' | tr -d '\n'
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
EOF
)

az role assignment create \
  --role "Container Registry Repository Reader" \
  --scope "$scope" \
  --assignee "user@example.com" \
  --description "Read access to backend/* and frontend/js/* repositories" \
  --condition "$condition" \
  --condition-version "2.0"
```

### Update Role Assignment Condition

```bash
az role assignment update \
  --role "Container Registry Repository Reader" \
  --scope "$scope" \
  --assignee "user@example.com" \
  --condition "$new_condition" \
  --condition-version "2.0"
```

---

## Token Commands

### Create Tokens

```bash
# Create token with inline repository permissions
az acr token create \
  --name MyToken \
  --registry myregistry \
  --repository samples/hello-world content/write content/read \
  --output json

# Create token with existing scope map
az acr token create \
  --name MyToken \
  --registry myregistry \
  --scope-map MyScopeMap

# Create disabled token
az acr token create \
  --name MyToken \
  --registry myregistry \
  --scope-map MyScopeMap \
  --status disabled

# Create token with wildcard permissions
az acr token create \
  --name MyTokenWildcard \
  --registry myregistry \
  --repository samples/* content/write content/read

# Create token with root-level wildcard (all repos)
az acr token create \
  --name MyRootToken \
  --registry myregistry \
  --repository * content/write content/read
```

### List and Show Tokens

```bash
# List all tokens
az acr token list --registry myregistry --output table

# Show token details
az acr token show --name MyToken --registry myregistry
```

### Update Tokens

```bash
# Change scope map
az acr token update \
  --name MyToken \
  --registry myregistry \
  --scope-map NewScopeMap

# Disable token
az acr token update \
  --name MyToken \
  --registry myregistry \
  --status disabled

# Enable token
az acr token update \
  --name MyToken \
  --registry myregistry \
  --status enabled
```

### Delete Tokens

```bash
az acr token delete \
  --name MyToken \
  --registry myregistry \
  --yes
```

### Token Credential Management

```bash
# Generate new password1 with no expiration
az acr token credential generate \
  --name MyToken \
  --registry myregistry \
  --password1

# Generate password1 with 30-day expiration
az acr token credential generate \
  --name MyToken \
  --registry myregistry \
  --password1 \
  --expiration-in-days 30

# Generate password2
az acr token credential generate \
  --name MyToken \
  --registry myregistry \
  --password2 \
  --expiration-in-days 90

# Store generated password in variable
TOKEN_PWD=$(az acr token credential generate \
  --name MyToken \
  --registry myregistry \
  --password1 \
  --expiration-in-days 30 \
  --query 'passwords[0].value' \
  --output tsv)

# Delete specific password
az acr token credential delete \
  --name MyToken \
  --registry myregistry \
  --password1
```

---

## Scope Map Commands

### Create Scope Maps

```bash
# Create scope map with single repository
az acr scope-map create \
  --name MyScopeMap \
  --registry myregistry \
  --repository samples/hello-world content/write content/read \
  --description "Read/write access to hello-world"

# Create scope map with multiple repositories
az acr scope-map create \
  --name MyScopeMap \
  --registry myregistry \
  --repository samples/hello-world content/read \
  --repository samples/nginx content/write content/read \
  --description "Multiple repository access"

# Create scope map with wildcard
az acr scope-map create \
  --name MyScopeMapWildcard \
  --registry myregistry \
  --repository samples/* content/write content/read \
  --description "Wildcard access to samples/*"

# Create scope map with all actions
az acr scope-map create \
  --name MyAdminScopeMap \
  --registry myregistry \
  --repository myrepo content/read content/write content/delete metadata/read metadata/write \
  --description "Full access to myrepo"
```

### List and Show Scope Maps

```bash
# List all scope maps (including system-defined)
az acr scope-map list --registry myregistry --output table

# Show scope map details
az acr scope-map show --name MyScopeMap --registry myregistry

# Show system-defined scope maps
az acr scope-map show --name _repositories_pull --registry myregistry
az acr scope-map show --name _repositories_push --registry myregistry
az acr scope-map show --name _repositories_admin --registry myregistry
```

### Update Scope Maps

```bash
# Add repository permissions
az acr scope-map update \
  --name MyScopeMap \
  --registry myregistry \
  --add-repository samples/nginx content/write content/read

# Remove repository permissions
az acr scope-map update \
  --name MyScopeMap \
  --registry myregistry \
  --remove-repository samples/hello-world content/write

# Add and remove in single command
az acr scope-map update \
  --name MyScopeMap \
  --registry myregistry \
  --add-repository samples/new-repo content/read \
  --remove-repository samples/old-repo content/read content/write
```

### Delete Scope Maps

```bash
az acr scope-map delete \
  --name MyScopeMap \
  --registry myregistry \
  --yes
```

---

## Connected Registry Token Commands

### Create Client Token for Connected Registry

```bash
# Create scope map for connected registry client
az acr scope-map create \
  --name ClientScopeMap \
  --registry myregistry \
  --repository hello-world content/read \
  --description "Client access to hello-world"

# Create client token
az acr token create \
  --name ClientToken \
  --registry myregistry \
  --scope-map ClientScopeMap

# Associate token with connected registry
az acr connected-registry update \
  --name myconnectedregistry \
  --registry myregistry \
  --add-client-token ClientToken
```

### Get Connected Registry Sync Token

```bash
# Get settings including sync token
az acr connected-registry get-settings \
  --name myconnectedregistry \
  --registry myregistry \
  --generate-password 1

# Show connected registry (includes sync token name)
az acr connected-registry show \
  --name myconnectedregistry \
  --registry myregistry
```

---

## Custom Role Commands

### List Available Permissions

```bash
# List all ACR permissions
az provider operation show --namespace Microsoft.ContainerRegistry

# Filter for specific operations
az provider operation show \
  --namespace Microsoft.ContainerRegistry \
  --query "resourceTypes[].operations[?contains(name, 'token')]" \
  --output table
```

### Create Custom Role

```bash
# Create from JSON file
az role definition create --role-definition custom-role.json

# Inline JSON (example)
az role definition create --role-definition '{
  "Name": "ACR Webhook Manager",
  "Description": "Manage ACR webhooks only",
  "Actions": [
    "Microsoft.ContainerRegistry/registries/webhooks/*"
  ],
  "AssignableScopes": ["/subscriptions/<subscription-id>"]
}'
```

### Update Custom Role

```bash
az role definition update --role-definition updated-role.json
```

### Delete Custom Role

```bash
az role definition delete --name "ACR Webhook Manager"
```

### List Custom Roles

```bash
az role definition list \
  --custom-role-only \
  --query "[?contains(assignableScopes[0], 'Microsoft.ContainerRegistry')]" \
  --output table
```

---

## ACR Tasks with ABAC

### Create Task with Identity for Source Registry

```bash
# System-assigned identity
az acr task create \
  --name mytask \
  --registry myregistry \
  --source-acr-auth-id [system] \
  --image myimage:{{.Run.ID}} \
  --file Dockerfile \
  --context https://github.com/myorg/myrepo.git

# User-assigned identity
az acr task create \
  --name mytask \
  --registry myregistry \
  --source-acr-auth-id /subscriptions/<sub>/resourceGroups/<rg>/providers/Microsoft.ManagedIdentity/userAssignedIdentities/<identity> \
  --image myimage:{{.Run.ID}} \
  --file Dockerfile \
  --context https://github.com/myorg/myrepo.git
```

### Update Task Identity

```bash
az acr task update \
  --name mytask \
  --registry myregistry \
  --source-acr-auth-id /subscriptions/<sub>/resourceGroups/<rg>/providers/Microsoft.ManagedIdentity/userAssignedIdentities/<identity>
```

### Quick Build with Caller Identity

```bash
# Use caller's identity for authentication
az acr build \
  --source-acr-auth-id [caller] \
  --registry myregistry \
  --image myimage:latest \
  .
```

### Disable Task Source Registry Access

```bash
az acr task update \
  --name mytask \
  --registry myregistry \
  --source-acr-auth-id none
```

---

## Verification Commands

### Verify Token Authentication

```bash
# Test docker login with token
echo $TOKEN_PWD | docker login --username MyToken --password-stdin myregistry.azurecr.io

# Test ACR login
az acr login --name myregistry --username MyToken --password $TOKEN_PWD

# Test repository access
az acr repository show-tags \
  --name myregistry \
  --repository samples/hello-world \
  --username MyToken \
  --password $TOKEN_PWD
```

### Verify Role Assignment

```bash
# Check effective permissions for identity
az role assignment list \
  --assignee user@example.com \
  --scope /subscriptions/<sub>/resourceGroups/<rg>/providers/Microsoft.ContainerRegistry/registries/<registry> \
  --include-inherited \
  --output table
```

---

## Source Documentation

- MS Learn: `/submodules/azure-management-docs/articles/container-registry/container-registry-rbac-built-in-roles-overview.md`
- MS Learn: `/submodules/azure-management-docs/articles/container-registry/container-registry-rbac-abac-repository-permissions.md`
- MS Learn: `/submodules/azure-management-docs/articles/container-registry/container-registry-token-based-repository-permissions.md`
- MS Learn: `/submodules/azure-management-docs/articles/container-registry/container-registry-rbac-custom-roles.md`
- Azure CLI Reference: `az role assignment`, `az acr token`, `az acr scope-map`

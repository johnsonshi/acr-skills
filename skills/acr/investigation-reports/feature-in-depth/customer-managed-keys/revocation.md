# Key Revocation for Customer-Managed Keys in Azure Container Registry

## Overview

Key revocation is the process of removing access to the customer-managed encryption key, which immediately blocks all registry operations. This is a powerful security measure for scenarios requiring immediate data protection, such as security incidents or compliance requirements.

## Understanding Key Revocation

### What Happens When a Key is Revoked

1. **Immediate Impact**: Registry becomes inaccessible
2. **Push Operations**: Blocked - cannot upload images
3. **Pull Operations**: Blocked - cannot download images
4. **Registry Content**: Remains encrypted and protected
5. **Recovery**: Possible by restoring key access

### Use Cases for Key Revocation

- **Security Incident Response**: Suspected unauthorized access
- **Compliance Requirements**: Legal or regulatory holds
- **Decommissioning**: Permanent registry shutdown
- **Access Control**: Emergency lockdown scenarios

## Revocation Methods

### Method 1: Remove Key Vault Access Policy

Remove the managed identity's access to the Key Vault:

```azurecli
az keyvault delete-policy \
  --resource-group <resource-group-name> \
  --name <key-vault-name> \
  --object-id <identity-principal-id>
```

**Effect**: Identity can no longer access the key, blocking all registry operations.

**Recovery**: Restore the access policy with proper permissions.

### Method 2: Modify Key Vault Permissions

If using Azure RBAC instead of access policies, remove the role assignment:

```azurecli
# List role assignments
az role assignment list \
  --scope <key-vault-resource-id> \
  --assignee <identity-principal-id>

# Delete the role assignment
az role assignment delete \
  --assignee <identity-principal-id> \
  --role "Key Vault Crypto Service Encryption User" \
  --scope <key-vault-resource-id>
```

**Effect**: Same as removing access policy - registry operations blocked.

### Method 3: Delete the Key

Delete the encryption key from Key Vault:

```azurecli
az keyvault key delete \
  --name <key-name> \
  --vault-name <key-vault-name>
```

**Important**: This operation requires the `keys/delete` permission.

**Effect**: Key is moved to soft-deleted state (if soft delete is enabled).

**Recovery**: If soft delete is enabled, the key can be recovered.

### Method 4: Purge the Key (Permanent)

To permanently delete a soft-deleted key:

```azurecli
az keyvault key purge \
  --name <key-name> \
  --vault-name <key-vault-name>
```

**WARNING**: This action is IRREVERSIBLE if purge protection is not enabled. The registry data becomes permanently inaccessible.

**Effect**: Key is permanently destroyed.

**Recovery**: Not possible - data is permanently encrypted.

## Revocation Scenarios

### Scenario 1: Temporary Lockdown (Reversible)

For situations requiring temporary access block:

```azurecli
# Step 1: Get the identity's principal ID
identityPrincipalID=$(az acr encryption show \
  --name <registry-name> \
  --query 'keyVaultProperties.identity' --output tsv)

# Step 2: Remove Key Vault access
az keyvault delete-policy \
  --resource-group <resource-group-name> \
  --name <key-vault-name> \
  --object-id $identityPrincipalID

# Step 3: Verify registry is locked
# Attempt to pull an image - should fail
docker pull <registry-name>.azurecr.io/myimage:tag
# Expected error: access denied or authentication failure
```

### Scenario 2: Security Incident Response

For suspected security breach:

```azurecli
# Step 1: Immediately revoke access
az keyvault delete-policy \
  --resource-group <resource-group-name> \
  --name <key-vault-name> \
  --object-id <identity-principal-id>

# Step 2: Enable Key Vault audit logging if not already enabled
az monitor diagnostic-settings create \
  --name "SecurityAudit" \
  --resource <key-vault-resource-id> \
  --logs '[{"category": "AuditEvent", "enabled": true, "retentionPolicy": {"enabled": true, "days": 90}}]' \
  --workspace <log-analytics-workspace-id>

# Step 3: Investigate and remediate

# Step 4: After investigation, restore access if appropriate
az keyvault set-policy \
  --resource-group <resource-group-name> \
  --name <key-vault-name> \
  --object-id <identity-principal-id> \
  --key-permissions get unwrapKey wrapKey
```

### Scenario 3: Permanent Decommissioning

For permanent registry shutdown:

```azurecli
# Step 1: Export any needed images first
# (Registry operations must still be possible)

# Step 2: Remove Key Vault access
az keyvault delete-policy \
  --resource-group <resource-group-name> \
  --name <key-vault-name> \
  --object-id <identity-principal-id>

# Step 3: Optionally delete the key
az keyvault key delete \
  --name <key-name> \
  --vault-name <key-vault-name>

# Step 4: If permanent deletion required (and purge protection allows)
# Wait for the soft delete retention period or
az keyvault key purge \
  --name <key-name> \
  --vault-name <key-vault-name>

# Step 5: Delete the registry
az acr delete --name <registry-name> --resource-group <resource-group-name>
```

## Recovery Procedures

### Recovering from Access Policy Revocation

Restore the access policy:

```azurecli
az keyvault set-policy \
  --resource-group <resource-group-name> \
  --name <key-vault-name> \
  --object-id <identity-principal-id> \
  --key-permissions get unwrapKey wrapKey
```

Verify recovery:

```azurecli
# Check encryption status
az acr encryption show --name <registry-name>

# Test registry access
az acr repository list --name <registry-name>
```

### Recovering a Soft-Deleted Key

If the key was deleted but soft delete is enabled:

```azurecli
# List soft-deleted keys
az keyvault key list-deleted --vault-name <key-vault-name>

# Recover the deleted key
az keyvault key recover \
  --name <key-name> \
  --vault-name <key-vault-name>
```

### Recovering a Soft-Deleted Key Vault

If the entire Key Vault was deleted:

```azurecli
# List soft-deleted vaults
az keyvault list-deleted

# Recover the vault
az keyvault recover --name <key-vault-name>
```

## Impact Analysis

### Operations Blocked by Revocation

| Operation | Status | Error |
|-----------|--------|-------|
| docker push | Blocked | Authentication/Access denied |
| docker pull | Blocked | Authentication/Access denied |
| az acr repository list | Blocked | Access denied |
| ACR Tasks | Blocked | Cannot access registry |
| Geo-replication sync | Blocked | Replication fails |
| Artifact streaming | Blocked | Cannot decrypt content |

### Operations Still Available After Revocation

| Operation | Status | Notes |
|-----------|--------|-------|
| az acr show | Works | Metadata accessible |
| az acr encryption show | Works | Shows encryption config |
| az acr update | Works | Non-encryption settings |
| Registry metrics | Works | Historical data available |

## Best Practices for Revocation

### 1. Plan Before Revoking

- Document the reason for revocation
- Identify all systems dependent on the registry
- Communicate with stakeholders
- Have a recovery plan ready

### 2. Enable Soft Delete and Purge Protection

Ensure Key Vault has these features enabled:

```azurecli
az keyvault update \
  --name <key-vault-name> \
  --enable-soft-delete true \
  --enable-purge-protection true
```

This provides a safety net for accidental deletions.

### 3. Test Revocation Procedures

In non-production environments:

1. Create a test registry with CMK
2. Perform revocation
3. Verify registry is locked
4. Perform recovery
5. Verify registry is accessible
6. Document timing and impacts

### 4. Monitor Revocation Events

Set up alerts for Key Vault access:

```azurecli
# Create action group for alerts
az monitor action-group create \
  --name "CMK-Alerts" \
  --resource-group <resource-group-name> \
  --short-name "cmkalerts" \
  --email-receiver name=admin email=admin@company.com

# Create alert rule for access policy changes
az monitor scheduled-query create \
  --name "KeyVault-Access-Changed" \
  --resource-group <resource-group-name> \
  --scopes <log-analytics-workspace-id> \
  --condition "AzureDiagnostics | where ResourceProvider == 'MICROSOFT.KEYVAULT' | where OperationName contains 'VaultAccessPolicy'" \
  --action-group <action-group-id>
```

### 5. Document Recovery Procedures

Maintain documentation including:

- Identity principal IDs
- Key Vault names and resource IDs
- Key names
- Required permissions
- Recovery steps
- Escalation contacts

## Revocation Decision Matrix

| Scenario | Recommended Action | Recovery |
|----------|-------------------|----------|
| Suspected breach | Remove access policy | Restore policy after investigation |
| Compliance hold | Remove access policy | Restore when hold is lifted |
| Employee offboarding | Rotate key with new identity | N/A |
| Permanent shutdown | Delete key | Not needed |
| Accidental exposure | Rotate to new key | N/A |
| Key compromise | Delete and purge key | Create new registry |

## Emergency Revocation Checklist

- [ ] Identify the reason for revocation
- [ ] Document current state (az acr encryption show)
- [ ] Notify affected teams
- [ ] Execute revocation (delete policy/key)
- [ ] Verify registry is locked
- [ ] Document the revocation event
- [ ] Begin incident response if applicable
- [ ] Plan recovery timeline if applicable
- [ ] Execute recovery when appropriate
- [ ] Verify registry functionality restored
- [ ] Post-incident review

## Source References

- `/submodules/azure-management-docs/articles/container-registry/tutorial-rotate-revoke-customer-managed-keys.md`
- `/submodules/azure-management-docs/articles/container-registry/tutorial-troubleshoot-customer-managed-keys.md`
- `/submodules/azure-management-docs/articles/container-registry/tutorial-customer-managed-keys.md`

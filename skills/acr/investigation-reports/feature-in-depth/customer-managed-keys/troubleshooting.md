# Troubleshooting Customer-Managed Keys in Azure Container Registry

## Overview

This guide provides comprehensive troubleshooting procedures for common issues encountered when using Customer-Managed Keys (CMK) with Azure Container Registry.

## Diagnostic Tools

### Health Check Command

The primary diagnostic tool for CMK issues:

```azurecli
az acr check-health --name <registry-name>
```

### Encryption Status Check

```azurecli
az acr encryption show --name <registry-name>
```

### Key Vault Diagnostics

```azurecli
# Check Key Vault access policies
az keyvault show --name <key-vault-name> --query "properties.accessPolicies"

# Check Key Vault network rules
az keyvault show --name <key-vault-name> --query "properties.networkAcls"

# List keys
az keyvault key list --vault-name <key-vault-name>
```

## Common Error: CMK_ERROR

### Description

The `CMK_ERROR` error in health checks indicates the registry cannot access the managed identity used for CMK configuration.

**Error Message:**
```
CMK_ERROR: The registry can't access the user-assigned or system-assigned managed identity used to configure registry encryption with a customer-managed key.
```

### Common Causes

1. Managed identity was deleted
2. Identity permissions were revoked
3. Key Vault access was removed
4. Network restrictions blocking access

### Resolution Steps

```azurecli
# Step 1: Check current encryption configuration
az acr encryption show --name <registry-name>

# Step 2: Verify identity exists
az identity show \
  --resource-group <resource-group-name> \
  --name <identity-name>

# Step 3: Re-assign identity if needed
az acr identity assign -n <registry-name> \
  --identities "/subscriptions/<subscription>/resourcegroups/<rg>/providers/Microsoft.ManagedIdentity/userAssignedIdentities/<identity-name>"

# Step 4: Verify Key Vault access
az keyvault show --name <key-vault-name> --query "properties.accessPolicies"
```

## Error: Unable to Remove Managed Identity

### Description

When trying to remove a user-assigned or system-assigned managed identity used for CMK encryption, you may see:

**Error Message:**
```
Azure resource '/subscriptions/xxxx/resourcegroups/myGroup/providers/Microsoft.ContainerRegistry/registries/myRegistry' does not have access to identity 'xxxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxx'. Try forcibly adding the identity to the registry <registry name>.
```

### Resolution for User-Assigned Identity

```azurecli
# Step 1: Reassign the user-assigned identity
az acr identity assign -n <registry-name> \
  --identities "/subscriptions/<subscription>/resourcegroups/<rg>/providers/Microsoft.ManagedIdentity/userAssignedIdentities/<identity-name>"

# Step 2: Change to a different key and identity
az acr encryption rotate-key \
  --name <registry-name> \
  --key-encryption-key <new-key-id> \
  --identity <new-identity-id>

# Step 3: Now you can remove the original identity
az acr identity remove -n <registry-name> \
  --identities "<original-identity-resource-id>"
```

### Resolution for System-Assigned Identity

If you encounter this error with a system-assigned identity:

1. Create an Azure support ticket
2. Request assistance in restoring the identity
3. Provide registry name and subscription details

## Error: HTTP 403 After Enabling Key Vault Firewall

### Description

After enabling a Key Vault firewall or virtual network, you receive HTTP 403 errors or experience issues with:
- Image import
- Automated key rotation
- Push/pull operations

### Resolution

```azurecli
# Option 1: Enable trusted services bypass
az keyvault update \
  --name <key-vault-name> \
  --bypass AzureServices

# Option 2: If using specific network rules, add ACR
# Reconfigure the managed identity and key
az acr encryption rotate-key \
  --name <registry-name> \
  --key-encryption-key <key-id> \
  --identity <identity-id>
```

### Verify Firewall Configuration

```azurecli
# Check current network rules
az keyvault show \
  --name <key-vault-name> \
  --query "properties.networkAcls"

# Expected output should show bypass for Azure services:
{
  "bypass": "AzureServices",
  "defaultAction": "Deny",
  ...
}
```

## Error: Identity Expiry

### Description

The identity attached to a registry experiences expiry issues, resulting in:

**Error Message:**
```
HTTP 403: The identity associated with the registry is inactive. This could be due to attempted removal of the identity.
```

### Background

- Identities attached to registries are set for auto-renewal
- Attempting to disassociate an identity can break auto-renewal
- After identity expiration (typically ~3 months), operations fail

### Resolution

```azurecli
# Manually reassign the identity to trigger renewal
az acr identity assign -n <registry-name> \
  --identities "/subscriptions/<subscription>/resourcegroups/<rg>/providers/Microsoft.ManagedIdentity/userAssignedIdentities/<identity-name>"
```

## Error: Accidental Key Vault or Key Deletion

### Description

The Key Vault or encryption key was accidentally deleted, making registry content inaccessible.

### Check Soft Delete Status

```azurecli
# List soft-deleted vaults
az keyvault list-deleted

# List soft-deleted keys
az keyvault key list-deleted --vault-name <key-vault-name>
```

### Recovery - Key Vault

```azurecli
# Recover deleted Key Vault
az keyvault recover --name <key-vault-name>

# Verify recovery
az keyvault show --name <key-vault-name>
```

### Recovery - Key

```azurecli
# Recover deleted key
az keyvault key recover \
  --name <key-name> \
  --vault-name <key-vault-name>

# Verify key is restored
az keyvault key show \
  --name <key-name> \
  --vault-name <key-vault-name>
```

### Prevention

Always enable purge protection:

```azurecli
az keyvault update \
  --name <key-vault-name> \
  --enable-purge-protection true
```

## Error: Key Version Mismatch

### Description

Registry is configured with a specific key version that no longer exists or has been rotated.

### Resolution

```azurecli
# Get current key versions
az keyvault key list-versions \
  --name <key-name> \
  --vault-name <key-vault-name>

# Rotate to the latest version
latestKeyID=$(az keyvault key show \
  --name <key-name> \
  --vault-name <key-vault-name> \
  --query 'key.kid' --output tsv)

az acr encryption rotate-key \
  --name <registry-name> \
  --key-encryption-key $latestKeyID \
  --identity <identity-id>
```

## Error: Invalid Key Type

### Description

Attempting to use an unsupported key type (e.g., Elliptic Curve keys).

**Error Message:**
```
Invalid key type. Azure Container Registry supports only RSA or RSA-HSM keys.
```

### Resolution

Create a new RSA key:

```azurecli
az keyvault key create \
  --name <new-key-name> \
  --vault-name <key-vault-name> \
  --kty RSA \
  --size 2048
```

## Troubleshooting Checklist

### Initial Diagnostics

- [ ] Run `az acr check-health --name <registry-name>`
- [ ] Check encryption status: `az acr encryption show --name <registry-name>`
- [ ] Verify Key Vault accessibility
- [ ] Check identity exists and has correct permissions

### Identity Issues

- [ ] Verify identity resource exists
- [ ] Check identity is assigned to registry
- [ ] Verify Key Vault access policy includes identity
- [ ] Check for RBAC role assignments if using RBAC model

### Key Vault Issues

- [ ] Verify Key Vault exists
- [ ] Check Key Vault firewall rules
- [ ] Verify trusted services are enabled
- [ ] Check Key Vault soft delete status
- [ ] Verify key exists and is enabled

### Network Issues

- [ ] Check private endpoint configurations
- [ ] Verify virtual network rules
- [ ] Test connectivity from ACR to Key Vault
- [ ] Check DNS resolution for Key Vault

### Key Issues

- [ ] Verify key type is RSA or RSA-HSM
- [ ] Check key version exists
- [ ] Verify key is not disabled
- [ ] Check key expiration date

## Logging and Monitoring

### Enable Key Vault Diagnostic Logging

```azurecli
az monitor diagnostic-settings create \
  --name "CMK-Diagnostics" \
  --resource <key-vault-resource-id> \
  --logs '[{"category": "AuditEvent", "enabled": true}]' \
  --metrics '[{"category": "AllMetrics", "enabled": true}]' \
  --workspace <log-analytics-workspace-id>
```

### Query Key Vault Logs

```kusto
AzureDiagnostics
| where ResourceProvider == "MICROSOFT.KEYVAULT"
| where OperationName contains "VaultAccessPolicy" or
        OperationName contains "KeyEncrypt" or
        OperationName contains "KeyDecrypt"
| project TimeGenerated, OperationName, ResultType, CallerIPAddress, identity_claim_oid_g
| order by TimeGenerated desc
```

### Set Up Alerts

```azurecli
# Alert for Key Vault access failures
az monitor metrics alert create \
  --name "KeyVault-CMK-Access-Failure" \
  --resource-group <resource-group-name> \
  --scopes <key-vault-resource-id> \
  --condition "total ServiceApiResult < 1 where ResultType includes 'Unauthorized'" \
  --action <action-group-id>
```

## Recovery Procedures Summary

| Issue | Immediate Action | Long-term Fix |
|-------|------------------|---------------|
| CMK_ERROR | Reassign identity | Review identity lifecycle management |
| 403 after firewall | Enable trusted services bypass | Configure proper network rules |
| Identity removal blocked | Reassign, rotate key, then remove | Plan identity changes carefully |
| Identity expired | Manually reassign identity | Monitor identity status |
| Key deleted | Recover from soft delete | Enable purge protection |
| Key Vault deleted | Recover vault | Enable purge protection |
| Wrong key type | Create RSA key | Document key requirements |

## Contact Support

If issues persist after following these troubleshooting steps:

1. Gather diagnostic information:
   - `az acr check-health --name <registry-name>`
   - `az acr encryption show --name <registry-name>`
   - Key Vault diagnostic logs

2. Create Azure support ticket:
   - Navigate to Azure Portal > Help + support
   - Create new support request
   - Service type: Azure Container Registry
   - Problem type: Customer-Managed Key

3. Provide:
   - Registry name and resource ID
   - Key Vault name and resource ID
   - Error messages and timestamps
   - Steps already attempted

## Source References

- `/submodules/azure-management-docs/articles/container-registry/tutorial-troubleshoot-customer-managed-keys.md`
- `/submodules/azure-management-docs/articles/container-registry/container-registry-health-error-reference.md`
- `/submodules/azure-management-docs/articles/container-registry/tutorial-rotate-revoke-customer-managed-keys.md`

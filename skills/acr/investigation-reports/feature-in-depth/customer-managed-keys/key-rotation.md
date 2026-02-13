# Key Rotation for Customer-Managed Keys in Azure Container Registry

## Overview

Key rotation is a critical security practice for maintaining the integrity and security of your customer-managed encryption keys. Azure Container Registry supports both automatic and manual key rotation methods.

## Rotation Methods

### Automatic Key Rotation (Recommended)

When a registry is encrypted with an **unversioned key** (no version specified in the key identifier), Azure Container Registry automatically detects and uses new key versions.

**Characteristics:**
- ACR checks Key Vault regularly for new key versions
- Updates occur automatically within approximately one hour
- No manual intervention required
- Recommended for most production scenarios

**Key Identifier Format (Unversioned):**
```
https://<vault-name>.vault.azure.net/keys/<key-name>
```

### Manual Key Rotation

When a registry is encrypted with a **versioned key** (specific version in the key identifier), you must manually update the registry to use new key versions.

**Characteristics:**
- Gives precise control over when rotation occurs
- Requires explicit action to update to new version
- Useful for regulated environments with change control processes

**Key Identifier Format (Versioned):**
```
https://<vault-name>.vault.azure.net/keys/<key-name>/<version>
```

## Automatic Rotation Setup

### Initial Configuration for Automatic Rotation

When creating a registry, use an unversioned key ID:

```azurecli
# Get key ID and strip the version
keyID=$(az keyvault key show \
  --name <key-name> \
  --vault-name <key-vault-name> \
  --query 'key.kid' --output tsv)

keyID=$(echo $keyID | sed -e "s/\/[^/]*$//")

# Create registry with unversioned key
az acr create \
  --resource-group <resource-group-name> \
  --name <container-registry-name> \
  --identity $identityID \
  --key-encryption-key $keyID \
  --sku Premium
```

### Creating a New Key Version (Triggers Automatic Update)

Simply create a new version of the existing key:

```azurecli
az keyvault key create \
  --name <key-name> \
  --vault-name <key-vault-name>
```

ACR will automatically detect and use the new version within one hour.

## Manual Rotation Procedures

### Using Azure CLI

You have three options for manual key rotation, depending on the identity you want to use:

#### Option 1: Rotate Key with Client ID of Managed Identity

```azurecli
az acr encryption rotate-key \
  --name <registry-name> \
  --key-encryption-key <new-key-id> \
  --identity <client-ID-of-managed-identity>
```

#### Option 2: Rotate Key with User-Assigned Identity

First, ensure the user-assigned identity has the required permissions (`get`, `wrap`, `unwrap`) on the Key Vault.

```azurecli
az acr encryption rotate-key \
  --name <registry-name> \
  --key-encryption-key <new-key-id> \
  --identity <resource-id-of-user-assigned-identity>
```

#### Option 3: Rotate Key with System-Assigned Identity

First, ensure the system-assigned identity is enabled and has the required permissions.

```azurecli
az acr encryption rotate-key \
  --name <registry-name> \
  --key-encryption-key <new-key-id> \
  --identity [system]
```

### Using Azure Portal

1. Navigate to your container registry in the Azure Portal
2. Under **Settings**, select **Encryption**
3. Click **Change key**
4. Choose one of the following:
   - **Select from Key Vault**: Browse to select an existing key vault and key
   - **Enter key URI**: Provide a key identifier directly
5. Select the key type:
   - **Unversioned key**: Enables automatic rotation
   - **Versioned key**: Requires manual rotation
6. Click **Save**

## Rotation Scenarios

### Scenario 1: Routine Key Rotation

For regular security maintenance:

```azurecli
# Step 1: Create new key version
az keyvault key create \
  --name <key-name> \
  --vault-name <key-vault-name>

# Step 2: If using versioned key, update registry
# Get the new key ID
newKeyID=$(az keyvault key show \
  --name <key-name> \
  --vault-name <key-vault-name> \
  --query 'key.kid' --output tsv)

# Rotate to new version
az acr encryption rotate-key \
  --name <registry-name> \
  --key-encryption-key $newKeyID \
  --identity <identity-id>

# Step 3: Verify rotation
az acr encryption show --name <registry-name>
```

### Scenario 2: Rotating to a Different Key Vault

When moving to a new Key Vault:

```azurecli
# Step 1: Ensure identity has access to new Key Vault
az keyvault set-policy \
  --name <new-key-vault-name> \
  --resource-group <resource-group-name> \
  --object-id <identity-principal-id> \
  --key-permissions get unwrapKey wrapKey

# Step 2: Create key in new Key Vault
az keyvault key create \
  --name <key-name> \
  --vault-name <new-key-vault-name>

# Step 3: Get new key ID
newKeyID=$(az keyvault key show \
  --name <key-name> \
  --vault-name <new-key-vault-name> \
  --query 'key.kid' --output tsv)

# Step 4: Rotate registry to new key
az acr encryption rotate-key \
  --name <registry-name> \
  --key-encryption-key $newKeyID \
  --identity <identity-id>
```

### Scenario 3: Changing Identity During Rotation

To switch to a different managed identity:

```azurecli
# Step 1: Ensure new identity has Key Vault access
az keyvault set-policy \
  --name <key-vault-name> \
  --resource-group <resource-group-name> \
  --object-id <new-identity-principal-id> \
  --key-permissions get unwrapKey wrapKey

# Step 2: Rotate key with new identity
az acr encryption rotate-key \
  --name <registry-name> \
  --key-encryption-key <key-id> \
  --identity <new-identity-resource-id>
```

### Scenario 4: Switching from Manual to Automatic Rotation

To enable automatic rotation on an existing versioned configuration:

```azurecli
# Get the unversioned key ID
keyID=$(az keyvault key show \
  --name <key-name> \
  --vault-name <key-vault-name> \
  --query 'key.kid' --output tsv)

# Strip the version
unversionedKeyID=$(echo $keyID | sed -e "s/\/[^/]*$//")

# Update registry to use unversioned key
az acr encryption rotate-key \
  --name <registry-name> \
  --key-encryption-key $unversionedKeyID \
  --identity <identity-id>
```

## Verifying Key Rotation

### Check Current Encryption Status

```azurecli
az acr encryption show --name <registry-name>
```

Example output after rotation:

```json
{
  "keyVaultProperties": {
    "identity": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
    "keyIdentifier": "https://myvault.vault.azure.net/keys/mykey",
    "keyRotationEnabled": true,
    "lastKeyRotationTimestamp": "2024-01-15T10:30:00Z",
    "versionedKeyIdentifier": "https://myvault.vault.azure.net/keys/mykey/abc123..."
  },
  "status": "enabled"
}
```

### Verify Using Azure Portal

1. Go to your container registry
2. Under **Settings**, select **Encryption**
3. Check:
   - Key identifier matches expected value
   - Key rotation status (enabled/disabled)
   - Last rotation timestamp

## Key Rotation Best Practices

### 1. Establish Rotation Policy

- Define rotation frequency based on compliance requirements
- Document procedures for different rotation scenarios
- Create runbooks for emergency rotation

### 2. Use Automatic Rotation When Possible

- Reduces operational overhead
- Ensures timely rotation
- Minimizes human error

### 3. Monitor Key Usage

Enable Key Vault diagnostic logging:

```azurecli
az monitor diagnostic-settings create \
  --name <diagnostic-setting-name> \
  --resource <key-vault-resource-id> \
  --logs '[{"category": "AuditEvent", "enabled": true}]' \
  --workspace <log-analytics-workspace-id>
```

### 4. Test Rotation Procedures

- Regularly test rotation in non-production environments
- Verify registry operations work after rotation
- Document any issues encountered

### 5. Maintain Key Version History

- Keep track of which key versions were used
- Retain old key versions until no longer needed
- Document reasons for rotation

## Rotation Timeline

```
Automatic Rotation:
+------------------+     +------------------+     +------------------+
| New key version  |---->| ACR detects new  |---->| Registry uses    |
| created in KV    |     | version (~1 hr)  |     | new key version  |
+------------------+     +------------------+     +------------------+

Manual Rotation:
+------------------+     +------------------+     +------------------+
| New key version  |---->| Admin runs       |---->| Registry uses    |
| created in KV    |     | rotate-key cmd   |     | new key version  |
+------------------+     +------------------+     +------------------+
                                |
                                | immediate
```

## Troubleshooting Rotation Issues

### Issue: Rotation Command Fails with 403

**Cause:** Identity lacks Key Vault permissions

**Solution:**
```azurecli
az keyvault set-policy \
  --name <key-vault-name> \
  --object-id <identity-principal-id> \
  --key-permissions get unwrapKey wrapKey
```

### Issue: Automatic Rotation Not Occurring

**Causes:**
- Registry configured with versioned key
- Key Vault firewall blocking access
- Identity permissions revoked

**Solution:**
1. Check if using versioned key ID
2. Verify Key Vault firewall settings
3. Confirm identity permissions

### Issue: Registry Operations Fail After Rotation

**Cause:** New key version not accessible

**Solution:**
1. Verify new key exists in Key Vault
2. Check identity has access to new key
3. If using different Key Vault, verify trusted services enabled

## Source References

- `/submodules/azure-management-docs/articles/container-registry/tutorial-rotate-revoke-customer-managed-keys.md`
- `/submodules/azure-management-docs/articles/container-registry/tutorial-enable-customer-managed-keys.md`
- `/submodules/azure-management-docs/articles/container-registry/tutorial-customer-managed-keys.md`

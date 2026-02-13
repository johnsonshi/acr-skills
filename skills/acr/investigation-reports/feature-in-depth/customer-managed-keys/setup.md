# Setting Up Customer-Managed Keys for Azure Container Registry

## Overview

This guide provides comprehensive instructions for enabling Customer-Managed Keys (CMK) on Azure Container Registry using the Azure CLI, Azure Portal, and Azure Resource Manager templates.

## Prerequisites

Before enabling CMK, ensure you have:

1. **Azure Subscription** with permissions to create resources
2. **Azure CLI** version 2.0.68 or later (for CLI method)
3. **Premium SKU Requirement** - CMK is only available for Premium registries
4. **Key Type Support** - Only RSA or RSA-HSM keys are supported (not EC keys)

## Method 1: Azure CLI

### Step 1: Create a Resource Group

```azurecli
az group create --name <resource-group-name> --location <location>
```

### Step 2: Create a User-Assigned Managed Identity

```azurecli
az identity create \
  --resource-group <resource-group-name> \
  --name <managed-identity-name>
```

Save the output values for later use:

```json
{
  "clientId": "00001111-aaaa-2222-bbbb-3333cccc4444",
  "id": "/subscriptions/.../providers/Microsoft.ManagedIdentity/userAssignedIdentities/myidentity",
  "principalId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
  ...
}
```

Store values in environment variables:

```azurecli
identityID=$(az identity show --resource-group <resource-group-name> --name <managed-identity-name> --query 'id' --output tsv)

identityPrincipalID=$(az identity show --resource-group <resource-group-name> --name <managed-identity-name> --query 'principalId' --output tsv)
```

### Step 3: Create an Azure Key Vault

Create a Key Vault with purge protection enabled (recommended):

```azurecli
az keyvault create --name <key-vault-name> \
  --resource-group <resource-group-name> \
  --enable-purge-protection
```

Store the Key Vault resource ID:

```azurecli
keyvaultID=$(az keyvault show --resource-group <resource-group-name> --name <key-vault-name> --query 'id' --output tsv)
```

### Step 4: Configure Key Vault Access

#### Option A: Access Policy Method

```azurecli
az keyvault set-policy \
  --resource-group <resource-group-name> \
  --name <key-vault-name> \
  --object-id $identityPrincipalID \
  --key-permissions get unwrapKey wrapKey
```

#### Option B: Azure RBAC Method

```azurecli
az role assignment create --assignee $identityPrincipalID \
  --role "Key Vault Crypto Service Encryption User" \
  --scope $keyvaultID
```

### Step 5: Enable Trusted Services (If Firewall Enabled)

If your Key Vault has firewall or VNet restrictions:

```azurecli
az keyvault update --name <key-vault-name> \
  --resource-group <resource-group-name> \
  --bypass AzureServices
```

### Step 6: Create the Encryption Key

```azurecli
az keyvault key create \
  --name <key-name> \
  --vault-name <key-vault-name>
```

### Step 7: Get the Key ID

#### For Automatic Key Rotation (Recommended)

Use an unversioned key ID (removes the version component):

```azurecli
keyID=$(az keyvault key show \
  --name <key-name> \
  --vault-name <key-vault-name> \
  --query 'key.kid' --output tsv)

keyID=$(echo $keyID | sed -e "s/\/[^/]*$//")
```

#### For Manual Key Rotation

Use a versioned key ID:

```azurecli
keyID=$(az keyvault key show \
  --name <key-name> \
  --vault-name <key-vault-name> \
  --query 'key.kid' --output tsv)
```

### Step 8: Create the Registry with CMK

```azurecli
az acr create \
  --resource-group <resource-group-name> \
  --name <container-registry-name> \
  --identity $identityID \
  --key-encryption-key $keyID \
  --sku Premium
```

### Step 9: Verify Encryption Status

```azurecli
az acr encryption show --name <container-registry-name>
```

Expected output:

```json
{
  "keyVaultProperties": {
    "identity": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
    "keyIdentifier": "https://myvault.vault.azure.net/keys/mykey/...",
    "keyRotationEnabled": true,
    "lastKeyRotationTimestamp": "...",
    "versionedKeyIdentifier": "https://myvault.vault.azure.net/keys/mykey/version..."
  },
  "status": "enabled"
}
```

---

## Method 2: Azure Portal

### Step 1: Create a User-Assigned Managed Identity

1. Navigate to the Azure Portal
2. Search for "Managed Identities"
3. Click **+ Create**
4. Fill in:
   - Subscription
   - Resource group
   - Region
   - Name
5. Click **Review + create** > **Create**
6. Note the identity name for later steps

### Step 2: Create a Key Vault

1. Search for "Key vaults" in the portal
2. Click **+ Create**
3. On the **Basics** tab:
   - Select subscription and resource group
   - Enter Key vault name
   - Select region
   - **Important**: Enable **Purge protection**
4. Click **Review + create** > **Create**

### Step 3: Configure Key Vault Access Policy

1. Go to your Key Vault
2. Select **Settings** > **Access policies**
3. Click **+ Add Access Policy**
4. Key permissions: Select **Get**, **Unwrap Key**, **Wrap Key**
5. Select principal: Choose your user-assigned managed identity
6. Click **Add**
7. Click **Save**

### Step 4: Create a Key

1. Go to your Key Vault
2. Select **Settings** > **Keys**
3. Click **+ Generate/Import**
4. Enter a unique name for the key
5. Key type: RSA (or RSA-HSM)
6. Accept other defaults
7. Click **Create**
8. After creation, click the key, then click the current version
9. Copy the **Key identifier** for later use

### Step 5: Create the Container Registry

1. Search for "Container registries"
2. Click **+ Create**
3. **Basics** tab:
   - Select subscription and resource group
   - Enter registry name
   - Select location
   - **SKU**: Select **Premium**
4. **Encryption** tab:
   - Customer-managed key: **Enabled**
   - Identity: Select your managed identity
   - Encryption key: Choose one:
     - **Select from Key Vault**: Browse to your key vault and key
     - **Enter key URI**: Paste the key identifier copied earlier
5. Click **Review + create** > **Create**

### Step 6: Verify Encryption Status

1. Go to your container registry
2. Under **Settings**, select **Encryption**
3. Verify the encryption status shows "Enabled"

---

## Method 3: Azure Resource Manager Template

### Template File (CMKtemplate.json)

```json
{
  "$schema": "https://schema.management.azure.com/schemas/2015-01-01/deploymentTemplate.json#",
  "contentVersion": "1.0.0.0",
  "parameters": {
    "vault_name": {
      "defaultValue": "",
      "type": "String"
    },
    "registry_name": {
      "defaultValue": "",
      "type": "String"
    },
    "identity_name": {
      "defaultValue": "",
      "type": "String"
    },
    "kek_id": {
      "type": "String"
    }
  },
  "variables": {},
  "resources": [
    {
      "type": "Microsoft.ContainerRegistry/registries",
      "apiVersion": "2019-12-01-preview",
      "name": "[parameters('registry_name')]",
      "location": "[resourceGroup().location]",
      "sku": {
        "name": "Premium",
        "tier": "Premium"
      },
      "identity": {
        "type": "UserAssigned",
        "userAssignedIdentities": {
          "[resourceID('Microsoft.ManagedIdentity/userAssignedIdentities', parameters('identity_name'))]": {}
        }
      },
      "dependsOn": [
        "[resourceId('Microsoft.ManagedIdentity/userAssignedIdentities', parameters('identity_name'))]"
      ],
      "properties": {
        "adminUserEnabled": false,
        "encryption": {
          "status": "enabled",
          "keyVaultProperties": {
            "identity": "[reference(resourceId('Microsoft.ManagedIdentity/userAssignedIdentities', parameters('identity_name')), '2023-07-31').clientId]",
            "KeyIdentifier": "[parameters('kek_id')]"
          }
        },
        "networkRuleSet": {
          "defaultAction": "Allow",
          "virtualNetworkRules": [],
          "ipRules": []
        },
        "policies": {
          "quarantinePolicy": {
            "status": "disabled"
          },
          "trustPolicy": {
            "type": "Notary",
            "status": "disabled"
          },
          "retentionPolicy": {
            "days": 7,
            "status": "disabled"
          }
        }
      }
    },
    {
      "type": "Microsoft.KeyVault/vaults/accessPolicies",
      "apiVersion": "2023-07-01",
      "name": "[concat(parameters('vault_name'), '/add')]",
      "dependsOn": [
        "[resourceId('Microsoft.ManagedIdentity/userAssignedIdentities', parameters('identity_name'))]"
      ],
      "properties": {
        "accessPolicies": [
          {
            "tenantId": "[subscription().tenantId]",
            "objectId": "[reference(resourceId('Microsoft.ManagedIdentity/userAssignedIdentities', parameters('identity_name')), '2023-07-01').principalId]",
            "permissions": {
              "keys": [
                "get",
                "unwrapKey",
                "wrapKey"
              ]
            }
          }
        ]
      }
    },
    {
      "type": "Microsoft.ManagedIdentity/userAssignedIdentities",
      "apiVersion": "2023-07-01",
      "name": "[parameters('identity_name')]",
      "location": "[resourceGroup().location]"
    }
  ]
}
```

### Deploy the Template

Prerequisites:
1. Create a Key Vault with a key already in place
2. Note the Key Vault name and key ID

Deploy command:

```azurecli
az deployment group create \
  --resource-group <resource-group-name> \
  --template-file CMKtemplate.json \
  --parameters \
    registry_name=<registry-name> \
    identity_name=<managed-identity-name> \
    vault_name=<key-vault-name> \
    kek_id=<key-vault-key-id>
```

Verify deployment:

```azurecli
az acr encryption show --name <registry-name>
```

---

## Complete Setup Checklist

- [ ] Resource group created
- [ ] User-assigned managed identity created
- [ ] Key Vault created with purge protection enabled
- [ ] Key Vault access configured (Access Policy or RBAC)
- [ ] Trusted services enabled (if firewall is used)
- [ ] RSA or RSA-HSM key created
- [ ] Key ID captured (versioned or unversioned)
- [ ] Premium registry created with CMK enabled
- [ ] Encryption status verified

## Post-Setup Tasks

1. **Enable Logging**: Configure Key Vault diagnostic settings for audit logging
2. **Set Up Alerts**: Create alerts for key access failures
3. **Document Configuration**: Record identity and key mappings
4. **Plan Key Rotation**: Establish key rotation procedures
5. **Test Recovery**: Document and test recovery procedures

## Common Setup Issues

### Issue: 403 Forbidden from Key Vault

**Causes:**
- Identity doesn't have correct permissions
- Key Vault firewall blocking access
- Trusted services not enabled

**Solution:**
```azurecli
# Check access policy
az keyvault show --name <vault-name> --query "properties.accessPolicies"

# Enable trusted services
az keyvault update --name <vault-name> --bypass AzureServices
```

### Issue: Identity Not Found

**Cause:** User-assigned identity was deleted or name misspelled

**Solution:**
```azurecli
# Verify identity exists
az identity show --resource-group <rg> --name <identity-name>
```

### Issue: Invalid Key Type

**Cause:** Attempting to use EC (elliptic curve) key

**Solution:** Create a new RSA or RSA-HSM key

## Source References

- `/submodules/azure-management-docs/articles/container-registry/tutorial-enable-customer-managed-keys.md`
- `/submodules/azure-management-docs/articles/container-registry/tutorial-customer-managed-keys.md`

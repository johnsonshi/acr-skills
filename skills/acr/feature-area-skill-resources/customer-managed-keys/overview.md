# ACR Customer-Managed Keys Skill

This skill provides comprehensive knowledge about Azure Container Registry customer-managed key encryption.

## When to Use This Skill

Use this skill when answering questions about:
- Encryption at rest
- Azure Key Vault integration
- Key rotation
- CMK configuration

## Overview

Customer-managed keys (CMK) provide additional encryption control using keys you manage in Azure Key Vault.

**Requirements:** Premium SKU

## Quick Setup

### 1. Create Key Vault and Key
```bash
# Create Key Vault
az keyvault create \
  --name myvault \
  --resource-group myRG \
  --enable-purge-protection

# Create RSA key
az keyvault key create \
  --vault-name myvault \
  --name mykey \
  --kty RSA \
  --size 2048
```

### 2. Create User-Assigned Identity
```bash
az identity create \
  --name myidentity \
  --resource-group myRG
```

### 3. Grant Key Vault Access
```bash
# Get identity principal ID
PRINCIPAL_ID=$(az identity show --name myidentity --resource-group myRG --query principalId -o tsv)

# Assign Key Vault roles
az role assignment create \
  --assignee $PRINCIPAL_ID \
  --scope $(az keyvault show --name myvault --query id -o tsv) \
  --role "Key Vault Crypto User"

az role assignment create \
  --assignee $PRINCIPAL_ID \
  --scope $(az keyvault show --name myvault --query id -o tsv) \
  --role "Key Vault Crypto Service Encryption User"
```

### 4. Create CMK Registry
```bash
az acr create \
  --name myregistry \
  --resource-group myRG \
  --sku Premium \
  --identity $(az identity show --name myidentity --resource-group myRG --query id -o tsv) \
  --key-encryption-key https://myvault.vault.azure.net/keys/mykey
```

## Architecture

```
┌─────────────────┐      ┌─────────────────┐
│   ACR Registry  │◄────▶│  Azure Key Vault│
│                 │      │                 │
│  ┌───────────┐  │      │  ┌───────────┐  │
│  │  DEK      │  │      │  │  KEK      │  │
│  │(Data Key) │  │      │  │(Key Key)  │  │
│  └───────────┘  │      │  └───────────┘  │
└─────────────────┘      └─────────────────┘
         │
         ▼
┌─────────────────┐
│  Encrypted      │
│  Storage        │
└─────────────────┘
```

## Key Rotation

### Automatic Rotation
```bash
# Enable auto-rotation (Key Vault)
az keyvault key rotation-policy update \
  --vault-name myvault \
  --name mykey \
  --value @rotation-policy.json
```

### Manual Rotation
```bash
# Create new key version
az keyvault key create \
  --vault-name myvault \
  --name mykey \
  --kty RSA

# Update registry to use new version
az acr encryption rotate-key \
  --name myregistry \
  --key-encryption-key https://myvault.vault.azure.net/keys/mykey
```

## Key Revocation

### Temporary Lockdown
```bash
# Remove identity's Key Vault access
az role assignment delete \
  --assignee $PRINCIPAL_ID \
  --scope $(az keyvault show --name myvault --query id -o tsv) \
  --role "Key Vault Crypto User"
```

### Recovery
```bash
# Restore access
az role assignment create \
  --assignee $PRINCIPAL_ID \
  --scope $(az keyvault show --name myvault --query id -o tsv) \
  --role "Key Vault Crypto User"
```

## Requirements

| Requirement | Detail |
|-------------|--------|
| SKU | Premium only |
| Key Type | RSA or RSA-HSM |
| Key Size | 2048, 3072, or 4096 bits |
| Identity | User-assigned managed identity |
| Key Vault | Purge protection enabled |

## Limitations

| Feature | CMK Compatible |
|---------|----------------|
| Content Trust (DCT) | ❌ |
| Geo-replication | ✅ |
| Private endpoints | ✅ |
| Zone redundancy | ✅ |
| Task logs | 24-hour retention only |

## Troubleshooting

### CMK_ERROR
```bash
# Check encryption status
az acr show --name myregistry --query encryption

# Verify Key Vault access
az keyvault key show --vault-name myvault --name mykey
```

### Identity Issues
```bash
# Verify identity assignment
az acr show --name myregistry --query identity
```

## Reference Documentation

- **Investigation reports**: `agent-investigation-reports/acr-features/customer-managed-keys/`
- **MS Learn docs**: `azure-management-docs/articles/container-registry/tutorial-customer-managed-keys.md`

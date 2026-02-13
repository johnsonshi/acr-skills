# Customer-Managed Keys (CMK) for Azure Container Registry

## Overview

Customer-Managed Keys (CMK) is a Premium-tier feature in Azure Container Registry that allows organizations to encrypt registry content using their own encryption keys stored in Azure Key Vault. This provides an additional layer of encryption on top of Azure's default service-managed encryption.

## Key Features

### 1. Dual-Layer Encryption
- Azure automatically encrypts all registry content at rest using service-managed keys
- CMK adds a supplementary encryption layer controlled entirely by the customer
- Both encryption layers work together to protect container images and artifacts

### 2. Azure Key Vault Integration
- Customer-managed keys are stored and managed in Azure Key Vault
- Supports server-side encryption through seamless Key Vault integration
- Customers can create their own encryption keys or use Azure Key Vault APIs to generate keys
- Full audit trail of key usage through Key Vault logging

### 3. Key Lifecycle Management
- Complete control and responsibility for key lifecycle
- Supports both automatic and manual key rotation
- Ability to revoke keys to immediately block registry access
- Integration with Key Vault's soft delete and purge protection features

### 4. Regulatory Compliance
- Helps meet guidelines for regulatory compliance (HIPAA, PCI-DSS, etc.)
- Provides demonstrable control over encryption keys for audit purposes
- Supports compliance requirements that mandate customer-controlled encryption

## Requirements and Prerequisites

### Service Tier
- **Premium SKU required** - CMK is only available in the Premium service tier
- Cannot be used with Basic or Standard registries

### Identity Requirements
- User-assigned managed identity must be configured initially
- System-assigned managed identity can be enabled later for key vault access
- Identity requires specific Key Vault permissions: `get`, `unwrapKey`, `wrapKey`

### Key Requirements
- Only RSA or RSA-HSM keys are supported
- Elliptic-curve (EC) keys are NOT supported
- Key must be stored in Azure Key Vault

### Timing Constraints
- CMK can only be enabled during registry creation
- Cannot be enabled on existing registries
- Cannot be disabled once enabled on a registry

## Limitations and Considerations

### Feature Incompatibilities
1. **Content Trust**: Not supported in CMK-encrypted registries
2. **ACR Tasks Log Retention**: Limited to 24 hours for task run logs
   - For longer retention, configure alternative log storage

### Operational Considerations
1. **Immutable Configuration**: Once enabled, encryption cannot be disabled
2. **Key Availability**: Registry availability depends on key accessibility
3. **Identity Dependency**: Registry operations require the managed identity to remain functional

## How It Works

### Encryption Flow
1. When content is pushed to the registry:
   - Azure first encrypts data with service-managed keys
   - CMK encryption is applied as an additional layer
   - Encrypted data is stored in Azure-managed storage

2. When content is pulled from the registry:
   - Registry uses managed identity to access the key in Key Vault
   - CMK decryption occurs first
   - Service-managed key decryption follows
   - Decrypted content is delivered to the client

### Key Access Pattern
```
Registry --> Managed Identity --> Key Vault --> Encryption Key
                |                     |
                |                     +--> Audit Logs
                |
                +--> Access Policy or RBAC permissions
```

## Integration with Other ACR Features

### Geo-Replication
- CMK-encrypted registries can be geo-replicated
- Each replica uses the same customer-managed key
- For high availability, plan for Key Vault failover and redundancy
- Reference: [Key Vault disaster recovery guidance](/azure/key-vault/general/disaster-recovery-guidance)

### Private Endpoints
- CMK is fully compatible with private endpoint configurations
- Key Vault can also be accessed via private endpoints
- Trusted services must be enabled if Key Vault has firewall restrictions

### Azure Policy
- Built-in policy: "Container Registries should be encrypted with a Customer-Managed Key (CMK)"
- Can audit and enforce CMK compliance across subscriptions
- Policy ID available via `az policy assignment list`

## Health Check Integration

The `az acr check-health` command includes CMK-specific health checks:

### CMK_ERROR
- **Meaning**: Registry cannot access the managed identity used for CMK configuration
- **Common Cause**: Managed identity was deleted or permissions were revoked
- **Solution**: Reassign the identity or troubleshoot identity access issues

## Best Practices

1. **Enable Purge Protection**: Always enable purge protection on the Key Vault to prevent accidental key deletion
2. **Use Automatic Rotation**: Configure automatic key rotation by omitting the key version
3. **Monitor Key Access**: Enable Key Vault logging and set up alerts for access failures
4. **Plan for DR**: Implement Key Vault redundancy and failover strategies
5. **Document Identity Mappings**: Maintain documentation of which identities access which key vaults
6. **Regular Rotation**: Follow organizational policies for key rotation frequency

## Related Documentation

- [Enable Customer-Managed Keys](./setup.md)
- [Key Rotation](./key-rotation.md)
- [Revocation](./revocation.md)
- [Troubleshooting](./troubleshooting.md)
- [Architecture Details](./architecture.md)

## Source References

- `/submodules/azure-management-docs/articles/container-registry/tutorial-customer-managed-keys.md`
- `/submodules/azure-management-docs/articles/container-registry/container-registry-storage.md`
- `/submodules/azure-management-docs/articles/container-registry/container-registry-azure-policy.md`
- `/submodules/azure-management-docs/articles/container-registry/container-registry-health-error-reference.md`

# Azure Key Vault Integration for Signing

## Overview

Azure Key Vault provides secure, centralized storage for signing keys and certificates used with Notation. The Key Vault plug-in (`notation-azure-kv`) enables Notation to sign and verify container images using keys stored in Key Vault without exposing private keys.

## Benefits

- **Secure Key Storage**: Private keys never leave Key Vault
- **Access Control**: Azure RBAC and access policies for fine-grained permissions
- **Audit Logging**: Track key usage through Key Vault diagnostics
- **Certificate Lifecycle**: Manage certificate creation, rotation, and renewal
- **Compliance**: Meet regulatory requirements for key management

## Prerequisites

- Azure Key Vault instance
- Notation CLI installed
- Azure CLI configured
- Appropriate Azure RBAC roles or access policies

## Installing the Key Vault Plug-in

```bash
# Install Key Vault plug-in v1.2.1
notation plugin install --url https://github.com/Azure/notation-azure-kv/releases/download/v1.2.1/notation-azure-kv_1.2.1_linux_amd64.tar.gz --sha256sum 67c5ccaaf28dd44d2b6572684d84e344a02c2258af1d65ead3910b3156d3eaf5

# Verify installation
notation plugin ls
```

**Expected Output:**
```
NAME        DESCRIPTION                                        VERSION   CAPABILITIES                ERROR
azure-kv    Sign artifacts with keys in Azure Key Vault        1.2.1     [SIGNATURE_GENERATOR.RAW]   <nil>
```

## Access Control

### Azure RBAC (Recommended)

For signing with **self-signed certificates**:
| Role | Purpose |
|------|---------|
| `Key Vault Certificates Officer` | Creating and reading certificates |
| `Key Vault Certificates User` | Reading existing certificates |
| `Key Vault Crypto User` | Signing operations |

For signing with **CA-issued certificates** (with full chain):
| Role | Purpose |
|------|---------|
| `Key Vault Secrets User` | Reading secrets (for certificate chain) |
| `Key Vault Certificates User` | Reading certificates |
| `Key Vault Crypto User` | Signing operations |

For signing with **CA-issued certificates** (without chain):
| Role | Purpose |
|------|---------|
| `Key Vault Certificates User` | Reading certificates |
| `Key Vault Crypto User` | Signing operations |

**Assign Roles:**
```bash
# Set subscription
az account set --subscription $AKV_SUB_ID

# Assign roles for self-signed certificates
USER_ID=$(az ad signed-in-user show --query id -o tsv)
az role assignment create --role "Key Vault Certificates Officer" --assignee $USER_ID --scope "/subscriptions/$AKV_SUB_ID/resourceGroups/$AKV_RG/providers/Microsoft.KeyVault/vaults/$AKV_NAME"
az role assignment create --role "Key Vault Crypto User" --assignee $USER_ID --scope "/subscriptions/$AKV_SUB_ID/resourceGroups/$AKV_RG/providers/Microsoft.KeyVault/vaults/$AKV_NAME"
```

### Access Policies (Legacy)

For self-signed certificates:
```bash
USER_ID=$(az ad signed-in-user show --query id -o tsv)
az keyvault set-policy -n $AKV_NAME \
    --certificate-permissions create get \
    --key-permissions sign \
    --object-id $USER_ID
```

For CA-issued certificates with full chain:
```bash
az keyvault set-policy -n $AKV_NAME \
    --key-permissions sign \
    --secret-permissions get \
    --certificate-permissions get \
    --object-id $USER_ID
```

## Certificate Configuration

### Certificate Requirements

Certificates must meet Notary Project requirements:

**X.509 Properties:**
- Subject: Must contain `CN`, `C`, `ST`, `O`
- Key Usage: `DigitalSignature` only
- Extended Key Usages: Empty or `1.3.6.1.5.5.7.3.3` (code signing)

**Key Properties:**
- Exportable: `false`
- Key Type: RSA 2048/3072/4096 or EC P-256/P-384/P-521
- Reuse Key: `true` (recommended)

**Content Type:**
- Use `application/x-pem-file` for Image Integrity integration

### Creating a Self-Signed Certificate

```bash
# Set variables
CERT_NAME=wabbit-networks-io
CERT_SUBJECT="CN=wabbit-networks.io,O=Notation,L=Seattle,ST=WA,C=US"

# Create certificate policy file
cat <<EOF > ./my_policy.json
{
    "issuerParameters": {
        "certificateTransparency": null,
        "name": "Self"
    },
    "keyProperties": {
        "exportable": false,
        "keySize": 2048,
        "keyType": "RSA",
        "reuseKey": true
    },
    "secretProperties": {
        "contentType": "application/x-pem-file"
    },
    "x509CertificateProperties": {
        "ekus": ["1.3.6.1.5.5.7.3.3"],
        "keyUsage": ["digitalSignature"],
        "subject": "$CERT_SUBJECT",
        "validityInMonths": 12
    }
}
EOF

# Create the certificate
az keyvault certificate create -n $CERT_NAME --vault-name $AKV_NAME -p @my_policy.json
```

### Creating a Certificate Signing Request (CSR)

For CA-issued certificates, create a CSR:

```bash
# Create CSR policy
cat <<EOF > ./csr_policy.json
{
    "issuerParameters": {
        "name": "Unknown"
    },
    "keyProperties": {
        "exportable": false,
        "keySize": 2048,
        "keyType": "RSA",
        "reuseKey": true
    },
    "secretProperties": {
        "contentType": "application/x-pem-file"
    },
    "x509CertificateProperties": {
        "ekus": ["1.3.6.1.5.5.7.3.3"],
        "keyUsage": ["digitalSignature"],
        "subject": "CN=myapp.example.com,O=MyOrg,L=Seattle,ST=WA,C=US"
    }
}
EOF

# Create certificate operation (generates CSR)
az keyvault certificate create -n $CERT_NAME --vault-name $AKV_NAME -p @csr_policy.json

# Get the CSR
az keyvault certificate pending show -n $CERT_NAME --vault-name $AKV_NAME --query csr -o tsv > mycsr.csr
```

Submit the CSR to your CA, then merge the signed certificate:
```bash
az keyvault certificate pending merge -n $CERT_NAME --vault-name $AKV_NAME --file signed_cert.cer
```

### Importing an Existing Certificate

```bash
az keyvault certificate import -n $CERT_NAME --vault-name $AKV_NAME --file certificate.pfx
```

**Important:** When importing, ensure the certificate includes the full chain for verification.

## Signing with Key Vault

### Get the Signing Key ID

```bash
# Get latest version of certificate's key ID
KEY_ID=$(az keyvault certificate show -n $CERT_NAME --vault-name $AKV_NAME --query 'kid' -o tsv)
```

### Sign with Self-Signed Certificate

```bash
notation sign --signature-format cose \
    --id $KEY_ID \
    --plugin azure-kv \
    --plugin-config self_signed=true \
    $IMAGE
```

### Sign with CA-Issued Certificate (Full Chain)

```bash
notation sign --signature-format cose \
    --id $KEY_ID \
    --plugin azure-kv \
    $IMAGE
```

### Sign with CA-Issued Certificate (Separate Chain)

```bash
# Provide CA bundle separately
notation sign --signature-format cose \
    --id $KEY_ID \
    --plugin azure-kv \
    --plugin-config ca_certs=/path/to/ca_bundle.pem \
    $IMAGE
```

## Authentication Methods

The Key Vault plug-in supports multiple authentication methods, tried in order:

| Priority | Credential Type | Plugin Config | Use Case |
|----------|----------------|---------------|----------|
| 1 | Environment | `environment` | CI/CD with service principal |
| 2 | Workload Identity | `workloadid` | Kubernetes pods |
| 3 | Managed Identity | `managedid` | Azure VMs, App Service |
| 4 | Azure CLI | `azurecli` | Interactive development |

### Specify Credential Type

```bash
# Force Azure CLI credential
notation sign --signature-format cose \
    --id $KEY_ID \
    --plugin azure-kv \
    --plugin-config credential_type=azurecli \
    $IMAGE

# Use managed identity
notation sign --signature-format cose \
    --id $KEY_ID \
    --plugin azure-kv \
    --plugin-config credential_type=managedid \
    $IMAGE
```

## Verifying Signatures

### Download Certificate for Trust Store

```bash
# Download public certificate
az keyvault certificate download \
    --name $CERT_NAME \
    --vault-name $AKV_NAME \
    --file $CERT_PATH
```

### Add to Trust Store

```bash
# Add to CA trust store
notation cert add --type ca --store wabbit-networks.io $CERT_PATH

# Verify
notation cert ls
```

### Configure Trust Policy

```bash
cat <<EOF > ./trustpolicy.json
{
    "version": "1.0",
    "trustPolicies": [
        {
            "name": "wabbit-networks-images",
            "registryScopes": [ "$REGISTRY/$REPO" ],
            "signatureVerification": {
                "level" : "strict"
            },
            "trustStores": [ "ca:wabbit-networks.io" ],
            "trustedIdentities": [
                "x509.subject: CN=wabbit-networks.io,O=Notation,L=Seattle,ST=WA,C=US"
            ]
        }
    ]
}
EOF

notation policy import ./trustpolicy.json
```

### Verify Image

```bash
notation verify $IMAGE
```

## Certificate Rotation

When certificates expire or need rotation:

1. **Create new certificate** in Key Vault
2. **Get new key ID** for the latest version
3. **Re-sign images** with new certificate
4. **Update trust stores** on verification side with new root/CA

```bash
# Key Vault automatically provides latest certificate version
KEY_ID=$(az keyvault certificate show -n $CERT_NAME --vault-name $AKV_NAME --query 'kid' -o tsv)
```

## Timestamping

Enable timestamping to extend trust beyond certificate validity:

```bash
notation sign --signature-format cose \
    --id $KEY_ID \
    --plugin azure-kv \
    --timestamp-url "http://timestamp.acs.microsoft.com/" \
    --timestamp-root-cert "msft-tsa-root-certificate-authority-2020.crt" \
    $IMAGE
```

## Troubleshooting

### Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| `Access denied` | Missing RBAC roles | Assign appropriate roles |
| `Certificate not found` | Wrong certificate name | Verify certificate exists |
| `Invalid key type` | Unsupported key algorithm | Use RSA or EC supported sizes |
| `Signature failed` | Missing `Sign` permission | Add Key Vault Crypto User role |

### Verify Access

```bash
# Check certificate exists
az keyvault certificate show -n $CERT_NAME --vault-name $AKV_NAME

# Check permissions
az keyvault certificate list --vault-name $AKV_NAME
```

## Best Practices

1. **Use Azure RBAC** over access policies for better security management
2. **Create dedicated Key Vault** for signing certificates only
3. **Enable soft delete** and purge protection on Key Vault
4. **Use managed identities** in CI/CD pipelines
5. **Enable timestamping** for long-term signature validity
6. **Regular certificate rotation** before expiration
7. **Monitor key usage** through Key Vault diagnostics

## Source Files Referenced

- `/submodules/azure-management-docs/articles/container-registry/container-registry-tutorial-sign-build-push.md`
- `/submodules/azure-management-docs/articles/container-registry/container-registry-tutorial-sign-trusted-ca.md`
- `/submodules/azure-management-docs/articles/container-registry/container-registry-tutorial-verify-with-ratify-aks.md`

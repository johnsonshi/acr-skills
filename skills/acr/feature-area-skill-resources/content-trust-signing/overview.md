# ACR Content Trust & Signing Skill

This skill provides comprehensive knowledge about container image signing and verification in Azure Container Registry.

## When to Use This Skill

Use this skill when answering questions about:
- Signing container images
- Notation CLI usage
- Docker Content Trust (legacy)
- Azure Key Vault for signing
- Ratify verification on AKS
- Supply chain security

## Overview

ACR supports two signing approaches:

| Approach | Status | Recommendation |
|----------|--------|----------------|
| **Notation (Notary Project)** | Active | ✅ Use this |
| **Docker Content Trust (DCT)** | Deprecated (removal: Mar 2028) | ⚠️ Migrate away |

## Notation Signing (Recommended)

### Install Notation
```bash
# Linux
curl -Lo notation.tar.gz https://github.com/notaryproject/notation/releases/download/v1.1.0/notation_1.1.0_linux_amd64.tar.gz
tar xvzf notation.tar.gz -C /usr/local/bin notation

# Install Azure Key Vault plugin
notation plugin install --url https://github.com/Azure/notation-azure-kv/releases/download/v1.0.2/notation-azure-kv_1.0.2_linux_amd64.tar.gz
```

### Sign with Self-Signed Certificate
```bash
# Generate test certificate
notation cert generate-test --default "wabbit-networks.io"

# Sign image
notation sign myregistry.azurecr.io/myapp@sha256:abc123...

# List signatures
notation list myregistry.azurecr.io/myapp@sha256:abc123...
```

### Sign with Azure Key Vault
```bash
# Add Key Vault key
notation key add \
  --plugin azure-kv \
  --id https://myvault.vault.azure.net/keys/mykey/v1 \
  --default \
  mykey

# Sign image
notation sign myregistry.azurecr.io/myapp@sha256:abc123...
```

## Verification Setup

### Configure Trust Policy
Create `~/.config/notation/trustpolicy.json`:
```json
{
  "version": "1.0",
  "trustPolicies": [
    {
      "name": "production-images",
      "registryScopes": ["myregistry.azurecr.io/*"],
      "signatureVerification": {
        "level": "strict"
      },
      "trustStores": ["ca:acme-rockets"],
      "trustedIdentities": [
        "x509.subject: C=US, ST=WA, O=acme.io, CN=SecureBuilder"
      ]
    }
  ]
}
```

### Verify Image
```bash
# Add CA certificate to trust store
notation cert add --type ca --store acme-rockets ./root-ca.crt

# Verify
notation verify myregistry.azurecr.io/myapp@sha256:abc123...
```

## Ratify on AKS (Deployment Verification)

### Install Ratify
```bash
# Add Helm repo
helm repo add ratify https://notaryproject.github.io/ratify
helm repo update

# Install with Azure Key Vault
helm install ratify ratify/ratify \
  --namespace gatekeeper-system \
  --set featureFlags.RATIFY_CERT_ROTATION=true \
  --set azureKeyVault.enabled=true \
  --set azureKeyVault.vaultURI=https://myvault.vault.azure.net \
  --set azureKeyVault.certificates[0].certificateName=signing-cert
```

### Azure Policy Enforcement
```bash
# Deny unsigned images
az policy assignment create \
  --name require-signed-images \
  --policy "container-only-allow-signed-images" \
  --scope /subscriptions/{sub}/resourceGroups/{rg}
```

## Docker Content Trust (Legacy)

> **Warning:** DCT is deprecated. Cannot enable on new registries after May 31, 2026. Complete removal March 31, 2028.

### Enable DCT
```bash
# Enable on registry
az acr config content-trust update --registry myregistry --status enabled

# Enable on client
export DOCKER_CONTENT_TRUST=1
```

### Push Signed Image
```bash
export DOCKER_CONTENT_TRUST=1
docker push myregistry.azurecr.io/myapp:v1
# Creates delegation key on first push
```

## GitHub Actions Integration

```yaml
- name: Sign container image
  uses: notaryproject/notation-action@v1
  with:
    plugin_name: azure-kv
    plugin_url: https://github.com/Azure/notation-azure-kv/releases/download/v1.0.2/notation-azure-kv_1.0.2_linux_amd64.tar.gz
    key_id: https://myvault.vault.azure.net/keys/mykey/v1
    target_artifact_reference: myregistry.azurecr.io/myapp@${{ steps.build.outputs.digest }}
```

## Required Roles

### For Notation Signing
| Role | Purpose |
|------|---------|
| AcrPush / Repository Writer | Push images and signatures |
| Key Vault Crypto User | Sign operations |
| Key Vault Certificates User | Read certificates |

### For DCT (Legacy)
| Role | Purpose |
|------|---------|
| AcrImageSigner | Push signed images |
| AcrPush | Push operations |

## Verification Levels

| Level | Behavior |
|-------|----------|
| `strict` | Full verification, fail on any issue |
| `permissive` | Verify, log failures |
| `audit` | Integrity only |
| `skip` | No verification |

## Key Findings

1. **DCT Deprecation**: Migrate to Notation before March 2028
2. **Key Storage**: Use Azure Key Vault for production
3. **CI/CD**: Use Notation GitHub Actions for pipelines
4. **AKS Enforcement**: Ratify + Azure Policy for deployment verification

## Reference Documentation

- **Investigation reports**: `agent-investigation-reports/acr-features/content-trust-signing/`
- **MS Learn docs**: `azure-management-docs/articles/container-registry/overview-sign-verify-artifacts.md`

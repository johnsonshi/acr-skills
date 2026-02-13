# GitHub Actions Integration for Container Signing

## Overview

Notation provides native GitHub Actions for signing and verifying container images in CI/CD workflows. This enables automated image signing as part of the build pipeline, ensuring all published images are cryptographically signed.

## Available GitHub Actions

| Action | Purpose |
|--------|---------|
| `notaryproject/notation-action/setup` | Install Notation CLI |
| `notaryproject/notation-action/sign` | Sign container images |
| `notaryproject/notation-action/verify` | Verify container image signatures |

## Prerequisites

- Azure Container Registry
- GitHub repository for workflows
- Authentication configured (service principal or federated credentials)
- Signing service: Azure Key Vault or Artifact Signing

## Authentication Setup

### Using Federated Credentials (Recommended)

1. **Create managed identity:**
```bash
az identity create -g <identity-resource-group> -n <identity-name>
CLIENT_ID=$(az identity show -g <identity-resource-group> -n <identity-name> --query clientId -o tsv)
```

2. **Assign ACR roles:**
```bash
# For non-ABAC registries
az role assignment create --assignee $CLIENT_ID --scope $ACR_SCOPE --role "acrpush"
az role assignment create --assignee $CLIENT_ID --scope $ACR_SCOPE --role "acrpull"

# For ABAC-enabled registries
az role assignment create --assignee $CLIENT_ID --scope $ACR_SCOPE --role "Container Registry Repository Reader"
az role assignment create --assignee $CLIENT_ID --scope $ACR_SCOPE --role "Container Registry Repository Writer"
```

3. **Assign Artifact Signing role (if using):**
```bash
AS_SCOPE=/subscriptions/<subscription-id>/resourceGroups/<ts-account-resource-group>/providers/Microsoft.CodeSigning/codeSigningAccounts/<ts-account>/certificateProfiles/<ts-cert-profile>
az role assignment create --assignee $CLIENT_ID --scope $AS_SCOPE --role "Artifact Signing Certificate Profile Signer"
```

4. **Configure GitHub to trust identity:**
   - Follow [Configure federated identity credentials](https://docs.microsoft.com/entra/workload-id/workload-identity-federation-create-trust-user-assigned-managed-identity)

5. **Create GitHub secrets:**

| Secret | Value |
|--------|-------|
| `AZURE_CLIENT_ID` | Managed identity client ID |
| `AZURE_SUBSCRIPTION_ID` | Azure subscription ID |
| `AZURE_TENANT_ID` | Azure tenant ID |

## Signing Workflow with Artifact Signing

### Store TSA Root Certificate

Store the timestamping authority root certificate in your repository:
```bash
curl -o .github/certs/msft-identity-verification-root-cert-authority-2020.crt \
    "http://www.microsoft.com/pkiops/certs/microsoft%20identity%20verification%20root%20certificate%20authority%202020.crt"
```

### Complete Signing Workflow

```yaml
# .github/workflows/sign-with-artifact-signing.yml
name: Build, Push, and Sign Container Image

on:
  push:

env:
  ACR_LOGIN_SERVER: myregistry.azurecr.io
  ACR_REPO_NAME: myapp
  IMAGE_TAG: ${{ github.sha }}
  PLUGIN_NAME: azure-artifactsigning
  PLUGIN_DOWNLOAD_URL: "https://github.com/Azure/artifact-signing-notation-plugin/releases/download/v1.0.0/notation-azure-artifactsigning_1.0.0_linux_amd64.tar.gz"
  PLUGIN_CHECKSUM: 2f45891a14aa9c88c9bee3d11a887c1adbe9d2d24e50de4bc4b4fa3fe595292f
  TSA_URL: "http://timestamp.acs.microsoft.com/"
  TSA_ROOT_CERT: .github/certs/msft-identity-verification-root-cert-authority-2020.crt
  AS_ACCOUNT_NAME: my-artifact-signing-account
  AS_CERT_PROFILE: my-cert-profile
  AS_ACCOUNT_URI: "https://eus.codesigning.azure.net/"

jobs:
  build-sign:
    runs-on: ubuntu-latest
    permissions:
      id-token: write
      contents: read
    steps:
      - name: Checkout
        uses: actions/checkout@v3

      - name: Prepare artifact reference
        id: prepare
        run: |
          echo "target_artifact_reference=${{ env.ACR_LOGIN_SERVER }}/${{ env.ACR_REPO_NAME }}:${{ env.IMAGE_TAG }}" >> "$GITHUB_ENV"

      - name: Azure login
        uses: Azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - name: ACR login
        run: az acr login --name ${{ env.ACR_LOGIN_SERVER }}

      - name: Build and push
        id: push
        uses: docker/build-push-action@v4
        with:
          push: true
          tags: ${{ env.target_artifact_reference }}

      - name: Retrieve digest
        run: |
          echo "target_artifact_reference=${{ env.ACR_LOGIN_SERVER }}/${{ env.ACR_REPO_NAME }}@${{ steps.push.outputs.digest }}" >> "$GITHUB_ENV"

      - name: Setup Notation
        uses: notaryproject/notation-action/setup@v1.2.2

      - name: Sign with Artifact Signing
        uses: notaryproject/notation-action/sign@v1
        with:
          timestamp_url: ${{ env.TSA_URL }}
          timestamp_root_cert: ${{ env.TSA_ROOT_CERT }}
          plugin_name: ${{ env.PLUGIN_NAME }}
          plugin_url: ${{ env.PLUGIN_DOWNLOAD_URL }}
          plugin_checksum: ${{ env.PLUGIN_CHECKSUM }}
          key_id: ${{ env.AS_CERT_PROFILE }}
          target_artifact_reference: ${{ env.target_artifact_reference }}
          signature_format: cose
          plugin_config: |-
            accountName=${{ env.AS_ACCOUNT_NAME }}
            baseUrl=${{ env.AS_ACCOUNT_URI }}
            certProfile=${{ env.AS_CERT_PROFILE }}
          force_referrers_tag: 'false'
```

## Signing Workflow with Key Vault

```yaml
# .github/workflows/sign-with-keyvault.yml
name: Build, Push, and Sign with Key Vault

on:
  push:

env:
  ACR_LOGIN_SERVER: myregistry.azurecr.io
  ACR_REPO_NAME: myapp
  IMAGE_TAG: ${{ github.sha }}
  PLUGIN_NAME: azure-kv
  PLUGIN_DOWNLOAD_URL: "https://github.com/Azure/notation-azure-kv/releases/download/v1.2.1/notation-azure-kv_1.2.1_linux_amd64.tar.gz"
  PLUGIN_CHECKSUM: 67c5ccaaf28dd44d2b6572684d84e344a02c2258af1d65ead3910b3156d3eaf5
  AKV_NAME: my-keyvault
  CERT_NAME: my-signing-cert

jobs:
  build-sign:
    runs-on: ubuntu-latest
    permissions:
      id-token: write
      contents: read
    steps:
      - name: Checkout
        uses: actions/checkout@v3

      - name: Azure login
        uses: Azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - name: ACR login
        run: az acr login --name ${{ env.ACR_LOGIN_SERVER }}

      - name: Build and push
        id: push
        uses: docker/build-push-action@v4
        with:
          push: true
          tags: ${{ env.ACR_LOGIN_SERVER }}/${{ env.ACR_REPO_NAME }}:${{ env.IMAGE_TAG }}

      - name: Get Key Vault key ID
        id: get-key-id
        run: |
          KEY_ID=$(az keyvault certificate show -n ${{ env.CERT_NAME }} --vault-name ${{ env.AKV_NAME }} --query 'kid' -o tsv)
          echo "key_id=$KEY_ID" >> $GITHUB_OUTPUT

      - name: Setup Notation
        uses: notaryproject/notation-action/setup@v1.2.2

      - name: Sign with Key Vault
        uses: notaryproject/notation-action/sign@v1
        with:
          plugin_name: ${{ env.PLUGIN_NAME }}
          plugin_url: ${{ env.PLUGIN_DOWNLOAD_URL }}
          plugin_checksum: ${{ env.PLUGIN_CHECKSUM }}
          key_id: ${{ steps.get-key-id.outputs.key_id }}
          target_artifact_reference: ${{ env.ACR_LOGIN_SERVER }}/${{ env.ACR_REPO_NAME }}@${{ steps.push.outputs.digest }}
          signature_format: cose
          plugin_config: |-
            self_signed=true
```

## Verification Workflow

### Prepare Trust Store and Policy

Create trust store directory structure:
```
.github/
├── trustpolicy/
│   └── trustpolicy.json
└── truststore/
    └── x509/
        ├── ca/
        │   └── mycerts/
        │       └── msft-identity-verification-root-cert-2020.crt
        └── tsa/
            └── mytsacerts/
                └── msft-identity-verification-tsa-root-cert-2020.crt
```

### Trust Policy Example

```json
{
  "version": "1.0",
  "trustPolicies": [
    {
      "name": "mypolicy",
      "registryScopes": [
        "myregistry.azurecr.io/myrepo1",
        "myregistry.azurecr.io/myrepo2"
      ],
      "signatureVerification": {
        "level": "strict"
      },
      "trustStores": ["ca:mycerts", "tsa:mytsacerts"],
      "trustedIdentities": [
        "x509.subject: C=US, ST=WA, L=Seattle, O=MyCompany.io, OU=Tools"
      ]
    }
  ]
}
```

### Verification Workflow

```yaml
# .github/workflows/verify-image.yml
name: Verify Container Image Signature

on:
  push:

env:
  ACR_LOGIN_SERVER: myregistry.azurecr.io
  ACR_REPO_NAME: myrepo
  IMAGE_TAG: v1

jobs:
  verify:
    runs-on: ubuntu-latest
    permissions:
      id-token: write
      contents: read
    steps:
      - name: Checkout
        uses: actions/checkout@v3

      - name: Azure login
        uses: Azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - name: ACR login
        run: az acr login --name ${{ env.ACR_LOGIN_SERVER }}

      - name: Setup Notation
        uses: notaryproject/notation-action/setup@v1.2.2

      - name: Verify OCI artifact
        uses: notaryproject/notation-action/verify@v1
        with:
          target_artifact_reference: ${{ env.ACR_LOGIN_SERVER }}/${{ env.ACR_REPO_NAME }}:${{ env.IMAGE_TAG }}
          trust_policy: .github/trustpolicy/trustpolicy.json
          trust_store: .github/truststore
```

## Action Parameters

### Setup Action (`notaryproject/notation-action/setup`)

| Parameter | Required | Description |
|-----------|----------|-------------|
| `version` | No | Notation CLI version (default: latest) |

### Sign Action (`notaryproject/notation-action/sign`)

| Parameter | Required | Description |
|-----------|----------|-------------|
| `plugin_name` | Yes | Name of the signing plugin |
| `plugin_url` | Yes | URL to download the plugin |
| `plugin_checksum` | Yes | SHA256 checksum of the plugin |
| `key_id` | Yes | Signing key identifier |
| `target_artifact_reference` | Yes | Image reference to sign (use digest) |
| `signature_format` | No | Signature format (`cose` recommended) |
| `plugin_config` | No | Plugin-specific configuration |
| `timestamp_url` | No | RFC 3161 timestamp server URL |
| `timestamp_root_cert` | No | Path to TSA root certificate |
| `force_referrers_tag` | No | Use referrers tag schema |

### Verify Action (`notaryproject/notation-action/verify`)

| Parameter | Required | Description |
|-----------|----------|-------------|
| `target_artifact_reference` | Yes | Image reference to verify |
| `trust_policy` | Yes | Path to trust policy JSON file |
| `trust_store` | Yes | Path to trust store directory |
| `allow_referrers_api` | No | Enable OCI Referrers API |

## Best Practices

1. **Always use image digest** for signing, not tags (tags are mutable)
2. **Store certificates in repository** for reproducible verification
3. **Enable timestamping** for long-term signature validity
4. **Use federated credentials** instead of service principal secrets
5. **Verify signatures** before deploying to production environments
6. **Pin plugin versions** with checksums for security

## Workflow Trigger Options

```yaml
# Trigger on push to main
on:
  push:
    branches: [main]

# Trigger on release
on:
  release:
    types: [published]

# Trigger on workflow dispatch
on:
  workflow_dispatch:
    inputs:
      image_tag:
        description: 'Image tag to sign'
        required: true
```

## Viewing Workflow Results

After successful workflow execution:
1. Check GitHub Actions logs for signing/verification output
2. In Azure Portal, navigate to ACR > Repositories > Image > Referrers
3. Verify `application/vnd.cncf.notary.signature` artifacts are listed

## Troubleshooting

### Common Issues

| Issue | Solution |
|-------|----------|
| Plugin download fails | Verify URL and checksum |
| Authentication error | Check federated credential configuration |
| Signing fails | Verify Key Vault/Artifact Signing permissions |
| Verification fails | Check trust policy and certificate paths |

### Debug Mode

Add debug output to workflow:
```yaml
- name: Debug - List signatures
  run: notation ls ${{ env.target_artifact_reference }}
```

## Source Files Referenced

- `/submodules/azure-management-docs/articles/container-registry/container-registry-tutorial-github-sign-notation-artifact-signing.md`
- `/submodules/azure-management-docs/articles/container-registry/container-registry-tutorial-github-verify-notation-artifact-signing.md`
- `/submodules/azure-management-docs/articles/container-registry/overview-sign-verify-artifacts.md`

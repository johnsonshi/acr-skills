# CLI Commands Reference for Container Signing

## Overview

This reference covers all CLI commands related to container image signing and verification in Azure Container Registry, including Azure CLI commands for DCT, Notation CLI commands, and related Key Vault operations.

## Azure CLI Commands

### Docker Content Trust (DCT)

#### Enable DCT on Registry

```bash
az acr config content-trust update --registry <registry-name> --status enabled
```

#### Disable DCT on Registry

```bash
az acr config content-trust update --registry <registry-name> --status disabled

# Shorthand
az acr config content-trust update -r myregistry --status disabled
```

#### View DCT Status

```bash
az acr config content-trust show --registry <registry-name>
```

### Role Assignments for Signing

#### Assign AcrImageSigner Role (DCT)

```bash
# Get registry resource ID
REGISTRY_ID=$(az acr show --name myregistry --query id --output tsv)

# Assign to user
az role assignment create --scope $REGISTRY_ID --role AcrImageSigner --assignee user@contoso.com

# Assign to service principal
az role assignment create --scope $REGISTRY_ID --role AcrImageSigner --assignee <app-id>
```

#### Assign Container Registry Roles (Notation)

```bash
# For ABAC-enabled registries
az role assignment create --role "Container Registry Repository Reader" --assignee $USER_ID --scope "/subscriptions/$SUB_ID/resourceGroups/$RG/providers/Microsoft.ContainerRegistry/registries/$ACR_NAME"
az role assignment create --role "Container Registry Repository Writer" --assignee $USER_ID --scope "/subscriptions/$SUB_ID/resourceGroups/$RG/providers/Microsoft.ContainerRegistry/registries/$ACR_NAME"

# For non-ABAC registries
az role assignment create --role "AcrPull" --assignee $USER_ID --scope "/subscriptions/$SUB_ID/resourceGroups/$RG/providers/Microsoft.ContainerRegistry/registries/$ACR_NAME"
az role assignment create --role "AcrPush" --assignee $USER_ID --scope "/subscriptions/$SUB_ID/resourceGroups/$RG/providers/Microsoft.ContainerRegistry/registries/$ACR_NAME"
```

### Azure Key Vault Commands

#### Create Certificate

```bash
az keyvault certificate create \
    --name <cert-name> \
    --vault-name <vault-name> \
    --policy @policy.json
```

#### Show Certificate Details

```bash
az keyvault certificate show \
    --name <cert-name> \
    --vault-name <vault-name>

# Get key ID for signing
KEY_ID=$(az keyvault certificate show -n $CERT_NAME --vault-name $AKV_NAME --query 'kid' -o tsv)
```

#### Download Public Certificate

```bash
az keyvault certificate download \
    --name <cert-name> \
    --vault-name <vault-name> \
    --file <output-path.pem>
```

#### Import Certificate

```bash
az keyvault certificate import \
    --name <cert-name> \
    --vault-name <vault-name> \
    --file <certificate.pfx>
```

#### Set Key Vault Access Policy

```bash
az keyvault set-policy \
    --name <vault-name> \
    --certificate-permissions create get \
    --key-permissions sign \
    --object-id <user-or-sp-object-id>
```

#### Assign Key Vault RBAC Roles

```bash
# For certificate operations
az role assignment create --role "Key Vault Certificates Officer" --assignee $USER_ID --scope "/subscriptions/$SUB_ID/resourceGroups/$RG/providers/Microsoft.KeyVault/vaults/$AKV_NAME"

# For signing operations
az role assignment create --role "Key Vault Crypto User" --assignee $USER_ID --scope "/subscriptions/$SUB_ID/resourceGroups/$RG/providers/Microsoft.KeyVault/vaults/$AKV_NAME"

# For reading certificates
az role assignment create --role "Key Vault Certificates User" --assignee $USER_ID --scope "/subscriptions/$SUB_ID/resourceGroups/$RG/providers/Microsoft.KeyVault/vaults/$AKV_NAME"

# For reading secrets (certificate chains)
az role assignment create --role "Key Vault Secrets User" --assignee $USER_ID --scope "/subscriptions/$SUB_ID/resourceGroups/$RG/providers/Microsoft.KeyVault/vaults/$AKV_NAME"
```

---

## Notation CLI Commands

### Installation

#### Install Notation (Linux)

```bash
# Download and install
curl -Lo notation.tar.gz https://github.com/notaryproject/notation/releases/download/v1.3.2/notation_1.3.2_linux_amd64.tar.gz
tar xvzf notation.tar.gz
cp ./notation /usr/local/bin

# Verify installation
notation version
```

#### Install Notation (Windows)

```powershell
Invoke-WebRequest -Uri "https://github.com/notaryproject/notation/releases/download/v1.3.2/notation_1.3.2_windows_amd64.zip" -OutFile notation.zip
Expand-Archive notation.zip -DestinationPath .
Move-Item -Path ".\notation\notation.exe" -Destination "$Env:ProgramFiles\Notation\notation.exe"
$env:PATH = "${Env:ProgramFiles}\Notation;${Env:PATH}"
```

### Plugin Management

#### Install Plugin

```bash
# Azure Key Vault plugin
notation plugin install --url https://github.com/Azure/notation-azure-kv/releases/download/v1.2.1/notation-azure-kv_1.2.1_linux_amd64.tar.gz --sha256sum 67c5ccaaf28dd44d2b6572684d84e344a02c2258af1d65ead3910b3156d3eaf5

# Artifact Signing plugin
notation plugin install --url "https://github.com/Azure/artifact-signing-notation-plugin/releases/download/v1.0.0/notation-azure-artifactsigning_1.0.0_linux_amd64.tar.gz" --sha256sum 2f45891a14aa9c88c9bee3d11a887c1adbe9d2d24e50de4bc4b4fa3fe595292f
```

#### List Plugins

```bash
notation plugin ls
```

### Signing Commands

#### Basic Signing with Key Vault

```bash
notation sign --signature-format cose \
    --id <key-id> \
    --plugin azure-kv \
    <image-reference>
```

#### Sign with Self-Signed Certificate

```bash
notation sign --signature-format cose \
    --id $KEY_ID \
    --plugin azure-kv \
    --plugin-config self_signed=true \
    $IMAGE
```

#### Sign with CA-Issued Certificate

```bash
# With embedded chain
notation sign --signature-format cose \
    --id $KEY_ID \
    --plugin azure-kv \
    $IMAGE

# With separate CA bundle
notation sign --signature-format cose \
    --id $KEY_ID \
    --plugin azure-kv \
    --plugin-config ca_certs=/path/to/ca_bundle.pem \
    $IMAGE
```

#### Sign with Timestamping

```bash
notation sign --signature-format cose \
    --id $KEY_ID \
    --plugin azure-kv \
    --timestamp-url "http://timestamp.acs.microsoft.com/" \
    --timestamp-root-cert "msft-tsa-root-certificate-authority-2020.crt" \
    $IMAGE
```

#### Sign with Artifact Signing

```bash
notation sign --signature-format cose \
    --timestamp-url "http://timestamp.acs.microsoft.com/" \
    --timestamp-root-cert "msft-tsa-root-certificate-authority-2020.crt" \
    --id $AS_CERT_PROFILE \
    --plugin azure-artifactsigning \
    --plugin-config accountName=$AS_ACCT_NAME \
    --plugin-config baseUrl=$AS_ACCT_URL \
    --plugin-config certProfile=$AS_CERT_PROFILE \
    $IMAGE
```

#### Specify Credential Type

```bash
notation sign --signature-format cose \
    --id $KEY_ID \
    --plugin azure-kv \
    --plugin-config credential_type=azurecli \
    $IMAGE
```

| Credential Type | Value |
|----------------|-------|
| Environment | `environment` |
| Workload Identity | `workloadid` |
| Managed Identity | `managedid` |
| Azure CLI | `azurecli` |

### Certificate Management

#### Add Certificate to Trust Store

```bash
# Add CA certificate
notation cert add --type ca --store <store-name> <certificate-path>

# Add TSA certificate
notation cert add --type tsa --store <store-name> <certificate-path>

# Shorthand
notation cert add -t ca -s mystore mycert.pem
notation cert add -t tsa -s mytsastore tsa-cert.pem
```

#### List Certificates

```bash
notation cert ls
```

#### Delete Certificate

```bash
notation cert delete --type ca --store <store-name> --all
notation cert delete --type ca --store <store-name> <certificate-name>
```

### Trust Policy Management

#### Import Trust Policy

```bash
notation policy import ./trustpolicy.json
```

#### Show Trust Policy

```bash
notation policy show
```

### Verification Commands

#### Verify Image

```bash
notation verify <image-reference>
```

#### Verify with Verbose Output

```bash
notation verify --verbose <image-reference>
```

### List Signatures

```bash
notation ls <image-reference>
```

**Example Output:**
```
myregistry.azurecr.io/myrepo@sha256:17cc5dd7dfb8739e19e33e43680e43071f07497ed716814f3ac80bd4aac1b58f
└── application/vnd.cncf.notary.signature
    └── sha256:d7258166ca820f5ab7190247663464f2dcb149df4d1b6c4943dcaac59157de8e
```

### Authentication

#### Login to Registry

```bash
notation login <registry-url>

# If already authenticated via az acr login or docker login, not needed
```

#### Logout

```bash
notation logout <registry-url>
```

---

## Docker CLI Commands (DCT)

### Enable DCT for Session

```bash
export DOCKER_CONTENT_TRUST=1
```

### Disable DCT for Session

```bash
export DOCKER_CONTENT_TRUST=0
# or
unset DOCKER_CONTENT_TRUST
```

### Push Signed Image

```bash
# With DCT enabled
docker push myregistry.azurecr.io/myimage:v1
```

### Pull Signed Image

```bash
# With DCT enabled
docker pull myregistry.azurecr.io/myimage:v1
```

### Build with DCT

```bash
# Enable for single command
docker build --disable-content-trust=false -t myregistry.azurecr.io/myimage:v1 .

# Disable for single command
docker build --disable-content-trust -t myregistry.azurecr.io/myimage:v1 .
```

---

## Trust Policy JSON Structure

```json
{
    "version": "1.0",
    "trustPolicies": [
        {
            "name": "policy-name",
            "registryScopes": [
                "registry.azurecr.io/repo1",
                "registry.azurecr.io/repo2"
            ],
            "signatureVerification": {
                "level": "strict"
            },
            "trustStores": [
                "ca:store-name",
                "tsa:tsa-store-name"
            ],
            "trustedIdentities": [
                "x509.subject: CN=example.com,O=MyOrg,L=Seattle,ST=WA,C=US"
            ]
        }
    ]
}
```

### Verification Levels

| Level | Description |
|-------|-------------|
| `strict` | Full verification required |
| `permissive` | Warn on failure, allow deployment |
| `audit` | Log only |
| `skip` | No verification |

---

## Certificate Policy JSON Structure

```json
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
        "subject": "CN=example.com,O=MyOrg,L=Seattle,ST=WA,C=US",
        "validityInMonths": 12
    }
}
```

---

## Quick Reference Card

### Sign Image (Key Vault, Self-Signed)
```bash
KEY_ID=$(az keyvault certificate show -n $CERT_NAME --vault-name $AKV_NAME --query 'kid' -o tsv)
notation sign --signature-format cose --id $KEY_ID --plugin azure-kv --plugin-config self_signed=true $IMAGE
```

### Sign Image (Artifact Signing)
```bash
notation sign --signature-format cose --timestamp-url "http://timestamp.acs.microsoft.com/" --timestamp-root-cert tsa.crt --id $CERT_PROFILE --plugin azure-artifactsigning --plugin-config accountName=$ACCOUNT --plugin-config baseUrl=$URL --plugin-config certProfile=$PROFILE $IMAGE
```

### Verify Image
```bash
notation cert add --type ca --store mystore cert.pem
notation policy import trustpolicy.json
notation verify $IMAGE
```

### List Signatures
```bash
notation ls $IMAGE
```

## Source Files Referenced

- `/submodules/azure-management-docs/articles/container-registry/container-registry-content-trust.md`
- `/submodules/azure-management-docs/articles/container-registry/container-registry-content-trust-deprecation.md`
- `/submodules/azure-management-docs/articles/container-registry/container-registry-tutorial-sign-build-push.md`
- `/submodules/azure-management-docs/articles/container-registry/container-registry-tutorial-sign-trusted-ca.md`
- `/submodules/azure-management-docs/articles/container-registry/container-registry-tutorial-sign-verify-notation-artifact-signing.md`

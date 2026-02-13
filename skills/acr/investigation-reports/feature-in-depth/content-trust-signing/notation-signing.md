# Notation Signing - Notary Project Integration

## Overview

Notation is an open-source supply-chain security tool developed by the Notary Project community and backed by Microsoft. It supports signing and verifying container images and other OCI artifacts using industry-standard cryptographic signatures.

## Key Features

- **OCI-Compliant Signatures**: Adheres to OCI standards for portability across cloud environments
- **Plug-in Architecture**: Supports multiple key providers through plug-ins
- **Trust Policies**: Fine-grained control over signature verification
- **Timestamping Support**: RFC 3161 compliant timestamping for extended trust
- **CI/CD Integration**: Native support for GitHub Actions and Azure DevOps

## Installation

### Installing Notation CLI

**Linux AMD64:**
```bash
# Download, extract, and install Notation v1.3.2
curl -Lo notation.tar.gz https://github.com/notaryproject/notation/releases/download/v1.3.2/notation_1.3.2_linux_amd64.tar.gz
tar xvzf notation.tar.gz

# Copy the Notation binary to the desired bin directory in $PATH
cp ./notation /usr/local/bin
```

**Windows:**
```powershell
# Download the Windows release
Invoke-WebRequest -Uri "https://github.com/notaryproject/notation/releases/download/v1.3.2/notation_1.3.2_windows_amd64.zip" -OutFile notation.zip

# Validate the checksum
$EXPECTED_SHA256SUM = "014f25a530eee17520c8e1eb7380e4bd02ff6fc04479a07a890954e3b7ddfdc7"
if ((Get-FileHash notation.zip -Algorithm SHA256).Hash -ne $EXPECTED_SHA256SUM) { Write-Error "Checksum mismatch"; exit 1 }

# Expand and install
Expand-Archive notation.zip -DestinationPath .
New-Item -ItemType Directory -Force -Path "$Env:ProgramFiles\Notation" | Out-Null
Move-Item -Path ".\notation\notation.exe" -Destination "$Env:ProgramFiles\Notation\notation.exe"

# Add to PATH for the current session
$env:PATH = "${Env:ProgramFiles}\Notation;${Env:PATH}"
```

### Installing the Azure Key Vault Plug-in

```bash
# Install Key Vault plug-in (notation-azure-kv) v1.2.1
notation plugin install --url https://github.com/Azure/notation-azure-kv/releases/download/v1.2.1/notation-azure-kv_1.2.1_linux_amd64.tar.gz --sha256sum 67c5ccaaf28dd44d2b6572684d84e344a02c2258af1d65ead3910b3156d3eaf5

# Verify installation
notation plugin ls
```

### Installing the Artifact Signing Plug-in

```bash
# Install Artifact Signing plug-in v1.0.0
notation plugin install --url "https://github.com/Azure/artifact-signing-notation-plugin/releases/download/v1.0.0/notation-azure-artifactsigning_1.0.0_linux_amd64.tar.gz" --sha256sum 2f45891a14aa9c88c9bee3d11a887c1adbe9d2d24e50de4bc4b4fa3fe595292f

# Verify installation
notation plugin ls
```

## Signing Container Images

### With Self-Signed Certificate (Key Vault)

1. **Create a certificate in Key Vault:**
```bash
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
        "subject": "CN=wabbit-networks.io,O=Notation,L=Seattle,ST=WA,C=US",
        "validityInMonths": 12
    }
}
EOF

# Create the certificate
az keyvault certificate create -n $CERT_NAME --vault-name $AKV_NAME -p @my_policy.json
```

2. **Sign the image:**
```bash
# Get the signing key ID
KEY_ID=$(az keyvault certificate show -n $CERT_NAME --vault-name $AKV_NAME --query 'kid' -o tsv)

# Sign the container image with COSE signature format
notation sign --signature-format cose --id $KEY_ID --plugin azure-kv --plugin-config self_signed=true $IMAGE
```

### With CA-Issued Certificate (Key Vault)

```bash
# Get the signing key ID
KEY_ID=$(az keyvault certificate show -n $CERT_NAME --vault-name $AKV_NAME --query 'kid' -o tsv)

# Sign with CA-issued certificate (no self_signed flag)
notation sign --signature-format cose $IMAGE --id $KEY_ID --plugin azure-kv

# If certificate doesn't contain the full chain, specify CA bundle:
notation sign --signature-format cose $IMAGE --id $KEY_ID --plugin azure-kv --plugin-config ca_certs=<ca_bundle_file>
```

### With Artifact Signing Service

```bash
# Download the timestamping root certificate
curl -o msft-tsa-root-certificate-authority-2020.crt $AS_TSA_ROOT_CERT

# Sign the image with Artifact Signing
notation sign --signature-format cose \
    --timestamp-url $AS_TSA_URL \
    --timestamp-root-cert "msft-tsa-root-certificate-authority-2020.crt" \
    --id $AS_CERT_PROFILE \
    --plugin azure-artifactsigning \
    --plugin-config accountName=$AS_ACCT_NAME \
    --plugin-config baseUrl=$AS_ACCT_URL \
    --plugin-config certProfile=$AS_CERT_PROFILE \
    $IMAGE
```

## Verifying Container Images

### Configure Trust Store

```bash
# Add root certificate to named trust store
STORE_TYPE="ca"
STORE_NAME="wabbit-networks.io"
notation cert add --type $STORE_TYPE --store $STORE_NAME $CERT_PATH

# For timestamping, add TSA root certificate
TSA_TRUST_STORE="myTsaRootCerts"
notation cert add -t tsa -s $TSA_TRUST_STORE msft-tsa-root-certificate-authority-2020.crt

# List certificates
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
            "trustStores": [ "$STORE_TYPE:$STORE_NAME" ],
            "trustedIdentities": [
                "x509.subject: $CERT_SUBJECT"
            ]
        }
    ]
}
EOF

# Import trust policy
notation policy import ./trustpolicy.json

# Show trust policy
notation policy show
```

### Verify Image

```bash
# Verify the container image
notation verify $IMAGE

# On success, returns:
# Successfully verified signature for myregistry.azurecr.io/myimage@sha256:...
```

## Signature Verification Levels

| Level | Description |
|-------|-------------|
| `strict` | Full verification - signature must be valid, trusted, and unaltered |
| `permissive` | Warnings on verification failures but allows image |
| `audit` | Log verification results without blocking |
| `skip` | Skip verification entirely |

## Trust Policy Concepts

### Registry Scopes
Define which repositories the policy applies to:
```json
"registryScopes": [ "myregistry.azurecr.io/myrepo", "myregistry.azurecr.io/otherrepo" ]
```

### Trust Stores
Reference named stores containing root certificates:
```json
"trustStores": [ "ca:wabbit-networks.io", "tsa:myTsaRootCerts" ]
```

### Trusted Identities
Specify which certificate subjects to trust:
```json
"trustedIdentities": [
    "x509.subject: CN=wabbit-networks.io,O=Notation,L=Seattle,ST=WA,C=US"
]
```

## Timestamping

RFC 3161 compliant timestamping extends trust beyond certificate validity:

- Signatures created before certificate expiry remain valid
- Reduces need to re-sign images when certificates expire
- Critical for short-lived certificates (Artifact Signing)

**Sign with timestamp:**
```bash
notation sign --signature-format cose \
    --timestamp-url "http://timestamp.acs.microsoft.com/" \
    --timestamp-root-cert "msft-tsa-root-certificate-authority-2020.crt" \
    --id $KEY_ID --plugin azure-kv $IMAGE
```

## View Signatures

```bash
# List signatures attached to an image
notation ls $IMAGE

# Output example:
# myregistry.azurecr.io/net-monitor@sha256:17cc5dd7dfb8739e19e33e43680e43071f07497ed716814f3ac80bd4aac1b58f
# └── application/vnd.cncf.notary.signature
#     └── sha256:d7258166ca820f5ab7190247663464f2dcb149df4d1b6c4943dcaac59157de8e
```

## Credential Types for Key Vault Authentication

| Credential Type | Plugin Config Value | Use Case |
|----------------|---------------------|----------|
| Environment | `environment` | CI/CD pipelines with service principal |
| Workload Identity | `workloadid` | Kubernetes workloads |
| Managed Identity | `managedid` | Azure VMs, Azure services |
| Azure CLI | `azurecli` | Interactive development |

**Example:**
```bash
notation sign --signature-format cose --id $KEY_ID --plugin azure-kv \
    --plugin-config self_signed=true \
    --plugin-config credential_type=azurecli $IMAGE
```

## OCI Referrers API

Since Notation v1.2.0, signatures are stored using OCI referrers tag schema by default. To use the OCI Referrers API instead:

```bash
notation sign --signature-format cose --id $KEY_ID --plugin azure-kv \
    --force-referrers-tag false $IMAGE
```

**Note:** Container Registry supports OCI Referrers API except for CMK-encrypted registries.

## Certificate Requirements

Notary Project certificates must meet specific requirements:

### Root and Intermediate Certificates
- `basicConstraints` extension: `critical`, `CA` field set to `true`
- `keyUsage` extension: `critical`, `keyCertSign` bit set

### Leaf Certificates
- Subject must contain: `CN`, `C`, `ST`, `O`
- X.509 key usage: `DigitalSignature` only
- Extended Key Usages: empty or `1.3.6.1.5.5.7.3.3` (code signing)
- Key: not exportable, supported type/size per Notary Project spec

## Source Files Referenced

- `/submodules/azure-management-docs/articles/container-registry/container-registry-tutorial-sign-build-push.md`
- `/submodules/azure-management-docs/articles/container-registry/container-registry-tutorial-sign-trusted-ca.md`
- `/submodules/azure-management-docs/articles/container-registry/container-registry-tutorial-sign-verify-notation-artifact-signing.md`
- `/submodules/azure-management-docs/articles/container-registry/overview-sign-verify-artifacts.md`

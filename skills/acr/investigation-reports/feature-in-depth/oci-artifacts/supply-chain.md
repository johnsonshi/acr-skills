# Azure Container Registry - Supply Chain Artifacts

## Overview

Azure Container Registry supports storing, managing, and distributing supply chain artifacts that provide critical metadata about container images and other OCI artifacts. These artifacts enable secure software supply chain practices including artifact signing, software bill of materials (SBOM) tracking, and vulnerability scanning results storage.

## Supply Chain Artifact Types

### 1. Signatures

Digital signatures cryptographically bind a publisher's identity to an artifact:

| Aspect | Description |
|--------|-------------|
| Purpose | Verify artifact authenticity and integrity |
| Tools | Notation (Notary Project), Cosign |
| Storage | Attached as referrer to subject artifact |
| Media Types | `application/vnd.cncf.notary.signature`, `application/cose` |

### 2. Software Bill of Materials (SBOM)

SBOMs document all components and dependencies in an artifact:

| Aspect | Description |
|--------|-------------|
| Purpose | Dependency tracking, vulnerability management |
| Formats | SPDX, CycloneDX |
| Storage | Attached as referrer to subject artifact |
| Example Type | `sbom/example`, `application/spdx+json` |

### 3. Vulnerability Scan Results

Security scan reports attached to artifacts:

| Aspect | Description |
|--------|-------------|
| Purpose | Document known vulnerabilities |
| Tools | Microsoft Defender, Trivy, Grype |
| Storage | Attached as referrer to subject artifact |

### 4. Attestations

Provenance and build attestations:

| Aspect | Description |
|--------|-------------|
| Purpose | Document build process and origin |
| Standards | SLSA, in-toto |
| Storage | Attached as referrer to subject artifact |

## Artifact Graph Architecture

Supply chain artifacts form a directed graph referencing subject artifacts:

```
container-image:v1
├── signature/example
│   └── sha256:555ea91f39e7fb30c06f3b7aa483663f067f2950dcb...
├── sbom/example
│   └── sha256:4f1843833c029ecf0524bc214a0df9a5787409fd27bed2160d83f8cc39fedef5
│       └── signature/example
│           └── sha256:3c43b8cb0c941ec165c9f33f197d7f75980a292400d340f1a51c6b325764aa93
├── vulnerability-scan/example
│   └── sha256:7a2b3c4d5e6f...
└── readme/example
    └── sha256:1a118663d1085e229ff1b2d4d89b5f6d67911f22e55...
```

## Signing Container Images

### Using Notation (Notary Project)

Notation is the recommended tool for signing OCI artifacts with Azure Key Vault integration.

#### Prerequisites

1. Install Notation CLI (v1.3.2+):
```bash
curl -Lo notation.tar.gz https://github.com/notaryproject/notation/releases/download/v1.3.2/notation_1.3.2_linux_amd64.tar.gz
tar xvzf notation.tar.gz
cp ./notation /usr/local/bin
```

2. Install Azure Key Vault Plugin:
```bash
notation plugin install --url https://github.com/Azure/notation-azure-kv/releases/download/v1.2.1/notation-azure-kv_1.2.1_linux_amd64.tar.gz --sha256sum 67c5ccaaf28dd44d2b6572684d84e344a02c2258af1d65ead3910b3156d3eaf5
```

#### Create Self-Signed Certificate

```bash
# Certificate policy
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
    "x509CertificateProperties": {
        "ekus": ["1.3.6.1.5.5.7.3.3"],
        "keyUsage": ["digitalSignature"],
        "subject": "CN=wabbit-networks.io,O=Notation,L=Seattle,ST=WA,C=US",
        "validityInMonths": 12
    }
}
EOF

# Create certificate
az keyvault certificate create -n $CERT_NAME --vault-name $AKV_NAME -p @my_policy.json
```

#### Sign an Image

```bash
# Get signing key ID
KEY_ID=$(az keyvault certificate show -n $CERT_NAME --vault-name $AKV_NAME --query 'kid' -o tsv)

# Sign with COSE format
notation sign --signature-format cose --id $KEY_ID --plugin azure-kv --plugin-config self_signed=true $IMAGE
```

#### Verify a Signature

```bash
# Download public certificate
az keyvault certificate download --name $CERT_NAME --vault-name $AKV_NAME --file $CERT_PATH

# Add to trust store
notation cert add --type ca --store wabbit-networks.io $CERT_PATH

# Configure trust policy
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

# Verify image
notation verify $IMAGE
```

### Signature Storage Options

#### OCI Referrers API (Default)
- Native OCI 1.1 specification
- Efficient signature discovery
- Supported by most ACR features

#### OCI Referrers Tag Schema
- Used for CMK-encrypted registries
- Automatic fallback when Referrers API unavailable
- Can be forced with `--force-referrers-tag false`

## Creating and Attaching SBOMs

### Using ORAS

```bash
# Create SBOM
echo '{"version": "0.0.0.0", "artifact": "'${IMAGE}'", "contents": "good"}' > sbom.json

# Attach to image
oras attach $IMAGE \
    --artifact-type sbom/example \
    ./sbom.json:application/json
```

### Signing the SBOM

```bash
# Get SBOM digest
SBOM_DIGEST=$(oras discover -o json \
                --artifact-type sbom/example \
                $IMAGE | jq -r ".manifests[0].digest")

# Create SBOM signature
echo '{"artifact": "'$IMAGE@$SBOM_DIGEST'", "signature": "jayden hancock"}' > sbom-signature.json

# Attach signature to SBOM
oras attach $IMAGE@$SBOM_DIGEST \
    --artifact-type 'signature/example' \
    ./sbom-signature.json:application/json
```

## Verification at Deployment (Ratify)

Ratify is a CNCF project that verifies supply chain artifacts during Kubernetes deployment.

### Ratify on AKS Setup

1. **Create Federated Identity**:
```bash
az identity federated-credential create \
    --name ratify-federated-credential \
    --identity-name "$IDENTITY_NAME" \
    --resource-group "$IDENTITY_RG" \
    --issuer "$AKS_OIDC_ISSUER" \
    --subject system:serviceaccount:"$RATIFY_NAMESPACE":"$RATIFY_SA_NAME"
```

2. **Install Ratify**:
```bash
helm install ratify ratify/ratify --atomic \
    --namespace $RATIFY_NAMESPACE --create-namespace \
    --set azureWorkloadIdentity.clientId=$IDENTITY_CLIENT_ID \
    --set oras.authProviders.azureWorkloadIdentityEnabled=true \
    --set azurekeyvault.enabled=true \
    --set azurekeyvault.vaultURI="https://$AKV_NAME.vault.azure.net" \
    --set azurekeyvault.certificates[0].name="$CERT_NAME" \
    --set notation.trustPolicies[0].registryScopes[0]="$REPO_URI" \
    --set notation.trustPolicies[0].trustStores[0]="ca:azurekeyvault" \
    --set notation.trustPolicies[0].trustedIdentities[0]="x509.subject: $SUBJECT"
```

3. **Apply Azure Policy**:
```bash
CUSTOM_POLICY=$(curl -L https://raw.githubusercontent.com/notaryproject/ratify/refs/tags/v1.4.0/library/default/customazurepolicy.json)
DEFINITION_ID=$(az policy definition create --name "ratify-default-custom-policy" \
    --rules "$(echo "$CUSTOM_POLICY" | jq .policyRule)" \
    --params "$(echo "$CUSTOM_POLICY" | jq .parameters)" \
    --mode "Microsoft.Kubernetes.Data" \
    --query id -o tsv)

az policy assignment create --policy "$DEFINITION_ID" \
    --name "ratify-default-custom-policy" \
    --scope "$POLICY_SCOPE"
```

### Policy Effects

| Effect | Behavior |
|--------|----------|
| Deny | Block unsigned/untrusted images |
| Audit | Allow deployment, mark as noncompliant |

## Timestamping

Timestamping extends signature validity beyond certificate expiration.

### Signing with Timestamp

```bash
notation sign --signature-format cose \
    --id $KEY_ID \
    --plugin azure-kv \
    --plugin-config self_signed=true \
    --timestamp-url https://timestamp.digicert.com \
    $IMAGE
```

### Verification with Timestamp

Add TSA certificate to trust store and update trust policy:
```json
{
    "trustStores": [ "ca:wabbit-networks.io", "tsa:timestamp-authority" ]
}
```

## Artifact Signing Services

### Azure Key Vault Integration
- User-managed certificate lifecycle
- Full control over signing keys
- Support for HSM-backed keys

### Azure Artifact Signing (Zero-Touch)
- Automatic certificate management
- Short-lived certificates
- Simplified signing experience

## Promoting Supply Chain Artifacts

Copy images with all referrers between environments:

```bash
oras copy -r $REGISTRY/dev/$REPO:$TAG $REGISTRY/staging/$REPO:$TAG
```

This copies:
- Subject image
- All signatures
- All SBOMs
- All nested referrers

## Discovering Supply Chain Artifacts

### View All Referrers

```bash
oras discover -o tree $IMAGE
```

### Filter by Type

```bash
oras discover -o json --artifact-type signature/example $IMAGE
```

### List Signatures (Notation)

```bash
notation ls $IMAGE
```

## Deleting Supply Chain Artifacts

When deleting a subject artifact, ACR automatically deletes all referrers:

```bash
# Deletes image AND all attached signatures, SBOMs, etc.
az acr repository delete --name $ACR_NAME --image $REPO:$TAG
```

Or delete specific referrers:

```bash
oras manifest delete $REGISTRY/$REPO@$DIGEST
```

## Best Practices

### 1. Sign Everything
- Sign container images
- Sign SBOMs
- Sign Helm charts

### 2. Use Timestamping
- Extend signature validity
- Handle certificate rotation
- Maintain compliance

### 3. Automate in CI/CD
- Sign in build pipelines
- Verify before deployment
- Generate SBOMs automatically

### 4. Implement Policy Enforcement
- Deploy Ratify on AKS
- Use Azure Policy
- Configure appropriate trust policies

## Related Documentation

- [Overview of Signing and Verifying OCI Artifacts](overview-sign-verify-artifacts.md)
- [Sign Container Images with Notation](container-registry-tutorial-sign-build-push.md)
- [Verify with Ratify and Azure Policy](container-registry-tutorial-verify-with-ratify-aks.md)

## Sources

- `/submodules/azure-management-docs/articles/container-registry/overview-sign-verify-artifacts.md`
- `/submodules/azure-management-docs/articles/container-registry/container-registry-tutorial-sign-build-push.md`
- `/submodules/azure-management-docs/articles/container-registry/container-registry-tutorial-verify-with-ratify-aks.md`
- `/submodules/azure-management-docs/articles/container-registry/container-registry-manage-artifact.md`
- `/submodules/acr/docs/container-registry-oras-artifacts.md`

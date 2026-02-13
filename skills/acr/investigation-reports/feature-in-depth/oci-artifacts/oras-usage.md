# Azure Container Registry - ORAS Usage Guide

## Overview

ORAS (OCI Registry as Storage) is the reference implementation and CLI tool for working with OCI artifacts. Azure Container Registry fully supports ORAS operations, enabling push, pull, attach, discover, and copy operations for any OCI-compliant artifact type.

## What is ORAS?

ORAS is an open-source project under the Cloud Native Computing Foundation (CNCF) that provides:
- A CLI tool for working with OCI artifacts
- Libraries for building OCI artifact tooling
- Reference implementation of OCI artifact operations

ORAS enables storing any content type in OCI-compliant registries, not just container images.

## Prerequisites

### ORAS CLI Installation

Install ORAS CLI version 1.2.3 or later:

**Linux (AMD64)**:
```bash
curl -LO https://github.com/oras-project/oras/releases/download/v1.2.3/oras_1.2.3_linux_amd64.tar.gz
tar xvzf oras_1.2.3_linux_amd64.tar.gz
cp ./oras /usr/local/bin/
```

For other platforms, see [ORAS installation documentation](https://oras.land/docs/installation).

### Environment Setup

```bash
ACR_NAME=myregistry
REGISTRY=$ACR_NAME.azurecr.io
REPO=net-monitor
TAG=v1
IMAGE=$REGISTRY/${REPO}:$TAG
```

## Authentication

### Method 1: Azure CLI Integration

```bash
az login
az acr login -n $ACR_NAME
```

After `az acr login`, ORAS can use the same credentials without additional authentication.

### Method 2: ORAS Login with Token

```bash
USER_NAME="00000000-0000-0000-0000-000000000000"
PASSWORD=$(az acr login --name $ACR_NAME --expose-token --output tsv --query accessToken)

oras login $REGISTRY \
  --username $USER_NAME \
  --password $PASSWORD
```

### Method 3: Service Principal

```bash
SERVICE_PRINCIPAL_NAME="oras-sp"
PASSWORD=$(az ad sp create-for-rbac --name $SERVICE_PRINCIPAL_NAME \
          --scopes $(az acr show --name $ACR_NAME --query id --output tsv) \
          --role acrpush \
          --query "password" --output tsv)
USER_NAME=$(az ad sp list --display-name $SERVICE_PRINCIPAL_NAME --query "[].appId" --output tsv)

oras login $REGISTRY \
  --username $USER_NAME \
  --password $PASSWORD
```

### Method 4: Repository-Scoped Token

```bash
USER_NAME="oras-token"
PASSWORD=$(az acr token create -n $USER_NAME \
              -r $ACR_NAME \
              --repository $REPO content/write \
              --only-show-errors \
              --query "credentials.passwords[0].value" -o tsv)

oras login $REGISTRY \
  --username $USER_NAME \
  --password $PASSWORD
```

## Core ORAS Commands

### 1. Push Artifacts (`oras push`)

#### Push a Single File

```bash
echo 'Readme Content' > readme.md

oras push $REGISTRY/samples/artifact:readme \
    --artifact-type readme/example \
    ./readme.md:application/markdown
```

Output:
```
Uploading 2fdeac43552b readme.md
Uploaded  2fdeac43552b readme.md
Pushed myregistry.azurecr.io/samples/artifact:readme
Digest: sha256:e2d60d1b171f08bd10e2ed171d56092e39c7bac1aec5d9dcf7748dd702682d53
```

#### Push Multiple Files

```bash
echo 'Readme Content' > readme.md
mkdir details/
echo 'Detailed Content' > details/readme-details.md

oras push $REGISTRY/samples/artifact:readme \
    --artifact-type readme/example \
    ./readme.md:application/markdown \
    ./details
```

### 2. Pull Artifacts (`oras pull`)

```bash
mkdir ./download
oras pull -o ./download $REGISTRY/samples/artifact:readme
```

View pulled files:
```bash
tree ./download
```

### 3. Attach Referencing Artifacts (`oras attach`)

Attach artifacts that reference a subject (parent) artifact:

```bash
# Create a signature file
echo '{"artifact": "'${IMAGE}'", "signature": "jayden hancock"}' > signature.json

# Attach signature to container image
oras attach $IMAGE \
    --artifact-type signature/example \
    ./signature.json:application/json
```

### 4. Discover References (`oras discover`)

View artifacts referencing a subject:

```bash
oras discover -o tree $IMAGE
```

Output:
```
myregistry.azurecr.io/net-monitor:v1
├── signature/example
│   └── sha256:555ea91f39e7fb30c06f3b7aa483663f067f2950dcb...
├── sbom/example
│   └── sha256:4f1843833c029ecf0524bc214a0df9a5787409fd27bed2160d83f8cc39fedef5
│       └── signature/example
│           └── sha256:3c43b8cb0c941ec165c9f33f197d7f75980a292400d340f1a51c6b325764aa93
└── readme/example
    └── sha256:1a118663d1085e229ff1b2d4d89b5f6d67911f22e55...
```

#### Filter by Artifact Type

```bash
SBOM_DIGEST=$(oras discover -o json \
                --artifact-type sbom/example \
                $IMAGE | jq -r ".manifests[0].digest")
```

### 5. Fetch Manifest (`oras manifest fetch`)

```bash
oras manifest fetch --pretty $REGISTRY/samples/artifact:readme
```

Output:
```json
{
  "mediaType": "application/vnd.oci.artifact.manifest.v1+json",
  "artifactType": "readme/example",
  "blobs": [
    {
      "mediaType": "application/markdown",
      "digest": "sha256:2fdeac43552b71eb9db534137714c7bad86b53a93c56ca96d4850c9b41b777fc",
      "size": 15,
      "annotations": {
        "org.opencontainers.image.title": "readme.md"
      }
    }
  ],
  "annotations": {
    "org.opencontainers.artifact.created": "2023-01-10T14:44:06Z"
  }
}
```

### 6. Delete Artifacts (`oras manifest delete`)

```bash
oras manifest delete $REGISTRY/samples/artifact:readme
```

### 7. Copy Artifacts (`oras copy`)

Copy artifacts and their referrers between registries:

```bash
TARGET_REPO=$REGISTRY/sample-staging/$REPO
oras copy -r $IMAGE $TARGET_REPO:$TAG
```

Output:
```
Copying 6bdea3cdc730 sbom-signature.json
Copying 78e159e81c6b sbom.json
Copied  6bdea3cdc730 sbom-signature.json
Copied  78e159e81c6b sbom.json
Copying 7cf1385c7f4d signature.json
Copied  7cf1385c7f4d signature.json
Copied myregistry.azurecr.io/net-monitor:v1 => myregistry.azurecr.io/sample-staging/net-monitor:v1
```

### 8. List Repository Tags (`oras repo tags`)

```bash
oras repo tags $REGISTRY/$REPO
```

## Building Artifact Graphs

### Creating a Supply Chain Graph

1. **Push Container Image** (or build with ACR Tasks):
```bash
az acr build -r $ACR_NAME -t $IMAGE https://github.com/wabbit-networks/net-monitor.git#main
```

2. **Attach Signature**:
```bash
echo '{"artifact": "'${IMAGE}'", "signature": "jayden hancock"}' > signature.json
oras attach $IMAGE \
    --artifact-type signature/example \
    ./signature.json:application/json
```

3. **Attach SBOM**:
```bash
echo '{"version": "0.0.0.0", "artifact": "'${IMAGE}'", "contents": "good"}' > sbom.json
oras attach $IMAGE \
    --artifact-type sbom/example \
    ./sbom.json:application/json
```

4. **Sign the SBOM**:
```bash
# Get SBOM digest
SBOM_DIGEST=$(oras discover -o json \
                --artifact-type sbom/example \
                $IMAGE | jq -r ".manifests[0].digest")

# Create SBOM signature
echo '{"artifact": "'$IMAGE@$SBOM_DIGEST'", "signature": "jayden hancock"}' > sbom-signature.json

# Attach SBOM signature
oras attach $IMAGE@$SBOM_DIGEST \
    --artifact-type 'signature/example' \
    ./sbom-signature.json:application/json
```

5. **View Complete Graph**:
```bash
oras discover -o tree $IMAGE
```

## OCI Referrers API vs Tag Schema

ACR supports two methods for storing referrer metadata:

### OCI Referrers API (Default)
- Native OCI 1.1 specification support
- More efficient discovery
- Supported by most ACR features

### OCI Referrers Tag Schema (Fallback)
- Used for CMK-encrypted registries
- Stores referrers using special tags
- ORAS falls back automatically when needed

## Working with Supply Chain Artifacts

### Promoting Artifacts Across Environments

Copy images with all referrers to staging:
```bash
oras copy -r $REGISTRY/$REPO:$TAG $REGISTRY/staging/$REPO:$TAG
```

### Pulling Specific Referrers

```bash
# Get document digest
DOC_DIGEST=$(oras discover -o json \
              --artifact-type 'readme/example' \
              $TARGET_REPO:$TAG | jq -r ".manifests[0].digest")

# Pull the document
oras pull -o ./download $TARGET_REPO@$DOC_DIGEST
```

## Best Practices

### 1. Use Appropriate Artifact Types

Define meaningful artifact types for different content:
- `signature/example` for signatures
- `sbom/example` for SBOMs
- `readme/example` for documentation
- `application/vnd.cncf.helm.config.v1+json` for Helm charts

### 2. Leverage Media Types

Specify correct media types for files:
```bash
oras push $REGISTRY/artifact:v1 \
    ./config.json:application/json \
    ./readme.md:application/markdown \
    ./data.tar.gz:application/gzip
```

### 3. Use Digests for Immutability

Reference artifacts by digest for reproducibility:
```bash
oras pull $REGISTRY/$REPO@sha256:abc123...
```

### 4. Tag Management

Keep tagged artifacts for primary content; use tagless referrers for supply chain:
```bash
# Tags for main artifacts
oras push $REGISTRY/myapp:v1.0.0 ...

# Tagless for referrers (automatic via attach)
oras attach $REGISTRY/myapp:v1.0.0 --artifact-type signature/example ...
```

## Troubleshooting

### Common Issues

1. **Authentication Errors**
   - Ensure `az acr login` or `oras login` completed successfully
   - Check token expiration (default 3 hours)

2. **Push Failures**
   - Verify repository naming conventions (lowercase)
   - Check network connectivity

3. **Referrers Not Found**
   - Ensure OCI 1.1 support in registry
   - Check if using CMK encryption (falls back to tag schema)

## Related Documentation

- [ORAS Official Documentation](https://oras.land/)
- [OCI Distribution Specification](https://github.com/opencontainers/distribution-spec)
- [OCI Artifacts Specification](https://github.com/opencontainers/artifacts)

## Sources

- `/submodules/azure-management-docs/articles/container-registry/container-registry-manage-artifact.md`
- `/submodules/acr/docs/container-registry-oras-artifacts.md`
- `/submodules/oras/docs/proposals/compatibility-mode.md`
- `/submodules/oras/docs/proposals/formatted-output.md`

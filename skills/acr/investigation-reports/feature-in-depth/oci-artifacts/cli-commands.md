# Azure Container Registry - CLI Commands for OCI Artifacts

## Overview

This document provides a comprehensive reference of CLI commands for managing OCI artifacts in Azure Container Registry. Commands are organized by tool and use case.

## Azure CLI Commands

### Repository Operations

#### List Repositories

```bash
az acr repository list --name $ACR_NAME
```

#### Show Repository Details

```bash
az acr repository show \
  --name $ACR_NAME \
  --repository $REPO
```

Output:
```json
{
  "changeableAttributes": {
    "deleteEnabled": true,
    "listEnabled": true,
    "readEnabled": true,
    "writeEnabled": true
  },
  "createdTime": "2023-10-05T12:11:37.6701689Z",
  "imageName": "helm/hello-world",
  "manifestCount": 1,
  "tagCount": 1
}
```

#### List Tags

```bash
az acr repository show-tags \
  --name $ACR_NAME \
  --repository $REPO
```

#### Delete Repository

```bash
az acr repository delete \
  --name $ACR_NAME \
  --repository $REPO \
  --yes
```

### Manifest Operations

#### List Manifests

```bash
az acr manifest list-metadata \
  --name $REPO \
  --registry $ACR_NAME
```

Output:
```json
[
  {
    "digest": "sha256:0a2e01852872580b2c2fea9380ff8d7b637d3928783c55beb3f21a6e58d5d108",
    "tags": ["latest", "v3"],
    "timestamp": "2023-07-12T15:52:00.2075864Z",
    "configMediaType": "application/vnd.cncf.helm.config.v1+json"
  }
]
```

#### Show Manifest Metadata

```bash
az acr manifest show-metadata \
  --registry $ACR_NAME \
  --name "$REPO:$TAG"
```

#### Show Manifest by Digest

```bash
az acr manifest show-metadata \
  --registry $ACR_NAME \
  --name "$REPO@$DIGEST"
```

#### Delete Manifest by Tag

```bash
az acr repository delete \
  --name $ACR_NAME \
  --image $REPO:$TAG \
  --yes
```

#### Delete Manifest by Digest

```bash
az acr repository delete \
  --name $ACR_NAME \
  --image $REPO@sha256:abc123... \
  --yes
```

### Import Operations

#### Import from Docker Hub

```bash
az acr import \
  --name $ACR_NAME \
  --source docker.io/library/hello-world:latest \
  --image hello-world:latest \
  --username <Docker Hub username> \
  --password <Docker Hub token>
```

#### Import from Another ACR

```bash
az acr import \
  --name $TARGET_ACR \
  --source $SOURCE_ACR.azurecr.io/$REPO:$TAG \
  --image $REPO:$TAG \
  --registry /subscriptions/$SUB_ID/resourceGroups/$RG/providers/Microsoft.ContainerRegistry/registries/$SOURCE_ACR
```

#### Import Helm Chart

```bash
az acr import \
  --name $ACR_NAME \
  --source docker.io/bitnami/nginx:latest \
  --image nginx:latest
```

#### Import by Digest

```bash
az acr import \
  --name $ACR_NAME \
  --source docker.io/library/hello-world@sha256:abc123 \
  --repository hello-world
```

### Authentication

#### Login to Registry

```bash
az acr login --name $ACR_NAME
```

#### Get Access Token

```bash
az acr login --name $ACR_NAME --expose-token --output tsv --query accessToken
```

### Build Operations

#### Build and Push Image

```bash
az acr build \
  --registry $ACR_NAME \
  --image $REPO:$TAG \
  https://github.com/user/repo.git#main
```

#### Build with Digest Output

```bash
DIGEST=$(az acr build -r $ACR_NAME -t $REGISTRY/${REPO}:$TAG $IMAGE_SOURCE \
  --no-logs --query "outputImages[0].digest" -o tsv)
```

## ORAS CLI Commands

### Authentication

#### Login to Registry

```bash
oras login $REGISTRY \
  --username $USER_NAME \
  --password $PASSWORD
```

#### Login with Password from Stdin

```bash
echo $PASSWORD | oras login $REGISTRY --username $USER_NAME --password-stdin
```

### Push Operations

#### Push Single File

```bash
oras push $REGISTRY/$REPO:$TAG \
  --artifact-type readme/example \
  ./readme.md:application/markdown
```

#### Push Multiple Files

```bash
oras push $REGISTRY/$REPO:$TAG \
  --artifact-type readme/example \
  ./readme.md:application/markdown \
  ./details/:application/vnd.oci.image.layer.v1.tar+gzip
```

#### Push with Annotations

```bash
oras push $REGISTRY/$REPO:$TAG \
  --artifact-type readme/example \
  --annotation "org.opencontainers.image.authors=Team" \
  ./readme.md:application/markdown
```

### Pull Operations

#### Pull to Directory

```bash
oras pull -o ./download $REGISTRY/$REPO:$TAG
```

#### Pull by Digest

```bash
oras pull -o ./download $REGISTRY/$REPO@sha256:abc123
```

### Attach Operations

#### Attach Signature

```bash
oras attach $REGISTRY/$REPO:$TAG \
  --artifact-type signature/example \
  ./signature.json:application/json
```

#### Attach SBOM

```bash
oras attach $REGISTRY/$REPO:$TAG \
  --artifact-type sbom/example \
  ./sbom.json:application/json
```

#### Attach to Digest

```bash
oras attach $REGISTRY/$REPO@sha256:abc123 \
  --artifact-type signature/example \
  ./signature.json:application/json
```

### Discover Operations

#### Discover Referrers (Tree View)

```bash
oras discover -o tree $REGISTRY/$REPO:$TAG
```

Output:
```
myregistry.azurecr.io/net-monitor:v1
├── signature/example
│   └── sha256:555ea91f39e7fb30...
├── sbom/example
│   └── sha256:4f1843833c029ecf...
└── readme/example
    └── sha256:1a118663d1085e22...
```

#### Discover Referrers (JSON Output)

```bash
oras discover -o json $REGISTRY/$REPO:$TAG
```

#### Filter by Artifact Type

```bash
oras discover -o json \
  --artifact-type signature/example \
  $REGISTRY/$REPO:$TAG
```

### Manifest Operations

#### Fetch Manifest

```bash
oras manifest fetch --pretty $REGISTRY/$REPO:$TAG
```

#### Delete Manifest

```bash
oras manifest delete $REGISTRY/$REPO:$TAG
```

#### Delete by Digest

```bash
oras manifest delete $REGISTRY/$REPO@sha256:abc123
```

### Copy Operations

#### Copy with Referrers

```bash
oras copy -r $REGISTRY/$REPO:$TAG $TARGET_REGISTRY/$REPO:$TAG
```

#### Copy Single Artifact

```bash
oras copy $REGISTRY/$REPO:$TAG $TARGET_REGISTRY/$REPO:$TAG
```

### Repository Operations

#### List Tags

```bash
oras repo tags $REGISTRY/$REPO
```

## Helm CLI Commands

### Authentication

#### Login to Registry

```bash
helm registry login $ACR_NAME.azurecr.io \
  --username $USER_NAME \
  --password $PASSWORD
```

### Push Operations

#### Push Chart

```bash
helm push hello-world-0.1.0.tgz oci://$ACR_NAME.azurecr.io/helm
```

### Pull Operations

#### Pull Chart

```bash
helm pull oci://$ACR_NAME.azurecr.io/helm/hello-world --version 0.1.0
```

### Install Operations

#### Install from Registry

```bash
helm install myrelease oci://$ACR_NAME.azurecr.io/helm/hello-world --version 0.1.0
```

### Package Operations

#### Package Chart

```bash
helm package ./hello-world
```

## Notation CLI Commands

### Plugin Management

#### List Plugins

```bash
notation plugin ls
```

#### Install Plugin

```bash
notation plugin install --url $PLUGIN_URL --sha256sum $CHECKSUM
```

### Signing Operations

#### Sign Image

```bash
notation sign --signature-format cose \
  --id $KEY_ID \
  --plugin azure-kv \
  --plugin-config self_signed=true \
  $REGISTRY/$REPO:$TAG
```

#### Sign with Timestamp

```bash
notation sign --signature-format cose \
  --id $KEY_ID \
  --plugin azure-kv \
  --plugin-config self_signed=true \
  --timestamp-url https://timestamp.digicert.com \
  $REGISTRY/$REPO:$TAG
```

### Verification Operations

#### Verify Image

```bash
notation verify $REGISTRY/$REPO:$TAG
```

#### List Signatures

```bash
notation ls $REGISTRY/$REPO:$TAG
```

### Certificate Management

#### Add Certificate to Trust Store

```bash
notation cert add --type ca --store $STORE_NAME $CERT_PATH
```

#### List Certificates

```bash
notation cert ls
```

### Policy Management

#### Import Trust Policy

```bash
notation policy import ./trustpolicy.json
```

#### Show Trust Policy

```bash
notation policy show
```

## Docker CLI Commands (OCI Compatible)

### Pull Operations

#### Pull by Tag

```bash
docker pull $ACR_NAME.azurecr.io/$REPO:$TAG
```

#### Pull by Digest

```bash
docker pull $ACR_NAME.azurecr.io/$REPO@sha256:abc123
```

### Push Operations

#### Push Image

```bash
docker push $ACR_NAME.azurecr.io/$REPO:$TAG
```

### Build Operations

#### Build with BuildKit OCI Output

```bash
docker buildx build --output type=oci,dest=./image.tar .
```

## Common Workflows

### Complete Artifact Publishing Workflow

```bash
# 1. Authenticate
az acr login --name $ACR_NAME

# 2. Build and push image
az acr build -r $ACR_NAME -t $REPO:$TAG .

# 3. Get image digest
DIGEST=$(az acr manifest list-metadata -n $REPO -r $ACR_NAME \
  --query "[?tags[?@=='$TAG']].digest" -o tsv)

# 4. Attach signature
oras attach $ACR_NAME.azurecr.io/$REPO@$DIGEST \
  --artifact-type signature/example \
  ./signature.json:application/json

# 5. Attach SBOM
oras attach $ACR_NAME.azurecr.io/$REPO@$DIGEST \
  --artifact-type sbom/example \
  ./sbom.json:application/json

# 6. View artifact graph
oras discover -o tree $ACR_NAME.azurecr.io/$REPO:$TAG
```

### Helm Chart Publishing Workflow

```bash
# 1. Authenticate
helm registry login $ACR_NAME.azurecr.io \
  --username $USER_NAME \
  --password $PASSWORD

# 2. Package chart
helm package ./mychart

# 3. Push to registry
helm push mychart-1.0.0.tgz oci://$ACR_NAME.azurecr.io/helm

# 4. Verify upload
az acr manifest list-metadata \
  --registry $ACR_NAME \
  --name helm/mychart
```

### Image Signing Workflow

```bash
# 1. Get key ID from Key Vault
KEY_ID=$(az keyvault certificate show -n $CERT_NAME --vault-name $AKV_NAME --query 'kid' -o tsv)

# 2. Sign image
notation sign --signature-format cose \
  --id $KEY_ID \
  --plugin azure-kv \
  --plugin-config self_signed=true \
  $ACR_NAME.azurecr.io/$REPO:$TAG

# 3. Verify signature
notation ls $ACR_NAME.azurecr.io/$REPO:$TAG

# 4. Verify image
notation verify $ACR_NAME.azurecr.io/$REPO:$TAG
```

## Error Handling

### Common Errors and Solutions

| Error | Cause | Solution |
|-------|-------|----------|
| `unauthorized` | Invalid credentials | Run `az acr login` or check token expiration |
| `manifest unknown` | Artifact not found | Verify repository and tag names |
| `denied` | Insufficient permissions | Check RBAC role assignments |
| `name invalid` | Invalid repository name | Use lowercase alphanumeric characters |

## Sources

- `/submodules/azure-management-docs/articles/container-registry/container-registry-manage-artifact.md`
- `/submodules/azure-management-docs/articles/container-registry/container-registry-helm-repos.md`
- `/submodules/azure-management-docs/articles/container-registry/container-registry-tutorial-sign-build-push.md`
- `/submodules/azure-management-docs/articles/container-registry/container-registry-import-images.md`
- `/submodules/azure-management-docs/articles/container-registry/container-registry-delete.md`
- `/submodules/acr/docs/container-registry-oras-artifacts.md`

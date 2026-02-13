# ACR Artifact Streaming - Setup Guide

## Prerequisites

Before enabling artifact streaming, ensure you meet the following requirements:

### Azure Container Registry Requirements

| Requirement | Specification |
|-------------|---------------|
| **Service Tier** | Premium SKU (required) |
| **Registry Type** | Standard ACR (CMK registries not supported) |
| **Image Platform** | Linux AMD64 only |

### Azure Kubernetes Service Requirements

| Requirement | Specification |
|-------------|---------------|
| **Kubernetes Version** | 1.26 or higher |
| **Ubuntu Version** | 20.04 or higher (for Ubuntu-based node pools) |
| **Container Runtime** | containerd (default for K8s 1.19+) |

### Tooling Requirements

| Tool | Minimum Version |
|------|-----------------|
| **Azure CLI** | 2.54.0 or higher |

## Setup Methods

### Method 1: Azure CLI

#### Step 1: Set Default Registry

```bash
az config set defaults.acr="<registry-name>"
```

#### Step 2: Import or Push an Image

Option A - Import from Docker Hub:
```bash
az acr import \
  --source docker.io/jupyter/all-spark-notebook:latest \
  -t jupyter/all-spark-notebook:latest
```

Option B - Push from local Docker:
```bash
docker tag <local-image> <registry-name>.azurecr.io/<repo>:<tag>
docker push <registry-name>.azurecr.io/<repo>:<tag>
```

#### Step 3: Create Streaming Artifact for Existing Image

```bash
az acr artifact-streaming create \
  --image <repo>:<tag>
```

Example:
```bash
az acr artifact-streaming create \
  --image jupyter/all-spark-notebook:latest
```

#### Step 4: Verify Streaming Artifact Creation

List referrers (streaming artifacts) for an image:
```bash
az acr manifest list-referrers \
  -n <repo>:<tag>
```

Example:
```bash
az acr manifest list-referrers \
  -n jupyter/all-spark-notebook:latest
```

#### Step 5: Enable Automatic Streaming for Repository

Enable automatic conversion for new pushes:
```bash
az acr artifact-streaming update \
  --repository <repo> \
  --enable-streaming true
```

Example:
```bash
az acr artifact-streaming update \
  --repository jupyter/all-spark-notebook \
  --enable-streaming true
```

#### Step 6: Verify Repository Streaming State

Check the conversion status after pushing a new image:
```bash
az acr artifact-streaming operation show \
  --image <repo>:<new-tag>
```

### Method 2: Azure Portal

#### Step 1: Navigate to Registry

1. Go to the [Azure Portal](https://portal.azure.com)
2. Navigate to your Azure Container Registry
3. Select **Services** > **Repositories**

#### Step 2: Enable Artifact Streaming on Repository

1. Select **Start artifact streaming**
2. Status changes to **Active**
3. New compatible images are automatically converted

#### Step 3: Create Streaming Artifact for Existing Images

1. From **Repositories**, select the target image
2. Select **Create streaming artifact**
3. View result in **Referrers** tab

#### Step 4: Disable Automatic Streaming (Optional)

1. Navigate to the repository
2. Select **Stop artifact streaming**
3. Status changes to **Inactive**

## CLI Command Reference

### Create Streaming Artifact

```bash
az acr artifact-streaming create \
  --registry <registry-name> \
  --image <repo>:<tag>
```

### Cancel Ongoing Conversion

```bash
az acr artifact-streaming operation cancel \
  --registry <registry-name> \
  --repository <repo> \
  --id <operation-id>
```

### Check Conversion Status

```bash
az acr artifact-streaming operation show \
  --registry <registry-name> \
  --image <repo>:<tag>
```

### Enable/Disable Auto-Streaming on Repository

Enable:
```bash
az acr artifact-streaming update \
  --registry <registry-name> \
  --repository <repo> \
  --enable-streaming true
```

Disable:
```bash
az acr artifact-streaming update \
  --registry <registry-name> \
  --repository <repo> \
  --enable-streaming false
```

### List Streaming Artifacts (Referrers)

```bash
az acr manifest list-referrers \
  --registry <registry-name> \
  -n <repo>:<tag>
```

## Supported Image Formats

### Supported Media Types

- `application/vnd.oci.image.manifest.v1+json`
- `application/vnd.oci.image.index.v1+json`
- `application/vnd.docker.distribution.manifest.v2+json`
- `application/vnd.docker.distribution.manifest.list.v2+json`

### Streaming Artifact Type

- `application/vnd.azure.artifact.streaming.v1`

Note: Streaming artifacts cannot be converted again (UNSUPPORTED_ARTIFACT_TYPE error).

## Storage Considerations

### Storage Impact

- Streaming artifacts are stored alongside original images
- May increase overall registry storage consumption
- Premium SKU includes 500 GiB; additional charges apply beyond threshold

### Retention Behavior

- Original and streaming versions deleted together
- With soft delete: only original can be viewed/restored during retention

## Verification Steps

### Verify Streaming Artifact Exists

```bash
az acr manifest list-referrers -n <repo>:<tag>
```

Expected output includes artifact with type `application/vnd.azure.artifact.streaming.v1`.

### Verify Repository Streaming State

```bash
az acr repository show \
  --repository <repo> \
  -o jsonc
```

Look for streaming-related attributes in the output.

---

## Legacy Setup: Project Teleport (Deprecated)

> **Warning**: Project Teleport was deprecated in November 2023. Use Artifact Streaming instead.

### Historical Setup Steps

These steps are preserved for reference only:

#### 1. Sign Up for Preview

Submit request at `aka.ms/teleport/signup` (no longer active)

#### 2. Set Environment Variables

```bash
ACR=myacr
ACR_URL=${ACR}.azurecr.io
```

#### 3. Verify Teleport Enabled

```bash
az acr repository show \
  --repository <repo> \
  -o jsonc
```

Look for `"teleportEnabled": true` in changeableAttributes.

#### 4. Check Layer Expansion Status

Using the check-expansion.sh script:

```bash
export ACR_USER=teleport-token
export ACR_PWD=$(az acr token create \
  --name teleport-token \
  --registry $ACR \
  --scope-map _repositories_pull \
  --query credentials.passwords[0].value -o tsv)

./check-expansion.sh ${ACR} <repo> <tag>
```

Script returns:
- Status 200: "Teleport: layers ready"
- Status 409: "Teleport: expanding layers"
- Status 404: "Teleport: not enabled"

#### 5. Repository Management Scripts

Find teleport-enabled repositories:
```bash
./find-teleport-enabled-repositories.sh <registry-name>
```

Enable/disable teleport on repository:
```bash
./edit-teleport-attribute.sh <registry-name> <repository> enable
./edit-teleport-attribute.sh <registry-name> <repository> disable
```

## Source Files

- Setup Guide: `submodules/azure-management-docs/articles/container-registry/container-registry-artifact-streaming.md`
- Legacy Teleport Setup: `submodules/acr/docs/teleport/aks-getting-started.md`
- Repository Management: `submodules/acr/docs/teleport/teleport-repository-management.md`
- Check Expansion Script: `submodules/acr/docs/teleport/check-expansion.sh`

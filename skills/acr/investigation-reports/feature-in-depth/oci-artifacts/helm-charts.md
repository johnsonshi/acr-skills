# Azure Container Registry - Helm Charts Support

## Overview

Azure Container Registry provides native support for storing Helm charts as OCI artifacts, enabling organizations to manage Kubernetes application packages alongside container images in a single registry. With Helm 3, charts are stored as OCI artifacts, eliminating the need for separate chart repositories.

## Helm Chart Storage Evolution

### Legacy Helm 2 Approach (Deprecated)
- Chart repositories based on `index.yaml` files
- Separate `az acr helm` commands (retired September 15, 2025)
- HTTP-based chart repository protocol

### Modern Helm 3 Approach (Current)
- Charts stored as OCI artifacts
- Native `helm` CLI commands for registry operations
- OCI Distribution Specification compliance
- GA support in Azure Container Registry

## Prerequisites

### Required Components

1. **Helm Client**: Version 3.7.2 or later (3.8.0+ recommended)
   ```console
   helm version
   ```

2. **Azure Container Registry**: Any tier (Basic, Standard, Premium)

3. **Azure CLI**: Version 2.0.71 or later (for registry operations)

## Helm Chart Media Types

When stored as OCI artifacts, Helm charts use specific media types:

| Component | Media Type |
|-----------|-----------|
| Config | `application/vnd.cncf.helm.config.v1+json` |
| Chart Content | `application/vnd.cncf.helm.chart.content.v1.tar+gzip` |
| Manifest | `application/vnd.oci.image.manifest.v1+json` |

## Workflow: Push and Pull Helm Charts

### Step 1: Set Environment Variables

```bash
ACR_NAME=<container-registry-name>
```

### Step 2: Create a Helm Chart

```console
mkdir helmtest
cd helmtest
helm create hello-world
```

### Step 3: Package the Chart

```console
cd hello-world
helm package .
# Output: Successfully packaged chart and saved it to: ./hello-world-0.1.0.tgz
```

### Step 4: Authenticate to Registry

**Option A: Using Helm Registry Login**
```bash
helm registry login $ACR_NAME.azurecr.io \
  --username $USER_NAME \
  --password $PASSWORD
```

**Option B: Using Azure CLI**
```bash
az acr login --name $ACR_NAME
```

### Step 5: Push Chart to Registry

```console
helm push hello-world-0.1.0.tgz oci://$ACR_NAME.azurecr.io/helm
```

Output:
```
Pushed: myregistry.azurecr.io/helm/hello-world:0.1.0
digest: sha256:5899db028dcf96aeaabdadfa5899db02589b2899b025899b059db02
```

### Step 6: Pull Chart from Registry

```console
helm pull oci://$ACR_NAME.azurecr.io/helm/hello-world --version 0.1.0
```

### Step 7: Install Chart to Kubernetes

```console
helm install myhelmtest oci://$ACR_NAME.azurecr.io/helm/hello-world --version 0.1.0
```

## Authentication Methods

### 1. Service Principal Authentication

For ABAC-enabled registries:
```bash
ROLE="Container Registry Repository Writer"
PASSWORD=$(az ad sp create-for-rbac --name $SERVICE_PRINCIPAL_NAME \
          --scopes $(az acr show --name $ACR_NAME --query id --output tsv) \
          --role "$ROLE" \
          --query "password" --output tsv)
```

For non-ABAC registries:
```bash
ROLE="AcrPush"
```

### 2. Microsoft Entra Identity

```bash
USER_NAME="00000000-0000-0000-0000-000000000000"
PASSWORD=$(az acr login --name $ACR_NAME --expose-token --output tsv --query accessToken)
```

### 3. Repository-Scoped Token

```bash
USER_NAME="helmtoken"
PASSWORD=$(az acr token create -n $USER_NAME \
              -r $ACR_NAME \
              --scope-map _repositories_admin \
              --only-show-errors \
              --query "credentials.passwords[0].value" -o tsv)
```

## Managing Helm Charts in ACR

### View Repository Properties

```bash
az acr repository show \
  --name $ACR_NAME \
  --repository helm/hello-world
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

### View Chart Metadata

```bash
az acr manifest list-metadata \
  --registry $ACR_NAME \
  --name helm/hello-world
```

Output shows `configMediaType` of `application/vnd.cncf.helm.config.v1+json`:
```json
[
  {
    "configMediaType": "application/vnd.cncf.helm.config.v1+json",
    "digest": "sha256:0c03b71c225c3ddff...",
    "imageSize": 3301,
    "mediaType": "application/vnd.oci.image.manifest.v1+json",
    "tags": ["0.1.0"]
  }
]
```

### Delete a Chart

```bash
az acr repository delete --name $ACR_NAME --image helm/hello-world:0.1.0
```

## Migration from Helm 2 to Helm 3

### Step 1: Enable OCI Support (if using Helm < 3.8.0)

```console
export HELM_EXPERIMENTAL_OCI=1
```

For Helm 3.8.0+, OCI support is enabled by default.

### Step 2: List Existing Charts

```console
helm search repo myregistry
```

### Step 3: Pull Charts Locally

```console
helm pull myregistry/ingress-nginx
```

### Step 4: Push as OCI Artifacts

```console
az acr login --name $ACR_NAME
helm push ingress-nginx-3.20.1.tgz oci://$ACR_NAME.azurecr.io/helm
```

### Step 5: Verify Upload

```bash
az acr repository list --name $ACR_NAME
```

### Step 6: Remove Legacy Repository (Optional)

```console
helm repo remove $ACR_NAME
```

## Important Considerations

### OCI Artifact Limitations

1. **No Search via Helm**: OCI artifact repositories are not discoverable using:
   - `helm search`
   - `helm repo list`

   Use `az acr repository` commands instead.

2. **Tag Naming**: Use lowercase letters and numbers; separate words with dashes.

3. **Version Management**: Chart version comes from `Chart.yaml`, not the tag.

### Best Practices

1. **Use Specific Versions**: Always specify `--version` in commands
2. **Repository Namespacing**: Use paths like `helm/hello-world` for organization
3. **Automate with ACR Tasks**: Integrate Helm chart builds into CI/CD pipelines
4. **Access Control**: Use RBAC for granular permissions on chart repositories

## Integration with Kubernetes

### Azure Kubernetes Service (AKS)

Connect ACR to AKS for seamless chart deployment:
```bash
az aks update -n myAKSCluster -g myResourceGroup --attach-acr myACR
```

### Install from ACR

```console
helm install myrelease oci://$ACR_NAME.azurecr.io/helm/hello-world --version 0.1.0
```

### Verify Installation

```console
helm get manifest myrelease
helm list
```

### Uninstall

```console
helm uninstall myrelease
```

## Troubleshooting

### Common Issues

1. **OCI Support Not Enabled**: Upgrade to Helm 3.8.0+ or set `HELM_EXPERIMENTAL_OCI=1`

2. **Authentication Failures**: Ensure proper credentials and role assignments

3. **Push Failures**: Verify chart package integrity and naming conventions

4. **Version Conflicts**: Check `Chart.yaml` version matches expected tag

## Related Documentation

- [Helm Official Documentation](https://helm.sh/docs/)
- [OCI Artifacts in ACR](container-registry-manage-artifact.md)
- [ACR Authentication Options](container-registry-authentication.md)
- [ACR Tasks for Helm](container-registry-tasks-overview.md)

## Sources

- `/submodules/azure-management-docs/articles/container-registry/container-registry-helm-repos.md`
- `/submodules/azure-management-docs/articles/container-registry/container-registry-image-formats.md`
- `/submodules/acr/docs/artifact-media-types.json`
- `/submodules/acr/notifications/helm-repo-failure-20200918-.md`

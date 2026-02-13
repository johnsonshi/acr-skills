# ACR Transfer Troubleshooting Guide

## Overview

This document provides comprehensive troubleshooting guidance for ACR Transfer, covering common issues with export pipelines, import pipelines, storage access, and blob transfers.

---

## Quick Diagnostic Checklist

Before diving into specific issues, verify:

- [ ] Registry is Premium SKU
- [ ] SAS tokens are valid and not expired
- [ ] SAS tokens have correct permissions
- [ ] Pipeline identities have Key Vault access
- [ ] Storage container exists
- [ ] Artifacts exist in source registry
- [ ] Blob size does not exceed 8 GB per layer

---

## Common Error Categories

### 1. Template Deployment Failures

#### Symptoms
- ARM deployment fails
- Pipeline resource creation fails
- Error during `az deployment group create`

#### Solutions

**Check the pipelineRunErrorMessage:**
```azurecli
az acr pipeline-run show \
  --resource-group $MyRG \
  --registry $MyReg \
  --name $MyPipelineRun \
  --query response.pipelineRunErrorMessage
```

**Common ARM deployment errors:**
- Refer to [Troubleshoot ARM template deployments](https://docs.microsoft.com/azure/azure-resource-manager/templates/template-tutorial-troubleshoot)

---

### 2. Key Vault Access Problems

#### Error: 403 Forbidden when accessing Key Vault

**Cause:** Pipeline identity lacks permissions to read secrets.

**Solution:**

1. Verify pipeline identity exists:
```azurecli
# For Export Pipeline
az acr export-pipeline show \
  --resource-group $MyRG \
  --registry $MyReg \
  --name $MyPipeline \
  --query identity
```

2. Get the principal ID:
```azurecli
PRINCIPAL_ID=$(az acr export-pipeline show \
  --resource-group $MyRG \
  --registry $MyReg \
  --name $MyPipeline \
  --query identity.principalId \
  --output tsv)
```

3. Add Key Vault access policy:
```azurecli
az keyvault set-policy \
  --name $MyKeyvault \
  --secret-permissions get \
  --object-id $PRINCIPAL_ID
```

#### For User-Assigned Identity

If using user-assigned identity, grant permissions to that identity instead:

```azurecli
# Get user-assigned identity principal ID
UA_PRINCIPAL_ID=$(az identity show \
  --name myIdentity \
  --resource-group $MyRG \
  --query principalId \
  --output tsv)

az keyvault set-policy \
  --name $MyKeyvault \
  --secret-permissions get \
  --object-id $UA_PRINCIPAL_ID
```

---

### 3. Storage Access Problems

#### Error: 403 Forbidden from storage

**Causes and Solutions:**

| Cause | Solution |
|-------|----------|
| SAS token expired | Generate new SAS token with future expiry |
| Storage account keys rotated | Regenerate SAS token |
| Insufficient permissions | Verify SAS token permissions |
| Missing resource types | Ensure `srt=sco` in SAS token |
| HTTP-only token | Ensure `spr=https` in SAS token |

#### Validating SAS Token

Test the SAS token directly:

```azurecli
# Try to list blobs using the SAS token
az storage blob list \
  --container-name transfer \
  --account-name $MyStorageAccount \
  --sas-token "$MY_SAS_TOKEN" \
  --output table
```

Or test in browser:
1. Open Microsoft Edge InPrivate window
2. Navigate to: `https://<storage-account>.blob.core.windows.net/<container>/<blob>?<sas-token>`

#### SAS Token Permission Requirements

| Pipeline Type | Required Permissions | SAS Parameter |
|---------------|---------------------|---------------|
| Export | Read, Write, List, Add | `sp=alrw` |
| Import | Read, Delete, List | `sp=dlr` |
| Import (without delete) | Read, List | `sp=lr` |

#### Regenerating SAS Token

```azurecli
# Export SAS
EXPORT_SAS=?$(az storage container generate-sas \
  --name transfer \
  --account-name $SOURCE_SA \
  --expiry 2025-12-31 \
  --permissions alrw \
  --https-only \
  --output tsv)

# Update secret in Key Vault
az keyvault secret set \
  --name acrexportsas \
  --value "$EXPORT_SAS" \
  --vault-name $SOURCE_KV
```

---

### 4. Export/Import Blob Problems

#### Blob Not Created During Export

**Causes and Solutions:**

| Symptom | Cause | Solution |
|---------|-------|----------|
| No blob created | Container doesn't exist | Create the container |
| Empty blob | Artifacts not found | Verify artifact names/tags |
| Partial blob | Export failed mid-way | Check pipelineRunErrorMessage |

#### Blob Not Overwritten

**Cause:** `OverwriteBlobs` option not set.

**Solution:**
```azurecli
az acr export-pipeline create \
  --resource-group $MyRG \
  --registry $MyReg \
  --name $MyPipeline \
  --secret-uri https://$MyKV.vault.azure.net/secrets/$MySecret \
  --storage-container-uri https://$MyStorage.blob.core.windows.net/$MyContainer \
  --options OverwriteBlobs
```

#### Blob Not Deleted After Import

**Cause:** `DeleteSourceBlobOnSuccess` option not set or SAS lacks Delete permission.

**Solution:**
1. Add Delete permission to import SAS token
2. Enable DeleteSourceBlobOnSuccess option:
```azurecli
az acr import-pipeline create \
  ... \
  --options DeleteSourceBlobOnSuccess
```

---

### 5. Source Trigger Import Problems

#### Import Not Triggered Automatically

**Checklist:**

| Requirement | How to Verify |
|-------------|---------------|
| Source trigger enabled | Check pipeline configuration |
| Blob Last Modified < 60 days | Check blob properties |
| Valid ContentMD5 | Check blob properties |
| Correct blob metadata | Must have `"category":"acr-transfer-blob"` |
| SAS has List permission | Verify SAS token |

#### Verifying Blob Properties

```azurecli
az storage blob show \
  --account-name $TARGET_SA \
  --container-name transfer \
  --name myblob \
  --query "[properties.contentMd5, properties.lastModified, metadata]"
```

#### Fixing Old Blobs (> 60 days)

Refresh the Last Modified time:
```azurecli
az storage blob metadata update \
  --account-name $TARGET_SA \
  --container-name transfer \
  --name myblob \
  --metadata refreshed=$(date +%s)
```

#### Adding Required Metadata

If metadata was stripped during transfer:
```azurecli
az storage blob metadata update \
  --account-name $TARGET_SA \
  --container-name transfer \
  --name myblob \
  --metadata category=acr-transfer-blob
```

---

### 6. AzCopy Issues

#### General AzCopy Troubleshooting

See [Troubleshoot AzCopy issues](https://docs.microsoft.com/azure/storage/common/storage-use-azcopy-configure)

#### Common AzCopy Problems

| Issue | Solution |
|-------|----------|
| Authentication failure | Regenerate SAS tokens |
| Network timeout | Retry with `--cap-mbps` flag |
| Blob exists | Use `--overwrite true` |
| Permission denied | Check SAS permissions |

---

### 7. Artifact Transfer Problems

#### Not All Artifacts Transferred

**Checklist:**

| Issue | Solution |
|-------|----------|
| Spelling errors | Verify artifact names exactly |
| Case sensitivity | Use exact case for tags |
| Exceeds limit | Maximum 50 artifacts per run |
| Pipeline incomplete | Wait for completion |

#### Verifying Artifact Names

```azurecli
# List repositories in source
az acr repository list --name $SOURCE_ACR

# List tags for a repository
az acr repository show-tags --name $SOURCE_ACR --repository myimage
```

#### Checking Pipeline Run Status

```azurecli
az acr pipeline-run show \
  --resource-group $MyRG \
  --registry $MyReg \
  --name $MyPipelineRun
```

---

### 8. Air-Gapped Environment Issues

#### Error: Foreign Layers / mcr.microsoft.com References

**Cause:** Image manifest contains non-distributable layers referencing external registries (e.g., Windows base images).

**Symptoms:**
- Errors pulling images in isolated environment
- References to `mcr.microsoft.com` in error messages

**Solution:**

1. Check image manifest for external references:
```azurecli
az acr manifest show \
  --name $SOURCE_ACR \
  --image myimage:tag
```

2. Push non-distributable layers to your ACR before export:

   Edit Docker daemon configuration (`/etc/docker/daemon.json` or `C:\ProgramData\docker\config\daemon.json`):
   ```json
   {
     "allow-nondistributable-artifacts": ["myregistry.azurecr.io"]
   }
   ```

3. Restart Docker daemon

4. Push the image to include non-distributable layers

5. Then run the export pipeline

---

### 9. Permission Errors

#### Error: Authorization Denied / Scope Invalid

**Cause:** User lacks `Container Registry Transfer Pipeline Contributor` role.

**Solution:**

```azurecli
# Assign the role
az role assignment create \
  --role "Container Registry Transfer Pipeline Contributor" \
  --assignee <user-or-service-principal> \
  --scope /subscriptions/$SUB_ID
```

---

### 10. Pipeline Run Redeployment Issues

#### Error: Pipeline Run Already Exists

**Cause:** Redeploying a PipelineRun with identical properties.

**Solution:**

Use `forceUpdateTag` parameter:

```azurecli
# CLI
az acr pipeline-run create \
  ... \
  --force-redeploy

# ARM Template
CURRENT_DATETIME=$(date +"%Y-%m-%d:%T")

az deployment group create \
  --resource-group $MyRG \
  --template-file azuredeploy.json \
  --name myPipelineRun \
  --parameters azuredeploy.parameters.json \
  --parameters forceUpdateTag=$CURRENT_DATETIME
```

---

## Diagnostic Commands

### Get Pipeline Run Error Message

```azurecli
az acr pipeline-run show \
  --resource-group $MyRG \
  --registry $MyReg \
  --name $MyPipelineRun \
  --query response.pipelineRunErrorMessage \
  --output tsv
```

### Get Deployment Correlation ID

For support tickets, provide the correlation ID:

```azurecli
az deployment group show \
  --resource-group $MyRG \
  --name $DeploymentName \
  --query properties.correlationId \
  --output tsv
```

### Check Registry SKU

```azurecli
az acr show --name $MyReg --query sku.name
```

### List All Transfer Resources

```azurecli
# Export pipelines
az acr export-pipeline list --resource-group $MyRG --registry $MyReg

# Import pipelines
az acr import-pipeline list --resource-group $MyRG --registry $MyReg

# Pipeline runs
az acr pipeline-run list --resource-group $MyRG --registry $MyReg
```

---

## Error Message Reference

| Error Message | Likely Cause | Solution |
|---------------|--------------|----------|
| `403 Forbidden` (Key Vault) | Identity lacks KV access | Add access policy |
| `403 Forbidden` (Storage) | Invalid/expired SAS token | Regenerate SAS token |
| `authorization to perform denied` | Missing RBAC role | Assign Transfer Pipeline Contributor |
| `scope is invalid` | Missing RBAC role | Assign Transfer Pipeline Contributor |
| `artifact not found` | Typo in artifact name | Verify exact name/tag |
| `maximum artifacts exceeded` | >50 artifacts | Split into multiple runs |
| `layer size exceeded` | Layer >8 GB | Reduce layer size |

---

## Support

For issues not covered in this guide:
1. Collect the deployment correlation ID
2. Collect the pipelineRunErrorMessage
3. Contact the Azure Container Registry team

---

## References

### Source Files
- `/submodules/azure-management-docs/articles/container-registry/container-registry-transfer-troubleshooting.md`
- `/submodules/azure-management-docs/articles/container-registry/container-registry-faq.yml`
- `/submodules/azure-management-docs/articles/container-registry/container-registry-transfer-cli.md`

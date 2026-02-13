# ACR Artifact Streaming - Troubleshooting Guide

## Overview

This document provides comprehensive troubleshooting guidance for Azure Container Registry (ACR) Artifact Streaming, covering both current implementation issues and legacy Project Teleport problems.

## Common Symptoms

| Symptom | Possible Cause | Quick Check |
|---------|----------------|-------------|
| Conversion operation failed | Various errors | Check error code in operation output |
| AKS pod deployment failed | Auth/network/streaming issues | Check pod events and conditions |
| "UpgradeIfStreamableDisabled" condition | Image source or config issue | Verify image is from ACR |
| Slow image pulls despite streaming | No streaming artifact | List referrers for image |

## Conversion Operation Errors

### Error Code Reference

| Error Code | Error Message | Resolution |
|------------|---------------|------------|
| `UNKNOWN_ERROR` | Conversion operation failed due to an unknown error | Retry the operation. If unsuccessful, contact support |
| `RESOURCE_NOT_FOUND` | Target resource isn't found | Verify image digest, check for typos, ensure image exists in target region |
| `UNSUPPORTED_PLATFORM` | Conversion isn't supported for image platform | Only linux/amd64 supported; use compatible images |
| `NO_SUPPORTED_PLATFORM_FOUND` | No supported platforms in index | Multi-arch image lacks linux/amd64 platform |
| `UNSUPPORTED_MEDIATYPE` | MediaType not supported | Use supported manifest types (see below) |
| `UNSUPPORTED_ARTIFACT_TYPE` | ArtifactType not supported | Cannot convert streaming artifacts again |
| `IMAGE_NOT_RUNNABLE` | Conversion not supported for nonrunnable images | Only runnable linux/amd64 images supported |

### Supported Media Types

Conversion only works with these media types:
- `application/vnd.oci.image.manifest.v1+json`
- `application/vnd.oci.image.index.v1+json`
- `application/vnd.docker.distribution.manifest.v2+json`
- `application/vnd.docker.distribution.manifest.list.v2+json`

### Checking Conversion Status

```bash
az acr artifact-streaming operation show \
  --registry <registry-name> \
  --image <repo>:<tag>
```

### Canceling Failed Conversion

```bash
az acr artifact-streaming operation cancel \
  --registry <registry-name> \
  --repository <repo> \
  --id <operation-id>
```

## AKS Pod Deployment Failures

### Image Pull Errors

**Example Error:**
```
Failed to pull image "mystreamingtest.azurecr.io/jupyter/all-spark-notebook:latest":
rpc error: code = Unknown desc = failed to pull and unpack image
"mystreamingtest.azurecr.io/jupyter/all-spark-notebook:latest":
failed to resolve reference: unexpected status from HEAD request to
http://localhost:8578/v2/jupyter/all-spark-notebook/manifests/latest?ns=mystreamingtest.azurecr.io:
503 Service Unavailable
```

**Troubleshooting Steps:**

1. **Verify ACR Permissions**
   ```bash
   az aks check-acr \
     --resource-group <rg> \
     --name <cluster> \
     --acr <registry>.azurecr.io
   ```

2. **Verify ACR Attachment**
   ```bash
   az aks show \
     --resource-group <rg> \
     --name <cluster> \
     --query "identityProfile.kubeletidentity.clientId" -o tsv
   ```

3. **Check AcrPull Role Assignment**
   ```bash
   az role assignment list \
     --assignee <kubelet-identity> \
     --scope <acr-resource-id>
   ```

### "UpgradeIfStreamableDisabled" Pod Condition

**Cause:** AKS detected the image is not from an Azure Container Registry or streaming is not available.

**Resolution:**
1. Verify image reference uses ACR URL (not Docker Hub or other registries)
2. Verify streaming artifact exists for the image
3. Check if ACR is properly attached to AKS

### No Streaming Information in Events (Digest Deployments)

When deploying with image digest instead of tag:
```yaml
image: myacr.azurecr.io/repo@sha256:abc123...
```

**Behavior:** Pod events won't show streaming-related messages, but streaming still occurs if available.

**Verification:** Check pod startup time - fast startup indicates streaming is working.

## Verifying Streaming Artifacts

### Check if Streaming Artifact Exists

```bash
az acr manifest list-referrers \
  --registry <registry-name> \
  -n <repo>:<tag>
```

**Expected Output (streaming enabled):**
```json
[
  {
    "artifactType": "application/vnd.azure.artifact.streaming.v1",
    "digest": "sha256:...",
    ...
  }
]
```

**Empty Output:** Streaming artifact not created - run conversion:
```bash
az acr artifact-streaming create --image <repo>:<tag>
```

### Check Repository Streaming State

```bash
az acr repository show \
  --repository <repo> \
  -o jsonc
```

Look for streaming-related attributes in the output.

## Prerequisites Verification

### Azure CLI Version

```bash
az version
```

Requires version 2.54.0 or higher.

### AKS Kubernetes Version

```bash
kubectl version --short
```

Requires 1.26 or higher for artifact streaming.

### Node OS Version

```bash
kubectl get nodes -o wide
```

Ubuntu nodes require 20.04 or higher.

## Soft Delete Interaction Issues

### Issue: Restored Image Missing Streaming Artifact

**Symptom:** After restoring a soft-deleted image, only the original is available.

**Cause:** Only original versions can be restored during retention period; streaming artifacts are not recoverable.

**Resolution:** Re-create streaming artifact after restoration:
```bash
az acr artifact-streaming create --image <repo>:<tag>
```

---

## Legacy: Project Teleport Troubleshooting (Deprecated)

> **Note**: Project Teleport was deprecated in November 2023. These steps are preserved for historical reference only.

### Collecting Teleportd Logs on AKS

The teleportd daemon runs on AKS nodes. Logs are available via journald.

**Option 1: SSH to Node**
```bash
journalctl -n all -u teleportd
```

**Option 2: Sidecar Container**

Deploy log collector:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: teleport-logs
spec:
  containers:
  - name: log-reader
    image: busybox
    args: [/bin/sh, -c, '/bin/journalctl -n all -u teleportd -f']
    volumeMounts:
    - name: rootfs
      mountPath: /
  # nodeSelector:
  #   teleport: "true"
  volumes:
  - name: rootfs
    hostPath:
      path: /
      type: Directory
```

Collect logs:
```bash
kubectl logs teleport-logs > ./teleport-daemon.log
```

### Kubernetes Events for Teleport

**Check Specific Pod:**
```bash
kubectl describe pod <pod-name>
```

**Check All Events:**
```bash
kubectl get events
```

Events sourced from teleportd include image/tag information and success/failure status.

### Checking Layer Expansion Status

Using check-expansion.sh script:

```bash
# Set credentials
export ACR_USER=teleport-token
export ACR_PWD=$(az acr token create \
  --name teleport-token \
  --registry $ACR \
  --scope-map _repositories_pull \
  --query credentials.passwords[0].value -o tsv)

# Check expansion
./check-expansion.sh <registry> <repo> <tag> --debug
```

**Status Codes:**
| HTTP Status | Meaning |
|-------------|---------|
| 200 | Layers ready for teleportation |
| 409 | Layers currently expanding |
| 404 | Teleport not enabled for this image |

### Repository Not Teleport-Enabled

**Check:**
```bash
az acr repository show \
  --repository <repo> \
  -o jsonc
```

Look for `"teleportEnabled": true` in `changeableAttributes`.

**Resolution (if false):**
- Push/re-push image to trigger teleport enablement
- Use edit-teleport-attribute.sh to manually enable

### Layer Mount Failures

**Symptom:** Image took too long despite events showing success.

**Cause:** Individual layer mount failures not reflected in high-level events.

**Debug:** Check teleportd logs for layer-specific errors.

### 10 Repository Limit

Project Teleport had a limit of 10 teleport-enabled repositories per registry.

**Check enabled repos:**
```bash
./find-teleport-enabled-repositories.sh <registry-name>
```

**Resolution:** Disable teleport on unused repositories:
```bash
./edit-teleport-attribute.sh <registry> <repo> disable
```

## General Best Practices for Troubleshooting

### 1. Start with Event Logs

```bash
kubectl describe pod <pod-name>
kubectl get events --sort-by='.lastTimestamp'
```

### 2. Verify Prerequisites

- ACR Premium tier
- K8s 1.26+
- Linux AMD64 images
- ACR attached to AKS

### 3. Check Streaming Artifact Availability

```bash
az acr manifest list-referrers -n <repo>:<tag>
```

### 4. Compare Pull Times

Deploy test pods and compare startup times with and without streaming to verify functionality.

### 5. Review ACR Metrics

Use Azure Monitor to track:
- Registry storage usage
- Pull operations
- Error rates

## Getting Support

- **Azure Support**: File support ticket via Azure Portal
- **Documentation**: [aka.ms/acr/artifact-streaming](https://aka.ms/acr/artifact-streaming)
- **GitHub Issues**: [Azure/acr](https://github.com/Azure/acr/issues)

## Source Files

- Current Troubleshooting: `submodules/azure-management-docs/articles/container-registry/troubleshoot-artifact-streaming.md`
- Legacy Teleportd Logs: `submodules/acr/docs/teleport/collecting-teleportd-logs-aks.md`
- Check Expansion Script: `submodules/acr/docs/teleport/check-expansion.sh`
- Repository Management: `submodules/acr/docs/teleport/teleport-repository-management.md`

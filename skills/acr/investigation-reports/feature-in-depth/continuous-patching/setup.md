# ACR Continuous Patching - Setup Guide

## Prerequisites

### Azure Requirements
- Azure subscription (not Free Trial - free credits are not supported)
- Existing Resource Group
- Azure Container Registry with ACR Tasks enabled
  - Note: ACR Tasks is not supported in the free tier

### CLI Requirements
- Azure CLI version 2.15.0 or later
- Azure Cloud Shell or local Azure CLI installation

## Installation

### Install the CLI Extension

For public preview (current):
```azurecli
az extension add -n acrcssc
```

For private preview (earlier versions):
```sh
az extension add --source aka.ms/acr/patching/wheel
```

## Configuration File

### Create Configuration JSON

Create a file named `continuouspatching.json`:

```json
{
    "version": "v1",
    "tag-convention": "<incremental|floating>",
    "repositories": [{
        "repository": "<Repository Name>",
        "tags": ["<comma-separated-tags>"],
        "enabled": <true|false>
    }]
}
```

### Schema Reference

| Field | Required | Description |
|-------|----------|-------------|
| `version` | Yes | Schema version. Always use `v1`. Do not modify. |
| `tag-convention` | No | `incremental` (default) or `floating` |
| `repositories` | Yes | Array of repository configurations |
| `repositories[].repository` | Yes | Repository name |
| `repositories[].tags` | Yes | Array of tags. Use `*` for all tags. |
| `repositories[].enabled` | Yes | Boolean to enable/disable repository |

### Example Configurations

**Patch all tags in a repository:**
```json
{
    "version": "v1",
    "tag-convention": "incremental",
    "repositories": [{
        "repository": "python",
        "tags": ["*"],
        "enabled": true
    }]
}
```

**Patch specific tags:**
```json
{
    "version": "v1",
    "tag-convention": "incremental",
    "repositories": [{
        "repository": "ubuntu",
        "tags": ["jammy-20240111", "jammy-20240125"],
        "enabled": true
    }]
}
```

**Multiple repositories with different settings:**
```json
{
    "version": "v1",
    "tag-convention": "floating",
    "repositories": [
        {
            "repository": "python",
            "tags": ["*"],
            "enabled": true
        },
        {
            "repository": "ubuntu",
            "tags": ["jammy-20240111", "jammy-20240125"],
            "enabled": true
        },
        {
            "repository": "nginx",
            "tags": ["1.25", "1.24"],
            "enabled": false
        }
    ]
}
```

## Workflow Creation

### Step 1: Login

```azurecli
# Login to Azure
az login

# Login to your registry
az acr login -n <myRegistry>
```

### Step 2: Dry Run (Recommended)

Verify which images will be selected before creating the workflow:

```azurecli
az acr supply-chain workflow create \
    -r myRegistry \
    -g myResourceGroup \
    -t continuouspatchv1 \
    --config ./continuouspatching.json \
    --schedule 1d \
    --dry-run
```

**Expected output:**
```
Ubuntu: jammy-20240111
Ubuntu: jammy-20240125
```

### Step 3: Create Workflow

Once satisfied with the dry-run results, create the workflow:

```azurecli
az acr supply-chain workflow create \
    -r myRegistry \
    -g myResourceGroup \
    -t continuouspatchv1 \
    --config ./continuouspatching.json \
    --schedule 1d
```

**With immediate execution:**
```azurecli
az acr supply-chain workflow create \
    -r myRegistry \
    -g myResourceGroup \
    -t continuouspatchv1 \
    --config ./continuouspatching.json \
    --schedule 1d \
    --run-immediately
```

### Schedule Options

| Value | Frequency | Example Use Case |
|-------|-----------|------------------|
| `1d` | Daily | High-security environments |
| `7d` | Weekly | Standard production workloads |
| `14d` | Bi-weekly | Lower-risk environments |
| `30d` | Monthly | Minimal patching requirements |

## Verification

### Azure Portal

1. Navigate to your registry in Azure Portal
2. Under **Services** > **Repositories**, verify `csscpolicies/patchpolicy` exists
3. Under **Services** > **Tasks**, verify these tasks exist:
   - `cssc-trigger-workflow`
   - `cssc-scan-image`
   - `cssc-patch-image`
4. Click **Runs** to view task execution status

### CLI

View workflow details:
```azurecli
az acr supply-chain workflow show \
    -r myRegistry \
    -g myResourceGroup \
    -t continuouspatchv1
```

## Updating Configuration

### Update Schedule or Configuration

```azurecli
az acr supply-chain workflow update \
    -r myRegistry \
    -g myResourceGroup \
    -t continuouspatchv1 \
    --config ./continuouspatching.json \
    --schedule 7d
```

**Recommended process for configuration changes:**
1. Edit the JSON configuration file
2. Run a dry-run to verify changes
3. Execute the update command

## Deletion

### Delete the Workflow

```azurecli
az acr supply-chain workflow delete \
    -r myregistry \
    -g myresourcegroup \
    -t continuouspatchv1
```

**What gets deleted:**
- `csscpolicies/patchpolicy` repository
- All three workflow tasks
- Queued runs
- Previous logs

## Incompatibilities

The following scenarios are **not supported**:

| Scenario | Status |
|----------|--------|
| ABAC-enabled registries | Incompatible |
| Repositories with PTC (Pull Through Cache) rules | Incompatible |
| Free Trial subscriptions | Not supported |
| Multi-arch images | Not supported |
| Windows images | Not supported |

## Image Limits

- **100 image limit** enforced for continuous patching
- Plan your tag selection carefully if you have many images

## Source Files

- MS Learn How-To: `submodules/azure-management-docs/articles/container-registry/how-to-continuous-patching.md`
- ACR Repo Docs: `submodules/acr/docs/preview/continuous-patching/README.md`

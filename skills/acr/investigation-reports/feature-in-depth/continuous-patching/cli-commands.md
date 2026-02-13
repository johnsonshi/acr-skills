# ACR Continuous Patching - CLI Commands Reference

## Extension Installation

### Install Extension (Public Preview)
```azurecli
az extension add -n acrcssc
```

### Install Extension (Private Preview)
```sh
az extension add --source aka.ms/acr/patching/wheel
```

## Workflow Management Commands

### Create Workflow

**Syntax:**
```azurecli
az acr supply-chain workflow create \
    -r <registryname> \
    -g <resourcegroupname> \
    -t continuouspatchv1 \
    --config <JSONfilepath> \
    --schedule <number of days> \
    [--dry-run] \
    [--run-immediately]
```

**Parameters:**

| Parameter | Required | Description |
|-----------|----------|-------------|
| `-r, --registry` | Yes | Registry name |
| `-g, --resource-group` | Yes | Resource group name |
| `-t, --type` | Yes | Workflow type. Use `continuouspatchv1` |
| `--config` | Yes | Path to JSON configuration file |
| `--schedule` | Yes | Patching frequency (1d to 30d) |
| `--dry-run` | No | Preview which images will be selected |
| `--run-immediately` | No | Trigger immediate patch run |

**Examples:**

Dry run to verify configuration:
```azurecli
az acr supply-chain workflow create \
    -r myRegistry \
    -g myResourceGroup \
    -t continuouspatchv1 \
    --config ./continuouspatching.json \
    --schedule 1d \
    --dry-run
```

Create with daily patching:
```azurecli
az acr supply-chain workflow create \
    -r myRegistry \
    -g myResourceGroup \
    -t continuouspatchv1 \
    --config ./continuouspatching.json \
    --schedule 1d
```

Create with immediate execution:
```azurecli
az acr supply-chain workflow create \
    -r myRegistry \
    -g myResourceGroup \
    -t continuouspatchv1 \
    --config ./continuouspatching.json \
    --schedule 7d \
    --run-immediately
```

### Show Workflow

**Syntax:**
```azurecli
az acr supply-chain workflow show \
    -r <registryname> \
    -g <resourcegroupname> \
    -t continuouspatchv1
```

**Output includes:**
- Schedule configuration
- Creation date
- Last modified date
- System metadata

**Example:**
```azurecli
az acr supply-chain workflow show \
    -r myRegistry \
    -g myResourceGroup \
    -t continuouspatchv1
```

### Update Workflow

**Syntax:**
```azurecli
az acr supply-chain workflow update \
    -r <registryname> \
    -g <resourcegroupname> \
    -t continuouspatchv1 \
    [--config <JSONfilepath>] \
    [--schedule <number of days>]
```

**Examples:**

Update schedule only:
```azurecli
az acr supply-chain workflow update \
    -r myRegistry \
    -g myResourceGroup \
    -t continuouspatchv1 \
    --schedule 7d
```

Update configuration and schedule:
```azurecli
az acr supply-chain workflow update \
    -r myRegistry \
    -g myResourceGroup \
    -t continuouspatchv1 \
    --config ./continuouspatching.json \
    --schedule 14d
```

### Delete Workflow

**Syntax:**
```azurecli
az acr supply-chain workflow delete \
    -r <registryname> \
    -g <resourcegroupname> \
    -t continuouspatchv1
```

**Example:**
```azurecli
az acr supply-chain workflow delete \
    -r myregistry \
    -g myresourcegroup \
    -t continuouspatchv1
```

**What gets deleted:**
- `csscpolicies/patchpolicy` repository
- All workflow tasks
- Queued runs
- Historical logs

### List Workflow Tasks

**Syntax:**
```azurecli
az acr supply-chain workflow list \
    -r <registryname> \
    -g <resourcegroupname> \
    -t continuouspatchv1 \
    [--run-status <status>]
```

**Status filter options:**

| Status | Description |
|--------|-------------|
| `Succeeded` | Successfully completed |
| `Failed` | Failed execution |
| `Running` | Currently executing |
| `Queued` | Waiting to run |
| `Skipped` | Skipped (no vulnerabilities or EOSL) |
| `Unknown` | Status not determined |

**Output includes:**
- Image name and tag
- Workflow type
- Scan status and date
- Scan task ID
- Patch status and date
- Patched image name/tag
- Patch task ID

**Examples:**

List all tasks:
```azurecli
az acr supply-chain workflow list \
    -r myRegistry \
    -g myResourceGroup \
    -t continuouspatchv1
```

List failed tasks only:
```azurecli
az acr supply-chain workflow list \
    -r myRegistry \
    -g myResourceGroup \
    -t continuouspatchv1 \
    --run-status Failed
```

List successful tasks:
```azurecli
az acr supply-chain workflow list \
    -r myRegistry \
    -g myResourceGroup \
    -t continuouspatchv1 \
    --run-status Succeeded
```

### Cancel Running Tasks

**Syntax:**
```azurecli
az acr supply-chain workflow cancel-run \
    -r <registryname> \
    -g <resourcegroupname> \
    --type continuouspatchv1
```

**Behavior:**
- Cancels tasks with status: Running, Queued, Started
- Only affects current schedule period
- Tasks are re-queued for next scheduled run

**Example:**
```azurecli
az acr supply-chain workflow cancel-run \
    -r myRegistry \
    -g myResourceGroup \
    --type continuouspatchv1
```

## Troubleshooting Commands

### List Failed Tasks

```azurecli
az acr task list-runs \
    -r <registryname> \
    -n cssc-patch-image \
    --run-status Failed \
    --top 10
```

### View Task Logs

```azurecli
az acr task logs \
    -r <registryname> \
    --run-id <run-id>
```

### Help Commands

View all options for any command:

```azurecli
az acr supply-chain workflow create --help
az acr supply-chain workflow show --help
az acr supply-chain workflow update --help
az acr supply-chain workflow delete --help
az acr supply-chain workflow list --help
az acr supply-chain workflow cancel-run --help
```

## Sample CLI Output

### Successful Scan and Patch
```
image: import:dotnetapp-manual
        scan status: Succeeded
        scan date: 2024-09-13 21:05:58.841962+00:00
        scan task ID: dt21
        patch status: Succeeded
        patch date: 2024-09-13 21:07:32.841962+00:00
        patch task ID: xyz2
        last patched image: import:dotnetapp-manual-patched
        workflow type: continuouspatchv1
```

### Scan Succeeded, No Patch Needed
```
image: import:dotnetapp-manual
        scan status: Succeeded
        scan date: 2024-09-13 21:05:58.841962+00:00
        scan task ID: dt21
        patch status: Skipped
        skipped patch reason: no vulnerability found in the image
        patch date: ---Not Available---
        patch task ID: ---Not Available---
        last patched image: ---Not Available---
        workflow type: continuouspatchv1
```

### Scan Failed
```
image: import:dotnetapp-manual
        scan status: Failed
        scan date: 2024-09-13 21:05:58.841962+00:00
        scan task ID: dt21
        patch status: ---Not Available---
        patch date: ---Not Available---
        patch task ID: ---Not Available---
        last patched image: ---Not Available---
        workflow type: continuouspatchv1
```

### Patch Failed
```
image: import:dotnetapp-manual
        scan status: Succeeded
        scan date: 2024-09-13 21:05:58.841962+00:00
        scan task ID: dt21
        patch status: Failed
        patch date: 2024-09-13 21:07:32.841962+00:00
        patch task ID: xyz2
        last patched image: ---No patch image available---
        workflow type: continuouspatchv1
```

### Currently Running
```
image: import:dotnetapp-manual
        scan status: Running
        scan date: 2024-09-13 21:05:58.841962+00:00
        scan task ID: dt21
        patch status: ---Not Available---
        patch date: ---Not Available---
        patch task ID: ---Not Available---
        last patched image: ---Not Available---
        workflow type: continuouspatchv1
```

## Configuration JSON Schema

### Full Schema
```json
{
    "version": "v1",
    "tag-convention": "incremental|floating",
    "repositories": [
        {
            "repository": "string",
            "tags": ["string", "*"],
            "enabled": true|false
        }
    ]
}
```

### Complete Example
```json
{
    "version": "v1",
    "tag-convention": "incremental",
    "repositories": [
        {
            "repository": "python",
            "tags": ["*"],
            "enabled": true
        },
        {
            "repository": "ubuntu",
            "tags": ["jammy-20240111", "jammy-20240125", "noble"],
            "enabled": true
        },
        {
            "repository": "nginx",
            "tags": ["1.25-alpine", "1.24-alpine"],
            "enabled": true
        },
        {
            "repository": "deprecated-app",
            "tags": ["*"],
            "enabled": false
        }
    ]
}
```

## Support Contact

For issues not resolved by troubleshooting:
- Email: acr-patching-preview@microsoft.com

## Source Files

- MS Learn How-To: `submodules/azure-management-docs/articles/container-registry/how-to-continuous-patching.md`
- ACR Repo Docs: `submodules/acr/docs/preview/continuous-patching/README.md`

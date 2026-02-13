# ACR Continuous Patching Skill

This skill provides comprehensive knowledge about Azure Container Registry continuous patching.

## When to Use This Skill

Use this skill when answering questions about:
- Automated vulnerability patching
- OS-level patching
- Copa/Copacetic
- Trivy scanning

## Overview

Continuous patching automatically detects and remediates OS-level vulnerabilities in container images.

**Status:** Preview

## How It Works

```
┌───────────────┐    ┌───────────────┐    ┌───────────────┐
│   Trigger     │───▶│    Trivy      │───▶│    Copa       │
│   (Schedule)  │    │   (Scan)      │    │   (Patch)     │
└───────────────┘    └───────────────┘    └───────────────┘
                                                  │
                                                  ▼
                                          ┌───────────────┐
                                          │ Patched Image │
                                          └───────────────┘
```

1. **Trigger**: Scheduled workflow runs
2. **Scan**: Trivy scans for vulnerabilities
3. **Patch**: Copa applies OS package patches
4. **Push**: Patched image pushed to registry

## Quick Setup

### Install Extension
```bash
az extension add --name acrcssc
```

### Create Workflow
```bash
# Create patching workflow
az acr cssc workflow create \
  --registry myregistry \
  --name myworkflow \
  --config-file workflow-config.json
```

### Configuration File
```json
{
  "version": "v1",
  "repositories": [
    {
      "repository": "myapp",
      "tags": ["latest", "v1"],
      "schedule": "0 2 * * 0",
      "scanScheduleMultiplier": 1
    }
  ],
  "continuousMode": {
    "floatingTag": {
      "enabled": true,
      "suffix": "-patched"
    }
  }
}
```

## Tagging Conventions

| Mode | Original | Patched |
|------|----------|---------|
| Incremental | myapp:v1 | myapp:v1-1, myapp:v1-2 |
| Floating | myapp:v1 | myapp:v1-patched |

## Supported Images

### Linux Distributions
| Distro | Package Manager |
|--------|-----------------|
| Ubuntu | apt |
| Debian | apt |
| Alpine | apk |
| RHEL/CentOS | yum/dnf |
| Fedora | dnf |
| openSUSE | zypper |

### Architectures
- AMD64 ✅
- ARM64 ✅
- Multi-arch ❌ (not supported)

## Workflow Management

```bash
# List workflows
az acr cssc workflow list --registry myregistry

# Show workflow
az acr cssc workflow show \
  --registry myregistry \
  --name myworkflow

# Update workflow
az acr cssc workflow update \
  --registry myregistry \
  --name myworkflow \
  --config-file updated-config.json

# Delete workflow
az acr cssc workflow delete \
  --registry myregistry \
  --name myworkflow

# Cancel run
az acr cssc workflow cancel-run \
  --registry myregistry \
  --name myworkflow \
  --run-id abc123
```

## Schedule Format

Cron expression: `minute hour day month weekday`

| Example | Description |
|---------|-------------|
| `0 2 * * 0` | 2 AM every Sunday |
| `0 0 1 * *` | Midnight on 1st of month |
| `0 */6 * * *` | Every 6 hours |

## What Gets Patched

✅ **Patched:**
- OS package vulnerabilities
- System library updates

❌ **Not Patched:**
- Application dependencies
- Custom binaries
- Base image changes
- EOSL (End of Service Life) images

## Pricing

- ACR Tasks compute time: ~$0.0001/second
- No additional feature cost
- Premium SKU recommended

## Incompatibilities

- ABAC-enabled registries
- Personal Trial Cloud (PTC)
- Free Trial subscriptions
- CMK-encrypted registries (some limitations)

## Reference Documentation

- **Investigation reports**: `agent-investigation-reports/acr-features/continuous-patching/`
- **MS Learn docs**: `azure-management-docs/articles/container-registry/how-to-continuous-patching.md`

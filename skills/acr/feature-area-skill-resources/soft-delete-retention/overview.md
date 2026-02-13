# ACR Soft Delete & Retention Skill

This skill provides comprehensive knowledge about Azure Container Registry lifecycle management features.

## When to Use This Skill

Use this skill when answering questions about:
- Soft delete policy
- Retention policy
- ACR Purge (auto-purge)
- Recovering deleted artifacts

## Features Overview

| Feature | SKU | Status | Purpose |
|---------|-----|--------|---------|
| Soft Delete | All | Preview | Recover deleted artifacts |
| Retention Policy | Premium | Preview | Auto-delete untagged manifests |
| ACR Purge | All | GA | Scheduled cleanup |

> **Important:** Soft Delete and Retention Policy are **mutually exclusive**!

## Soft Delete

### Enable Soft Delete
```bash
az acr config soft-delete update \
  --registry myregistry \
  --status enabled \
  --days 7
```

### View Soft-Deleted Artifacts
```bash
az acr manifest list-deleted \
  --registry myregistry \
  --name myrepo
```

### Restore Artifact
```bash
az acr manifest restore \
  --registry myregistry \
  --name myrepo \
  --digest sha256:abc123...
```

### Limitations
- Not compatible with geo-replication
- Not compatible with zone redundancy
- Not compatible with artifact cache

## Retention Policy (Premium)

### Enable Retention
```bash
az acr config retention update \
  --registry myregistry \
  --status enabled \
  --days 7 \
  --type UntaggedManifests
```

### Check Status
```bash
az acr config retention show --registry myregistry
```

### Key Behavior
- Only affects **untagged** manifests
- Only applies to manifests created **after** policy enabled
- Digest-based pulls may fail for deleted manifests

## ACR Purge (Recommended)

Most flexible cleanup option using ACR Tasks:

### On-Demand Purge
```bash
# Purge images older than 30 days
az acr run \
  --registry myregistry \
  --cmd "acr purge --filter 'myrepo:.*' --ago 30d" \
  /dev/null

# Dry run (preview what would be deleted)
az acr run \
  --registry myregistry \
  --cmd "acr purge --filter 'myrepo:.*' --ago 30d --dry-run" \
  /dev/null

# Include untagged manifests
az acr run \
  --registry myregistry \
  --cmd "acr purge --filter 'myrepo:.*' --ago 30d --untagged" \
  /dev/null
```

### Scheduled Purge
```bash
az acr task create \
  --name daily-purge \
  --registry myregistry \
  --cmd "acr purge --filter '.*:.*' --ago 90d --untagged" \
  --context /dev/null \
  --schedule "0 1 * * *"
```

### Filter Patterns
| Pattern | Matches |
|---------|---------|
| `myrepo:.*` | All tags in myrepo |
| `myrepo:v1.*` | Tags starting with v1 |
| `.*:.*` | All repos, all tags |
| `myrepo:^(?!latest)` | All except 'latest' |

### Age Format
| Format | Duration |
|--------|----------|
| `1d` | 1 day |
| `7d` | 7 days |
| `30d` | 30 days |
| `1h` | 1 hour |

## Decision Matrix

| Scenario | Recommendation |
|----------|----------------|
| Need to recover deleted images | Soft Delete |
| Auto-clean untagged manifests | Retention Policy |
| Custom cleanup rules | ACR Purge |
| Geo-replicated registry | ACR Purge only |

## Manual Delete

```bash
# Delete by tag
az acr repository delete \
  --name myregistry \
  --image myrepo:v1

# Delete by digest
az acr repository delete \
  --name myregistry \
  --image myrepo@sha256:abc123...

# Delete entire repository
az acr repository delete \
  --name myregistry \
  --repository myrepo
```

## Image Locking

Prevent deletion:
```bash
az acr repository update \
  --name myregistry \
  --image myrepo:v1 \
  --delete-enabled false
```

## Reference Documentation

- **Investigation reports**: `agent-investigation-reports/acr-features/soft-delete-retention/`
- **MS Learn docs**: `azure-management-docs/articles/container-registry/container-registry-soft-delete-policy.md`

# ACR Soft Delete and Retention - CLI Commands Reference

This document provides a comprehensive reference of all Azure CLI commands related to soft delete, retention policy, auto-purge, and related image lifecycle management features.

## Table of Contents

1. [Soft Delete Commands](#soft-delete-commands)
2. [Retention Policy Commands](#retention-policy-commands)
3. [ACR Purge Commands](#acr-purge-commands)
4. [Manual Delete Commands](#manual-delete-commands)
5. [Image Locking Commands](#image-locking-commands)
6. [Utility Commands](#utility-commands)

---

## Soft Delete Commands

### Configure Soft Delete Policy

```bash
# Enable soft delete with custom retention
az acr config soft-delete update \
  --registry <registry-name> \
  --status enabled \
  --days <1-90>

# Disable soft delete
az acr config soft-delete update \
  --registry <registry-name> \
  --status disabled

# Show soft delete configuration
az acr config soft-delete show \
  --registry <registry-name>
```

### List Soft-Deleted Artifacts

```bash
# List soft-deleted repositories
az acr repository list-deleted \
  --name <registry-name>

# List soft-deleted manifests in a repository
az acr manifest list-deleted \
  --registry <registry-name> \
  --name <repository-name>

# List soft-deleted tags in a repository
az acr manifest list-deleted-tags \
  --registry <registry-name> \
  --name <repository-name>

# Filter soft-deleted tags by tag name
az acr manifest list-deleted-tags \
  --registry <registry-name> \
  --name <repository-name>:<tag>
```

### Restore Soft-Deleted Artifacts

```bash
# Restore by tag and digest
az acr manifest restore \
  --registry <registry-name> \
  --name <repository>:<tag> \
  --digest <sha256:digest>

# Restore most recently deleted manifest by tag
az acr manifest restore \
  --registry <registry-name> \
  --name <repository>:<tag>

# Force restore (overwrite existing tag)
az acr manifest restore \
  --registry <registry-name> \
  --name <repository>:<tag> \
  --digest <sha256:digest> \
  --force
```

---

## Retention Policy Commands

### Configure Retention Policy

```bash
# Enable retention policy
az acr config retention update \
  --registry <registry-name> \
  --status enabled \
  --days <0-365> \
  --type UntaggedManifests

# Disable retention policy
az acr config retention update \
  --registry <registry-name> \
  --status disabled \
  --type UntaggedManifests

# Show retention policy configuration
az acr config retention show \
  --registry <registry-name>
```

### Verify Retention Policy

```bash
# List manifests in a repository
az acr manifest list-metadata \
  --name <repository-name> \
  --registry <registry-name>

# Check for untagged manifests
az acr manifest list-metadata \
  --name <repository-name> \
  --registry <registry-name> \
  --query "[?tags==null || length(tags)==\`0\`]"
```

---

## ACR Purge Commands

### On-Demand Purge

```bash
# Basic purge - delete tags older than specified duration
az acr run \
  --cmd "acr purge --filter '<repository>:.*' --ago <duration>" \
  --registry <registry-name> \
  /dev/null

# Purge with untagged manifests
az acr run \
  --cmd "acr purge --filter '<repository>:.*' --ago <duration> --untagged" \
  --registry <registry-name> \
  /dev/null

# Dry run (preview without deletion)
az acr run \
  --cmd "acr purge --filter '<repository>:.*' --ago <duration> --dry-run" \
  --registry <registry-name> \
  /dev/null

# Keep N most recent tags
az acr run \
  --cmd "acr purge --filter '<repository>:.*' --ago <duration> --keep <N>" \
  --registry <registry-name> \
  /dev/null

# Multiple repository filters
az acr run \
  --cmd "acr purge --filter 'repo1:.*' --filter 'repo2:.*' --ago <duration>" \
  --registry <registry-name> \
  /dev/null

# Extended timeout for large purges
az acr run \
  --cmd "acr purge --filter '<repository>:.*' --ago <duration> --untagged" \
  --registry <registry-name> \
  --timeout 3600 \
  /dev/null
```

### Scheduled Purge Tasks

```bash
# Create scheduled purge task
az acr task create \
  --name <task-name> \
  --cmd "acr purge --filter '<repository>:.*' --ago <duration>" \
  --schedule "<cron-expression>" \
  --registry <registry-name> \
  --context /dev/null

# Show task configuration
az acr task show \
  --name <task-name> \
  --registry <registry-name>

# List task runs
az acr task list-runs \
  --name <task-name> \
  --registry <registry-name> \
  --output table

# View task run logs
az acr task logs \
  --registry <registry-name> \
  --run-id <run-id>

# Manually trigger a task
az acr task run \
  --name <task-name> \
  --registry <registry-name>
```

### Timer Management

```bash
# List timers for a task
az acr task timer list \
  --name <task-name> \
  --registry <registry-name>

# Add a timer to a task
az acr task timer add \
  --name <task-name> \
  --registry <registry-name> \
  --timer-name <timer-name> \
  --schedule "<cron-expression>"

# Update a timer
az acr task timer update \
  --name <task-name> \
  --registry <registry-name> \
  --timer-name <timer-name> \
  --schedule "<new-cron-expression>"

# Remove a timer
az acr task timer remove \
  --name <task-name> \
  --registry <registry-name> \
  --timer-name <timer-name>
```

---

## Manual Delete Commands

### Delete Repository

```bash
# Delete entire repository (all images, tags, manifests)
az acr repository delete \
  --name <registry-name> \
  --repository <repository-name> \
  --yes
```

### Delete by Tag

```bash
# Delete image by tag (deletes manifest and unique layers)
az acr repository delete \
  --name <registry-name> \
  --image <repository>:<tag> \
  --yes
```

### Delete by Manifest Digest

```bash
# List manifest digests
az acr manifest list-metadata \
  --name <repository-name> \
  --registry <registry-name>

# Delete by digest
az acr repository delete \
  --name <registry-name> \
  --image <repository>@<sha256:digest> \
  --yes
```

### Untag (Remove Tag Reference Only)

```bash
# Untag an image (manifest remains as untagged)
az acr repository untag \
  --name <registry-name> \
  --image <repository>:<tag>
```

### Delete by Timestamp

```bash
# List manifests older than a date
az acr manifest list-metadata \
  --name <repository-name> \
  --registry <registry-name> \
  --orderby time_asc \
  --query "[?lastUpdateTime < '2024-01-01'].[digest, lastUpdateTime]" \
  --output tsv
```

---

## Image Locking Commands

### Show Attributes

```bash
# Show repository attributes
az acr repository show \
  --name <registry-name> \
  --repository <repository-name> \
  --output jsonc

# Show image attributes
az acr repository show \
  --name <registry-name> \
  --image <repository>:<tag> \
  --output jsonc

# Show manifest metadata
az acr manifest show-metadata \
  --registry <registry-name> \
  --name <repository>:<tag>
```

### Lock Images/Repositories

```bash
# Lock image (prevent write and delete)
az acr repository update \
  --name <registry-name> \
  --image <repository>:<tag> \
  --write-enabled false

# Lock image by digest
az acr repository update \
  --name <registry-name> \
  --image <repository>@<sha256:digest> \
  --write-enabled false

# Lock entire repository
az acr repository update \
  --name <registry-name> \
  --repository <repository-name> \
  --write-enabled false

# Protect from deletion only (allow updates)
az acr repository update \
  --name <registry-name> \
  --image <repository>:<tag> \
  --delete-enabled false \
  --write-enabled true

# Prevent read operations
az acr repository update \
  --name <registry-name> \
  --image <repository>:<tag> \
  --read-enabled false
```

### Unlock Images/Repositories

```bash
# Unlock image
az acr repository update \
  --name <registry-name> \
  --image <repository>:<tag> \
  --delete-enabled true \
  --write-enabled true

# Unlock repository
az acr repository update \
  --name <registry-name> \
  --repository <repository-name> \
  --delete-enabled true \
  --write-enabled true

# Unlock manifest (if locked separately)
az acr repository update \
  --name <registry-name> \
  --image <repository>@<sha256:digest> \
  --delete-enabled true \
  --write-enabled true
```

---

## Utility Commands

### Storage Usage

```bash
# Show registry storage usage
az acr show-usage \
  --resource-group <resource-group> \
  --name <registry-name> \
  --output table
```

### Repository Management

```bash
# List all repositories
az acr repository list \
  --name <registry-name>

# List tags in a repository
az acr repository show-tags \
  --name <registry-name> \
  --repository <repository-name>

# List manifests with metadata
az acr manifest list-metadata \
  --name <repository-name> \
  --registry <registry-name> \
  --output table
```

### ACR Purge Help

```bash
# View acr purge help
az acr run \
  --registry <registry-name> \
  --cmd 'acr purge --help' \
  /dev/null
```

---

## Common Patterns and Examples

### Pattern 1: Enable Soft Delete for Recovery

```bash
# Enable soft delete with 14-day recovery window
az acr config soft-delete update -r myregistry --days 14 --status enabled

# Verify configuration
az acr config soft-delete show -r myregistry
```

### Pattern 2: Enable Retention for Storage Optimization

```bash
# Enable retention policy (Premium only)
az acr config retention update --registry myregistry --status enabled --days 7 --type UntaggedManifests

# Verify configuration
az acr config retention show --registry myregistry
```

### Pattern 3: Set Up Daily Purge Task

```bash
# Create daily purge task at midnight UTC
az acr task create --name dailyPurge \
  --cmd "acr purge --filter '.*:.*' --ago 30d --untagged" \
  --schedule "0 0 * * *" \
  --registry myregistry \
  --context /dev/null

# Verify task
az acr task show --name dailyPurge --registry myregistry
```

### Pattern 4: Protect Production Images

```bash
# Lock production images
az acr repository update --name myregistry --image myapp:production --write-enabled false
az acr repository update --name myregistry --image myapp:v1.0.0 --write-enabled false

# Verify lock status
az acr repository show --name myregistry --image myapp:production
```

### Pattern 5: Recover Accidentally Deleted Image

```bash
# Find deleted manifests
az acr manifest list-deleted -r myregistry -n myapp

# Find deleted tags
az acr manifest list-deleted-tags -r myregistry -n myapp:latest

# Restore the image
az acr manifest restore -r myregistry -n myapp:latest -d sha256:abc123

# Verify restoration
az acr repository show-tags -n myregistry --repository myapp
```

### Pattern 6: Clean Development Images Only

```bash
# Use filter pattern to target dev images
az acr run \
  --cmd "acr purge --filter 'dev/.*:.*' --ago 7d --untagged --dry-run" \
  --registry myregistry \
  /dev/null

# If output looks correct, run without dry-run
az acr run \
  --cmd "acr purge --filter 'dev/.*:.*' --ago 7d --untagged" \
  --registry myregistry \
  /dev/null
```

---

## Duration Reference for --ago Parameter

| Duration | Format |
|----------|--------|
| 1 day | `1d` |
| 1 week | `7d` |
| 2 weeks | `14d` |
| 1 month (approx) | `30d` |
| 1 hour | `1h` |
| 30 minutes | `30m` |
| Combined | `2d3h30m` |

---

## Cron Expression Reference

| Pattern | Description |
|---------|-------------|
| `"0 0 * * *"` | Daily at midnight UTC |
| `"0 0 * * 0"` | Weekly on Sunday at midnight |
| `"0 0 1 * *"` | Monthly on the 1st at midnight |
| `"0 */6 * * *"` | Every 6 hours |
| `"30 9 * * 1-5"` | Weekdays at 9:30 UTC |
| `"0 0 * * Mon,Wed,Fri"` | Mon/Wed/Fri at midnight |

**Format**: `{minute} {hour} {day} {month} {day-of-week}`

---

## Source References

- Soft Delete: `/submodules/azure-management-docs/articles/container-registry/container-registry-soft-delete-policy.md`
- Retention Policy: `/submodules/azure-management-docs/articles/container-registry/container-registry-retention-policy.md`
- Auto-Purge: `/submodules/azure-management-docs/articles/container-registry/container-registry-auto-purge.md`
- Delete Images: `/submodules/azure-management-docs/articles/container-registry/container-registry-delete.md`
- Image Locking: `/submodules/azure-management-docs/articles/container-registry/container-registry-image-lock.md`
- Scheduled Tasks: `/submodules/azure-management-docs/articles/container-registry/container-registry-tasks-scheduled.md`

# ACR Auto-Purge (acr purge) - Detailed Documentation

## Overview

**ACR Purge** (also known as **Auto-Purge**) is a mechanism for automatically deleting container images and manifests from Azure Container Registry based on age and tag filters. It runs as a containerized command within ACR Tasks, providing flexible, scheduled cleanup capabilities.

## Feature Status

- **Status**: Preview
- **Availability**: All SKUs (Basic, Standard, Premium)
- **Container Image**: `mcr.microsoft.com/acr/acr-cli:0.17`
- **Source Code**: [GitHub - Azure/acr-cli](https://github.com/Azure/acr-cli)

## How ACR Purge Works

### Execution Model

ACR Purge runs as an ACR Task, meaning:
1. It authenticates automatically with the registry where the task runs
2. It can be run on-demand or on a schedule (timer triggers)
3. It streams logs and supports all ACR Task features

### Container Command

The `acr purge` command is an alias for the full container image path:
```
mcr.microsoft.com/acr/acr-cli:0.17 purge
```

### Standard Execution Pattern

```bash
az acr run --registry <REGISTRY> --cmd 'acr purge <OPTIONS>' /dev/null
```

## Command Syntax and Parameters

### Required Parameters

| Parameter | Description | Example |
|-----------|-------------|---------|
| `--filter` | Repository and tag filter (regex) | `--filter "hello-world:.*"` |
| `--ago` | Duration (Go-style) for age threshold | `--ago 7d` |

### Optional Parameters

| Parameter | Description | Default |
|-----------|-------------|---------|
| `--untagged` | Also delete untagged manifests | false |
| `--dry-run` | Simulate without deleting | false |
| `--keep` | Keep N most recent tags | 0 (none) |
| `--concurrency` | Number of parallel purge tasks | Auto |

### Duration Format (--ago)

Uses Go-style duration strings:

| Unit | Symbol | Example |
|------|--------|---------|
| Days | `d` | `7d` = 7 days |
| Hours | `h` | `12h` = 12 hours |
| Minutes | `m` | `30m` = 30 minutes |

**Combinations**: `2d3h6m` = 2 days, 3 hours, 6 minutes

### Filter Pattern Examples

| Pattern | Matches |
|---------|---------|
| `"hello-world:.*"` | All tags in `hello-world` repository |
| `"hello-world:^1.*"` | Tags starting with `1` in `hello-world` |
| `".*/cache:.*"` | All tags in repositories ending with `/cache` |
| `"samples/.*:.*"` | All images in `samples/` namespace |

## Execution Methods

### Method 1: On-Demand Execution

```bash
# Environment variable for command
PURGE_CMD="acr purge --filter 'hello-world:.*' --ago 1d"

# Run immediately
az acr run \
  --cmd "$PURGE_CMD" \
  --registry myregistry \
  /dev/null
```

### Method 2: Scheduled Execution (ACR Task)

```bash
# Create a daily scheduled task
PURGE_CMD="acr purge --filter 'hello-world:.*' --ago 7d"

az acr task create --name purgeTask \
  --cmd "$PURGE_CMD" \
  --schedule "0 0 * * *" \
  --registry myregistry \
  --context /dev/null
```

### Cron Expression Reference

| Expression | Schedule |
|------------|----------|
| `"0 0 * * *"` | Daily at midnight UTC |
| `"0 1 * * Sun"` | Weekly on Sunday at 1:00 UTC |
| `"*/5 * * * *"` | Every 5 minutes |
| `"0 9-17 * * *"` | Hourly from 9:00-17:00 UTC |
| `"30 9 * * 1-5"` | Weekdays at 9:30 UTC |

**Format**: `{minute} {hour} {day} {month} {day-of-week}`

## Practical Examples

### Example 1: Basic Daily Purge

Delete all tags older than 7 days from `hello-world`:

```bash
PURGE_CMD="acr purge --filter 'hello-world:.*' --ago 7d"

az acr run \
  --cmd "$PURGE_CMD" \
  --registry myregistry \
  /dev/null
```

### Example 2: Purge with Untagged Manifests

Delete tags and untagged manifests older than 1 day:

```bash
PURGE_CMD="acr purge --filter 'hello-world:.*' --untagged --ago 1d"

az acr run \
  --cmd "$PURGE_CMD" \
  --registry myregistry \
  /dev/null
```

### Example 3: Multiple Repositories

Purge multiple development repositories:

```bash
PURGE_CMD="acr purge \
  --filter 'samples/devimage1:.*' \
  --filter 'samples/devimage2:.*' \
  --ago 0d --untagged"

az acr run \
  --cmd "$PURGE_CMD" \
  --registry myregistry \
  /dev/null
```

### Example 4: Keep Recent Tags

Delete old tags but keep the 5 most recent:

```bash
PURGE_CMD="acr purge --filter 'myapp:.*' --ago 30d --keep 5"

az acr run \
  --cmd "$PURGE_CMD" \
  --registry myregistry \
  /dev/null
```

### Example 5: Dry Run (Preview Changes)

See what would be deleted without actually deleting:

```bash
PURGE_CMD="acr purge \
  --filter 'samples/devimage1:.*' \
  --ago 0d --untagged --dry-run"

az acr run \
  --cmd "$PURGE_CMD" \
  --registry myregistry \
  /dev/null
```

**Sample dry-run output**:
```
Deleting tags for repository: samples/devimage1
myregistry.azurecr.io/samples/devimage1:232889b
myregistry.azurecr.io/samples/devimage1:a21776a
Deleting manifests for repository: samples/devimage1
myregistry.azurecr.io/samples/devimage1@sha256:81b6f9c92844bbbb5d0a101b22f7c2a7949e40f8ea90c8b3bc396879d95e788b

Number of deleted tags: 2
Number of deleted manifests: 1
```

### Example 6: Weekly Scheduled Purge

Create a weekly cleanup task:

```bash
PURGE_CMD="acr purge \
  --filter 'samples/devimage1:.*' \
  --filter 'samples/devimage2:.*' \
  --ago 0d --untagged"

az acr task create --name weeklyPurgeTask \
  --cmd "$PURGE_CMD" \
  --schedule "0 1 * * Sun" \
  --registry myregistry \
  --context /dev/null
```

## Handling Large Purge Operations

### Timeout Considerations

| Task Type | Default Timeout | Maximum |
|-----------|-----------------|---------|
| On-demand | 600 seconds (10 min) | Configurable |
| Scheduled | 3600 seconds (1 hr) | Configurable |

### Extending Timeout

For large-scale purges:

```bash
PURGE_CMD="acr purge --filter 'hello-world:.*' --ago 1d --untagged"

az acr run \
  --cmd "$PURGE_CMD" \
  --registry myregistry \
  --timeout 3600 \
  /dev/null
```

### Handling Partial Completion

If timeout occurs:
- Only a subset of tags/manifests are deleted
- Re-run the command to continue
- Consider using `--concurrency` for parallel processing

## Important Behaviors

### Image Locking Interaction

`acr purge` respects image locking attributes:
- Images with `write-enabled=false` are **NOT** deleted
- Repositories with `delete-enabled=false` are **NOT** deleted

```bash
# Lock important images before running purge
az acr repository update --name myregistry --image myapp:production --write-enabled false
```

### --untagged Filter Behavior

**Important**: The `--untagged` filter does NOT respond to the `--ago` filter.
- All untagged manifests matching the repository filter are candidates for deletion
- Age filtering only applies to tagged images

### What Gets Deleted by Default

By default, `acr purge`:
- Deletes **tag references** only
- Does NOT delete underlying manifests and layer data
- Use `--untagged` to also delete untagged manifests

## Comparison with Other Cleanup Methods

| Feature | ACR Purge | Retention Policy | Manual Delete |
|---------|-----------|------------------|---------------|
| **Target** | Tagged images by filter | Untagged manifests only | Specific items |
| **Scheduling** | Cron-based | Automatic | Manual |
| **Filtering** | Regex patterns, age | None (all untagged) | Manual selection |
| **SKU** | All | Premium only | All |
| **Recovery** | No | No | No (unless Soft Delete) |

## Best Practices

### 1. Always Use Dry Run First

```bash
# Preview what will be deleted
PURGE_CMD="acr purge --filter 'myapp:.*' --ago 30d --dry-run"
az acr run --cmd "$PURGE_CMD" --registry myregistry /dev/null

# If output looks correct, run without --dry-run
```

### 2. Protect Production Images

```bash
# Lock production images before enabling purge tasks
az acr repository update --name myregistry --image myapp:production --write-enabled false

# Now purge won't affect locked images
```

### 3. Use Unique Tags for Production

Instead of `:latest`, use unique tags:
- `myapp:v1.0.0-abc123`
- `myapp:2024-01-15-build42`

This makes it easier to write purge filters that don't affect production.

### 4. Combine with Retention Policy (Premium)

For comprehensive cleanup:
- Use **Retention Policy** for automatic untagged manifest cleanup
- Use **ACR Purge** for age-based cleanup of tagged images

### 5. Monitor Purge Tasks

```bash
# View task runs
az acr task list-runs --name purgeTask --registry myregistry --output table

# View specific run logs
az acr task logs --registry myregistry --run-id <run-id>
```

## Task Management Commands

### View Scheduled Tasks

```bash
az acr task show --name purgeTask --registry myregistry --output table
```

### List Timer Triggers

```bash
az acr task timer list --name purgeTask --registry myregistry
```

### Update Task Schedule

```bash
az acr task timer update \
  --name purgeTask \
  --registry myregistry \
  --timer-name t1 \
  --schedule "0 2 * * *"
```

### Delete Task

```bash
az acr task delete --name purgeTask --registry myregistry
```

## Troubleshooting

### Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| No images deleted | Filter doesn't match | Test filter with `--dry-run` |
| Timeout exceeded | Too many images | Increase `--timeout` value |
| Locked images not deleted | `write-enabled=false` | Expected behavior; unlock if needed |
| Task not running | Schedule syntax error | Verify cron expression |

### Debugging Filter Patterns

```bash
# Test your filter with dry-run
az acr run \
  --cmd "acr purge --filter 'your-filter:.*' --ago 0d --dry-run" \
  --registry myregistry \
  /dev/null
```

### Viewing Task Logs

```bash
# Recent runs
az acr task list-runs --registry myregistry --output table

# Specific run logs
az acr task logs --registry myregistry --run-id cf1a
```

## ACR CLI Additional Commands

The `acr purge` is part of the `acr-cli` tool. To see all options:

```bash
az acr run --registry myregistry --cmd 'acr purge --help' /dev/null
```

## Source References

- Primary documentation: `/submodules/azure-management-docs/articles/container-registry/container-registry-auto-purge.md`
- Scheduled tasks: `/submodules/azure-management-docs/articles/container-registry/container-registry-tasks-scheduled.md`
- ACR Tasks overview: `/submodules/azure-management-docs/articles/container-registry/container-registry-tasks-overview.md`
- GitHub acr-cli: `https://github.com/Azure/acr-cli`
- ACR repo reference: `/submodules/acr/README.md` (mentions Auto-purge)

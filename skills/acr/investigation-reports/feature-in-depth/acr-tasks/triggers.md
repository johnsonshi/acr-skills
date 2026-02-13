# ACR Tasks - Triggers

## Overview

ACR Tasks supports three types of automatic triggers that initiate task execution without manual intervention:
1. **Source Code Triggers** - Git commits and pull requests
2. **Base Image Triggers** - Updates to base images
3. **Timer Triggers** - Scheduled execution using cron expressions

## Source Code Triggers

### Git Commit Triggers

Automatically build when code is committed to a repository.

**Supported Platforms:**
- GitHub (public and private repos)
- Azure DevOps

**Trigger Types:**
| Trigger | Default | Description |
|---------|---------|-------------|
| Commit | Enabled | Triggers on git push |
| Pull Request | Disabled | Triggers on PR creation/update |

> **Note:** GitHub Enterprise repos are NOT currently supported.

### Creating a Git-Triggered Task

```bash
az acr task create \
    --registry myregistry \
    --name build-on-commit \
    --image myapp:{{.Run.ID}} \
    --context https://github.com/user/repo.git#main \
    --file Dockerfile \
    --git-access-token $GIT_PAT \
    --commit-trigger-enabled true \
    --pull-request-trigger-enabled false
```

### Personal Access Token Requirements

#### GitHub PAT Scopes:
| Repo Type | Required Scopes |
|-----------|-----------------|
| Public | `repo:status`, `public_repo` |
| Private | `repo` (Full control) |

#### Azure DevOps PAT Scopes:
| Repo Type | Required Scopes |
|-----------|-----------------|
| Public | `Code (Read)` |
| Private | `Code (Read)` |

### How It Works

1. ACR Tasks creates a webhook in your repository
2. On commit/PR, webhook notifies ACR
3. ACR queues and executes the task
4. Build logs are stored and available via CLI/Portal

### Configure Trigger Branches

```bash
# Trigger only on specific branch
az acr task create \
    --registry myregistry \
    --name my-task \
    --context https://github.com/user/repo.git#develop \
    ...
```

### Enable/Disable Commit Triggers

```bash
# Disable commit trigger
az acr task update \
    --registry myregistry \
    --name my-task \
    --commit-trigger-enabled false

# Enable pull request trigger
az acr task update \
    --registry myregistry \
    --name my-task \
    --pull-request-trigger-enabled true
```

## Base Image Triggers

### Overview

Automatically rebuild application images when their base image is updated. This enables automated OS and framework patching.

### How Base Image Detection Works

1. When building, ACR Tasks analyzes `FROM` statements
2. Dependencies are registered with the task
3. When base image is updated, dependent tasks are triggered

### Supported Base Image Locations

- Same Azure container registry
- Different Azure container registry (same or different region)
- Docker Hub public repositories
- Microsoft Container Registry (MCR)

### Update Notification Timing

| Location | Detection Time |
|----------|---------------|
| Azure Container Registry | Immediate |
| Docker Hub / MCR | 10-60 minutes (random interval) |

### Creating a Base Image Triggered Task

```bash
# Create task with base image trigger (enabled by default)
az acr task create \
    --registry myregistry \
    --name app-build \
    --image myapp:{{.Run.ID}} \
    --context https://github.com/user/repo.git#main \
    --file Dockerfile \
    --git-access-token $GIT_PAT
```

### Initial Run Requirement

**Important:** You must run the task at least once for ACR Tasks to detect and track base image dependencies:

```bash
az acr task run --registry myregistry --name app-build
```

### Enable/Disable Base Image Trigger

```bash
# Disable base image trigger
az acr task update \
    --registry myregistry \
    --name app-build \
    --base-image-trigger-enabled false

# Re-enable
az acr task update \
    --registry myregistry \
    --name app-build \
    --base-image-trigger-enabled true
```

### Base Image Best Practices

1. **Use Stable Tags**: Use tags like `node:16-alpine` instead of `node:latest`
2. **Import Base Images**: Copy public base images to your registry for immediate triggers
3. **Track Runtime Images**: Only runtime (application) base images are tracked, not buildtime (multi-stage) images

### Import Base Images for Faster Detection

```bash
# Import base image to your registry
az acr import \
    --name myregistry \
    --source docker.io/library/node:16-alpine \
    --image baseimages/node:16-alpine

# Use imported image in Dockerfile
# FROM myregistry.azurecr.io/baseimages/node:16-alpine
```

## Timer Triggers (Scheduled Tasks)

### Overview

Run tasks on a defined schedule using cron expressions. Useful for:
- Maintenance operations
- Regular testing
- Image cleanup
- Periodic rebuilds

### Cron Expression Format

```
{minute} {hour} {day} {month} {day-of-week}
```

**Time Zone:** UTC only

**Field Values:**
| Field | Values | Special Characters |
|-------|--------|-------------------|
| Minute | 0-59 | `*`, `-`, `,`, `/` |
| Hour | 0-23 | `*`, `-`, `,`, `/` |
| Day | 1-31 | `*`, `-`, `,`, `/` |
| Month | 1-12 or Jan-Dec | `*`, `-`, `,`, `/` |
| Day of Week | 0-6 or Sun-Sat | `*`, `-`, `,`, `/` |

### Cron Examples

| Expression | Description |
|------------|-------------|
| `*/5 * * * *` | Every 5 minutes |
| `0 * * * *` | Every hour (on the hour) |
| `0 */2 * * *` | Every 2 hours |
| `0 9-17 * * *` | Every hour 9am-5pm UTC |
| `30 9 * * *` | Daily at 9:30 UTC |
| `30 9 * * 1-5` | Weekdays at 9:30 UTC |
| `0 21 * * *` | Daily at 9pm UTC |
| `30 9 * Jan Mon` | Mondays in January at 9:30 UTC |

### Creating a Scheduled Task

```bash
# Task with timer trigger
az acr task create \
    --registry myregistry \
    --name daily-build \
    --image myapp:{{.Run.ID}} \
    --context https://github.com/user/repo.git#main \
    --file Dockerfile \
    --git-access-token $GIT_PAT \
    --schedule "0 21 * * *"
```

### Task Without Source Context

```bash
# Run a command on schedule without source code
az acr task create \
    --registry myregistry \
    --name cleanup-task \
    --cmd mcr.microsoft.com/hello-world \
    --schedule "0 21 * * *" \
    --context /dev/null
```

### Managing Timer Triggers

#### List Timers
```bash
az acr task timer list \
    --registry myregistry \
    --name my-task
```

#### Add Timer
```bash
az acr task timer add \
    --registry myregistry \
    --name my-task \
    --timer-name morning-run \
    --schedule "30 10 * * *"
```

#### Update Timer
```bash
az acr task timer update \
    --registry myregistry \
    --name my-task \
    --timer-name morning-run \
    --schedule "30 11 * * *"
```

#### Remove Timer
```bash
az acr task timer remove \
    --registry myregistry \
    --name my-task \
    --timer-name morning-run
```

### Multiple Timer Triggers

A single task can have multiple timer triggers:

```bash
# Create task
az acr task create \
    --registry myregistry \
    --name multi-schedule \
    --cmd myregistry.azurecr.io/myapp \
    --schedule "0 6 * * *" \
    --context /dev/null

# Add second timer
az acr task timer add \
    --registry myregistry \
    --name multi-schedule \
    --timer-name evening-run \
    --schedule "0 18 * * *"
```

## Combining Triggers

A single task can use multiple trigger types:

```bash
az acr task create \
    --registry myregistry \
    --name comprehensive-task \
    --image myapp:{{.Run.ID}} \
    --context https://github.com/user/repo.git#main \
    --file Dockerfile \
    --git-access-token $GIT_PAT \
    --commit-trigger-enabled true \
    --base-image-trigger-enabled true \
    --schedule "0 3 * * *"
```

This task will trigger on:
- Git commits to main branch
- Base image updates
- Daily at 3am UTC

## Viewing Triggered Runs

### List Task Runs
```bash
az acr task list-runs \
    --registry myregistry \
    --name my-task \
    --output table
```

**Output:**
```
RUN ID    TASK      PLATFORM    STATUS     TRIGGER       STARTED               DURATION
--------  --------  ----------  ---------  ------------  --------------------  ----------
ca15      my-task   linux       Succeeded  Timer         2023-11-20T21:00:23Z  00:00:06
ca14      my-task   linux       Succeeded  Image Update  2023-11-20T20:53:35Z  00:00:06
ca13      my-task   linux       Succeeded  Commit        2023-11-20T19:30:00Z  00:00:08
ca12      my-task   linux       Succeeded  Manual        2023-11-20T18:00:00Z  00:00:07
```

### View Run Logs
```bash
az acr task logs \
    --registry myregistry \
    --run-id ca15
```

## Troubleshooting Triggers

### Git Triggers Not Firing

1. **Check webhook** - Verify webhook exists in repository settings
2. **Check PAT** - Ensure token has correct permissions and hasn't expired
3. **Check branch** - Confirm pushing to the correct branch
4. **Check task status** - Ensure task is enabled

```bash
az acr task show --registry myregistry --name my-task
```

### Base Image Triggers Not Firing

1. **Run task once** - Initial run required for dependency detection
2. **Check tag** - Use stable tags, not `:latest`
3. **Check trigger status** - Verify base image trigger is enabled
4. **Check timing** - Public registry updates have 10-60 min delay

### Timer Triggers Not Firing

1. **Check cron syntax** - Validate expression format
2. **Check timezone** - All times are UTC
3. **Check task status** - Ensure task is enabled
4. **List timers** - Verify timer configuration

```bash
az acr task timer list --registry myregistry --name my-task
```

## Related Topics

- [Quick Tasks](./quick-tasks.md)
- [Multi-Step Tasks](./multi-step-tasks.md)
- [CLI Commands](./cli-commands.md)

# ACR Tasks - CLI Commands Reference

## Overview

This reference covers all Azure CLI commands for ACR Tasks operations.

## Quick Build Commands

### az acr build

Build and push a container image.

```bash
az acr build \
    --registry <registry-name> \
    --image <image:tag> \
    [--file <Dockerfile>] \
    [--platform <os/arch>] \
    [--build-arg <key=value>] \
    [--no-push] \
    [--timeout <seconds>] \
    [--agent-pool <pool-name>] \
    [--resource-group <group>] \
    <context>
```

**Key Parameters:**
| Parameter | Description |
|-----------|-------------|
| `--registry`, `-r` | Registry name |
| `--image`, `-t` | Image name:tag |
| `--file`, `-f` | Dockerfile path |
| `--platform` | Target platform (Linux/amd64, Windows/amd64, etc.) |
| `--build-arg` | Build-time variables |
| `--no-push` | Build only, don't push |
| `--timeout` | Build timeout in seconds (default: 3600) |
| `--agent-pool` | Dedicated agent pool |
| `--source-acr-auth-id` | Identity for ABAC registries |

**Examples:**
```bash
# Basic build
az acr build --registry myregistry --image myapp:v1 .

# Build from GitHub
az acr build --registry myregistry --image myapp:v1 \
    https://github.com/user/repo.git#main

# Build for ARM64
az acr build --registry myregistry --image myapp:v1 \
    --platform Linux/arm64 .

# Build without pushing
az acr build --registry myregistry --image myapp:v1 --no-push .
```

### az acr run

Run a task from a YAML file or execute a command.

```bash
az acr run \
    --registry <registry-name> \
    [--file <yaml-file>] \
    [--cmd <command>] \
    [--set <key=value>] \
    [--set-secret <key=value>] \
    [--timeout <seconds>] \
    [--agent-pool <pool-name>] \
    <context>
```

**Examples:**
```bash
# Run YAML task
az acr run --registry myregistry -f task.yaml \
    https://github.com/Azure-Samples/acr-tasks.git

# Run single command
az acr run --registry myregistry \
    --cmd '$Registry/myimage:tag' /dev/null

# Run with variables
az acr run --registry myregistry -f task.yaml \
    --set version=v2 \
    https://github.com/user/repo.git
```

## Task Management Commands

### az acr task create

Create a new ACR Task.

```bash
az acr task create \
    --registry <registry-name> \
    --name <task-name> \
    [--image <image:tag>] \
    [--file <yaml-or-dockerfile>] \
    [--context <source-location>] \
    [--git-access-token <token>] \
    [--commit-trigger-enabled <true|false>] \
    [--pull-request-trigger-enabled <true|false>] \
    [--base-image-trigger-enabled <true|false>] \
    [--schedule <cron-expression>] \
    [--assign-identity [system]|<resource-id>] \
    [--platform <os/arch>] \
    [--agent-pool <pool-name>] \
    [--timeout <seconds>] \
    [--arg <key=value>] \
    [--set <key=value>]
```

**Examples:**
```bash
# Create task with git trigger
az acr task create \
    --registry myregistry \
    --name build-task \
    --image myapp:{{.Run.ID}} \
    --context https://github.com/user/repo.git#main \
    --file Dockerfile \
    --git-access-token $GIT_PAT

# Create scheduled task
az acr task create \
    --registry myregistry \
    --name daily-task \
    --image myapp:{{.Run.ID}} \
    --context https://github.com/user/repo.git#main \
    --file Dockerfile \
    --git-access-token $GIT_PAT \
    --schedule "0 21 * * *"

# Create task with managed identity
az acr task create \
    --registry myregistry \
    --name identity-task \
    --image myapp:{{.Run.ID}} \
    --context https://github.com/user/repo.git#main \
    --file Dockerfile \
    --assign-identity

# Create multi-step task
az acr task create \
    --registry myregistry \
    --name multistep-task \
    --context https://github.com/user/repo.git#main \
    --file acr-task.yaml \
    --git-access-token $GIT_PAT
```

### az acr task update

Update an existing task.

```bash
az acr task update \
    --registry <registry-name> \
    --name <task-name> \
    [--image <image:tag>] \
    [--file <yaml-or-dockerfile>] \
    [--context <source-location>] \
    [--commit-trigger-enabled <true|false>] \
    [--base-image-trigger-enabled <true|false>] \
    [--status <Enabled|Disabled>]
```

**Examples:**
```bash
# Disable task
az acr task update --registry myregistry --name my-task --status Disabled

# Update context
az acr task update --registry myregistry --name my-task \
    --context https://github.com/user/repo.git#develop

# Disable base image trigger
az acr task update --registry myregistry --name my-task \
    --base-image-trigger-enabled false
```

### az acr task show

Display task details.

```bash
az acr task show \
    --registry <registry-name> \
    --name <task-name> \
    [--with-secure-properties]
```

### az acr task list

List all tasks in a registry.

```bash
az acr task list \
    --registry <registry-name> \
    [--output table]
```

### az acr task delete

Delete a task.

```bash
az acr task delete \
    --registry <registry-name> \
    --name <task-name> \
    [--yes]
```

## Task Execution Commands

### az acr task run

Manually trigger a task.

```bash
az acr task run \
    --registry <registry-name> \
    --name <task-name> \
    [--set <key=value>] \
    [--set-secret <key=value>] \
    [--no-logs]
```

**Examples:**
```bash
# Run task
az acr task run --registry myregistry --name my-task

# Run with variables
az acr task run --registry myregistry --name my-task \
    --set version=v2 \
    --set-secret apikey=secret123

# Run without streaming logs
az acr task run --registry myregistry --name my-task --no-logs
```

### az acr task list-runs

List task runs.

```bash
az acr task list-runs \
    --registry <registry-name> \
    [--name <task-name>] \
    [--run-status <Queued|Started|Running|Succeeded|Failed|Canceled>] \
    [--output table]
```

**Examples:**
```bash
# List all runs
az acr task list-runs --registry myregistry --output table

# List runs for specific task
az acr task list-runs --registry myregistry --name my-task --output table

# List failed runs
az acr task list-runs --registry myregistry --run-status Failed
```

### az acr task logs

View task run logs.

```bash
az acr task logs \
    --registry <registry-name> \
    [--name <task-name>] \
    [--run-id <run-id>] \
    [--image <image:tag>]
```

**Examples:**
```bash
# View logs for last run of a task
az acr task logs --registry myregistry --name my-task

# View logs for specific run
az acr task logs --registry myregistry --run-id ca1

# View logs for image build
az acr task logs --registry myregistry --image myapp:v1
```

### az acr task cancel-run

Cancel a running task.

```bash
az acr task cancel-run \
    --registry <registry-name> \
    --run-id <run-id>
```

## Timer Commands

### az acr task timer add

Add a timer trigger to a task.

```bash
az acr task timer add \
    --registry <registry-name> \
    --name <task-name> \
    --timer-name <timer-name> \
    --schedule <cron-expression> \
    [--enabled <true|false>]
```

### az acr task timer update

Update a timer trigger.

```bash
az acr task timer update \
    --registry <registry-name> \
    --name <task-name> \
    --timer-name <timer-name> \
    [--schedule <cron-expression>] \
    [--enabled <true|false>]
```

### az acr task timer list

List timer triggers for a task.

```bash
az acr task timer list \
    --registry <registry-name> \
    --name <task-name>
```

### az acr task timer remove

Remove a timer trigger.

```bash
az acr task timer remove \
    --registry <registry-name> \
    --name <task-name> \
    --timer-name <timer-name>
```

## Credential Commands

### az acr task credential add

Add credentials for external registry authentication.

```bash
az acr task credential add \
    --registry <registry-name> \
    --name <task-name> \
    --login-server <external-registry> \
    [--username <username>] \
    [--password <password>] \
    [--use-identity <[system]|client-id>]
```

**Examples:**
```bash
# Add with username/password
az acr task credential add \
    --registry myregistry \
    --name my-task \
    --login-server docker.io \
    --username myuser \
    --password mypassword

# Add with managed identity
az acr task credential add \
    --registry myregistry \
    --name my-task \
    --login-server otherregistry.azurecr.io \
    --use-identity [system]
```

### az acr task credential update

Update task credentials.

```bash
az acr task credential update \
    --registry <registry-name> \
    --name <task-name> \
    --login-server <external-registry> \
    [--username <username>] \
    [--password <password>]
```

### az acr task credential list

List task credentials.

```bash
az acr task credential list \
    --registry <registry-name> \
    --name <task-name>
```

### az acr task credential remove

Remove task credentials.

```bash
az acr task credential remove \
    --registry <registry-name> \
    --name <task-name> \
    --login-server <external-registry>
```

## Agent Pool Commands

### az acr agentpool create

Create a dedicated agent pool.

```bash
az acr agentpool create \
    --registry <registry-name> \
    --name <pool-name> \
    [--tier <S1|S2|S3|I6>] \
    [--count <instance-count>] \
    [--subnet-id <vnet-subnet-id>]
```

**Examples:**
```bash
# Create basic pool
az acr agentpool create \
    --registry myregistry \
    --name mypool \
    --tier S2

# Create pool in VNet
subnetId=$(az network vnet subnet show \
    --resource-group myRG \
    --vnet-name myVNet \
    --name mySubnet \
    --query id --output tsv)

az acr agentpool create \
    --registry myregistry \
    --name mypool \
    --tier S2 \
    --subnet-id $subnetId
```

### az acr agentpool update

Update an agent pool.

```bash
az acr agentpool update \
    --registry <registry-name> \
    --name <pool-name> \
    [--count <instance-count>]
```

### az acr agentpool show

Show agent pool details.

```bash
az acr agentpool show \
    --registry <registry-name> \
    --name <pool-name> \
    [--queue-count]
```

### az acr agentpool list

List agent pools.

```bash
az acr agentpool list \
    --registry <registry-name>
```

### az acr agentpool delete

Delete an agent pool.

```bash
az acr agentpool delete \
    --registry <registry-name> \
    --name <pool-name> \
    [--yes]
```

## Buildpack Commands

### az acr pack build

Build an image using Cloud Native Buildpacks (preview).

```bash
az acr pack build \
    --registry <registry-name> \
    --image <image:tag> \
    --builder <builder-image> \
    [--pull] \
    [--platform <os/arch>] \
    <context>
```

**Examples:**
```bash
# Build Node.js app
az acr pack build \
    --registry myregistry \
    --image node-app:1.0 \
    --pull \
    --builder cloudfoundry/cnb:cflinuxfs3 \
    https://github.com/Azure-Samples/nodejs-docs-hello-world.git

# Build Java app
az acr pack build \
    --registry myregistry \
    --image java-app:{{.Run.ID}} \
    --pull \
    --builder heroku/buildpacks:18 \
    https://github.com/buildpack/sample-java-app.git
```

## Common Workflows

### Complete CI/CD Setup

```bash
# 1. Create task with all triggers
az acr task create \
    --registry myregistry \
    --name ci-pipeline \
    --image myapp:{{.Run.ID}} \
    --context https://github.com/user/repo.git#main \
    --file Dockerfile \
    --git-access-token $GIT_PAT \
    --commit-trigger-enabled true \
    --base-image-trigger-enabled true \
    --schedule "0 3 * * *" \
    --assign-identity

# 2. Add additional timer
az acr task timer add \
    --registry myregistry \
    --name ci-pipeline \
    --timer-name nightly \
    --schedule "0 3 * * *"

# 3. Test run
az acr task run --registry myregistry --name ci-pipeline

# 4. View runs
az acr task list-runs --registry myregistry --name ci-pipeline --output table
```

### Cross-Registry Setup

```bash
# 1. Create identity
az identity create --resource-group myRG --name taskIdentity
resourceID=$(az identity show --resource-group myRG --name taskIdentity --query id -o tsv)
principalID=$(az identity show --resource-group myRG --name taskIdentity --query principalId -o tsv)

# 2. Grant access to source registry
baseRegID=$(az acr show --name baseregistry --query id -o tsv)
az role assignment create --assignee $principalID --scope $baseRegID --role AcrPull

# 3. Create task with identity
az acr task create \
    --registry myregistry \
    --name cross-reg \
    --context /dev/null \
    --file task.yaml \
    --assign-identity $resourceID

# 4. Add credentials
az acr task credential add \
    --registry myregistry \
    --name cross-reg \
    --login-server baseregistry.azurecr.io \
    --use-identity $clientID
```

## Related Topics

- [Quick Tasks](./quick-tasks.md)
- [Multi-Step Tasks](./multi-step-tasks.md)
- [Triggers](./triggers.md)
- [Authentication](./authentication.md)

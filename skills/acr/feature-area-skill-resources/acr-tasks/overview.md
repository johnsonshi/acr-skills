# ACR Tasks Skill

This skill provides comprehensive knowledge about Azure Container Registry Tasks for automated container building.

## When to Use This Skill

Use this skill when answering questions about:
- Cloud-based container image builds
- Automated build triggers
- Multi-step task workflows
- ACR Tasks YAML syntax
- Base image update triggers

## Overview

ACR Tasks provides cloud-based container image building, testing, and patching without requiring local Docker.

## Task Types

| Type | Command | Use Case |
|------|---------|----------|
| Quick Task | `az acr build` | On-demand single builds |
| Triggered Task | `az acr task create` | Automated builds on events |
| Multi-Step Task | `az acr task create -f task.yaml` | Complex workflows |

## Quick Task (On-Demand Build)

```bash
# Build from local context
az acr build --registry myregistry --image myapp:v1 .

# Build from GitHub
az acr build --registry myregistry --image myapp:v1 \
  https://github.com/myorg/myrepo.git

# Build specific platform
az acr build --registry myregistry --image myapp:v1 \
  --platform linux/arm64 .
```

## Triggered Tasks

### Source Code Trigger
```bash
az acr task create \
  --name mytask \
  --registry myregistry \
  --image myapp:{{.Run.ID}} \
  --context https://github.com/myorg/myrepo.git \
  --file Dockerfile \
  --git-access-token $PAT \
  --commit-trigger-enabled true
```

### Base Image Update Trigger
```bash
az acr task create \
  --name mytask \
  --registry myregistry \
  --image myapp:v1 \
  --context /dev/null \
  --file Dockerfile \
  --base-image-trigger-enabled true
```

### Timer Trigger
```bash
az acr task create \
  --name mytask \
  --registry myregistry \
  --image myapp:v1 \
  --context https://github.com/myorg/myrepo.git \
  --file Dockerfile \
  --schedule "0 21 * * *"  # Daily at 9PM UTC
```

## Multi-Step Task YAML

```yaml
version: v1.1.0
steps:
  # Build image
  - build: -t {{.Run.Registry}}/myapp:{{.Run.ID}} .

  # Run tests
  - cmd: {{.Run.Registry}}/myapp:{{.Run.ID}}
    args: ["npm", "test"]

  # Push if tests pass
  - push:
    - {{.Run.Registry}}/myapp:{{.Run.ID}}
    - {{.Run.Registry}}/myapp:latest
```

### Step Types
| Step | Purpose | Example |
|------|---------|---------|
| `build` | Build image | `-t myimage:v1 .` |
| `push` | Push to registry | `["myimage:v1"]` |
| `cmd` | Run container | Execute tests, scripts |

## Run Variables

| Variable | Alias | Description |
|----------|-------|-------------|
| `{{.Run.Registry}}` | `$Registry` | Registry login server |
| `{{.Run.ID}}` | `$ID` | Unique run ID |
| `{{.Run.Date}}` | `$Date` | UTC date |
| `{{.Run.Commit}}` | `$Commit` | Git commit SHA |
| `{{.Run.Branch}}` | `$Branch` | Git branch |

## Platform Support

| OS | Architectures |
|----|---------------|
| Linux | amd64, arm, arm64, 386 |
| Windows | amd64 |

```bash
# Specify platform
az acr build --platform linux/arm64/v8 ...
```

## Authentication Options

### Managed Identity
```bash
# Create task with system-assigned identity
az acr task create \
  --name mytask \
  --registry myregistry \
  --assign-identity \
  ...

# Create task with user-assigned identity
az acr task create \
  --name mytask \
  --registry myregistry \
  --assign-identity /subscriptions/.../userAssignedIdentities/myidentity \
  ...
```

### Cross-Registry Access
```bash
az acr task credential add \
  --name mytask \
  --registry myregistry \
  --login-server otherregistry.azurecr.io \
  --use-identity [system]
```

## Agent Pools (Premium)

Dedicated compute for tasks with VNet support:

| Tier | vCPUs | Use Case |
|------|-------|----------|
| S1 | 2 | Light builds |
| S2 | 4 | Standard builds |
| S3 | 8 | Heavy builds |
| I6 | 64 | Isolated workloads |

```bash
# Create agent pool
az acr agentpool create \
  --registry myregistry \
  --name mypool \
  --tier S2 \
  --count 2

# Use agent pool
az acr build --registry myregistry --agent-pool mypool ...
```

## Task Management

```bash
# List tasks
az acr task list --registry myregistry -o table

# Show task details
az acr task show --name mytask --registry myregistry

# Run task manually
az acr task run --name mytask --registry myregistry

# View logs
az acr task logs --registry myregistry --run-id abc123

# List runs
az acr task list-runs --registry myregistry -o table
```

## Common Patterns

### Build and Test
```yaml
version: v1.1.0
steps:
  - build: -t {{.Run.Registry}}/app:{{.Run.ID}} .
  - cmd: {{.Run.Registry}}/app:{{.Run.ID}} npm test
  - push: ["{{.Run.Registry}}/app:{{.Run.ID}}"]
```

### Parallel Builds
```yaml
version: v1.1.0
steps:
  - id: build-api
    build: -t {{.Run.Registry}}/api:v1 ./api
  - id: build-web
    build: -t {{.Run.Registry}}/web:v1 ./web
  - push:
    - {{.Run.Registry}}/api:v1
    - {{.Run.Registry}}/web:v1
```

## Reference Documentation

- **Investigation reports**: `agent-investigation-reports/acr-features/acr-tasks/`
- **MS Learn docs**: `azure-management-docs/articles/container-registry/container-registry-tasks-overview.md`

# ACR Tasks - Multi-Step Tasks

## Overview

Multi-step tasks extend ACR Tasks beyond single-image builds to provide complex, multi-container workflows. They enable building, testing, and deploying multiple container images with dependencies.

## What are Multi-Step Tasks?

Multi-step tasks are defined in YAML files and can:
- Build and push multiple container images
- Run containers as test environments
- Execute commands within containers
- Chain steps with dependencies
- Run steps in parallel or sequentially

## Task Step Types

### 1. build

Builds a container image using `docker build` syntax:

```yaml
steps:
  - build: -t $Registry/myapp:$ID -f Dockerfile .
```

**Key Parameters:**
- `-t` / `--image`: Image name and tag
- `-f` / `--file`: Dockerfile path
- `context`: Build context directory

### 2. push

Pushes images to a container registry:

```yaml
steps:
  - push: ["$Registry/myapp:$ID"]
```

**Supports:**
- Azure Container Registry
- Docker Hub
- Other private registries

### 3. cmd

Runs a container as a command:

```yaml
steps:
  - cmd: $Registry/myapp:$ID --arg1 value1
```

**Use Cases:**
- Running tests
- Database migrations
- Deployment commands
- Utility scripts

## Basic Examples

### Simple Build and Push

```yaml
version: v1.1.0
steps:
  - build: -t $Registry/hello-world:$ID .
  - push: ["$Registry/hello-world:$ID"]
```

### Build, Test, and Push

```yaml
version: v1.1.0
steps:
  - id: build
    build: -t $Registry/myapp:$ID .
  - id: test
    cmd: $Registry/myapp:$ID npm test
  - id: push
    push: ["$Registry/myapp:$ID"]
    when: ["test"]
```

### Parallel Builds

```yaml
version: v1.1.0
steps:
  - id: build-web
    build: -t $Registry/web:$ID ./web
    when: ["-"]
  - id: build-api
    build: -t $Registry/api:$ID ./api
    when: ["-"]
  - id: push
    push:
      - $Registry/web:$ID
      - $Registry/api:$ID
    when: ["build-web", "build-api"]
```

## Complex Workflow Example

```yaml
version: v1.1.0
steps:
  # Build application and test images in parallel
  - id: build-web
    build: -t $Registry/hello-world:$ID .
    when: ["-"]
  - id: build-tests
    build: -t $Registry/hello-world-tests ./funcTests
    when: ["-"]

  # Push after both builds complete
  - id: push
    push: ["$Registry/helloworld:$ID"]
    when: ["build-web", "build-tests"]

  # Run web application
  - id: hello-world-web
    cmd: $Registry/helloworld:$ID
    detach: true
    ports: ["8080:80"]

  # Run functional tests against running app
  - id: funcTests
    cmd: $Registry/helloworld-tests:$ID
    env: ["host=hello-world-web:80"]
    when: ["hello-world-web"]

  # Package Helm chart
  - cmd: $Registry/functions/helm package --app-version $ID -d ./helm ./helm/helloworld/
    when: ["funcTests"]

  # Deploy with Helm
  - cmd: $Registry/functions/helm upgrade helloworld ./helm/helloworld/ --reuse-values --set helloworld.image=$Registry/helloworld:$ID
```

## Step Dependencies

### The `when` Property

Controls step execution order:

| Value | Behavior |
|-------|----------|
| `when: ["-"]` | Run immediately (no dependencies) |
| `when: ["step-id"]` | Run after specified step completes |
| `when: ["id1", "id2"]` | Run after both steps complete |
| (not specified) | Run after previous step in file |

### Sequential vs Parallel Execution

**Sequential (default):**
```yaml
steps:
  - build: -t $Registry/app:1 .      # Runs first
  - cmd: $Registry/app:1 npm test    # Runs second
  - push: ["$Registry/app:1"]        # Runs third
```

**Parallel:**
```yaml
steps:
  - id: step1
    build: -t $Registry/app1:1 ./app1
    when: ["-"]                       # Runs immediately
  - id: step2
    build: -t $Registry/app2:1 ./app2
    when: ["-"]                       # Runs immediately (parallel)
  - push: ["$Registry/app1:1", "$Registry/app2:1"]
    when: ["step1", "step2"]          # Waits for both
```

## Step Properties

### Common Properties

| Property | Type | Description |
|----------|------|-------------|
| `id` | string | Unique step identifier |
| `when` | array | Step dependencies |
| `timeout` | int | Max seconds for step (default: 600) |
| `startDelay` | int | Seconds to wait before starting |
| `retries` | int | Retry count on failure |
| `retryDelay` | int | Seconds between retries |
| `ignoreErrors` | bool | Continue on step failure |
| `env` | array | Environment variables |
| `workingDirectory` | string | Working directory for step |

### Container-Specific Properties

| Property | Type | Description |
|----------|------|-------------|
| `detach` | bool | Run in background |
| `ports` | array | Port mappings |
| `expose` | array | Exposed ports |
| `entryPoint` | string | Override ENTRYPOINT |
| `keep` | bool | Keep container after execution |
| `network` | object | Network configuration |
| `privileged` | bool | Privileged mode |
| `user` | string | Username or UID |

## Running Multi-Step Tasks

### Manual Run with az acr run

```bash
az acr run \
    --registry myregistry \
    -f task.yaml \
    https://github.com/Azure-Samples/acr-tasks.git
```

### Create Triggered Task

```bash
az acr task create \
    --registry myregistry \
    --name mytask \
    --context https://github.com/user/repo.git#main \
    --file task.yaml \
    --git-access-token $PAT
```

### Trigger Task Manually

```bash
az acr task run \
    --registry myregistry \
    --name mytask
```

## Multi-Registry Tasks

Build and push to multiple registries:

```yaml
version: v1.1.0
steps:
  - build: -t $Registry/myapp:$ID .
  - build: -t {{.Values.secondRegistry}}/myapp:$Date .
  - push:
    - $Registry/myapp:$ID
    - {{.Values.secondRegistry}}/myapp:$Date
```

Create task with secondary registry:

```bash
az acr task create \
    --registry myregistry \
    --name multi-registry-task \
    --file task.yaml \
    --context https://github.com/user/repo.git \
    --git-access-token $PAT \
    --set secondRegistry=otherregistry.azurecr.io

# Add credentials for secondary registry
az acr task credential add \
    --name multi-registry-task \
    --registry myregistry \
    --login-server otherregistry.azurecr.io \
    --username $SP_ID \
    --password $SP_PASSWORD
```

## Testing Containers

### Run Tests Against Running Container

```yaml
version: v1.1.0
steps:
  # Build app
  - id: build
    build: -t $Registry/myapp:$ID .

  # Start app in background
  - id: app
    cmd: $Registry/myapp:$ID
    detach: true
    ports: ["8080:80"]

  # Build test image
  - id: build-tests
    build: -t $Registry/myapp-tests:$ID ./tests

  # Run tests (can access app via DNS name 'app')
  - id: run-tests
    cmd: $Registry/myapp-tests:$ID
    env: ["APP_URL=http://app:80"]

  # Stop app
  - cmd: docker stop app

  # Push only if tests pass
  - push: ["$Registry/myapp:$ID"]
    when: ["run-tests"]
```

### Container Networking

Steps in the same task share a Docker network. Use step `id` as DNS hostname:

```yaml
steps:
  - id: database
    cmd: postgres:13
    detach: true
    env: ["POSTGRES_PASSWORD=secret"]

  - id: app
    cmd: $Registry/myapp:$ID
    env: ["DATABASE_URL=postgres://postgres:secret@database:5432/app"]
    when: ["database"]
```

## Best Practices

1. **Use Step IDs**: Always provide meaningful IDs for reference
2. **Explicit Dependencies**: Use `when` to control execution order
3. **Parallel Where Possible**: Use `when: ["-"]` for independent builds
4. **Clean Up**: Stop detached containers explicitly
5. **Fail Fast**: Use appropriate timeouts and retries
6. **Parameterize**: Use variables for flexibility

## Common Patterns

### Build Matrix
```yaml
steps:
  - build: -t $Registry/app:$ID-amd64 --platform linux/amd64 .
    when: ["-"]
  - build: -t $Registry/app:$ID-arm64 --platform linux/arm64 .
    when: ["-"]
```

### Conditional Push
```yaml
steps:
  - id: test
    cmd: $Registry/app:$ID npm test
    ignoreErrors: false
  - push: ["$Registry/app:$ID"]
    when: ["test"]  # Only push if tests pass
```

### Integration Testing
```yaml
steps:
  - id: deps
    cmd: docker-compose up -d
    detach: true
  - id: tests
    cmd: $Registry/integration-tests:$ID
  - cmd: docker-compose down
```

## Related Topics

- [YAML Reference](./yaml-reference.md)
- [Triggers](./triggers.md)
- [Authentication](./authentication.md)

# ACR Tasks - YAML Reference

## Overview

ACR Tasks uses YAML files to define multi-step task workflows. This reference covers all available primitives, properties, and syntax.

## File Format

### Basic Structure

```yaml
version: v1.1.0
stepTimeout: 600
workingDirectory: /workspace
env:
  - KEY=value
secrets:
  - id: mysecret
    keyvault: https://myvault.vault.azure.net/secrets/MySecret
steps:
  - build: ...
  - push: ...
  - cmd: ...
```

### Supported File Extensions

ACR Tasks recognizes these extensions as task files:
- `.yaml`, `.yml`
- `.toml`, `.json`
- `.sh`, `.bash`, `.zsh`
- `.ps1`, `.ps`, `.cmd`, `.bat`
- `.ts`, `.js`, `.php`, `.py`, `.rb`, `.lua`

Files without these extensions are treated as Dockerfiles.

## Task Properties (Global)

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `version` | string | `v1.0.0` | YAML schema version |
| `stepTimeout` | int | 600 | Default timeout per step (seconds) |
| `workingDirectory` | string | `/workspace` (Linux), `c:\workspace` (Windows) | Default working directory |
| `env` | array | none | Global environment variables |
| `secrets` | array | none | Secret definitions |
| `networks` | array | none | Network definitions |
| `volumes` | array | none | Volume definitions |

## Step Types

### build

Builds a container image using docker build syntax.

**Syntax:**
```yaml
- build: -t <image:tag> -f <Dockerfile> <context>
```

**Full Form:**
```yaml
- build: -t $Registry/myapp:$ID -f Dockerfile .
  id: build-step
  when: ["-"]
  timeout: 1200
  env:
    - BUILD_ENV=production
  workingDirectory: ./src
```

**Build Parameters:**
| Parameter | Description |
|-----------|-------------|
| `-t` / `--image` | Image name and tag |
| `-f` / `--file` | Dockerfile path |
| `--build-arg` | Build-time variables |
| `--target` | Multi-stage build target |
| `--no-cache` | Disable layer caching |
| `--pull` | Always pull base image |

### push

Pushes images to a container registry.

**Inline Syntax:**
```yaml
- push: ["$Registry/myapp:$ID"]
```

**Nested Syntax:**
```yaml
- push:
  - $Registry/myapp:$ID
  - $Registry/myapp:latest
```

**With Properties:**
```yaml
- push: ["$Registry/myapp:$ID"]
  id: push-step
  when: ["build-step"]
  timeout: 300
```

### cmd

Runs a container as a command.

**Basic Syntax:**
```yaml
- cmd: <image:tag> [arguments]
```

**Examples:**
```yaml
# Simple command
- cmd: alpine echo "Hello World"

# With registry reference
- cmd: $Registry/myapp:$ID npm test

# With all properties
- cmd: $Registry/myapp:$ID
  id: run-tests
  entryPoint: npm
  args: ["test", "--coverage"]
  env:
    - NODE_ENV=test
  detach: false
  ports: ["8080:80"]
  expose: ["3000"]
  workingDirectory: /app
  timeout: 300
  when: ["build"]
```

## Step Properties

### Common Properties (All Step Types)

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `id` | string | `acb_step_%d` | Unique step identifier |
| `when` | array | previous step | Dependencies |
| `timeout` | int | 600 | Max execution time (seconds) |
| `startDelay` | int | 0 | Delay before starting (seconds) |
| `retries` | int | 0 | Retry count on failure |
| `retryDelay` | int | 0 | Delay between retries (seconds) |
| `ignoreErrors` | bool | false | Continue on failure |
| `env` | array | none | Environment variables |
| `workingDirectory` | string | inherited | Working directory |

### Container Properties (build, cmd)

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `detach` | bool | false | Run in background |
| `entryPoint` | string | image default | Override ENTRYPOINT |
| `ports` | array | none | Port mappings (host:container) |
| `expose` | array | none | Exposed ports |
| `keep` | bool | false | Keep container after exit |
| `network` | object | default | Network configuration |
| `isolation` | string | `default` | Container isolation level |
| `privileged` | bool | false | Privileged mode |
| `user` | string | none | User name or UID |
| `pull` | bool | false | Force image pull |
| `repeat` | int | 0 | Repeat execution count |
| `disableWorkingDirectoryOverride` | bool | false | Use container's workdir |
| `secret` | object | none | Secret mount |
| `volumeMount` | object | none | Volume mount |

## Run Variables

### Built-in Variables

| Variable | Alias | Description |
|----------|-------|-------------|
| `{{.Run.ID}}` | `$ID` | Unique run identifier |
| `{{.Run.Registry}}` | `$Registry` | Registry login server |
| `{{.Run.RegistryName}}` | `$RegistryName` | Registry name only |
| `{{.Run.Date}}` | `$Date` | Run start date (UTC) |
| `{{.Run.SharedVolume}}` | `$SharedVolume` | Shared volume ID |
| `{{.Run.OS}}` | `$OS` | Operating system |
| `{{.Run.Architecture}}` | `$Architecture` | CPU architecture |
| `{{.Run.Commit}}` | `$Commit` | Git commit SHA |
| `{{.Run.Branch}}` | `$Branch` | Git branch name |
| `{{.Run.TaskName}}` | - | Task name |

### Usage Examples

```yaml
steps:
  # Using aliases (v1.1.0+)
  - build: -t $Registry/myapp:$ID .
  - build: -t $Registry/myapp:$Date .

  # Using full variables
  - build: -t {{.Run.Registry}}/myapp:{{.Run.ID}} .
```

## Custom Variables

### Define Aliases

```yaml
version: v1.1.0
alias:
  values:
    repo: myrepo
    version: v2
steps:
  - build: -t $Registry/$repo:$version .
```

### External Alias Files

```yaml
version: v1.1.0
alias:
  src:
    - 'https://storage.blob.core.windows.net/aliases/common.yml?sastoken'
steps:
  - build: -t $Registry/$CommonRepo:$ID .
```

### Runtime Variables

Pass at runtime with `--set`:

```bash
az acr task run --name mytask --registry myregistry --set version=v2
```

```yaml
steps:
  - build: -t $Registry/myapp:{{.Values.version}} .
```

## Image Aliases

Pre-defined aliases for common images:

| Alias | Image |
|-------|-------|
| `acr` | `mcr.microsoft.com/acr/acr-cli:0.14` |
| `az` | `mcr.microsoft.com/acr/azure-cli:9fb281c` |
| `bash` | `mcr.microsoft.com/acr/bash:9fb281c` |
| `curl` | `mcr.microsoft.com/acr/curl:9fb281c` |
| `cssc` | `mcr.microsoft.com/acr/cssc:9fb281c` |

**Usage:**
```yaml
steps:
  - cmd: acr purge --registry $RegistryName --filter samples/hello-world:.* --ago 7d
  - cmd: az acr repository list --name $RegistryName
  - cmd: bash echo "Hello from bash"
```

## Secrets

### Key Vault Integration

```yaml
version: v1.1.0
secrets:
  - id: username
    keyvault: https://myvault.vault.azure.net/secrets/DbUsername
  - id: password
    keyvault: https://myvault.vault.azure.net/secrets/DbPassword
    clientID: <managed-identity-client-id>  # Optional
steps:
  - cmd: bash echo '{{.Secrets.password}}' | docker login --username '{{.Secrets.username}}' --password-stdin
```

### Secret Volumes

```yaml
version: v1.1.0
volumes:
  - name: secrets
    secret:
      mysecret.txt: <base64-encoded-value>
steps:
  - cmd: myapp
    volumeMounts:
      - name: secrets
        mountPath: /run/secrets
```

### Command-Line Secrets

```bash
az acr run -f task.yaml --set-secret mysecret=secretvalue .
```

## Networks

```yaml
version: v1.1.0
networks:
  - name: mynetwork
    driver: bridge
    ipv6: false
    skipCreation: false
    isDefault: false
steps:
  - cmd: myapp
    network:
      name: mynetwork
```

## Volumes

```yaml
version: v1.1.0
volumes:
  - name: myvolume
    secret:
      config.json: <base64-content>
steps:
  - build: -t $Registry/myapp:$ID .
    volumeMount:
      name: myvolume
      mountPath: /config
```

## When (Dependencies)

### Syntax

```yaml
# No dependencies - run immediately
- build: ...
  when: ["-"]

# Single dependency
- push: ...
  when: ["build"]

# Multiple dependencies (AND)
- push: ...
  when: ["build-web", "build-api"]

# Default - depends on previous step
- push: ...  # No when specified
```

### Parallel Execution

```yaml
steps:
  # These run in parallel
  - id: build-a
    build: -t $Registry/a:$ID ./a
    when: ["-"]
  - id: build-b
    build: -t $Registry/b:$ID ./b
    when: ["-"]
  - id: build-c
    build: -t $Registry/c:$ID ./c
    when: ["-"]

  # This waits for all three
  - push:
    - $Registry/a:$ID
    - $Registry/b:$ID
    - $Registry/c:$ID
    when: ["build-a", "build-b", "build-c"]
```

## Complete Examples

### CI/CD Pipeline

```yaml
version: v1.1.0
stepTimeout: 900
env:
  - DOTNET_CLI_TELEMETRY_OPTOUT=1
steps:
  # Build
  - id: build
    build: -t $Registry/myapp:$ID .
    when: ["-"]

  # Unit Tests
  - id: unit-tests
    cmd: $Registry/myapp:$ID dotnet test
    when: ["build"]

  # Integration Tests
  - id: start-deps
    cmd: docker-compose up -d
    detach: true
    when: ["build"]

  - id: integration-tests
    cmd: $Registry/myapp:$ID dotnet test --filter Category=Integration
    env:
      - DB_CONNECTION=Server=db;Database=test
    when: ["start-deps"]

  - id: stop-deps
    cmd: docker-compose down
    when: ["integration-tests"]
    ignoreErrors: true

  # Push on success
  - id: push
    push:
      - $Registry/myapp:$ID
      - $Registry/myapp:latest
    when: ["unit-tests", "integration-tests"]
```

### Multi-Architecture Build

```yaml
version: v1.1.0
steps:
  - id: build-amd64
    build: -t $Registry/myapp:$ID-amd64 --platform linux/amd64 .
    when: ["-"]

  - id: build-arm64
    build: -t $Registry/myapp:$ID-arm64 --platform linux/arm64 .
    when: ["-"]

  - push:
    - $Registry/myapp:$ID-amd64
    - $Registry/myapp:$ID-arm64
    when: ["build-amd64", "build-arm64"]
```

### Helm Deployment

```yaml
version: v1.1.0
secrets:
  - id: kubeconfig
    keyvault: https://myvault.vault.azure.net/secrets/KubeConfig
steps:
  - id: build
    build: -t $Registry/myapp:$ID .

  - id: push
    push: ["$Registry/myapp:$ID"]
    when: ["build"]

  - id: package
    cmd: helm package --app-version $ID ./charts/myapp
    when: ["push"]

  - id: deploy
    cmd: helm upgrade myapp ./charts/myapp-*.tgz --install --set image.tag=$ID
    env:
      - KUBECONFIG=/run/secrets/kubeconfig
    volumeMounts:
      - name: secrets
        mountPath: /run/secrets
    when: ["package"]
```

## BuildKit Support

Enable BuildKit for advanced features:

```yaml
version: v1.1.0
env:
  - DOCKER_BUILDKIT=1
steps:
  - build: -t $Registry/myapp:$ID .
    secret:
      id: npmrc
      src: /run/secrets/npmrc
```

## Related Topics

- [Multi-Step Tasks](./multi-step-tasks.md)
- [Quick Tasks](./quick-tasks.md)
- [Authentication](./authentication.md)

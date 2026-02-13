# ACR Tasks - Quick Tasks

## Overview

Quick Tasks provide on-demand container image builds in the cloud without requiring a local Docker installation. They represent the "inner-loop" development experience moved to the cloud.

## What are Quick Tasks?

Quick Tasks allow you to:
- Build and push container images to ACR on-demand
- Validate Dockerfiles before committing code
- Run builds remotely using `az acr build`
- Test container builds without local Docker Engine

## Key Benefits

1. **No Local Docker Required**: Build images entirely in Azure
2. **Fast Validation**: Test build definitions before git commit
3. **CI/CD Integration**: Easily integrate with pipelines via service principals
4. **Dependency Detection**: Automatically discovers base image dependencies

## Primary Commands

### az acr build

The main command for quick tasks:

```bash
az acr build \
    --registry <registry-name> \
    --image <image:tag> \
    --file <Dockerfile> \
    <context>
```

### Parameters

| Parameter | Description | Required |
|-----------|-------------|----------|
| `--registry` / `-r` | Registry name | Yes |
| `--image` / `-t` | Image name and tag | Yes (for push) |
| `--file` / `-f` | Dockerfile path | No (defaults to ./Dockerfile) |
| `--platform` | Target platform (Linux/amd64, etc.) | No |
| `--build-arg` | Build-time variables | No |
| `--no-push` | Don't push after building | No |
| `--agent-pool` | Dedicated agent pool name | No |

## Examples

### Basic Build and Push

```bash
az acr build \
    --registry myregistry \
    --image myapp:v1 \
    .
```

### Build from GitHub

```bash
az acr build \
    --registry myregistry \
    --image myapp:v1 \
    https://github.com/Azure-Samples/acr-build-helloworld-node.git
```

### Build Specific Branch

```bash
az acr build \
    --registry myregistry \
    --image myapp:v1 \
    https://github.com/user/repo.git#mybranch
```

### Build from Subfolder

```bash
az acr build \
    --registry myregistry \
    --image myapp:v1 \
    https://github.com/user/repo.git#main:src/app
```

### Build with Build Arguments

```bash
az acr build \
    --registry myregistry \
    --image myapp:v1 \
    --build-arg NODE_VERSION=16 \
    .
```

### Build for Different Platform

```bash
az acr build \
    --registry myregistry \
    --image myapp:v1 \
    --platform Linux/arm64 \
    .
```

### Build Without Pushing

```bash
az acr build \
    --registry myregistry \
    --image myapp:v1 \
    --no-push \
    .
```

### Build on Agent Pool

```bash
az acr build \
    --registry myregistry \
    --image myapp:v1 \
    --agent-pool myagentpool \
    .
```

## az acr run

Run commands or containers in the ACR Tasks environment:

```bash
az acr run \
    --registry myregistry \
    --cmd '$Registry/myimage:tag' \
    /dev/null
```

### Run with YAML File

```bash
az acr run \
    --registry myregistry \
    -f task.yaml \
    https://github.com/Azure-Samples/acr-tasks.git
```

## Output and Logs

### Build Output Structure

A successful build shows:
1. Context upload progress
2. Build agent assignment
3. Docker build steps
4. Push progress (if enabled)
5. Dependency information
6. Run ID and status

### Example Output

```
Packing source code into tar file to upload...
Sending build context (4.813 KiB) to ACR...
Queued a build with build ID: da1
Waiting for build agent...
Step 1/5 : FROM node:15-alpine
...
Successfully built 20a27b90eb29
Successfully tagged myregistry.azurecr.io/helloacrtasks:v1
...
The following dependencies were found:
- image:
    registry: myregistry.azurecr.io
    repository: helloacrtasks
    tag: v1
  runtime-dependency:
    registry: registry.hub.docker.com
    repository: library/node
    tag: 15-alpine
Run ID: da1 was successful after 1m9s
```

## Dependency Detection

ACR Tasks automatically detects:
- Base image dependencies (FROM statements)
- Runtime dependencies from public registries
- Git commit information

This enables:
- Automatic rebuilds on base image updates
- Tracking of image lineage
- Security scanning integration

## Context Sources

### Local Directory
```bash
az acr build --registry myregistry --image app:v1 ./src
```

### GitHub (Public)
```bash
az acr build --registry myregistry --image app:v1 \
    https://github.com/user/repo.git
```

### GitHub (Private)
```bash
az acr build --registry myregistry --image app:v1 \
    https://github.com/user/private-repo.git \
    --auth-mode None
```
(Requires configured credentials or PAT)

### Azure DevOps
```bash
az acr build --registry myregistry --image app:v1 \
    https://dev.azure.com/org/project/_git/repo
```

### Remote Tarball
```bash
az acr build --registry myregistry --image app:v1 \
    http://myserver.com/source.tar.gz
```

## ABAC-Enabled Registries

For registries with Attribute-Based Access Control enabled:

```bash
az acr build \
    --registry myregistry \
    --image app:v1 \
    --source-acr-auth-id [caller] \
    .
```

The `--source-acr-auth-id [caller]` flag passes the caller's identity for authentication.

## Best Practices

1. **Use Specific Tags**: Avoid `:latest`, use semantic versioning
2. **Minimize Context Size**: Use `.dockerignore` to exclude unnecessary files
3. **Cache Layers**: Order Dockerfile commands for optimal caching
4. **Test Locally First**: Validate Dockerfiles before cloud builds
5. **Use Build Arguments**: Parameterize for flexibility

## Troubleshooting

### Common Issues

1. **Context Too Large**
   - Add `.dockerignore` file
   - Exclude `node_modules`, `.git`, etc.

2. **Build Timeout**
   - Default timeout is 600 seconds
   - Use `--timeout` to extend

3. **Authentication Errors**
   - Ensure `az login` is current
   - Check registry permissions

4. **Platform Mismatch**
   - Specify `--platform` explicitly
   - Check base image platform support

## Related Topics

- [Multi-Step Tasks](./multi-step-tasks.md)
- [Triggers](./triggers.md)
- [CLI Commands](./cli-commands.md)

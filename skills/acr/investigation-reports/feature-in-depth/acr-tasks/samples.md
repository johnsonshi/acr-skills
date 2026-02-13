# ACR Tasks - Samples

## Overview

This document provides sample YAML task files and Dockerfiles for common ACR Tasks scenarios.

## Official Sample Repository

Microsoft maintains an official samples repository:
- **GitHub:** https://github.com/Azure-Samples/acr-tasks

## Basic Samples

### Build Image

Simple single-image build from Dockerfile.

**build-hello-world.yaml:**
```yaml
version: v1.1.0
steps:
  - build: -t $Registry/hello-world:$ID .
```

**Dockerfile:**
```dockerfile
FROM node:15-alpine
COPY . /src
RUN cd /src && npm install
EXPOSE 80
CMD ["node", "/src/server.js"]
```

**Run:**
```bash
az acr run -f build-hello-world.yaml https://github.com/Azure-Samples/acr-tasks.git
```

### Run Container

Execute a container command.

**bash-echo.yaml:**
```yaml
version: v1.1.0
steps:
  - cmd: bash echo "Hello World"
```

**Run:**
```bash
az acr run -f bash-echo.yaml https://github.com/Azure-Samples/acr-tasks.git
```

### Build and Push

Build and push to registry.

**build-push-hello-world.yaml:**
```yaml
version: v1.1.0
steps:
  - build: -t $Registry/hello-world:$ID .
  - push: ["$Registry/hello-world:$ID"]
```

**Run:**
```bash
az acr run -f build-push-hello-world.yaml https://github.com/Azure-Samples/acr-tasks.git
```

### Build and Run

Build image then run it.

**build-run-hello-world.yaml:**
```yaml
version: v1.1.0
steps:
  - build: -t $Registry/hello-world:$ID .
  - cmd: $Registry/hello-world:$ID
```

## Parallel Builds

### Build Multiple Images in Parallel

**build-push-hello-world-multi.yaml:**
```yaml
version: v1.1.0
steps:
  - id: build-frontend
    build: -t $Registry/frontend:$ID ./frontend
    when: ["-"]
  - id: build-backend
    build: -t $Registry/backend:$ID ./backend
    when: ["-"]
  - push:
    - $Registry/frontend:$ID
    - $Registry/backend:$ID
    when: ["build-frontend", "build-backend"]
```

### Parallel Build and Test

**when-parallel.yaml:**
```yaml
version: v1.1.0
steps:
  - id: build
    build: -t $Registry/hello-world:$ID .
    when: ["-"]
  - id: build-tests
    build: -t $Registry/hello-world-tests:$ID ./tests
    when: ["-"]
  - id: test
    cmd: $Registry/hello-world-tests:$ID
    when: ["build", "build-tests"]
  - push: ["$Registry/hello-world:$ID"]
    when: ["test"]
```

## Testing Patterns

### Integration Testing

**integration-test.yaml:**
```yaml
version: v1.1.0
steps:
  # Build the application
  - id: build-app
    build: -t $Registry/myapp:$ID .

  # Start dependencies
  - id: database
    cmd: postgres:13
    detach: true
    env:
      - POSTGRES_PASSWORD=testpass
      - POSTGRES_DB=testdb

  # Start the application
  - id: app
    cmd: $Registry/myapp:$ID
    detach: true
    env:
      - DATABASE_URL=postgres://postgres:testpass@database:5432/testdb
    ports: ["8080:80"]
    when: ["build-app", "database"]

  # Build and run tests
  - id: build-tests
    build: -t $Registry/myapp-tests:$ID ./tests
    when: ["build-app"]

  - id: run-tests
    cmd: $Registry/myapp-tests:$ID
    env:
      - API_URL=http://app:80
    when: ["app", "build-tests"]

  # Cleanup
  - cmd: docker stop app database
    ignoreErrors: true

  # Push only if tests pass
  - push: ["$Registry/myapp:$ID"]
    when: ["run-tests"]
```

### Unit Testing

**unit-test.yaml:**
```yaml
version: v1.1.0
steps:
  - id: build
    build: -t $Registry/myapp:$ID .

  - id: unit-tests
    cmd: $Registry/myapp:$ID npm test
    when: ["build"]

  - id: coverage
    cmd: $Registry/myapp:$ID npm run coverage
    when: ["unit-tests"]

  - push: ["$Registry/myapp:$ID"]
    when: ["coverage"]
```

## Multi-Registry Samples

### Push to Multiple Registries

**multipleRegistries/testtask.yaml:**
```yaml
version: v1.1.0
steps:
  - build: -t $Registry/myapp:$ID .
  - build: -t {{.Values.secondRegistry}}/myapp:$ID .
  - push:
    - $Registry/myapp:$ID
    - {{.Values.secondRegistry}}/myapp:$ID
```

**Run with second registry:**
```bash
az acr run -f testtask.yaml \
    --set secondRegistry=otherregistry.azurecr.io \
    https://github.com/Azure-Samples/acr-tasks.git
```

### Cross-Registry Base Image

**cross-registry.yaml:**
```yaml
version: v1.1.0
steps:
  - build: -t $Registry/myapp:$ID --build-arg BASE_REGISTRY={{.Values.baseRegistry}} .
  - push: ["$Registry/myapp:$ID"]
```

**Dockerfile:**
```dockerfile
ARG BASE_REGISTRY
FROM ${BASE_REGISTRY}/baseimages/node:16-alpine
COPY . /app
WORKDIR /app
RUN npm install
CMD ["npm", "start"]
```

## Helm Deployment

### Build, Package, and Deploy

**helm-deploy.yaml:**
```yaml
version: v1.1.0
secrets:
  - id: kubeconfig
    keyvault: https://myvault.vault.azure.net/secrets/KubeConfig
steps:
  # Build application
  - id: build
    build: -t $Registry/myapp:$ID .

  # Push image
  - id: push
    push: ["$Registry/myapp:$ID"]
    when: ["build"]

  # Package Helm chart
  - id: package
    cmd: helm package --app-version $ID -d ./dist ./charts/myapp
    when: ["push"]

  # Deploy to Kubernetes
  - id: deploy
    cmd: helm upgrade myapp ./dist/myapp-*.tgz --install --set image.tag=$ID
    env:
      - KUBECONFIG=/run/secrets/kubeconfig
    volumeMounts:
      - name: kubesecret
        mountPath: /run/secrets
    when: ["package"]

volumes:
  - name: kubesecret
    secret:
      kubeconfig: {{.Secrets.kubeconfig}}
```

## Cleanup and Maintenance

### Purge Old Images

**purge-task.yaml:**
```yaml
version: v1.1.0
steps:
  # List tags
  - cmd: acr tag list --registry $RegistryName --repository samples/hello-world

  # Purge old tags
  - cmd: acr purge --registry $RegistryName --filter samples/hello-world:.* --ago 30d --dry-run

  # Actually purge (remove --dry-run when ready)
  # - cmd: acr purge --registry $RegistryName --filter samples/hello-world:.* --ago 30d
```

**Create scheduled purge task:**
```bash
az acr task create \
    --registry myregistry \
    --name purge-old-images \
    --cmd 'acr purge --registry $RegistryName --filter samples/*:.* --ago 30d' \
    --schedule "0 3 * * *" \
    --context /dev/null
```

## Secrets and Security

### Docker Hub Login

**docker-hub-login.yaml:**
```yaml
version: v1.1.0
secrets:
  - id: dockeruser
    keyvault: https://myvault.vault.azure.net/secrets/DockerHubUsername
  - id: dockerpass
    keyvault: https://myvault.vault.azure.net/secrets/DockerHubPassword
steps:
  - cmd: bash echo '{{.Secrets.dockerpass}}' | docker login --username '{{.Secrets.dockeruser}}' --password-stdin
  - build: -t {{.Values.dockerRepo}}:$ID .
  - push: ["{{.Values.dockerRepo}}:$ID"]
```

### NPM Authentication

**npm-private.yaml:**
```yaml
version: v1.1.0
env:
  - DOCKER_BUILDKIT=1
secrets:
  - id: npmtoken
    keyvault: https://myvault.vault.azure.net/secrets/NpmToken
steps:
  - build: -t $Registry/myapp:$ID --secret id=npmrc,src=/run/secrets/npmrc .

volumes:
  - name: npmsecret
    secret:
      npmrc: |
        //registry.npmjs.org/:_authToken={{.Secrets.npmtoken}}
```

**Dockerfile:**
```dockerfile
# syntax=docker/dockerfile:1
FROM node:16-alpine
WORKDIR /app
COPY package*.json ./
RUN --mount=type=secret,id=npmrc,target=/root/.npmrc npm ci
COPY . .
RUN npm run build
CMD ["npm", "start"]
```

## Multi-Architecture Builds

### ARM and AMD64

**multi-arch.yaml:**
```yaml
version: v1.1.0
steps:
  - id: build-amd64
    build: -t $Registry/myapp:$ID-amd64 --platform linux/amd64 .
    when: ["-"]

  - id: build-arm64
    build: -t $Registry/myapp:$ID-arm64 --platform linux/arm64 .
    when: ["-"]

  - id: build-armv7
    build: -t $Registry/myapp:$ID-armv7 --platform linux/arm/v7 .
    when: ["-"]

  - push:
    - $Registry/myapp:$ID-amd64
    - $Registry/myapp:$ID-arm64
    - $Registry/myapp:$ID-armv7
    when: ["build-amd64", "build-arm64", "build-armv7"]
```

## Build Matrix

### Multiple Configurations

**build-matrix.yaml:**
```yaml
version: v1.1.0
steps:
  # Build for different Node versions
  - id: node16
    build: -t $Registry/myapp:$ID-node16 --build-arg NODE_VERSION=16 .
    when: ["-"]

  - id: node18
    build: -t $Registry/myapp:$ID-node18 --build-arg NODE_VERSION=18 .
    when: ["-"]

  - id: node20
    build: -t $Registry/myapp:$ID-node20 --build-arg NODE_VERSION=20 .
    when: ["-"]

  # Test each version
  - id: test-node16
    cmd: $Registry/myapp:$ID-node16 npm test
    when: ["node16"]

  - id: test-node18
    cmd: $Registry/myapp:$ID-node18 npm test
    when: ["node18"]

  - id: test-node20
    cmd: $Registry/myapp:$ID-node20 npm test
    when: ["node20"]

  # Push all versions
  - push:
    - $Registry/myapp:$ID-node16
    - $Registry/myapp:$ID-node18
    - $Registry/myapp:$ID-node20
    when: ["test-node16", "test-node18", "test-node20"]
```

**Dockerfile:**
```dockerfile
ARG NODE_VERSION=18
FROM node:${NODE_VERSION}-alpine
WORKDIR /app
COPY . .
RUN npm install
CMD ["npm", "start"]
```

## Sequential vs Parallel Examples

### Sequential (Default)

```yaml
version: v1.1.0
steps:
  - build: -t $Registry/step1:$ID ./step1  # Runs first
  - build: -t $Registry/step2:$ID ./step2  # Waits for step1
  - build: -t $Registry/step3:$ID ./step3  # Waits for step2
```

### Parallel with Dependencies

```yaml
version: v1.1.0
steps:
  # Parallel builds
  - id: a
    build: -t $Registry/a:$ID ./a
    when: ["-"]
  - id: b
    build: -t $Registry/b:$ID ./b
    when: ["-"]
  - id: c
    build: -t $Registry/c:$ID ./c
    when: ["-"]

  # Wait for a and b only
  - id: d
    build: -t $Registry/d:$ID ./d
    when: ["a", "b"]

  # Wait for all
  - push:
    - $Registry/a:$ID
    - $Registry/b:$ID
    - $Registry/c:$ID
    - $Registry/d:$ID
    when: ["a", "b", "c", "d"]
```

## Additional Resources

- **GitHub Samples:** https://github.com/Azure-Samples/acr-tasks
- **ACR cmd Repo:** https://github.com/AzureCR/cmd (containers as commands)
- **ARM Templates:** https://github.com/Azure/acr/tree/master/docs/tasks/run-as-deployment

## Related Topics

- [YAML Reference](./yaml-reference.md)
- [Multi-Step Tasks](./multi-step-tasks.md)
- [Authentication](./authentication.md)

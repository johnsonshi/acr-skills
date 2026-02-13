# ACR Connected Registry - Pull Images

## Overview

This document covers how to configure client access and pull images from a connected registry. Clients use standard tools like the Docker CLI to interact with the connected registry.

## Prerequisites

1. **Connected Registry Deployed**: A connected registry must be deployed and in **Online** state
2. **Repository Synchronized**: The repository must be configured to sync with the connected registry
3. **Client Token**: A token with pull permissions must be created and associated with the connected registry
4. **Network Access**: Client must have network access to the connected registry endpoint

## Client Token Configuration

### Step 1: Create a Scope Map

Create a scope map that defines repository permissions:

```azurecli
# Set variables
REGISTRY_NAME=<container-registry-name>

# Create scope map for read access
az acr scope-map create \
  --name hello-world-scopemap \
  --registry $REGISTRY_NAME \
  --repository hello-world content/read \
  --description "Scope map for the connected registry."
```

### Scope Map Permissions

| Permission | Description |
|------------|-------------|
| `content/read` | Pull images and artifacts |
| `content/write` | Push images and artifacts (ReadWrite mode only) |
| `metadata/read` | Read image metadata |
| `metadata/write` | Write image metadata |

### Step 2: Create a Client Token

Create a token and associate it with the scope map:

```azurecli
az acr token create \
  --name myconnectedregistry-client-token \
  --registry $REGISTRY_NAME \
  --scope-map hello-world-scopemap
```

This command outputs the token credentials including passwords.

> **Important**: Save the generated passwords immediately. They are one-time passwords and cannot be retrieved later.

### Step 3: Associate Token with Connected Registry

Update the connected registry to accept the client token:

```azurecli
az acr connected-registry update \
  --name $CONNECTED_REGISTRY_NAME \
  --registry $REGISTRY_NAME \
  --add-client-token myconnectedregistry-client-token
```

## Pulling Images

### Using Docker CLI

#### Step 1: Login to Connected Registry

```bash
docker login \
  --username myconnectedregistry-client-token \
  --password <token_password> \
  <IP_address_or_FQDN_of_connected_registry>:<port>
```

#### Step 2: Pull the Image

```bash
docker pull <IP_address_or_FQDN_of_connected_registry>:<port>/hello-world
```

### Example with Port

```bash
# Login
docker login --username mytoken --password mypassword 192.168.1.100:8080

# Pull image
docker pull 192.168.1.100:8080/hello-world:latest
```

## Kubernetes Deployments

### Step 1: Create Image Pull Secret

```bash
kubectl create secret docker-registry regcred \
  --docker-server=<connected-registry-IP>:<port> \
  --docker-username=<token-name> \
  --docker-password=<token-password> \
  --docker-email=<email>
```

### Step 2: Use Secret in Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-world-deployment
  labels:
    app: hello-world
spec:
  selector:
    matchLabels:
      app: hello-world
  replicas: 1
  template:
    metadata:
      labels:
        app: hello-world
    spec:
      imagePullSecrets:
        - name: regcred
      containers:
        - name: hello-world
          image: <connected-registry-IP>:<port>/hello-world:latest
```

### Step 3: Apply Deployment

```bash
kubectl apply -f deployment.yaml
```

## HTTP vs HTTPS Access

### HTTPS (Recommended)

For secure production deployments, use HTTPS with TLS certificates:

```bash
# Login over HTTPS (default port 443)
docker login --username mytoken --password mypassword myregistry.local:443
```

### HTTP (Insecure - Testing Only)

> **Warning**: HTTP is not secure and should only be used for testing.

For HTTP access, configure Docker daemon to allow insecure registries:

1. **Edit Docker daemon configuration** (`/etc/docker/daemon.json`):

```json
{
  "insecure-registries": ["<connected-registry-IP>:<port>"]
}
```

2. **Restart Docker daemon**:

```bash
systemctl restart docker
```

3. **Login and pull**:

```bash
docker login --username mytoken --password mypassword 192.168.1.100:8080
docker pull 192.168.1.100:8080/hello-world
```

## Configuring Container Runtime (containerd)

### For HTTPS with Custom CA

1. **Create certificate directory**:

```bash
mkdir -p /etc/containerd/certs.d/<service-IP>:443
```

2. **Copy CA certificate**:

```bash
cp ca.crt /etc/containerd/certs.d/<service-IP>:443/
```

3. **Update containerd config** (`/etc/containerd/config.toml`):

```toml
[plugins."io.containerd.grpc.v1.cri".registry]
  config_path = "/etc/containerd/certs.d"
  [plugins."io.containerd.grpc.v1.cri".registry.configs."<service-IP>:443".tls]
    ca_file = "/etc/containerd/certs.d/<service-IP>:443/ca.crt"
```

4. **Restart containerd**:

```bash
systemctl restart containerd
```

### For HTTP (Insecure)

```toml
[plugins."io.containerd.grpc.v1.cri".registry]
  [plugins."io.containerd.grpc.v1.cri".registry.mirrors]
    [plugins."io.containerd.grpc.v1.cri".registry.mirrors."<service-IP>:80"]
      endpoint = ["http://<service-IP>:80"]
  [plugins."io.containerd.grpc.v1.cri".registry.configs]
    [plugins."io.containerd.grpc.v1.cri".registry.configs."<service-IP>:80".tls]
      insecure_skip_verify = true
```

## Token Management

### Regenerate Token Password

```azurecli
az acr token credential generate \
  --name myconnectedregistry-client-token \
  --registry $REGISTRY_NAME \
  --expiration-in-days 30 \
  --password1
```

### Update Token Permissions

```azurecli
az acr scope-map update \
  --name hello-world-scopemap \
  --registry $REGISTRY_NAME \
  --add-repository new-repo content/read
```

### View Token Details

```azurecli
az acr token show \
  --name myconnectedregistry-client-token \
  --registry $REGISTRY_NAME
```

## Troubleshooting Pull Issues

### Error: Unauthorized

```
Error response from daemon: pull access denied for <repo>, repository does not exist or may require 'docker login'
```

**Causes**:
- Not logged in to the connected registry
- Token doesn't have permission for the repository

**Solutions**:
1. Run `docker login` with correct credentials
2. Update scope map to include the repository
3. Wait for token permissions to sync

### Error: Manifest Unknown

```
Error response from daemon: manifest for <image>:<tag> not found: manifest unknown
```

**Causes**:
- Repository not configured for synchronization
- Image not yet synced from parent

**Solutions**:
1. Add repository to connected registry sync:
```azurecli
az acr connected-registry update \
  --registry $REGISTRY_NAME \
  --name $CONNECTED_REGISTRY_NAME \
  --add-repository <repository-name>
```
2. Wait for synchronization to complete

### Error: HTTP to HTTPS Client

```
Error response from daemon: Get https://<IP>/v2/: http: server gave HTTP response to HTTPS client
```

**Cause**: Trying to access HTTP registry with HTTPS

**Solution**: Configure Docker for insecure registry access (see HTTP section)

### Error: Certificate Signed by Unknown Authority

```
x509: certificate signed by unknown authority
```

**Cause**: Custom CA certificate not trusted

**Solutions**:
1. Add CA certificate to system trust store
2. Configure Docker/containerd to trust the CA (see containerd configuration)

## Best Practices

### 1. Use Descriptive Token Names
Name tokens based on their purpose: `factory-floor-pull-token`, `dev-team-token`

### 2. Limit Token Scope
Grant only necessary permissions to each token

### 3. Rotate Credentials Regularly
Regenerate token passwords periodically

### 4. Use HTTPS in Production
Never use HTTP in production environments

### 5. Monitor Token Usage
Track which tokens are accessing the registry

## Connected Registry Endpoint

The connected registry endpoint is:
- The IP address or FQDN of the server/device hosting the connected registry
- The port configured for the service (default: 8080 for HTTP, 443 for HTTPS)

Find the endpoint:
```bash
# For Kubernetes deployments
kubectl get svc -n connected-registry

# Shows the ClusterIP
```

## Sources

- `/Users/johnsonshi/repos/acr-skills/submodules/azure-management-docs/articles/container-registry/pull-images-from-connected-registry.md`
- `/Users/johnsonshi/repos/acr-skills/submodules/acr/docs/preview/connected-registry/quickstart-pull-images-from-connected-registry.md`
- `/Users/johnsonshi/repos/acr-skills/submodules/acr/docs/preview/connected-registry/quickstart-deploy-connected-registry-kubernetes.md`
- `/Users/johnsonshi/repos/acr-skills/submodules/acr/docs/preview/connected-registry/troubleshooting.md`

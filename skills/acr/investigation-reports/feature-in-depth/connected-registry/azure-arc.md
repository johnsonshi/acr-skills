# ACR Connected Registry - Azure Arc Integration

## Overview

Azure Arc-enabled Kubernetes is the **recommended deployment platform** for Connected Registry. The Connected Registry Arc extension simplifies deployment and management of connected registries on Arc-enabled Kubernetes clusters.

## Prerequisites

### Azure Resources

1. **Azure Container Registry (ACR)** - Premium SKU required
2. **Connected Registry Resource** - Created in the cloud registry
3. **Azure Arc-enabled Kubernetes Cluster** - Connected to Azure

### Tools and Extensions

- **Azure CLI** (latest version)
- **connectedk8s** extension
- **k8s-extension** extension

```azurecli
# Install required Azure CLI extensions
az extension add --name k8s-extension
az extension add --name connectedk8s
```

## Deployment Process

### Step 1: Create Connected Registry Resource

First, create the connected registry resource in Azure:

```azurecli
# Set environment variables
REGISTRY_NAME=<container-registry-name>
CONNECTED_REGISTRY_RW=<connected-registry-name>

# Create connected registry
az acr connected-registry create \
  --registry $REGISTRY_NAME \
  --name $CONNECTED_REGISTRY_RW \
  --repository "hello-world" \
  --mode ReadWrite
```

### Step 2: Enable Dedicated Data Endpoint

Required for connected registry communication:

```azurecli
az acr update --name $REGISTRY_NAME --data-endpoint-enabled
```

### Step 3: Generate Connection String

Create the protected settings file with the connection string:

#### Bash

```bash
cat << EOF > protected-settings-extension.json
{
  "connectionString": "$(az acr connected-registry get-settings \
  --name myconnectedregistry \
  --registry myacrregistry \
  --parent-protocol https \
  --generate-password 1 \
  --query ACR_REGISTRY_CONNECTION_STRING --output tsv --yes)"
}
EOF
```

#### PowerShell

```powershell
echo "{\"connectionString\":\"$(az acr connected-registry get-settings `
--name myconnectedregistry `
--registry myacrregistry `
--parent-protocol https `
--generate-password 1 `
--query ACR_REGISTRY_CONNECTION_STRING `
--output tsv `
--yes | tr -d '\r')\" }" > settings.json
```

### Step 4: Deploy the Arc Extension

Deploy the connected registry extension to the Arc-enabled cluster:

```azurecli
az k8s-extension create \
  --cluster-name myarck8scluster \
  --cluster-type connectedClusters \
  --extension-type Microsoft.ContainerRegistry.ConnectedRegistry \
  --name myconnectedregistry \
  --resource-group myresourcegroup \
  --config service.clusterIP=192.100.100.1 \
  --config-protected-file protected-settings-extension.json \
  --auto-upgrade-minor-version true
```

#### Key Parameters

| Parameter | Description |
|-----------|-------------|
| `--cluster-name` | Name of the Arc-enabled Kubernetes cluster |
| `--cluster-type` | Must be `connectedClusters` |
| `--extension-type` | `Microsoft.ContainerRegistry.ConnectedRegistry` |
| `--name` | Name matching the connected registry resource |
| `--config service.clusterIP` | IP from cluster's service IP range |
| `--config-protected-file` | JSON file with connection string |
| `--auto-upgrade-minor-version` | Enable automatic minor version upgrades |

### Step 5: Verify Deployment

Check extension status:

```azurecli
az k8s-extension show \
  --name myconnectedregistry \
  --cluster-name myarck8scluster \
  --resource-group myresourcegroup \
  --cluster-type connectedClusters
```

Check connected registry status:

```azurecli
az acr connected-registry list --registry myacrregistry --output table
```

Expected output:

```
NAME                  MODE       CONNECTION STATE    PARENT         LOGIN SERVER                 LAST SYNC(UTC)
--------------------  ---------  ------------------  -------------  ---------------------------  -----------------
myconnectedregistry   ReadWrite  online              myacrregistry  myacrregistry.azurecr.io     2026-01-09 12:00:00
```

## Security Configuration Options

### Default Configuration (Secure by Default)

The default deployment includes:
- **HTTPS enabled** with TLS encryption
- **ReadOnly mode** (can be changed to ReadWrite)
- **Trust distribution** to cluster nodes
- **Cert-manager service** for certificate management

### Using Pre-installed Cert-Manager

If you already have cert-manager on your cluster:

```azurecli
az k8s-extension create \
  --cluster-name myarck8scluster \
  --cluster-type connectedClusters \
  --extension-type Microsoft.ContainerRegistry.ConnectedRegistry \
  --name myconnectedregistry \
  --resource-group myresourcegroup \
  --config service.clusterIP=192.100.100.1 \
  --config cert-manager.install=false \
  --config-protected-file protected-settings-extension.json
```

### Bring Your Own Certificate (BYOC)

Use your own certificates for TLS:

1. **Create self-signed certificate** (or use CA-signed):

```bash
mkdir /certs
openssl req -newkey rsa:4096 -nodes -sha256 \
  -keyout /certs/mycert.key -x509 -days 365 \
  -out /certs/mycert.crt \
  -addext "subjectAltName = IP:<service IP>"
```

2. **Base64 encode certificates**:

```bash
export TLS_CRT=$(cat mycert.crt | base64 -w0)
export TLS_KEY=$(cat mycert.key | base64 -w0)
```

3. **Create protected settings file**:

```json
{
  "connectionString": "[connection string here]",
  "tls.crt": "[base64-encoded public cert]",
  "tls.key": "[base64-encoded private key]",
  "tls.cacrt": "[base64-encoded CA cert]"
}
```

4. **Deploy with BYOC**:

```azurecli
az k8s-extension create \
  --cluster-name myarck8scluster \
  --cluster-type connectedClusters \
  --extension-type Microsoft.ContainerRegistry.ConnectedRegistry \
  --name myconnectedregistry \
  --resource-group myresourcegroup \
  --config service.clusterIP=192.100.100.1 \
  --config cert-manager.enabled=false \
  --config cert-manager.install=false \
  --config-protected-file protected-settings-extension.json
```

### Using Kubernetes Secret

Create a Kubernetes TLS secret and reference it:

1. **Create Kubernetes secret**:

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Secret
metadata:
  name: k8secret
  type: kubernetes.io/tls
data:
  ca.crt: $TLS_CRT
  tls.crt: $TLS_CRT
  tls.key: $TLS_KEY
EOF
```

2. **Create protected settings file**:

```json
{
  "connectionString": "[connection string here]",
  "tls.secret": "k8secret"
}
```

3. **Deploy with Kubernetes secret**:

```azurecli
az k8s-extension create \
  --cluster-name myarck8scluster \
  --cluster-type connectedClusters \
  --extension-type Microsoft.ContainerRegistry.ConnectedRegistry \
  --name myconnectedregistry \
  --resource-group myresourcegroup \
  --config service.clusterIP=192.100.100.1 \
  --config cert-manager.enabled=false \
  --config cert-manager.install=false \
  --config-protected-file protected-settings-extension.json
```

### Disable Default Trust Distribution

Use your own trust distribution:

```azurecli
az k8s-extension create \
  --cluster-name myarck8scluster \
  --cluster-type connectedClusters \
  --extension-type Microsoft.ContainerRegistry.ConnectedRegistry \
  --name myconnectedregistry \
  --resource-group myresourcegroup \
  --config service.clusterIP=192.100.100.1 \
  --config trustDistribution.enabled=false \
  --config cert-manager.enabled=false \
  --config cert-manager.install=false \
  --config-protected-file protected-settings-extension.json
```

## Deploying Pods from Connected Registry

### Step 1: Create Image Pull Secret

```bash
kubectl create secret docker-registry regcred \
  --docker-server=192.100.100.1 \
  --docker-username=mytoken \
  --docker-password=mypassword
```

### Step 2: Deploy Pod

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
          image: 192.100.100.1/hello-world:latest
```

Apply the deployment:

```bash
kubectl apply -f deployment.yaml
```

## Extension Management

### Update Extension

```azurecli
az k8s-extension update \
  --cluster-name myarck8scluster \
  --cluster-type connectedClusters \
  --name myconnectedregistry \
  --resource-group myresourcegroup \
  --config-protected-file protected-settings-extension.json
```

### Specific Version Deployment

```azurecli
az k8s-extension create \
  --cluster-name myarck8scluster \
  --cluster-type connectedClusters \
  --extension-type Microsoft.ContainerRegistry.ConnectedRegistry \
  --name myconnectedregistry \
  --resource-group myresourcegroup \
  --version <version-number> \
  --auto-upgrade-minor-version false \
  --config service.clusterIP=192.100.100.1 \
  --config-protected-file protected-settings-extension.json
```

## Clean Up

### Delete Extension

```azurecli
az k8s-extension delete \
  --name myconnectedregistry \
  --cluster-name myarcakscluster \
  --resource-group myresourcegroup \
  --cluster-type connectedClusters
```

### Delete Connected Registry Resource

```azurecli
az acr connected-registry delete \
  --registry myacrregistry \
  --name myconnectedregistry \
  --resource-group myresourcegroup
```

## Important Notes

1. **ClusterIP Selection**: The `service.clusterIP` must be within the cluster's service IP range and not already in use
2. **Connection String Security**: Store connection strings securely; passwords are one-time and cannot be retrieved
3. **Single Instance**: Only one instance of a connected registry can be active at a time
4. **Deactivation Required**: Before redeploying, deactivate the connected registry using `az acr connected-registry deactivate`

## Sources

- `/Users/johnsonshi/repos/acr-skills/submodules/azure-management-docs/articles/container-registry/quickstart-connected-registry-arc-cli.md`
- `/Users/johnsonshi/repos/acr-skills/submodules/azure-management-docs/articles/container-registry/tutorial-connected-registry-arc.md`
- `/Users/johnsonshi/repos/acr-skills/submodules/azure-management-docs/articles/container-registry/intro-connected-registry.md`

# ACR Artifact Streaming - AKS Integration

## Overview

This document covers the integration between Azure Container Registry (ACR) Artifact Streaming and Azure Kubernetes Service (AKS), including setup, configuration, and operational guidance.

## Prerequisites for AKS Integration

### AKS Cluster Requirements

| Requirement | Specification |
|-------------|---------------|
| Kubernetes Version | 1.26 or higher |
| Container Runtime | containerd (default for K8s 1.19+) |
| Node OS | Ubuntu 20.04+ or Azure Linux (not OS Guard) |
| Node Architecture | AMD64 only |

### ACR Requirements

| Requirement | Specification |
|-------------|---------------|
| Service Tier | Premium |
| Image Platform | Linux AMD64 |
| Streaming Artifacts | Pre-created for images |

### Incompatible Configurations

The following AKS configurations are NOT supported with artifact streaming:

- Azure Linux with OS Guard
- Pod Sandboxing
- Confidential Virtual Machines (CVMs)
- Gen 1 virtual machines
- ARM64 nodes

## Attaching ACR to AKS

### Option 1: Attach During Cluster Creation

```bash
az aks create \
  --resource-group <resource-group> \
  --name <cluster-name> \
  --attach-acr <acr-name>
```

### Option 2: Attach to Existing Cluster

```bash
az aks update \
  --resource-group <resource-group> \
  --name <cluster-name> \
  --attach-acr <acr-name>
```

### Verify ACR Attachment

```bash
az aks check-acr \
  --resource-group <resource-group> \
  --name <cluster-name> \
  --acr <acr-name>.azurecr.io
```

## Deploying with Artifact Streaming

### Basic Deployment

Once artifact streaming is configured in ACR and AKS is attached, deployments automatically use streaming when available:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
      - name: my-app
        image: <acr-name>.azurecr.io/<repo>:<tag>
        ports:
        - containerPort: 80
```

### Verifying Streaming is Active

Check pod events for streaming information:

```bash
kubectl describe pod <pod-name>
```

Look for events like:
```
Events:
  Type    Reason    Age    Message
  ----    ------    ----   -------
  Normal  Pulling   30s    Pulling image with artifact streaming
  Normal  Pulled    25s    Successfully pulled image (streamed)
```

### Digest-Based Deployment Note

When deploying with digest instead of tag:

```yaml
image: myregistry.azurecr.io/repo@sha256:4ef83ea6b0f7763c230e696709d8d8c398e21f65542db36e82961908bcf58d18
```

- Pod events won't show streaming-related information
- Streaming still occurs if container engine detects streamable content
- Fast container startup still achieved

## Pod Conditions and Events

### Streaming-Related Pod Conditions

| Condition | Meaning |
|-----------|---------|
| `UpgradeIfStreamableDisabled` | Image not from ACR or streaming issue detected |

### Interpreting Events

**Successful streaming:**
```
Normal  Pulled    Successfully pulled image in <fast-time>
```

**Standard pull (no streaming):**
```
Normal  Pulled    Successfully pulled image in <standard-time>
```

**Image already cached:**
```
Normal  Pulled    Container image already present on machine
```

---

## Legacy: Project Teleport AKS Integration (Deprecated)

> **Warning**: Project Teleport was deprecated in November 2023. The following is preserved for historical reference.

### Prerequisites (Legacy)

- Azure CLI 2.13.0+
- aks-preview extension 0.4.73+
- Project Teleport enabled on ACR (required signup)
- `EnableACRTeleport` feature flag enabled

### Registering AKS Feature Flag

```bash
az feature register \
  --namespace "Microsoft.ContainerService" \
  --name "EnableACRTeleport"
```

Check registration:
```bash
az feature list \
  --query "[?contains(name, 'Microsoft.ContainerService/EnableACRTeleport')].{Name:name,State:properties.state}" \
  -o table
```

Refresh provider:
```bash
az provider register --namespace Microsoft.ContainerService
```

### Creating Teleport-Enabled AKS Cluster

```bash
az aks create \
  --generate-ssh-keys \
  -g ${AKS_RG} \
  -n ${AKS} \
  --attach-acr $ACR \
  --kubernetes-version ${K8S_VERSION} \
  -l $LOCATION \
  --aks-custom-headers EnableACRTeleport=true
```

### Adding Teleport-Enabled Node Pool

```bash
az aks nodepool add \
  --name teleportpool \
  --cluster-name ${AKS} \
  --resource-group ${AKS_RG} \
  --kubernetes-version ${K8S_VERSION} \
  --aks-custom-headers EnableACRTeleport=true
```

### Node Labels

Teleport-enabled nodes have the label:
```
kubernetes.azure.com/enable-acr-teleport-plugin=true
```

Verify:
```bash
kubectl describe node <node-name> | grep teleport
```

### Sample Teleport Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: azure-vote-front-teleport
spec:
  replicas: 1
  selector:
    matchLabels:
      app: azure-vote-front-teleport
  template:
    metadata:
      labels:
        app: azure-vote-front-teleport
    spec:
      nodeSelector:
        "beta.kubernetes.io/os": linux
        "acr-teleport": enabled
      containers:
      - name: azure-vote-front-teleport
        image: <registryName>.azurecr.io/azure-vote-front:v1
        ports:
        - containerPort: 80
        env:
        - name: REDIS
          value: "azure-vote-back-teleport"
```

### Sample Standard (Non-Teleport) Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: azure-vote-front-shuttle
spec:
  replicas: 1
  selector:
    matchLabels:
      app: azure-vote-front-shuttle
  template:
    metadata:
      labels:
        app: azure-vote-front-shuttle
    spec:
      nodeSelector:
        "beta.kubernetes.io/os": linux
        "acr-teleport": disabled
      containers:
      - name: azure-vote-front-shuttle
        image: <registryName>.azurecr.io/azure-vote-front:v1
        ports:
        - containerPort: 80
```

### Performance Comparison (Legacy Data)

From Project Teleport testing with azure-vote-front (944MB, 28 layers):

| Method | Pull Time |
|--------|-----------|
| Standard Docker Pull | 34.7 seconds |
| Project Teleport | 8.0 seconds |

With squashed image (929MB, 1 layer):

| Method | Pull Time |
|--------|-----------|
| Standard Docker Pull | 31.7 seconds |
| Project Teleport | 1.9 seconds |

### Clearing Node Image Cache

To test fresh pulls, scale node pool to 0 and back:

```bash
# Scale down
az aks scale \
  --resource-group $AKS_RG \
  --name $AKS \
  --nodepool-name <nodepool> \
  --node-count 0

# Scale up
az aks scale \
  --resource-group $AKS_RG \
  --name $AKS \
  --nodepool-name <nodepool> \
  --node-count 1
```

## Best Practices

### 1. Verify Streaming Artifacts Before Deployment

Always confirm streaming artifacts exist before expecting streaming benefits:

```bash
az acr manifest list-referrers -n <repo>:<tag>
```

### 2. Use Same Region for Best Performance

Deploy AKS clusters in the same region as ACR for optimal performance:

```bash
# Check ACR location
az acr show --name <acr-name> --query location

# Create AKS in same region
az aks create --location <same-region> ...
```

### 3. Monitor Pod Startup Times

Track pod startup metrics to measure streaming effectiveness:

```bash
kubectl get events --sort-by='.lastTimestamp' | grep -E "Pulled|Pulling"
```

### 4. Enable Auto-Streaming for Frequently Updated Repositories

```bash
az acr artifact-streaming update \
  --repository <repo> \
  --enable-streaming true
```

### 5. Consider Layer Count

Streaming performance is affected by layer count:
- Fewer layers = faster mounting
- Consider image optimization for streaming-heavy workloads

## Troubleshooting AKS Integration

See [troubleshooting.md](./troubleshooting.md) for detailed troubleshooting guidance.

## Source Files

- AKS Integration (Legacy): `submodules/acr/docs/teleport/aks-getting-started.md`
- Performance Comparison: `submodules/acr/docs/teleport/aks-teleport-comparison.md`
- Sample Deployments: `submodules/acr/docs/teleport/samples/`
- Azure Linux OS Guard (shows incompatibility): `submodules/azure-management-docs/articles/azure-linux/intro-azure-linux-os-guard.md`
- Current Documentation: `submodules/azure-management-docs/articles/container-registry/container-registry-artifact-streaming.md`

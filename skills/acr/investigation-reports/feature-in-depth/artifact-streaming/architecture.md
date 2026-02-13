# ACR Artifact Streaming - Architecture

## Overview

This document describes the technical architecture of ACR Artifact Streaming, including both the current public preview implementation and the legacy Project Teleport architecture for historical context.

## High-Level Architecture

```
+---------------------+          +-------------------------+
|    Container        |          |  Azure Container        |
|    Registry (ACR)   |          |  Registry Storage       |
|                     |          |                         |
| +----------------+  |          | +--------------------+  |
| | Original Image |  |          | | Compressed Blobs   |  |
| | (Manifest)     |  |          | | (Azure Blob Store) |  |
| +----------------+  |          | +--------------------+  |
|         |           |          |                         |
|         v           |          | +--------------------+  |
| +----------------+  |          | | Streaming Artifacts|  |
| | Streaming      |  | <------> | | (Optimized format) |  |
| | Artifact       |  |          | +--------------------+  |
| | (Referrer)     |  |          |                         |
| +----------------+  |          +-------------------------+
+---------------------+
          |
          | OCI Distribution API
          v
+---------------------+          +-------------------------+
|   Azure Kubernetes  |          |   AKS Node Pool         |
|   Service (AKS)     |          |                         |
|                     |          | +--------------------+  |
| +----------------+  |          | | containerd         |  |
| | Artifact       |  | -------> | | (Container Runtime)|  |
| | Streaming      |  |          | +--------------------+  |
| | Plugin         |  |          |          |              |
| +----------------+  |          |          v              |
+---------------------+          | +--------------------+  |
                                 | | Container          |  |
                                 | | (Running Workload) |  |
                                 | +--------------------+  |
                                 +-------------------------+
```

## Current Architecture (Artifact Streaming Public Preview)

### Registry-Side Components

#### 1. Streaming Artifact Generator

When artifact streaming is enabled, ACR converts compatible images:

```
Original Image Push/Import
        |
        v
+----------------------+
| Compatibility Check  |
| - Linux AMD64?       |
| - Supported media    |
|   type?              |
| - Not already        |
|   streaming?         |
+----------------------+
        |
        v (if compatible)
+----------------------+
| Streaming Artifact   |
| Generation           |
| - Create optimized   |
|   artifact format    |
| - Link via OCI       |
|   referrers          |
+----------------------+
        |
        v
+----------------------+
| Storage              |
| - Original: Blob     |
| - Streaming: Opt.    |
|   format             |
+----------------------+
```

#### 2. OCI Referrers Integration

Streaming artifacts use OCI referrers to link to original images:

```json
{
  "schemaVersion": 2,
  "mediaType": "application/vnd.oci.image.manifest.v1+json",
  "artifactType": "application/vnd.azure.artifact.streaming.v1",
  "config": { ... },
  "layers": [ ... ],
  "subject": {
    "mediaType": "application/vnd.docker.distribution.manifest.v2+json",
    "digest": "sha256:original-image-digest",
    "size": 12345
  }
}
```

### AKS-Side Components

#### 1. Container Runtime Integration

AKS clusters use containerd with artifact streaming capabilities:

```
Pod Schedule Request
        |
        v
+----------------------+
| kubelet              |
| - Image pull request |
+----------------------+
        |
        v
+----------------------+
| containerd           |
| - Check for          |
|   streaming artifact |
| - Stream layers on   |
|   demand             |
+----------------------+
        |
        v
+----------------------+
| Overlay Filesystem   |
| - Mount streamed     |
|   layers             |
| - Lazy load content  |
+----------------------+
```

#### 2. Pod Events and Conditions

AKS surfaces streaming information through pod events:

```yaml
Events:
  Type    Reason     Message
  ----    ------     -------
  Normal  Pulling    Pulling image with artifact streaming
  Normal  Pulled     Successfully pulled image (streamed)
  Normal  Started    Started container
```

Special condition: `UpgradeIfStreamableDisabled` indicates streaming issues.

## Legacy Architecture (Project Teleport)

### Pre-Expanded Layer Storage

Project Teleport used a fundamentally different approach:

```
Original Image Push
        |
        v
+----------------------+
| Layer Expansion      |
| Service              |
| - Decompress layers  |
| - Create VHD files   |
+----------------------+
        |
        v
+----------------------+
| Azure Premium Files  |
| - Store expanded     |
|   VHD per layer      |
| - Enable SMB mount   |
+----------------------+
        |
        v
+----------------------+
| Mount API            |
| /mount/v1/{repo}/    |
|   _manifests/{digest}|
+----------------------+
```

### Teleportd Daemon Architecture

```
+-------------------------+
|     AKS Node            |
|                         |
| +--------------------+  |
| | teleportd daemon   |  |
| | - Journald logging |  |
| | - Layer mount mgmt |  |
| +--------------------+  |
|          |              |
|          v              |
| +--------------------+  |
| | containerd         |  |
| | - Snapshot plugin  |  |
| +--------------------+  |
|          |              |
|          v              |
| +--------------------+  |
| | SMB Mounts         |  |
| | - Pre-expanded     |  |
| |   layers from ACR  |  |
| +--------------------+  |
+-------------------------+
```

### Orca Client (Legacy)

The Orca client provided teleport protocol support before native integration:

```
+-------------------+     +-------------------+
|   orca client     |     |   docker client   |
| (teleport-enabled)|     |   (standard)      |
+-------------------+     +-------------------+
         |                         |
         v                         v
+-------------------+     +-------------------+
| ACR Teleport API  |     | ACR Standard API  |
| - Mount points    |     | - Blob URLs       |
| - SMB URLs        |     |                   |
+-------------------+     +-------------------+
```

## Layer Handling Architecture

### Traditional Image Pull Flow

```
1. GET /v2/{repo}/manifests/{tag}
   <- Manifest with layer digests

2. For each layer:
   GET /v2/{repo}/blobs/{digest}
   <- Compressed blob data

3. Decompress each layer locally

4. Overlay filesystem assembly

5. Container start
```

### Artifact Streaming Flow

```
1. GET /v2/{repo}/manifests/{tag}
   <- Manifest with layer digests

2. Check for streaming artifact:
   GET /v2/{repo}/referrers/{digest}
   <- Streaming artifact reference

3. Stream layers on-demand:
   - Only fetch accessed content
   - No full decompression needed

4. Container start (faster)
```

### Project Teleport Flow (Legacy)

```
1. GET /v2/{repo}/manifests/{tag}
   <- Manifest with layer digests

2. GET /mount/v1/{repo}/_manifests/{digest}
   <- SMB mount points for each layer

3. SMB mount each layer as pre-expanded VHD

4. Container start (no decompression)
```

## Storage Architecture

### Artifact Streaming Storage Model

```
Registry Storage
|
+-- repositories/
|   +-- {repo}/
|       +-- _manifests/
|       |   +-- {original-digest}  -> Original manifest
|       |   +-- {streaming-digest} -> Streaming manifest
|       +-- _layers/
|       |   +-- {layer-digest}     -> Compressed blob
|       +-- _referrers/
|           +-- {original-digest}  -> Links to streaming
```

### Project Teleport Storage Model (Legacy)

```
Azure Blob Storage (Original)
|
+-- repositories/{repo}/_layers/{digest} -> Compressed blob

Azure Premium Files (Expanded)
|
+-- {layer-digest}.vhd -> Pre-expanded layer as VHD
```

## Authentication Flow

### Artifact Streaming Authentication

```
AKS Managed Identity
        |
        v
+----------------------+
| ACR Auth             |
| - OAuth2 token       |
| - Scope: repository  |
|   pull               |
+----------------------+
        |
        v
+----------------------+
| Image Pull           |
| - Standard OCI pull  |
| - Streaming artifact |
|   resolution         |
+----------------------+
```

### Project Teleport Authentication (Legacy)

```
ACR Token (repository-scoped)
        |
        v
+----------------------+
| Standard API Auth    |
| - Bearer token       |
+----------------------+
        |
        v
+----------------------+
| Mount API Auth       |
| - Basic auth         |
| - /mount/v1/{repo}   |
+----------------------+
```

## Performance Characteristics

### Layer Count Impact

Project Teleport performance data demonstrated:

| Image Size | Layers | Docker Pull | Teleport |
|------------|--------|-------------|----------|
| 944 MB     | 28     | 34.7s       | 8.0s     |
| 929 MB     | 1      | 31.7s       | 1.9s     |

Key insight: Mount overhead per layer means fewer layers = better performance.

### Artifact Streaming Performance

Claimed benefits:
- 15%+ reduction in time-to-pod readiness
- Most significant for images >30GB
- Benefits scale with image size

## Network Architecture

### Same-Region Optimization

```
+------------+     Same Region      +------------+
|    ACR     | <------------------> |    AKS     |
| (Premium)  |   Low latency link   | (Cluster)  |
+------------+                      +------------+
```

### Cross-Region Support

```
+------------+                      +------------+
|    ACR     |                      |    AKS     |
| (Region A) | <--- Internet --->   | (Region B) |
+------------+                      +------------+
     |
     | Geo-replicated (optional)
     v
+------------+
|    ACR     |
| (Region B) |
+------------+
```

## Source Files

- Current Architecture: `submodules/azure-management-docs/articles/container-registry/container-registry-artifact-streaming.md`
- Legacy Teleport Architecture: `submodules/acr/docs/blog/teleport.md`
- AKS Integration: `submodules/acr/docs/teleport/aks-getting-started.md`
- Performance Analysis: `submodules/acr/docs/teleport/aks-teleport-comparison.md`

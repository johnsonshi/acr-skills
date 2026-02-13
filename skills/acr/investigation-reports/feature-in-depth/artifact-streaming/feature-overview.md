# ACR Artifact Streaming - Feature Overview

## Executive Summary

Azure Container Registry (ACR) Artifact Streaming is a preview feature that dramatically accelerates container workloads by reducing image pull times. This feature evolved from the earlier "Project Teleport" initiative and enables container images to be streamed to Azure Kubernetes Service (AKS) clusters, significantly reducing time-to-pod readiness.

## Feature Evolution

### Project Teleport (Legacy - Private Preview)

Project Teleport was the original codename for this technology, developed starting in early 2018. Key characteristics:

- **Status**: Deprecated and archived as of November 2023
- **Approach**: SMB-mounted pre-expanded layers from ACR to container hosts
- **Technology**: Used Azure Premium Files to store expanded layers as VHD files
- **Client**: Orca client for teleport protocol support

### Artifact Streaming (Current - Public Preview)

The successor to Project Teleport with improved architecture:

- **Status**: Public Preview (as of 2023-2024)
- **Approach**: Stream container images directly to AKS clusters
- **Technology**: Generates "streaming artifacts" that reference original images
- **Integration**: Native AKS and ACR integration

## Core Capabilities

### What Artifact Streaming Provides

1. **Reduced Image Pull Latency**: Can reduce time-to-pod readiness by over 15% (especially beneficial for images larger than 30GB)

2. **Single Registry Storage**: Store container images within a single registry and stream to AKS clusters in multiple regions

3. **Automatic Conversion**: When enabled on a repository, newly pushed compatible images are automatically converted to streaming artifacts

4. **Backward Compatibility**: Maintains access to both original and streaming artifacts even after disabling artifact streaming

5. **Cross-Region Support**: Works across regions regardless of geo-replication status

6. **Private Endpoint Support**: Compatible with private endpoints

## How It Works

### Image Conversion Process

1. **Push/Import Image**: A container image is pushed or imported to ACR
2. **Create Streaming Artifact**: ACR generates a streaming artifact from the image
3. **Storage**: Both original and streaming versions are stored in the same registry
4. **Referrers**: Streaming artifacts are linked to the original via OCI referrers

### Streaming to AKS

1. **Pod Deployment**: When a pod requests an image from ACR
2. **Detection**: AKS detects if a streaming artifact exists for the requested image
3. **Streaming**: Container engine streams the image content to the node
4. **Fast Startup**: Container starts faster as full image download is not required

## Pricing and Availability

### Service Tier Requirements

- **Required Tier**: Premium SKU only
- **Feature Cost**: No additional cost for the feature itself
- **Storage Consideration**: May increase overall registry storage consumption
- **Threshold Warning**: Additional charges may apply if consumption exceeds the included 500 GiB Premium SKU threshold

### Regional Availability

Artifact streaming works across all ACR regions where Premium tier is available. Unlike Project Teleport, there are no region restrictions for the public preview.

## Current Limitations (Preview)

### Platform Support

| Platform | Support Status |
|----------|---------------|
| Linux AMD64 | Supported |
| Linux ARM64 | Not supported |
| Windows | Not supported |
| Multi-arch (AMD64 only) | Partially supported |

### Technical Requirements

| Requirement | Specification |
|-------------|---------------|
| AKS Ubuntu Version | 20.04 or higher |
| Kubernetes Version | 1.26 or higher |
| CMK Registries | Not supported |
| Kubernetes regcred | Not supported |

### Soft Delete Interaction

When artifact streaming is used with soft delete policy enabled:
- Deleting an artifact removes both original and streaming versions
- Only the original version can be viewed or restored during retention period

## Comparison: Project Teleport vs Artifact Streaming

| Aspect | Project Teleport | Artifact Streaming |
|--------|------------------|-------------------|
| Status | Deprecated | Public Preview |
| Signup | Required (waitlist) | Self-service |
| Repository Limit | 10 repositories | No limit |
| Region Restrictions | Limited regions | All Premium regions |
| Geo-replication | Not supported | Supported |
| Private Links | Not supported | Supported |
| Platform Support | Linux only | Linux AMD64 only |
| Storage Technology | Azure Premium Files (VHD) | Streaming artifacts |
| AKS Integration | Manual feature flag | Native integration |

## Use Cases

### Ideal Scenarios

1. **Large Container Images**: Images over 30GB see the most benefit
2. **Multi-Region Deployments**: Stream from single registry to multiple AKS clusters
3. **High-Scale Workloads**: Reduce node scaling latency
4. **Development/Testing**: Faster iteration cycles with large images

### Not Recommended For

1. **Small Images**: Minimal benefit for images under a few hundred MB
2. **Windows Workloads**: Not currently supported
3. **ARM-based Nodes**: Not supported in preview
4. **CMK-encrypted Registries**: Not compatible

## Key Terminology

| Term | Definition |
|------|------------|
| **Streaming Artifact** | The converted version of an image optimized for streaming |
| **Active State** | Repository state where new pushes automatically generate streaming artifacts |
| **Inactive State** | Repository state where automatic conversion is disabled |
| **Referrers** | OCI mechanism linking streaming artifacts to original images |
| **Teleport** | Legacy name for the streaming technology |
| **Layer Expansion** | Legacy term for converting compressed layers to streamable format |

## Source Documentation

- MS Learn: `submodules/azure-management-docs/articles/container-registry/container-registry-artifact-streaming.md`
- MS Learn Troubleshooting: `submodules/azure-management-docs/articles/container-registry/troubleshoot-artifact-streaming.md`
- ACR Repo Preview Announcement: `submodules/acr/docs/preview/artifact-streaming/README.md`
- Legacy Teleport Documentation: `submodules/acr/docs/teleport/README.md`
- Teleport Blog Post: `submodules/acr/docs/blog/teleport.md`

# ACR Image Management - Feature Overview

## Executive Summary

Azure Container Registry (ACR) provides comprehensive image management capabilities for storing, managing, and distributing container images and OCI artifacts. This document provides an overview of the key image management features available in ACR.

## Core Concepts

### Registry Structure

```
Registry
├── Repository 1
│   ├── Image:tag1
│   ├── Image:tag2
│   └── Image@sha256:digest
├── Repository 2
│   └── ...
└── Namespace/Repository 3
    └── ...
```

### Key Terminology

| Term | Description |
|------|-------------|
| **Registry** | A service that stores and distributes container images and artifacts |
| **Repository** | A collection of images with the same name but different tags |
| **Tag** | A label that specifies an image version (e.g., `v1.0`, `latest`) |
| **Manifest** | JSON file that uniquely identifies an image, referencing its layers |
| **Manifest Digest** | A unique SHA-256 hash identifying a specific manifest |
| **Layer** | Individual components that make up a container image |

### Addressing Artifacts

- **By tag**: `myregistry.azurecr.io/repository:tag`
- **By digest**: `myregistry.azurecr.io/repository@sha256:abc123...`

## Image Management Features

### 1. Image Import
Import images from public registries (Docker Hub, MCR), other ACR registries, or private registries without Docker CLI commands.

**Key Benefits:**
- No local Docker installation required
- Multi-architecture image support
- Cross-subscription and cross-tenant support
- Network-restricted registry support via trusted services

### 2. Image Deletion
Multiple methods to delete image data and manage registry size:
- Delete entire repositories
- Delete by tag
- Delete by manifest digest
- Automatic purging via ACR Tasks

### 3. Image Locking (Immutability)
Prevent images from being deleted or overwritten:
- Lock individual tags
- Lock by manifest digest
- Lock entire repositories
- Protect from deletion while allowing updates

### 4. Tagging Strategies
Best practices for image versioning:
- **Stable tags** for base images (`:1.0`, `:latest`)
- **Unique tags** for deployments (`:build-123`, `:abc456`)
- Semantic versioning patterns

### 5. Multi-Architecture Images
Support for images targeting multiple platforms:
- Manifest lists combining multiple architectures
- Automatic platform selection during pull
- Build and push workflows for multi-arch

### 6. Retention and Cleanup Policies
Automated storage management:
- **Retention policy** for untagged manifests (Premium tier)
- **Soft delete policy** for recovery (Preview)
- **ACR Purge** for scheduled cleanup tasks

## Supported Content Formats

| Format | Description |
|--------|-------------|
| Docker Image Manifest V2 (Schema 1 & 2) | Standard Docker images |
| OCI Images | Open Container Initiative format |
| OCI Artifacts | Supply chain artifacts (SBOMs, signatures) |
| Helm Charts | Kubernetes application packages |
| Multi-arch Images | Manifest lists for multiple platforms |

## Service Tiers and Feature Availability

| Feature | Basic | Standard | Premium |
|---------|-------|----------|---------|
| Image Import | Yes | Yes | Yes |
| Image Delete | Yes | Yes | Yes |
| Image Lock | Yes | Yes | Yes |
| Retention Policy | No | No | Yes |
| Soft Delete | Yes | Yes | Yes |
| ACR Purge | Yes | Yes | Yes |
| Geo-replication | No | No | Yes |

## CLI Commands Overview

| Operation | Command |
|-----------|---------|
| Import image | `az acr import` |
| Delete repository | `az acr repository delete --repository` |
| Delete image | `az acr repository delete --image` |
| Lock image | `az acr repository update --write-enabled false` |
| List manifests | `az acr manifest list-metadata` |
| Show manifest | `az acr manifest show-metadata` |
| Untag image | `az acr repository untag` |
| Set retention | `az acr config retention update` |
| Enable soft delete | `az acr config soft-delete update` |

## Related Documentation

- [import.md](./import.md) - Detailed import operations
- [delete.md](./delete.md) - Deletion methods and best practices
- [locking.md](./locking.md) - Image immutability features
- [tagging.md](./tagging.md) - Tagging and versioning strategies
- [multi-arch.md](./multi-arch.md) - Multi-architecture image support
- [cli-commands.md](./cli-commands.md) - Complete CLI reference

## Source Documentation

### MS Learn Documentation
- `/submodules/azure-management-docs/articles/container-registry/container-registry-concepts.md`
- `/submodules/azure-management-docs/articles/container-registry/container-registry-best-practices.md`
- `/submodules/azure-management-docs/articles/container-registry/container-registry-image-formats.md`

### ACR Repository Documentation
- `/submodules/acr/docs/README.md`
- `/submodules/acr/docs/FAQ.md`

# Azure Container Registry - OCI Artifacts Feature Overview

## Executive Summary

Azure Container Registry (ACR) provides comprehensive support for Open Container Initiative (OCI) artifacts, enabling storage, management, and distribution of a wide range of cloud-native artifacts beyond traditional container images. ACR fully supports the OCI Distribution Specification, making it a vendor-neutral, cloud-agnostic solution for storing container images and other content types (artifacts).

## What Are OCI Artifacts?

OCI artifacts represent any content type that can be stored in an OCI-compliant registry. While container images are the most common artifact type, the OCI Distribution Specification enables registries to store a wide variety of content including:

- **Container Images**: Docker and OCI format images
- **Helm Charts**: Kubernetes application packages
- **Software Bill of Materials (SBOMs)**: Artifact dependency lists
- **Signatures**: Cryptographic signatures for artifact verification
- **Security Scan Results**: Vulnerability scan reports
- **Configuration Bundles**: Application configuration packages
- **AI/ML Models**: Machine learning model artifacts
- **Open Policy Agent (OPA) Bundles**: Policy definitions
- **Singularity Images**: High-performance computing container format

## ACR OCI Support Architecture

### Supported Content Formats

ACR supports multiple manifest and content formats:

| Format Type | Media Type | Description |
|-------------|-----------|-------------|
| Docker Image Manifest V2, Schema 1 | `application/vnd.docker.distribution.manifest.v1+json` | Legacy Docker format |
| Docker Image Manifest V2, Schema 2 | `application/vnd.docker.distribution.manifest.v2+json` | Modern Docker format with manifest lists |
| OCI Image Manifest | `application/vnd.oci.image.manifest.v1+json` | OCI-compliant image format |
| OCI Artifact Manifest | `application/vnd.oci.artifact.manifest.v1+json` | Generic artifact manifest |
| Helm Chart Config | `application/vnd.cncf.helm.config.v1+json` | Helm chart configuration |
| Helm Chart Content | `application/vnd.cncf.helm.chart.content.v1.tar+gzip` | Helm chart package |
| OPA Bundle Config | `application/vnd.cncf.openpolicyagent.config.v1+json` | OPA policy bundles |
| Singularity Image | `application/vnd.sylabs.sif.config.v1+json` | Singularity container format |

### OCI 1.1 Specification Support

ACR supports the OCI 1.1 Distribution Specification which introduces:

1. **Referrers API**: Enables discovering references to a subject artifact
2. **Artifact References**: Allows creating graphs of related artifacts (signatures, SBOMs attached to images)
3. **Subject Field**: Links artifacts to parent/subject artifacts
4. **artifactType Field**: Differentiates between artifact types

## Key Capabilities

### 1. Multi-Artifact Storage

ACR can store and manage multiple artifact types in the same registry:
- Container images (Docker and OCI format)
- Helm charts as OCI artifacts
- Supply chain artifacts (signatures, SBOMs)
- Custom artifact types

### 2. Artifact Graph Support

ACR supports creating hierarchical graphs of artifacts:
```
container-image:v1
├── signature/example
│   └── sha256:555ea91f39e7fb30...
├── sbom/example
│   └── sha256:4f1843833c029ecf...
│       └── signature/example
│           └── sha256:3c43b8cb0c941ec1...
└── readme/example
    └── sha256:1a118663d1085e22...
```

### 3. Tag-Free Artifact Management

Supply chain artifacts (signatures, SBOMs) can be stored without tags:
- Artifacts referenced by digest
- Reduces tag clutter in repositories
- Automatic association with subject artifacts

### 4. Artifact Discovery

The OCI Referrers API enables:
- Discovering all artifacts referencing a subject
- Filtering by artifact type
- Building complete artifact dependency graphs

## Service Tier Support

| Feature | Basic | Standard | Premium |
|---------|-------|----------|---------|
| OCI Artifact Storage | Yes | Yes | Yes |
| Helm Charts as OCI | Yes | Yes | Yes |
| ORAS Push/Pull | Yes | Yes | Yes |
| Artifact Referrers API | Yes | Yes | Yes |
| Geo-replicated Artifacts | No | No | Yes |
| CMK Encryption* | No | No | Yes |

*Note: CMK-encrypted registries use OCI Referrers Tag Schema instead of Referrers API

## Use Cases

### 1. Software Supply Chain Security
Store and verify signed container images with attached:
- Digital signatures (Notation/Notary Project)
- SBOMs for dependency tracking
- Vulnerability scan results

### 2. Helm Chart Distribution
Distribute Helm charts as OCI artifacts:
- Native registry storage (no separate chart repository)
- Version control via tags
- Integrated access control

### 3. Multi-Architecture Support
Store multi-architecture images with:
- Image index (manifest list) support
- Platform-specific variants
- Unified pull experience

### 4. Configuration Management
Store configuration bundles:
- OPA policy bundles
- Kubernetes configuration
- Application settings

## Limitations and Considerations

### Current Limitations
1. **CMK-Encrypted Registries**: Use Referrers Tag Schema instead of Referrers API
2. **Geo-Replication**: Supply chain artifacts may have delayed replication in some scenarios
3. **Import**: Maximum 50 manifests per imported image

### Best Practices
1. Always reference artifacts by digest for immutability
2. Use appropriate artifact types for different content
3. Leverage ORAS CLI for non-container artifacts
4. Implement signature verification in CI/CD pipelines

## Related Documentation

- [Content Formats Supported in ACR](container-registry-image-formats.md)
- [Push and Pull OCI Artifacts with ORAS](container-registry-manage-artifact.md)
- [Push and Pull Helm Charts](container-registry-helm-repos.md)
- [Overview of Signing and Verifying OCI Artifacts](overview-sign-verify-artifacts.md)

## Sources

- `/submodules/azure-management-docs/articles/container-registry/container-registry-image-formats.md`
- `/submodules/azure-management-docs/articles/container-registry/container-registry-manage-artifact.md`
- `/submodules/azure-management-docs/articles/container-registry/container-registry-concepts.md`
- `/submodules/acr/docs/container-registry-oras-artifacts.md`
- `/submodules/acr/docs/artifact-media-types.json`

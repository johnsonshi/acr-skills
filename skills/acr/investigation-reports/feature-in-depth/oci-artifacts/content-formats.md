# Azure Container Registry - Content Formats Reference

## Overview

Azure Container Registry supports a wide range of content formats based on the OCI Distribution Specification. This document provides a comprehensive reference of all supported content formats, their media types, and usage scenarios.

## Container Image Formats

### Docker Image Formats

#### Docker Image Manifest V2, Schema 1 (Legacy)

| Property | Value |
|----------|-------|
| Media Type | `application/vnd.docker.distribution.manifest.v1+json` |
| Status | Legacy, supported for backward compatibility |
| Features | Basic image manifest |

#### Docker Image Manifest V2, Schema 2

| Property | Value |
|----------|-------|
| Media Type | `application/vnd.docker.distribution.manifest.v2+json` |
| Status | Current Docker standard |
| Features | Manifest lists, multi-architecture support |

#### Docker Manifest List

| Property | Value |
|----------|-------|
| Media Type | `application/vnd.docker.distribution.manifest.list.v2+json` |
| Purpose | Multi-architecture image index |
| Use Case | Single tag for multiple platforms |

### OCI Image Formats

#### OCI Image Manifest

| Property | Value |
|----------|-------|
| Media Type | `application/vnd.oci.image.manifest.v1+json` |
| Specification | OCI Image Format Specification |
| Features | Platform-agnostic, extensible |

#### OCI Image Index

| Property | Value |
|----------|-------|
| Media Type | `application/vnd.oci.image.index.v1+json` |
| Purpose | Multi-architecture support |
| Specification | OCI Image Format Specification |

#### OCI Image Config

| Property | Value |
|----------|-------|
| Media Type | `application/vnd.oci.image.config.v1+json` |
| Purpose | Image configuration blob |
| Contents | Environment, entrypoint, working directory |

#### OCI Image Layers

| Layer Type | Media Type |
|------------|-----------|
| Uncompressed tar | `application/vnd.oci.image.layer.v1.tar` |
| Gzip compressed | `application/vnd.oci.image.layer.v1.tar+gzip` |
| Zstd compressed | `application/vnd.oci.image.layer.v1.tar+zstd` |
| Non-distributable | `application/vnd.oci.image.layer.nondistributable.v1.tar` |
| Non-distributable (gzip) | `application/vnd.oci.image.layer.nondistributable.v1.tar+gzip` |

## OCI Artifact Formats

### Generic OCI Artifact Manifest

| Property | Value |
|----------|-------|
| Media Type | `application/vnd.oci.artifact.manifest.v1+json` |
| Purpose | Generic artifact storage |
| Key Field | `artifactType` specifies content type |

### Artifact Type Examples

Common artifact types in OCI artifacts:

| Artifact Type | Description |
|---------------|-------------|
| `readme/example` | Documentation files |
| `signature/example` | Digital signatures |
| `sbom/example` | Software bill of materials |
| Custom types | User-defined artifact types |

## Helm Chart Formats

### Helm Chart Config

| Property | Value |
|----------|-------|
| Media Type | `application/vnd.cncf.helm.config.v1+json` |
| Purpose | Helm chart configuration |
| Contents | Chart metadata |

### Helm Chart Content

| Property | Value |
|----------|-------|
| Media Type | `application/vnd.cncf.helm.chart.content.v1.tar+gzip` |
| Purpose | Packaged Helm chart |
| Contents | Chart templates, values, dependencies |

## Signature Formats

### Notary Project Signatures

| Property | Value |
|----------|-------|
| Media Type | `application/vnd.cncf.notary.signature` |
| Tool | Notation CLI |
| Format | COSE or JWS |

### COSE Signatures

| Property | Value |
|----------|-------|
| Media Type | `application/cose` |
| Standard | CBOR Object Signing and Encryption |
| Use Case | Artifact signatures |

## SBOM Formats

### SPDX Format

| Property | Value |
|----------|-------|
| Media Type | `application/spdx+json` |
| Standard | SPDX 2.3 |
| Use Case | Dependency tracking |

### CycloneDX Format

| Property | Value |
|----------|-------|
| Media Type | `application/vnd.cyclonedx+json` |
| Standard | CycloneDX 1.4+ |
| Use Case | Vulnerability management |

## Policy Formats

### Open Policy Agent (OPA)

| Property | Value |
|----------|-------|
| Config Media Type | `application/vnd.cncf.openpolicyagent.config.v1+json` |
| Bundle Media Type | `application/vnd.cncf.openpolicyagent.layer.v1.tar+gzip` |
| Use Case | Policy-as-code bundles |

## Specialized Container Formats

### Singularity Image Format (SIF)

| Property | Value |
|----------|-------|
| Config Media Type | `application/vnd.sylabs.sif.config.v1+json` |
| Use Case | High-performance computing |
| Environment | Scientific computing, HPC clusters |

## Media Type Categories

### Configuration Blobs

| Format | Media Type |
|--------|-----------|
| Docker | `application/vnd.docker.container.image.v1+json` |
| OCI | `application/vnd.oci.image.config.v1+json` |
| Helm | `application/vnd.cncf.helm.config.v1+json` |
| OPA | `application/vnd.cncf.openpolicyagent.config.v1+json` |
| Singularity | `application/vnd.sylabs.sif.config.v1+json` |

### Content Blobs

| Format | Media Type |
|--------|-----------|
| Markdown | `application/markdown` |
| JSON | `application/json` |
| YAML | `application/yaml` |
| Gzip archive | `application/gzip` |
| Tar archive | `application/x-tar` |
| Binary | `application/octet-stream` |

## IANA Media Type References

OCI artifacts support any valid IANA media type. Common categories:

### Application Types
- `application/json`
- `application/xml`
- `application/yaml`
- `application/pdf`
- `application/zip`

### Text Types
- `text/plain`
- `text/markdown`
- `text/html`
- `text/csv`

### Custom Types
- Follow pattern: `application/vnd.<vendor>.<type>+<format>`
- Example: `application/vnd.mycompany.config.v1+json`

## Content Format Detection

ACR automatically detects content format based on:

1. **Manifest Media Type**: Primary identifier
2. **Config Media Type**: Secondary classification
3. **Layer Media Types**: Content format hints

### Viewing Content Format

```bash
# View manifest media type
az acr manifest show-metadata \
  --registry $ACR_NAME \
  --name $REPO:$TAG

# View detailed manifest
oras manifest fetch --pretty $REGISTRY/$REPO:$TAG
```

## Format Compatibility Matrix

| Content Type | Docker CLI | ORAS CLI | Helm CLI | ACR Import |
|-------------|------------|----------|----------|------------|
| Docker Images | Yes | Yes | No | Yes |
| OCI Images | Yes | Yes | No | Yes |
| Helm Charts | No | Yes | Yes | Yes |
| OCI Artifacts | No | Yes | No | Limited |
| Signatures | No | Yes | No | No |
| SBOMs | No | Yes | No | No |

## Multi-Architecture Support

### Platform Specification

Platforms are specified in image index:
```json
{
  "os": "linux",
  "architecture": "amd64",
  "variant": "v8"
}
```

### Supported Platforms

| OS | Architectures |
|----|---------------|
| Linux | amd64, arm64, arm/v7, arm/v6, 386, ppc64le, s390x |
| Windows | amd64 |

## Best Practices

### 1. Use Standard Media Types
- Prefer IANA-registered types
- Use OCI standard types when available
- Document custom types

### 2. Version Media Types
- Include version in custom types
- Example: `application/vnd.mycompany.config.v1+json`

### 3. Layer Compression
- Use gzip for broad compatibility
- Consider zstd for performance
- Avoid uncompressed for large layers

### 4. Artifact Type Selection
- Use descriptive artifact types
- Follow naming conventions
- Document in team guidelines

## Related Documentation

- [OCI Image Format Specification](https://github.com/opencontainers/image-spec)
- [OCI Distribution Specification](https://github.com/opencontainers/distribution-spec)
- [IANA Media Types](https://www.iana.org/assignments/media-types)

## Sources

- `/submodules/azure-management-docs/articles/container-registry/container-registry-image-formats.md`
- `/submodules/azure-management-docs/articles/container-registry/container-registry-concepts.md`
- `/submodules/acr/docs/artifact-media-types.json`
- `/submodules/azure-management-docs/articles/container-registry/container-registry-manage-artifact.md`

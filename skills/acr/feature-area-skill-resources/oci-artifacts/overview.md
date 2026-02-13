# ACR OCI Artifacts Skill

This skill provides comprehensive knowledge about OCI artifacts support in Azure Container Registry.

## When to Use This Skill

Use this skill when answering questions about:
- Helm charts in OCI registries
- ORAS CLI usage
- Supply chain artifacts (SBOMs, signatures)
- Non-container artifact storage

## Overview

ACR supports OCI Distribution Specification for storing any artifact type, not just container images.

## Supported Artifact Types

| Artifact | Media Type |
|----------|------------|
| Container Images | `application/vnd.oci.image.manifest.v1+json` |
| Helm Charts | `application/vnd.cncf.helm.chart.config.v1+json` |
| Signatures | `application/vnd.cncf.notary.signature` |
| SBOMs (SPDX) | `application/spdx+json` |
| SBOMs (CycloneDX) | `application/vnd.cyclonedx+json` |
| OPA Bundles | `application/vnd.oci.image.layer.v1.tar+gzip` |
| Singularity (SIF) | `application/vnd.sylabs.sif.layer.v1.sif` |

## Helm Charts

### Push Chart
```bash
# Login
helm registry login myregistry.azurecr.io

# Package chart
helm package ./mychart

# Push to ACR
helm push mychart-1.0.0.tgz oci://myregistry.azurecr.io/charts
```

### Pull Chart
```bash
helm pull oci://myregistry.azurecr.io/charts/mychart --version 1.0.0
```

### Install from ACR
```bash
helm install myrelease oci://myregistry.azurecr.io/charts/mychart --version 1.0.0
```

## ORAS CLI

### Push Artifact
```bash
# Login
oras login myregistry.azurecr.io

# Push file
oras push myregistry.azurecr.io/myartifact:v1 ./file.json

# Push with artifact type
oras push myregistry.azurecr.io/myartifact:v1 \
  --artifact-type application/vnd.example.config \
  ./config.yaml

# Push multiple files
oras push myregistry.azurecr.io/myartifact:v1 \
  ./file1.json:application/json \
  ./file2.txt:text/plain
```

### Pull Artifact
```bash
oras pull myregistry.azurecr.io/myartifact:v1 -o ./output/
```

### Attach to Image (Referrers)
```bash
# Attach SBOM to image
oras attach \
  --artifact-type application/spdx+json \
  myregistry.azurecr.io/myapp@sha256:abc123... \
  ./sbom.json

# Discover attached artifacts
oras discover myregistry.azurecr.io/myapp:v1
```

### Copy Between Registries
```bash
# Copy with referrers (signatures, SBOMs)
oras cp -r source.azurecr.io/myapp:v1 target.azurecr.io/myapp:v1
```

## Supply Chain Artifacts

### SBOM Attachment
```bash
# Generate SBOM
syft myregistry.azurecr.io/myapp:v1 -o spdx-json > sbom.json

# Attach to image
oras attach \
  --artifact-type application/spdx+json \
  myregistry.azurecr.io/myapp@sha256:abc123... \
  ./sbom.json
```

### Signature Attachment
```bash
# Sign with Notation
notation sign myregistry.azurecr.io/myapp@sha256:abc123...

# Signature stored as referrer artifact
```

### Discover Artifact Graph
```bash
oras discover myregistry.azurecr.io/myapp:v1 --format tree
```
```
myregistry.azurecr.io/myapp:v1
├── application/spdx+json (sbom)
└── application/vnd.cncf.notary.signature (signature)
```

## Referrers API (OCI 1.1)

ACR supports the OCI 1.1 Referrers API for artifact graphs:

```bash
# List referrers via API
curl -H "Authorization: Bearer $TOKEN" \
  https://myregistry.azurecr.io/v2/myapp/referrers/sha256:abc123...
```

### Fallback for CMK Registries
CMK-encrypted registries use Referrers Tag Schema fallback instead of Referrers API.

## Azure CLI Commands

```bash
# List repositories
az acr repository list --name myregistry

# Show manifest
az acr manifest show \
  --registry myregistry \
  --name myartifact:v1

# List tags
az acr repository show-tags \
  --name myregistry \
  --repository myartifact

# Delete artifact
az acr repository delete \
  --name myregistry \
  --image myartifact:v1
```

## Reference Documentation

- **Investigation reports**: `agent-investigation-reports/acr-features/oci-artifacts/`
- **MS Learn docs**: `azure-management-docs/articles/container-registry/container-registry-manage-artifact.md`

# ACR Content Trust & Signing - Feature Overview

## Executive Summary

Azure Container Registry provides comprehensive container image signing and verification capabilities to ensure the integrity and authenticity of container images throughout the software supply chain. ACR supports two primary signing approaches:

1. **Docker Content Trust (DCT)** - Legacy approach being deprecated (removal: March 31, 2028)
2. **Notary Project / Notation** - Modern, recommended approach using OCI-compliant signatures

## Why Signing Matters

Container images and OCI artifacts flow through the software supply chain from creation to deployment. Without safeguards, attackers could:

- **Alter artifacts** while they're stored or transferred
- **Swap trusted images** with malicious ones
- **Insert unverified base images** or dependencies into builds

Signing and verification ensure:

- **Integrity**: The artifact is exactly as the publisher created it (unmodified)
- **Authenticity**: The artifact truly came from the expected, trusted publisher

## Signing Approaches Comparison

| Feature | Docker Content Trust (DCT) | Notary Project / Notation |
|---------|---------------------------|---------------------------|
| Status | **Deprecated** - Removal March 31, 2028 | **Recommended** - Active development |
| Standard | Docker-proprietary | OCI-compliant (cross-industry standard) |
| Key Storage | Local filesystem | Azure Key Vault, Artifact Signing |
| Certificate Management | Manual | Automated options available |
| CI/CD Integration | Limited | GitHub Actions, Azure DevOps |
| Kubernetes Verification | N/A | Ratify + Azure Policy on AKS |
| Portability | Docker-only | Any OCI-compliant registry |

## Key Components

### 1. Notation CLI
- Open-source command-line tool from Notary Project
- Signs and verifies container images and OCI artifacts
- Supports plug-in architecture for key providers

### 2. Azure Key Vault Integration
- Secure storage for signing keys and certificates
- Key Vault plug-in (`notation-azure-kv`) for Notation
- Supports self-signed and CA-issued certificates

### 3. Artifact Signing Service
- Zero-touch certificate lifecycle management
- Short-lived certificates with automatic rotation
- Simplified signing experience

### 4. Ratify
- CNCF sandbox project for signature verification
- Integrates with Azure Policy on AKS
- Enforces deployment policies based on signature status

## Signing Scenarios

### 1. Image Publisher Signs Images in CI/CD Pipelines
Container image publishers can sign images as part of their build process:
- GitHub Actions workflows using Notation GitHub actions
- Azure DevOps pipelines with Notation tasks
- Local signing using Notation CLI

### 2. Image Consumer Verifies During Deployment on AKS
Organizations can enforce signature verification at deployment time:
- Ratify installed on AKS clusters
- Azure Policy enforcement (Deny or Audit modes)
- Only signed images from trusted identities are deployed

### 3. Base Image Verification in CI/CD
Developers can verify base image signatures before building:
- Protects builds from inheriting vulnerabilities
- Enforces use of only trusted upstream components

### 4. Verification of Other OCI Artifacts
Beyond container images, signature verification supports:
- Software Bills of Materials (SBOMs)
- Helm charts
- Configuration bundles
- AI models

## Timeline and Migration

| Date | Event |
|------|-------|
| May 31, 2026 | DCT cannot be enabled on new registries |
| March 31, 2028 | DCT completely removed from ACR |
| Now | Begin transition to Notary Project/Notation |

## Required Roles and Permissions

### For Docker Content Trust (Legacy)
- **AcrImageSigner**: Required for pushing signed images with DCT
- **AcrPush**: Required alongside AcrImageSigner for push operations
- **AcrPull**: Sufficient for pulling signed images

### For Notation Signing
- **Container Registry Repository Writer** (ABAC) or **AcrPush** (non-ABAC): For pushing images and signatures
- **Container Registry Repository Reader** (ABAC) or **AcrPull** (non-ABAC): For pulling images and signatures
- **Key Vault Certificates Officer**: For creating certificates in Key Vault
- **Key Vault Certificates User**: For reading certificates
- **Key Vault Crypto User**: For signing operations
- **Artifact Signing Certificate Profile Signer**: For using Artifact Signing service

## Related Documentation

- [Notation Signing with Self-Signed Certificates](./notation-signing.md)
- [Docker Content Trust (Legacy)](./dct-legacy.md)
- [Azure Key Vault Integration](./azure-key-vault.md)
- [Verification with Ratify on AKS](./verification.md)
- [GitHub Actions Integration](./github-actions.md)
- [CLI Commands Reference](./cli-commands.md)

## Source Files Referenced

- `/submodules/azure-management-docs/articles/container-registry/overview-sign-verify-artifacts.md`
- `/submodules/azure-management-docs/articles/container-registry/container-registry-content-trust.md`
- `/submodules/azure-management-docs/articles/container-registry/container-registry-content-trust-deprecation.md`
- `/submodules/acr/docs/image-signing.md`
- `/submodules/acr/docs/roles-and-permissions.md`

# ACR Artifact Cache - Feature Overview

## Introduction

Azure Container Registry (ACR) Artifact Cache is a feature that enables you to cache container images from public and private upstream registries into your Azure Container Registry. This pull-through cache mechanism improves performance, reliability, and helps mitigate rate limiting issues from public registries like Docker Hub.

## Key Benefits

### 1. Faster and More Reliable Pull Operations
- Artifact cache enables faster pull operations by caching images locally in your ACR
- Leverages ACR features like geo-replication for higher availability
- Uses availability zone support for improved reliability
- Pulls from the region closest to your Azure resources when using geo-replicated ACR

### 2. Rate Limit Mitigation
- Addresses pull limits imposed by public registries (especially Docker Hub)
- Authenticating cache rules with upstream credentials helps avoid rate limiting
- Once cached, images are pulled from your local ACR instead of the upstream registry

### 3. Network Security and Compliance
- Access cached registries over private networks
- Align with firewall configurations and compliance standards
- Use all ACR security features including:
  - Private networks
  - Firewall configuration
  - Service Principals
  - Managed identities

### 4. Cost Optimization
- Reduces egress costs from repeatedly pulling the same images
- Decreases bandwidth usage to external registries

## Service Tier Availability

Artifact cache is available in **all** Azure Container Registry service tiers:

| SKU | Artifact Cache Support |
|-----|------------------------|
| Basic | Yes |
| Standard | Yes |
| Premium | Yes |

## Core Terminology

### Cache Rule
A rule that defines how to pull artifacts from a supported upstream repository into your cache. A cache rule contains four parts:

| Component | Description | Example |
|-----------|-------------|---------|
| **Rule Name** | Unique identifier for the cache rule | `Hello-World-Cache` |
| **Source** | The name of the source registry | `docker.io` |
| **Repository Path** | The source path to find artifacts | `docker.io/library/hello-world` |
| **New ACR Repository Namespace** | Destination path in your ACR | `hello-world` |

**Important**: The target repository namespace cannot already exist inside the ACR instance.

### Credentials
Authentication information for accessing private or authenticated upstream registries:

| Component | Description |
|-----------|-------------|
| **Credentials Name** | Identifier for the credential set |
| **Source Registry Login Server** | The login server of the source registry |
| **Source Authentication** | Key Vault locations storing credentials |
| **Username Secret** | Secret containing the username |
| **Password Secret** | Secret containing the password |

## Supported Upstream Registries

| Upstream Registry | Authentication Support | Availability |
|-------------------|------------------------|--------------|
| Docker Hub | Authenticated pulls only | Azure CLI, Azure portal |
| Microsoft Artifact Registry (mcr.microsoft.com) | Unauthenticated pulls only | Azure CLI, Azure portal |
| AWS Elastic Container Registry (ECR) Public Gallery | Unauthenticated pulls only | Azure CLI, Azure portal |
| GitHub Container Registry (ghcr.io) | Both authenticated and unauthenticated | Azure CLI, Azure portal |
| Quay (quay.io) | Both authenticated and unauthenticated | Azure CLI, Azure portal |
| Kubernetes Container Image Registry (registry.k8s.io) | Both authenticated and unauthenticated | Azure CLI |
| Google Artifact Registry (*.pkg.dev) | Authenticated pulls only | Azure CLI |
| Legacy Google Container Registry (gcr.io) | Both authenticated and unauthenticated | Azure CLI |

### Docker Hub Special Considerations

- **Credentials are required** - You must generate a credential set for Docker Hub
- Public Docker Hub images mapped to the `library` namespace are automatically handled
- If you don't include the `library` path, artifact cache will automatically include it

### Google Artifact Registry Authentication

For private Google Artifact Registry (GAR):

1. **Recommended**: Use a Service Account Key (created in Google Cloud Console)
2. Define a custom expiry date (e.g., 3 months)
3. Persist the key in Azure Key Vault
4. Set the username to:
   - `_json_key` for JSON format key as provided
   - `_json_key_base64` for base64-encoded key contents

**Note**: Access tokens (from gcloud CLI) are not recommended as they expire after 1 hour.

## Current Limitations

### 1. Pull-Through Behavior
- Cache only occurs **after at least one image pull** is complete
- For every new image, a new pull must be initiated
- Artifact cache does NOT automatically pull new tags when available
- This is not a proactive sync mechanism

### 2. Cache Rule Limits
- Maximum of **1,000 cache rules** per registry

### 3. Overlapping Rules
- Cache rules cannot overlap with other cache rules
- If a cache rule exists for a registry path, you cannot add another overlapping rule
- Static (fixed) rules can overlap with wildcard rules
- Wildcard rules cannot overlap with other wildcard rules

## How It Works

1. **Create Cache Rule**: Define a mapping from upstream registry/repository to your ACR namespace
2. **Configure Credentials** (if required): Store upstream registry credentials in Azure Key Vault
3. **First Pull**: Pull the image from your ACR (this triggers the cache)
4. **Subsequent Pulls**: Images are served from your local ACR cache
5. **New Tags**: Require a new pull to cache (not automatic)

## Best Practices

1. **Always authenticate** when caching from Docker Hub to avoid rate limits
2. **Use geo-replicated ACR** for optimal pull performance across regions
3. **Combine with private networks** for enhanced security
4. **Plan repository namespaces** carefully as they cannot pre-exist
5. **Monitor cache rule usage** to stay within the 1,000 rule limit
6. **Use wildcards** for flexible caching of multiple repositories
7. **Implement retry logic** in your applications for resilience

## Related Features

- **Geo-Replication**: Combine with artifact cache for optimal global performance
- **Private Link**: Access cached images over private networks
- **Azure Key Vault Integration**: Secure storage for upstream credentials
- **Managed Identities**: Secure authentication to Key Vault

## Use Cases

1. **CI/CD Pipelines**: Cache base images to speed up builds and avoid rate limits
2. **Kubernetes Deployments**: Reliable image pulls for AKS clusters
3. **Air-Gapped Environments**: Cache images for restricted network access
4. **Multi-Region Deployments**: Combine with geo-replication for global availability
5. **Development Teams**: Share cached images across teams while maintaining control

## References

- [Artifact Cache Overview](https://learn.microsoft.com/azure/container-registry/artifact-cache-overview)
- [Enable Artifact Cache - CLI](https://learn.microsoft.com/azure/container-registry/artifact-cache-cli)
- [Enable Artifact Cache - Portal](https://learn.microsoft.com/azure/container-registry/artifact-cache-portal)
- [Wildcard Support](https://learn.microsoft.com/azure/container-registry/wildcards-artifact-cache)
- [Troubleshoot Artifact Cache](https://learn.microsoft.com/azure/container-registry/troubleshoot-artifact-cache)

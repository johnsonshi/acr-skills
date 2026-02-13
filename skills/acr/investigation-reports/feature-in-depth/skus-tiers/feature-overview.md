# Azure Container Registry SKUs and Service Tiers - Feature Overview

## Introduction

Azure Container Registry (ACR) is available in three service tiers, also known as SKUs or pricing plans: **Basic**, **Standard**, and **Premium**. Each tier provides different features, capacities, and limits designed to accommodate various use cases from development and testing to enterprise production workloads.

All three tiers share the same programmatic capabilities and data plane APIs, benefiting from Azure-managed storage with encryption-at-rest. The tiers differ primarily in included storage, image throughput, and access to advanced features.

## SKU Descriptions

### Basic Tier

**Purpose**: Cost-optimized entry point for developers learning about Azure Container Registry.

**Key Characteristics**:
- Most affordable option for getting started
- Same core capabilities as Standard and Premium (Microsoft Entra authentication, image deletion, webhooks)
- Lower included storage and image throughput
- Best suited for lower usage scenarios and development/testing environments
- Some advanced features are not available

**Use Cases**:
- Individual developers learning ACR
- Small development teams
- Proof of concept projects
- Low-traffic personal projects

### Standard Tier

**Purpose**: Production-ready tier for most enterprise scenarios with increased capacity.

**Key Characteristics**:
- Increased included storage compared to Basic
- Higher image throughput than Basic
- Same feature set as Basic
- Satisfies the needs of many production scenarios
- Better suited for teams and production workloads

**Use Cases**:
- Production applications with moderate traffic
- Small to medium enterprise deployments
- CI/CD pipelines with regular image builds
- Multi-team development environments

### Premium Tier

**Purpose**: Enterprise-grade tier for high-volume scenarios with advanced features.

**Key Characteristics**:
- Highest amount of included storage
- Maximum concurrent operations and throughput
- All features from Basic and Standard
- Exclusive access to advanced enterprise features
- Designed for high-volume, hyper-scale scenarios

**Exclusive Premium Features**:
- **Geo-replication**: Manage a single registry across multiple Azure regions
- **Private Link with Private Endpoints**: Restrict registry access via private network
- **Zone Redundancy**: Enhanced availability using Azure availability zones
- **Content Trust (Docker Content Trust)**: Image signing and verification (being deprecated)
- **Customer-Managed Keys (CMK)**: Use your own encryption keys stored in Azure Key Vault
- **Dedicated Agent Pools**: Run ACR Tasks on dedicated compute pools
- **Artifact Streaming**: Stream container images to AKS clusters (Preview)
- **Connected Registry**: On-premises or remote registry replicas
- **Dedicated Data Endpoints**: Mitigate data exfiltration with dedicated endpoints
- **Retention Policy for Untagged Manifests**: Automatic cleanup policies (Preview)
- **IP Network Rules**: Configure firewall rules for registry access
- **Higher API Concurrency and Bandwidth**: Better performance for large-scale deployments

**Use Cases**:
- Large enterprise deployments
- Multi-region global applications
- High-security environments requiring private network access
- Organizations with compliance requirements for encryption
- IoT and edge computing scenarios
- High-volume CI/CD pipelines

## Core Features Available Across All Tiers

### Authentication and Security
- Microsoft Entra ID authentication integration
- Service principal authentication
- Admin account (not recommended for production)
- Azure RBAC for access control
- Token-based repository permissions with scope maps

### Registry Operations
- Push and pull container images
- Support for Docker and OCI image formats
- Helm chart storage
- OCI artifact storage
- Image deletion capabilities
- Repository management

### Integration
- Webhooks for registry events
- Azure CLI and PowerShell support
- REST API access
- Azure Portal management
- Integration with Azure DevOps and GitHub Actions

### Storage
- Encryption-at-rest for all stored content
- Azure-managed storage
- Regional data storage with data residency compliance
- Soft delete policy for artifact recovery (Preview, all tiers)

### ACR Tasks (All Tiers)
- Quick tasks for on-demand builds
- Triggered tasks (source code updates, base image updates, scheduled)
- Multi-step tasks
- Multi-platform builds (Linux, Windows, ARM)

## Feature Availability Matrix Summary

| Feature | Basic | Standard | Premium |
|---------|-------|----------|---------|
| Microsoft Entra Authentication | Yes | Yes | Yes |
| Image Push/Pull | Yes | Yes | Yes |
| Webhooks | Yes | Yes | Yes |
| Image Deletion | Yes | Yes | Yes |
| ACR Tasks | Yes | Yes | Yes |
| Token-based Permissions | Yes | Yes | Yes |
| Soft Delete Policy | Yes | Yes | Yes |
| Zone Redundancy (Auto) | Yes | Yes | Yes |
| Geo-replication | No | No | Yes |
| Private Link | No | No | Yes |
| Customer-Managed Keys | No | No | Yes |
| Content Trust | No | No | Yes |
| Dedicated Agent Pools | No | No | Yes |
| Connected Registry | No | No | Yes |
| Artifact Streaming | No | No | Yes |
| Dedicated Data Endpoints | No | No | Yes |
| Retention Policy | No | No | Yes |
| IP Network Rules | No | No | Yes |

## Zone Redundancy Update (2026)

As of recent updates, zone redundancy is now enabled by default for all Azure Container Registries in regions that support availability zones. This applies to:
- All SKUs (Basic, Standard, Premium)
- Both new and existing registries in supported regions
- Premium registries with geo-replication (all replicas in supported regions)
- No additional cost

**Important Note**: The Azure portal and tooling may not yet accurately reflect this update. The `zoneRedundancy` property may show as false, but zone redundancy is active for registries in supported regions.

## Choosing the Right Tier

### Choose Basic When:
- Learning Azure Container Registry
- Running development or testing workloads
- Operating on a limited budget
- Low storage and throughput requirements
- No need for advanced security features

### Choose Standard When:
- Running production workloads
- Need more storage and throughput than Basic
- Multiple teams sharing a registry
- Moderate CI/CD pipeline usage
- No need for Premium-exclusive features

### Choose Premium When:
- Operating in multiple Azure regions
- Requiring private network access
- Compliance requirements for encryption (CMK)
- High-volume, high-throughput scenarios
- Need for advanced features like connected registries
- Edge/IoT deployment scenarios
- Enterprise security requirements

## Related Documentation

- [Azure Container Registry SKU Features and Limits](container-registry-skus.md)
- [Container Image Storage](container-registry-storage.md)
- [Best Practices for Azure Container Registry](container-registry-best-practices.md)
- [Azure Container Registry Pricing](https://azure.microsoft.com/pricing/details/container-registry/)

# Azure Container Registry - Detailed Tier Comparison Matrix

## Overview

This document provides a comprehensive feature comparison matrix across all Azure Container Registry service tiers (Basic, Standard, and Premium).

## Complete Feature Comparison Matrix

### Storage and Throughput

| Feature | Basic | Standard | Premium |
|---------|-------|----------|---------|
| Included Storage | 10 GiB | 100 GiB | 500 GiB |
| Maximum Storage Limit | 100 GiB | 500 GiB | 4 TiB+ (contact support) |
| Additional Storage Cost | Per-GB rate | Per-GB rate | Per-GB rate |
| Image Throughput | Lower | Moderate | Highest |
| API Concurrency | Lower | Moderate | Highest |
| Bandwidth Throughput | Lower | Moderate | Highest |
| Concurrent Read Operations | Limited | Moderate | High |
| Concurrent Write Operations | Limited | Moderate | High |

### Authentication and Access Control

| Feature | Basic | Standard | Premium |
|---------|-------|----------|---------|
| Microsoft Entra ID Authentication | Yes | Yes | Yes |
| Service Principal Authentication | Yes | Yes | Yes |
| Admin Account | Yes | Yes | Yes |
| Azure RBAC | Yes | Yes | Yes |
| Token-based Repository Permissions | Yes | Yes | Yes |
| Scope Maps for Fine-grained Access | Yes | Yes | Yes |
| Wildcard Permissions in Scope Maps | Yes | Yes | Yes |
| ABAC (Microsoft Entra) Repository Permissions | Yes | Yes | Yes |
| Maximum Tokens | 20,000 | 20,000 | 20,000 |
| Maximum Scope Maps | 20,000 | 20,000 | 20,000 |

### Webhooks

| Feature | Basic | Standard | Premium |
|---------|-------|----------|---------|
| Webhook Support | Yes | Yes | Yes |
| Maximum Webhooks | 10 | 100 | 500 |
| Regional Webhooks (Geo-replicated) | N/A | N/A | Yes |

### Networking and Security

| Feature | Basic | Standard | Premium |
|---------|-------|----------|---------|
| Public Network Access | Yes | Yes | Yes |
| Private Link with Private Endpoints | No | No | Yes |
| Maximum Private Endpoints | N/A | N/A | 200 |
| IP Access Rules (Firewall) | No | No | Yes |
| Maximum IP Rules | N/A | N/A | 200 |
| Virtual Network Rules | No | No | Yes |
| Maximum VNet Rules | N/A | N/A | 100 |
| Dedicated Data Endpoints | No | No | Yes |
| Service Endpoints | No | No | Yes |

### Replication and High Availability

| Feature | Basic | Standard | Premium |
|---------|-------|----------|---------|
| Zone Redundancy (Default) | Yes | Yes | Yes |
| Geo-replication | No | No | Yes |
| Maximum Geo-replications | N/A | N/A | Configurable |
| Regional Failover | No | No | Yes (with geo-replication) |
| Cross-region Content Sync | No | No | Yes |
| Network-close Image Storage | No | No | Yes (with geo-replication) |

### Encryption

| Feature | Basic | Standard | Premium |
|---------|-------|----------|---------|
| Encryption-at-rest (Azure-managed) | Yes | Yes | Yes |
| Customer-Managed Keys (CMK) | No | No | Yes |
| Automatic Key Rotation | N/A | N/A | Yes |
| Manual Key Rotation | N/A | N/A | Yes |
| Azure Key Vault Integration | N/A | N/A | Yes |

### Image Signing and Trust

| Feature | Basic | Standard | Premium |
|---------|-------|----------|---------|
| Docker Content Trust (DCT) | No | No | Yes |
| Image Tag Signing | No | No | Yes |
| Signature Verification | No | No | Yes |
| Notary Integration | No | No | Yes |

**Note**: Docker Content Trust is being deprecated. DCT cannot be enabled on new registries after May 31, 2026, and will be completely removed on March 31, 2028. Transition to Notary Project is recommended.

### ACR Tasks

| Feature | Basic | Standard | Premium |
|---------|-------|----------|---------|
| Quick Tasks (az acr build) | Yes | Yes | Yes |
| Automatically Triggered Tasks | Yes | Yes | Yes |
| Source Code Update Triggers | Yes | Yes | Yes |
| Base Image Update Triggers | Yes | Yes | Yes |
| Scheduled Tasks | Yes | Yes | Yes |
| Multi-step Tasks | Yes | Yes | Yes |
| Multi-platform Builds | Yes | Yes | Yes |
| Linux Platform Support | Yes | Yes | Yes |
| Windows Platform Support | Yes | Yes | Yes |
| ARM Architecture Support | Yes | Yes | Yes |
| Dedicated Agent Pools | No | No | Yes |
| VNet-integrated Agent Pools | No | No | Yes |

### Agent Pool Tiers (Premium Only)

| Tier | Type | CPU | Memory (GB) |
|------|------|-----|-------------|
| S1 | Standard | 2 | 3 |
| S2 | Standard | 4 | 8 |
| S3 | Standard | 8 | 16 |
| I6 | Isolated | 64 | 216 |

### Policies and Retention

| Feature | Basic | Standard | Premium |
|---------|-------|----------|---------|
| Soft Delete Policy | Yes | Yes | Yes |
| Soft Delete Retention Period | 1-90 days | 1-90 days | 1-90 days |
| Retention Policy (Untagged Manifests) | No | No | Yes (Preview) |
| Untagged Manifest Retention Period | N/A | N/A | 0-365 days |
| Image Lock | Yes | Yes | Yes |

### Connected Registry (Premium Only)

| Feature | Basic | Standard | Premium |
|---------|-------|----------|---------|
| Connected Registry Support | No | No | Yes |
| Maximum Connected Registries | N/A | N/A | 50 |
| ReadOnly Mode | N/A | N/A | Yes |
| ReadWrite Mode | N/A | N/A | Yes |
| Nested Connected Registries | N/A | N/A | Yes |
| IoT Edge Integration | N/A | N/A | Yes |
| Azure Arc Integration | N/A | N/A | Yes |
| Maximum Clients per Registry | N/A | N/A | 50 |

### Artifact Streaming (Premium Only, Preview)

| Feature | Basic | Standard | Premium |
|---------|-------|----------|---------|
| Artifact Streaming | No | No | Yes |
| Automatic Streaming Conversion | No | No | Yes |
| AKS Integration | No | No | Yes |
| Linux AMD64 Support | N/A | N/A | Yes |
| Multi-architecture Support | N/A | N/A | Partial (AMD64 only) |
| Private Endpoint Support | N/A | N/A | Yes |

### Storage Features

| Feature | Basic | Standard | Premium |
|---------|-------|----------|---------|
| Managed Storage | Yes | Yes | Yes |
| Regional Data Storage | Yes | Yes | Yes |
| Data Residency Compliance | Yes | Yes | Yes |
| Scalable Repository Count | Yes | Yes | Yes |
| Image Layer Deduplication | Yes | Yes | Yes |
| Content-addressable Storage | Yes | Yes | Yes |

### Image and Artifact Management

| Feature | Basic | Standard | Premium |
|---------|-------|----------|---------|
| Docker Images | Yes | Yes | Yes |
| OCI Images | Yes | Yes | Yes |
| Helm Charts | Yes | Yes | Yes |
| OCI Artifacts | Yes | Yes | Yes |
| Multi-architecture Images | Yes | Yes | Yes |
| Manifest Lists | Yes | Yes | Yes |
| Image Import | Yes | Yes | Yes |
| Cross-registry Import | Yes | Yes | Yes |
| Tag Management | Yes | Yes | Yes |
| Repository Delete | Yes | Yes | Yes |

### Integration and APIs

| Feature | Basic | Standard | Premium |
|---------|-------|----------|---------|
| Azure Portal | Yes | Yes | Yes |
| Azure CLI | Yes | Yes | Yes |
| Azure PowerShell | Yes | Yes | Yes |
| REST API | Yes | Yes | Yes |
| Docker CLI | Yes | Yes | Yes |
| Azure DevOps Integration | Yes | Yes | Yes |
| GitHub Actions Integration | Yes | Yes | Yes |
| Event Grid Integration | Yes | Yes | Yes |
| Azure Monitor Integration | Yes | Yes | Yes |
| Diagnostic Logs | Yes | Yes | Yes |
| Metrics | Yes | Yes | Yes |

### Anonymous Access

| Feature | Basic | Standard | Premium |
|---------|-------|----------|---------|
| Anonymous Pull Access | Yes | Yes | Yes |

## Tier Upgrade Path

### Basic to Standard
- No downtime during upgrade
- All existing content preserved
- No feature removal required

### Standard to Premium
- No downtime during upgrade
- All existing content preserved
- Enables Premium-exclusive features

### Premium to Standard/Basic (Downgrade)
- No downtime during downgrade
- Must remove Premium-exclusive resources first:
  - Delete all geo-replications
  - Delete all connected registries
  - Remove private endpoints
  - Disable CMK encryption (if applicable)
  - Remove IP/VNet rules
- Storage must fit within target tier limits

## Summary Recommendations

### Basic Tier
Best for: Development, testing, learning, personal projects
Not suitable for: Production workloads, enterprise deployments

### Standard Tier
Best for: Production workloads, small to medium enterprises
Not suitable for: Multi-region deployments, high-security requirements

### Premium Tier
Best for: Enterprise deployments, multi-region, high-security, IoT/Edge
Required for: Geo-replication, private endpoints, CMK, connected registries

## Related Documentation

- [Azure Container Registry SKU Features and Limits](container-registry-skus.md)
- [Change Registry SKU](container-registry-skus.md#change-registry-sku)
- [Azure Container Registry Pricing](https://azure.microsoft.com/pricing/details/container-registry/)

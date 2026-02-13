# Azure Container Registry - Pricing Guide

## Overview

Azure Container Registry pricing is based on the service tier (SKU) selected, with costs varying by included storage, additional storage consumption, data transfer, and Premium-exclusive features like geo-replication.

## Pricing Structure

### Base Registry Costs

Each tier has a daily rate that includes:
- Registry management
- Included storage allocation
- Core features for that tier

| Tier | Included Storage | Max Capacity | Primary Use Case |
|------|------------------|--------------|------------------|
| Basic | 10 GiB | 40 TiB | Development/Testing |
| Standard | 100 GiB | 40 TiB | Production |
| Premium | 500 GiB | 40 TiB | Enterprise/Multi-region |

> **Note**: All tiers support the same maximum storage capacity of 40 TiB. "Included Storage" is what's covered by the base price; additional usage is billed per-GiB.

**Pricing Reference**: [Azure Container Registry Pricing](https://azure.microsoft.com/pricing/details/container-registry/)

### Additional Storage Costs

Storage beyond the included allocation is charged at a per-GB rate:

- Charged based on actual consumption
- Same per-GB rate across all tiers
- For geo-replicated registries, storage is consumed in each replica region

**Storage Calculation for Geo-replicated Registries:**
```
Total Storage = Home Region Storage x Number of Replicas
```

### Data Transfer Costs

**Inbound Data Transfer (Push)**:
- Generally free for data coming into Azure

**Outbound Data Transfer (Pull)**:
- Standard Azure bandwidth pricing applies
- Varies by region and destination
- Cross-region transfers may incur additional costs

**Reference**: [Bandwidth Pricing](https://azure.microsoft.com/pricing/details/bandwidth/)

### Geo-replication Costs (Premium Only)

- Each geo-replica incurs the Premium registry daily rate
- Storage is replicated across all regions
- Network transfer between replicas is handled by Azure

**Example Cost Calculation:**
```
Registry with 3 replicas (East US + West US + West Europe):
- 3 x Premium daily rate
- Storage charged per replica
- Data transfer for initial sync and updates
```

### Connected Registry Costs (Premium Only)

Starting August 1, 2025:
- Monthly charge applied to the Azure subscription
- Associated with the parent registry
- On-premises deployment costs (compute, storage) are separate

### ACR Tasks Costs

- Task execution is charged based on compute time
- Dedicated agent pools (Premium) have separate billing
- Billed based on pool allocation, not just execution

### Artifact Streaming Costs (Premium Only)

- May increase overall registry storage consumption
- If exceeding 500 GiB threshold, additional storage charges apply
- No separate feature charge beyond storage

## Cost Optimization Strategies

### 1. Choose the Right Tier

| If You Need | Choose |
|-------------|--------|
| Low storage, basic features | Basic |
| Moderate storage, production use | Standard |
| Advanced features, multi-region | Premium |

### 2. Manage Storage Efficiently

**Reduce Storage Consumption:**
- Delete unused images and repositories regularly
- Use auto-purge (az acr run --cmd) for scheduled cleanup
- Enable retention policy for untagged manifests (Premium)
- Avoid pushing duplicate layers

**Example Cleanup Command:**
```bash
az acr repository delete --name myregistry --repository myrepo --yes
```

### 3. Optimize Geo-replication

- Only replicate to regions where you have active deployments
- Remove replicas in regions you no longer need
- Consider the trade-off between latency and cost

### 4. Use Webhooks Wisely

- Consolidate webhooks where possible
- Remove unused webhooks
- Each tier has webhook limits (Basic: 10, Standard: 100, Premium: 500)

### 5. Efficient Image Management

**Reduce Image Size:**
- Use multi-stage Docker builds
- Choose minimal base images
- Remove unnecessary files and dependencies

**Tag Management:**
- Use meaningful tag strategies
- Avoid proliferation of unused tags
- Implement tag retention policies

### 6. Network Cost Optimization

**For Geo-replicated Registries:**
- Pull from the nearest replica to reduce egress
- Use dedicated data endpoints for firewall rules
- Consider private endpoints to reduce public egress

**For Private Endpoints:**
- Traffic stays within Azure backbone network
- May reduce bandwidth costs for high-volume scenarios

## Cost Monitoring and Budgeting

### Azure Cost Management

1. **Set Up Budgets**: Create budget alerts for registry spending
2. **Monitor Usage**: Track storage and bandwidth consumption
3. **Analyze Costs**: Use cost analysis to identify optimization opportunities

### Monitor Registry Usage

**Azure CLI:**
```bash
az acr show-usage --name myregistry --output table
```

**Azure Portal:**
- Navigate to registry > Overview
- View current storage consumption
- Check resource counts (webhooks, replications, etc.)

### Cost Alerts

Set up alerts for:
- Storage approaching tier limits
- Unexpected spike in pull operations
- Geo-replication costs

## Pricing Examples

### Example 1: Small Development Team

**Setup:**
- Basic tier
- 5 GiB storage used
- Single region
- 10 developers

**Costs:**
- Basic daily rate
- No additional storage (within 10 GiB included)
- Minimal egress costs

### Example 2: Production Application

**Setup:**
- Standard tier
- 80 GiB storage used
- Single region
- CI/CD pipeline with regular builds

**Costs:**
- Standard daily rate
- No additional storage (within 100 GiB included)
- Moderate egress for deployments

### Example 3: Enterprise Multi-region

**Setup:**
- Premium tier
- 400 GiB storage
- 3 geo-replicated regions
- Private endpoints
- Connected registry for edge

**Costs:**
- Premium daily rate x 3 (for geo-replication)
- Storage x 3 (replicated across regions)
- Connected registry monthly fee (starting Aug 2025)
- Reduced egress due to local pulls

### Example 4: High-security Financial Services

**Setup:**
- Premium tier
- 600 GiB storage (above 500 GiB threshold)
- Customer-managed keys
- Private endpoints in 2 regions
- Zone redundancy

**Costs:**
- Premium daily rate x 2 (geo-replication)
- Additional storage charges (100 GiB above threshold)
- Azure Key Vault costs (separate service)
- Private endpoint infrastructure

## Free Tier and Credits

### No Free Tier
Azure Container Registry does not offer a permanent free tier.

### Azure Free Account Credits
- New Azure accounts receive credits that can be used for ACR
- ACR Tasks may be temporarily paused from free credits (check current status)

### Azure DevTest Pricing
- Development/test subscriptions may have discounted rates
- Check Azure pricing calculator for specific scenarios

## Pricing Calculator

Use the Azure Pricing Calculator to estimate costs:
1. Navigate to [Azure Pricing Calculator](https://azure.microsoft.com/pricing/calculator/)
2. Search for "Container Registry"
3. Select tier and configure options:
   - Region
   - Storage amount
   - Geo-replication regions
   - Expected bandwidth

## Cost Comparison: Tier Selection

| Scenario | Recommended Tier | Cost Justification |
|----------|------------------|-------------------|
| Personal project | Basic | Lowest cost, sufficient features |
| Startup MVP | Standard | Balance of cost and capacity |
| Enterprise single-region | Standard or Premium | Depends on security requirements |
| Enterprise multi-region | Premium | Required for geo-replication |
| Regulated industry | Premium | Required for CMK, private endpoints |
| IoT/Edge deployment | Premium | Required for connected registry |

## Related Resources

- [Azure Container Registry Pricing Page](https://azure.microsoft.com/pricing/details/container-registry/)
- [Bandwidth Pricing](https://azure.microsoft.com/pricing/details/bandwidth/)
- [Azure Pricing Calculator](https://azure.microsoft.com/pricing/calculator/)
- [Azure Cost Management](https://docs.microsoft.com/azure/cost-management-billing/)
- [Best Practices for Managing Registry Size](container-registry-best-practices.md#manage-registry-size)

# ACR SKUs & Service Tiers Skill

This skill provides comprehensive knowledge about Azure Container Registry service tiers.

## When to Use This Skill

Use this skill when answering questions about:
- Basic vs Standard vs Premium
- Feature availability
- Limits and quotas
- Pricing
- Upgrading tiers

## Service Tiers

| Feature | Basic | Standard | Premium |
|---------|-------|----------|---------|
| **Included Storage** | 10 GiB | 100 GiB | 500 GiB |
| **Max Storage Capacity** | 40 TiB | 40 TiB | 40 TiB |
| **Webhooks** | 2 | 10 | 500 |
| **Geo-replication** | ❌ | ❌ | ✅ |
| **Private endpoints** | ❌ | ❌ | ✅ |
| **Zone redundancy** | ✅ (default) | ✅ (default) | ✅ |
| **Customer-managed keys** | ❌ | ❌ | ✅ |
| **Connected registry** | ❌ | ❌ | ✅ |
| **Artifact streaming** | ❌ | ❌ | ✅ |
| **Agent pools** | ❌ | ❌ | ✅ |
| **Anonymous pull** | ❌ | ✅ | ✅ |

## Premium-Only Features

- Geo-replication
- Private endpoints / Private Link
- Customer-managed keys (CMK)
- Connected registry
- Artifact streaming
- ACR Task agent pools
- Retention policy
- IP firewall rules (200 rules)
- VNet service endpoints (100 rules)

## Storage Limits

All tiers support up to **40 TiB** maximum storage capacity. The "included storage" is what's covered by the base tier price; usage beyond this is charged per-GiB/month.

| SKU | Included Storage | Max Capacity | Overage |
|-----|------------------|--------------|---------|
| Basic | 10 GiB | 40 TiB | Charged per-GiB |
| Standard | 100 GiB | 40 TiB | Charged per-GiB |
| Premium | 500 GiB | 40 TiB | Charged per-GiB |

## Throughput

| SKU | ReadOps/min | WriteOps/min |
|-----|-------------|--------------|
| Basic | 1,000 | 100 |
| Standard | 3,000 | 500 |
| Premium | 10,000 | 2,000 |

## Quick Commands

### Create Registry
```bash
# Basic
az acr create --name myregistry --resource-group myRG --sku Basic

# Standard
az acr create --name myregistry --resource-group myRG --sku Standard

# Premium
az acr create --name myregistry --resource-group myRG --sku Premium
```

### Check SKU
```bash
az acr show --name myregistry --query sku.name
```

### Upgrade SKU
```bash
az acr update --name myregistry --sku Premium
```

### Downgrade SKU
```bash
# Must disable Premium features first!
az acr replication delete --registry myregistry --location westeurope
az acr update --name myregistry --public-network-enabled true
# Then downgrade
az acr update --name myregistry --sku Standard
```

## Upgrade Path

```
Basic → Standard → Premium
  ↑         ↑         ↓
  └─────────┴─────────┘
```

### Before Downgrading from Premium
- Delete all geo-replications (except home)
- Delete all private endpoints
- Disable CMK encryption
- Delete connected registries
- Disable retention policy
- Ensure IP/VNet rules within limits

## Pricing Factors

| Factor | Basic | Standard | Premium |
|--------|-------|----------|---------|
| Base registry | $ | $$ | $$$ |
| Storage overage | Per GiB/month | Per GiB/month | Per GiB/month |
| Data transfer | Egress charges | Egress charges | Egress charges |
| Geo-replication | N/A | N/A | Per replica |
| Connected registry | N/A | N/A | Per instance (Aug 2025) |

## Cost Optimization

1. **Choose Right SKU**: Start with Standard, upgrade when needed
2. **Clean Up**: Use ACR Purge to delete old images
3. **Artifact Cache**: Reduce Docker Hub pulls
4. **Regional Placement**: Place registry near workloads

## Migration Checklist

### Basic → Standard
- Minimal changes, automatic

### Standard → Premium
- Minimal changes, automatic
- New features available immediately

### Premium → Standard
- [ ] Delete geo-replications
- [ ] Delete private endpoints
- [ ] Disable CMK
- [ ] Delete connected registries
- [ ] Disable retention policy
- [ ] Check firewall rule limits
- [ ] Update IaC templates

## Common Questions

**Q: Can I test Premium features?**
A: Upgrade to Premium, test, downgrade if needed. Pro-rated billing.

**Q: Zone redundancy requires Premium?**
A: No, zone redundancy is now default for all SKUs (2026 update).

**Q: What happens if I exceed storage?**
A: Charged for overage, no service impact.

## Reference Documentation

- **Investigation reports**: `agent-investigation-reports/acr-features/skus-tiers/`
- **MS Learn docs**: `azure-management-docs/articles/container-registry/container-registry-skus.md`

# ACR Geo-Replication Best Practices

## Planning and Design

### 1. Network-Close Deployment Strategy

**Best Practice**: Place registry replicas in the same regions where you deploy containers.

- Minimize latency by locating replicas close to container hosts
- Reduce network egress fees by pulling from local replicas
- Consider all regions where AKS clusters, App Service, or other container workloads run

```
Deployment Regions:
- AKS in East US      -> Replica in East US
- AKS in West Europe  -> Replica in West Europe
- AKS in Southeast Asia -> Replica in Southeast Asia
```

### 2. Region Selection Considerations

| Factor | Consideration |
|--------|---------------|
| Workload locations | Place replicas where containers run |
| Compliance requirements | Consider data residency requirements |
| Cost optimization | Balance number of replicas with cost |
| Latency requirements | More replicas = lower latency globally |
| Disaster recovery | Multiple regions provide failover options |

### 3. Home Region Planning

**Best Practice**: Select your home region strategically.

The home region is where management operations are handled:
- Choose a region close to your primary operations team
- Consider a region with high availability and reliability
- If home region has an outage, management operations pause (data plane continues)

## High Availability

### 4. Combine Geo-Replication with Zone Redundancy

**Best Practice**: Enable zone redundancy in all replica regions that support it.

```bash
# Create zone-redundant registry
az acr create \
  --name myregistry \
  --resource-group myResourceGroup \
  --sku Premium \
  --location eastus \
  --zone-redundancy enabled

# Create zone-redundant replica
az acr replication create \
  --location westeurope \
  --registry myregistry \
  --resource-group myResourceGroup \
  --zone-redundancy enabled
```

**Benefits:**
- Zone redundancy: Within-region resilience (availability zones)
- Geo-replication: Cross-region resilience
- Combined: Maximum availability guarantee

### 5. Regional Outage Preparedness

**Best Practice**: Plan for regional outages before they happen.

**If a replica region fails:**
- Traffic automatically routes to remaining regions
- No manual intervention required
- Consider the increased load on remaining regions

**If home region fails:**
- Data plane operations continue (push/pull)
- Management operations unavailable until recovery
- Cannot add/remove replicas or modify network rules

### 6. Customer-Managed Key Considerations

**Best Practice**: Plan key vault failover for CMK-encrypted registries.

- Review Azure Key Vault failover and redundancy guidance
- Ensure key availability across all geo-replicated regions
- Test key recovery procedures regularly

## Performance Optimization

### 7. DNS Configuration for Push Operations

**Best Practice**: Configure geo-replicated registry in the same Azure regions as push sources.

**Problem**: DNS resolution issues can cause push failures:
- Traffic Manager routes to network-closest replica
- If two nearby regions exist, layers may distribute across both
- Push fails when manifest validation occurs

**Solutions:**
1. Apply client-side DNS cache (e.g., `dnsmasq`) on Linux hosts
2. Configure replicas in same regions as push sources
3. When working outside Azure, choose the closest region

### 8. Image Size Optimization

**Best Practice**: Optimize image sizes to speed up replication.

| Strategy | Benefit |
|----------|---------|
| Multi-stage builds | Smaller final images |
| Minimal base images | Less data to replicate |
| Layer optimization | 5-10 layers is optimal |
| Shared base layers | Reduced storage/transfer |

### 9. Enable Dedicated Data Endpoints

**Best Practice**: Enable dedicated data endpoints for firewall-restricted environments.

```bash
az acr update --name myregistry --data-endpoint-enabled
```

**Benefits:**
- Tightly scoped firewall rules per registry
- Minimize data exfiltration risks
- Regional endpoints for each replica

## Webhook Integration

### 10. Configure Regional Webhooks

**Best Practice**: Set up webhooks to track replication completion per region.

```bash
# Create webhook for specific region
az acr webhook create \
  --name eastus-webhook \
  --registry myregistry \
  --location eastus \
  --uri https://myserver.com/webhook/eastus \
  --actions push delete
```

**Use Cases:**
- Trigger regional deployments when images replicate
- Monitor replication health
- Integrate with CI/CD pipelines

### 11. Webhook-Based Deployment Workflows

**Best Practice**: Use webhooks to automate regional deployments.

```
1. Push image to registry
        |
        v
2. Image replicates to regions
        |
        v
3. Regional webhooks fire on completion
        |
        v
4. Deployment automation triggered per region
```

## Security Best Practices

### 12. Private Link for Geo-Replicated Registries

**Best Practice**: Configure private endpoints in all replica regions.

```bash
# Create private endpoint for each region
# Each replica needs its own private endpoint configuration
# DNS records required for each regional data endpoint
```

**Requirements:**
- Private endpoint per region
- DNS records for data endpoints: `<registry>.<region>.data.azurecr.io`
- Network policies disabled in subnets

**Important**: When adding a new geo-replication with private link:
- Verify user identity has `Microsoft.Network/privateEndpoints/privateLinkServiceProxies/write` permission
- Private endpoint connection may require manual approval

### 13. Firewall Rules for Data Endpoints

**Best Practice**: Configure firewall rules for all regional data endpoints.

```
# Allow access to:
- myregistry.azurecr.io (login server)
- myregistry.eastus.data.azurecr.io
- myregistry.westus.data.azurecr.io
- myregistry.westeurope.data.azurecr.io
```

### 14. Network Security Group Rules

**Best Practice**: Use service tags for simplified NSG management.

```
Service Tag: AzureContainerRegistry
- Global: AzureContainerRegistry
- Regional: AzureContainerRegistry.EastUS
```

## Monitoring and Operations

### 15. Monitor Replication Status

**Best Practice**: Regularly monitor replication health and status.

```bash
# List replications with status
az acr replication list --registry myregistry --output table

# Check endpoints
az acr show-endpoints --name myregistry
```

### 16. Track Storage Usage

**Best Practice**: Monitor storage usage across all replicas.

```bash
az acr show-usage --resource-group myResourceGroup --name myregistry --output table
```

**Note**: In geo-replicated registries, storage usage shown is for home region. Multiply by number of replicas for total storage.

### 17. Configure Resource Logs

**Best Practice**: Enable diagnostic logging for all regions.

- Enable `ContainerRegistryLoginEvents` for authentication tracking
- Enable `ContainerRegistryRepositoryEvents` for push/pull operations
- Monitor the `Geolocation` dimension in metrics

## Cost Optimization

### 18. Right-Size Your Replication

**Best Practice**: Only create replicas in regions where you need them.

| Consideration | Recommendation |
|---------------|----------------|
| Active deployment regions | Create replicas |
| Development/test only | May not need replica |
| Disaster recovery | Create replica for failover |
| Latency-sensitive | Create replica in each region |

### 19. Storage Management

**Best Practice**: Implement retention policies to manage storage costs.

- Configure retention policies for untagged manifests
- Regularly purge unused images
- Storage costs apply per region for geo-replicated registries

## Things to Avoid

### 20. Common Pitfalls

| Avoid | Reason |
|-------|--------|
| Soft delete with geo-replication | Not compatible |
| Force-pushing to all regions | Unnecessary; replication is automatic |
| Skipping zone redundancy | Missing within-region resilience |
| Ignoring DNS issues | Can cause push failures |
| Undersized DNS TTL | May cause routing issues |

### 21. Troubleshooting Disabled Routing

**Best Practice**: Understand when to use disabled routing (for troubleshooting only).

```bash
# Disable routing to a replica (troubleshooting)
az acr replication update --name westus \
  --registry myregistry \
  --resource-group myResourceGroup \
  --region-endpoint-enabled false

# Re-enable after troubleshooting
az acr replication update --name westus \
  --registry myregistry \
  --resource-group myResourceGroup \
  --region-endpoint-enabled true
```

**Note**: Data synchronization continues regardless of routing status.

## Sources

- `/Users/johnsonshi/repos/acr-skills/submodules/azure-management-docs/articles/container-registry/container-registry-geo-replication.md`
- `/Users/johnsonshi/repos/acr-skills/submodules/azure-management-docs/articles/container-registry/container-registry-best-practices.md`
- `/Users/johnsonshi/repos/acr-skills/submodules/azure-management-docs/articles/container-registry/container-registry-troubleshoot-performance.md`
- `/Users/johnsonshi/repos/acr-skills/submodules/azure-management-docs/articles/container-registry/zone-redundancy.md`
- `/Users/johnsonshi/repos/acr-skills/submodules/azure-management-docs/articles/container-registry/container-registry-private-link.md`
- `/Users/johnsonshi/repos/acr-skills/submodules/azure-management-docs/articles/container-registry/container-registry-firewall-access-rules.md`

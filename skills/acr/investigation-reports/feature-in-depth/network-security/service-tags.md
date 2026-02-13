# ACR Service Tags

## Overview

Azure Service Tags are groups of IP address prefixes managed by Microsoft that simplify network security group (NSG) and firewall rule configuration. For Azure Container Registry, service tags enable you to define network access controls without manually tracking IP address changes.

## ACR Service Tag: `AzureContainerRegistry`

The `AzureContainerRegistry` service tag represents all IP address prefixes used by Azure Container Registry services. Microsoft automatically updates this tag as addresses change, eliminating the need for manual IP management.

### Regional Variants

| Tag Format | Scope |
|------------|-------|
| `AzureContainerRegistry` | All ACR IP ranges globally |
| `AzureContainerRegistry.<region>` | Region-specific IPs |

**Examples**:
- `AzureContainerRegistry` - Global
- `AzureContainerRegistry.EastUS` - East US region
- `AzureContainerRegistry.WestEurope` - West Europe region
- `AzureContainerRegistry.AustraliaEast` - Australia East region

## Use Cases

Service tags are used in three primary ACR scenarios:

### 1. Image Import

When importing images from external registries behind firewalls:

- ACR sends requests through `AzureContainerRegistry` service tag IPs
- External registry firewall needs inbound rule for these IPs
- Used when importing from public or other Azure registries

**Configuration**: External firewall must allow inbound from `AzureContainerRegistry` IPs

### 2. Webhooks

When ACR sends webhook notifications to external endpoints:

- ACR originates requests from service tag IPs
- Webhook endpoint firewall needs inbound rule
- Applies to both registry-level and repository-scoped webhooks

**Configuration**: Webhook endpoint firewall must allow inbound from `AzureContainerRegistry` IPs

### 3. ACR Tasks

When ACR Tasks access external resources:

- Task execution requests originate from service tag IPs
- External resources behind firewalls need inbound rules
- Includes base image pulls, package restores, etc.

**Configuration**: External resource firewalls must allow inbound from `AzureContainerRegistry` IPs

## NSG Configuration

### Outbound Rule Example

Allow outbound traffic to ACR from a VNet:

```bash
az network nsg rule create \
  --resource-group myResourceGroup \
  --nsg-name myNSG \
  --name AllowACROutbound \
  --priority 100 \
  --direction Outbound \
  --access Allow \
  --protocol Tcp \
  --source-address-prefixes VirtualNetwork \
  --destination-address-prefixes AzureContainerRegistry \
  --destination-port-ranges 443
```

### Region-Specific Outbound Rule

Restrict to specific region:

```bash
az network nsg rule create \
  --resource-group myResourceGroup \
  --nsg-name myNSG \
  --name AllowACREastUSOutbound \
  --priority 100 \
  --direction Outbound \
  --access Allow \
  --protocol Tcp \
  --source-address-prefixes VirtualNetwork \
  --destination-address-prefixes AzureContainerRegistry.EastUS \
  --destination-port-ranges 443
```

### Azure Firewall Configuration

Using Azure Firewall application rules:

```bash
az network firewall application-rule create \
  --resource-group myResourceGroup \
  --firewall-name myFirewall \
  --collection-name AllowACR \
  --name AllowACROutbound \
  --protocols Https=443 \
  --source-addresses 10.0.0.0/24 \
  --target-fqdns "*.azurecr.io"
```

## IP Address Ranges

### Downloading Service Tag JSON

1. Download from Microsoft:
   - URL: https://www.microsoft.com/download/details.aspx?id=56519
   - File: "Azure IP Ranges and Service Tags - Public Cloud"
   - Format: JSON

2. Update frequency: Weekly

### JSON Structure

**Global ACR ranges**:

```json
{
  "name": "AzureContainerRegistry",
  "id": "AzureContainerRegistry",
  "properties": {
    "changeNumber": 10,
    "region": "",
    "platform": "Azure",
    "systemService": "AzureContainerRegistry",
    "addressPrefixes": [
      "13.66.140.72/29",
      "13.67.8.128/29",
      ...
    ]
  }
}
```

**Regional ACR ranges**:

```json
{
  "name": "AzureContainerRegistry.AustraliaEast",
  "id": "AzureContainerRegistry.AustraliaEast",
  "properties": {
    "changeNumber": 1,
    "region": "australiaeast",
    "platform": "Azure",
    "systemService": "AzureContainerRegistry",
    "addressPrefixes": [
      "13.70.72.136/29",
      ...
    ]
  }
}
```

### Storage Service Tag

For registries not using dedicated data endpoints, you also need `Storage` tag:

```json
{
  "name": "Storage",
  "id": "Storage",
  "properties": {
    "systemService": "AzureStorage",
    "addressPrefixes": [
      "13.65.107.32/28",
      ...
    ]
  }
}
```

Regional storage: `Storage.<region>` (e.g., `Storage.EastUS`)

## Best Practices

### 1. Use Service Tags Over IP Lists

- Service tags are automatically updated by Microsoft
- IP lists require manual maintenance
- Reduces operational overhead

### 2. Regional Scoping

When possible, use regional service tags:
- Reduces attack surface
- More precise access control
- Example: `AzureContainerRegistry.EastUS` instead of `AzureContainerRegistry`

### 3. Combine with NSG Rules

```bash
# Allow ACR outbound
az network nsg rule create \
  --name AllowACR \
  --nsg-name myNSG \
  --priority 100 \
  --destination-address-prefixes AzureContainerRegistry \
  --destination-port-ranges 443 \
  --direction Outbound \
  --access Allow \
  --protocol Tcp

# Allow Storage outbound (if not using dedicated data endpoints)
az network nsg rule create \
  --name AllowStorage \
  --nsg-name myNSG \
  --priority 110 \
  --destination-address-prefixes Storage \
  --destination-port-ranges 443 \
  --direction Outbound \
  --access Allow \
  --protocol Tcp
```

### 4. Security Monitoring

- Monitor traffic using Azure Monitor
- Use Network Watcher for flow logs
- Detect unauthorized traffic patterns

### 5. Regular Review

- Review network security configurations periodically
- Update rules as requirements change
- Remove unused rules

## Agent Pool Firewall Rules

For ACR Task agent pools in VNets, required service tags:

| Service Tag | Port | Purpose |
|-------------|------|---------|
| AzureKeyVault | 443 | Secrets |
| Storage | 443 | Image layers |
| EventHub | 443 | Logging |
| AzureActiveDirectory | 443 | Authentication |
| AzureMonitor | 443, 12000 | Diagnostics |

If registry doesn't use Private Link, also add:
- `AzureContainerRegistry` or `Microsoft.ContainerRegistry` service endpoint

## Microsoft Container Registry (MCR)

For access to MCR (Microsoft-published images like Windows Server):

- Refer to MCR client firewall rules documentation
- URL: https://github.com/microsoft/containerregistry/blob/main/docs/client-firewall-rules.md

## Important Notes

### IP Range Changes

> **IMPORTANT**: IP address ranges for Azure services can change. Updates are published weekly. Download the JSON file regularly and update your access rules. For Azure-based deployments, use service tags instead of IP addresses.

### Service Tag vs IP List

| Approach | Pros | Cons |
|----------|------|------|
| Service Tags | Auto-updated, simple | Only in Azure resources |
| IP Lists | Works anywhere | Manual updates required |

Use service tags when:
- Configuring Azure NSGs
- Using Azure Firewall

Use IP lists when:
- Configuring on-premises firewalls
- Configuring third-party firewalls
- External service configurations

## Troubleshooting

### Service Tag Not Recognized

Ensure you're using the correct format:
- `AzureContainerRegistry` (not `Azure.ContainerRegistry`)
- Case-sensitive in some contexts

### Missing IP Ranges

If ACR requests fail despite service tag rules:
1. Check if `Storage` tag is also needed
2. Verify dedicated data endpoints are configured
3. Download latest IP ranges JSON

### Regional Tag Missing

Some regions may not have dedicated tags:
- Use global `AzureContainerRegistry` tag
- Check latest JSON for available regional tags

## Source Files

- `/submodules/azure-management-docs/articles/container-registry/container-registry-service-tag.md`
- `/submodules/azure-management-docs/articles/container-registry/container-registry-firewall-access-rules.md`
- `/submodules/azure-management-docs/articles/container-registry/tasks-agent-pools.md`

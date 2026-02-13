# ACR Zone Redundancy - Supported Regions

## Overview

Zone redundancy requires Azure regions that support availability zones. This document provides guidance on region selection and availability zone support for Azure Container Registry.

## Azure Availability Zone Requirements

For zone redundancy to be effective, the Azure region must have:

- At least **three separate availability zones**
- Physical separation between zones
- Independent power, cooling, and networking per zone

## Finding Supported Regions

Azure regularly expands availability zone support to new regions. To find current supported regions:

### Official Documentation Reference

Refer to the Azure reliability documentation for the most up-to-date list:
- [Azure regions with availability zones](/azure/reliability/regions-list)

### Azure CLI Method

```azurecli
# List regions that support availability zones
az account list-locations --query "[?availabilityZoneMappings != null].{Name:name, DisplayName:displayName}" -o table
```

## Commonly Used Regions with Availability Zone Support

The following regions are commonly used with zone-redundant ACR deployments (verify current support in official docs):

### Americas

| Region | Display Name |
|--------|--------------|
| eastus | East US |
| eastus2 | East US 2 |
| westus2 | West US 2 |
| westus3 | West US 3 |
| centralus | Central US |
| southcentralus | South Central US |
| canadacentral | Canada Central |
| brazilsouth | Brazil South |

### Europe

| Region | Display Name |
|--------|--------------|
| westeurope | West Europe |
| northeurope | North Europe |
| uksouth | UK South |
| francecentral | France Central |
| germanywestcentral | Germany West Central |
| swedencentral | Sweden Central |
| norwayeast | Norway East |
| switzerlandnorth | Switzerland North |

### Asia Pacific

| Region | Display Name |
|--------|--------------|
| australiaeast | Australia East |
| southeastasia | Southeast Asia |
| eastasia | East Asia |
| japaneast | Japan East |
| koreacentral | Korea Central |
| centralindia | Central India |

### Middle East & Africa

| Region | Display Name |
|--------|--------------|
| uaenorth | UAE North |
| southafricanorth | South Africa North |
| qatarcentral | Qatar Central |

## Region Selection Best Practices

### 1. Proximity to Deployments

Select regions closest to where containers will be deployed:
- Reduces network latency for image pulls
- Minimizes egress costs
- Improves deployment performance

### 2. Multi-Region Strategy

For global applications, combine zone redundancy with geo-replication:

```
┌─────────────────────────────────────────────────────────────────────┐
│  Recommended Multi-Region Architecture                               │
│                                                                      │
│  Americas:          Europe:              Asia-Pacific:               │
│  ┌──────────────┐   ┌──────────────┐    ┌──────────────┐            │
│  │ East US      │   │ West Europe  │    │ Southeast    │            │
│  │ (Home, ZR)   │   │ (Replica,ZR) │    │ Asia(Rep,ZR) │            │
│  └──────────────┘   └──────────────┘    └──────────────┘            │
│         │                  │                   │                     │
│         └──────────────────┼───────────────────┘                     │
│                      Geo-Replicated                                  │
└─────────────────────────────────────────────────────────────────────┘
```

### 3. Compliance Considerations

Some regions have specific data residency requirements:

- **Brazil South**: Registry data confined to region (data residency)
- **Southeast Asia**: Registry data confined to region (data residency)
- Check local regulations for data sovereignty requirements

### 4. Feature Availability

Not all features may be available in all regions:
- Verify Premium features availability
- Check for preview feature region restrictions
- ORAS Artifacts preview originally limited to South Central US

## Automatic Zone Redundancy

As of the 2026 update, zone redundancy is **automatically enabled** for all registries in supported regions:

- **No explicit configuration needed** for new registries
- **Applied retroactively** to existing registries
- **All SKUs** (Basic, Standard, Premium) benefit
- **No additional cost**

## Verifying Region Support

### Check Registry Zone Redundancy Status

```azurecli
# Show registry zone redundancy status
az acr show --name <registry-name> --query "[location, zoneRedundancy]"
```

### Check Replica Zone Redundancy Status

```azurecli
# List all replications with zone redundancy status
az acr replication list --registry <registry-name> \
  --query "[].{Location:location, ZoneRedundancy:zoneRedundancy}" \
  --output table
```

## Region Pairing for Disaster Recovery

Consider Azure region pairs when planning for disaster recovery:

| Primary Region | Paired Region |
|---------------|---------------|
| East US | West US |
| East US 2 | Central US |
| West Europe | North Europe |
| Southeast Asia | East Asia |
| Australia East | Australia Southeast |

Zone redundancy within regions + geo-replication across paired regions provides maximum resilience.

## Migration Considerations

### Moving to Zone-Redundant Regions

If your registry is in a region without availability zones:

1. **Create new registry** in zone-supported region
2. **Import images** using `az acr import`
3. **Update client configurations** to new registry
4. **Delete old registry** after migration complete

### Import Example

```azurecli
# Import from old registry to new zone-redundant registry
az acr import \
  --name <new-registry-name> \
  --source <old-registry-name>.azurecr.io/<repository>:<tag> \
  --resource-group <resource-group>
```

## External Resources

- [Azure regions with availability zones](/azure/reliability/regions-list)
- [ACR Region availability](https://azure.microsoft.com/regions/services/)
- [ACR Pricing by region](https://azure.microsoft.com/pricing/details/container-registry/)

## Sources

- `/Users/johnsonshi/repos/acr-skills/submodules/azure-management-docs/articles/container-registry/zone-redundancy.md`
- `/Users/johnsonshi/repos/acr-skills/submodules/azure-management-docs/articles/container-registry/container-registry-storage.md`
- `/Users/johnsonshi/repos/acr-skills/submodules/azure-management-docs/articles/container-registry/TOC.yml`
- `/Users/johnsonshi/repos/acr-skills/submodules/acr/docs/container-registry-oras-artifacts.md`

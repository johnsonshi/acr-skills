# ACR Connected Registry - Feature Overview

## What is Connected Registry?

The Connected Registry is a feature of Azure Container Registry (ACR) that enables an **on-premises or remote replica** that synchronizes container images and OCI (Open Container Initiative) artifacts with your cloud-based Azure container registry.

### Key Purpose

Connected Registry helps **speed up access to registry artifacts** hosted on-premises or remotely by providing a local, performant registry solution that regularly synchronizes content with a cloud-based Azure container registry.

### Why Use Connected Registry?

While a cloud-based Azure container registry provides features like geo-replication, integrated security, Azure-managed storage, and integration with Azure development/deployment pipelines, customers often need to:

- **Extend cloud investments** to on-premises and field solutions
- **Operate in environments** with intermittent or limited connectivity to the cloud
- **Maintain performance and reliability** where container workloads need images and artifacts available nearby

## Use Case Scenarios

Connected Registry is designed for:

1. **Connected factories** - Industrial environments requiring local container image access
2. **Point-of-sale retail locations** - Distributed retail operations
3. **Shipping, oil-drilling, mining** - Occasionally connected environments
4. **Edge computing** - IoT Edge and Azure Arc-enabled Kubernetes deployments
5. **Local development** - Push images locally and sync to cloud

## Pricing and Availability

### Service Tier Requirement

- **Premium SKU only** - Connected registry is available only in the Premium service tier of Azure Container Registry

### Regional Availability

- Can be deployed in **any region** where Azure Container Registry is available
- Connected registry is coupled with the registry's **home region data endpoint**
- Automatic migration for geo-replication is **not supported**

### Billing

Changes to connected registry billing begin on **August 1, 2025**:
- Monthly charge applied to the Azure subscription associated with the parent registry
- See [Azure Container Registry pricing](https://azure.microsoft.com/pricing/details/container-registry/) for details

## Key Components

### Parent Registry

- The cloud-based Azure container registry that serves as the parent
- Top parent in the connected registry hierarchy is always an Azure container registry
- Can be in any Azure cloud or in a private deployment of Azure Stack Hub

### Connected Registry Resource

- Each connected registry is managed as a **resource within the cloud-based ACR**
- Created in Azure and then deployed to on-premises infrastructure
- Has an **activation status** indicating deployment state:
  - **Active** - Deployed on-premises
  - **Inactive** - Not currently deployed

### Sync Token

- Automatically generated token for authenticating with the parent registry
- Used for synchronization and management operations between connected registry and its parent

### Client Tokens

- Non-Microsoft Entra tokens for client access to the connected registry
- Scoped for pull or push access to specific repositories

## Deployment Options

Connected Registry can be deployed on:

1. **Azure Arc-enabled Kubernetes** clusters (recommended)
2. **Azure IoT Edge** devices
3. **Azure Stack Hub** environments
4. Any environment supporting container workloads on-premises

## Current Limitations

| Limitation | Value |
|------------|-------|
| Maximum tokens and scope maps per registry | 20,000 each |
| Repository permissions per scope map | 500 |
| Maximum clients per connected registry | 50 |
| Maximum connected registries per container registry | 50 |
| Continuous sync minMessageTtl | 1 day |
| Continuous sync maxMessageTtl | 90 days |
| Occasional sync minSyncWindow | 1 hour |
| Occasional sync maxSyncWindow | 7 days |

### Feature Limitations

- **Image locking** through repository, manifest, or tag metadata is not supported
- **Repository delete** is not supported in ReadOnly mode
- **Resource logs** for connected registries are not supported
- **Garbage collection** of deleted artifacts is not supported
- **Deletion** requires manual removal of on-premises containers and cloud scope maps/tokens
- No automatic migration for **geo-replication**

## Key Features Summary

| Feature | Description |
|---------|-------------|
| Local caching | Container images cached on-premises for fast access |
| Synchronization | Regular sync with cloud registry |
| Two modes | ReadWrite (pull/push) and ReadOnly (pull only) |
| Token-based auth | Uses ACR tokens for client and sync authentication |
| Hierarchical deployment | Supports nested connected registries |
| TLS encryption | Secure communication with cert-manager or BYOC |
| Offline resilience | Operates during cloud connectivity outages |

## Sources

- `/Users/johnsonshi/repos/acr-skills/submodules/azure-management-docs/articles/container-registry/intro-connected-registry.md`
- `/Users/johnsonshi/repos/acr-skills/submodules/acr/docs/preview/connected-registry/intro-connected-registry.md`
- `/Users/johnsonshi/repos/acr-skills/submodules/acr/docs/preview/connected-registry/README.md`

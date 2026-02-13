# ACR Task Agent Pools - Feature Overview

## Executive Summary

ACR Task Agent Pools provide Azure-managed dedicated compute environments for running Azure Container Registry (ACR) Tasks. Instead of using the default shared compute environment, you can configure dedicated VM pools that offer enhanced networking capabilities, scalability, and isolation for your container build and automation workloads.

## What Are ACR Task Agent Pools?

An ACR Task Agent Pool is an Azure-managed VM pool (*agent pool*) that enables running Azure Container Registry tasks in a dedicated compute environment. After configuring one or more pools in your registry, you can choose a pool to run a task in place of the service's default compute environment.

### Key Definition

Agent pools are dedicated compute resources provisioned specifically for your registry. They are separate from the default multi-tenant ACR Tasks compute infrastructure, giving you more control over the execution environment.

## Core Benefits

### 1. Virtual Network (VNet) Support
- Assign an agent pool to an Azure VNet
- Access resources within the VNet such as:
  - Container registries (including private registries)
  - Azure Key Vault
  - Azure Storage
- Enables scenarios requiring network isolation and private connectivity

### 2. Scalability
- Increase the number of instances for compute-intensive tasks
- Scale down to zero when not in use
- Billing is based on pool allocation (pay only for provisioned capacity)

### 3. Flexible Resource Options
- Multiple pool tiers with different CPU and memory configurations
- Choose resources appropriate for your workload requirements
- Standard tiers for general workloads
- Isolated tier for high-performance, security-sensitive workloads

### 4. Azure Management
- Task pools are patched and maintained by Azure
- Provides reserved allocation without manual VM maintenance
- No need to manage individual VMs
- Hybrid managed pools balance control and management overhead

## Feature Availability

### Service Tier Requirement
- **Premium** container registry service tier required
- Not available in Basic or Standard tiers

### Preview Status
This feature is currently in **preview**, which means:
- Some limitations apply
- Aspects may change prior to General Availability (GA)
- Subject to supplemental terms of use

### Regional Availability
Agent pools are available in the following regions:
- **US Regions**: West US 2, South Central US, East US 2, East US, Central US
- **European Regions**: West Europe, North Europe, Switzerland North, Sweden Central
- **Canadian Regions**: Canada Central
- **Asian Regions**: East Asia
- **India Regions**: Jio India West, Jio India Central
- **Government Regions**: USGov Arizona, USGov Texas, USGov Virginia

## Pool Tiers

| Tier | Type     | CPU (vCPU) | Memory (GB) | Use Case |
|------|----------|------------|-------------|----------|
| S1   | Standard | 2          | 3           | Light workloads, simple builds |
| S2   | Standard | 4          | 8           | Medium workloads, typical builds |
| S3   | Standard | 8          | 16          | Heavy workloads, complex builds |
| I6   | Isolated | 64         | 216         | High-performance, security-sensitive workloads |

### Tier Selection Guidance

- **S1**: Suitable for basic container builds with minimal dependencies
- **S2**: Good balance for most CI/CD pipelines and build scenarios
- **S3**: Recommended for multi-stage builds, large images, or compute-intensive tasks
- **I6**: Enterprise scenarios requiring isolation, very large builds, or specialized compliance requirements

## Quota and Limits

### Default vCPU Quotas
- **Standard agent pools (S1, S2, S3)**: 16 vCPU total per registry
- **Isolated agent pools (I6)**: 0 vCPU (requires support ticket to enable)

### Increasing Quotas
To increase your quota allocation, open a support request through the Azure portal.

## Preview Limitations

1. **Linux Only**: Agent pools currently support only Linux nodes; Windows nodes are not supported
2. **Regional Restrictions**: Only available in specific preview regions
3. **Quota Limits**: Default quotas may require support tickets to increase

## Relationship to Network Bypass Policy

As of **June 1, 2025**, the `networkRuleBypassAllowedForTasks` setting affects tasks using System-Assigned Managed Identity (SAMI) tokens. If this setting is `false` (the default), network bypass for SAMI-based tasks is denied.

Agent pools provide an alternative approach:
- When configured in a VNet, agent pools can access network-restricted registries
- Eliminates the need for network bypass policy configuration
- Provides more control over network access

## Source Documentation

- Microsoft Learn: `/articles/container-registry/tasks-agent-pools.md`
- ACR GitHub Repo: `/docs/tasks/agentpool/README.md`
- Network Bypass Policy: `/articles/container-registry/manage-network-bypass-policy-for-tasks.md`

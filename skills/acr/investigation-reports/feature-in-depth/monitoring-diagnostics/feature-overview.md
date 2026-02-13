# ACR Monitoring & Diagnostics - Feature Overview

## Executive Summary

Azure Container Registry (ACR) provides comprehensive monitoring and diagnostics capabilities through integration with Azure Monitor, diagnostic logging, health checks, and Azure Policy compliance. These features enable users to track registry usage, troubleshoot issues, audit operations, and ensure compliance with security standards.

## Core Monitoring Components

### 1. Azure Monitor Integration
- **Platform Metrics**: Storage used, push/pull operations, geo-replication metrics
- **Resource Logs**: Authentication events, repository operations, activity logs
- **Metrics Explorer**: Analyze metrics with other Azure services
- **Alerts**: Configure alert rules for metrics, logs, and activity events

### 2. Diagnostic Settings
- Configure routing of logs to Log Analytics, Storage, or Event Hubs
- Enable collection of resource logs (not collected by default)
- Support for ContainerRegistryRepositoryEvents and ContainerRegistryLoginEvents tables

### 3. Health Check Commands
- `az acr check-health` for environment and registry diagnostics
- Checks Docker daemon, CLI version, DNS resolution, authentication
- Error codes with documented solutions

### 4. Azure Policy Compliance
- Built-in policy definitions for security and compliance
- Regulatory compliance controls (CIS, NIST, PCI-DSS, etc.)
- Policy assignment at resource group, subscription, or management group level

## Key Features by Category

| Category | Feature | Description |
|----------|---------|-------------|
| Metrics | Storage Used | Total storage consumed by registry |
| Metrics | Successful Push/Pull Count | Number of successful operations |
| Logs | Repository Events | Push, pull, delete, untag operations |
| Logs | Login Events | Authentication attempts and status |
| Logs | Activity Log | Management operations (create, delete, update) |
| Health | Environment Check | Docker, CLI, network diagnostics |
| Health | Registry Access | Authentication, DNS, connectivity |
| Policy | Built-in Definitions | Network, encryption, access control policies |

## Integration Points

1. **Azure Monitor Logs (Log Analytics)**: Query registry data using KQL
2. **Azure Storage**: Archive logs for long-term retention
3. **Event Hubs**: Stream logs to external SIEM systems
4. **Azure Advisor**: Get recommendations for registry optimization
5. **Microsoft Defender for Cloud**: Vulnerability scanning integration

## Availability by SKU

| Feature | Basic | Standard | Premium |
|---------|-------|----------|---------|
| Platform Metrics | Yes | Yes | Yes |
| Resource Logs | Yes | Yes | Yes |
| Diagnostic Settings | Yes | Yes | Yes |
| Health Check | Yes | Yes | Yes |
| Azure Policy | Yes | Yes | Yes |
| Geo-replication Metrics | No | No | Yes |

## Related Documentation

- [Monitor Azure Container Registry](https://learn.microsoft.com/azure/container-registry/monitor-container-registry)
- [Monitoring Data Reference](https://learn.microsoft.com/azure/container-registry/monitor-container-registry-reference)
- [Check Health of Registry](https://learn.microsoft.com/azure/container-registry/container-registry-check-health)
- [Health Check Error Reference](https://learn.microsoft.com/azure/container-registry/container-registry-health-error-reference)
- [Azure Policy for ACR](https://learn.microsoft.com/azure/container-registry/container-registry-azure-policy)

## Source Files

- `/submodules/azure-management-docs/articles/container-registry/monitor-container-registry.md`
- `/submodules/azure-management-docs/articles/container-registry/monitor-container-registry-reference.md`
- `/submodules/azure-management-docs/articles/container-registry/container-registry-check-health.md`
- `/submodules/azure-management-docs/articles/container-registry/container-registry-health-error-reference.md`
- `/submodules/azure-management-docs/articles/container-registry/container-registry-azure-policy.md`
- `/submodules/azure-management-docs/articles/container-registry/policy-reference.md`
- `/submodules/azure-management-docs/articles/container-registry/security-controls-policy.md`
- `/submodules/acr/docs/Troubleshooting Guide.md`

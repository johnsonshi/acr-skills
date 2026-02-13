# ACR Task Agent Pools - Architecture

## Architectural Overview

ACR Task Agent Pools represent a dedicated compute layer within the Azure Container Registry service architecture. They provide isolated, customer-specific VM resources for executing ACR Tasks instead of using the shared multi-tenant compute infrastructure.

## Component Architecture

```
+------------------------+
|   Azure Container      |
|   Registry (Premium)   |
+------------------------+
           |
           v
+------------------------+
|   ACR Tasks Service    |
|   (Task Orchestration) |
+------------------------+
           |
    +------+------+
    |             |
    v             v
+--------+   +------------------+
| Default|   |  Agent Pool(s)   |
| Compute|   |  (Dedicated VMs) |
| (Shared)|  +------------------+
+--------+            |
                      v
              +---------------+
              | Azure VNet    |
              | (Optional)    |
              +---------------+
                      |
              +-------+-------+
              |       |       |
              v       v       v
           +---+   +---+   +---+
           |KV |   |ACR|   |Stg|
           +---+   +---+   +---+
```

## Key Architectural Components

### 1. Task Orchestration Layer

The ACR Tasks service manages:
- Task definitions and scheduling
- Run queue management
- Routing to appropriate compute (default or agent pool)
- Log collection and storage

### 2. Compute Layer Options

#### Default Compute (Multi-tenant)
- Shared Azure-managed infrastructure
- Automatic scaling
- No VNet access
- No customization options

#### Agent Pool (Dedicated)
- Customer-specific VM instances
- Manual scaling (via count parameter)
- VNet integration capability
- Tier selection for resource allocation

### 3. Network Integration Layer

When agent pools are deployed in a VNet:

```
+----------------------------------+
|          Azure VNet              |
|  +----------------------------+  |
|  |     Agent Pool Subnet      |  |
|  |  +----------+ +----------+ |  |
|  |  | Agent VM | | Agent VM | |  |
|  |  | Instance | | Instance | |  |
|  |  +----------+ +----------+ |  |
|  +----------------------------+  |
|              |                   |
|  +-----------+-----------+       |
|  |                       |       |
|  v                       v       |
|+------+  +--------+  +-------+   |
||Target|  | Azure  |  | Azure |   |
|| ACR  |  |KeyVault|  |Storage|   |
||(Private)|        |           |   |
|+------+  +--------+  +-------+   |
+----------------------------------+
```

## Instance Architecture

### Agent Pool Instance Lifecycle

```
[Create Pool] --> [Provisioning] --> [Ready] --> [Running Tasks] --> [Idle]
     |                                                |              |
     v                                                v              v
[Set Count=0] <----------------------------------[Scale Down] <--[Update]
```

### Instance States
1. **Provisioning**: VM being created and configured
2. **Ready**: VM ready to accept tasks
3. **Running**: Actively executing a task
4. **Idle**: Available for new tasks
5. **Scaling**: Count being adjusted

## Service Dependencies

Agent pools require outbound connectivity to the following Azure services:

| Service | Port | Purpose |
|---------|------|---------|
| Azure Key Vault | 443 | Secrets management |
| Azure Storage | 443 | Image layer storage |
| Azure Event Hub | 443 | Telemetry and events |
| Azure Active Directory | 443 | Authentication |
| Azure Monitor | 443, 12000 | Diagnostics and logging |

### Service Endpoint Requirements

When using VNet-deployed agent pools, enable service endpoints for:
- `Microsoft.AzureActiveDirectory`
- `Microsoft.ContainerRegistry` (if not using private link)
- `Microsoft.EventHub`
- `Microsoft.KeyVault`
- `Microsoft.Storage`

> **Note**: Azure Monitor currently lacks a service endpoint. Without outbound Monitor connectivity, agents cannot emit diagnostic logs but may still operate.

## Task Execution Architecture

### Task Routing Flow

```
1. Task Submitted
       |
       v
2. ACR Tasks Service
       |
       +-- No agent pool specified --> Default Compute
       |
       +-- Agent pool specified --> Check pool availability
                                           |
                                    +------+------+
                                    |             |
                                    v             v
                              Pool has      Pool empty
                              instances     (count=0)
                                    |             |
                                    v             v
                              Queue task    Task fails
                              on pool       (no capacity)
```

### Task Execution on Agent

```
[Task Received] --> [Clone Context] --> [Execute Steps] --> [Push Results]
        |                |                     |                  |
        v                v                     v                  v
   Authenticate    Git clone or        Docker build/       Push images
   to services     HTTP download       run commands        to registry
```

## Storage Architecture

Agent pools use Azure Storage for:
- **Task Context**: Build context files, Dockerfiles, source code
- **Layer Cache**: Intermediate build layers (when applicable)
- **Logs**: Task execution logs and outputs

### Pre-cached Images

Agent pool nodes maintain pre-cached images for common scenarios:
- ACR CLI images (`mcr.microsoft.com/acr/acr-cli`)
- Base build images

> **Important**: Pre-cached versions update frequently. If tasks reference specific tags, the network must allow pulling from the source registry.

## DNS Architecture

### Custom DNS Considerations

When the subnet has a custom DNS server configured:
- The configuration is inherited by build agents at runtime
- The IP range used by custom DNS must not conflict with Docker's default range (`172.17.0.0/16`)

### Private Registry DNS

For accessing private registries via private endpoints:
- DNS must resolve `{registry}.azurecr.io` to private IP
- Data endpoint DNS (`{registry}.{region}.data.azurecr.io`) must also resolve correctly

## High Availability Considerations

### Pool-Level Availability

- Pools can have multiple instances for task distribution
- No built-in cross-region redundancy for pools
- Pools are scoped to a single registry

### Recommendations

1. **Multiple Instances**: Configure count > 1 for production workloads
2. **Monitor Queue**: Use `--queue-count` to track pending tasks
3. **Separate Pools**: Consider separate pools for different workload types

## Resource Isolation Model

### Standard Tiers (S1, S2, S3)
- Dedicated VMs within shared Azure infrastructure
- Network isolation through VNet deployment
- Resource guaranteed at tier specification

### Isolated Tier (I6)
- Enhanced isolation for security-sensitive workloads
- Larger resource allocation (64 vCPU, 216 GB memory)
- Suitable for compliance requirements

## Integration Points

### With ACR Tasks
- Agent pools are referenced in task definitions
- The `--agent-pool` parameter routes tasks to specific pools

### With Azure RBAC
- `Container Registry Tasks Contributor` role manages agent pools
- Pool management requires Premium registry permissions

### With Private Link
- Agent pools in VNet can access private-link-enabled registries
- Alternative to public IP-based access

## Source Documentation References

- Microsoft Learn: `tasks-agent-pools.md` - Lines 15-27, 91-157
- ACR GitHub: `docs/tasks/agentpool/README.md` - Lines 1-77
- Private Link: `container-registry-private-link.md` - Lines 296-305

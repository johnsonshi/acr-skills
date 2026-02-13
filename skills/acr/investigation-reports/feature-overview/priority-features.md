# ACR Priority Features for Deep Investigation

## Tier 1: Core Features (Highest Priority)

These features are essential for any ACR deployment:

| Feature | Why Priority | SKU Requirement |
|---------|--------------|-----------------|
| **Authentication** | Required for any registry access | All |
| **RBAC & Authorization** | Security foundation | All |
| **Image Management** | Core registry operations | All |
| **SKUs & Service Tiers** | Determines available features | - |

## Tier 2: Enterprise Features (High Priority)

Critical for production enterprise deployments:

| Feature | Why Priority | SKU Requirement |
|---------|--------------|-----------------|
| **Private Endpoints** | Network isolation, compliance | Premium |
| **Network Security** | Firewall, service endpoints | All (limits vary) |
| **Geo-Replication** | Multi-region, disaster recovery | Premium |
| **Customer-Managed Keys** | Data encryption compliance | Premium |
| **Vulnerability Scanning** | Security compliance | All (with Defender) |
| **Content Trust & Signing** | Supply chain security | All |

## Tier 3: Operational Features (Medium Priority)

Important for operational efficiency:

| Feature | Why Priority | SKU Requirement |
|---------|--------------|-----------------|
| **ACR Tasks** | CI/CD automation | All |
| **Artifact Cache** | Reduce external dependencies | All |
| **Soft Delete & Retention** | Data protection | Premium (retention) |
| **Monitoring & Diagnostics** | Observability | All |
| **Webhooks** | Event-driven workflows | All (limits vary) |
| **Event Grid** | Advanced event routing | All |

## Tier 4: Advanced Features (Specialized)

For specific use cases:

| Feature | Use Case | SKU Requirement |
|---------|----------|-----------------|
| **Connected Registry** | Edge/IoT, disconnected environments | Premium |
| **ACR Transfer** | Air-gapped environments | Premium |
| **Artifact Streaming** | Fast AKS pod startup | Premium |
| **Task Agent Pools** | Network-isolated builds | Premium |
| **Continuous Patching** | Automated vulnerability remediation | Premium |
| **Zone Redundancy** | Regional high availability | All (default 2026) |

## Tier 5: Supporting Features (Lower Priority)

Supporting capabilities:

| Feature | Purpose | SKU Requirement |
|---------|---------|-----------------|
| **OCI Artifacts** | Helm charts, SBOMs, custom artifacts | All |
| **Data Loss Prevention** | Export control | Premium |
| **Dedicated Data Endpoints** | Granular firewall rules | Premium |

---

## Investigation Depth Recommendations

### Full Deep Dive (8+ report files)
- Authentication
- ACR Tasks
- Connected Registry
- Content Trust & Signing

### Standard Investigation (5-7 report files)
- RBAC & Authorization
- Private Endpoints
- Geo-Replication
- Artifact Cache
- Customer-Managed Keys
- Image Management
- Vulnerability Scanning

### Focused Investigation (3-5 report files)
- Network Security
- Zone Redundancy
- Task Agent Pools
- Artifact Streaming
- ACR Transfer
- Soft Delete & Retention
- Continuous Patching

### Brief Investigation (2-3 report files)
- Webhooks
- Event Grid
- Monitoring & Diagnostics
- OCI Artifacts
- SKUs & Service Tiers
- Data Loss Prevention
- Dedicated Data Endpoints

---

## Feature Dependencies

```
Authentication ──→ All Features (prerequisite)
         │
         ├──→ RBAC ──→ Scope Maps & Tokens
         │
         └──→ Private Endpoints ──→ Network Security
                     │
                     └──→ Connected Registry
                     └──→ ACR Transfer
                     └──→ Data Loss Prevention

Premium SKU ──→ Geo-Replication
           ──→ Private Endpoints
           ──→ CMK
           ──→ Connected Registry
           ──→ Artifact Streaming
           ──→ Agent Pools
           ──→ Retention Policy

Defender ──→ Vulnerability Scanning ──→ Continuous Patching
```

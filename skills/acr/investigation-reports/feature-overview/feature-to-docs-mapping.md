# ACR Feature to Documentation Mapping

## Feature → Documentation Sources

| Feature Area | MS Learn Files | Azure/acr Files |
|--------------|----------------|-----------------|
| **Authentication** | `container-registry-authentication*.md` (10 files) | `docs/AAD-OAuth.md`, `docs/Token-BasicAuth.md` |
| **RBAC & Authorization** | `container-registry-rbac-*.md` (5 files) | `docs/roles-and-permissions.md`, `docs/preview/abac-repo-permissions/` |
| **Private Endpoints** | `container-registry-private-link.md` | - |
| **Network Security** | `container-registry-access-selected-networks.md`, `container-registry-vnet.md`, `container-registry-firewall-access-rules.md`, `container-registry-service-tag.md`, `allow-access-trusted-services.md` | - |
| **Geo-Replication** | `container-registry-geo-replication.md`, tutorials (3 files) | - |
| **Zone Redundancy** | `zone-redundancy.md` | - |
| **ACR Tasks** | `container-registry-tasks-*.md` (15 files), tutorials (5 files) | `docs/tasks/` (full directory) |
| **Task Agent Pools** | `tasks-agent-pools.md` | `docs/tasks/agentpool/` |
| **Content Trust & Signing** | 9 files including Notation, DCT, Ratify tutorials | `docs/image-signing.md` |
| **Image Management** | `container-registry-import-images.md`, `container-registry-delete.md`, `container-registry-image-lock.md`, etc. (10 files) | - |
| **Artifact Cache** | `artifact-cache-*.md` (5 files) | - |
| **Artifact Streaming** | `container-registry-artifact-streaming.md`, `troubleshoot-artifact-streaming.md` | `docs/teleport/`, `docs/preview/artifact-streaming/` |
| **Connected Registry** | 6 files in MS Learn | `docs/preview/connected-registry/` (extensive) |
| **ACR Transfer** | `container-registry-transfer-*.md` (4 files) | `docs/image-transfer/` (ARM templates) |
| **Continuous Patching** | `key-concept-continuous-patching.md`, `how-to-continuous-patching.md` | `docs/preview/continuous-patching/` |
| **Customer-Managed Keys** | `tutorial-customer-managed-keys.md` (4 files) | - |
| **Soft Delete & Retention** | `container-registry-soft-delete-policy.md`, `container-registry-retention-policy.md`, `container-registry-auto-purge.md` | - |
| **Webhooks** | `container-registry-webhook.md`, `container-registry-webhook-reference.md` | - |
| **Event Grid** | `container-registry-event-grid-quickstart.md` | - |
| **Monitoring & Diagnostics** | `monitor-container-registry.md`, `monitor-container-registry-reference.md` | - |
| **Vulnerability Scanning** | `scan-images-defender.md`, `buffer-gate-public-content.md` | `docs/preview/quarantine/` |
| **OCI Artifacts** | `container-registry-helm-repos.md`, `container-registry-manage-artifact.md` | `docs/container-registry-oras-artifacts.md` |
| **SKUs & Service Tiers** | `container-registry-skus.md` | - |
| **Data Loss Prevention** | `data-loss-prevention.md` | - |
| **Dedicated Data Endpoints** | `container-registry-dedicated-data-endpoints.md` | - |

---

## Documentation Depth by Feature

| Feature | MS Learn Depth | Azure/acr Depth | Combined |
|---------|----------------|-----------------|----------|
| ACR Tasks | ★★★★★ | ★★★★★ | Extensive |
| Connected Registry | ★★★★ | ★★★★★ | Extensive |
| Authentication | ★★★★★ | ★★★ | Very Good |
| Content Trust | ★★★★★ | ★★ | Very Good |
| Artifact Streaming | ★★★ | ★★★★★ | Very Good |
| Geo-Replication | ★★★★★ | ★ | Good |
| Image Management | ★★★★★ | ★ | Good |
| Private Endpoints | ★★★★ | ★ | Good |
| Artifact Cache | ★★★★ | ★ | Good |
| Network Security | ★★★★ | ★ | Good |
| CMK | ★★★★ | ★ | Good |
| RBAC | ★★★★ | ★★★ | Good |
| ACR Transfer | ★★★ | ★★★★ | Good |
| Continuous Patching | ★★★ | ★★★★ | Good |
| Webhooks | ★★★ | ★ | Moderate |
| Monitoring | ★★★ | ★ | Moderate |
| Zone Redundancy | ★★ | ★ | Basic |
| Event Grid | ★★ | ★ | Basic |
| SKUs | ★★★ | ★ | Basic |

---

## Primary Sources by Feature Type

### MS Learn Primary (GA Features)
- Authentication & RBAC
- Private Endpoints & Network Security
- Geo-Replication & Zone Redundancy
- Image Management
- Artifact Cache
- Customer-Managed Keys
- Monitoring & Webhooks
- SKUs & Tiers

### Azure/acr Primary (Preview/Advanced)
- Connected Registry (preview details)
- Artifact Streaming (teleport docs)
- ACR Tasks (samples, advanced scenarios)
- ACR Transfer (ARM templates)
- Continuous Patching (preview)
- Quarantine Pattern

# ACR Feature Taxonomy

## Feature Hierarchy

```
Azure Container Registry
├── Authentication & Access
│   ├── Authentication
│   │   ├── Entra ID (Azure AD)
│   │   ├── Service Principal
│   │   ├── Managed Identity
│   │   ├── Repository-Scoped Tokens
│   │   └── Anonymous Pull
│   └── Authorization (RBAC)
│       ├── Built-in Roles (AcrPull, AcrPush, AcrDelete, etc.)
│       ├── ABAC (Attribute-Based Access Control)
│       ├── Custom Roles
│       └── Scope Maps & Tokens
│
├── Networking & Security
│   ├── Private Endpoints
│   │   ├── Private Link
│   │   ├── Private DNS Zones
│   │   └── NSG Configuration
│   ├── Network Security
│   │   ├── Firewall Rules (IP Rules)
│   │   ├── Service Endpoints
│   │   ├── Service Tags
│   │   └── Trusted Services
│   ├── Data Loss Prevention
│   │   └── Export Policy
│   └── Dedicated Data Endpoints
│       └── Regional Data Endpoints
│
├── High Availability & Replication
│   ├── Geo-Replication
│   │   ├── Multi-Region Replication
│   │   ├── Regional Endpoints
│   │   └── Traffic Routing
│   └── Zone Redundancy
│       └── Availability Zones
│
├── Build & Automation
│   ├── ACR Tasks
│   │   ├── Quick Tasks
│   │   ├── Multi-Step Tasks
│   │   ├── Scheduled Tasks
│   │   ├── Triggers (Git, Base Image)
│   │   └── YAML Reference
│   └── Task Agent Pools
│       ├── Dedicated Compute
│       └── VNet Integration
│
├── Security & Trust
│   ├── Content Trust & Signing
│   │   ├── Notation (Notary Project)
│   │   ├── Docker Content Trust (Legacy)
│   │   └── Verification (Ratify)
│   ├── Vulnerability Scanning
│   │   ├── Microsoft Defender
│   │   └── Image Quarantine
│   ├── Customer-Managed Keys
│   │   ├── Key Vault Integration
│   │   └── Key Rotation
│   └── Continuous Patching
│       ├── Trivy Scanning
│       └── Copa Patching
│
├── Image & Artifact Operations
│   ├── Image Management
│   │   ├── Import
│   │   ├── Delete
│   │   ├── Lock
│   │   ├── Tagging
│   │   └── Multi-Architecture
│   ├── Artifact Cache
│   │   ├── Pull-Through Cache
│   │   ├── Cache Rules
│   │   └── Upstream Credentials
│   ├── OCI Artifacts
│   │   ├── Helm Charts
│   │   ├── ORAS
│   │   ├── SBOMs
│   │   └── Supply Chain Artifacts
│   └── Artifact Streaming
│       └── Lazy Loading for AKS
│
├── Edge & Air-Gapped
│   ├── Connected Registry
│   │   ├── Edge Deployments
│   │   ├── Azure Arc Integration
│   │   └── IoT Edge
│   └── ACR Transfer
│       ├── Export Pipelines
│       ├── Import Pipelines
│       └── Air-Gapped Environments
│
├── Lifecycle Management
│   ├── Soft Delete & Retention
│   │   ├── Soft Delete Policy
│   │   ├── Retention Policy
│   │   └── Auto-Purge (acr purge)
│   └── SKUs & Service Tiers
│       ├── Basic
│       ├── Standard
│       └── Premium
│
└── Observability & Events
    ├── Webhooks
    │   ├── Push Events
    │   ├── Delete Events
    │   └── Quarantine Events
    ├── Event Grid
    │   ├── Event Subscriptions
    │   └── Event Filtering
    └── Monitoring & Diagnostics
        ├── Azure Monitor
        ├── Metrics
        ├── Logs
        └── Health Checks
```

---

## Feature Count by Category

| Category | Feature Areas | Sub-Features |
|----------|---------------|--------------|
| Authentication & Access | 2 | 9 |
| Networking & Security | 4 | 12 |
| High Availability | 2 | 5 |
| Build & Automation | 2 | 8 |
| Security & Trust | 4 | 10 |
| Image Operations | 4 | 15 |
| Edge & Air-Gapped | 2 | 6 |
| Lifecycle Management | 2 | 6 |
| Observability & Events | 3 | 9 |
| **Total** | **25** | **80+** |

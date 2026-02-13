# ACR Connected Registry - Architecture

## Overview Architecture

The Connected Registry architecture consists of a hierarchical structure where:

1. **Cloud Registry (Parent)** - The top-level Azure Container Registry in the cloud
2. **Connected Registries (Children)** - On-premises or remote replicas that sync with the parent

```
┌─────────────────────────────────────────────────────────────────┐
│                         Azure Cloud                              │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │              Azure Container Registry                      │  │
│  │                   (Parent Registry)                        │  │
│  │  ┌─────────────────┐  ┌─────────────────┐                 │  │
│  │  │  Repository A   │  │  Repository B   │                 │  │
│  │  └─────────────────┘  └─────────────────┘                 │  │
│  │                                                            │  │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐     │  │
│  │  │ Sync Token   │  │ Client Token │  │  Scope Map   │     │  │
│  │  └──────────────┘  └──────────────┘  └──────────────┘     │  │
│  └───────────────────────────────────────────────────────────┘  │
│                              │                                   │
│                              │ Data Endpoint                     │
│                              ▼ (*.data.azurecr.io)               │
└──────────────────────────────┼───────────────────────────────────┘
                               │
                    ╔══════════╧══════════╗
                    ║   Sync Protocol     ║
                    ║   (HTTPS/TLS)       ║
                    ╚══════════╤══════════╝
                               │
┌──────────────────────────────┼───────────────────────────────────┐
│                   On-Premises / Edge                              │
│                              │                                    │
│  ┌───────────────────────────┴───────────────────────────────┐   │
│  │           Connected Registry (ReadWrite)                   │   │
│  │  ┌─────────────────┐  ┌─────────────────┐                 │   │
│  │  │  Repository A   │  │  Repository B   │                 │   │
│  │  │    (Synced)     │  │    (Synced)     │                 │   │
│  │  └─────────────────┘  └─────────────────┘                 │   │
│  └───────────────────────────┬───────────────────────────────┘   │
│                              │                                    │
│              ┌───────────────┴───────────────┐                   │
│              │                               │                    │
│  ┌───────────┴───────────┐   ┌───────────────┴───────────┐       │
│  │ Connected Registry    │   │ Connected Registry        │       │
│  │     (ReadOnly)        │   │     (ReadOnly)            │       │
│  │  - Child of above     │   │  - Child of above         │       │
│  └───────────────────────┘   └───────────────────────────┘       │
│                                                                   │
└───────────────────────────────────────────────────────────────────┘
```

## Component Architecture

### 1. Cloud Components

#### Azure Container Registry
- **Login Server**: `<registry-name>.azurecr.io` - Used for authentication
- **Data Endpoint**: `<registry-name>.<region>.data.azurecr.io` - Used for synchronization messages
- Must have **dedicated data endpoint enabled** for connected registry communication

#### Connected Registry Resource
- Managed ARM resource within the cloud registry
- Stores configuration and sync settings
- Tracks activation status and connection state

#### Tokens and Scope Maps
- **Sync Token**: Auto-generated for connected registry to authenticate with parent
- **Client Tokens**: Created for on-premises clients to access connected registry
- **Scope Maps**: Define repository permissions for tokens

### 2. On-Premises Components

#### Connected Registry Runtime
- Container-based service running on-premises
- Available as:
  - **Arc extension** for Arc-enabled Kubernetes
  - **IoT Edge module** for IoT Edge devices
  - **Helm chart** for generic Kubernetes
  - **Docker container** for standalone deployment

#### Data Storage
- Local storage for cached images and artifacts
- Configurable storage location via environment variables
- Uses Persistent Volume Claims (PVC) on Kubernetes

#### Certificate Management
- **Cert-Manager** service for TLS certificate lifecycle
- Support for BYOC (Bring Your Own Certificate)
- Trust distribution to client nodes

## Registry Hierarchy

### Hierarchy Rules

```
Cloud Registry (Top Parent)
    │
    ├── Connected Registry A (ReadWrite or ReadOnly)
    │       │
    │       ├── Connected Registry A1 (ReadWrite or ReadOnly if parent is ReadWrite)
    │       │                         (ReadOnly only if parent is ReadOnly)
    │       │
    │       └── Connected Registry A2 (Same rules)
    │
    └── Connected Registry B (ReadWrite or ReadOnly)
            │
            └── Connected Registry B1 (Same rules)
```

### Parent-Child Compatibility

| Parent Mode | Allowed Child Modes |
|-------------|---------------------|
| ReadWrite (Registry) | ReadWrite, ReadOnly (Mirror) |
| ReadOnly (Mirror) | ReadOnly (Mirror) only |

### Nested IoT Edge Example (ISA-95 Architecture)

```
Layer 5: Enterprise Network (Internet Access)
┌─────────────────────────────────────────────────────┐
│  Azure Cloud Registry                               │
│           │                                         │
│           ▼                                         │
│  IoT Edge VM + Connected Registry (ReadWrite)       │
│  - Syncs with cloud registry                        │
│  - Serves Layer 4                                   │
└─────────────────────────────────────────────────────┘
           │
Layer 4: Site Business Planning and Logistics
┌─────────────────────────────────────────────────────┐
│  IoT Edge VM + Connected Registry (ReadWrite)       │
│  - Syncs with Layer 5 connected registry            │
│  - Serves Layer 3                                   │
└─────────────────────────────────────────────────────┘
           │
Layer 3: Industrial Security Zone
┌─────────────────────────────────────────────────────┐
│  IoT Edge VM + Connected Registry (ReadOnly)        │
│  - Syncs with Layer 4 connected registry            │
│  - Read-only for lower layers                       │
└─────────────────────────────────────────────────────┘
```

## Authentication Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    Authentication Flow                           │
└─────────────────────────────────────────────────────────────────┘

┌──────────────┐                     ┌──────────────────────────┐
│              │   Client Token      │                          │
│   Client     │────────────────────▶│   Connected Registry     │
│  (Docker)    │   (repository       │      (On-premises)       │
│              │    scoped access)   │                          │
└──────────────┘                     └────────────┬─────────────┘
                                                  │
                                                  │ Sync Token
                                                  │ (sync + gateway
                                                  │  permissions)
                                                  ▼
                                     ┌──────────────────────────┐
                                     │     Parent Registry      │
                                     │  (Cloud or Connected)    │
                                     └──────────────────────────┘
```

### Token Types

#### Sync Token
- **Purpose**: Authenticate connected registry with parent
- **Auto-generated**: Created when connected registry resource is created
- **Permissions**:
  - Synchronize selected repositories
  - Read/write sync messages on gateway
- **Endpoints Accessed**:
  - Login server: `<registry>.azurecr.io`
  - Data endpoint: `<registry>.<region>.data.azurecr.io`

#### Client Token
- **Purpose**: Authenticate on-premises clients with connected registry
- **User-created**: Created and associated with connected registry
- **Permissions**: Scoped to specific repositories for read/write
- **Endpoint Accessed**: Connected registry IP/FQDN

## Network Architecture

### Required Network Connectivity

```
Connected Registry ──────┬────────▶ Parent Login Server
                         │         (authentication)
                         │
                         └────────▶ Parent Data Endpoint
                                   (sync messages)
```

### Endpoints

| Endpoint Type | Format | Purpose |
|---------------|--------|---------|
| Login Server | `<registry>.azurecr.io` | Authentication |
| Data Endpoint | `<registry>.<region>.data.azurecr.io` | Synchronization |
| Connected Registry | IP address or FQDN | Client access |

### Port Requirements

| Port | Protocol | Purpose |
|------|----------|---------|
| 443 | HTTPS | Cloud registry communication |
| 8080 | HTTP/HTTPS | Connected registry service (configurable) |
| 8000 | HTTP | API Proxy (IoT Edge) |

## Kubernetes Deployment Architecture

### Arc-Enabled Kubernetes

```
┌─────────────────────────────────────────────────────────────────┐
│                 Arc-Enabled Kubernetes Cluster                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │              connected-registry namespace                  │   │
│  │                                                            │   │
│  │  ┌─────────────────┐  ┌─────────────────┐                 │   │
│  │  │ Connected       │  │ Cert-Manager    │                 │   │
│  │  │ Registry Pod    │  │ (TLS certs)     │                 │   │
│  │  └────────┬────────┘  └─────────────────┘                 │   │
│  │           │                                                │   │
│  │  ┌────────┴────────┐  ┌─────────────────┐                 │   │
│  │  │ ClusterIP       │  │ PVC             │                 │   │
│  │  │ Service         │  │ (Storage)       │                 │   │
│  │  │ (192.x.x.x)     │  │                 │                 │   │
│  │  └─────────────────┘  └─────────────────┘                 │   │
│  │                                                            │   │
│  │  ┌─────────────────────────────────────────────────────┐  │   │
│  │  │            Trust Distribution DaemonSet              │  │   │
│  │  │   (Distributes CA certs to all cluster nodes)        │  │   │
│  │  └─────────────────────────────────────────────────────┘  │   │
│  │                                                            │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Key Kubernetes Resources

| Resource | Purpose |
|----------|---------|
| Deployment | Connected registry pod |
| Service (ClusterIP) | Internal access to registry |
| PersistentVolumeClaim | Data storage for images |
| Secret | Connection string, TLS certificates |
| DaemonSet | Trust distribution to nodes |

## Data Flow

### Image Sync Flow (Cloud to Connected Registry)

```
1. Cloud Registry                    2. Connected Registry
   ┌─────────────┐                     ┌─────────────┐
   │ Repository  │──── Sync ──────────▶│ Repository  │
   │ hello-world │    (scheduled or    │ hello-world │
   │             │     continuous)     │ (cached)    │
   └─────────────┘                     └─────────────┘
```

### Image Push Flow (Connected Registry to Cloud)

```
1. Client                          2. Connected Registry        3. Cloud Registry
   ┌────────┐                        ┌─────────────┐              ┌─────────────┐
   │ docker │──── push ─────────────▶│ Repository  │──── sync ───▶│ Repository  │
   │ push   │    (ReadWrite mode)    │ myapp:v1    │              │ myapp:v1    │
   └────────┘                        └─────────────┘              └─────────────┘
```

## Sources

- `/Users/johnsonshi/repos/acr-skills/submodules/azure-management-docs/articles/container-registry/intro-connected-registry.md`
- `/Users/johnsonshi/repos/acr-skills/submodules/acr/docs/preview/connected-registry/overview-connected-registry-access.md`
- `/Users/johnsonshi/repos/acr-skills/submodules/acr/docs/preview/connected-registry/overview-connected-registry-and-iot-edge.md`
- `/Users/johnsonshi/repos/acr-skills/submodules/azure-management-docs/articles/container-registry/quickstart-connected-registry-arc-cli.md`

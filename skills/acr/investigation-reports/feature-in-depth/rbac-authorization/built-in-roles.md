# Azure Container Registry Built-in Roles

## Overview

ACR provides multiple built-in roles categorized into three types:
- **Control Plane Roles** - Manage registries without data plane access
- **Data Plane Roles** - Access images/artifacts without registry management
- **Privileged Roles** - Broad permissions across control and data planes

> **Important**: Role behavior differs based on the registry's "Role assignment permissions mode":
> - `RBAC Registry Permissions` - Traditional mode
> - `RBAC Registry + ABAC Repository Permissions` - ABAC-enabled mode

---

## Control Plane Roles

### Container Registry Contributor and Data Access Configuration Administrator

**Use Case**: Registry administrators, CI/CD pipelines, or automation needing to create and configure registries without image push/pull permissions.

**Control Plane Permissions**:
- Create, update, view, list, and delete registries
- Configure registry SKUs and availability zones
- Manage geo-replications
- Manage connected registries
- Configure system-assigned managed identity
- Configure network access (public access, firewall rules, VNet service endpoints)
- Configure private endpoints
- Configure authentication settings (admin user, anonymous pull, tokens, auth-as-arm)
- Configure registry policies (retention, quarantine, soft-delete, export policy)
- Configure diagnostics and monitoring (webhooks, Event Grid)

**Data Plane Permissions**: None

---

### Container Registry Configuration Reader and Data Access Configuration Reader

**Use Case**: Auditors, monitoring systems, and vulnerability scanners needing read-only registry configuration access.

**Control Plane Permissions**:
- View and list registries
- View and list role assignments (read-only)
- View and list geo-replications
- View and list connected registries
- View and list managed identities
- View and list network access settings
- View and list private endpoint settings
- View and list authentication settings
- View and list registry policies
- View and list diagnostics settings

**Data Plane Permissions**: None

---

### Container Registry Tasks Contributor

**Use Case**: CI/CD pipelines or automation tools managing ACR Tasks without broader registry access.

**Control Plane Permissions**:
- Manage ACR Tasks (definitions, runs, logs)
- Manage task agent pools
- Run quick builds (`az acr build`) and quick runs (`az acr run`)
- Configure task managed identities
- Manage auto-purge on ACR Tasks

**Data Plane Permissions**: None

> **Note for ABAC-enabled registries**: ACR Tasks don't have default data plane access. Tasks must have `Container Registry Repository Reader/Writer/Contributor` roles assigned to their identity.

---

### Container Registry Transfer Pipeline Contributor

**Use Case**: Managing ACR transfer pipelines for artifact transfers across network/tenant/air-gap boundaries.

**Control Plane Permissions**:
- Manage import pipelines
- Manage export pipelines
- Manage import/export pipeline runs

**Data Plane Permissions**: None

---

### Container Registry Data Importer and Data Reader

**Use Case**: Pipelines that import images with `az acr import` and need to validate import success.

**Control Plane Permissions**:
- Trigger image imports with `az acr import`

**Data Plane Permissions**:
- Pull images and artifacts
- View and list OCI referrer artifacts
- View and list image metadata and tags
- View and list all repositories
- View artifact streaming configuration

---

## Data Plane Roles

### For ABAC-Enabled Registries ("RBAC Registry + ABAC Repository Permissions")

#### Container Registry Repository Reader

**Use Case**: Container hosts, orchestrators, vulnerability scanners, developers needing pull-only access.

**Permissions**:
- Pull images and artifacts
- View and list OCI referrer artifacts
- View and list tags and metadata
- View repository and image locking policies
- View artifact streaming configuration

**ABAC Support**: Yes - can scope to specific repositories

**Does NOT Include**: Repository catalog listing (cannot list repositories)

---

#### Container Registry Repository Writer

**Use Case**: CI/CD pipelines, developers building and pushing images, image signing with OCI referrers.

**Permissions**:
- Push and pull images and artifacts
- Create, view, and list OCI referrer artifacts
- Manage tags (create, read, list, retag, untag)
- Manage repository and image locking policies
- Enable (but not disable) artifact streaming

**ABAC Support**: Yes - can scope to specific repositories

**Does NOT Include**: Repository catalog listing, artifact deletion

---

#### Container Registry Repository Contributor

**Use Case**: Identities managing image lifecycle and cleanup.

**Permissions**:
- Push, pull, and **delete** images and artifacts
- Create, view, list, and **delete** OCI referrer artifacts
- Manage and **delete** tags
- Manage repository and image locking policies
- Enable **and disable** artifact streaming

**ABAC Support**: Yes - can scope to specific repositories

**Does NOT Include**: Repository catalog listing

---

#### Container Registry Repository Catalog Lister

**Use Case**: CI/CD pipelines, developers, vulnerability scanners needing to list all repositories.

**Permissions**:
- List all repositories via `_catalog` API endpoint

**ABAC Support**: No - always grants catalog listing for entire registry

**Does NOT Include**: Push, pull, or view images/artifacts within repositories

---

### For Non-ABAC Registries ("RBAC Registry Permissions")

#### AcrPull

**Use Case**: Container hosts, orchestrators, vulnerability scanners needing pull-only access.

**Permissions**:
- Pull images and artifacts
- View and list OCI referrer artifacts
- View and list tags and metadata
- View and list all repositories
- View artifact streaming configuration for images

**Does NOT Include**: Policies management, quarantined/soft-deleted artifacts access

---

#### AcrPush

**Use Case**: CI/CD pipelines, developers building and pushing images.

**Permissions**:
- Push and pull images and artifacts
- Create, view, list, and delete OCI referrer artifacts
- Manage tags (create, read, list, retag, untag)
- View and list all repositories
- Enable (but not disable) artifact streaming

**Does NOT Include**: Policies management, repository configuration

---

#### AcrDelete

**Use Case**: Identities managing image lifecycle and cleanup.

**Permissions**:
- Delete images, artifacts, digests, and tags
- Disable artifact streaming by deleting streaming OCI referrer

---

#### AcrImageSigner

**Use Case**: Automated processes signing images with Docker Content Trust (DCT).

**Permissions**:
- Sign container images with Docker Content Trust

> **Note**: Docker Content Trust is being deprecated.

---

### Quarantine Roles (Both Registry Modes)

#### AcrQuarantineWriter

**Use Case**: CI/CD pipelines and vulnerability scanners managing quarantined images.

**Permissions**:
- List and read quarantined artifacts
- Modify artifact quarantine status

---

#### AcrQuarantineReader

**Use Case**: Processes that list, read, and pull quarantined images.

**Permissions**:
- View and list (but not manage) quarantined artifacts

---

## Privileged Roles

### Owner

**Use Case**: Administrators needing complete control including role assignment permissions.

**Control Plane Permissions**:
- Full registry management
- Manage role assignments
- All permissions from other control plane roles

**Data Plane Permissions**:
- **ABAC-enabled registries**: None
- **Non-ABAC registries**: Full push, pull, delete, tag management, signing

---

### Contributor

**Use Case**: Identities needing registry management without role assignment permissions.

**Control Plane Permissions**:
- Same as Owner, except role assignment management
- Can only view and list role assignments

**Data Plane Permissions**:
- **ABAC-enabled registries**: None
- **Non-ABAC registries**: Full push, pull, delete, tag management, signing

---

### Reader

**Use Case**: Identities needing read-only registry and configuration access.

**Control Plane Permissions**:
- View and list registries
- View and list all configurations
- View and list role assignments (read-only)

**Data Plane Permissions**:
- **ABAC-enabled registries**: None
- **Non-ABAC registries**: Pull, view tags, list repositories

---

## Role Permissions Matrix (Traditional ACR Roles)

| Role | ARM Access | Create/Delete ACR | Push | Pull | Policy Changes | Quarantine State | Pull Quarantine | Sign Images |
|------|------------|-------------------|------|------|----------------|------------------|-----------------|-------------|
| Owner | X | X | X | X | X | | | |
| Contributor | X | X | X | X | X | | | |
| Reader | X | | | X | | | | |
| AcrPush | | | X | X | | | | |
| AcrPull | | | | X | | | | |
| AcrDelete | | | | | | | | |
| AcrQuarantineWriter | | | | | | X | X | |
| AcrQuarantineReader | | | | | | | X | |
| AcrImageSigner | | | | | | | | X |

---

## Source Documentation

- MS Learn: `/submodules/azure-management-docs/articles/container-registry/container-registry-rbac-built-in-roles-directory-reference.md`
- MS Learn: `/submodules/azure-management-docs/articles/container-registry/container-registry-rbac-built-in-roles-overview.md`
- ACR Repo: `/submodules/acr/docs/roles-and-permissions.md`

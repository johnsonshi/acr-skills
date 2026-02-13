# Azure Container Registry RBAC & Authorization Overview

## Executive Summary

Azure Container Registry (ACR) provides comprehensive role-based access control (RBAC) through Microsoft Entra ID, enabling granular permission management for container registries. ACR supports both **registry-level RBAC** and **repository-level ABAC (Attribute-Based Access Control)** for fine-grained permissions.

## Access Control Mechanisms

ACR supports three main access control mechanisms:

### 1. Microsoft Entra RBAC (Role-Based Access Control)
- Uses Azure built-in roles or custom roles
- Assigns permissions to users, groups, service principals, and managed identities
- Scopes assignments at subscription, resource group, or registry level
- Supports optional ABAC conditions for repository-level permissions

### 2. Microsoft Entra ABAC (Attribute-Based Access Control)
- Extends RBAC with repository-specific conditions
- Scopes role assignments to specific repositories or repository prefixes
- Requires registry configuration with "RBAC Registry + ABAC Repository Permissions" mode
- Uses ABAC-enabled roles like `Container Registry Repository Reader/Writer/Contributor`

### 3. Non-Microsoft Entra Token-Based Permissions
- Uses tokens and scope maps for repository-scoped access
- Doesn't require Microsoft Entra identities
- Ideal for IoT devices, external organizations, and cross-tenant scenarios
- Used by connected registries for sync and client tokens

## Role Assignment Permissions Modes

ACR registries can be configured with one of two modes:

### RBAC Registry Permissions (Traditional Mode)
- Supports standard RBAC role assignments
- No ABAC conditions supported
- Uses roles: `AcrPull`, `AcrPush`, `AcrDelete`, `AcrImageSigner`
- Privileged roles (Owner, Contributor, Reader) have data plane permissions

### RBAC Registry + ABAC Repository Permissions (ABAC Mode)
- Supports RBAC with optional ABAC conditions
- Scopes role assignments to specific repositories
- Uses ABAC-enabled roles: `Container Registry Repository Reader/Writer/Contributor`
- Privileged roles (Owner, Contributor, Reader) have NO data plane permissions
- Requires explicit data plane role assignments for image access

## Identity Types Supported

ACR roles can be assigned to:

1. **Individual Users** - Microsoft Entra user identities
2. **Managed Identities**:
   - Azure Kubernetes Service (AKS) kubelet identity
   - Azure Container Apps (ACA) identity
   - Azure Container Instances (ACI) identity
   - Azure Machine Learning workspace identity
   - Azure App Service identity
   - Azure Web Apps identity
   - Azure Batch identity
   - Azure Functions identity
   - Azure DevOps Pipelines identity
3. **Service Principals** - Including cross-tenant scenarios
4. **Groups** - Microsoft Entra security groups

## Key Features

### Principle of Least Privilege
- Assign only the permissions necessary for an identity's intended function
- Use the most restrictive role that meets requirements
- Prefer ABAC-enabled roles for repository-specific access

### Separation of Control Plane and Data Plane
- **Control Plane**: Create, configure, manage, delete registries
- **Data Plane**: Push, pull, delete images and artifacts within repositories

### Repository-Level Permissions
- Scope access to specific repositories using ABAC conditions
- Use wildcards/prefixes for multiple repositories (e.g., `backend/*`)
- Catalog listing requires separate `Container Registry Repository Catalog Lister` role

## Source Documentation

- MS Learn: `/submodules/azure-management-docs/articles/container-registry/container-registry-rbac-built-in-roles-overview.md`
- MS Learn: `/submodules/azure-management-docs/articles/container-registry/container-registry-rbac-abac-repository-permissions.md`
- MS Learn: `/submodules/azure-management-docs/articles/container-registry/container-registry-rbac-built-in-roles-directory-reference.md`
- MS Learn: `/submodules/azure-management-docs/articles/container-registry/container-registry-rbac-custom-roles.md`
- MS Learn: `/submodules/azure-management-docs/articles/container-registry/container-registry-token-based-repository-permissions.md`
- ACR Repo: `/submodules/acr/docs/roles-and-permissions.md`

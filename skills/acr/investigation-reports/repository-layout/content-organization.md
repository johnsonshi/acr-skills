# ACR Documentation Content Organization

## How Content Is Organized

### MS Learn Documentation

Content is organized by **task/feature area** with a flat file structure:

| Category | File Count | Topics Covered |
|----------|------------|----------------|
| Overview & Introduction | 7 | Concepts, SKUs, storage, formats, best practices |
| Quickstarts | 10 | Getting started with CLI, Portal, PowerShell, ARM, Bicep, Terraform |
| Authentication | 10 | Managed identity, service principal, Kubernetes, ACI |
| Authorization & RBAC | 5 | Built-in roles, ABAC, custom roles, tokens |
| Networking & Security | 8 | Private endpoints, firewalls, service endpoints |
| Geo-Replication & HA | 2 | Multi-region, zone redundancy |
| Image Management | 10 | Import, delete, lock, soft delete, retention |
| Artifact Cache | 5 | Overview, CLI setup, Portal setup, troubleshooting |
| ACR Transfer | 4 | Air-gapped transfers, pipelines |
| Connected Registry | 6 | Edge/on-prem registries |
| ACR Tasks | 20 | Builds, multi-step, triggers, authentication |
| Signing & Verification | 9 | Notation, DCT, Ratify |
| Vulnerability Management | 4 | Defender, continuous patching |
| Customer-Managed Keys | 4 | Encryption, key rotation |
| Monitoring & Webhooks | 4 | Azure Monitor, webhook schema |
| Troubleshooting | 6 | FAQ, common issues, health checks |

### Azure/acr GitHub Repository

Content is organized by **feature area** with nested folders:

| Folder | Purpose |
|--------|---------|
| `docs/` | Core documentation |
| `docs/preview/` | Preview feature documentation |
| `docs/tasks/` | ACR Tasks deep-dive |
| `docs/teleport/` | Artifact Streaming (Project Teleport) |
| `docs/integration/` | CI/CD integrations |
| `docs/image-transfer/` | ARM templates for transfer |
| `samples/` | Code samples in Java and .NET |

---

## Navigation Structure

### MS Learn TOC.yml Organization

```yaml
- name: Overview
  items:
    - Introduction
    - Concepts
    - SKUs and limits

- name: Quickstarts
  items:
    - Portal, CLI, PowerShell, ARM, Bicep, Terraform

- name: Tutorials
  items:
    - ACR Tasks tutorials
    - Geo-replication tutorials
    - CMK tutorials

- name: Concepts
  items:
    - Authentication
    - Authorization (RBAC)
    - Networking

- name: How-to guides
  items:
    - Image management
    - Security features
    - High availability

- name: Reference
  items:
    - CLI reference
    - REST API
    - Troubleshooting
```

### Azure/acr README Navigation

The repository README provides links to:
1. Official MS Learn documentation
2. Preview features (in `docs/preview/`)
3. Code samples
4. Troubleshooting guides
5. FAQ and roadmap

---

## Cross-Reference Patterns

### MS Learn → Azure/acr
- MS Learn docs link to Azure/acr for:
  - Preview features not yet in MS Learn
  - Code samples and ARM templates
  - Community issues and discussions

### Azure/acr → MS Learn
- Azure/acr docs link to MS Learn for:
  - GA feature documentation
  - Official tutorials and quickstarts
  - REST API reference

---

## Content Types

| Type | MS Learn | Azure/acr |
|------|----------|-----------|
| Conceptual | ✅ Primary | Limited |
| Quickstarts | ✅ Primary | Limited |
| Tutorials | ✅ Primary | ✅ Detailed |
| How-to guides | ✅ Primary | ✅ Detailed |
| Reference | ✅ Primary | Schemas, templates |
| Samples | Limited | ✅ Primary |
| Preview features | Limited | ✅ Primary |
| Troubleshooting | Both | Both |

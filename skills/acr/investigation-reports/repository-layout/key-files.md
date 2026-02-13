# ACR Documentation Key Files

## MS Learn Key Files

### Navigation & Structure
| File | Purpose |
|------|---------|
| `TOC.yml` | Main table of contents defining navigation structure |
| `index.yml` | Landing page configuration with card-based links |
| `container-registry-faq.yml` | FAQ in structured YAML format |
| `breadcrumb/TOC.yml` | Breadcrumb navigation for page hierarchy |

### Core Documentation
| File | Description |
|------|-------------|
| `container-registry-intro.md` | Introduction to Azure Container Registry |
| `container-registry-concepts.md` | About registries, repositories, images, and artifacts |
| `container-registry-skus.md` | SKU features, limits, and pricing tiers |
| `container-registry-best-practices.md` | Best practices for using ACR |

### Authentication & Authorization
| File | Description |
|------|-------------|
| `container-registry-authentication.md` | Authentication overview |
| `container-registry-rbac-built-in-roles-overview.md` | RBAC and built-in roles overview |
| `container-registry-rbac-abac-repository-permissions.md` | Azure ABAC repository permissions |

### Networking
| File | Description |
|------|-------------|
| `container-registry-private-link.md` | Set up private endpoint with Private Link |
| `container-registry-access-selected-networks.md` | Configure public registry access |
| `container-registry-firewall-access-rules.md` | Access behind a firewall |

### Key Features
| File | Description |
|------|-------------|
| `container-registry-tasks-overview.md` | About ACR Tasks |
| `container-registry-tasks-reference-yaml.md` | Tasks YAML reference |
| `container-registry-geo-replication.md` | Geo-replication overview |
| `artifact-cache-overview.md` | Artifact cache overview |
| `intro-connected-registry.md` | About connected registry |

---

## Azure/acr Key Files

### Root Level
| File | Purpose |
|------|---------|
| `README.md` | Repository overview with links |
| `SECURITY.md` | Security policy |
| `LICENSE.txt` | License information |

### Documentation Hub
| File | Description |
|------|-------------|
| `docs/README.md` | Documentation overview |
| `docs/AAD-OAuth.md` | AAD integration and OAuth2 authentication |
| `docs/Token-BasicAuth.md` | Bearer token authentication with Basic Auth |
| `docs/Troubleshooting Guide.md` | Common issues and solutions |
| `docs/roles-and-permissions.md` | RBAC roles and permissions |

### Preview Features
| File | Description |
|------|-------------|
| `docs/preview/connected-registry/README.md` | Connected registry preview |
| `docs/preview/artifact-streaming/README.md` | Artifact streaming preview |
| `docs/preview/continuous-patching/README.md` | Continuous patching preview |
| `docs/preview/quarantine/readme.md` | Quarantine pattern |

### ACR Tasks
| File | Description |
|------|-------------|
| `docs/tasks/container-registry-tasks-overview.md` | Tasks overview |
| `docs/tasks/agentpool/README.md` | Dedicated agent pools |
| `docs/tasks/buildx/README.md` | Docker buildx support |

### Samples
| File | Description |
|------|-------------|
| `samples/java/task/README.md` | Java SDK sample for ACR Tasks |
| `samples/dotnetcore/task/README.md` | .NET Core SDK sample for ACR Tasks |
| `samples/dotnetcore/image-transfer/README.md` | ACR Transfer pipeline sample |

### ARM Templates
| Location | Purpose |
|----------|---------|
| `docs/image-transfer/ExportPipelines/` | Export pipeline templates |
| `docs/image-transfer/ImportPipelines/` | Import pipeline templates |
| `docs/tasks/run-as-deployment/` | Task deployment templates |

---

## Configuration Files

### VuePress (docs site)
| File | Purpose |
|------|---------|
| `docs/.vuepress/config.js` | Site configuration |
| `docs/.vuepress/public/files/` | Static assets (SVG icons) |

### GitHub
| File | Purpose |
|------|---------|
| `.github/ISSUE_TEMPLATE/bug_report.md` | Bug report template |
| `.github/ISSUE_TEMPLATE/feature_request.md` | Feature request template |
| `.github/workflows/nodejs.yml` | Node.js build workflow |
| `.github/workflows/stale.yml` | Stale issue management |

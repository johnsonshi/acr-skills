# ACR Documentation Directory Structure

## Source Repositories

Two primary documentation sources were analyzed:

1. **MS Learn Documentation** (`azure-management-docs/articles/container-registry/`)
2. **Azure/acr GitHub Repository** (`acr/`)

---

## MS Learn Documentation Structure

```
articles/container-registry/
├── index.yml                    # Landing page configuration
├── TOC.yml                      # Table of contents navigation
├── container-registry-faq.yml   # FAQ in YAML format
├── breadcrumb/
│   └── TOC.yml                  # Breadcrumb navigation
├── includes/                    # Reusable content snippets (12 files)
├── media/                       # Screenshots and diagrams
└── [111 main documentation files]
```

### File Count
| File Type | Count | Description |
|-----------|-------|-------------|
| Markdown (`.md`) | 123 | Main documentation articles |
| YAML (`.yml`) | 4 | Table of contents, FAQ, and index files |
| **Total** | 127 | Complete documentation set |

---

## Azure/acr GitHub Repository Structure

```
acr/
├── .github/                    # GitHub-specific configurations
│   ├── ISSUE_TEMPLATE/         # Issue templates for bugs, features, roadmap
│   └── workflows/              # GitHub Actions workflows
├── docs/                       # Main documentation folder
│   ├── .vuepress/              # VuePress configuration for docs site
│   ├── blog/                   # Blog posts and announcements
│   ├── custom-domain/          # Custom domain configuration (Preview)
│   ├── image-transfer/         # ACR Transfer feature (ARM templates)
│   ├── integration/            # CI/CD integrations
│   ├── media/                  # Images and diagrams
│   ├── preview/                # Preview features documentation
│   ├── tasks/                  # ACR Tasks documentation
│   └── teleport/               # Project Teleport (Artifact Streaming)
├── samples/                    # Code samples (Java, .NET Core)
├── notifications/              # Service notifications and outage information
└── README.md                   # Repository overview
```

### Preview Features Location
```
docs/preview/
├── abac-repo-permissions/      # Microsoft Entra ABAC for repository permissions
├── artifact-streaming/         # Artifact streaming for faster container startup
├── connected-registry/         # On-premises registry replica
├── continuous-patching/        # Automated OS vulnerability patching
└── quarantine/                 # Quarantine pattern for vulnerability scanning
```

---

## File Naming Conventions

### MS Learn Patterns
1. `container-registry-{topic}.md` - Core documentation
2. `container-registry-get-started-{tool}.md` - Quickstarts by tool
3. `container-registry-auth-{method}.md` - Authentication methods
4. `container-registry-tasks-{topic}.md` - ACR Tasks documentation
5. `container-registry-tutorial-{topic}.md` - Tutorials
6. `container-registry-rbac-{topic}.md` - RBAC/permissions
7. `container-registry-transfer-{topic}.md` - Transfer features
8. `container-registry-troubleshoot-{topic}.md` - Troubleshooting
9. `artifact-cache-{topic}.md` - Artifact cache features

### Azure/acr Patterns
1. `README.md` in each feature folder - Overview
2. `quickstart-*.md` - Quick start guides
3. `troubleshooting.md` - Troubleshooting guides
4. `release-notes.md` - Version history

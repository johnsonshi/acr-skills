# ACR Skills

Claude Code skills for Azure Container Registry (ACR) and container ecosystem tools.

## Installation

Add skills to your Claude Code environment using:

```bash
# Add the ACR skill
npx skills add johnsonshi/acr-skills --skill acr
```

## Available Skills

| Skill | Status | Description |
|-------|--------|-------------|
| `acr` | Available | Comprehensive Azure Container Registry knowledge - authentication, networking, geo-replication, ACR Tasks, signing, artifact cache, and all 25 feature areas |

## Work in Progress

The following ecosystem tool skills are being actively developed:

| Skill | Status | Description |
|-------|--------|-------------|
| `oras` | In Progress | ORAS CLI for pushing/pulling OCI artifacts |
| `notation` | Planned | Notation CLI for container image signing |
| `ratify` | Planned | Ratify verification engine for Kubernetes |

## Skill Coverage

### ACR Skill (25 Feature Areas)

- **Authentication & Access**: Authentication, RBAC & Authorization
- **Networking & Security**: Private Endpoints, Network Security, Data Loss Prevention, Dedicated Data Endpoints
- **High Availability**: Geo-Replication, Zone Redundancy
- **Build & Automation**: ACR Tasks, Task Agent Pools
- **Security & Trust**: Content Trust & Signing, Vulnerability Scanning, Customer-Managed Keys, Continuous Patching
- **Image & Artifact Operations**: Image Management, Artifact Cache, Artifact Streaming, OCI Artifacts
- **Edge & Air-Gapped**: Connected Registry, ACR Transfer
- **Lifecycle & Observability**: Soft Delete & Retention, Webhooks, Event Grid, Monitoring & Diagnostics, SKUs & Service Tiers

## Repository Structure

```
acr-skills/
├── skills/
│   └── acr/
│       ├── SKILL.md                      # Main skill definition
│       ├── feature-area-skill-resources/ # 25 feature-specific knowledge files
│       └── investigation-reports/        # Research documentation
└── submodules/                           # Source documentation references
    ├── azure-management-docs/            # MS Learn docs
    ├── acr/                              # Azure/acr GitHub repo
    ├── oras/                             # ORAS CLI source
    ├── notation/                         # Notation signing CLI
    └── ratify/                           # Ratify verification engine
```

## Contributing

Skills are built from comprehensive research of official documentation and source code. Each skill follows the deep-knowledge pattern with:

1. **SKILL.md** - Parent skill with frontmatter, quick reference, and feature routing
2. **feature-area-skill-resources/** - Synthesized knowledge per feature area
3. **investigation-reports/** - Raw research organized by phase

## License

MIT

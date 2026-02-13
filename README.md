# ACR Skills

Agent skills for Azure Container Registry (ACR) and container ecosystem tools.

## Installation

Add skills to your environment using:

```bash
# Add the ACR skill
npx skills add johnsonshi/acr-skills --skill acr
```

## Available Skills

| Skill | Status | Description |
|-------|--------|-------------|
| `acr` | Available | Comprehensive Azure Container Registry knowledge covering authentication, networking, geo-replication, ACR Tasks, signing, artifact cache, and more |

## Work in Progress

The following ecosystem tool skills are being actively developed:

| Skill | Status | Description |
|-------|--------|-------------|
| `oras` | In Progress | ORAS CLI for pushing/pulling OCI artifacts |
| `notation` | Planned | Notation CLI for container image signing |
| `ratify` | Planned | Ratify verification engine for Kubernetes |

## Skill Coverage

The complete list of ACR feature areas is encoded in the skill itself (`skills/acr/SKILL.md`). Refer to the skill for the authoritative and up-to-date feature coverage.

## Repository Structure

```
acr-skills/
├── skills/
│   └── acr/
│       ├── SKILL.md                      # Main skill definition
│       ├── feature-area-skill-resources/ # Feature-specific knowledge files
│       └── investigation-reports/        # Research documentation
└── submodules/                           # Source documentation references
    ├── azure-management-docs/            # MS Learn docs repo containing ACR docs
    ├── acr/                              # Azure/acr GitHub repo
    ├── oras/                             # ORAS CLI source repo
    ├── oras-www/                         # ORAS documentation website repo
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

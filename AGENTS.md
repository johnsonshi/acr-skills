# AGENTS.md

This file provides guidance to AI coding agents when working with code in this repository.

## Repository Purpose

This repository contains skills for Azure Container Registry (ACR) and related container ecosystem tools. Skills are structured knowledge bases that help AI agents answer questions about ACR features.

## Repository Structure

```
acr-skills/
├── skills/
│   └── acr/
│       ├── SKILL.md                      # Parent skill with frontmatter
│       ├── feature-area-skill-resources/ # Feature-specific skill files
│       │   └── {feature}/overview.md
│       └── investigation-reports/        # Research documentation
│           ├── repository-layout/        # Phase 1: Repo structure
│           ├── feature-overview/         # Phase 2: Feature taxonomy
│           └── feature-in-depth/         # Phase 3: Per-feature research
└── submodules/                           # Source documentation (git submodules)
    ├── azure-management-docs/            # MS Learn docs (focus: articles/container-registry/)
    ├── acr/                              # Azure/acr GitHub repo (preview features, samples)
    ├── oras/                             # ORAS CLI source
    ├── notation/                         # Notation signing CLI source
    └── ratify/                           # Ratify verification engine source
```

## Skill Structure

Skills follow the deep-knowledge-skill-creator pattern:

- **SKILL.md**: Parent skill with YAML frontmatter (`name`, `description` with triggers), quick reference, and feature routing table
- **feature-area-skill-resources/{feature}/overview.md**: Synthesized knowledge per feature area
- **investigation-reports/**: Raw research organized by phase

## ACR Feature Areas

The complete list of ACR feature areas is encoded in the skill itself (`skills/acr/SKILL.md`). Refer to the skill for the authoritative and up-to-date feature coverage.

## Working with Submodules

```bash
# Initialize submodules after clone
git submodule update --init --recursive

# Update submodules to latest
git submodule update --remote
```

## Key Documentation Paths

When researching ACR features, check both sources:

| Source | Path | Content Type |
|--------|------|--------------|
| MS Learn | `submodules/azure-management-docs/articles/container-registry/` | GA feature docs |
| Azure/acr | `submodules/acr/docs/` | Preview features, samples, ARM templates |
| Azure/acr | `submodules/acr/docs/preview/` | Connected registry, artifact streaming, continuous patching |

## Work in Progress

The following ecosystem tool skills are being actively developed:

| Skill | Status | Description |
|-------|--------|-------------|
| `oras` | In Progress | ORAS CLI for pushing/pulling OCI artifacts |
| `notation` | Planned | Notation CLI for container image signing |
| `ratify` | Planned | Ratify verification engine for Kubernetes |

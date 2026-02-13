# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Purpose

This repository contains Claude Code skills for Azure Container Registry (ACR) and related container ecosystem tools. Skills are structured knowledge bases that help Claude answer questions about ACR features.

## Repository Structure

```
acr-skills/
├── skills/
│   └── acr/
│       ├── SKILL.md                      # Parent skill with frontmatter
│       ├── feature-area-skill-resources/ # 25 feature-specific skill files
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
- **feature-area-skill-resources/{feature}/overview.md**: Synthesized knowledge per feature area (25 total)
- **investigation-reports/**: Raw research organized by phase

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
| MS Learn | `submodules/azure-management-docs/articles/container-registry/` | GA feature docs (123 files) |
| Azure/acr | `submodules/acr/docs/` | Preview features, samples, ARM templates |
| Azure/acr | `submodules/acr/docs/preview/` | Connected registry, artifact streaming, continuous patching |

## ACR Feature Areas (25)

Authentication, RBAC & Authorization, Private Endpoints, Network Security, Geo-Replication, Zone Redundancy, ACR Tasks, Task Agent Pools, Content Trust & Signing, Image Management, Artifact Cache, Artifact Streaming, Connected Registry, ACR Transfer, Continuous Patching, Customer-Managed Keys, Soft Delete & Retention, Webhooks, Event Grid, Monitoring & Diagnostics, Vulnerability Scanning, OCI Artifacts, SKUs & Tiers, Data Loss Prevention, Dedicated Data Endpoints

## Work in Progress

The following ecosystem tool skills are being actively developed:

| Skill | Status | Description |
|-------|--------|-------------|
| `oras` | In Progress | ORAS CLI for pushing/pulling OCI artifacts |
| `notation` | Planned | Notation CLI for container image signing |
| `ratify` | Planned | Ratify verification engine for Kubernetes |

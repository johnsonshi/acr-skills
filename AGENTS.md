# AGENTS.md

This file provides guidance to AI coding agents when working with code in this repository.

## Repository Purpose

This repository contains skills for Azure Container Registry (ACR) and related container ecosystem tools. Skills are structured knowledge bases that help AI agents answer questions about ACR features.

See [README.md](README.md) for installation instructions, available skills, and repository structure.

## Skill Structure

Skills follow the deep-knowledge-skill-creator pattern:

- **SKILL.md**: Parent skill with YAML frontmatter (`name`, `description` with triggers), quick reference, and feature routing table
- **feature-area-skill-resources/{feature}/overview.md**: Synthesized knowledge per feature area
- **investigation-reports/**: Raw research organized by phase

## Feature Areas

The complete list of feature areas for each skill is encoded in the skill's `SKILL.md`. Refer to the skill for the authoritative and up-to-date feature coverage.

## Working with Submodules

```bash
# Initialize submodules after clone
git submodule update --init --recursive

# Update submodules to latest
git submodule update --remote
```

## Key Documentation Paths

When researching features, check these sources:

| Source | Path | Content Type |
|--------|------|--------------|
| MS Learn | `submodules/azure-management-docs/articles/container-registry/` | ACR GA feature docs |
| Azure/acr | `submodules/acr/docs/` | Preview features, samples, ARM templates |
| Azure/acr | `submodules/acr/docs/preview/` | Connected registry, artifact streaming, continuous patching |
| ORAS | `submodules/oras/` | ORAS CLI source code |
| ORAS | `submodules/oras-www/` | ORAS documentation website |

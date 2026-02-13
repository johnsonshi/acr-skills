# AGENTS.md

This file provides guidance to AI coding agents when working with code in this repository.

## Repository Purpose

This repository contains skills for Azure Container Registry (ACR) and related container ecosystem tools. Skills are structured knowledge bases that help AI agents answer questions about ACR features.

See [README.md](README.md) for installation instructions, available skills, repository structure, and key documentation paths.

## Skill Structure

Skills follow the deep-knowledge-skill-creator pattern:

- **SKILL.md**: Parent skill with YAML frontmatter (`name`, `description`), quick reference, and feature routing table
- **SKILL.md frontmatter rule**: `description` must be a single-line string. Multi-line YAML block styles (`|` or `>`) are not supported.
- **feature-area-skill-resources/{feature}/overview.md**: Synthesized knowledge per feature area
- **investigation-reports/**: Raw research organized by phase
- **investigation-reports/repository-layout/**: Directory layout report artifacts (for example `directory-structure.md`, `key-files.md`, `content-organization.md`)

## Feature Areas

The complete list of feature areas for each skill is encoded in the skill's `SKILL.md`. Refer to the skill for the authoritative and up-to-date feature coverage.

## Working with Submodules

```bash
# Initialize submodules after clone
git submodule update --init --recursive

# Update submodules to latest
git submodule update --remote
```

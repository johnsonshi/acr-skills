# ACR Continuous Patching - Feature Overview

## Summary

Azure Container Registry (ACR) Continuous Patching is an automated feature that detects and remediates operating system (OS) level vulnerabilities in container images. It eliminates the need for manual rebuilding or waiting for upstream updates by automatically scanning images with Trivy and applying security fixes using Copa (Copacetic).

## What is Continuous Patching?

Continuous Patching is a supply chain security workflow in ACR that:

1. **Automatically scans** container images for OS-level vulnerabilities on a configurable schedule
2. **Applies patches** to fix detected vulnerabilities without requiring source code access
3. **Creates new tagged images** with the patched content
4. **Maintains traceability** through versioned tags (incremental or floating)

## Key Benefits

### Speed and Automation
- Vulnerabilities can appear daily, but image publishers may only release updates monthly
- Continuous Patching removes dependency on upstream release cycles
- Patches are applied as soon as new OS vulnerabilities are detected

### Security Without Source Code Access
- No need for access to original Dockerfiles or build pipelines
- Patches are applied directly to existing container images in your registry
- Maintains security hygiene even for third-party or vendor images

### Compliance and Governance
- Helps meet security compliance requirements through regular patching
- Provides audit trail through versioned patched images
- Integrates with existing ACR infrastructure

## Technology Stack

### Trivy
- Open-source vulnerability scanner (https://trivy.dev/)
- Scans container images for known OS-level CVEs
- Identifies vulnerable packages managed by system package managers (apt, yum, etc.)

### Copa (Copacetic)
- Open-source patching tool (https://project-copacetic.github.io/copacetic/website/)
- Applies OS-level security patches without rebuilding images
- Creates new image layers with updated packages

## How It Works

1. **Configuration**: User defines target repositories and tags in a JSON configuration file
2. **Scheduling**: Workflow runs on a configurable schedule (1-30 days)
3. **Scanning**: Trivy scans each configured image for OS vulnerabilities
4. **Patching**: Copa applies available patches for detected vulnerabilities
5. **Tagging**: New patched images are created with versioned tags

## Workflow Components

Three ACR Tasks are created to execute the workflow:

| Task | Purpose |
|------|---------|
| `cssc-trigger-workflow` | Scans configuration and triggers image scanning |
| `cssc-scan-image` | Scans images for OS vulnerabilities using Trivy |
| `cssc-patch-image` | Applies patches using Copa |

## Pricing

Continuous Patching operates on a consumption model:
- Each patch utilizes ACR Task compute resources
- Charged at default ACR Task pricing: **$0.0001/second** of task running time
- No additional licensing fees

## Preview Status

As of March 2025, Continuous Patching is in **public preview**.

## Related Features

- **Microsoft Defender for Containers**: Provides vulnerability scanning and security recommendations
- **ACR Tasks**: Underlying infrastructure for running patching workflows
- **Content Trust**: Can be combined with image signing for supply chain security

## Source Files

- MS Learn Concepts: `submodules/azure-management-docs/articles/container-registry/key-concept-continuous-patching.md`
- MS Learn How-To: `submodules/azure-management-docs/articles/container-registry/how-to-continuous-patching.md`
- ACR Repo Docs: `submodules/acr/docs/preview/continuous-patching/README.md`

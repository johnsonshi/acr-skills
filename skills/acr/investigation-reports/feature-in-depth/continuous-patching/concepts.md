# ACR Continuous Patching - Concepts

## Core Concepts

### OS-Level vs Application-Level Vulnerabilities

Continuous Patching specifically targets **OS-level vulnerabilities**:

| Type | Supported | Examples |
|------|-----------|----------|
| OS-level | Yes | Packages managed by apt, yum, apk, dnf |
| Application-level | No | Go modules, Python pip, Node.js npm |

OS-level vulnerabilities originate from system packages installed by the operating system's package manager. Application-level vulnerabilities from programming language ecosystems are not addressed by this feature.

## Tagging Conventions

Because Continuous Patching creates new images for each patch cycle, ACR uses tag conventions to version and identify patched images.

### Incremental Tagging (Default)

Each patch increments a numerical suffix on the original tag.

**Example progression:**
```
python:3.11        (original)
python:3.11-1      (first patch)
python:3.11-2      (second patch)
python:3.11-3      (third patch)
```

**Suffix Rules:**
- `-1` to `-999`: Interpreted as patch version tags
- `-x` where x > 999: Treated as part of the original tag name
  - Example: `ubuntu:jammy-20240530` is an original tag, not a patched one

**Important Considerations:**
- Avoid manually pushing tags with suffixes `-1` to `-999` for images you want patched
- If `-999` versions are reached, Continuous Patching returns an error
- Maximum 999 patch versions per original tag

**Best For:**
- Environments requiring auditability
- Rollback capability
- Strict versioning requirements
- Compliance scenarios

### Floating Tagging

A single mutable tag (`-patched`) always references the latest patched version.

**Example progression:**
```
python:3.11              (original)
python:3.11-patched      (always points to latest patch)
```

Each subsequent patch updates the `-patched` tag to reference the newest patched image.

**Best For:**
- CI/CD pipelines that need automatic updates
- Reduced complexity (no need to update downstream references)
- Scenarios where rollback is less critical

**Trade-offs:**
- No strict versioning
- Difficult to roll back to previous patch versions
- Less visibility into patch history

## Workflow Architecture

### Three-Task Model

```
┌─────────────────────────┐
│  cssc-trigger-workflow  │
│  (Configuration scan)   │
└───────────┬─────────────┘
            │ Triggers for each image
            ▼
┌─────────────────────────┐
│    cssc-scan-image      │
│    (Trivy scanning)     │
└───────────┬─────────────┘
            │ If vulnerabilities found
            ▼
┌─────────────────────────┐
│    cssc-patch-image     │
│    (Copa patching)      │
└─────────────────────────┘
```

### Configuration Storage

The JSON configuration is stored as an artifact in the registry:
- Repository: `csscpolicies/patchpolicy`
- Contains the patching configuration schema
- Automatically managed by the workflow

## Schedule Behavior

### Fixed-Day Multiplier System

The schedule parameter follows a fixed-day multiplier starting from day 1 of each month.

**Example 1: Weekly schedule (7d)**
- Command run on: 3rd of the month
- Next scheduled run: 7th (first multiple of 7 after the 3rd)
- Subsequent runs: 14th, 21st, 28th

**Example 2: Every 3 days (3d)**
- Command run on: 7th of the month
- Next scheduled run: 9th (next multiple of 3 after 7)
- Subsequent runs: 12th, 15th, 18th...

### Monthly Reset

The schedule counter resets on the 1st of every month:
- Workflow always runs on the 1st regardless of schedule
- Then follows the specified schedule for the remainder of the month

**Example:**
- Schedule: 7d
- Last January run: 28th
- Next runs: February 1st, February 8th, February 15th...

### Immediate Execution

The `--run-immediately` flag triggers an instant patch run:
- Subsequent scheduled runs remain aligned to the day multiple system
- Does not affect the overall schedule pattern

## Image Lifecycle

### Original Image
The base image as pushed or imported to the registry.

### Patched Image
A new image created by applying OS patches:
- Contains same application layers
- Updated OS package layers
- New tag per tagging convention

### Skipped Images
Images that are skipped from patching:
- No OS vulnerabilities detected
- End of Service Life (EOSL) operating systems
- Incompatible image formats

## End of Service Life (EOSL) Images

EOSL images are **not supported** by Continuous Patching:

**Definition:** Images where the underlying OS no longer receives updates, security patches, or technical support.

**Examples:**
- Debian 8
- Fedora 28
- Ubuntu 16.04 (after April 2021)
- CentOS 6

**Behavior:** EOSL images are skipped during patching despite having vulnerabilities.

**Recommendation:** Upgrade to a supported OS version.

## Vulnerability Detection

### What Gets Detected
- Known CVEs (Common Vulnerabilities and Exposures)
- Vulnerabilities with available patches from OS vendors
- Security issues in system packages

### What Gets Patched
- Only vulnerabilities with available fixes
- OS packages with updated versions in package repositories
- Security fixes that don't require application changes

## Source Files

- MS Learn Concepts: `submodules/azure-management-docs/articles/container-registry/key-concept-continuous-patching.md`
- ACR Repo Docs: `submodules/acr/docs/preview/continuous-patching/README.md`

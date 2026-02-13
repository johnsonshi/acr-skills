# ACR Continuous Patching - Supported Images

## Support Matrix

### Operating System Support

| OS Type | Supported | Notes |
|---------|-----------|-------|
| Linux-based images | Yes | Primary supported platform |
| Windows-based images | No | Not supported in preview |

### Package Manager Support

Continuous Patching works with OS-level packages managed by system package managers:

| Package Manager | OS | Supported |
|-----------------|-----|-----------|
| apt / dpkg | Debian, Ubuntu | Yes |
| yum / rpm | RHEL, CentOS, Fedora | Yes |
| apk | Alpine Linux | Yes |
| dnf | Fedora, RHEL 8+ | Yes |
| zypper | openSUSE, SLES | Yes |

### Vulnerability Types

| Vulnerability Source | Supported | Examples |
|---------------------|-----------|----------|
| OS system packages | Yes | openssl, curl, glibc |
| Application packages | No | npm modules, pip packages, Go modules |
| Language runtime | Partial | Only if installed via OS package manager |

## Supported Linux Distributions

### Actively Supported (Full Patching)

These distributions receive regular security updates and are fully supported:

| Distribution | Supported Versions |
|--------------|-------------------|
| Ubuntu | 20.04 LTS, 22.04 LTS, 24.04 LTS |
| Debian | 11 (bullseye), 12 (bookworm) |
| Alpine Linux | 3.18, 3.19, 3.20 |
| RHEL | 8, 9 |
| CentOS Stream | 8, 9 |
| Fedora | Current and previous release |
| Amazon Linux | 2, 2023 |
| Oracle Linux | 8, 9 |
| openSUSE Leap | 15.5, 15.6 |
| SLES | 15 SP4, 15 SP5 |

### End of Service Life (EOSL) - Not Supported

Images based on these operating systems are **skipped** during patching:

| Distribution | End of Life | Status |
|--------------|-------------|--------|
| Debian 8 (jessie) | June 2020 | EOSL - Skipped |
| Debian 9 (stretch) | June 2022 | EOSL - Skipped |
| Ubuntu 16.04 | April 2021 | EOSL - Skipped |
| Ubuntu 18.04 | April 2023 | EOSL - Skipped |
| CentOS 6 | November 2020 | EOSL - Skipped |
| CentOS 7 | June 2024 | EOSL - Skipped |
| CentOS 8 (non-Stream) | December 2021 | EOSL - Skipped |
| Fedora < 39 | Various | EOSL - Skipped |
| Alpine < 3.17 | Various | EOSL - Skipped |

**EOSL Image Behavior:**
- Images are identified during scanning
- Marked as "skipped" in workflow output
- No patch is attempted
- Recommendation: Upgrade to supported OS version

## Image Architecture Support

| Architecture | Supported |
|--------------|-----------|
| amd64 (x86_64) | Yes |
| arm64 | Yes (single-arch only) |
| Multi-architecture manifests | No |

### Multi-Arch Limitation

Multi-architecture images (manifest lists) are **not supported**:
- You cannot patch images that contain multiple platform variants
- Solution: Patch each architecture variant separately

## Image Format Support

| Format | Supported |
|--------|-----------|
| Docker v2 images | Yes |
| OCI images | Yes |
| Docker v1 images | No |

## What Gets Patched

### Patched Components
- System packages with known CVEs
- Packages with available security updates from OS vendor
- Dependencies of system packages

### Not Patched
- Application-level dependencies (npm, pip, go modules, etc.)
- Custom binaries not from package managers
- Statically compiled applications
- Configuration vulnerabilities

## Common Base Images

### Officially Supported Base Images

These popular base images work well with Continuous Patching:

| Image | Package Manager | Notes |
|-------|-----------------|-------|
| `ubuntu:22.04` | apt | Fully supported |
| `debian:12` | apt | Fully supported |
| `alpine:3.19` | apk | Fully supported |
| `mcr.microsoft.com/cbl-mariner/base/core:2.0` | tdnf | Azure Linux |
| `amazonlinux:2023` | dnf | Fully supported |
| `redhat/ubi9` | dnf | Fully supported |

### Microsoft Container Images

Microsoft-published images from MCR (Microsoft Container Registry) can be imported and patched:

| Image Family | Supported |
|--------------|-----------|
| .NET Runtime/SDK | Yes |
| Azure CLI | Yes |
| Azure Functions | Yes |
| CBL-Mariner/Azure Linux | Yes |

## Limitations Summary

| Limitation | Impact |
|------------|--------|
| 100 image limit | Maximum images per workflow configuration |
| Windows images | Not supported |
| Multi-arch images | Not supported |
| EOSL images | Skipped during patching |
| Application packages | Not patched |
| ABAC-enabled registries | Incompatible |
| PTC repositories | Incompatible |
| Free Trial subscriptions | Not supported |

## Recommendations

### For Best Results

1. **Use current LTS distributions** for base images
2. **Import official images** from Docker Hub or MCR before patching
3. **Avoid EOSL operating systems** - upgrade to supported versions
4. **Use single-arch images** if you need multi-platform support, patch each separately
5. **Keep application dependencies** updated through your build pipeline

### Migration Path for EOSL Images

If you have EOSL images:
1. Identify affected images using the workflow list command
2. Rebuild images with updated base OS
3. Update your configuration to target new images

## Source Files

- MS Learn Concepts: `submodules/azure-management-docs/articles/container-registry/key-concept-continuous-patching.md`
- ACR Repo Docs: `submodules/acr/docs/preview/continuous-patching/README.md`

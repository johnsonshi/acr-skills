# ACR Artifact Cache - Wildcard Support

## Overview

Wildcards in ACR Artifact Cache allow you to create flexible cache rules that match multiple repositories from upstream registries. Using wildcards reduces the number of individual cache rules needed and simplifies management of cached content.

## Wildcard Syntax

Wildcards use the asterisk character (`*`) to match multiple paths within a container image registry.

**Basic Pattern:**
```
<acr-registry>/<local-path>/* => <upstream-registry>/<upstream-path>/*
```

## Types of Wildcard Rules

### 1. Registry Level Wildcard

Caches **all repositories** from an upstream registry.

**Pattern:**
```
contoso.azurecr.io/* => mcr.microsoft.com/*
```

**Behavior:**
| Pull Request | Maps To |
|--------------|---------|
| `contoso.azurecr.io/myapp/image1` | `mcr.microsoft.com/myapp/image1` |
| `contoso.azurecr.io/myapp/image2` | `mcr.microsoft.com/myapp/image2` |
| `contoso.azurecr.io/otherapp/tool` | `mcr.microsoft.com/otherapp/tool` |

**CLI Example:**
```bash
az acr cache create \
  --registry contoso \
  --name mcr-mirror \
  --source-repo "mcr.microsoft.com/*" \
  --target-repo "*"
```

### 2. Repository Level Wildcard

Caches repositories under a **specific namespace prefix**.

**Pattern:**
```
contoso.azurecr.io/dotnet/* => mcr.microsoft.com/dotnet/*
```

**Behavior:**
| Pull Request | Maps To |
|--------------|---------|
| `contoso.azurecr.io/dotnet/sdk` | `mcr.microsoft.com/dotnet/sdk` |
| `contoso.azurecr.io/dotnet/runtime` | `mcr.microsoft.com/dotnet/runtime` |
| `contoso.azurecr.io/dotnet/aspnet` | `mcr.microsoft.com/dotnet/aspnet` |

**CLI Example:**
```bash
az acr cache create \
  --registry contoso \
  --name dotnet-images \
  --source-repo "mcr.microsoft.com/dotnet/*" \
  --target-repo "dotnet/*"
```

### 3. Multi-Registry Namespace Mapping

Different upstream registries mapped to different local namespaces.

**Patterns:**
```
contoso.azurecr.io/library/dotnet/* => mcr.microsoft.com/dotnet/*
contoso.azurecr.io/library/python/* => docker.io/library/python/*
```

**Behavior:**
| Pull Request | Maps To |
|--------------|---------|
| `contoso.azurecr.io/library/dotnet/app1` | `mcr.microsoft.com/dotnet/app1` |
| `contoso.azurecr.io/library/python/app3` | `docker.io/library/python/app3` |

**CLI Example:**
```bash
# MCR dotnet mapping
az acr cache create \
  --registry contoso \
  --name mcr-dotnet \
  --source-repo "mcr.microsoft.com/dotnet/*" \
  --target-repo "library/dotnet/*"

# Docker Hub python mapping (with credentials)
az acr cache create \
  --registry contoso \
  --name dockerhub-python \
  --source-repo "docker.io/library/python/*" \
  --target-repo "library/python/*" \
  --cred-set DockerHubCreds
```

## Wildcard Rule Constraints

### Overlapping Rules - NOT Allowed Between Wildcards

Wildcard rules **cannot overlap** with other wildcard rules targeting the same or overlapping paths.

**Example 1: Blocked - Same Registry Different Wildcards**

```
Existing: contoso.azurecr.io/* => mcr.microsoft.com/*
New:      contoso.azurecr.io/library/* => docker.io/library/*  [BLOCKED]
```

**Why blocked:** `contoso.azurecr.io/library/*` overlaps with `contoso.azurecr.io/*`

**Example 2: Blocked - Nested Paths**

```
Existing: contoso.azurecr.io/library/* => mcr.microsoft.com/library/*
New:      contoso.azurecr.io/library/dotnet/* => docker.io/library/dotnet/*  [BLOCKED]
```

**Why blocked:** `contoso.azurecr.io/library/dotnet/*` overlaps with `contoso.azurecr.io/library/*`

**Example 3: Blocked - Same Target Path Different Sources**

```
Existing: contoso.azurecr.io/* => mcr.microsoft.com/*
New:      contoso.azurecr.io/* => docker.io/library/*  [BLOCKED]
```

**Why blocked:** Same target repository path `contoso.azurecr.io/*` cannot have multiple rules.

### Static Rules CAN Overlap with Wildcards

Static (fixed) cache rules **can overlap** with wildcard rules. Static rules take precedence.

**Example 1: Allowed - Static Overlapping Wildcard**

```
Existing: contoso.azurecr.io/* => mcr.microsoft.com/*
New:      contoso.azurecr.io/library/dotnet => docker.io/library/dotnet  [ALLOWED]
```

**Why allowed:** `contoso.azurecr.io/library/dotnet` is a static path that can override the wildcard.

**Resulting Behavior:**
| Pull Request | Matching Rule |
|--------------|---------------|
| `contoso.azurecr.io/library/dotnet:8.0` | Static rule (docker.io) |
| `contoso.azurecr.io/library/nginx:latest` | Wildcard rule (mcr) |

### Static Rules Cannot Duplicate Targets

Multiple static rules **cannot target the same path**.

**Example: Blocked**

```
Existing: contoso.azurecr.io/myimage => mcr.microsoft.com/myimage
New:      contoso.azurecr.io/myimage => docker.io/library/myimage  [BLOCKED]
```

**Why blocked:** Two rules targeting exact same path `contoso.azurecr.io/myimage`.

## Rule Resolution Order

When a pull request matches multiple rules:

1. **Static rules** are evaluated first
2. **More specific wildcards** are matched before broader wildcards
3. **First matching rule** is applied

**Resolution Example:**

Given these rules:
```
Rule A (wildcard):  contoso.azurecr.io/* => mcr.microsoft.com/*
Rule B (static):    contoso.azurecr.io/nginx => docker.io/library/nginx
Rule C (wildcard):  contoso.azurecr.io/dotnet/* => mcr.microsoft.com/dotnet/*
```

| Pull Request | Matched Rule | Source |
|--------------|--------------|--------|
| `contoso.azurecr.io/nginx:latest` | Rule B | docker.io |
| `contoso.azurecr.io/dotnet/sdk:8.0` | Rule C | mcr (dotnet) |
| `contoso.azurecr.io/azure-cli:latest` | Rule A | mcr (root) |

## Common Wildcard Patterns

### Pattern 1: Full Registry Mirror

Mirror all images from a public registry:

```bash
# Mirror all of MCR
az acr cache create -r contoso -n mcr-full \
  -s "mcr.microsoft.com/*" -t "mcr/*"

# Mirror all of Quay
az acr cache create -r contoso -n quay-full \
  -s "quay.io/*" -t "quay/*"
```

### Pattern 2: Vendor-Specific Namespace

Cache specific vendor images:

```bash
# All Microsoft .NET images
az acr cache create -r contoso -n msft-dotnet \
  -s "mcr.microsoft.com/dotnet/*" -t "microsoft/dotnet/*"

# All Bitnami images from Docker Hub
az acr cache create -r contoso -n bitnami \
  -s "docker.io/bitnami/*" -t "bitnami/*" \
  -c DockerHubCreds
```

### Pattern 3: Team-Based Organization

Organize by team namespaces:

```bash
# Backend team - .NET images
az acr cache create -r contoso -n backend-dotnet \
  -s "mcr.microsoft.com/dotnet/*" -t "backend/dotnet/*"

# Frontend team - Node images
az acr cache create -r contoso -n frontend-node \
  -s "docker.io/library/node/*" -t "frontend/node/*" \
  -c DockerHubCreds

# DevOps team - Azure tools
az acr cache create -r contoso -n devops-azure \
  -s "mcr.microsoft.com/azure-cli" -t "devops/azure-cli"
```

### Pattern 4: Base Images with Overrides

Use wildcards for general coverage with static overrides:

```bash
# General MCR mirror
az acr cache create -r contoso -n mcr-general \
  -s "mcr.microsoft.com/*" -t "*"

# Override specific image to use Docker Hub version
az acr cache create -r contoso -n nginx-override \
  -s "docker.io/library/nginx" -t "nginx" \
  -c DockerHubCreds
```

## Managing Wildcard Rules

### List All Cache Rules

```bash
az acr cache list --registry contoso --output table
```

### Check for Overlapping Rules Before Creation

Before creating a new wildcard rule, list existing rules and verify no overlaps:

```bash
# List existing rules
az acr cache list --registry contoso --query "[].{name:name, source:sourceRepository, target:targetRepository}"
```

### Delete Conflicting Rules

If you need to change wildcard scope, delete existing rules first:

```bash
# Delete existing broad wildcard
az acr cache delete --registry contoso --name mcr-full

# Create new more specific wildcards
az acr cache create -r contoso -n mcr-dotnet \
  -s "mcr.microsoft.com/dotnet/*" -t "dotnet/*"

az acr cache create -r contoso -n mcr-azure \
  -s "mcr.microsoft.com/azure-*" -t "azure/*"
```

## Wildcard Best Practices

### 1. Start Specific, Expand Carefully

Begin with specific rules and only use broad wildcards when needed:

```bash
# Start with specific images you actually use
az acr cache create -r contoso -n dotnet-sdk -s "mcr.microsoft.com/dotnet/sdk" -t "dotnet/sdk"
az acr cache create -r contoso -n dotnet-runtime -s "mcr.microsoft.com/dotnet/runtime" -t "dotnet/runtime"

# Expand to wildcard only if you need many more
az acr cache create -r contoso -n dotnet-all -s "mcr.microsoft.com/dotnet/*" -t "dotnet/*"
```

### 2. Document Wildcard Rules

Maintain documentation of your wildcard strategy:

```markdown
| Rule Name | Pattern | Purpose |
|-----------|---------|---------|
| mcr-dotnet | mcr.microsoft.com/dotnet/* | All .NET runtime images |
| quay-coreos | quay.io/coreos/* | CoreOS container tooling |
| nginx-override | docker.io/library/nginx | Specific nginx version |
```

### 3. Plan Namespace Hierarchy

Design your target namespace structure before creating rules:

```
ACR Structure:
├── microsoft/
│   ├── dotnet/
│   └── azure/
├── docker/
│   └── library/
└── vendor/
    ├── bitnami/
    └── hashicorp/
```

### 4. Use Static Rules for Critical Images

Override wildcards with static rules for images requiring specific control:

```bash
# Wildcard for general coverage
az acr cache create -r contoso -n general \
  -s "docker.io/library/*" -t "base/*" -c DockerCreds

# Static override for production nginx (pin to specific source)
az acr cache create -r contoso -n nginx-prod \
  -s "docker.io/library/nginx" -t "base/nginx" -c DockerCreds
```

### 5. Consider Cache Rule Limits

With a limit of 1,000 rules, wildcards help conserve:
- One wildcard can replace hundreds of static rules
- Balance between granularity and rule count
- Monitor rule usage and consolidate when appropriate

## Troubleshooting Wildcard Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| "Cache rule conflicts with existing rule" | Overlapping wildcard patterns | Delete conflicting rule or use static override |
| Wrong image pulled | Static rule taking precedence | Review rule resolution order |
| Image not found | Wildcard doesn't match path | Verify wildcard pattern matches upstream structure |
| Can't create broader wildcard | Narrower wildcard exists | Delete narrow wildcard first |

## References

- [Wildcard Support Documentation](https://learn.microsoft.com/azure/container-registry/wildcards-artifact-cache)
- [Artifact Cache Overview](https://learn.microsoft.com/azure/container-registry/artifact-cache-overview)
- [Troubleshoot Artifact Cache](https://learn.microsoft.com/azure/container-registry/troubleshoot-artifact-cache)

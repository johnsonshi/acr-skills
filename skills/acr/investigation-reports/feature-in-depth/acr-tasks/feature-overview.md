# ACR Tasks - Feature Overview

## What is ACR Tasks?

Azure Container Registry Tasks (ACR Tasks) is a suite of features within Azure Container Registry that provides cloud-based container image building, testing, and patching capabilities. It enables automated container lifecycle management without requiring local Docker installations.

## Core Capabilities

### 1. Cloud-Based Container Building
- Build container images in Azure's cloud infrastructure
- Support for multiple platforms: Linux, Windows, ARM architectures
- No local Docker Engine required
- Uses familiar `docker build` syntax

### 2. Automated Triggers
ACR Tasks supports three types of automatic triggers:
- **Source Code Updates**: Trigger builds on git commits or pull requests
- **Base Image Updates**: Automatically rebuild when base images are updated
- **Scheduled Triggers**: Run tasks on defined schedules using cron expressions

### 3. Multi-Step Workflows
- Define complex build, test, and deploy workflows using YAML
- Chain multiple containers together with dependencies
- Run containers as commands with parameters
- Support parallel and sequential execution

## Task Scenarios

### Quick Tasks
On-demand, single-image builds using `az acr build`:
- Offload builds to the cloud
- Build and push without local Docker
- Validate Dockerfiles before committing

### Automatically Triggered Tasks
Tasks that respond to events:
- Git commits (GitHub, Azure DevOps)
- Pull requests
- Base image updates in same or different registry
- Timer/cron schedules

### Multi-Step Tasks
Complex workflows defined in YAML:
- Build multiple images in series or parallel
- Run unit and functional tests
- Deploy using Helm or other tools
- Capture test results

## Key Features

### Context Locations
ACR Tasks supports building from various sources:
| Context Type | Example |
|--------------|---------|
| Local file system | `/home/user/projects/myapp` |
| GitHub main branch | `https://github.com/user/repo.git` |
| GitHub branch | `https://github.com/user/repo.git#mybranch` |
| GitHub subfolder | `https://github.com/user/repo.git#branch:folder` |
| GitHub commit | `https://github.com/user/repo.git#commit-sha:folder` |
| Azure DevOps | `https://dev.azure.com/user/project/_git/repo#branch:folder` |
| Remote tarball | `http://server/app.tar.gz` |
| OCI artifact | `oci://registry.azurecr.io/artifact:tag` |

### Platform Support
| OS | Architectures |
|----|---------------|
| Linux | AMD64, ARM, ARM64, 386 |
| Windows | AMD64 |

Specify with `--platform` flag (e.g., `--platform Linux/arm64/v8`)

### Run Variables
Built-in variables for task definitions:
- `$Registry` / `{{.Run.Registry}}` - Registry server name
- `$ID` / `{{.Run.ID}}` - Unique run ID
- `$Date` / `{{.Run.Date}}` - Current UTC date
- `$Commit` / `{{.Run.Commit}}` - Git commit SHA
- `$Branch` / `{{.Run.Branch}}` - Git branch name

### Image Aliases
Pre-configured image aliases for common tools:
- `acr` - ACR CLI
- `az` - Azure CLI
- `bash` - Bash shell
- `curl` - curl utility

## Service Tiers

ACR Tasks is available in all Azure Container Registry tiers:
- **Basic**: Limited task execution
- **Standard**: Standard task capabilities
- **Premium**: Full features including agent pools, VNet support

## Agent Pools (Premium)

Dedicated compute pools for task execution:
- **VNet Support**: Access resources in virtual networks
- **Scalable**: Adjust instance count based on workload
- **Flexible Tiers**: S1 (2 CPU), S2 (4 CPU), S3 (8 CPU), I6 (64 CPU isolated)
- **Azure Managed**: Automatically patched and maintained

## Authentication Options

### Managed Identities
- **System-assigned**: Automatic identity tied to task lifecycle
- **User-assigned**: Persistent identity across multiple resources

### Service Principals
- Role-based access control
- Scoped permissions

### Key Vault Integration
- Secure credential storage
- Access secrets during task execution

## Task Output and Logging

- **Streamed Logs**: Real-time output for manual runs
- **Stored Logs**: Retained for 30 days by default
- **Portal Access**: View logs in Azure portal
- **CLI Access**: `az acr task logs --run-id <id>`

## Pricing Considerations

- Tasks are billed based on compute time
- Agent pools have dedicated billing based on allocation
- Premium tier required for agent pools
- Temporary pause from Azure free credits (check current status)

## Integration Points

### CI/CD Integration
- Azure DevOps pipelines
- GitHub Actions
- Jenkins
- Any system that can call Azure CLI

### Azure Resource Manager Templates
- Deploy registries with immediate task queuing
- Automate task creation with ARM/Bicep

### Event Grid
- Trigger external workflows from task events
- Integration with Azure Functions, Logic Apps

## Security Considerations

- Command-line data may be logged in diagnostics
- Avoid sensitive data in URIs or commands
- Use Key Vault for secrets
- Use managed identities instead of credentials where possible

## Related Documentation

- Quick Tasks: [quick-tasks.md](./quick-tasks.md)
- Multi-Step Tasks: [multi-step-tasks.md](./multi-step-tasks.md)
- YAML Reference: [yaml-reference.md](./yaml-reference.md)
- Triggers: [triggers.md](./triggers.md)
- Authentication: [authentication.md](./authentication.md)
- CLI Commands: [cli-commands.md](./cli-commands.md)
- Samples: [samples.md](./samples.md)

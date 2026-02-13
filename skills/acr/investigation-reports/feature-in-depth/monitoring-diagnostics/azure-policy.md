# Azure Policy for ACR

## Overview

Azure Policy is a governance service that enables you to create, assign, and manage policy definitions that enforce rules and effects over your Azure Container Registry resources. Built-in policies help audit and ensure compliance with corporate standards and service level agreements.

**Note:** There is no charge for using Azure Policy.

## Built-in Policy Definitions

Azure provides built-in policy definitions specific to Azure Container Registry. These cover security, networking, encryption, and access control requirements.

### Network Security Policies

| Policy | Effect | Description |
|--------|--------|-------------|
| Container registries should not allow unrestricted network access | Audit, Deny | Ensures network access is restricted |
| Container registries should use private link | Audit, Deny | Requires private endpoint configuration |
| Container registries should disable public network access | Audit, Deny | Enforces private-only access |

### Encryption Policies

| Policy | Effect | Description |
|--------|--------|-------------|
| Container registries should be encrypted with a customer-managed key | Audit, Deny | Requires CMK encryption |

### Authentication & Access Policies

| Policy | Effect | Description |
|--------|--------|-------------|
| Container registries should have local admin account disabled | Audit, Deny | Enforces AAD-only authentication |
| Container registries should have anonymous pull disabled | Audit, Deny | Prevents unauthenticated access |

### Image Security Policies

| Policy | Effect | Description |
|--------|--------|-------------|
| Container images should have vulnerability findings resolved | Audit | Requires vulnerability scan compliance |
| Container registries should have SKU that supports Private Links | Audit, Deny | Ensures Premium SKU for private endpoints |

## Creating Policy Assignments

### Azure Portal

1. Navigate to **Azure Policy** service
2. Select **Assignments** > **Assign policy**
3. Choose scope (subscription, resource group, or management group)
4. Select policy definition
5. Configure parameters and effects
6. Review and create

### Azure CLI

List container registry policies:
```bash
az policy assignment list \
  --query "[?contains(displayName,'Container Registries')].{name:displayName, ID:id}" \
  --output table
```

Create policy assignment:
```bash
az policy assignment create \
  --name "require-acr-private-link" \
  --display-name "Require ACR Private Link" \
  --policy "/providers/Microsoft.Authorization/policyDefinitions/<policy-id>" \
  --scope "/subscriptions/<subscription-id>" \
  --params '{"effect": {"value": "Audit"}}'
```

### ARM Template

```json
{
  "type": "Microsoft.Authorization/policyAssignments",
  "apiVersion": "2021-06-01",
  "name": "acr-require-cmk",
  "properties": {
    "displayName": "Require CMK for Container Registries",
    "policyDefinitionId": "/providers/Microsoft.Authorization/policyDefinitions/<definition-id>",
    "scope": "[subscription().id]",
    "parameters": {
      "effect": {
        "value": "Audit"
      }
    }
  }
}
```

## Policy Assignment Scope

Policies can be scoped at multiple levels:

| Scope Level | Description |
|-------------|-------------|
| Management Group | All subscriptions in group |
| Subscription | All resources in subscription |
| Resource Group | All resources in group |
| Resource | Individual registry |

## Policy Enforcement Modes

| Mode | Description |
|------|-------------|
| Enabled | Policy actively enforces compliance |
| Disabled | Policy evaluates but doesn't enforce |

Enable or disable enforcement at any time.

## Reviewing Policy Compliance

### Azure Portal

1. Navigate to **Policy** service
2. Select **Compliance**
3. Filter by state or search for policies
4. Select policy to view:
   - Aggregate compliance
   - Non-compliant resources
   - Events and changes

### Azure CLI

Get compliance state for all resources under a policy:
```bash
az policy state list --resource <policyID>
```

Get compliance for specific registry:
```bash
az policy state list \
  --resource myregistry \
  --namespace Microsoft.ContainerRegistry \
  --resource-type registries \
  --resource-group myresourcegroup
```

### Compliance States

| State | Description |
|-------|-------------|
| Compliant | Resource meets policy requirements |
| NonCompliant | Resource violates policy |
| Exempt | Resource explicitly excluded |
| Conflicting | Multiple conflicting policies |
| NotStarted | Evaluation pending |

## Regulatory Compliance

Azure Policy provides built-in initiatives for regulatory compliance frameworks:

### Supported Standards

| Standard | Description |
|----------|-------------|
| Azure Security Benchmark | Microsoft security best practices |
| CIS Azure Foundations | Center for Internet Security benchmarks |
| NIST SP 800-53 | US federal security controls |
| PCI-DSS | Payment card industry standards |
| ISO 27001 | Information security management |
| SOC 2 | Service organization controls |
| HIPAA HITRUST | Healthcare data protection |

### Security Controls for ACR

Common compliance domains:
- **Network Security**: Private endpoints, firewall rules
- **Data Protection**: Encryption at rest (CMK)
- **Identity & Access**: Authentication requirements
- **Logging & Monitoring**: Diagnostic settings
- **Vulnerability Management**: Image scanning

## Policy Effects

| Effect | Description |
|--------|-------------|
| Audit | Log non-compliance without blocking |
| AuditIfNotExists | Audit if related resource missing |
| Deny | Block non-compliant operations |
| DenyAction | Block specific actions |
| DeployIfNotExists | Deploy required resources |
| Disabled | Policy not enforced |
| Modify | Add or update tags/properties |

## Evaluation Triggers

Policies are evaluated:
- When assignment is created or updated
- When resources are created or modified
- On regular compliance scans (every 24 hours)
- On-demand evaluation request

**Note:** After creating/updating a policy, allow time for evaluation to complete.

## Determining Non-Compliance

When a resource is non-compliant:

1. Check the policy definition for requirements
2. Review resource configuration
3. Identify the specific violation
4. View remediation guidance

### Common Non-Compliance Reasons

| Policy | Common Cause | Fix |
|--------|--------------|-----|
| Require private link | No private endpoint | Configure private endpoint |
| Disable public access | Public endpoint enabled | Disable public network access |
| Require CMK | Platform-managed encryption | Configure customer-managed key |
| Disable admin | Admin account enabled | Disable admin account |

## Custom Policy Definitions

Create custom policies for organization-specific requirements:

```json
{
  "mode": "Indexed",
  "policyRule": {
    "if": {
      "allOf": [
        {
          "field": "type",
          "equals": "Microsoft.ContainerRegistry/registries"
        },
        {
          "field": "Microsoft.ContainerRegistry/registries/sku.name",
          "notEquals": "Premium"
        }
      ]
    },
    "then": {
      "effect": "deny"
    }
  },
  "parameters": {}
}
```

## Policy Exemptions

Exempt specific resources from policy evaluation:

```bash
az policy exemption create \
  --name "dev-registry-exemption" \
  --policy-assignment "<assignment-id>" \
  --exemption-category "Waiver" \
  --scope "/subscriptions/<sub>/resourceGroups/<rg>/providers/Microsoft.ContainerRegistry/registries/<registry>"
```

Exemption categories:
- **Waiver**: Intentional exception
- **Mitigated**: Alternative control in place

## Best Practices

1. **Start with Audit mode**: Understand impact before enforcing
2. **Use initiatives**: Group related policies together
3. **Assign at appropriate scope**: Match business requirements
4. **Review compliance regularly**: Address violations promptly
5. **Document exemptions**: Track why resources are exempt
6. **Use tags for organization**: Filter and report by tag
7. **Integrate with CI/CD**: Check compliance before deployment

## Integration with ACR Features

| ACR Feature | Related Policy |
|-------------|----------------|
| Private Endpoints | Require private link |
| CMK Encryption | Require CMK |
| Anonymous Pull | Disable anonymous pull |
| Admin Account | Disable local admin |
| Network Rules | Require network restrictions |
| Premium SKU | Require specific SKU |

## Source References

- `/submodules/azure-management-docs/articles/container-registry/container-registry-azure-policy.md`
- `/submodules/azure-management-docs/articles/container-registry/policy-reference.md`
- `/submodules/azure-management-docs/articles/container-registry/security-controls-policy.md`

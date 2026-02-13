# Signature Verification with Ratify on AKS

## Overview

Ratify is a CNCF sandbox project that provides a verification engine for container images and OCI artifacts on Kubernetes. It integrates with Azure Policy to enforce signature verification policies on Azure Kubernetes Service (AKS) clusters, preventing deployment of unsigned or untrusted images.

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                         AKS Cluster                              │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────────────┐  │
│  │  Workload   │───>│  Gatekeeper │───>│      Ratify         │  │
│  │ Deployment  │    │   (Policy)  │    │ (Verification Engine)│  │
│  └─────────────┘    └─────────────┘    └──────────┬──────────┘  │
│                                                    │             │
│                                         ┌─────────▼─────────┐   │
│                                         │  Azure Container  │   │
│                                         │     Registry      │   │
│                                         └─────────┬─────────┘   │
│                                                    │             │
│                                         ┌─────────▼─────────┐   │
│                                         │   Azure Key Vault │   │
│                                         │  (Certificates)   │   │
│                                         └───────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

## Use Cases

### 1. Key Vault Certificate Management
- Image producer signs images using Notation with Key Vault
- Ratify verifies signatures against trusted certificates from Key Vault
- Policy enforcement: Deny or Audit unsigned images

### 2. Artifact Signing Service
- Image producer signs images using Notation with Artifact Signing
- Short-lived certificates with zero-touch management
- Timestamping enables trust beyond certificate validity

## Prerequisites

- AKS cluster with OIDC issuer enabled
- Azure Container Registry connected to AKS
- Azure Policy add-on enabled on AKS
- User-assigned managed identity
- Helm and kubectl installed

### Enable Azure Policy Add-on

```bash
# Check if enabled
az aks show --resource-group $AKS_RG --name $AKS_NAME --query addonProfiles.azurePolicy

# Enable if needed
az aks enable-addons --addons azure-policy --name $AKS_NAME --resource-group $AKS_RG
```

## Setting Up Ratify

### Step 1: Create Managed Identity

```bash
# Create identity
az identity create -g $IDENTITY_RG -n $IDENTITY_NAME

# Get identity details
export IDENTITY_CLIENT_ID=$(az identity show --name $IDENTITY_NAME --resource-group $IDENTITY_RG --query 'clientId' -o tsv)
export IDENTITY_OBJECT_ID=$(az identity show --name $IDENTITY_NAME --resource-group $IDENTITY_RG --query 'principalId' -o tsv)
```

### Step 2: Create Federated Identity Credential

```bash
export AKS_OIDC_ISSUER=$(az aks show -n $AKS_NAME -g $AKS_RG --query "oidcIssuerProfile.issuerUrl" -otsv)
export RATIFY_NAMESPACE="gatekeeper-system"
export RATIFY_SA_NAME="ratify-admin"

az identity federated-credential create \
    --name ratify-federated-credential \
    --identity-name "$IDENTITY_NAME" \
    --resource-group "$IDENTITY_RG" \
    --issuer "$AKS_OIDC_ISSUER" \
    --subject system:serviceaccount:"$RATIFY_NAMESPACE":"$RATIFY_SA_NAME"
```

### Step 3: Grant Container Registry Access

```bash
az role assignment create \
    --role acrpull \
    --assignee-object-id ${IDENTITY_OBJECT_ID} \
    --scope subscriptions/${ACR_SUB}/resourceGroups/${ACR_RG}/providers/Microsoft.ContainerRegistry/registries/${ACR_NAME}
```

### Step 4: Grant Key Vault Access (if using Key Vault)

```bash
az role assignment create \
    --role "Key Vault Secrets User" \
    --assignee ${IDENTITY_OBJECT_ID} \
    --scope "/subscriptions/${AKV_SUB}/resourceGroups/${AKV_RG}/providers/Microsoft.KeyVault/vaults/${AKV_NAME}"
```

## Installing Ratify with Helm

### For Key Vault Certificate Management

```bash
export CHART_VER="1.15.0"
export REPO_URI="$ACR_NAME.azurecr.io/<namespace>/<repo>"
export SUBJECT="<Subject-of-signing-certificate>"
export AKV_TENANT_ID="$(az account show --query tenantId --output tsv)"

helm repo add ratify https://notaryproject.github.io/ratify
helm repo update

helm install ratify ratify/ratify --atomic \
    --namespace $RATIFY_NAMESPACE --create-namespace \
    --version $CHART_VER \
    --set provider.enableMutation=false \
    --set featureFlags.RATIFY_CERT_ROTATION=true \
    --set azureWorkloadIdentity.clientId=$IDENTITY_CLIENT_ID \
    --set oras.authProviders.azureWorkloadIdentityEnabled=true \
    --set azurekeyvault.enabled=true \
    --set azurekeyvault.vaultURI="https://$AKV_NAME.vault.azure.net" \
    --set azurekeyvault.certificates[0].name="$CERT_NAME" \
    --set azurekeyvault.tenantId="$AKV_TENANT_ID" \
    --set notation.trustPolicies[0].registryScopes[0]="$REPO_URI" \
    --set notation.trustPolicies[0].trustStores[0]="ca:azurekeyvault" \
    --set notation.trustPolicies[0].trustedIdentities[0]="x509.subject: $SUBJECT"
```

### For Artifact Signing

```bash
# Download and convert certificates to PEM format
export TS_ROOT_CERT_URL="https://www.microsoft.com/pkiops/certs/Microsoft%20Enterprise%20Identity%20Verification%20Root%20Certificate%20Authority%202020.crt"
export TSA_ROOT_CERT_URL="http://www.microsoft.com/pkiops/certs/microsoft%20identity%20verification%20root%20certificate%20authority%202020.crt"

curl -o msft-identity-verification-root-cert-2020.crt $TS_ROOT_CERT_URL
openssl x509 -in msft-identity-verification-root-cert-2020.crt -out msft-identity-verification-root-cert-2020.pem -outform PEM

curl -o msft-tsa-root-certificate-authority-2020.crt $TSA_ROOT_CERT_URL
openssl x509 -in msft-tsa-root-certificate-authority-2020.crt -out msft-tsa-root-certificate-authority-2020.pem -outform PEM

# Install Ratify
export TS_ROOT_CERT_FILEPATH="msft-identity-verification-root-cert-2020.pem"
export TSA_ROOT_CERT_FILEPATH="msft-tsa-root-certificate-authority-2020.pem"

helm install ratify ratify/ratify --atomic \
    --namespace $RATIFY_NAMESPACE --create-namespace \
    --version $CHART_VER \
    --set provider.enableMutation=false \
    --set featureFlags.RATIFY_CERT_ROTATION=true \
    --set azureWorkloadIdentity.clientId=$IDENTITY_CLIENT_ID \
    --set oras.authProviders.azureWorkloadIdentityEnabled=true \
    --set-file notationCerts[0]=$TS_ROOT_CERT_FILEPATH \
    --set-file notationCerts[1]=$TSA_ROOT_CERT_FILEPATH \
    --set notation.trustPolicies[0].registryScopes[0]="$REPO_URI" \
    --set notation.trustPolicies[0].trustStores[0]="ca:notationCerts[0]" \
    --set notation.trustPolicies[0].trustStores[1]="tsa:notationCerts[1]" \
    --set notation.trustPolicies[0].trustedIdentities[0]="x509.subject: $SUBJECT"
```

## Azure Policy Configuration

### Create Custom Policy

```bash
export CUSTOM_POLICY=$(curl -L https://raw.githubusercontent.com/notaryproject/ratify/refs/tags/v1.4.0/library/default/customazurepolicy.json)
export DEFINITION_NAME="ratify-default-custom-policy"
export DEFINITION_ID=$(az policy definition create \
    --name "$DEFINITION_NAME" \
    --rules "$(echo "$CUSTOM_POLICY" | jq .policyRule)" \
    --params "$(echo "$CUSTOM_POLICY" | jq .parameters)" \
    --mode "Microsoft.Kubernetes.Data" \
    --query id -o tsv)
```

### Assign Policy with Deny Effect

```bash
export POLICY_SCOPE=$(az aks show -g "$AKS_RG" -n "$AKS_NAME" --query id -o tsv)
az policy assignment create \
    --policy "$DEFINITION_ID" \
    --name "$DEFINITION_NAME" \
    --scope "$POLICY_SCOPE"
```

### Assign Policy with Audit Effect

```bash
az policy assignment create \
    --policy "$DEFINITION_ID" \
    --name "$DEFINITION_NAME" \
    --scope "$POLICY_SCOPE" \
    -p "{\"effect\": {\"value\":\"Audit\"}}"
```

### Verify Policy Status

```bash
# Check constraint template
kubectl get constraintTemplate ratifyverification

# Expected output:
# NAME                 AGE
# ratifyverification   11m
```

## Policy Effects

| Effect | Behavior |
|--------|----------|
| **Deny** | Block deployment of unsigned or untrusted images |
| **Audit** | Allow deployment but mark as non-compliant |

## Testing Verification

### Deploy Signed Image (Should Succeed)

```bash
kubectl run demo-signed --image=$IMAGE_SIGNED

# Output: pod/demo-signed created
```

### Deploy Unsigned Image (Should Fail with Deny)

```bash
kubectl run demo-unsigned --image=$IMAGE_UNSIGNED

# Output:
# Error from server (Forbidden): admission webhook "validation.gatekeeper.sh"
# denied the request: [azurepolicy-ratifyverification-xxx]
# Subject failed verification: $IMAGE_UNSIGNED
```

### Check Ratify Logs

```bash
kubectl logs <ratify-pod> -n $RATIFY_NAMESPACE
# Search for: verification response for subject $IMAGE
# Check errorReason field for denial reasons
```

## Advanced Configuration

### Multiple Repositories

```bash
--set notation.trustPolicies[0].registryScopes[0]="$REPO_URI_1" \
--set notation.trustPolicies[0].registryScopes[1]="$REPO_URI_2"
```

### Multiple Trusted Identities

```bash
--set notation.trustPolicies[0].trustedIdentities[0]="x509.subject: $SUBJECT_1" \
--set notation.trustPolicies[0].trustedIdentities[1]="x509.subject: $SUBJECT_2"
```

### Timestamping Support (Key Vault)

```bash
--set-file notationCerts[0]="$TSA_ROOT_CERT_FILE" \
--set notation.trustPolicies[0].trustStores[1]="tsa:notationCerts[0]"
```

## Alternative: AKS Image Integrity (Preview)

For a managed experience without deploying Ratify directly:
- Use [AKS Image Integrity policy (preview)](https://docs.microsoft.com/azure/aks/image-integrity)
- Built-in Azure Policy integration
- Simplified configuration

## Updating Configuration

Ratify configurations are Kubernetes custom resources:

```bash
# Update Key Vault settings
kubectl apply -f keymanagementprovider.yaml

# Update trust policies
kubectl apply -f verifier.yaml

# Update registry access
kubectl apply -f store.yaml
```

## Without Azure Policy (OPA Gatekeeper)

If Azure Policy is disabled:

```bash
# Install Gatekeeper
helm repo add gatekeeper https://open-policy-agent.github.io/gatekeeper/charts
helm install gatekeeper/gatekeeper \
    --name-template=gatekeeper \
    --namespace gatekeeper-system --create-namespace \
    --set enableExternalData=true \
    --set validatingWebhookTimeoutSeconds=5 \
    --set mutatingWebhookTimeoutSeconds=2 \
    --set externaldataProviderResponseCacheTTL=10s

# Install Ratify (same as above, then apply policies)
kubectl apply -f https://notaryproject.github.io/ratify/library/default/template.yaml
kubectl apply -f https://notaryproject.github.io/ratify/library/default/samples/constraint.yaml
```

## Cleanup

```bash
# Uninstall Ratify
helm delete ratify --namespace $RATIFY_NAMESPACE

# Delete CRDs
kubectl delete crd stores.config.ratify.deislabs.io \
    verifiers.config.ratify.deislabs.io \
    certificatestores.config.ratify.deislabs.io \
    policies.config.ratify.deislabs.io \
    keymanagementproviders.config.ratify.deislabs.io \
    namespacedkeymanagementproviders.config.ratify.deislabs.io \
    namespacedpolicies.config.ratify.deislabs.io \
    namespacedstores.config.ratify.deislabs.io \
    namespacedverifiers.config.ratify.deislabs.io

# Delete policy assignment
az policy assignment delete --name "$DEFINITION_NAME" --scope "$POLICY_SCOPE"
az policy definition delete --name "$DEFINITION_NAME"
```

## Troubleshooting

### Image Not Linked to Trust Policy

Images outside configured `registryScopes` will fail verification. Add additional scopes:
```bash
--set notation.trustPolicies[0].registryScopes[1]="$ADDITIONAL_REPO_URI"
```

### Certificate Access Issues

Verify managed identity has proper roles:
```bash
# For Key Vault
az role assignment list --assignee $IDENTITY_OBJECT_ID --scope "/subscriptions/$AKV_SUB/resourceGroups/$AKV_RG/providers/Microsoft.KeyVault/vaults/$AKV_NAME"
```

### Policy Not Taking Effect

Azure Policy assignment takes ~15 minutes to propagate:
```bash
kubectl get constraintTemplate ratifyverification
```

## Source Files Referenced

- `/submodules/azure-management-docs/articles/container-registry/container-registry-tutorial-verify-with-ratify-aks.md`
- `/submodules/azure-management-docs/articles/container-registry/overview-sign-verify-artifacts.md`

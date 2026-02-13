# Azure Container Registry Authentication - Integration Patterns

## Azure Kubernetes Service (AKS) Integration

### Pattern 1: AKS Managed Identity (Same Tenant)

**Description**: AKS cluster uses its kubelet managed identity to authenticate with ACR.

**Prerequisites**:
- AKS cluster and ACR in the same Microsoft Entra tenant
- Can be different Azure subscriptions
- AKS must be configured with managed identity (not service principal)

**Configuration Steps**:

1. **Attach ACR to AKS cluster**:
```bash
az aks update --name <aks-cluster> --resource-group <resource-group> --attach-acr <acr-name>
```

2. **Or assign role manually**:
```bash
# Get AKS kubelet identity
kubeletIdentityId=$(az aks show --name <aks-cluster> --resource-group <resource-group> --query identityProfile.kubeletidentity.clientId -o tsv)

# Get ACR resource ID
acrId=$(az acr show --name <acr-name> --query id -o tsv)

# Assign AcrPull role (non-ABAC registry)
az role assignment create --assignee $kubeletIdentityId --scope $acrId --role AcrPull

# Or for ABAC-enabled registry
az role assignment create --assignee $kubeletIdentityId --scope $acrId --role "Container Registry Repository Reader"
```

**Pod Configuration**:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-app
spec:
  containers:
  - name: app
    image: <acr-name>.azurecr.io/myimage:latest
    # No imagePullSecrets needed - AKS handles authentication
```

### Pattern 2: AKS Cross-Tenant Authentication

**Description**: AKS in Tenant A pulls from ACR in Tenant B using multitenant service principal.

**Prerequisites**:
- AKS cluster configured with service principal authentication
- Contributor role on AKS subscription
- Role Based Access Control Administrator on ACR

**Configuration Steps**:

1. **Create multitenant app in Tenant A**:
   - Register new app in Microsoft Entra ID
   - Set "Supported account types" to "Accounts in any organizational directory"
   - Set redirect URI to `https://www.microsoft.com` (Web platform)
   - Note the Application (client) ID

2. **Create client secret**:
   - Navigate to Certificates & secrets
   - Add new client secret
   - Save the secret value

3. **Provision app in Tenant B**:
   - Access consent URL with Tenant B admin account:
   ```
   https://login.microsoftonline.com/<TenantB-ID>/oauth2/authorize?client_id=<AppID>&response_type=code&redirect_uri=https://www.microsoft.com
   ```
   - Consent on behalf of organization

4. **Assign role in Tenant B**:
```bash
# In Tenant B context
az role assignment create \
    --assignee <app-client-id> \
    --scope <acr-resource-id> \
    --role "Container Registry Repository Reader"  # For ABAC registries
```

5. **Update AKS service principal**:
```bash
az aks update-credentials \
    --name <aks-cluster> \
    --resource-group <resource-group> \
    --reset-service-principal \
    --client-secret <new-client-secret> \
    --service-principal <app-client-id>
```

### Pattern 3: Non-AKS Kubernetes with Pull Secrets

**Description**: Any Kubernetes cluster authenticates to ACR using imagePullSecrets.

**Configuration Steps**:

1. **Create service principal**:
```bash
az ad sp create-for-rbac \
    --name <sp-name> \
    --scopes <acr-resource-id> \
    --role AcrPull
```

2. **Create Kubernetes secret**:
```bash
kubectl create secret docker-registry acr-secret \
    --namespace <namespace> \
    --docker-server=<registry>.azurecr.io \
    --docker-username=<service-principal-id> \
    --docker-password=<service-principal-password>
```

3. **Reference in Pod**:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-app
spec:
  containers:
  - name: app
    image: <registry>.azurecr.io/myimage:latest
    imagePullPolicy: IfNotPresent
  imagePullSecrets:
  - name: acr-secret
```

4. **Reference in ServiceAccount** (applies to all pods using the account):
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-service-account
imagePullSecrets:
- name: acr-secret
```

---

## Azure Container Instances (ACI) Integration

### Pattern 1: Service Principal Authentication

**Description**: ACI uses service principal credentials to pull images.

```bash
az container create \
    --resource-group <resource-group> \
    --name <container-name> \
    --image <registry>.azurecr.io/myimage:v1 \
    --registry-login-server <registry>.azurecr.io \
    --registry-username <service-principal-id> \
    --registry-password <service-principal-password> \
    --cpu 1 \
    --memory 1.5
```

### Pattern 2: Managed Identity Authentication

**Description**: ACI uses system or user-assigned managed identity.

1. **Create user-assigned identity**:
```bash
az identity create --resource-group <resource-group> --name <identity-name>
```

2. **Assign ACR role**:
```bash
identityPrincipalId=$(az identity show --resource-group <resource-group> --name <identity-name> --query principalId -o tsv)
acrId=$(az acr show --name <registry> --query id -o tsv)

az role assignment create --assignee $identityPrincipalId --scope $acrId --role AcrPull
```

3. **Create container with managed identity**:
```bash
identityId=$(az identity show --resource-group <resource-group> --name <identity-name> --query id -o tsv)

az container create \
    --resource-group <resource-group> \
    --name <container-name> \
    --image <registry>.azurecr.io/myimage:v1 \
    --assign-identity $identityId \
    --acr-identity $identityId
```

---

## Azure App Service Integration

### Pattern 1: Managed Identity (Recommended)

**Description**: App Service uses managed identity to pull container images.

1. **Enable system-assigned identity**:
```bash
az webapp identity assign --name <app-name> --resource-group <resource-group>
```

2. **Get identity principal ID**:
```bash
principalId=$(az webapp identity show --name <app-name> --resource-group <resource-group> --query principalId -o tsv)
```

3. **Assign ACR role**:
```bash
acrId=$(az acr show --name <registry> --query id -o tsv)
az role assignment create --assignee $principalId --scope $acrId --role AcrPull
```

4. **Configure App Service to use ACR with identity**:
```bash
az webapp config container set \
    --name <app-name> \
    --resource-group <resource-group> \
    --docker-custom-image-name <registry>.azurecr.io/myimage:latest \
    --docker-registry-server-url https://<registry>.azurecr.io
```

### Pattern 2: Admin Account (Portal Deployments)

**Description**: Portal uses admin credentials for quick deployments.

**Note**: Only recommended for testing/development.

1. **Enable admin account**:
```bash
az acr update --name <registry> --admin-enabled true
```

2. **In Azure Portal**:
   - Navigate to App Service > Deployment Center
   - Select Container Registry as source
   - Choose registry and authenticate with admin credentials

---

## Azure DevOps Pipelines Integration

### Pattern 1: Service Connection with Service Principal

**Description**: Azure DevOps uses service connection to authenticate to ACR.

1. **Create service connection**:
   - Project Settings > Service connections > New
   - Select "Docker Registry"
   - Choose "Azure Container Registry"
   - Select subscription and registry

2. **Pipeline YAML**:
```yaml
trigger:
  - main

pool:
  vmImage: 'ubuntu-latest'

steps:
- task: Docker@2
  displayName: 'Build and Push'
  inputs:
    containerRegistry: 'my-acr-connection'
    repository: 'myapp'
    command: 'buildAndPush'
    Dockerfile: '**/Dockerfile'
    tags: |
      $(Build.BuildId)
      latest
```

### Pattern 2: Managed Identity (Self-Hosted Agents)

**Description**: Self-hosted agents on Azure VMs use managed identity.

1. **Enable identity on agent VM**:
```bash
az vm identity assign --resource-group <resource-group> --name <vm-name>
```

2. **Assign ACR role**:
```bash
principalId=$(az vm show --resource-group <resource-group> --name <vm-name> --query identity.principalId -o tsv)
acrId=$(az acr show --name <registry> --query id -o tsv)
az role assignment create --assignee $principalId --scope $acrId --role AcrPush
```

3. **Pipeline with az acr login**:
```yaml
steps:
- task: AzureCLI@2
  inputs:
    azureSubscription: 'my-azure-subscription'
    scriptType: 'bash'
    scriptLocation: 'inlineScript'
    inlineScript: |
      az acr login --name <registry>
      docker build -t <registry>.azurecr.io/myapp:$(Build.BuildId) .
      docker push <registry>.azurecr.io/myapp:$(Build.BuildId)
```

---

## GitHub Actions Integration

### Pattern 1: Service Principal with GitHub Secrets

**Description**: GitHub Actions uses service principal stored in secrets.

1. **Create service principal**:
```bash
az ad sp create-for-rbac \
    --name "github-actions-acr" \
    --scopes <acr-resource-id> \
    --role AcrPush \
    --sdk-auth
```

2. **Store in GitHub Secrets**:
   - `AZURE_CREDENTIALS`: Full JSON output from above command
   - `REGISTRY_LOGIN_SERVER`: `<registry>.azurecr.io`
   - `REGISTRY_USERNAME`: Service principal client ID
   - `REGISTRY_PASSWORD`: Service principal password

3. **GitHub Actions Workflow**:
```yaml
name: Build and Push to ACR

on:
  push:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3

    - name: Login to ACR
      uses: docker/login-action@v2
      with:
        registry: ${{ secrets.REGISTRY_LOGIN_SERVER }}
        username: ${{ secrets.REGISTRY_USERNAME }}
        password: ${{ secrets.REGISTRY_PASSWORD }}

    - name: Build and push
      uses: docker/build-push-action@v4
      with:
        context: .
        push: true
        tags: ${{ secrets.REGISTRY_LOGIN_SERVER }}/myapp:${{ github.sha }}
```

### Pattern 2: Azure Login Action

```yaml
steps:
- name: Azure Login
  uses: azure/login@v1
  with:
    creds: ${{ secrets.AZURE_CREDENTIALS }}

- name: ACR Login
  run: az acr login --name <registry>

- name: Build and Push
  run: |
    docker build -t <registry>.azurecr.io/myapp:${{ github.sha }} .
    docker push <registry>.azurecr.io/myapp:${{ github.sha }}
```

---

## Azure Machine Learning Integration

### Pattern 1: Workspace Managed Identity

**Description**: Azure ML workspace uses managed identity for ACR access.

1. **Get workspace identity**:
```bash
principalId=$(az ml workspace show --name <workspace> --resource-group <resource-group> --query identity.principalId -o tsv)
```

2. **Assign ACR role**:
```bash
acrId=$(az acr show --name <registry> --query id -o tsv)
az role assignment create --assignee $principalId --scope $acrId --role AcrPull
```

3. **Use in ML environment**:
```python
from azure.ai.ml.entities import Environment

env = Environment(
    name="my-custom-env",
    image="<registry>.azurecr.io/ml-base:latest"
)
```

---

## ACR Tasks Integration

### Pattern 1: Task with System-Assigned Identity

**Description**: ACR Task uses system-assigned identity for cross-registry authentication.

1. **Create task with identity**:
```bash
az acr task create \
    --name <task-name> \
    --registry <source-registry> \
    --image myapp:{{.Run.ID}} \
    --context https://github.com/myorg/myrepo.git \
    --file Dockerfile \
    --assign-identity
```

2. **Get task identity principal ID**:
```bash
principalId=$(az acr task show --name <task-name> --registry <source-registry> --query identity.principalId -o tsv)
```

3. **Assign role to target registry**:
```bash
targetAcrId=$(az acr show --name <target-registry> --query id -o tsv)
az role assignment create --assignee $principalId --scope $targetAcrId --role AcrPush
```

4. **Add credentials to task**:
```bash
az acr task credential add \
    --name <task-name> \
    --registry <source-registry> \
    --login-server <target-registry>.azurecr.io \
    --use-identity [system]
```

### Pattern 2: ABAC-Enabled Source Registry

**Description**: Task with identity for ABAC-enabled source registry.

```bash
az acr task create \
    --name <task-name> \
    --registry <source-registry> \
    --image myapp:{{.Run.ID}} \
    --context https://github.com/myorg/myrepo.git \
    --file Dockerfile \
    --assign-identity \
    --source-acr-auth-id [system]  # Required for ABAC registries
```

---

## Azure Functions Integration

### Pattern: Container Image from ACR

1. **Create function app with managed identity**:
```bash
az functionapp create \
    --name <function-name> \
    --resource-group <resource-group> \
    --storage-account <storage-account> \
    --plan <app-service-plan> \
    --deployment-container-image-name <registry>.azurecr.io/myfunction:latest \
    --assign-identity [system]
```

2. **Assign ACR pull role**:
```bash
principalId=$(az functionapp identity show --name <function-name> --resource-group <resource-group> --query principalId -o tsv)
acrId=$(az acr show --name <registry> --query id -o tsv)
az role assignment create --assignee $principalId --scope $acrId --role AcrPull
```

3. **Configure ACR settings**:
```bash
az functionapp config container set \
    --name <function-name> \
    --resource-group <resource-group> \
    --docker-registry-server-url https://<registry>.azurecr.io
```

---

## Network-Restricted Registry Integration

### Pattern: Private Endpoint with Trusted Services

**Description**: Azure services access network-restricted ACR.

1. **Enable trusted services**:
```bash
az acr update --name <registry> --allow-trusted-services true
```

2. **Configure managed identity on service**:
   - Enable managed identity on the Azure service
   - Assign appropriate ACR role to the identity

**Supported trusted services**:
- Azure Container Instances (with managed identity)
- Microsoft Defender for Cloud
- Azure Machine Learning (with managed identity)
- Azure Container Registry (for import operations)

---

## IoT Device Integration

### Pattern: Repository-Scoped Tokens

**Description**: IoT devices use limited-scope tokens for specific repositories.

1. **Create scope map**:
```bash
az acr scope-map create \
    --name iot-device-scope \
    --registry <registry> \
    --repository firmware content/read \
    --description "Read-only access to firmware repository"
```

2. **Create token**:
```bash
az acr token create \
    --name iot-device-001 \
    --registry <registry> \
    --scope-map iot-device-scope
```

3. **Generate password with expiration**:
```bash
az acr token credential generate \
    --name iot-device-001 \
    --registry <registry> \
    --expiration-in-days 365 \
    --password1
```

4. **Device authentication**:
```bash
# On IoT device
docker login <registry>.azurecr.io \
    --username iot-device-001 \
    --password <token-password>

docker pull <registry>.azurecr.io/firmware:latest
```

---

## Sources

- `/submodules/azure-management-docs/articles/container-registry/authenticate-kubernetes-options.md`
- `/submodules/azure-management-docs/articles/container-registry/authenticate-aks-cross-tenant.md`
- `/submodules/azure-management-docs/articles/container-registry/container-registry-auth-kubernetes.md`
- `/submodules/azure-management-docs/articles/container-registry/container-registry-auth-aci.md`
- `/submodules/azure-management-docs/articles/container-registry/allow-access-trusted-services.md`
- `/submodules/azure-management-docs/articles/container-registry/container-registry-tasks-authentication-managed-identity.md`
- `/submodules/azure-management-docs/articles/container-registry/container-registry-token-based-repository-permissions.md`
- `/submodules/acr/docs/integration/github-actions/github-actions.md`

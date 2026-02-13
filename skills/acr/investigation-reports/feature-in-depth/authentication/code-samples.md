# Azure Container Registry Authentication - Code Samples

## Azure CLI Authentication Examples

### Individual Login with Microsoft Entra ID

```bash
#!/bin/bash
# Login to Azure
az login

# Login to ACR (creates Docker credential)
az acr login --name myregistry

# Verify login
docker pull myregistry.azurecr.io/hello-world:latest
```

### Login Without Docker Daemon

```bash
#!/bin/bash
# Get access token for environments without Docker
TOKEN=$(az acr login --name myregistry --expose-token --output tsv --query accessToken)

# Use with docker login
echo $TOKEN | docker login myregistry.azurecr.io --username 00000000-0000-0000-0000-000000000000 --password-stdin

# Use with Helm
echo $TOKEN | helm registry login myregistry.azurecr.io --username 00000000-0000-0000-0000-000000000000 --password-stdin
```

### ACR-Scoped Authentication (Recommended)

```bash
#!/bin/bash
# Login with ACR-scoped token (more secure)
az login --scope https://containerregistry.azure.net/.default

# Then login to registry
az acr login --name myregistry
```

## Service Principal Authentication

### Create Service Principal for ACR

```bash
#!/bin/bash
# Variables
ACR_NAME="myregistry"
SERVICE_PRINCIPAL_NAME="acr-service-principal"

# Get ACR resource ID
ACR_REGISTRY_ID=$(az acr show --name $ACR_NAME --query "id" --output tsv)

# Create service principal with AcrPush role
PASSWORD=$(az ad sp create-for-rbac --name $SERVICE_PRINCIPAL_NAME \
  --scopes $ACR_REGISTRY_ID \
  --role acrpush \
  --query "password" --output tsv)

USER_NAME=$(az ad sp list --display-name $SERVICE_PRINCIPAL_NAME --query "[].appId" --output tsv)

echo "Service Principal ID: $USER_NAME"
echo "Service Principal Password: $PASSWORD"
```

### Login with Service Principal

```bash
#!/bin/bash
# Using environment variables
export SP_APP_ID="xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
export SP_PASSWORD="your-client-secret"
export REGISTRY="myregistry.azurecr.io"

# Docker login with service principal
docker login $REGISTRY --username $SP_APP_ID --password $SP_PASSWORD

# Or using stdin for password
echo $SP_PASSWORD | docker login $REGISTRY --username $SP_APP_ID --password-stdin
```

### Service Principal with Certificate

```bash
#!/bin/bash
SP_APP_ID="xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
SP_TENANT_ID="xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
CERT_PATH="/path/to/cert.pem"

# Login to Azure CLI with certificate
az login --service-principal --username $SP_APP_ID --tenant $SP_TENANT_ID --password $CERT_PATH

# Then login to ACR
az acr login --name myregistry
```

## Managed Identity Authentication

### System-Assigned Managed Identity on VM

```bash
#!/bin/bash
# On an Azure VM with system-assigned managed identity enabled

# Login with managed identity
az login --identity

# Login to ACR
az acr login --name myregistry

# Pull image
docker pull myregistry.azurecr.io/myimage:latest
```

### User-Assigned Managed Identity on VM

```bash
#!/bin/bash
# Variables
IDENTITY_CLIENT_ID="xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"

# Login with specific user-assigned identity
az login --identity --username $IDENTITY_CLIENT_ID

# Login to ACR
az acr login --name myregistry

# Pull image
docker pull myregistry.azurecr.io/myimage:latest
```

### Assign Managed Identity to ACR

```bash
#!/bin/bash
# Variables
IDENTITY_PRINCIPAL_ID="xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
ACR_NAME="myregistry"

# Get ACR resource ID
ACR_ID=$(az acr show --name $ACR_NAME --query id --output tsv)

# Assign AcrPull role (for non-ABAC registries)
az role assignment create \
  --assignee $IDENTITY_PRINCIPAL_ID \
  --scope $ACR_ID \
  --role AcrPull

# Or for ABAC-enabled registries
az role assignment create \
  --assignee $IDENTITY_PRINCIPAL_ID \
  --scope $ACR_ID \
  --role "Container Registry Repository Reader"
```

## OAuth2 Token Exchange

### Exchange AAD Token for ACR Refresh Token

```bash
#!/bin/bash
# Variables
REGISTRY="myregistry.azurecr.io"
TENANT="xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"

# Get AAD access token
AAD_ACCESS_TOKEN=$(az account get-access-token --query accessToken --output tsv)

# Exchange for ACR refresh token
ACR_REFRESH_TOKEN=$(curl -s -X POST -H "Content-Type: application/x-www-form-urlencoded" \
  -d "grant_type=access_token&service=$REGISTRY&tenant=$TENANT&access_token=$AAD_ACCESS_TOKEN" \
  "https://$REGISTRY/oauth2/exchange" | jq -r '.refresh_token')

echo "ACR Refresh Token: $ACR_REFRESH_TOKEN"
```

### Get ACR Access Token

```bash
#!/bin/bash
# Variables
REGISTRY="myregistry.azurecr.io"
ACR_REFRESH_TOKEN="eyJ..." # From previous step
SCOPE="repository:myrepo:pull"

# Get access token for specific scope
ACR_ACCESS_TOKEN=$(curl -s -X POST -H "Content-Type: application/x-www-form-urlencoded" \
  -d "grant_type=refresh_token&service=$REGISTRY&scope=$SCOPE&refresh_token=$ACR_REFRESH_TOKEN" \
  "https://$REGISTRY/oauth2/token" | jq -r '.access_token')

echo "ACR Access Token: $ACR_ACCESS_TOKEN"
```

### Use ACR Token for Docker Login

```bash
#!/bin/bash
REGISTRY="myregistry.azurecr.io"
ACR_REFRESH_TOKEN="eyJ..."

# Docker login with ACR refresh token
docker login $REGISTRY --username 00000000-0000-0000-0000-000000000000 --password $ACR_REFRESH_TOKEN
```

## Repository Token Examples

### Create Repository Token

```bash
#!/bin/bash
# Create token with inline permissions
az acr token create \
  --name mytoken \
  --registry myregistry \
  --repository samples/hello-world content/write content/read

# Get token credentials
az acr token credential generate \
  --name mytoken \
  --registry myregistry \
  --password1 \
  --expiration-in-days 30
```

### Create Scope Map and Token

```bash
#!/bin/bash
# Create scope map
az acr scope-map create \
  --name myscope \
  --registry myregistry \
  --repository samples/app1 content/read content/write \
  --repository samples/app2 content/read \
  --description "Custom scope for CI/CD"

# Create token with scope map
az acr token create \
  --name cicd-token \
  --registry myregistry \
  --scope-map myscope

# Generate password
az acr token credential generate \
  --name cicd-token \
  --registry myregistry \
  --password1
```

### Wildcard Scope Map

```bash
#!/bin/bash
# Create scope map with wildcard
az acr scope-map create \
  --name dev-scope \
  --registry myregistry \
  --repository "dev/*" content/read content/write \
  --description "Access to all dev repositories"

# Create token
az acr token create \
  --name dev-token \
  --registry myregistry \
  --scope-map dev-scope
```

### Authenticate with Repository Token

```bash
#!/bin/bash
TOKEN_NAME="mytoken"
TOKEN_PASSWORD="xxxxxxxx"
REGISTRY="myregistry.azurecr.io"

# Docker login with token
docker login $REGISTRY --username $TOKEN_NAME --password $TOKEN_PASSWORD

# Or using az acr login
az acr login --name myregistry --username $TOKEN_NAME --password $TOKEN_PASSWORD
```

## Kubernetes Integration Examples

### Create Image Pull Secret

```bash
#!/bin/bash
# Using service principal
kubectl create secret docker-registry acr-secret \
  --namespace default \
  --docker-server=myregistry.azurecr.io \
  --docker-username=<service-principal-id> \
  --docker-password=<service-principal-password>
```

### Pod YAML with Pull Secret

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-app
  namespace: default
spec:
  containers:
  - name: my-app
    image: myregistry.azurecr.io/myapp:v1
    imagePullPolicy: IfNotPresent
  imagePullSecrets:
  - name: acr-secret
```

### Attach ACR to AKS

```bash
#!/bin/bash
# Attach ACR to AKS (automatic role assignment)
az aks update \
  --name myAKSCluster \
  --resource-group myResourceGroup \
  --attach-acr myregistry

# Verify attachment
az aks check-acr \
  --name myAKSCluster \
  --resource-group myResourceGroup \
  --acr myregistry
```

## ACI Integration Examples

### Deploy to ACI with Service Principal

```bash
#!/bin/bash
az container create \
  --resource-group myResourceGroup \
  --name mycontainer \
  --image myregistry.azurecr.io/myapp:v1 \
  --cpu 1 --memory 1 \
  --registry-login-server myregistry.azurecr.io \
  --registry-username <service-principal-id> \
  --registry-password <service-principal-password>
```

### Deploy to ACI with Managed Identity

```yaml
# YAML deployment with managed identity
apiVersion: 2019-12-01
location: eastus
name: mycontainergroup
type: Microsoft.ContainerInstance/containerGroups
identity:
  type: UserAssigned
  userAssignedIdentities:
    /subscriptions/<sub>/resourceGroups/<rg>/providers/Microsoft.ManagedIdentity/userAssignedIdentities/myidentity: {}
properties:
  containers:
  - name: mycontainer
    properties:
      image: myregistry.azurecr.io/myapp:v1
      resources:
        requests:
          cpu: 1
          memoryInGB: 1
  imageRegistryCredentials:
  - server: myregistry.azurecr.io
    identity: /subscriptions/<sub>/resourceGroups/<rg>/providers/Microsoft.ManagedIdentity/userAssignedIdentities/myidentity
  osType: Linux
```

## API Call Examples

### Catalog Listing with Basic Auth

```bash
#!/bin/bash
REGISTRY="myregistry.azurecr.io"
USERNAME="admin-username"
PASSWORD="admin-password"

# Encode credentials
CREDENTIALS=$(echo -n "$USERNAME:$PASSWORD" | base64)

# Get catalog
curl -s -H "Authorization: Basic $CREDENTIALS" \
  "https://$REGISTRY/v2/_catalog"
```

### Catalog Listing with Bearer Token

```bash
#!/bin/bash
REGISTRY="myregistry.azurecr.io"
USERNAME="admin-username"
PASSWORD="admin-password"
CREDENTIALS=$(echo -n "$USERNAME:$PASSWORD" | base64)

# Get the scope from challenge
SCOPE="registry:catalog:*"

# Get access token
ACCESS_TOKEN=$(curl -s -H "Authorization: Basic $CREDENTIALS" \
  "https://$REGISTRY/oauth2/token?service=$REGISTRY&scope=$SCOPE" | jq -r '.access_token')

# Get catalog with bearer token
curl -s -H "Authorization: Bearer $ACCESS_TOKEN" \
  "https://$REGISTRY/v2/_catalog"
```

### Tag Listing

```bash
#!/bin/bash
REGISTRY="myregistry.azurecr.io"
REPOSITORY="myapp"
ACCESS_TOKEN="eyJ..."

# List tags
curl -s -H "Authorization: Bearer $ACCESS_TOKEN" \
  "https://$REGISTRY/v2/$REPOSITORY/tags/list"
```

### Paginated Catalog Listing

```bash
#!/bin/bash
REGISTRY="myregistry.azurecr.io"
ACCESS_TOKEN="eyJ..."
LIMIT=10

OPERATION="/v2/_catalog?n=$LIMIT"
HEADERS=$(mktemp)

while [ -n "$OPERATION" ]; do
  echo "Fetching: $OPERATION"

  CATALOG=$(curl -s -H "Authorization: Bearer $ACCESS_TOKEN" \
    "https://$REGISTRY$OPERATION" -D $HEADERS)

  echo "Results:"
  echo $CATALOG | jq '.repositories[]'

  # Get next page from Link header
  OPERATION=$(grep -i "^Link:" $HEADERS | sed -n 's/^Link: <\(.*\)>.*/\1/p' | tr -d '\r')
done

rm $HEADERS
```

## GitHub Actions Examples

### Build and Push with Service Principal

```yaml
name: Build and Push

on:
  push:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3

    - name: Login to ACR
      uses: azure/docker-login@v1
      with:
        login-server: myregistry.azurecr.io
        username: ${{ secrets.ACR_USERNAME }}
        password: ${{ secrets.ACR_PASSWORD }}

    - name: Build and Push
      run: |
        docker build -t myregistry.azurecr.io/myapp:${{ github.sha }} .
        docker push myregistry.azurecr.io/myapp:${{ github.sha }}
```

### Build and Push with OIDC

```yaml
name: Build with OIDC

on:
  push:
    branches: [main]

permissions:
  id-token: write
  contents: read

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3

    - name: Azure Login
      uses: azure/login@v1
      with:
        client-id: ${{ secrets.AZURE_CLIENT_ID }}
        tenant-id: ${{ secrets.AZURE_TENANT_ID }}
        subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

    - name: Login to ACR
      run: az acr login --name myregistry

    - name: Build and Push
      run: |
        docker build -t myregistry.azurecr.io/myapp:${{ github.sha }} .
        docker push myregistry.azurecr.io/myapp:${{ github.sha }}
```

## Azure DevOps Pipeline Examples

### Build Pipeline with ACR

```yaml
trigger:
- main

pool:
  vmImage: 'ubuntu-latest'

variables:
  acrName: 'myregistry'
  imageName: 'myapp'

steps:
- task: Docker@2
  displayName: 'Login to ACR'
  inputs:
    command: 'login'
    containerRegistry: 'ACRServiceConnection'

- task: Docker@2
  displayName: 'Build and Push'
  inputs:
    command: 'buildAndPush'
    repository: '$(imageName)'
    dockerfile: '**/Dockerfile'
    containerRegistry: 'ACRServiceConnection'
    tags: |
      $(Build.BuildId)
      latest
```

## PowerShell Examples

### Login with PowerShell

```powershell
# Login to Azure
Connect-AzAccount

# Login to ACR
Connect-AzContainerRegistry -Name myregistry

# Verify
docker pull myregistry.azurecr.io/hello-world:latest
```

### Create Service Principal with PowerShell

```powershell
# Variables
$acrName = "myregistry"
$spName = "acr-sp"

# Get ACR resource ID
$acr = Get-AzContainerRegistry -Name $acrName
$acrId = $acr.Id

# Create service principal
$sp = New-AzADServicePrincipal -DisplayName $spName -Role "AcrPush" -Scope $acrId

# Get credentials
$clientId = $sp.AppId
$secret = $sp.PasswordCredentials.SecretText

Write-Output "Client ID: $clientId"
Write-Output "Secret: $secret"
```

### Assign Role with PowerShell

```powershell
# Variables
$principalId = "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
$acrName = "myregistry"

# Get ACR
$acr = Get-AzContainerRegistry -Name $acrName

# Assign role
New-AzRoleAssignment -ObjectId $principalId -Scope $acr.Id -RoleDefinitionName "AcrPull"
```

## Python SDK Examples

### Authenticate and Pull Image Manifest

```python
from azure.identity import DefaultAzureCredential
from azure.containerregistry import ContainerRegistryClient

# Use default credentials (works with managed identity, CLI, etc.)
credential = DefaultAzureCredential()

# Create client
endpoint = "https://myregistry.azurecr.io"
client = ContainerRegistryClient(endpoint, credential)

# List repositories
for repository in client.list_repository_names():
    print(repository)

# List tags for a repository
for tag in client.list_tag_properties("myapp"):
    print(f"Tag: {tag.name}, Created: {tag.created_on}")
```

### Authenticate with Service Principal

```python
from azure.identity import ClientSecretCredential
from azure.containerregistry import ContainerRegistryClient

# Service principal credentials
tenant_id = "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
client_id = "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
client_secret = "your-client-secret"

credential = ClientSecretCredential(tenant_id, client_id, client_secret)

# Create client
endpoint = "https://myregistry.azurecr.io"
client = ContainerRegistryClient(endpoint, credential)

# Use client...
```

## Sources

- `/Users/johnsonshi/repos/acr-skills/submodules/acr/docs/AAD-OAuth.md`
- `/Users/johnsonshi/repos/acr-skills/submodules/acr/docs/Token-BasicAuth.md`
- `/Users/johnsonshi/repos/acr-skills/submodules/azure-management-docs/articles/container-registry/container-registry-authentication.md`
- `/Users/johnsonshi/repos/acr-skills/submodules/azure-management-docs/articles/container-registry/container-registry-token-based-repository-permissions.md`
- `/Users/johnsonshi/repos/acr-skills/submodules/azure-management-docs/articles/container-registry/includes/container-registry-service-principal.md`
- `/Users/johnsonshi/repos/acr-skills/submodules/acr/docs/integration/github-actions/github-actions.md`

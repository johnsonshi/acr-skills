# ACR Artifact Cache - Setup with Azure Portal

## Prerequisites

Before configuring artifact cache in the Azure portal, ensure you have:

1. **Azure Subscription** with an active account
2. **Existing ACR Instance** (Basic, Standard, or Premium tier)
3. **Azure Key Vault** (for storing credentials for authenticated pulls)
4. **Permissions** to create and retrieve secrets from Key Vault

## Step 1: Navigate to Your Container Registry

1. Sign in to the [Azure portal](https://portal.azure.com)
2. Search for **Container registries** in the top search bar
3. Select your container registry from the list

## Step 2: Access the Cache Feature

1. In your container registry's blade, find the service menu on the left
2. Under **Services**, select **Cache**
3. You'll see two tabs:
   - **Cache rules** - Manage your caching rules
   - **Credentials** - Manage authentication credentials

## Step 3: Create New Credentials (If Required)

Credentials are required for:
- Docker Hub (always required)
- Google Artifact Registry
- Private upstream registries

### 3.1 Navigate to Credentials Tab

1. In the Cache pane, select the **Credentials** tab
2. Click **Create credentials**

### 3.2 Enter Credential Details

Fill in the following information:

| Field | Description | Example |
|-------|-------------|---------|
| **Name** | A unique name for your credentials | `dockerhub-creds` |
| **Source Authentication** | How to provide credentials | `Select from Key Vault` or `Enter secret URIs` |

### 3.3 Option A: Select from Key Vault

If you choose "Select from Key Vault":

1. **Key Vault**: Select your Azure Key Vault from the dropdown
2. **Username secret**: Select the secret containing your username
3. **Password secret**: Select the secret containing your password/token

> **Note**: You must first create secrets in your Key Vault containing your upstream registry credentials.

### 3.4 Option B: Enter Secret URIs

If you choose "Enter secret URIs":

1. **Username secret URI**: Enter the full URI to your username secret
   - Example: `https://mykeyvault.vault.azure.net/secrets/dockerhub-username`
2. **Password secret URI**: Enter the full URI to your password secret
   - Example: `https://mykeyvault.vault.azure.net/secrets/dockerhub-password`

### 3.5 Create the Credentials

1. Review your settings
2. Click **Create**
3. Wait for the credential set to be created

### 3.6 Assign Key Vault Permissions

After creating credentials, you may need to assign Key Vault access permissions:

**Using Azure RBAC (Recommended):**

1. Navigate to your Azure Key Vault
2. Go to **Access control (IAM)**
3. Click **Add** > **Add role assignment**
4. Select **Key Vault Secrets User** role
5. Click **Next**
6. Select **Managed identity**
7. Click **+ Select members**
8. Choose **Credential set** as the managed identity type
9. Select your credential set
10. Click **Select** and then **Review + assign**

**Using Access Policies:**

1. Navigate to your Azure Key Vault
2. Go to **Access policies**
3. Click **Create**
4. Under **Secret permissions**, select **Get**
5. Click **Next**
6. Search for and select your credential set's managed identity
7. Click **Next** and then **Create**

## Step 4: Create a Cache Rule

### 4.1 Navigate to Cache Rules Tab

1. In your container registry's **Cache** pane
2. Select the **Cache rules** tab
3. Click **Create rule**

### 4.2 Enter Rule Details

Fill in the following information in the **New cache rule** pane:

| Field | Description | Example |
|-------|-------------|---------|
| **Rule name** | Unique name for the cache rule | `nginx-cache` |
| **Source** | Select the upstream registry type | `Docker Hub` |
| **Repository Path** | Full path to the upstream repository | `library/nginx` |
| **Authentication** | Optional - check if credentials needed | Checked/Unchecked |
| **Destination** | Local ACR repository namespace | `nginx` |

### 4.3 Configure Source Registry

Select from available source registry types:
- **Docker Hub** (`docker.io`)
- **Microsoft Artifact Registry** (`mcr.microsoft.com`)
- **GitHub Container Registry** (`ghcr.io`)
- **Quay** (`quay.io`)
- **AWS ECR Public** (`public.ecr.aws`)

### 4.4 Enter Repository Path

Enter the complete path to the repository you want to cache:

**Examples by registry:**

| Registry | Repository Path Format | Example |
|----------|------------------------|---------|
| Docker Hub | `library/<image>` or `<user>/<image>` | `library/nginx`, `bitnami/redis` |
| MCR | `<namespace>/<image>` | `dotnet/runtime`, `azure-cli` |
| GitHub | `<owner>/<image>` | `actions/runner` |
| Quay | `<namespace>/<image>` | `coreos/etcd` |

### 4.5 Configure Authentication (If Required)

If the **Authentication** checkbox appears:

**For Unauthenticated Registries:**
- Leave the checkbox unchecked

**For Authenticated Registries:**
1. Check the **Authentication** box
2. Choose credential option:
   - **Select credentials**: Choose from existing credential sets
   - **Create new credentials**: Follow Step 3 above

### 4.6 Set Destination Namespace

Enter the name of the ACR repository where cached images will be stored:

- This becomes part of your pull path: `<acrname>.azurecr.io/<destination>:<tag>`
- **Important**: This repository must NOT already exist in your ACR

### 4.7 Create the Rule

1. Review all settings
2. Click **Create**
3. Wait for the cache rule to be created
4. The rule appears in your cache rules list

## Step 5: Pull Your Image

After creating the cache rule:

### 5.1 Login to Your ACR

```bash
az acr login --name <your-registry-name>
# OR
docker login <your-registry-name>.azurecr.io
```

### 5.2 Pull the Cached Image

```bash
docker pull <your-registry-name>.azurecr.io/<destination>:<tag>
```

**Example:**
```bash
docker pull myregistry.azurecr.io/nginx:latest
```

The first pull will fetch from the upstream registry and cache it. Subsequent pulls will come from your ACR cache.

## Complete Portal Walkthrough: Docker Hub Image

### Example: Caching nginx from Docker Hub

**Step 1: Create Key Vault Secrets (Prerequisites)**

1. Navigate to your Azure Key Vault
2. Go to **Secrets**
3. Click **Generate/Import**
4. Create username secret:
   - Name: `dockerhub-username`
   - Value: Your Docker Hub username
5. Create password secret:
   - Name: `dockerhub-token`
   - Value: Your Docker Hub access token

**Step 2: Create Credentials**

1. Go to your ACR > **Cache** > **Credentials**
2. Click **Create credentials**
3. Enter:
   - Name: `dockerhub-creds`
   - Source Authentication: Select from Key Vault
   - Key Vault: Select your Key Vault
   - Username secret: `dockerhub-username`
   - Password secret: `dockerhub-token`
4. Click **Create**

**Step 3: Assign Key Vault Access**

1. Go to Key Vault > **Access control (IAM)**
2. Add role assignment:
   - Role: Key Vault Secrets User
   - Assign to: Managed identity
   - Select: Your credential set

**Step 4: Create Cache Rule**

1. Go to your ACR > **Cache** > **Cache rules**
2. Click **Create rule**
3. Enter:
   - Rule name: `nginx-cache`
   - Source: Docker Hub
   - Repository Path: `library/nginx`
   - Authentication: Checked
   - Select credentials: `dockerhub-creds`
   - Destination: `nginx`
4. Click **Create**

**Step 5: Test the Cache**

```bash
# Login
az acr login --name myregistry

# Pull (triggers caching)
docker pull myregistry.azurecr.io/nginx:latest

# Subsequent pulls come from cache
docker pull myregistry.azurecr.io/nginx:alpine
```

## Complete Portal Walkthrough: MCR Image (Unauthenticated)

### Example: Caching .NET Runtime from MCR

**Step 1: Create Cache Rule**

1. Go to your ACR > **Cache** > **Cache rules**
2. Click **Create rule**
3. Enter:
   - Rule name: `dotnet-runtime-cache`
   - Source: Microsoft Artifact Registry
   - Repository Path: `dotnet/runtime`
   - Authentication: Leave unchecked (not required for MCR)
   - Destination: `dotnet/runtime`
4. Click **Create**

**Step 2: Test the Cache**

```bash
# Login
az acr login --name myregistry

# Pull (triggers caching)
docker pull myregistry.azurecr.io/dotnet/runtime:8.0
```

## Managing Cache Rules in Portal

### View Cache Rule Details

1. Go to **Cache** > **Cache rules**
2. Click on a rule name to view details
3. See:
   - Source repository
   - Target repository
   - Associated credentials
   - Creation date

### Edit Cache Rule

1. Go to **Cache** > **Cache rules**
2. Click on the rule to edit
3. Modify settings as needed
4. Save changes

### Delete Cache Rule

1. Go to **Cache** > **Cache rules**
2. Select the rule(s) to delete
3. Click **Delete**
4. Confirm deletion

## Managing Credentials in Portal

### View Credential Details

1. Go to **Cache** > **Credentials**
2. Click on a credential set name
3. View:
   - Login server
   - Secret URIs
   - Identity information

### Edit Credentials

1. Go to **Cache** > **Credentials**
2. Click on the credential set
3. Update secret URIs or authentication method
4. Save changes

### Delete Credentials

1. Go to **Cache** > **Credentials**
2. Select the credential set(s) to delete
3. Click **Delete**
4. Confirm deletion

> **Note**: Remove credential associations from cache rules before deleting credential sets.

## Troubleshooting Portal Issues

### Common Issues

| Issue | Possible Cause | Solution |
|-------|---------------|----------|
| Can't create credentials | Key Vault not accessible | Verify Key Vault permissions |
| Credential set shows unhealthy | Invalid secrets | Update secrets in Key Vault |
| Can't create cache rule | Overlapping rule exists | Check for existing rules |
| Rule creation fails | Target repo exists | Choose different destination |
| Pull fails after rule creation | Key Vault access not granted | Assign RBAC role |

### Checking Credential Health

1. Go to **Cache** > **Credentials**
2. Check the **Status** column
3. Unhealthy credentials show a warning icon
4. Click on unhealthy credentials to view details and fix

## Best Practices

1. **Use descriptive names** for rules and credentials
2. **Test credentials** before creating cache rules
3. **Verify Key Vault access** immediately after creating credentials
4. **Plan namespace strategy** before creating multiple rules
5. **Monitor usage** to stay within rule limits
6. **Document your cache rules** for team reference
7. **Use access tokens** instead of passwords when possible

## References

- [Enable Artifact Cache - Portal Documentation](https://learn.microsoft.com/azure/container-registry/artifact-cache-portal)
- [Azure Key Vault Portal Guide](https://learn.microsoft.com/azure/key-vault/)
- [ACR Portal Quickstart](https://learn.microsoft.com/azure/container-registry/container-registry-get-started-portal)

# Docker Content Trust (DCT) - Legacy

## Deprecation Notice

> **IMPORTANT**: Docker Content Trust (DCT) is being deprecated and will be completely removed from Azure Container Registry on **March 31, 2028**.
>
> - **May 31, 2026**: DCT cannot be enabled on new registries or registries that haven't enabled it previously
> - **March 31, 2028**: DCT will be completely removed from Azure Container Registry
>
> **Recommendation**: Transition to Notary Project/Notation for container image signing.

## Overview

Docker Content Trust (DCT) allows image publishers to sign their images and allows image consumers to verify that the images they pull are signed. ACR implements the Docker Content Trust model to enable pushing and pulling signed images.

**Key Limitations:**
- Premium service tier only
- Repository-scoped tokens do NOT support DCT push/pull of signed images
- Cannot use `acr import` to import DCT-signed images (signatures not visible after import)
- Admin account cannot sign images
- Being deprecated in favor of Notary Project

## How DCT Works

### Trust Model

DCT provides two key assurances:
1. **Source verification**: Consumers can verify the publisher (source) of the image
2. **Integrity verification**: Images weren't modified after publication

### Signing Keys

DCT uses several types of cryptographic signing keys:

| Key Type | Purpose | Storage |
|----------|---------|---------|
| **Root Key** | Most sensitive key, signs delegation and timestamp keys | Local (`~/.docker/trust/private`) |
| **Repository Key** | Signs image tags for a specific repository | Local (`~/.docker/trust/private`) |
| **Timestamp Key** | Ensures signature freshness | Registry server |
| **Snapshot Key** | Prevents mix-and-match attacks | Registry server |

**Key Management Best Practices:**
```bash
# Backup keys securely
umask 077; tar -zcvf docker_private_keys_backup.tar.gz ~/.docker/trust/private; umask 022
```

**WARNING**: If you lose your root key, you lose access to all signed tags in repositories that key signed. There is no recovery mechanism.

## Enabling DCT

### Enable DCT for the Registry

**Azure Portal:**
1. Navigate to your registry
2. Under **Policies**, select **Content trust**
3. Select **Enabled**, then **Save**

**Azure CLI:**
```bash
az acr config content-trust update -r myregistry --status enabled
```

### Enable DCT for Docker Client

**Enable for shell session:**
```bash
export DOCKER_CONTENT_TRUST=1
```

**Enable for single command:**
```bash
docker build --disable-content-trust=false -t myacr.azurecr.io/myimage:v1 .
```

**Disable for single command (when session is enabled):**
```bash
docker build --disable-content-trust -t myacr.azurecr.io/myimage:v1 .
```

## Required Permissions

### AcrImageSigner Role

The `AcrImageSigner` role is required to push signed images. This role includes:
- `Microsoft.ContainerRegistry/registries/sign/write`
- `Microsoft.ContainerRegistry/registries/trustedCollections/write`

**Assign via Azure Portal:**
1. Go to registry > **Access Control (IAM)**
2. Select **Add** > **Add role assignment**
3. Select **AcrImageSigner** role
4. Assign to user or service principal

**Assign via Azure CLI:**
```bash
# Get registry resource ID
REGISTRY_ID=$(az acr show --name myregistry --query id --output tsv)

# Assign to user
az role assignment create --scope $REGISTRY_ID --role AcrImageSigner --assignee user@contoso.com

# Assign to service principal
az role assignment create --scope $REGISTRY_ID --role AcrImageSigner --assignee <service-principal-id>
```

**Important Notes:**
- Cannot grant to admin account
- Cannot grant to users with classic system administrator role
- After role changes, run `az acr login` to refresh local identity token

### Roles for Pulling Signed Images

Only the `AcrPull` role is required for pulling signed images. No additional roles like `AcrImageSigner` are needed.

## Pushing Signed Images

```bash
# Enable DCT
export DOCKER_CONTENT_TRUST=1

# Push image (first push initializes trust)
docker push myregistry.azurecr.io/myimage:v1
```

**First-time Push Output:**
```
The push refers to repository [myregistry.azurecr.io/myimage]
ee83fc5847cb: Pushed
v1: digest: sha256:aca41a608e5eb015f1ec6755f490f3be26b48010b178e78c00eac21ffbe246f1 size: 524
Signing and pushing trust metadata
You are about to create a new root signing key passphrase. This passphrase
will be used to protect the most sensitive key in your signing system. Please
choose a long, complex passphrase and be careful to keep the password and the
key file itself secure and backed up...
Enter passphrase for new root key with ID 4c6c56a:
Repeat passphrase for new root key with ID 4c6c56a:
Enter passphrase for new repository key with ID bcd6d98:
Repeat passphrase for new repository key with ID bcd6d98:
Finished initializing "myregistry.azurecr.io/myimage"
Successfully signed myregistry.azurecr.io/myimage:v1
```

**Subsequent Pushes:**
- Same repository: Only repository key passphrase required
- New repository: New repository key generated

## Pulling Signed Images

```bash
# Enable DCT
export DOCKER_CONTENT_TRUST=1

# Pull signed image
docker pull myregistry.azurecr.io/myimage:signed
```

**Successful Pull Output:**
```
Pull (1 of 1): myregistry.azurecr.io/myimage:signed@sha256:0800d17e37fb4f8194495b1a188f121e5b54efb52b5d93dc9e0ed97fce49564b
sha256:0800d17e37fb4f8194495b1a188f121e5b54efb52b5d93dc9e0ed97fce49564b: Pulling from myimage
8e3ba11ec2a2: Pull complete
Digest: sha256:0800d17e37fb4f8194495b1a188f121e5b54efb52b5d93dc9e0ed97fce49564b
Status: Downloaded newer image for myregistry.azurecr.io/myimage@sha256:0800d17e37fb4f8194495b1a188f121e5b54efb52b5d93dc9e0ed97fce49564b
Tagging myregistry.azurecr.io/myimage@sha256:0800d17e37fb4f8194495b1a188f121e5b54efb52b5d93dc9e0ed97fce49564b as myregistry.azurecr.io/myimage:signed
```

**Error When Pulling Unsigned Image:**
```
$ docker pull myregistry.azurecr.io/myimage:unsigned
Error: remote trust data does not exist
```

## Behind the Scenes

When pulling with DCT enabled:
1. Docker client requests tag-to-SHA-256 digest mapping via Notary CLI library
2. Client validates signatures on the trust data
3. Docker Engine performs "pull by digest" using SHA-256 checksum
4. Engine validates image manifest against the checksum

**Compatibility Note:** ACR is compatible with Notary Server API but doesn't officially support Notary CLI. Recommended Notary version: 0.6.0

## Key Management

### Key Storage Location
```
~/.docker/trust/private
```

### Lost Root Key Recovery

**WARNING**: There is no way to recover a lost root key.

If the root key is lost, the only option is to disable and re-enable DCT for the registry:

1. Go to registry in Azure Portal
2. Under **Policies**, select **Content Trust**
3. Select **Disabled**, then **Save**
4. Re-enable DCT

**CAUTION**: This action:
- **Permanently deletes** ALL signatures for ALL signed tags in EVERY repository
- Cannot be undone
- Does NOT delete the images themselves

## Disabling DCT

### Disable for Client

```bash
# Unset environment variable
unset DOCKER_CONTENT_TRUST

# Or set to 0
export DOCKER_CONTENT_TRUST=0
```

### Disable for Registry

**Azure Portal:**
1. Navigate to registry > **Policies** > **Content Trust**
2. Select **Disabled** > **Save**

**Azure CLI:**
```bash
az acr config content-trust update -r myregistry --status disabled
```

## Transition to Notary Project

### Why Migrate?

Docker Content Trust no longer meets modern supply-chain security requirements:
- Limited to Docker ecosystem
- No native cloud key management integration
- No Kubernetes verification support
- No CI/CD pipeline integration

### Benefits of Notary Project

| Feature | Notary Project Advantage |
|---------|-------------------------|
| **Portability** | OCI-compliant signatures work across cloud environments |
| **Key Management** | Azure Key Vault integration |
| **Certificate Management** | Artifact Signing for zero-touch management |
| **CI/CD** | Native GitHub Actions and Azure DevOps support |
| **Kubernetes** | Ratify + Azure Policy verification on AKS |
| **Timestamping** | RFC 3161 compliant for extended trust |

### Migration Steps

1. **Disable DCT on client:**
   ```bash
   export DOCKER_CONTENT_TRUST=0
   # or
   unset DOCKER_CONTENT_TRUST
   ```

2. **Disable DCT on registry:**
   ```bash
   az acr config content-trust update -r myregistry --status disabled
   ```

3. **Install Notation CLI and plug-ins** (see [notation-signing.md](./notation-signing.md))

4. **Configure Azure Key Vault or Artifact Signing**

5. **Re-sign images using Notation**

6. **Update CI/CD pipelines** to use Notation GitHub actions or Azure DevOps tasks

7. **Deploy Ratify on AKS** for runtime verification

## Related Resources

- [Transition from Docker Content Trust to Notary Project](./feature-overview.md)
- [Notation Signing Guide](./notation-signing.md)
- [Azure Key Vault Integration](./azure-key-vault.md)
- [Docker Content Trust Documentation](https://docs.docker.com/engine/security/trust/content_trust)

## Source Files Referenced

- `/submodules/azure-management-docs/articles/container-registry/container-registry-content-trust.md`
- `/submodules/azure-management-docs/articles/container-registry/container-registry-content-trust-deprecation.md`
- `/submodules/acr/docs/image-signing.md`

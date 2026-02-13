# Azure Container Registry Private Endpoints - DNS Configuration Guide

## Overview

DNS configuration is critical for private endpoints to function correctly. When properly configured, the registry's public FQDN (e.g., `myregistry.azurecr.io`) resolves to private IP addresses within your virtual network, enabling seamless private connectivity without application changes.

## DNS Resolution Flow

### How DNS Resolution Works with Private Endpoints

```
Client Request: myregistry.azurecr.io
         |
         v
+---------------------------+
|   Public DNS Resolution   |
| myregistry.azurecr.io     |
|         |                 |
|         v                 |
| CNAME: myregistry.        |
|   privatelink.azurecr.io  |
+---------------------------+
         |
         v
+---------------------------+
|  Private DNS Zone         |
|  privatelink.azurecr.io   |
|         |                 |
|         v                 |
| A Record: 10.0.0.5        |
| (Private IP)              |
+---------------------------+
         |
         v
   Client connects to
   private IP 10.0.0.5
```

## Private DNS Zone Configuration

### Required DNS Zone

Create a private DNS zone named `privatelink.azurecr.io` to enable automatic resolution:

```bash
az network private-dns zone create \
  --resource-group myResourceGroup \
  --name "privatelink.azurecr.io"
```

### VNet Link

Link the private DNS zone to your virtual network:

```bash
az network private-dns link vnet create \
  --resource-group myResourceGroup \
  --zone-name "privatelink.azurecr.io" \
  --name MyDNSLink \
  --virtual-network myVNet \
  --registration-enabled false
```

**Important**: Set `--registration-enabled false` since the private endpoint manages DNS record creation.

## DNS Records Required

### Single-Region Registry

For a registry in West US:

| Record Name | Record Type | Value | Purpose |
|-------------|-------------|-------|---------|
| `myregistry` | A | `10.0.0.5` | Registry REST API endpoint |
| `myregistry.westus.data` | A | `10.0.0.6` | Data endpoint for image layers |

### Geo-Replicated Registry

For a registry with replicas in multiple regions:

| Record Name | Record Type | Value | Purpose |
|-------------|-------------|-------|---------|
| `myregistry` | A | `10.0.0.5` | Registry REST API |
| `myregistry.westus.data` | A | `10.0.0.6` | Primary data endpoint |
| `myregistry.eastus.data` | A | `10.0.0.7` | East US replica data endpoint |
| `myregistry.westeurope.data` | A | `10.0.0.8` | West Europe replica data endpoint |

## Manual DNS Record Configuration

### Step 1: Get Private IP Addresses

```bash
# Get network interface ID from private endpoint
NETWORK_INTERFACE_ID=$(az network private-endpoint show \
  --name myPrivateEndpoint \
  --resource-group myResourceGroup \
  --query 'networkInterfaces[0].id' \
  --output tsv)

# Get registry private IP
REGISTRY_PRIVATE_IP=$(az network nic show \
  --ids $NETWORK_INTERFACE_ID \
  --query "ipConfigurations[?privateLinkConnectionProperties.requiredMemberName=='registry'].privateIPAddress" \
  --output tsv)

# Get data endpoint private IP
DATA_ENDPOINT_PRIVATE_IP=$(az network nic show \
  --ids $NETWORK_INTERFACE_ID \
  --query "ipConfigurations[?privateLinkConnectionProperties.requiredMemberName=='registry_data_westus'].privateIPAddress" \
  --output tsv)

# Get FQDNs
REGISTRY_FQDN=$(az network nic show \
  --ids $NETWORK_INTERFACE_ID \
  --query "ipConfigurations[?privateLinkConnectionProperties.requiredMemberName=='registry'].privateLinkConnectionProperties.fqdns" \
  --output tsv)

DATA_ENDPOINT_FQDN=$(az network nic show \
  --ids $NETWORK_INTERFACE_ID \
  --query "ipConfigurations[?privateLinkConnectionProperties.requiredMemberName=='registry_data_westus'].privateLinkConnectionProperties.fqdns" \
  --output tsv)
```

### Step 2: Create A-Record Sets

```bash
# Create A-record set for registry endpoint
az network private-dns record-set a create \
  --name myregistry \
  --zone-name privatelink.azurecr.io \
  --resource-group myResourceGroup

# Create A-record set for data endpoint
az network private-dns record-set a create \
  --name myregistry.westus.data \
  --zone-name privatelink.azurecr.io \
  --resource-group myResourceGroup
```

### Step 3: Add A-Records

```bash
# Add registry A-record
az network private-dns record-set a add-record \
  --record-set-name myregistry \
  --zone-name privatelink.azurecr.io \
  --resource-group myResourceGroup \
  --ipv4-address $REGISTRY_PRIVATE_IP

# Add data endpoint A-record
az network private-dns record-set a add-record \
  --record-set-name myregistry.westus.data \
  --zone-name privatelink.azurecr.io \
  --resource-group myResourceGroup \
  --ipv4-address $DATA_ENDPOINT_PRIVATE_IP
```

## Automatic DNS Configuration (Portal)

When using the Azure portal:

1. During private endpoint creation, select **Yes** for **Integrate with private DNS zone**
2. Select `(New) privatelink.azurecr.io` or an existing zone
3. Azure automatically:
   - Creates the private DNS zone (if new)
   - Links it to your VNet
   - Creates required A-records
   - Updates records when endpoints change

## Custom DNS Server Configuration

### Scenario: On-Premises DNS with Azure Private Endpoint

If you have custom DNS servers (on-premises or in Azure), configure a forwarder to Azure DNS:

```
On-Premises DNS Server
         |
         | Forward queries for
         | privatelink.azurecr.io
         v
+---------------------------+
|    Azure DNS Forwarder    |
|    (168.63.129.16)        |
+---------------------------+
         |
         v
+---------------------------+
|  Azure Private DNS Zone   |
|  privatelink.azurecr.io   |
+---------------------------+
```

### Windows DNS Server Configuration

1. Open DNS Manager
2. Right-click **Conditional Forwarders**
3. Select **New Conditional Forwarder**
4. Configure:
   - DNS Domain: `privatelink.azurecr.io`
   - IP Address: `168.63.129.16` (Azure DNS)

### Linux DNS Configuration (BIND)

Add to your BIND configuration:

```
zone "privatelink.azurecr.io" {
    type forward;
    forward only;
    forwarders { 168.63.129.16; };
};
```

### Azure Firewall DNS Proxy

If using Azure Firewall as DNS proxy:

1. Enable DNS proxy in Azure Firewall
2. Configure VNet DNS servers to point to Azure Firewall private IP
3. Azure Firewall forwards queries appropriately

## DNS Configuration for Hybrid Scenarios

### ExpressRoute/VPN Connected Networks

```
+-------------------+        +------------------------+
| On-Premises       |        |      Azure VNet        |
|                   |        |                        |
| +---------------+ | ExpressRoute/VPN +------------+ |
| | DNS Server    |<-------->| Azure DNS    |        |
| +---------------+ |        | (168.63.129.16)        |
|        |         |        |        |              |
|        v         |        |        v              |
| Forward to Azure |        | Private DNS Zone      |
| for privatelink  |        | privatelink.azurecr.io|
+-------------------+        +------------------------+
```

### Configuration Steps for On-Premises Access

1. **Create DNS Forwarder VM in Azure** (optional but recommended):

```bash
# Deploy a DNS forwarder VM or use Azure Private DNS Resolver
az network private-resolver create \
  --name myDnsResolver \
  --resource-group myResourceGroup \
  --location westus2 \
  --id /subscriptions/{subscription-id}/resourceGroups/myResourceGroup/providers/Microsoft.Network/virtualNetworks/myVNet
```

2. **Configure On-Premises DNS** to forward to Azure:
   - Forward `privatelink.azurecr.io` to Azure DNS or forwarder VM
   - Forward `azurecr.io` to Azure DNS (for CNAME resolution)

3. **Test Resolution** from on-premises:

```bash
nslookup myregistry.azurecr.io
# Should resolve to private IP
```

## Multi-Region High Availability DNS

### Best Practice: Separate Resource Groups per Region

For geo-replicated registries with private endpoints in multiple regions:

```
Region: West US
+--------------------------------+
| Resource Group: rg-westus      |
|                                |
| - Private DNS Zone             |
|   privatelink.azurecr.io       |
| - VNet Link (westus-vnet)      |
| - Private Endpoint             |
+--------------------------------+

Region: East US
+--------------------------------+
| Resource Group: rg-eastus      |
|                                |
| - Private DNS Zone             |
|   privatelink.azurecr.io       |
| - VNet Link (eastus-vnet)      |
| - Private Endpoint             |
+--------------------------------+
```

**Important**: Using separate resource groups per region with separate private DNS zones prevents DNS resolution conflicts and ensures regional isolation.

## DNS Validation and Testing

### Validate from Within VNet

SSH into a VM within the VNet and run:

```bash
# Using dig
dig myregistry.azurecr.io

# Expected output:
# ;; ANSWER SECTION:
# myregistry.azurecr.io.      1783  IN  CNAME  myregistry.privatelink.azurecr.io.
# myregistry.privatelink.azurecr.io. 10 IN A   10.0.0.5

# Using nslookup
nslookup myregistry.azurecr.io

# Expected output:
# Name:    myregistry.privatelink.azurecr.io
# Address: 10.0.0.5
```

### Validate Data Endpoint Resolution

```bash
dig myregistry.westus.data.azurecr.io

# Expected output showing private IP for data endpoint
```

### Compare with Public Resolution

From outside the VNet (public internet):

```bash
dig myregistry.azurecr.io

# Public resolution output (different from private):
# myregistry.azurecr.io.        2881  IN  CNAME  myregistry.privatelink.azurecr.io.
# myregistry.privatelink.azurecr.io.  2881  IN  CNAME  xxxx.xx.azcr.io.
# xxxx.xx.azcr.io.             300   IN  CNAME  xxxx-xxx-reg.trafficmanager.net.
# xxxx-xxx-reg.trafficmanager.net. 300 IN CNAME xxxx.westus.cloudapp.azure.com.
# xxxx.westus.cloudapp.azure.com.  10  IN  A     20.45.122.144  (Public IP)
```

### Using az acr check-health with VNet

```bash
# Verify DNS configuration from within VNet
az acr check-health \
  --name myregistry \
  --vnet myVNet
```

## Troubleshooting DNS Issues

### Issue: Registry FQDN Resolves to Public IP

**Symptoms:**
- `dig myregistry.azurecr.io` returns public IP
- Registry operations fail with network/timeout errors

**Solutions:**

1. Verify private DNS zone exists:
```bash
az network private-dns zone list --output table
```

2. Verify VNet link:
```bash
az network private-dns link vnet list \
  --zone-name privatelink.azurecr.io \
  --resource-group myResourceGroup \
  --output table
```

3. Verify A-records:
```bash
az network private-dns record-set a list \
  --zone-name privatelink.azurecr.io \
  --resource-group myResourceGroup \
  --output table
```

### Issue: "Unresolvable Host" Error

**Symptoms:**
- Error message: `unresolvable host`
- Often occurs after deleting a private endpoint

**Solution:**
Delete the VNet link to the private DNS zone and recreate if necessary:

```bash
# Delete old link
az network private-dns link vnet delete \
  --zone-name privatelink.azurecr.io \
  --resource-group myResourceGroup \
  --name myDNSLink

# Recreate with new endpoint
az network private-dns link vnet create \
  --zone-name privatelink.azurecr.io \
  --resource-group myResourceGroup \
  --name myDNSLink \
  --virtual-network myVNet \
  --registration-enabled false
```

### Issue: Missing Data Endpoint Records

**Symptoms:**
- Authentication succeeds but image pull/push fails
- Layer download times out

**Solution:**
Ensure A-records exist for all data endpoints:

```bash
# List all IP configurations on the private endpoint NIC
az network nic show \
  --ids $NETWORK_INTERFACE_ID \
  --query "ipConfigurations[].{Name:name, IP:privateIPAddress, Member:privateLinkConnectionProperties.requiredMemberName}" \
  --output table
```

Create missing records for each `registry_data_*` member.

### Issue: Geo-Replica Data Endpoint Not Resolving

**Symptoms:**
- Pull from replica region fails
- DNS doesn't resolve replica data endpoint

**Solution:**
When adding geo-replicas, manually add DNS records:

```bash
# For each replica region
az network private-dns record-set a create \
  --name myregistry.eastus.data \
  --zone-name privatelink.azurecr.io \
  --resource-group myResourceGroup

az network private-dns record-set a add-record \
  --record-set-name myregistry.eastus.data \
  --zone-name privatelink.azurecr.io \
  --resource-group myResourceGroup \
  --ipv4-address <replica-data-endpoint-private-ip>
```

## DNS Configuration Options Summary

| Scenario | DNS Solution | Configuration |
|----------|--------------|---------------|
| Single VNet, Azure-managed | Azure Private DNS Zone | Automatic with portal integration |
| Multiple VNets, same region | Private DNS Zone + multiple VNet links | Link each VNet to same zone |
| Multi-region deployment | Separate DNS zones per region | Independent zones, avoid conflicts |
| On-premises access | Conditional forwarder | Forward `privatelink.azurecr.io` to Azure DNS |
| Hybrid with Azure Firewall | DNS Proxy | Azure Firewall forwards DNS queries |
| Custom DNS server | Server-level forwarder | Forward to 168.63.129.16 |

## Sources

- [Azure Private Endpoint DNS Configuration](/azure/private-link/private-endpoint-dns)
- [ACR Private Link Documentation](https://aka.ms/acr/privatelink)
- [Azure Private DNS Overview](/azure/dns/private-dns-overview)
- [Troubleshoot Azure Private Endpoint Connectivity](/azure/private-link/troubleshoot-private-endpoint-connectivity)

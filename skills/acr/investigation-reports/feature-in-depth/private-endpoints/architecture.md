# Azure Container Registry Private Endpoints - Architecture

## Overview

Azure Container Registry private endpoints use Azure Private Link to establish secure, private connectivity between your virtual network and the registry service. This architecture ensures that all traffic remains on the Microsoft backbone network.

## Architecture Diagram

```
+------------------------------------------------------------------+
|                        Azure Virtual Network                      |
|                                                                   |
|  +-------------------+      +----------------------------------+  |
|  |   Client VM /     |      |      Private Endpoint            |  |
|  |   AKS Node /      |----->|  (NIC with Private IPs)          |  |
|  |   Container Host  |      |                                  |  |
|  +-------------------+      |  10.0.0.5 -> registry endpoint   |  |
|                             |  10.0.0.6 -> data endpoint       |  |
|                             +----------------------------------+  |
|                                          |                        |
|  +----------------------------------+    |                        |
|  |    Private DNS Zone             |    |                        |
|  |    privatelink.azurecr.io       |    |                        |
|  |                                 |    |                        |
|  |  myregistry -> 10.0.0.5        |    |                        |
|  |  myregistry.westus.data        |    |                        |
|  |          -> 10.0.0.6           |    |                        |
|  +----------------------------------+    |                        |
+------------------------------------------|------------------------+
                                           |
                              Azure Private Link
                              (Microsoft Backbone)
                                           |
                                           v
+------------------------------------------------------------------+
|                  Azure Container Registry                         |
|                      (Premium SKU)                                |
|                                                                   |
|  +----------------------------+  +-----------------------------+  |
|  |   Registry REST API        |  |   Data Storage (Blobs)      |  |
|  |   myregistry.azurecr.io    |  |   myregistry.westus.data    |  |
|  |                            |  |        .azurecr.io          |  |
|  |   - Authentication         |  |                             |  |
|  |   - Manifest operations    |  |   - Image layers            |  |
|  |   - Tag management         |  |   - Blob storage            |  |
|  +----------------------------+  +-----------------------------+  |
+------------------------------------------------------------------+
```

## Component Details

### 1. Private Endpoint

The private endpoint is a network interface that connects your virtual network to the Azure Container Registry service.

**Characteristics:**
- Deployed in a subnet within your virtual network
- Receives private IP addresses from your VNet's address space
- Appears as a standard network interface

**IP Allocation:**
- One IP for the registry REST API endpoint
- One IP for the data endpoint (per region)
- Additional IPs for geo-replicated regions

### 2. Network Interface Configuration

Each private endpoint creates IP configurations for:

| Member Name | FQDN Pattern | Purpose |
|-------------|--------------|---------|
| `registry` | `<registry>.azurecr.io` | REST API, authentication |
| `registry_data_<region>` | `<registry>.<region>.data.azurecr.io` | Image layer storage |

### 3. Private DNS Zone

The private DNS zone `privatelink.azurecr.io` enables DNS resolution of the registry FQDN to private IP addresses.

**DNS Resolution Flow:**

```
1. Client requests: myregistry.azurecr.io
                    |
                    v
2. Public DNS returns: myregistry.privatelink.azurecr.io (CNAME)
                    |
                    v
3. Private DNS Zone resolves: 10.0.0.5 (A record)
                    |
                    v
4. Client connects to private IP in VNet
```

**Required DNS Records:**

| Record Name | Type | Value |
|-------------|------|-------|
| `myregistry` | A | `10.0.0.5` (registry private IP) |
| `myregistry.westus.data` | A | `10.0.0.6` (data endpoint private IP) |

### 4. VNet Link

Associates the private DNS zone with your virtual network, enabling DNS queries from VNet resources to resolve private endpoint addresses.

## Network Flow

### Image Pull Operation

```
1. docker pull myregistry.azurecr.io/image:tag
   |
   v
2. DNS Resolution (Private DNS Zone)
   myregistry.azurecr.io -> 10.0.0.5
   |
   v
3. Authentication Request (via Private Link)
   Client -> 10.0.0.5 -> Azure Backbone -> ACR REST API
   |
   v
4. Layer Download (via Private Link)
   myregistry.westus.data.azurecr.io -> 10.0.0.6
   Client -> 10.0.0.6 -> Azure Backbone -> ACR Storage
```

### Image Push Operation

```
1. docker push myregistry.azurecr.io/image:tag
   |
   v
2. DNS Resolution (Private DNS Zone)
   |
   v
3. Authentication & Manifest Upload
   Client -> Private Endpoint -> ACR REST API
   |
   v
4. Layer Upload
   Client -> Private Endpoint -> ACR Data Endpoint -> Blob Storage
```

## Geo-Replicated Registry Architecture

For geo-replicated registries, each region requires additional configuration:

```
+------------------------------------------------------------------+
|                Virtual Network (Region: West US)                  |
|                                                                   |
|  +----------------------------------+                             |
|  |       Private Endpoint           |                             |
|  |                                  |                             |
|  |  IP 1: registry endpoint         |                             |
|  |  IP 2: westus data endpoint      |                             |
|  |  IP 3: eastus data endpoint      | <-- For geo-replica        |
|  +----------------------------------+                             |
+------------------------------------------------------------------+
                         |
          Azure Private Link (Backbone)
                         |
                         v
+------------------------------------------------------------------+
|              Azure Container Registry (Premium)                   |
|                                                                   |
|  +------------------------+    +-------------------------+        |
|  |    West US (Primary)   |    |    East US (Replica)    |        |
|  |                        |    |                         |        |
|  |  - Registry API        |<-->|  - Data storage         |        |
|  |  - Data storage        |    |  - Replicated content   |        |
|  +------------------------+    +-------------------------+        |
+------------------------------------------------------------------+
```

## On-Premises Connectivity Architecture

### Via Azure ExpressRoute

```
+-------------------+        +----------------------+
|   On-Premises     |        |   Azure VNet         |
|   Network         |        |                      |
|                   |        |  +----------------+  |
|  +------------+   |   ExpressRoute   |  Private   |  |
|  |  Docker    |---|----|  Private    |  Endpoint  |  |
|  |  Client    |   |   |   Peering    |            |  |
|  +------------+   |        +----------------+  |
|                   |        |                      |
|  +------------+   |        |  +----------------+  |
|  |  Custom    |   |        |  | Private DNS    |  |
|  |  DNS       |----------->|  | Zone           |  |
|  +------------+   |        |  +----------------+  |
+-------------------+        +----------------------+
```

### Via VPN Gateway

```
+-------------------+        +----------------------+
|   On-Premises     |        |   Azure VNet         |
|   Network         |        |                      |
|                   |        |  +----------------+  |
|  +------------+   |  Site-to-Site  |  Private   |  |
|  |  Container |---|----|   VPN      |  Endpoint  |  |
|  |  Host      |   |   |   Gateway   |            |  |
|  +------------+   |        +----------------+  |
+-------------------+        +----------------------+
```

## Security Architecture

### Network Security Groups (NSG)

**Important**: You must disable network policies in the subnet to allow private endpoint creation:

```bash
az network vnet subnet update \
  --name mySubnet \
  --vnet-name myVNet \
  --resource-group myResourceGroup \
  --disable-private-endpoint-network-policies
```

### Traffic Flow Security

| Traffic Type | Path | Security |
|--------------|------|----------|
| VNet to Registry | Private Link | Azure Backbone, encrypted |
| On-prem to Registry | ExpressRoute/VPN + Private Link | Encrypted tunnel |
| Public Internet | Blocked (when disabled) | No exposure |

## High Availability Considerations

### Multi-Region Private Endpoint

For high availability, deploy private endpoints in multiple regions with geo-replicated registries:

```
+------------------+     +------------------+
| VNet Region A    |     | VNet Region B    |
|                  |     |                  |
| +-------------+  |     | +-------------+  |
| | Private EP  |  |     | | Private EP  |  |
| +-------------+  |     | +-------------+  |
+----------|-------+     +----------|-------+
           |                        |
           v                        v
+------------------------------------------+
|     Geo-Replicated ACR (Premium)         |
|                                          |
|  Region A Replica    Region B Replica    |
+------------------------------------------+
```

### Best Practices

1. **Separate Resource Groups**: Use separate resource groups per region for private DNS zones to avoid DNS resolution conflicts
2. **Zone Redundancy**: Enable zone redundancy in the registry for regional high availability
3. **DNS Configuration**: Use Azure Private DNS zones for automatic DNS updates

## Integration with Azure Services

### AKS Integration

```
+--------------------------------------------------+
|                  Azure VNet                       |
|                                                   |
|  +-------------------+    +-------------------+   |
|  |    AKS Cluster    |    |  Private Endpoint |   |
|  |                   |--->|                   |   |
|  |  kubelet pulls    |    |  ACR Registry     |   |
|  |  images           |    |                   |   |
|  +-------------------+    +-------------------+   |
+--------------------------------------------------+
```

### Azure Container Instances

ACI can access private endpoint-enabled registries using managed identity authentication with the "allow trusted services" setting enabled.

## Limitations and Constraints

1. **No Bastion Support**: Azure Bastion endpoints are not supported with private endpoints
2. **Service Endpoint Conflict**: Cannot use both private endpoints and service endpoints simultaneously
3. **Cross-Subscription**: Requires resource provider registration in the subscription containing the private link
4. **Subnet Requirements**: Must disable network policies in the subnet

## Sources

- [Azure Private Link Architecture](/azure/private-link/private-link-overview)
- [ACR Private Link Documentation](https://aka.ms/acr/privatelink)
- [Private Endpoint DNS Configuration](/azure/private-link/private-endpoint-dns)

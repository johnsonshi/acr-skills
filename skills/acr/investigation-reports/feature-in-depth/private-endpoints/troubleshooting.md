# Azure Container Registry Private Endpoints - Troubleshooting Guide

## Overview

This guide provides comprehensive troubleshooting steps for common issues encountered when using private endpoints with Azure Container Registry.

## Diagnostic Tools

### 1. az acr check-health

The primary diagnostic tool for ACR connectivity issues:

```bash
# Basic health check
az acr check-health --name myregistry

# With VNet DNS validation
az acr check-health --name myregistry --vnet myVNet

# Continue on errors to see all issues
az acr check-health --name myregistry --ignore-errors --yes
```

### 2. DNS Resolution Tools

```bash
# Using dig
dig myregistry.azurecr.io

# Using nslookup
nslookup myregistry.azurecr.io

# Check data endpoint
dig myregistry.westus.data.azurecr.io
```

### 3. Network Connectivity Tests

```bash
# Test HTTPS connectivity
curl -v https://myregistry.azurecr.io/v2/

# Test with specific IP
curl -v --resolve myregistry.azurecr.io:443:10.0.0.5 https://myregistry.azurecr.io/v2/
```

## Common Issues and Solutions

---

### Issue 1: Unable to Push or Pull Images - DNS Resolution Fails

**Symptoms:**
- `dial tcp: lookup myregistry.azurecr.io: no such host`
- `Could not connect to the registry login server`
- DNS resolves to public IP instead of private IP

**Diagnosis:**

```bash
# Check DNS resolution from within VNet
dig myregistry.azurecr.io

# Expected (private endpoint configured):
# myregistry.azurecr.io -> myregistry.privatelink.azurecr.io -> 10.0.0.5

# Incorrect (resolving to public):
# myregistry.azurecr.io -> myregistry.privatelink.azurecr.io -> xxxx.trafficmanager.net -> public IP
```

**Solutions:**

1. **Verify Private DNS Zone exists:**
```bash
az network private-dns zone list \
  --resource-group myResourceGroup \
  --output table
```

2. **Verify VNet link:**
```bash
az network private-dns link vnet list \
  --zone-name privatelink.azurecr.io \
  --resource-group myResourceGroup \
  --output table
```

3. **Verify A-records:**
```bash
az network private-dns record-set a list \
  --zone-name privatelink.azurecr.io \
  --resource-group myResourceGroup \
  --output table
```

4. **Recreate missing DNS records:**
```bash
# Get the private IPs
NIC_ID=$(az network private-endpoint show --name myEndpoint -g myRG --query 'networkInterfaces[0].id' -o tsv)
REGISTRY_IP=$(az network nic show --ids $NIC_ID --query "ipConfigurations[?privateLinkConnectionProperties.requiredMemberName=='registry'].privateIPAddress" -o tsv)

# Create A-record
az network private-dns record-set a add-record \
  --record-set-name myregistry \
  --zone-name privatelink.azurecr.io \
  --resource-group myResourceGroup \
  --ipv4-address $REGISTRY_IP
```

---

### Issue 2: Authentication Succeeds but Image Pull Fails

**Symptoms:**
- `az acr login` succeeds
- `docker pull` times out or fails with layer download errors
- Error: `Client.Timeout exceeded while awaiting headers`

**Cause:** Data endpoint DNS not configured or missing.

**Diagnosis:**

```bash
# Verify data endpoint resolution
dig myregistry.westus.data.azurecr.io

# Check all private endpoint IPs
NIC_ID=$(az network private-endpoint show --name myEndpoint -g myRG --query 'networkInterfaces[0].id' -o tsv)
az network nic show --ids $NIC_ID --query "ipConfigurations[].{Member:privateLinkConnectionProperties.requiredMemberName, IP:privateIPAddress}" -o table
```

**Solution:**

```bash
# Get data endpoint IP
DATA_IP=$(az network nic show --ids $NIC_ID \
  --query "ipConfigurations[?privateLinkConnectionProperties.requiredMemberName=='registry_data_westus'].privateIPAddress" -o tsv)

# Create DNS record for data endpoint
az network private-dns record-set a create \
  --name myregistry.westus.data \
  --zone-name privatelink.azurecr.io \
  --resource-group myResourceGroup

az network private-dns record-set a add-record \
  --record-set-name myregistry.westus.data \
  --zone-name privatelink.azurecr.io \
  --resource-group myResourceGroup \
  --ipv4-address $DATA_IP
```

---

### Issue 3: "host is not reachable" Error

**Symptoms:**
- Error: `host is not reachable`
- Cannot access registry configured with private endpoint

**Causes:**
1. Client not in the VNet
2. NSG blocking traffic
3. Missing private endpoint
4. DNS misconfiguration

**Diagnosis:**

```bash
# Verify client is in VNet
# (from client VM)
hostname -I

# Check NSG rules
az network nsg rule list \
  --nsg-name myNSG \
  --resource-group myResourceGroup \
  --output table

# Check private endpoint status
az network private-endpoint show \
  --name myEndpoint \
  --resource-group myResourceGroup \
  --query 'provisioningState'
```

**Solutions:**

1. **Ensure client is in the VNet** or has connectivity via ExpressRoute/VPN

2. **Check subnet network policies:**
```bash
az network vnet subnet show \
  --name mySubnet \
  --vnet-name myVNet \
  --resource-group myResourceGroup \
  --query 'privateEndpointNetworkPolicies'

# Should be "Disabled"
```

3. **Verify private endpoint connection is approved:**
```bash
az acr private-endpoint-connection list \
  --registry-name myregistry \
  --query "[].{Name:name, Status:privateLinkServiceConnectionState.status}" -o table
```

---

### Issue 4: "unresolvable host" Error After Deleting Private Endpoint

**Symptoms:**
- Error: `unresolvable host`
- Occurs after deleting a private endpoint
- DNS still points to old/invalid IPs

**Cause:** VNet link to private DNS zone still exists with stale records.

**Solution:**

```bash
# Delete old DNS records
az network private-dns record-set a delete \
  --name myregistry \
  --zone-name privatelink.azurecr.io \
  --resource-group myResourceGroup \
  --yes

az network private-dns record-set a delete \
  --name myregistry.westus.data \
  --zone-name privatelink.azurecr.io \
  --resource-group myResourceGroup \
  --yes

# Optionally delete and recreate VNet link
az network private-dns link vnet delete \
  --zone-name privatelink.azurecr.io \
  --resource-group myResourceGroup \
  --name myVNetLink \
  --yes
```

---

### Issue 5: 403 Forbidden Error

**Symptoms:**
- `Error response from daemon: login attempt failed with status: 403 Forbidden`
- `Looks like you don't have access to registry`

**Causes:**
1. Public network access disabled and client not using private endpoint
2. IP firewall rules blocking access
3. RBAC permissions issue
4. HTTPS proxy not configured correctly

**Diagnosis:**

```bash
# Check public network access setting
az acr show --name myregistry --query 'publicNetworkAccess'

# Check network rules
az acr network-rule list --name myregistry

# Check default action
az acr show --name myregistry --query 'networkRuleSet.defaultAction'
```

**Solutions:**

1. **For clients that should use private endpoint:**
   - Ensure client is in VNet or connected via ExpressRoute/VPN
   - Verify DNS resolution returns private IP

2. **For clients that need public access:**
```bash
# Add IP to allowed list
az acr network-rule add --name myregistry --ip-address <client-public-ip>

# Or enable public access
az acr update --name myregistry --public-network-enabled true
```

3. **For HTTPS proxy issues:**
   - Configure Docker daemon for proxy: `/etc/systemd/system/docker.service.d/http-proxy.conf`
   - Ensure proxy allows traffic to registry FQDN

---

### Issue 6: Geo-Replicated Registry - Replica Pull Fails

**Symptoms:**
- Pulls from primary region work
- Pulls from replica region fail
- Data endpoint for replica region not resolving

**Cause:** Missing DNS records for geo-replica data endpoints.

**Diagnosis:**

```bash
# List all replicas
az acr replication list --registry myregistry -o table

# Check DNS for replica data endpoint
dig myregistry.eastus.data.azurecr.io
```

**Solution:**

```bash
# Get replica data endpoint IP
REPLICA_DATA_IP=$(az network nic show --ids $NIC_ID \
  --query "ipConfigurations[?privateLinkConnectionProperties.requiredMemberName=='registry_data_eastus'].privateIPAddress" -o tsv)

# Create DNS record
az network private-dns record-set a create \
  --name myregistry.eastus.data \
  --zone-name privatelink.azurecr.io \
  --resource-group myResourceGroup

az network private-dns record-set a add-record \
  --record-set-name myregistry.eastus.data \
  --zone-name privatelink.azurecr.io \
  --resource-group myResourceGroup \
  --ipv4-address $REPLICA_DATA_IP
```

---

### Issue 7: Private Endpoint Connection Stuck in "Pending"

**Symptoms:**
- Private endpoint created but connection shows "Pending"
- Cannot use the private endpoint

**Cause:** Connection requires manual approval or permission issue.

**Diagnosis:**

```bash
az acr private-endpoint-connection list \
  --registry-name myregistry \
  --query "[].{Name:name, Status:privateLinkServiceConnectionState.status, Description:privateLinkServiceConnectionState.description}" -o table
```

**Solution:**

```bash
# Approve the connection
az acr private-endpoint-connection approve \
  --registry-name myregistry \
  --name <connection-name> \
  --description "Approved"
```

---

### Issue 8: ACR Tasks / az acr build Fails with Private Endpoint

**Symptoms:**
- `az acr build` commands fail
- ACR Tasks cannot push/pull images
- Error when public access is disabled

**Cause:** ACR Tasks uses public IPs for outbound requests. When public access is disabled, Tasks cannot communicate with the registry.

**Solutions:**

1. **Use Dedicated Agent Pools:**
```bash
az acr agentpool create \
  --name myAgentPool \
  --registry myregistry \
  --resource-group myResourceGroup
```

2. **Add ACR Service Tag IPs to Firewall:**
   - Add regional `AzureContainerRegistry` service tag IPs to allowed list
   - Reference: [Azure IP Ranges](https://www.microsoft.com/download/details.aspx?id=56519)

3. **Keep a subset of public access:**
```bash
# Allow specific IPs for Tasks
az acr network-rule add \
  --name myregistry \
  --ip-address <tasks-outbound-ip>
```

---

### Issue 9: Microsoft Defender for Cloud Cannot Scan Images

**Symptoms:**
- Vulnerability scanning not working
- No scan results in Defender for Cloud
- Scanning limitation with network-restricted registry

**Cause:** Microsoft Defender for Cloud cannot scan images in registries with:
- Private endpoints (as sole access method)
- Service endpoints
- Public IP access rules blocking Defender

**Solutions:**

1. **Enable Trusted Services:**
```bash
az acr update --name myregistry --allow-trusted-services true
```

2. **Allow Defender access via network rules** (if using IP rules)

3. **Accept limitation**: Some network configurations prevent vulnerability scanning

---

### Issue 10: On-Premises Client Cannot Access Private Endpoint

**Symptoms:**
- On-premises clients cannot resolve or connect to registry
- Works from Azure VMs but not on-premises

**Cause:** DNS not configured to forward to Azure Private DNS.

**Solutions:**

1. **Configure on-premises DNS forwarder:**
   - Forward `privatelink.azurecr.io` zone to Azure DNS (168.63.129.16)
   - Or forward to a DNS forwarder VM in Azure

2. **Verify ExpressRoute/VPN connectivity:**
```bash
# From on-premises
traceroute <private-endpoint-ip>
telnet <private-endpoint-ip> 443
```

3. **Use Azure Private DNS Resolver:**
```bash
az network private-resolver create \
  --name myResolver \
  --resource-group myResourceGroup \
  --location westus2 \
  --vnet-id /subscriptions/.../virtualNetworks/myVNet
```

---

## Troubleshooting Checklist

### Quick Diagnostic Steps

1. **Verify registry SKU is Premium:**
```bash
az acr show --name myregistry --query 'sku.name'
```

2. **Check private endpoint exists and is provisioned:**
```bash
az network private-endpoint show --name myEndpoint -g myRG --query 'provisioningState'
```

3. **Verify connection is approved:**
```bash
az acr private-endpoint-connection list --registry-name myregistry --query "[].privateLinkServiceConnectionState.status"
```

4. **Test DNS resolution from client:**
```bash
dig myregistry.azurecr.io
dig myregistry.westus.data.azurecr.io
```

5. **Verify private DNS zone and VNet link:**
```bash
az network private-dns zone show -g myRG -n privatelink.azurecr.io
az network private-dns link vnet list -g myRG -z privatelink.azurecr.io
```

6. **Check A-records exist:**
```bash
az network private-dns record-set a list -g myRG -z privatelink.azurecr.io -o table
```

7. **Test connectivity:**
```bash
az acr check-health --name myregistry --vnet myVNet
```

### Log Collection

Enable diagnostic logging for troubleshooting:

```bash
# Enable resource logs
az monitor diagnostic-settings create \
  --name myDiagSettings \
  --resource /subscriptions/.../registries/myregistry \
  --logs '[{"category":"ContainerRegistryRepositoryEvents","enabled":true},{"category":"ContainerRegistryLoginEvents","enabled":true}]' \
  --workspace /subscriptions/.../workspaces/myWorkspace
```

Query logs:
```kusto
ContainerRegistryLoginEvents
| where TimeGenerated > ago(1h)
| where ResultType != "200"
| project TimeGenerated, CallerIpAddress, LoginServer, ResultType, ResultDescription
```

## Additional Resources

- [Troubleshoot Network Issues with Registry](https://docs.microsoft.com/azure/container-registry/container-registry-troubleshoot-access)
- [Troubleshoot Azure Private Endpoint Connectivity](/azure/private-link/troubleshoot-private-endpoint-connectivity)
- [ACR Health Check Error Reference](https://docs.microsoft.com/azure/container-registry/container-registry-health-error-reference)
- [ACR FAQ](https://docs.microsoft.com/azure/container-registry/container-registry-faq)

## Getting Help

If issues persist after troubleshooting:

1. **Community Support**: [Stack Overflow - Azure Container Registry](https://aka.ms/acr/stack-overflow)
2. **Feature Requests**: [Azure Feedback](https://aka.ms/acr/uservoice)
3. **GitHub Issues**: [ACR GitHub](https://aka.ms/acr/issues)
4. **Azure Support**: [Create Support Ticket](https://aka.ms/acr/support/create-ticket)

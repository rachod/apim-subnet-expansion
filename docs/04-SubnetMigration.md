# Subnet Migration

[← Previous: Backup Configuration](03-BackupConfiguration.md) | [Next: Pre-Migration Checklist →](06-PreMigrationChecklist.md)

---

> ⚠️ **IMPORTANT:** This operation causes **30-45 minutes of downtime** for the APIM gateway. Plan this during a maintenance window.

---

## Create the New Larger Subnet

```bash
# Create the LARGE subnet (/25 - 123 usable IPs, sufficient for future growth)
az network vnet subnet create \
  --resource-group $RG \
  --vnet-name $VNET_NAME \
  --name $SUBNET_LARGE \
  --address-prefixes $SUBNET_LARGE_PREFIX \
  --network-security-group $NSG_NAME
```

> **Note:** The new subnet must be in the **same VNet** as the current one. Ensure the NSG attached has the same APIM-required rules (port 3443 inbound, Storage/SQL/KeyVault outbound).

---

## Migrate APIM to the New Subnet

```bash
# Get the new subnet resource ID
SUBNET_LARGE_ID=$(az network vnet subnet show \
  --resource-group $RG \
  --vnet-name $VNET_NAME \
  --name $SUBNET_LARGE \
  --query id -o tsv)

# Migrate APIM to the new larger subnet
az apim update \
  --resource-group $RG \
  --name $APIM_NAME \
  --set virtualNetworkConfiguration.subnetResourceId=$SUBNET_LARGE_ID
```

> ⏱️ **Note:** Subnet migration takes approximately 30-45 minutes.

---

## Wait for Migration to Complete

Poll the provisioning state until it returns `Succeeded`:

```bash
# Poll every 60 seconds
while true; do
  STATE=$(az apim show --resource-group $RG --name $APIM_NAME \
    --query provisioningState -o tsv)
  echo "$(date +%H:%M:%S) - provisioningState: $STATE"
  if [ "$STATE" = "Succeeded" ]; then
    echo "✓ Migration complete!"
    break
  fi
  sleep 60
done
```

---

## Verify Migration

```bash
# Verify APIM is now in the new subnet
az apim show \
  --resource-group $RG \
  --name $APIM_NAME \
  --query "{name:name, provisioningState:provisioningState, privateIPs:privateIPAddresses, subnet:virtualNetworkConfiguration.subnetResourceId}" \
  -o json

# Verify the new subnet has APIM IPs
az network vnet subnet show \
  --resource-group $RG \
  --vnet-name $VNET_NAME \
  --name $SUBNET_LARGE \
  --query "{name:name, addressPrefix:addressPrefix, ipConfigCount:length(ipConfigurations || \`[]\`)}" \
  -o json
```

---

## Verify Network Connectivity

```bash
# Check all APIM dependencies are reachable from new subnet
TOKEN=$(az account get-access-token --query accessToken -o tsv)
SUB_ID=$(az account show --query id -o tsv)

az rest --method get \
  --url "/subscriptions/$SUB_ID/resourceGroups/$RG/providers/Microsoft.ApiManagement/service/$APIM_NAME/networkStatus?api-version=2022-08-01" \
  --query "[].networkStatus.connectivityStatus[].{name:name, status:status}" \
  -o table
```

All entries should show `status: success`.

---

## Post-Migration: Update DNS and Routing

Since APIM's private IP changes after subnet migration, update these:

**Update Private DNS Zone:**

```bash
# Get the new private IP
NEW_PRIVATE_IP=$(az apim show --resource-group $RG --name $APIM_NAME \
  --query "privateIPAddresses[0]" -o tsv)

echo "New APIM Private IP: $NEW_PRIVATE_IP"

# Update A record in Private DNS Zone
az network private-dns record-set a update \
  --resource-group $RG \
  --zone-name "azure-api.net" \
  --name $APIM_NAME \
  --set aRecords[0].ipv4Address=$NEW_PRIVATE_IP
```

**Update Application Gateway backend (if applicable):**

```bash
az network application-gateway address-pool update \
  --resource-group $RG \
  --gateway-name $AGW_NAME \
  --name $BACKEND_POOL \
  --servers $NEW_PRIVATE_IP
```

---

> [Next: Pre-Migration Checklist →](06-PreMigrationChecklist.md)

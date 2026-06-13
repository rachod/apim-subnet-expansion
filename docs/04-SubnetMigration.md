# Subnet Migration

[← Previous: Backup Configuration](03-BackupConfiguration.md) | [Next: Pre-Migration Checklist →](06-PreMigrationChecklist.md)

---

> ⚠️ **IMPORTANT:** This operation causes **30-45 minutes of downtime** for the APIM gateway. Plan this during a maintenance window.

---

## Pre-Migration Validation

Run these checks to confirm all prerequisites are met before starting the migration:

```bash
echo "============================================================"
echo " APIM Subnet Migration - Pre-Flight Checks"
echo "============================================================"
echo ""

PASS=0
FAIL=0

# --- Check 1: APIM is in Succeeded state ---
echo "1. Checking APIM provisioning state..."
APIM_STATE=$(az apim show --resource-group $RG --name $APIM_NAME \
  --query provisioningState -o tsv 2>/dev/null)
if [ "$APIM_STATE" = "Succeeded" ]; then
  echo "   ✅ APIM state: $APIM_STATE"
  PASS=$((PASS+1))
else
  echo "   ❌ APIM state: $APIM_STATE (must be 'Succeeded' before migration)"
  FAIL=$((FAIL+1))
fi

echo ""

# --- Check 2: APIM is Premium SKU ---
echo "2. Checking APIM SKU..."
APIM_SKU=$(az apim show --resource-group $RG --name $APIM_NAME \
  --query sku.name -o tsv 2>/dev/null)
if [ "$APIM_SKU" = "Premium" ]; then
  echo "   ✅ SKU: $APIM_SKU"
  PASS=$((PASS+1))
else
  echo "   ❌ SKU: $APIM_SKU (VNet integration requires Premium)"
  FAIL=$((FAIL+1))
fi

echo ""

# --- Check 3: APIM is on stv2 platform ---
echo "3. Checking APIM platform version..."
PLATFORM=$(az apim show --resource-group $RG --name $APIM_NAME \
  --query platformVersion -o tsv 2>/dev/null)
if [ "$PLATFORM" = "stv2" ]; then
  echo "   ✅ Platform: $PLATFORM"
  PASS=$((PASS+1))
else
  echo "   ❌ Platform: $PLATFORM (stv1 must be migrated to stv2 first)"
  FAIL=$((FAIL+1))
fi

echo ""

# --- Check 4: Target subnet exists ---
echo "4. Checking target subnet exists..."
SUBNET_EXISTS=$(az network vnet subnet show --resource-group $RG \
  --vnet-name $VNET_NAME --name $SUBNET_LARGE \
  --query name -o tsv 2>/dev/null)
if [ "$SUBNET_EXISTS" = "$SUBNET_LARGE" ]; then
  echo "   ✅ Subnet found: $SUBNET_LARGE"
  PASS=$((PASS+1))
else
  echo "   ❌ Subnet '$SUBNET_LARGE' not found in VNet '$VNET_NAME'"
  FAIL=$((FAIL+1))
fi

echo ""

# --- Check 5: Target subnet has enough IPs ---
echo "5. Checking target subnet size..."
SUBNET_PREFIX=$(az network vnet subnet show --resource-group $RG \
  --vnet-name $VNET_NAME --name $SUBNET_LARGE \
  --query addressPrefix -o tsv 2>/dev/null)
CIDR=$(echo $SUBNET_PREFIX | cut -d'/' -f2)
TOTAL_IPS=$((2 ** (32 - CIDR)))
USABLE_IPS=$((TOTAL_IPS - 5))
CURRENT_CAPACITY=$(az apim show --resource-group $RG --name $APIM_NAME \
  --query sku.capacity -o tsv 2>/dev/null)
IPS_NEEDED=$(( (CURRENT_CAPACITY * 2) + 1 ))

if [ "$USABLE_IPS" -ge "$IPS_NEEDED" ]; then
  echo "   ✅ Subnet $SUBNET_PREFIX has $USABLE_IPS usable IPs (need $IPS_NEEDED for $CURRENT_CAPACITY units)"
  PASS=$((PASS+1))
else
  echo "   ❌ Subnet $SUBNET_PREFIX has only $USABLE_IPS usable IPs (need $IPS_NEEDED for $CURRENT_CAPACITY units)"
  FAIL=$((FAIL+1))
fi

echo ""

# --- Check 6: Target subnet is empty (no existing resources) ---
echo "6. Checking target subnet is empty..."
EXISTING_IPS=$(az network vnet subnet show --resource-group $RG \
  --vnet-name $VNET_NAME --name $SUBNET_LARGE \
  --query "length(ipConfigurations || \`[]\`)" -o tsv 2>/dev/null)
if [ "$EXISTING_IPS" = "0" ] || [ -z "$EXISTING_IPS" ]; then
  echo "   ✅ Subnet is empty (no existing IP allocations)"
  PASS=$((PASS+1))
else
  echo "   ⚠️  Subnet has $EXISTING_IPS existing IP allocations (may reduce available capacity)"
  PASS=$((PASS+1))
fi

echo ""

# --- Check 7: NSG is attached to target subnet ---
echo "7. Checking NSG on target subnet..."
SUBNET_NSG=$(az network vnet subnet show --resource-group $RG \
  --vnet-name $VNET_NAME --name $SUBNET_LARGE \
  --query "networkSecurityGroup.id" -o tsv 2>/dev/null)
if [ -n "$SUBNET_NSG" ] && [ "$SUBNET_NSG" != "None" ]; then
  NSG_SHORT=$(echo $SUBNET_NSG | awk -F'/' '{print $NF}')
  echo "   ✅ NSG attached: $NSG_SHORT"
  PASS=$((PASS+1))
else
  echo "   ❌ No NSG attached to target subnet (APIM requires NSG with port 3443)"
  FAIL=$((FAIL+1))
fi

echo ""

# --- Check 8: NSG has required APIM management rule (port 3443) ---
echo "8. Checking NSG has APIM management rule (port 3443)..."
if [ -n "$NSG_SHORT" ]; then
  RULE_3443=$(az network nsg rule list --resource-group $RG --nsg-name $NSG_SHORT \
    --query "[?destinationPortRanges[?contains(@,'3443')] || destinationPortRange=='3443'].name" \
    -o tsv 2>/dev/null)
  if [ -n "$RULE_3443" ]; then
    echo "   ✅ Port 3443 rule found: $RULE_3443"
    PASS=$((PASS+1))
  else
    echo "   ❌ No inbound rule for port 3443 (required for APIM management)"
    FAIL=$((FAIL+1))
  fi
else
  echo "   ❌ Cannot check - no NSG attached"
  FAIL=$((FAIL+1))
fi

echo ""

# --- Check 9: Target subnet is in same VNet as current ---
echo "9. Checking target subnet is in same VNet..."
CURRENT_SUBNET=$(az apim show --resource-group $RG --name $APIM_NAME \
  --query virtualNetworkConfiguration.subnetResourceId -o tsv 2>/dev/null)
CURRENT_VNET=$(echo $CURRENT_SUBNET | awk -F'/subnets/' '{print $1}')
TARGET_VNET="/subscriptions/$(az account show --query id -o tsv)/resourceGroups/$RG/providers/Microsoft.Network/virtualNetworks/$VNET_NAME"

if echo "$CURRENT_VNET" | grep -qi "$VNET_NAME"; then
  echo "   ✅ Same VNet: $VNET_NAME"
  PASS=$((PASS+1))
else
  echo "   ❌ Different VNet detected. Migration requires same VNet."
  FAIL=$((FAIL+1))
fi

echo ""

# --- Check 10: Backup exists ---
echo "10. Checking APIM backup exists..."
if [ -n "$BACKUP_STORAGE_ACCOUNT" ]; then
  BLOB_COUNT=$(az storage blob list --container-name $BACKUP_CONTAINER \
    --account-name $BACKUP_STORAGE_ACCOUNT --auth-mode login \
    --query "length([?contains(name,'apim-backup')])" -o tsv 2>/dev/null)
  if [ "$BLOB_COUNT" -gt 0 ] 2>/dev/null; then
    echo "   ✅ Backup found ($BLOB_COUNT backup blob(s) in storage)"
    PASS=$((PASS+1))
  else
    echo "   ❌ No backup found. Run backup before migration (see 03-BackupConfiguration.md)"
    FAIL=$((FAIL+1))
  fi
else
  echo "   ⚠️  BACKUP_STORAGE_ACCOUNT not set - cannot verify backup"
  FAIL=$((FAIL+1))
fi

echo ""
echo "============================================================"
echo " PRE-FLIGHT RESULTS: $PASS passed, $FAIL failed"
echo "============================================================"

if [ "$FAIL" -gt 0 ]; then
  echo ""
  echo " ❌ DO NOT PROCEED - Fix the failed checks above before migrating."
  echo ""
else
  echo ""
  echo " ✅ ALL CHECKS PASSED - Safe to proceed with migration."
  echo ""
fi
```

> **Note:** If any check fails, resolve the issue before proceeding. Common fixes:
> - **APIM not Succeeded:** Wait for any in-progress operation to complete
> - **stv1 platform:** Migrate to stv2 first (`az apim update --set platformVersion=stv2`)
> - **No NSG:** Attach NSG with APIM rules to the target subnet
> - **No backup:** Run [Backup Configuration](03-BackupConfiguration.md) steps first

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

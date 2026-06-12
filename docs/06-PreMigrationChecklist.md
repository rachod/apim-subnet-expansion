# Pre-Migration Checklist

[← Previous: Subnet Migration](04-SubnetMigration.md) | [↑ Back to README](../README.md)

---

## Production Migration Checklist

Use this checklist when performing the subnet migration in production.

| # | Phase | Action | Command | Owner |
|---|-------|--------|---------|-------|
| 1 | **Pre** | Confirm new subnet CIDR doesn't overlap with existing subnets | `az network vnet subnet list --resource-group $RG --vnet-name $VNET_NAME -o table` | Network Team |
| 2 | **Pre** | Create new subnet with proper NSG rules | `az network vnet subnet create --resource-group $RG --vnet-name $VNET_NAME --name $SUBNET_LARGE --address-prefixes $SUBNET_LARGE_PREFIX --network-security-group $NSG_NAME` | Network Team |
| 3 | **Pre** | Backup APIM configuration | `az apim backup --resource-group $RG --name $APIM_NAME --storage-account-name $BACKUP_STORAGE_ACCOUNT --storage-account-container $BACKUP_CONTAINER --backup-name $BACKUP_BLOB --storage-account-resource-group $RG` | Platform Team |
| 4 | **Pre** | Export individual configs (git-friendly) | See [Backup Configuration](03-BackupConfiguration.md#export-individual-configurations-git-friendly) | Platform Team |
| 5 | **Pre** | Schedule maintenance window (30-45 min downtime) | N/A (communication) | Operations |
| 6 | **Pre** | Notify dependent services of APIM IP change | N/A (communication) | App Team |
| 7 | **Execute** | Migrate APIM to new subnet | `az apim update --resource-group $RG --name $APIM_NAME --set virtualNetworkConfiguration.subnetResourceId=$SUBNET_LARGE_ID` | Platform Team |
| 8 | **Execute** | Wait for migration to complete | `az apim show --resource-group $RG --name $APIM_NAME --query provisioningState -o tsv` (poll until `Succeeded`) | Platform Team |
| 9 | **Verify** | Verify APIM health and connectivity | `az apim show --resource-group $RG --name $APIM_NAME --query "{state:provisioningState, privateIPs:privateIPAddresses, subnet:virtualNetworkConfiguration.subnetResourceId}" -o json` | Platform Team |
| 10 | **Verify** | Verify network connectivity status | `az rest --method get --url "/subscriptions/$SUB_ID/resourceGroups/$RG/providers/Microsoft.ApiManagement/service/$APIM_NAME/networkStatus?api-version=2022-08-01"` | Platform Team |
| 11 | **Post** | Scale to desired unit count | `az apim update --resource-group $RG --name $APIM_NAME --sku-capacity 8` | Platform Team |
| 12 | **Post** | Verify scaling success | `az apim show --resource-group $RG --name $APIM_NAME --query "{capacity:sku.capacity, provisioningState:provisioningState}" -o json` | Platform Team |
| 13 | **Post** | Update private DNS zones with new IP | `az network private-dns record-set a update --resource-group $RG --zone-name $DNS_ZONE --name $APIM_NAME --set aRecords[0].ipv4Address=$NEW_PRIVATE_IP` | Network Team |
| 14 | **Post** | Update Application Gateway backend (if applicable) | `az network application-gateway address-pool update --resource-group $RG --gateway-name $AGW_NAME --name $BACKEND_POOL --servers $NEW_PRIVATE_IP` | Network Team |
| 15 | **Post** | Update firewall rules referencing old subnet/IPs | `az network firewall network-rule collection list --resource-group $RG --firewall-name $FW_NAME -o table` | Security Team |
| 16 | **Post** | Delete old subnet (after validation) | `az network vnet subnet delete --resource-group $RG --vnet-name $VNET_NAME --name $SUBNET_SMALL` | Network Team |

---

## Key Considerations

1. **Downtime:** Subnet migration causes gateway downtime (~30-45 min based on our PoC). API calls will fail during this period.

2. **IP Address Change:** APIM private IP **will change** after migration. Update:
   - Private DNS zones
   - Application Gateway backend pools (if applicable)
   - Firewall rules
   - Any hardcoded IP references

3. **Subnet Size Recommendation:** Use **/25 or larger** for Premium APIM to allow future scaling up to 12+ units.

4. **Same VNet Required:** The new subnet must be in the **same VNet** as the current one (or you'll need a full redeployment).

5. **NSG Rules:** Ensure the new subnet has the same NSG rules as the old one (APIM management port 3443, etc.).

6. **Cooldown:** Wait for APIM to exit `Updating` state between operations. Do not chain multiple updates.

---

## Rollback Procedure

If the migration fails or causes issues:

**Option A: Restore configuration (if APIs/policies are broken)**

```bash
az apim restore \
  --resource-group $RG \
  --name $APIM_NAME \
  --storage-account-name $BACKUP_STORAGE_ACCOUNT \
  --storage-account-container $BACKUP_CONTAINER \
  --backup-name $BACKUP_BLOB \
  --storage-account-resource-group $RG
```

**Option B: Move back to old subnet (if network is broken)**

```bash
SUBNET_SMALL_ID=$(az network vnet subnet show \
  --resource-group $RG \
  --vnet-name $VNET_NAME \
  --name $SUBNET_SMALL \
  --query id -o tsv)

az apim update \
  --resource-group $RG \
  --name $APIM_NAME \
  --set virtualNetworkConfiguration.subnetResourceId=$SUBNET_SMALL_ID
```

> ⚠️ Rolling back to the old subnet means you will still have the original scaling limitation.

---

## Useful Diagnostic Commands

```bash
# Check APIM provisioning state
az apim show --resource-group $RG --name $APIM_NAME \
  --query "{status:provisioningState, vnetType:virtualNetworkType, subnet:virtualNetworkConfiguration.subnetResourceId}" -o json

# List all IPs used in a subnet
az network vnet subnet show --resource-group $RG --vnet-name $VNET_NAME --name $SUBNET_LARGE \
  --query "ipConfigurations[].id" -o tsv

# Check APIM platform version
az apim show --resource-group $RG --name $APIM_NAME \
  --query "platformVersion" -o tsv

# View available IPs in subnet
az network vnet subnet list-available-ips \
  --resource-group $RG \
  --vnet-name $VNET_NAME \
  --name $SUBNET_LARGE \
  -o table
```

---

> [↑ Back to README](../README.md)

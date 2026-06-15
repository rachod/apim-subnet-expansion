# Backup APIM Configuration

[← Previous: Prerequisites](01-Prerequisites.md) | [Next: Subnet Migration →](04-SubnetMigration.md)

---

> ⚠️ **CRITICAL:** Always back up APIM configuration before performing subnet migration. While the migration preserves configuration, having a backup ensures you can recover if anything goes wrong.

---

## Set Backup Variables

```bash
BACKUP_STORAGE_ACCOUNT="stapimbackup$(openssl rand -hex 4)"
BACKUP_CONTAINER="apim-backup"
BACKUP_BLOB="apim-backup-$(date +%Y%m%d-%H%M%S)"
```

---

## Create Storage Account for Backup

```bash
# Create storage account (same region as APIM)
az storage account create \
  --resource-group $RG \
  --name $BACKUP_STORAGE_ACCOUNT \
  --location $LOCATION \
  --sku Standard_LRS \
  --kind StorageV2

# Create container for backup
az storage container create \
  --name $BACKUP_CONTAINER \
  --account-name $BACKUP_STORAGE_ACCOUNT \
  --auth-mode login
```

---

## Backup APIM Using Built-in Backup API

```bash
# Enable managed identity on APIM (required for backup)
az apim update \
  --resource-group $RG \
  --name $APIM_NAME \
  --set identity.type=SystemAssigned

# Get APIM managed identity principal ID
APIM_IDENTITY=$(az apim show --resource-group $RG --name $APIM_NAME \
  --query identity.principalId -o tsv)

# Assign Storage Blob Data Contributor role to APIM identity
STORAGE_ID=$(az storage account show --resource-group $RG \
  --name $BACKUP_STORAGE_ACCOUNT --query id -o tsv)

az role assignment create \
  --assignee $APIM_IDENTITY \
  --role "Storage Blob Data Contributor" \
  --scope $STORAGE_ID

# Wait for role propagation
sleep 30

# Perform APIM backup (backs up ALL configuration)
az apim backup \
  --resource-group $RG \
  --name $APIM_NAME \
  --storage-account-name $BACKUP_STORAGE_ACCOUNT \
  --storage-account-container $BACKUP_CONTAINER \
  --backup-name $BACKUP_BLOB \
  --storage-account-resource-group $RG
```

> ⏱️ Backup takes 2-10 minutes depending on configuration size.

---

## Export Individual Configurations (Git-Friendly)

For more granular backup and version control:

```bash
# Create local backup directory
BACKUP_DIR="./apim-backup-$(date +%Y%m%d-%H%M%S)"
mkdir -p $BACKUP_DIR/{apis,products,policies,named-values,subscriptions,certificates}

# Export all APIs
echo "=== Exporting APIs ==="
az apim api list --resource-group $RG --service-name $APIM_NAME -o json \
  > $BACKUP_DIR/apis/api-list.json

# Export each API definition and operations
for API_ID in $(az apim api list --resource-group $RG --service-name $APIM_NAME \
  --query "[].name" -o tsv); do
  echo "  Exporting API: $API_ID"

  az apim api show --resource-group $RG --service-name $APIM_NAME \
    --api-id "$API_ID" -o json > "$BACKUP_DIR/apis/${API_ID}.json"

  az apim api operation list --resource-group $RG --service-name $APIM_NAME \
    --api-id "$API_ID" -o json > "$BACKUP_DIR/apis/${API_ID}-operations.json"

  az apim api policy show --resource-group $RG --service-name $APIM_NAME \
    --api-id "$API_ID" -o json > "$BACKUP_DIR/policies/${API_ID}-policy.json" 2>/dev/null || true
done

# Export products
echo "=== Exporting Products ==="
az apim product list --resource-group $RG --service-name $APIM_NAME -o json \
  > $BACKUP_DIR/products/product-list.json

# Export named values
echo "=== Exporting Named Values ==="
az apim nv list --resource-group $RG --service-name $APIM_NAME -o json \
  > $BACKUP_DIR/named-values/named-values.json

# Export subscriptions
echo "=== Exporting Subscriptions ==="
az apim subscription list --resource-group $RG --service-name $APIM_NAME -o json \
  > $BACKUP_DIR/subscriptions/subscriptions.json

# Export global policy
echo "=== Exporting Global Policy ==="
az apim policy show --resource-group $RG --service-name $APIM_NAME \
  -o json > $BACKUP_DIR/policies/global-policy.json 2>/dev/null || true

# Export hostname configurations (custom domains & certificates)
echo "=== Exporting Hostname Configurations ==="
az apim show --resource-group $RG --name $APIM_NAME \
  --query "hostnameConfigurations" -o json > $BACKUP_DIR/certificates/hostname-configs.json
```

---

## Verify Backup

```bash
# Verify blob backup exists in storage
az storage blob list \
  --container-name $BACKUP_CONTAINER \
  --account-name $BACKUP_STORAGE_ACCOUNT \
  --auth-mode login \
  --query "[].{name:name, size:properties.contentLength, lastModified:properties.lastModified}" \
  -o table

# Verify local backup files
echo "=== Local Backup Contents ==="
ls -la $BACKUP_DIR/apis/
ls -la $BACKUP_DIR/policies/
```

---

## Restore (If Needed After Failed Migration)

```bash
# Restore APIM from backup
az apim restore \
  --resource-group $RG \
  --name $APIM_NAME \
  --storage-account-name $BACKUP_STORAGE_ACCOUNT \
  --storage-account-container $BACKUP_CONTAINER \
  --backup-name $BACKUP_BLOB \
  --storage-account-resource-group $RG
```

> ⚠️ **Restore notes:**
> - Restore **overwrites ALL** current configuration
> - Restore does **NOT** change the VNet/subnet configuration
> - If APIM is in a broken state after migration, restore only recovers API configs, not network settings
> - For network rollback, use: `az apim update --set virtualNetworkConfiguration.subnetResourceId=<OLD_SUBNET_ID>`

---

## What is NOT Included in Backup

> ⚠️ The following items are **not captured** by `az apim backup` and must be reconfigured manually if lost:

| Item | Notes |
|------|-------|
| **Usage/analytics data** | Use [REST API](https://learn.microsoft.com/en-us/rest/api/apimanagement/) to export analytics separately |
| **Custom domain TLS/SSL certificates** | Must re-upload certificates after restore |
| **Custom CA certificates** | Intermediate or root certificates uploaded by the customer |
| **Virtual network integration settings** | VNet/subnet configuration, private IPs |
| **Managed identity configuration** | System-assigned and user-assigned identities |
| **Azure Monitor diagnostic settings** | Log/metric export configurations |
| **Protocols and cipher settings** | Custom TLS protocol and cipher configurations |
| **Developer portal content** | Published portal customizations |
| **Credential manager (authorizations)** | OAuth connections and authorization configurations |

**Additional constraints:**
- Backup expires after **30 days** — restore will fail after expiration
- The **pricing tier must match** between source and target during restore
- Avoid management changes while backup is in progress — changes may be excluded

> 📖 Reference: [What is not backed up — Microsoft Docs](https://learn.microsoft.com/en-us/azure/api-management/api-management-howto-disaster-recovery-backup-restore?tabs=cli#what-is-not-backed-up)

---

> [Next: Subnet Migration →](04-SubnetMigration.md)

# Prerequisites

[↑ Back to README](../README.md) | [Next: Backup Configuration →](03-BackupConfiguration.md)

---

## Requirements

| Requirement | Details |
|-------------|---------|
| Azure Subscription | With permissions to create APIM, VNet, NSG, Storage |
| Azure CLI | Version 2.50+ (`az --version`) |
| APIM SKU | Premium (required for VNet integration) |
| VNet Mode | Internal |
| Region | Southeast Asia (or your target region) |

---

## Set Variables

```bash
# Configuration - update these for your environment
RG="rg-apim-subnet-poc"
LOCATION="southeastasia"
VNET_NAME="vnet-apim-poc"
VNET_PREFIX="10.0.0.0/16"
SUBNET_SMALL="snet-apim-small"
SUBNET_SMALL_PREFIX="10.0.1.0/28"       # Only 11 usable IPs - TOO SMALL
SUBNET_LARGE="snet-apim-large"
SUBNET_LARGE_PREFIX="10.0.2.0/25"       # 123 usable IPs - sufficient for scaling
APIM_NAME="apim-subnet-poc"
PUBLISHER_EMAIL="admin@contoso.com"
PUBLISHER_NAME="Contoso"
NSG_NAME="nsg-apim"
```

---

## Create Resource Group

```bash
az group create \
  --name $RG \
  --location $LOCATION
```

---

## Create NSG with Required APIM Rules

APIM in Internal VNet mode requires specific NSG rules:

```bash
# Create NSG
az network nsg create \
  --resource-group $RG \
  --name $NSG_NAME \
  --location $LOCATION
```

**Inbound Rules:**

```bash
# Allow APIM management endpoint (required)
az network nsg rule create \
  --resource-group $RG \
  --nsg-name $NSG_NAME \
  --name Allow-APIM-Management \
  --priority 100 \
  --direction Inbound \
  --access Allow \
  --protocol Tcp \
  --source-address-prefixes ApiManagement \
  --source-port-ranges '*' \
  --destination-address-prefixes VirtualNetwork \
  --destination-port-ranges 3443

# Allow Azure Load Balancer
az network nsg rule create \
  --resource-group $RG \
  --nsg-name $NSG_NAME \
  --name Allow-AzureLoadBalancer \
  --priority 110 \
  --direction Inbound \
  --access Allow \
  --protocol '*' \
  --source-address-prefixes AzureLoadBalancer \
  --source-port-ranges '*' \
  --destination-address-prefixes VirtualNetwork \
  --destination-port-ranges '*'

# Allow HTTPS from VNet (for internal API calls)
az network nsg rule create \
  --resource-group $RG \
  --nsg-name $NSG_NAME \
  --name Allow-HTTPS-Inbound \
  --priority 120 \
  --direction Inbound \
  --access Allow \
  --protocol Tcp \
  --source-address-prefixes VirtualNetwork \
  --source-port-ranges '*' \
  --destination-address-prefixes VirtualNetwork \
  --destination-port-ranges 443
```

**Outbound Rules:**

```bash
# Allow Azure Storage
az network nsg rule create \
  --resource-group $RG \
  --nsg-name $NSG_NAME \
  --name Allow-Storage-Outbound \
  --priority 100 \
  --direction Outbound \
  --access Allow \
  --protocol Tcp \
  --source-address-prefixes VirtualNetwork \
  --source-port-ranges '*' \
  --destination-address-prefixes Storage \
  --destination-port-ranges '443'

# Allow Azure SQL
az network nsg rule create \
  --resource-group $RG \
  --nsg-name $NSG_NAME \
  --name Allow-SQL-Outbound \
  --priority 110 \
  --direction Outbound \
  --access Allow \
  --protocol Tcp \
  --source-address-prefixes VirtualNetwork \
  --source-port-ranges '*' \
  --destination-address-prefixes Sql \
  --destination-port-ranges 1433

# Allow Azure Key Vault
az network nsg rule create \
  --resource-group $RG \
  --nsg-name $NSG_NAME \
  --name Allow-KeyVault-Outbound \
  --priority 120 \
  --direction Outbound \
  --access Allow \
  --protocol Tcp \
  --source-address-prefixes VirtualNetwork \
  --source-port-ranges '*' \
  --destination-address-prefixes AzureKeyVault \
  --destination-port-ranges 443

# Allow Azure Monitor (metrics, diagnostics, logging)
az network nsg rule create \
  --resource-group $RG \
  --nsg-name $NSG_NAME \
  --name Allow-AzureMonitor-Outbound \
  --priority 130 \
  --direction Outbound \
  --access Allow \
  --protocol Tcp \
  --source-address-prefixes VirtualNetwork \
  --source-port-ranges '*' \
  --destination-address-prefixes AzureMonitor \
  --destination-port-ranges '1886 443'

# Allow Azure Active Directory
az network nsg rule create \
  --resource-group $RG \
  --nsg-name $NSG_NAME \
  --name Allow-AzureAD-Outbound \
  --priority 140 \
  --direction Outbound \
  --access Allow \
  --protocol Tcp \
  --source-address-prefixes VirtualNetwork \
  --source-port-ranges '*' \
  --destination-address-prefixes AzureActiveDirectory \
  --destination-port-ranges 443

# Allow Azure Event Hub (for logging)
az network nsg rule create \
  --resource-group $RG \
  --nsg-name $NSG_NAME \
  --name Allow-EventHub-Outbound \
  --priority 150 \
  --direction Outbound \
  --access Allow \
  --protocol Tcp \
  --source-address-prefixes VirtualNetwork \
  --source-port-ranges '*' \
  --destination-address-prefixes EventHub \
  --destination-port-ranges '5671 5672 443'

# Allow SMTP Relay (for email notifications, optional)
az network nsg rule create \
  --resource-group $RG \
  --nsg-name $NSG_NAME \
  --name Allow-SMTP-Outbound \
  --priority 160 \
  --direction Outbound \
  --access Allow \
  --protocol Tcp \
  --source-address-prefixes VirtualNetwork \
  --source-port-ranges '*' \
  --destination-address-prefixes Internet \
  --destination-port-ranges '25 587 25028'
```

> ⚠️ **IMPORTANT: If your NSG has a "Deny All Internet Outbound" rule**, all Allow rules above **must have a lower priority number** (higher priority) than the Deny rule. For example, if your Deny rule is at priority 999, ensure all APIM Allow rules are at priorities 100-998.

**Example — Deny All Internet Outbound rule (commonly applied by security teams):**

```bash
# This is the rule that blocks APIM if Allow rules are not added above it:
# Rule: 999  Deny_All_Internet_Outbound  Outbound  Deny  *  Internet

# Verify your Deny rule priority:
az network nsg rule show \
  --resource-group $RG \
  --nsg-name $NSG_NAME \
  --name Deny_All_Internet_Outbound \
  --query "{name:name, priority:priority, access:access, direction:direction}" \
  -o table

# All Allow rules must have priority < 999 (lower number = evaluated first)
# Our rules use priorities 100-160, so they are processed BEFORE the deny rule.
```

**Full list of required APIM outbound dependencies:**

| Priority | Rule Name | Destination Service Tag | Ports | Required |
|----------|-----------|------------------------|-------|----------|
| 100 | Allow-Storage-Outbound | `Storage` | 443 | ✅ Yes |
| 110 | Allow-SQL-Outbound | `Sql` | 1433 | ✅ Yes |
| 120 | Allow-KeyVault-Outbound | `AzureKeyVault` | 443 | ✅ Yes |
| 130 | Allow-AzureMonitor-Outbound | `AzureMonitor` | 1886, 443 | ✅ Yes |
| 140 | Allow-AzureAD-Outbound | `AzureActiveDirectory` | 443 | ✅ Yes |
| 150 | Allow-EventHub-Outbound | `EventHub` | 5671, 5672, 443 | ⚠️ If logging enabled |
| 160 | Allow-SMTP-Outbound | `Internet` | 25, 587, 25028 | ⚠️ If email enabled |
| 999 | Deny_All_Internet_Outbound | `Internet` | * | 🔒 Security policy |

---

## Create VNet with Small Subnet

```bash
# Create VNet
az network vnet create \
  --resource-group $RG \
  --name $VNET_NAME \
  --address-prefixes $VNET_PREFIX \
  --location $LOCATION

# Create the SMALL subnet (/28 - insufficient for scaling)
az network vnet subnet create \
  --resource-group $RG \
  --vnet-name $VNET_NAME \
  --name $SUBNET_SMALL \
  --address-prefixes $SUBNET_SMALL_PREFIX \
  --network-security-group $NSG_NAME
```

---

> [Next: Backup Configuration →](03-BackupConfiguration.md)

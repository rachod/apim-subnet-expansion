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
```

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

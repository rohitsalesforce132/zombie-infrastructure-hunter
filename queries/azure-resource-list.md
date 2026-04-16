# Azure Resource List Queries

> Extract all resources from Azure subscriptions for zombie analysis.

## List Everything

```bash
# All resources in current subscription
az resource list --output json > all-resources.json

# Human-readable table
az resource list --query '[].{Name:name, Type:type, RG:resourceGroup, Location:location, Created:createdTime}' -o table

# Count by type
az resource list --query '[].type' -o tsv | sort | uniq -c | sort -rn
```

## By Resource Group

```bash
# List all resource groups
az group list --query '[].{Name:name, Location:location, Tags:tags}' -o table

# Resources per resource group
az group list --query '[].name' -o tsv | while read rg; do
    count=$(az resource list --resource-group "$rg" --query 'length([])' -o tsv)
    echo "$rg: $count resources"
done
```

## By Resource Type

```bash
# AKS clusters
az aks list --query '[].{Name:name, RG:resourceGroup, Location:location, Tags:tags, Provisioning:provisioningState}' -o table

# Storage accounts
az storage account list --query '[].{Name:name, RG:resourceGroup, Location:location, Kind:kind, Tags:tags}' -o table

# Virtual machines
az vm list --query '[].{Name:name, RG:resourceGroup, Location:location, Tags:tags, Power:powerState}' -o table

# SQL databases
az sql db list --query '[].{Name:name, Server:serverName, Status:status, Tags:tags}' -o table

# VNets
az network vnet list --query '[].{Name:name, RG:resourceGroup, Location:location, Tags:tags}' -o table

# Public IPs
az network public-ip list --query '[].{Name:name, RG:resourceGroup, IP:ipAddress, Tags:tags}' -o table

# Load balancers
az network lb list --query '[].{Name:name, RG:resourceGroup, Location:location, Tags:tags}' -o table

# Key vaults
az keyvault list --query '[].{Name:name, RG:resourceGroup, Location:location, Tags:tags}' -o table

# Container registries
az acr list --query '[].{Name:name, RG:resourceGroup, SKU:sku.name, Tags:tags}' -o table

# App Services
az webapp list --query '[].{Name:name, RG:resourceGroup, State:state, Tags:tags}' -o table
```

## With Ownership Tags

```bash
# Resources with owner tag
az resource list --query "[?tags.owner != null].{Name:name, Owner:tags.owner, Type:type}" -o table

# Resources WITHOUT owner tag (zombie signal)
az resource list --query "[?tags.owner == null].{Name:name, Type:type, RG:resourceGroup}" -o table

# Resources with cost-center tag
az resource list --query "[?tags.'cost-center' != null].{Name:name, CostCenter:tags.'cost-center'}" -o table

# Resources WITHOUT cost-center tag
az resource list --query "[?tags.'cost-center' == null].{Name:name, Type:type}" -o table
```

## Filter by Age

```bash
# Resources older than 90 days (potential zombies)
CUTOFF=$(date -d '90 days ago' +%Y-%m-%d)
az resource list --query "[?createdTime < '$CUTOFF'].{Name:name, Created:createdTime, Type:type}" -o table

# Resources older than 180 days
CUTOFF=$(date -d '180 days ago' +%Y-%m-%d)
az resource list --query "[?createdTime < '$CUTOFF'].{Name:name, Created:createdTime, Type:type}" -o table

# Resources created in last 30 days (definitely NOT zombies)
az resource list --query "[?createdTime > '$(date -d '30 days ago' +%Y-%m-%d)'].{Name:name, Created:createdTime}" -o table
```

## Export for Analysis

```bash
# Full JSON export for script processing
az resource list --output json > azure-resources-full.json

# CSV export for spreadsheet
az resource list --query '[].{Name:name, Type:type, RG:resourceGroup, Location:location, Owner:tags.owner, Created:createdTime}' -o csv > azure-resources.csv

# Just names for diffing
az resource list --query '[].name' -o tsv > azure-resource-names.txt
```

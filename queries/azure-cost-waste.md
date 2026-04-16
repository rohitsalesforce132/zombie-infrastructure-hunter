# Azure Cost Waste Analysis

> Identify how much zombie resources are costing.

## Current Monthly Spend

```bash
# Total monthly cost
az costmanagement query --type ActualCost --timeframe MonthToDate \
    --scope "/subscriptions/$(az account show --query id -o tsv)" \
    --output json

# Cost by resource group
az costmanagement query --type ActualCost --timeframe MonthToDate \
    --scope "/subscriptions/$(az account show --query id -o tsv)" \
    --query 'rows[].[0,1,2]' -o table

# Cost by service type
az costmanagement query --type ActualCost --timeframe MonthToDate \
    --scope "/subscriptions/$(az account show --query id -o tsv)" \
    --dimension ServiceName
```

## Cost by Resource Type

```bash
# AKS clusters cost
az costmanagement query --type ActualCost --timeframe MonthToDate \
    --scope "/subscriptions/$(az account show --query id -o tsv)" \
    --filter '{"dimensions":{"name":"ServiceName","values":["Kubernetes Service"]}}'

# Storage cost
az costmanagement query --type ActualCost --timeframe MonthToDate \
    --scope "/subscriptions/$(az account show --query id -o tsv)" \
    --filter '{"dimensions":{"name":"ServiceName","values":["Storage"]}}'

# Virtual Machines cost
az costmanagement query --type ActualCost --timeframe MonthToDate \
    --scope "/subscriptions/$(az account show --query id -o tsv)" \
    --filter '{"dimensions":{"name":"ServiceName","values":["Virtual Machines"]}}'

# Networking cost
az costmanagement query --type ActualCost --timeframe MonthToDate \
    --scope "/subscriptions/$(az account show --query id -o tsv)" \
    --filter '{"dimensions":{"name":"ServiceName","values":["Virtual Network","Public IP","Load Balancer"]}}'
```

## Estimated Zombie Cost

```bash
# For each zombie candidate, estimate monthly cost:

# Stopped VMs (disk cost only)
az vm list --show-details --query "[?powerState == 'Stopped']" -o json | \
  jq -r '.[] | .name + ": ~$" + ((.storageProfile.osDisk.diskSizeGb * 0.05) | tostring) + "/month (disk only)"'

# Unattached disks
az disk list --query "[?diskState == 'Unattached']" -o json | \
  jq -r '.[] | .name + ": ~$" + ((.diskSizeGb * 0.05) | tostring) + "/month"'

# Unused public IPs (~$3/month each)
az network public-ip list --query "[?ipConfiguration == null]" -o json | \
  jq -r '.[] | .name + ": ~$3/month"'

# Idle AKS clusters (worker nodes)
az aks list --query '[].{Name:name, NodeCount:agentPoolProfiles[0].count, SKU:agentPoolProfiles[0].vmSize}' -o json | \
  jq -r '.[] | .Name + ": " + (.NodeCount|tostring) + " x " + .SKU + " (~$" + ((.NodeCount * 160) | tostring) + "/month)"'
```

## Cost Trend (Last 3 Months)

```bash
# Monthly cost trend
for month in $(seq 2 -1 0); do
    start=$(date -d "$month months ago" +%Y-%m-01)
    end=$(date -d "$((month-1)) months ago" +%Y-%m-01)
    cost=$(az costmanagement query --type ActualCost \
        --scope "/subscriptions/$(az account show --query id -o tsv)" \
        --timeframe Custom --from "$start" --to "$end" \
        --query 'properties.rows[0][0]' -o tsv 2>/dev/null || echo "N/A")
    echo "$start: \$$cost"
done
```

## Savings Estimation Table

| Zombie Type | Typical Cost | Detection Method |
|------------|-------------|-----------------|
| Idle AKS cluster | $1,600/node/month | No deployments, no pods |
| Stopped VM (not deallocated) | $50-500/month | powerState = Stopped |
| Deallocated VM (disk only) | $5-50/month | Disk still exists |
| Unattached disk | $5-50/month | diskState = Unattached |
| Unused public IP | $3/month | No ipConfiguration |
| Empty resource group | $0 (but messy) | 0 resources |
| Unused storage account | $1-100/month | No containers or no access |
| Unused NIC | $0 (but messy) | No VM attachment |
| Unused NSG | $0 (but messy) | No subnet/NIC attachment |

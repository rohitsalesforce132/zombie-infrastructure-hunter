# Azure Orphan Detection Queries

> Find resources with no dependencies, no connections, no reason to exist.

## Unattached Disks

```bash
# Managed disks not attached to any VM
az disk list --query "[?diskState == 'Unattached'].{Name:name, RG:resourceGroup, Size:diskSizeGb, SKU:sku.name}" -o table

# Cost of unattached disks
az disk list --query "[?diskState == 'Unattached'].{Name:name, SizeGb:diskSizeGb}" -o json | \
  jq -r '.[] | "\(.name): \(.SizeGb) GB (~$\( (.SizeGb * 0.05) | floor )/month"'
```

## Unassigned Public IPs

```bash
# Public IPs not associated with any resource
az network public-ip list --query "[?ipConfiguration == null].{Name:name, RG:resourceGroup, IP:ipAddress, SKU:sku.name}" -o table

# Cost: ~$3/month per unused static IP
```

## Empty Resource Groups

```bash
# Resource groups with 0 resources
az group list --query '[].name' -o tsv | while read rg; do
    count=$(az resource list --resource-group "$rg" --query 'length([])' -o tsv)
    if [ "$count" = "0" ]; then
        echo "EMPTY: $rg"
    fi
done
```

## Idle AKS Clusters

```bash
# AKS clusters with no running pods or deployments
az aks list --query '[].{Name:name, RG:resourceGroup}' -o tsv | while IFS=$'\t' read -r name rg; do
    # Check if any user pods running
    pods=$(az aks command invoke --resource-group "$rg" --name "$name" \
        --command "kubectl get pods --all-namespaces --no-headers 2>/dev/null | grep -v kube-system | wc -l" \
        --query 'logs' -o tsv 2>/dev/null || echo "unknown")
    echo "$name ($rg): $pods user pods"
done
```

## Stopped VMs

```bash
# VMs in deallocated/stopped state (still incurring disk costs)
az vm list --show-details --query "[?powerState != 'Running'].{Name:name, RG:resourceGroup, State:powerState}" -o table

# Stopped but NOT deallocated (still paying for compute!)
az vm list --show-details --query "[?powerState == 'Stopped'].{Name:name, RG:resourceGroup}" -o table
```

## Unused Network Interfaces

```bash
# NICs not attached to any VM
az network nic list --query "[?virtualMachine == null].{Name:name, RG:resourceGroup}" -o table
```

## Unused Network Security Groups

```bash
# NSGs not attached to any subnet or NIC
az network nsg list --query '[].{Name:name, RG:resourceGroup}' -o tsv | while IFS=$'\t' read -r name rg; do
    subnets=$(az network vnet subnet list --resource-group "$rg" --vnet-name $(az network vnet list --resource-group "$rg" --query "[?contains(networkSecurityGroup.id, '$name')].name" -o tsv 2>/dev/null) --query 'length([])' -o tsv 2>/dev/null || echo "0")
    nics=$(az network nic list --resource-group "$rg" --query "[?contains(networkSecurityGroup.id, '$name')].name" -o tsv 2>/dev/null || echo "")
    if [ -z "$nics" ] && [ "$subnets" = "0" ]; then
        echo "ORPHAN NSG: $name ($rg)"
    fi
done
```

## Storage Accounts with No Activity

```bash
# Check last blob modification time
az storage account list --query '[].{Name:name, RG:resourceGroup}' -o tsv | while IFS=$'\t' read -r name rg; do
    key=$(az storage account keys list --account-name "$name" --query '[0].value' -o tsv 2>/dev/null)
    if [ -n "$key" ]; then
        containers=$(az storage container list --account-name "$name" --account-key "$key" --query 'length([])' -o tsv 2>/dev/null || echo "0")
        if [ "$containers" = "0" ]; then
            echo "EMPTY STORAGE: $name ($rg)"
        fi
    fi
done
```

## Cross-Reference: Not in Terraform

```bash
# Get TF-managed resource names
terraform state list | sed 's/\[.*//; s/\.this//; s/data\.//' | awk -F'.' '{print $NF}' | sort > tf-resources.txt

# Get Azure resource names
az resource list --query '[].name' -o tsv | sort > azure-resources.txt

# Find resources in Azure but NOT in TF
comm -23 azure-resources.txt tf-resources.txt > not-in-tf.txt
cat not-in-tf.txt
```

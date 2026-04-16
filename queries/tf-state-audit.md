# Terraform State Audit

> Compare what Terraform manages vs what actually exists in Azure.

## List TF-Managed Resources

```bash
# All resources in TF state
terraform state list

# Count by type
terraform state list | awk -F'[' '{print $1}' | sed 's/^data\.//' | sort | uniq -c | sort -rn

# Full resource details
terraform state list | while read res; do
    echo "=== $res ==="
    terraform state show "$res" 2>/dev/null | head -20
done
```

## Compare TF State vs Azure Reality

```bash
# Export TF resource names
terraform state list | sed 's/\[.*//' | awk -F'.' '{print $NF}' | sort > tf-managed.txt

# Export Azure resource names
az resource list --query '[].name' -o tsv | sort > azure-actual.txt

# Resources in Azure but NOT in TF (potential zombies)
echo "=== IN AZURE, NOT IN TF ==="
comm -23 azure-actual.txt tf-managed.txt

# Resources in TF but NOT in Azure (drift/orphans)
echo "=== IN TF, NOT IN AZURE ==="
comm -13 azure-actual.txt tf-managed.txt
```

## TF State Freshness

```bash
# When was state last updated?
terraform state pull | jq '{serial: .serial, lineage: .lineage, version: .version}'

# Check for resources that have drifted
terraform plan -detailed-exitcode 2>/dev/null
# Exit code 2 = drift detected
# Exit code 0 = no changes
# Exit code 1 = error
```

## Resources Missing Tags

```bash
# TF resources without required tags
terraform state list | while read res; do
    tags=$(terraform state show "$res" 2>/dev/null | grep -A5 "tags = " | grep "owner")
    if [ -z "$tags" ]; then
        echo "NO OWNER TAG: $res"
    fi
done
```

## State File Analysis

```bash
# Total resources managed
echo "Total TF resources: $(terraform state list | wc -l)"

# Resources by type
echo "By type:"
terraform state list | awk -F'_' '{print $1}' | sed 's/\[.*//' | sort | uniq -c | sort -rn

# Check for duplicate resources
terraform state list | awk -F'[' '{print $1}' | sort | uniq -d
```

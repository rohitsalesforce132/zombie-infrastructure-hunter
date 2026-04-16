# Decommission Plan Template

> Checklist for safely decommissioning a zombie resource.

## Pre-Deletion Checklist

### 1. Verify No Dependencies
```bash
# Check TF references
terraform graph | grep "[RESOURCE_NAME]"

# Check k8s references
kubectl get all -A -o yaml | grep "[RESOURCE_NAME]"

# Check application configs
grep -r "[RESOURCE_NAME]" /path/to/configs/

# Check context graph
grep -r "[RESOURCE_NAME]" nodes/ decisions/ incidents/
```

- [ ] No Terraform references found
- [ ] No Kubernetes references found
- [ ] No application config references found
- [ ] No context graph references found

### 2. Confirm with Stakeholders
- [ ] Owner/team contacted (if owner exists)
- [ ] Platform team confirmed (for infra resources)
- [ ] Security team confirmed (for compliance resources)
- [ ] No pending projects need this resource

### 3. Backup (If Needed)
```bash
# For storage accounts: export data
az storage blob download-batch --destination ./backup/ --source [container] --account-name [name]

# For databases: export
az sql db export --server [server] --name [db] --admin-user [user] --admin-password [pass] --storage-key-type StorageAccessKey --storage-key [key] --storage-uri https://[account].blob.core.windows.net/backup/[db].bacpac

# For VMs: capture image (if needed)
az vm deallocate --resource-group [rg] --name [vm]
az vm generalize --resource-group [rg] --name [vm]
az image create --resource-group [rg] --name [vm]-backup --source [vm]
```

- [ ] Data backed up (if applicable)
- [ ] Backup verified

### 4. Schedule Deletion
- **Deletion window:** [date/time]
- **Notify:** [who needs to know]
- **Rollback plan:** [how to undo if something breaks]

### 5. Execute Deletion
```bash
# Resource group (deletes everything inside)
az group delete --name [rg-name] --yes --no-wait

# Individual resource
az [resource-type] delete --name [name] --resource-group [rg] --yes

# Verify deletion
az resource list --resource-group [rg] --query 'length([])'
```

- [ ] Resource deleted
- [ ] Deletion verified in Azure portal

### 6. Post-Deletion
- [ ] Remove TF state reference (if managed): `terraform state rm [resource]`
- [ ] Remove context graph node (if exists)
- [ ] Update zombie report with savings
- [ ] Record in memory log: "Deleted [resource], saving $[amount]/month"

## Monthly Savings Tracking

| Resource | Deleted Date | Monthly Savings | Annual Savings |
|----------|-------------|----------------|---------------|
| [name] | [date] | $[amount] | $[amount] |
| **Total** | | **$[total]** | **$[total]** |

# Gap Detection for Infrastructure

> Using Knowledge Weaving gap detection patterns to find undocumented resources.

From [[Agentic Knowledge Mesh]] → Knowledge Weaving Patterns → Gap Detection.

## Gap Types Applied to Infrastructure

### Gap 1: Orphan Entity — Resource exists but no context node

**Detection:**
```bash
# Get all Azure resources
AZURE_RESOURCES=$(az resource list --query '[].name' -o tsv)

# Get all context graph nodes
CG_NODES=$(ls nodes/*.md 2>/dev/null | sed 's/nodes\///; s/\.md//')

# Find orphans (in Azure but not in Context Graph)
for resource in $AZURE_RESOURCES; do
    if ! echo "$CG_NODES" | grep -qi "$resource"; then
        echo "ORPHAN: $resource (no context graph node)"
    fi
done
```

**Zombie signal:** Strong. If nobody documented it, nobody cares about it.

### Gap 2: Stale Decision — Context node exists but never updated

**Detection:**
```bash
# Find context nodes older than 6 months
find nodes/ -name "*.md" -mtime +180 -exec echo "STALE: {}" \;

# Check if node has change history
for node in nodes/*.md; do
    if ! grep -q "Change History" "$node"; then
        echo "NO HISTORY: $node"
    fi
done
```

**Zombie signal:** Medium. May be intentionally stable, or may be forgotten.

### Gap 3: Missing Dependency — Resource has no inbound connections

**Detection:**
```bash
# Check TF graph for references
terraform graph | grep -c "RESOURCE_NAME"

# Check k8s manifests
grep -r "RESOURCE_NAME" k8s-manifests/ 2>/dev/null

# Check context graph dependencies
grep -A10 "depended on by" nodes/RESOURCE_NAME.md 2>/dev/null
```

**Zombie signal:** Strong if no dependencies found AND no activity.

### Gap 4: Missing Owner — No owner tag or owner inactive

**Detection:**
```bash
# Find resources with no owner tag
az resource list --query "[?tags.owner == null].{name:name, type:type}" -o table

# Find resources where owner has no recent activity
az resource list --query '[].{name:name, owner:tags.owner}' -o json | \
  jq -r '.[] | select(.owner != null) | .owner' | sort -u | \
  while read owner; do
    activity=$(az monitor activity-log list --caller "$owner" --max-events 1 2>/dev/null | jq length)
    if [ "$activity" = "0" ]; then
        echo "INACTIVE OWNER: $owner"
    fi
  done
```

**Zombie signal:** Strong. No owner = nobody to ask before deleting.

### Gap 5: Outside IaC — Resource not in Terraform state

**Detection:**
```bash
# List TF-managed resources
TF_RESOURCES=$(terraform state list | awk -F'[' '{print $1}')

# List Azure resources
AZURE_RESOURCES=$(az resource list --query '[].name' -o tsv)

# Find Azure resources not in TF state
for resource in $AZURE_RESOURCES; do
    if ! echo "$TF_RESOURCES" | grep -q "$resource"; then
        echo "NOT IN TF: $resource"
    fi
done
```

**Zombie signal:** Medium. Could be intentionally managed outside TF, but often indicates ad-hoc creation.

## Combined Gap Score

Each gap detected adds to the zombie likelihood:

| Gaps Detected | Zombie Likelihood | Tier |
|--------------|------------------|------|
| 4-5 gaps | Very High | 🧟 Tier 1 |
| 3 gaps | High | 🧟‍♂️ Tier 2 |
| 2 gaps | Medium | ⚠️ Tier 3 |
| 1 gap | Low | 💤 Tier 4 |
| 0 gaps | None | ✅ Active |

## Anti-Patterns (False Positives)

Not every undocumented resource is a zombie. Watch for:

| Anti-Pattern | Why It's Not Zombie | Example |
|-------------|-------------------|---------|
| DR/backup resources | Intentionally dormant | DR cluster in Central US |
| Compliance resources | Required but never accessed | Audit log storage |
| New resources | Just created, not documented yet | Resource < 30 days old |
| Shared infrastructure | Used by many, no single owner | Core VNet, DNS zones |
| Scheduled resources | Intentionally off (cost savings) | Dev cluster powered down nights |

These should be classified as **Dormant (Tier 4)**, not Zombie, and documented.

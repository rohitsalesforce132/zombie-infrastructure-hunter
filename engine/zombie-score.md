# Zombie Score Algorithm

> 5 dimensions, 0-20 points each, total 0-100.

## Dimension 1: Documentation (0-20)

| Signal | Score |
|--------|-------|
| Context graph node exists AND fresh (< 3 months) | 20 |
| Context graph node exists but stale (3-6 months) | 15 |
| Context graph node exists but very stale (6+ months) | 10 |
| TF code exists with comments | 8 |
| TF code exists but no comments | 5 |
| Azure tags with description | 3 |
| No documentation at all | 0 |

### How to Check

```bash
# Check if context graph node exists
ls nodes/${RESOURCE_NAME}.md 2>/dev/null && echo "EXISTS" || echo "MISSING"

# Check node freshness
stat -c %Y nodes/${RESOURCE_NAME}.md  # Last modified timestamp

# Check TF code for comments
grep -c "^#" terraform/${RESOURCE}.tf

# Check Azure tags
az resource show --ids $RESOURCE_ID --query 'tags'
```

## Dimension 2: Ownership (0-20)

| Signal | Score |
|--------|-------|
| Owner tag exists, owner active in org, recent activity | 20 |
| Owner tag exists, owner active, no recent activity | 15 |
| Owner tag exists, owner status unknown | 10 |
| Owner tag exists, owner LEFT org | 3 |
| No owner tag but team tag exists | 8 |
| No owner, no team | 0 |

### How to Check

```bash
# Get owner tag
az resource show --ids $RESOURCE_ID --query 'tags.owner' -o tsv

# Get team tag
az resource show --ids $RESOURCE_ID --query 'tags.team' -o tsv

# Check if owner has recent Azure activity
az monitor activity-log list --caller "owner@domain.com" --max-events 1
```

## Dimension 3: Dependencies (0-20)

| Signal | Score |
|--------|-------|
| Other resources/services actively depend on it | 20 |
| Other resources depend on it but low traffic | 15 |
| Dependencies exist but all dormant | 10 |
| Dependencies exist in TF but not actually used | 5 |
| No dependencies found | 0 |

### How to Check

```bash
# Check TF dependencies
terraform graph | grep $RESOURCE_NAME

# Check k8s references (for AKS resources)
kubectl get deployments -o json | jq '.[].spec.template.spec.containers[].env[] | select(.value | contains("'$RESOURCE_NAME'"))'

# Check context graph dependency section
grep -A20 "depended on by" nodes/${RESOURCE_NAME}.md

# Check networking references
az network list-outbound-dependencies --resource-group $RG --name $RESOURCE
```

## Dimension 4: Activity (0-20)

| Signal | Score |
|--------|-------|
| Active in last 7 days | 20 |
| Active in last 30 days | 15 |
| Active in last 90 days | 10 |
| Active in last 180 days | 5 |
| No activity in 180+ days | 0 |

### How to Check

```bash
# Check Azure activity log for this resource
az monitor activity-log list --resource $RESOURCE_NAME --max-events 10

# Check storage access (for storage accounts)
az storage blob list --account-name $ACCOUNT --container-name logs --num-results 1

# Check SQL/database connections (for databases)
az sql db show-usage --resource-group $RG --server $SERVER --name $DB

# Check VM usage (for VMs)
az vm get-instance-view --resource-group $RG --name $VM --query 'instanceView.statuses'
```

## Dimension 5: Incident Relevance (0-20)

| Signal | Score |
|--------|-------|
| Involved in incident in last 30 days | 20 |
| Involved in incident in last 90 days | 15 |
| Involved in incident in last 180 days | 10 |
| Referenced in context graph incidents section | 8 |
| No incident history | 0 |

### How to Check

```bash
# Search context graph incident nodes
grep -r "$RESOURCE_NAME" context-graph/incidents/

# Search incident replay engine
grep -r "$RESOURCE_NAME" incident-replay-engine/replays/

# Search decision traces
grep -r "$RESOURCE_NAME" context-graph/decisions/

# Check Azure Monitor alerts for this resource
az monitor activity-log list --resource $RESOURCE_NAME --filter "eventStatus eq 'Failed'"
```

## Total Score Classification

```bash
# Calculate zombie score
DOC_SCORE=0    # Documentation (0-20)
OWNER_SCORE=0  # Ownership (0-20)
DEP_SCORE=0    # Dependencies (0-20)
ACT_SCORE=0    # Activity (0-20)
INC_SCORE=0    # Incident Relevance (0-20)

TOTAL=$((DOC_SCORE + OWNER_SCORE + DEP_SCORE + ACT_SCORE + INC_SCORE))

# Classify
if [ $TOTAL -le 20 ]; then
    echo "🧟 TIER 1 — Definite Zombie (score: $TOTAL/100)"
elif [ $TOTAL -le 40 ]; then
    echo "🧟‍♂️ TIER 2 — Suspected Zombie (score: $TOTAL/100)"
elif [ $TOTAL -le 60 ]; then
    echo "⚠️ TIER 3 — Unknown (score: $TOTAL/100)"
elif [ $TOTAL -le 80 ]; then
    echo "💤 TIER 4 — Dormant (score: $TOTAL/100)"
else
    echo "✅ ACTIVE — Healthy resource (score: $TOTAL/100)"
fi
```

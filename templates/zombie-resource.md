# Zombie Finding Template

```markdown
# Zombie Finding: [RESOURCE_NAME]

## Basic Info
- **Type:** [resource-type]
- **Resource Group:** [rg-name]
- **Location:** [location]
- **Created:** [date]
- **Monthly Cost:** [$amount]

## Zombie Score: [SCORE]/100

| Dimension | Score | Evidence |
|-----------|-------|---------|
| Documentation | [0-20] | [context node exists? last updated?] |
| Ownership | [0-20] | [owner tag? owner active?] |
| Dependencies | [0-20] | [what depends on this?] |
| Activity | [0-20] | [last activity date] |
| Incident Relevance | [0-20] | [ever in an incident?] |

## Classification: [TIER]

- 🧟 Tier 1: Definite Zombie (0-20)
- 🧟‍♂️ Tier 2: Suspected Zombie (21-40)
- ⚠️ Tier 3: Unknown (41-60)
- 💤 Tier 4: Dormant (61-80)
- ✅ Active (81-100)

## Evidence

### What We Found
[Description of why this is a zombie candidate]

### Activity Log (last 90 days)
[Paste activity log output or "No activity found"]

### Dependencies
[List any dependencies found or "No dependencies"]

### Cost
[Monthly cost and what it's paying for]

## Recommendation

**Action:** [DELETE | VERIFY | INVESTIGATE | KEEP]

**Reasoning:** [Why this action is recommended]

**Risk if Deleted:** [What could break]

**Verification Needed:** [Who to ask, what to check]

## Decommission Steps (if DELETE)

- [ ] Verify no dependencies (check TF, k8s, configs)
- [ ] Confirm with team/owner
- [ ] Snapshot or backup if needed
- [ ] Delete resource: `az [resource-type] delete --name [name] --resource-group [rg]`
- [ ] Verify deletion in Azure portal
- [ ] Update context graph (remove node if exists)
- [ ] Record savings: $[amount]/month
```

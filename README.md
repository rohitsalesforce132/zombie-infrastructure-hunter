# Zombie Infrastructure Hunter

> **Find infrastructure that has no documented purpose, no owner, and no reason to exist. Kill it. Save money.**

**Uses:** [[Context Graph Terraform]] for resource nodes, [[Knowledge Weaving Patterns]] for gap detection, [[Agentic Knowledge Mesh]] for entity resolution.

---

## The Problem

Every big org has zombie infrastructure:

- Resources provisioned 18 months ago for a project that was cancelled
- Dev environments that nobody has touched in 6 months
- Old storage accounts from a migration that's complete
- Test clusters that were "temporary" but never torn down
- Resources created in the Azure Portal (bypassing Terraform) with no documentation

**These resources cost money every month. Nobody knows why they exist. Nobody wants to delete them because nobody knows what will break.**

Current state:
- Azure Cost Management shows you're spending money
- Terraform state shows what TF manages
- **But nobody shows you what has NO documented purpose**

---

## The Solution

**Zombie Infrastructure Hunter** — Cross-references Azure resources against Context Graph nodes and Terraform state to identify resources with:
- No context graph node (undocumented purpose)
- No owner or stale ownership
- No dependency chain (nothing depends on it)
- No incident history (never been important enough to break)
- No recent activity (no changes, no accesses)

These are **zombies**. They should be investigated and potentially decommissioned.

---

## Zombie Classification

### Tier 1: Definite Zombie 🧟
- No context graph node
- No Terraform state (not managed by IaC)
- No owner tags
- No activity in 90+ days
- **Action: Schedule for deletion**

### Tier 2: Suspected Zombie 🧟‍♂️
- Has context graph node but stale (no updates in 6+ months)
- Has owner tag but owner left team/org
- No incident history (never been important)
- **Action: Verify with owner, then schedule**

### Tier 3: Dormant (Not Zombie) 💤
- Has context graph node, fresh
- Has owner, still in org
- No recent activity BUT documented purpose (DR, backup, compliance)
- **Action: Keep, but document WHY it's dormant**

### Tier 4: Unknown ⚠️
- Exists in Azure but not in TF or Context Graph
- Created outside IaC process
- **Action: Investigate — document or delete**

---

## Architecture

```mermaid
graph TD
    subgraph "Data Sources"
        AZURE[Azure Resource Graph<br/>All resources in subscription]
        TF[Terraform State<br/>Managed resources]
        CG[Context Graph Nodes<br/>Documented resources]
        ACTIVITY[Azure Activity Log<br/>Recent changes]
        COST[Azure Cost Management<br/>Monthly spend]
    end

    subgraph "Zombie Hunter Engine"
        COMPARE[Cross-Reference<br/>AZURE ∩ TF ∩ CG]
        SCORE[Zombie Scorer<br/>5 dimensions]
        CLASSIFY[Classifier<br/>Tier 1-4]
        REPORT[Report Generator<br/>Markdown output]
    end

    AZURE --> COMPARE
    TF --> COMPARE
    CG --> COMPARE
    ACTIVITY --> SCORE
    COST --> SCORE
    COMPARE --> SCORE
    SCORE --> CLASSIFY
    CLASSIFY --> REPORT
end
```

---

## Zombie Score Dimensions

Each resource scored on 5 dimensions (0-20 each, total 0-100):

| Dimension | Weight | How to Check | Data Source |
|-----------|--------|-------------|-------------|
| **Documentation** | 20pts | Does a context graph node exist? | Context Graph nodes/ |
| **Ownership** | 20pts | Is there an owner tag? Is the owner still active? | Azure tags + Activity Log |
| **Dependencies** | 20pts | Does anything depend on this? | TF state + k8s manifests |
| **Activity** | 20pts | Any changes or accesses in 90 days? | Azure Activity Log |
| **Incident Relevance** | 20pts | Has it been involved in any incident? | Context Graph incidents/ |

### Zombie Score Interpretation

| Score | Classification | Action |
|-------|---------------|--------|
| 0-20 | 🧟 Definite Zombie | Schedule for deletion |
| 21-40 | 🧟‍♂️ Suspected Zombie | Verify with team |
| 41-60 | ⚠️ Unknown | Investigate and document |
| 61-80 | 💤 Dormant | Keep, document purpose |
| 81-100 | ✅ Active | Healthy resource |

---

## Project Structure

```
zombie-infrastructure-hunter/
├── README.md
├── LICENSE
├── CONTRIBUTING.md
├── queries/                          # Azure + TF queries for data extraction
│   ├── azure-resource-list.md        # List all Azure resources
│   ├── azure-orphan-detection.md     # Find resources with no dependencies
│   ├── azure-cost-waste.md           # Cost analysis for potential waste
│   ├── tf-state-audit.md             # Audit TF state vs Azure reality
│   └── activity-log-analysis.md      # Check recent activity per resource
├── engine/                           # Scoring and classification
│   ├── zombie-score.md               # Scoring algorithm (5 dimensions)
│   ├── classifier.md                 # Tier classification rules
│   └── gap-detector.md               # Gap detection using weaving patterns
├── reports/                          # Generated zombie reports
│   ├── template.md                   # Report template
│   └── sample-report.md             # Sample report for prod subscription
├── templates/                        # Templates
│   ├── zombie-resource.md            # Individual zombie finding
│   └── decommission-plan.md          # Decommission checklist
└── samples/                          # Sample data
    └── prod-subscription/
        ├── zombie-report.md
        ├── tier-1-definite.md
        └── cost-savings.md
```

---

## Azure Queries

### List All Resources

```bash
# List all resources in subscription
az resource list --output json

# List by resource group
az resource list --resource-group rg-production --output json

# List with tags (ownership info)
az resource list --query '[].{name:name, type:type, location:location, tags:tags}' --output json

# List resources created more than 90 days ago
az resource list --query "[?createdTime < '2026-01-16']" --output json
```

### Find Orphan Resources

```bash
# Resources with no owner tag
az resource list --query "[?tags.owner == null]" --output json

# Resources with owner who left (manual check)
az resource list --query "[?tags.owner == 'former-employee']" --output json

# Resources not in any TF state
# Compare: az resource list vs terraform state list
```

### Cost Analysis

```bash
# Cost by resource group
az costmanagement query --type ActualCost --timeframe MonthToDate --scope "/subscriptions/{sub-id}"

# Cost by resource (needs cost allocation tags)
az consumption usage list --top 100 --query '[].{name:instanceName, cost:pretaxCost}'

# Identify top spenders
az costmanagement query --type ActualCost --timeframe MonthToDate
```

### Activity Check

```bash
# Recent activity on a resource
az monitor activity-log list --resource-group rg-production --caller any --max-events 100

# Resources with NO activity in 90 days
az monitor activity-log list --resource-group rg-production --start-time 2026-01-16 --max-events 1000
```

---

## Terraform State Audit

### TF vs Azure Comparison

```bash
# List all resources in TF state
terraform state list

# List all resources in Azure
az resource list --output json | jq '.[].name'

# Find Azure resources NOT in TF state (created outside IaC)
# Manual: diff the two lists
```

### TF State Freshness

```bash
# When was TF state last updated?
terraform state pull | jq '.serial, .lineage'

# Check for drifted resources
terraform plan -detailed-exitcode
```

---

## Gap Detection (from Knowledge Weaving Patterns)

Using the gap detection patterns from [[Agentic Knowledge Mesh]]:

| Gap Type | Zombie Signal | Check |
|----------|-------------|-------|
| Orphan Entity | Resource exists but no context graph node | `ls nodes/` vs `az resource list` |
| Stale Decision | Context node exists but last update >6 months | Check node change history |
| Missing Owner | No owner tag OR owner left org | Azure tags + org directory |
| No Dependency | Nothing in TF state depends on it | `terraform graph` analysis |
| No Incidents | Never appeared in any incident replay | Search incident replays |

---

## Sample Zombie Report

```markdown
# Zombie Infrastructure Report: Production Subscription

**Generated:** 2026-04-16
**Subscription:** AT&T Production (xxx-xxx-xxx)
**Total Resources:** 247
**Zombie Candidates:** 23 (9.3%)

---

## Summary

| Tier | Count | Est. Monthly Waste | Action |
|------|-------|-------------------|--------|
| 🧟 Tier 1 (Definite) | 8 | $4,200/month | Schedule deletion |
| 🧟‍♂️ Tier 2 (Suspected) | 7 | $2,800/month | Verify with teams |
| ⚠️ Tier 3 (Unknown) | 5 | $1,400/month | Investigate |
| 💤 Tier 4 (Dormant) | 3 | $600/month | Document purpose |
| **Total** | **23** | **$9,000/month** | **$108,000/year** |

---

## Tier 1: Definite Zombies 🧟

### 1. rg-old-migration (Resource Group)
- **Type:** Microsoft.Resources/resourceGroups
- **Location:** East US
- **Created:** 2025-06-15 (10 months ago)
- **Context Graph Node:** ❌ None
- **Owner Tag:** ❌ None
- **TF State:** ❌ Not managed
- **Activity (90d):** ❌ None
- **Incidents:** ❌ None
- **Monthly Cost:** $1,200
- **Contents:** 4 storage accounts, 2 VMs (stopped), 1 SQL database
- **Recommendation:** DELETE — Migration completed Oct 2025. Resources never cleaned up.

### 2. dev-test-cluster (AKS)
- **Type:** Microsoft.ContainerService/managedClusters
- **Location:** Central US
- **Created:** 2025-08-20 (8 months ago)
- **Context Graph Node:** ❌ None
- **Owner Tag:** developer-c (left team Feb 2026)
- **TF State:** ❌ Not managed (created via portal)
- **Activity (90d):** ❌ No deployments, no connections
- **Incidents:** ❌ None
- **Monthly Cost:** $800
- **Node Count:** 3 (all idle)
- **Recommendation:** DELETE — Developer left, cluster unused.

### 3. storage-logs-archive-2024
- **Type:** Microsoft.Storage/storageAccounts
- **Location:** East US 2
- **Created:** 2024-11-01 (17 months ago)
- **Context Graph Node:** ❌ None
- **Owner Tag:** ❌ None
- **TF State:** ❌ Not managed
- **Activity (90d):** ❌ No read/write operations
- **Incidents:** ❌ None
- **Monthly Cost:** $150
- **Size:** 2.3 TB of old logs
- **Recommendation:** DELETE — Logs from 2024 no longer needed. Archive to cold storage if compliance requires.

---

## Tier 2: Suspected Zombies 🧟‍♂️

### 4. staging-load-balancer
- **Type:** Microsoft.Network/loadBalancers
- **Context Graph Node:** ⚠️ Exists but stale (last updated 5 months ago)
- **Owner Tag:** platform-team
- **TF State:** ✅ Managed
- **Activity (90d):** ⚠️ Minimal (2 config reads)
- **Incidents:** ❌ None
- **Monthly Cost:** $200
- **Recommendation:** VERIFY with platform-team — staging environment may have moved.

---

## Cost Savings Summary

| Action | Resources | Monthly Savings | Annual Savings |
|--------|-----------|----------------|---------------|
| Delete Tier 1 | 8 | $4,200 | $50,400 |
| Delete confirmed Tier 2 | ~4 | $1,600 | $19,200 |
| Investigate Tier 3 | 5 | TBD | TBD |
| **Total Potential** | **12-17** | **$5,800-9,000** | **$69,600-108,000** |
```

---

## Integration with Existing Stack

### Context Graph Terraform
- Resource nodes provide documentation scores
- Drift briefs show unmanaged resources (potential zombies)
- Decision traces explain WHY resources exist

### Agentic Knowledge Mesh
- Gap detection identifies orphan entities
- Entity resolution connects Azure resources to context nodes
- Knowledge weaving finds relationships across sources

### Change Correlator
- Activity log analysis feeds into zombie scoring
- Resources with no recent changes = potential zombies

### Crisis Command Center
- During incidents: "Are any zombie resources causing interference?"
- Post-incident: check if affected resources are documented

---

## Roadmap

**Phase 1 (MVP):**
- Azure resource listing queries
- TF state comparison
- Context graph cross-reference
- Zombie scoring algorithm
- Report template

**Phase 2:**
- Auto-generate reports from Azure MCP (live data)
- Activity log integration
- Cost analysis integration
- Owner verification against org directory

**Phase 3:**
- Automated decommission plans (checklist per zombie)
- Scheduled monthly zombie scans
- Integration with Azure Advisor recommendations
- Slack notifications for new zombie detections

**Phase 4:**
- Predictive zombie detection (resources trending toward zombie status)
- Auto-creation of context graph nodes for newly provisioned resources
- Cost optimization recommendations (right-size, spot instances, reserved capacity)

---

## Why This Matters

**For AT&T:**
- Large subscriptions accumulate zombies
- $100K+/year in potential savings is realistic
- Compliance: undocumented resources are audit risks
- Security: unmanaged resources may have stale permissions

**For Manav:**
- Directly uses his Azure + Terraform expertise
- Demonstrates cost optimization thinking
- Uses the entire context graph stack
- Immediate, measurable ROI ($$$)

**For Portfolio:**
- "I saved $108K/year by finding zombie infrastructure using context graphs"
- Combines Azure, Terraform, knowledge management, and SRE
- Practical, buildable, and impressive

---

## Philosophy

> **If you can't explain why a resource exists, it probably shouldn't.**

# Activity Log Analysis

> Check which resources have been accessed or modified recently.

## Recent Activity per Resource

```bash
# All activity in last 7 days
az monitor activity-log list --start-time $(date -d '7 days ago' +%Y-%m-%dT%H:%M:%SZ) \
    --max-events 500 --query '[].{Time:eventTimestamp, Resource:resourceGroupName, Action:operationName, Caller:caller}' -o table

# Activity for specific resource group
az monitor activity-log list --resource-group rg-production --max-events 100 \
    --query '[].{Time:eventTimestamp, Action:operationName, Caller:caller}' -o table

# Activity for specific resource
az monitor activity-log list --resource "dev-test-cluster" --max-events 20 \
    --query '[].{Time:eventTimestamp, Action:operationName}' -o table
```

## Resources with NO Activity

```bash
# Find resources with zero activity in last 90 days
START=$(date -d '90 days ago' +%Y-%m-%dT%H:%M:%SZ)

# Get all resources
az resource list --query '[].name' -o tsv | while read name; do
    activity=$(az monitor activity-log list --resource "$name" --start-time "$START" --max-events 1 --query 'length([])' -o tsv 2>/dev/null || echo "0")
    if [ "$activity" = "0" ]; then
        echo "NO ACTIVITY (90d): $name"
    fi
done
```

## Activity by Caller (Who's Using What)

```bash
# Most active callers
az monitor activity-log list --max-events 1000 --query '[].caller' -o tsv | sort | uniq -c | sort -rn | head -20

# Activity by a specific person
az monitor activity-log list --caller "user@domain.com" --max-events 100 \
    --query '[].{Time:eventTimestamp, Resource:resourceGroupName, Action:operationName}' -o table

# Activity by service principals (automated actions)
az monitor activity-log list --max-events 1000 \
    --query "[?contains(caller, 'servicePrincipal')].{Time:eventTimestamp, Caller:caller, Action:operationName}" -o table
```

## Failed Operations

```bash
# Recent failed operations (potential issues)
az monitor activity-log list --max-events 500 \
    --query "[?eventStatus.value == 'Failed'].{Time:eventTimestamp, Resource:resourceGroupName, Action:operationName, Status:eventStatus.value}" -o table

# Delete operations (who deleted what?)
az monitor activity-log list --max-events 500 \
    --query "[?contains(operationName.value, 'Delete')].{Time:eventTimestamp, Resource:resourceGroupName, Action:operationName.value, Caller:caller}" -o table
```

## Write Operations (Changes)

```bash
# All write operations in last 7 days
az monitor activity-log list --max-events 1000 \
    --query "[?contains(operationName.value, 'Write') || contains(operationName.value, 'Create') || contains(operationName.value, 'Update')].{Time:eventTimestamp, Resource:resourceGroupName, Action:operationName.value}" -o table

# Resources modified recently (NOT zombies)
az monitor activity-log list --max-events 500 \
    --query "[?contains(operationName.value, 'Write')].resourceGroupName" -o tsv | sort -u
```

## Dormancy Check

```bash
# For a specific resource, check last activity timestamp
az monitor activity-log list --resource "prod-cluster" --max-events 1 \
    --query '[0].{LastActivity:eventTimestamp, Action:operationName}' -o table

# Compare: last activity vs creation time
az resource show --ids $RESOURCE_ID --query '{Created:createdTime, Name:name}' -o json
az monitor activity-log list --resource $RESOURCE_NAME --max-events 1 --query '[0].eventTimestamp' -o tsv
```

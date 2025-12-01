---
layout: default
title: Auto-Purge
parent: Durable Task Scheduler
nav_order: 5
permalink: /docs/durable-task-scheduler/auto-purge/
---

# Auto-Purge Retention Policies
{: .no_toc }

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

To prevent reaching the memory limit of a capacity unit (CU), it's best practice to periodically purge orchestration history data. The Durable Task Scheduler offers a lightweight, configurable autopurge feature that helps you manage orchestration data clean-up without manual intervention.
{: .fs-6 .fw-300 }

---

## Overview

Autopurge operates asynchronously in the background, optimized to minimize system resource usage and prevent interference with other Durable Task operations. Although autopurge doesn't adhere to a strict schedule, its clean-up rate generally aligns with your orchestration scheduling rate.

### How It Works

Autopurge is an **opt-in** feature. You enable it by defining retention policies that control how long to keep the data of orchestrations in certain statuses.

The autopurge feature only purges orchestration data associated with **terminal statuses**—orchestrations that have reached a final state with no further scheduling, event processing, or work item generation.

### Terminal Statuses (Eligible for Purge)

| Status | Description |
|--------|-------------|
| `Completed` | Successfully finished orchestrations |
| `Failed` | Orchestrations that ended with an error |
| `Canceled` | Orchestrations that were canceled |
| `Terminated` | Orchestrations that were forcefully terminated |

### Non-Terminal Statuses (Not Purged)

Autopurge ignores orchestration data associated with non-terminal statuses:

| Status | Description |
|--------|-------------|
| `Pending` | Orchestrations waiting to start |
| `Running` | Currently executing orchestrations |
| `Suspended` | Paused orchestrations that may resume |
| `Continued_As_New` | Orchestrations that have continued as new instances |

---

## Retention Policy Configuration

### Policy Value

Retention values specify the number of **days** to retain orchestration data:

| Setting | Value |
|---------|-------|
| **Minimum** | 0 (purge as soon as possible) |
| **Default** | 30 days |
| **Maximum** | 90 days |

{: .note }
> The retention period refers to the time since the orchestration entered a terminal state. For example, if you set a retention value of 1 day and an orchestration takes 10 days to finish, autopurge won't delete it until the day after it completes.

### Types of Policies

#### Default Policy

A default policy purges orchestration data **regardless** of status:

```json
{
  "retentionPeriodInDays": 2
}
```

This purges all terminal orchestration data after 2 days.

#### Specific Policies

Specific policies define purging for particular orchestration states, overriding the default:

```json
{
  "retentionPeriodInDays": 1,
  "orchestrationState": "Completed"
}
```

This keeps completed orchestration data for 1 day only.

### Combining Policies

You can combine default and specific policies. Specific policies override the default for their targeted status:

```json
[
  {
    "retentionPeriodInDays": 1
  },
  {
    "retentionPeriodInDays": 0,
    "orchestrationState": "Completed"
  },
  {
    "retentionPeriodInDays": 60,
    "orchestrationState": "Failed"
  }
]
```

**Result:**
- `Completed` orchestrations are purged immediately (0 days)
- `Failed` orchestrations are retained for 60 days
- `Canceled` and `Terminated` orchestrations use the default (1 day)

{: .important }
> Retention policies are applied to **all** task hubs in a scheduler.

---

## Enable Autopurge

### Using Durable Task CLI

First, ensure you have the latest CLI extension:

```bash
az extension add --name durabletask
az extension update --name durabletask
```

Create or update the retention policy:

```bash
az durabletask retention-policy create \
  --scheduler-name SCHEDULER_NAME \
  --resource-group RESOURCE_GROUP \
  --default-days 1 \
  --completed-days 0 \
  --failed-days 60
```

**Available Parameters:**

| Parameter | Description |
|-----------|-------------|
| `--default-days` or `-d` | Default retention for all terminal orchestrations |
| `--completed-days` or `-c` | Retention for completed orchestrations |
| `--failed-days` or `-f` | Retention for failed orchestrations |
| `--canceled-days` or `-x` | Retention for canceled orchestrations |
| `--terminated-days` or `-t` | Retention for terminated orchestrations |

**Example Response:**

```json
{
  "id": "/subscriptions/.../retentionPolicies/default",
  "name": "default",
  "properties": {
    "provisioningState": "Succeeded",
    "retentionPolicies": [
      { "retentionPeriodInDays": 1 },
      { "orchestrationState": "Completed", "retentionPeriodInDays": 0 },
      { "orchestrationState": "Failed", "retentionPeriodInDays": 60 }
    ]
  },
  "type": "microsoft.durabletask/schedulers/retentionpolicies"
}
```

### Using Azure Resource Manager (ARM)

```http
PUT https://management.azure.com/subscriptions/{subscriptionId}/resourceGroups/{resourceGroup}/providers/Microsoft.DurableTask/schedulers/{schedulerName}/retentionPolicies/default?api-version=2025-04-01-preview

{
  "properties": {
    "retentionPolicies": [
      {
        "retentionPeriodInDays": 1
      },
      {
        "retentionPeriodInDays": 0,
        "orchestrationState": "Completed"
      },
      {
        "retentionPeriodInDays": 60,
        "orchestrationState": "Failed"
      }
    ]
  }
}
```

### Using Bicep

```bicep
resource retentionPolicy 'Microsoft.DurableTask/schedulers/retentionPolicies@2025-04-01-preview' = {
  parent: scheduler
  name: 'default'
  properties: {
    retentionPolicies: [
      {
        retentionPeriodInDays: 1
      }
      {
        retentionPeriodInDays: 0
        orchestrationState: 'Completed'
      }
      {
        retentionPeriodInDays: 60
        orchestrationState: 'Failed'
      }
    ]
  }
}
```

---

## Disable Autopurge

### Using Durable Task CLI

```bash
az durabletask retention-policy delete \
  --scheduler-name SCHEDULER_NAME \
  --resource-group RESOURCE_GROUP
```

The Durable Task Scheduler stops cleaning orchestration data within 5-10 minutes.

### Using Azure Resource Manager

```http
DELETE https://management.azure.com/subscriptions/{subscriptionId}/resourceGroups/{resourceGroup}/providers/Microsoft.DurableTask/schedulers/{schedulerName}/retentionPolicies/default?api-version=2025-04-01-preview
```

### Using Bicep

Remove the `retentionPolicies` resource from your Bicep file.

---

## Best Practices

### ✅ Recommended

| Practice | Reason |
|----------|--------|
| **Enable autopurge in production** | Prevents storage bloat and maintains performance |
| **Set shorter retention for completed orchestrations** | Completed orchestrations rarely need long retention |
| **Set longer retention for failed orchestrations** | Allows time for debugging and investigation |
| **Monitor storage usage** | Track CU utilization to adjust policies |

### ⚠️ Considerations

| Consideration | Details |
|---------------|---------|
| **Compliance requirements** | Some regulations may require longer retention |
| **Debugging needs** | Ensure sufficient time to investigate issues |
| **Storage costs** | Longer retention increases storage usage |
| **Audit trails** | Export important data before purging |

---

## Example Scenarios

### Development Environment

Aggressive purging to save resources:

```bash
az durabletask retention-policy create \
  --scheduler-name dev-scheduler \
  --resource-group dev-rg \
  --default-days 0
```

### Production Environment

Balance between debugging needs and storage efficiency:

```bash
az durabletask retention-policy create \
  --scheduler-name prod-scheduler \
  --resource-group prod-rg \
  --default-days 7 \
  --completed-days 1 \
  --failed-days 30
```

### Compliance-Heavy Environment

Longer retention for audit purposes:

```bash
az durabletask retention-policy create \
  --scheduler-name compliance-scheduler \
  --resource-group compliance-rg \
  --default-days 90
```

---

## Next Steps

- [Pricing and SKUs →](./pricing.md)
- [Dashboard →](./dashboard.md)
- [Troubleshooting →](./troubleshooting.md)

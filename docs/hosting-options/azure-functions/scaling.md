---
layout: default
title: Scaling
parent: Azure Functions (Durable Functions)
grand_parent: Hosting Options
nav_order: 5
permalink: /docs/hosting-options/azure-functions/scaling/
---

# Durable Functions Scaling
{: .no_toc }

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

Learn how to configure scaling for Azure Durable Functions to handle varying workloads efficiently.

---

## Overview

Azure Functions provides automatic scaling capabilities that work seamlessly with Durable Functions. The scaling behavior depends on your chosen hosting plan and storage backend.

---

## Hosting Plans Comparison

| Hosting Plan | Scale Range | Scale to Zero | Best For |
|--------------|-------------|---------------|----------|
| **Consumption** | 0 → 200 instances | ✅ Yes | Event-driven, variable workloads |
| **Flex Consumption** | 0 → 1000 instances | ✅ Yes | High-scale serverless with VNet |
| **Premium** | 0 → 100 instances | ⚠️ Optional | Pre-warmed, consistent performance |
| **Dedicated** | Fixed | ❌ No | Predictable workloads |
| **Container Apps** | 0 → 300 replicas | ✅ Yes | Containerized workloads |

---

## Consumption Plan Scaling

The Consumption plan automatically scales based on the number of incoming events. For Durable Functions:

- **Orchestrator functions** scale based on control queue depth
- **Activity functions** scale based on work-item queue depth
- **Entity functions** scale based on entity queue depth

### Configuration

```json
// host.json
{
  "version": "2.0",
  "extensions": {
    "durableTask": {
      "maxConcurrentActivityFunctions": 10,
      "maxConcurrentOrchestratorFunctions": 10
    }
  }
}
```

### Key Settings

| Setting | Default | Description |
|---------|---------|-------------|
| `maxConcurrentActivityFunctions` | 10 × CPU cores | Max concurrent activity executions per instance |
| `maxConcurrentOrchestratorFunctions` | 10 × CPU cores | Max concurrent orchestrator executions per instance |
| `maxConcurrentEntityFunctions` | 10 × CPU cores | Max concurrent entity operations per instance |

---

## Flex Consumption Plan Scaling

Flex Consumption provides enhanced scaling with faster scale-out and VNet support.

### Configuration

```json
// host.json
{
  "version": "2.0",
  "extensions": {
    "durableTask": {
      "storageProvider": {
        "type": "azureManaged",
        "connectionStringName": "DurableTaskSchedulerConnection"
      }
    }
  },
  "concurrency": {
    "dynamicConcurrencyEnabled": true,
    "snapshotPersistenceEnabled": true
  }
}
```

### Benefits

- **Faster scale-out**: Up to 1000 instances
- **VNet integration**: Secure networking
- **Always-ready instances**: Pre-provisioned for instant response
- **Per-function scaling**: Independent scaling per function

---

## Premium Plan Scaling

The Premium plan provides pre-warmed instances for consistent performance.

### Configuration

```json
// host.json
{
  "version": "2.0",
  "extensions": {
    "durableTask": {
      "maxConcurrentActivityFunctions": 20,
      "maxConcurrentOrchestratorFunctions": 20
    }
  }
}
```

### Pre-Warmed Instances

Configure always-ready instances to avoid cold starts:

```bash
az functionapp update \
  --name my-function-app \
  --resource-group rg-durable \
  --min-instances 1
```

---

## Storage Backend Impact on Scaling

### Durable Task Scheduler

The Durable Task Scheduler provides the best scaling characteristics:

- **No partition limits**: Unlimited horizontal scaling
- **Automatic load balancing**: Work distributed across instances
- **High throughput**: Optimized for performance

### Azure Storage

Azure Storage has partition-based scaling:

- **16 partitions maximum**: Limits scale-out
- **Partition key distribution**: Affects performance
- **Blob lease coordination**: Controls partition assignment

### MSSQL

SQL-based backend with different characteristics:

- **Database-limited**: Scaling depends on SQL tier
- **Connection pooling**: Important for performance
- **Lock contention**: Can affect high-concurrency scenarios

---

## Performance Tuning

### Orchestrator Performance

```json
{
  "extensions": {
    "durableTask": {
      "maxConcurrentOrchestratorFunctions": 20,
      "extendedSessionsEnabled": true,
      "extendedSessionIdleTimeoutInSeconds": 30
    }
  }
}
```

| Setting | Impact |
|---------|--------|
| `extendedSessionsEnabled` | Keeps orchestrator in memory between replays |
| `extendedSessionIdleTimeoutInSeconds` | How long to keep session active |

### Activity Performance

```json
{
  "extensions": {
    "durableTask": {
      "maxConcurrentActivityFunctions": 50,
      "maxQueuePollingInterval": "00:00:05"
    }
  }
}
```

| Setting | Impact |
|---------|--------|
| `maxConcurrentActivityFunctions` | Parallel activity execution per instance |
| `maxQueuePollingInterval` | How often to check for new work |

---

## Monitoring Scaling

### Application Insights Metrics

Monitor these metrics to understand scaling behavior:

| Metric | Description |
|--------|-------------|
| `durable_task_orchestration_count` | Active orchestrations |
| `durable_task_activity_count` | Active activities |
| `durable_task_queue_depth` | Pending work items |
| `functionapp_instance_count` | Current instance count |

### Kusto Query Examples

```kusto
// Orchestration throughput
customMetrics
| where name == "durable_task_orchestration_completed"
| summarize count() by bin(timestamp, 1m)
| render timechart

// Instance scaling
performanceCounters
| where name == "functionAppInstanceCount"
| summarize avg(value) by bin(timestamp, 1m)
| render timechart

// Queue depth over time
customMetrics
| where name == "durable_task_queue_depth"
| summarize max(value) by bin(timestamp, 1m)
| render timechart
```

---

## Best Practices

### 1. Right-Size Concurrency

Set concurrency based on your workload:

```json
{
  "extensions": {
    "durableTask": {
      // For I/O-bound activities
      "maxConcurrentActivityFunctions": 50,
      // For CPU-bound activities
      // "maxConcurrentActivityFunctions": 4
    }
  }
}
```

### 2. Use Fan-Out Wisely

Limit parallel activities to avoid overwhelming downstream services:

```csharp
// Good: Batched fan-out
var batches = workItems.Chunk(100);
foreach (var batch in batches)
{
    var tasks = batch.Select(item => 
        context.CallActivityAsync("ProcessItem", item));
    await Task.WhenAll(tasks);
}

// Avoid: Unbounded fan-out
var tasks = workItems.Select(item => 
    context.CallActivityAsync("ProcessItem", item));
await Task.WhenAll(tasks); // Could be thousands!
```

### 3. Configure Retry Policies

Prevent cascading failures:

```csharp
var retryOptions = new TaskOptions(new TaskRetryOptions(
    new RetryPolicy(
        maxNumberOfAttempts: 3,
        firstRetryInterval: TimeSpan.FromSeconds(5),
        backoffCoefficient: 2.0
    )
));

await context.CallActivityAsync("ProcessItem", item, retryOptions);
```

### 4. Monitor and Alert

Set up alerts for scaling issues:

```bash
az monitor metrics alert create \
  --name "High Queue Depth" \
  --resource-group rg-durable \
  --scopes /subscriptions/.../resourceGroups/rg-durable/providers/Microsoft.Web/sites/my-function-app \
  --condition "avg durable_task_queue_depth > 1000" \
  --window-size 5m \
  --evaluation-frequency 1m
```

---

## Troubleshooting

### Scaling Not Triggering

| Symptom | Cause | Solution |
|---------|-------|----------|
| Instances not increasing | Concurrency too high | Lower `maxConcurrentActivityFunctions` |
| Slow scale-out | Consumption plan cold start | Consider Premium plan |
| Queue depth growing | Insufficient instances | Increase max instances or plan |

### Performance Issues

| Symptom | Cause | Solution |
|---------|-------|----------|
| High latency | Too many concurrent functions | Reduce concurrency settings |
| Timeouts | Activity taking too long | Optimize activity code |
| Memory pressure | Large payloads | Reduce payload size |

---

## Next Steps

- [Configure Deployment →](./deployment.md)
- [Learn about Patterns →](../../patterns/index.md)
- [Monitor with Dashboard →](../../durable-task-scheduler/dashboard.md)

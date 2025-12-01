---
layout: default
title: Fan-Out/Fan-In
parent: Orchestration Patterns
nav_order: 2
permalink: /docs/patterns/fan-out-fan-in/
---

# Fan-Out/Fan-In Pattern
{: .no_toc }

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

Execute multiple activities in parallel and wait for all of them to complete before continuing.

---

## Overview

The fan-out/fan-in pattern allows you to execute multiple activities concurrently and then aggregate their results. This is ideal for batch processing, parallel operations, and scenarios where work can be divided.

```
┌──────────────────────────────────────────────────────────────────┐
│                       FAN-OUT / FAN-IN                            │
├──────────────────────────────────────────────────────────────────┤
│                                                                   │
│                         Input (List)                              │
│                              │                                    │
│                              ▼                                    │
│                    ┌─────────────────┐                           │
│                    │   Orchestrator   │                           │
│                    │    (Fan-Out)     │                           │
│                    └────────┬────────┘                           │
│                             │                                     │
│               ┌─────────────┼─────────────┐                      │
│               │             │             │                       │
│               ▼             ▼             ▼                       │
│         ┌─────────┐   ┌─────────┐   ┌─────────┐                  │
│         │Activity │   │Activity │   │Activity │  ... N tasks     │
│         │  Item 1 │   │  Item 2 │   │  Item 3 │                  │
│         └────┬────┘   └────┬────┘   └────┬────┘                  │
│              │             │             │                        │
│              └─────────────┼─────────────┘                       │
│                            │                                      │
│                            ▼                                      │
│                   ┌─────────────────┐                            │
│                   │   Orchestrator   │                            │
│                   │    (Fan-In)      │                            │
│                   └────────┬────────┘                            │
│                            │                                      │
│                            ▼                                      │
│                      Aggregated Result                            │
│                                                                   │
└──────────────────────────────────────────────────────────────────┘
```

---

## Use Cases

- **Image Processing**: Resize multiple images in parallel
- **Report Generation**: Generate sections concurrently, then combine
- **Data Validation**: Validate multiple records simultaneously
- **API Aggregation**: Call multiple APIs and combine responses
- **Batch Processing**: Process items from a queue in parallel

---

## Implementation

### C# (.NET)

```csharp
using Microsoft.DurableTask;

[DurableTask(nameof(BatchProcessingOrchestrator))]
public class BatchProcessingOrchestrator : TaskOrchestrator<List<WorkItem>, BatchResult>
{
    public override async Task<BatchResult> RunAsync(
        TaskOrchestrationContext context, 
        List<WorkItem> workItems)
    {
        // Fan-out: Schedule all activities in parallel
        var tasks = workItems.Select(item =>
            context.CallActivityAsync<ItemResult>(
                nameof(ProcessItemActivity),
                item
            )
        ).ToList();
        
        // Fan-in: Wait for all to complete
        var results = await Task.WhenAll(tasks);
        
        // Aggregate results
        return new BatchResult
        {
            TotalProcessed = results.Length,
            SuccessCount = results.Count(r => r.Success),
            FailureCount = results.Count(r => !r.Success),
            Results = results.ToList()
        };
    }
}

// Activity
[DurableTask(nameof(ProcessItemActivity))]
public class ProcessItemActivity : TaskActivity<WorkItem, ItemResult>
{
    public override async Task<ItemResult> RunAsync(
        TaskActivityContext context, 
        WorkItem item)
    {
        // Process the individual item
        var processed = await ProcessAsync(item);
        
        return new ItemResult
        {
            ItemId = item.Id,
            Success = true,
            Data = processed
        };
    }
}
```

### Python

```python
from durabletask import task

@task.orchestrator
def batch_processing_orchestrator(ctx: task.OrchestrationContext, work_items: list):
    """Process multiple items in parallel."""
    
    # Fan-out: Create parallel tasks
    parallel_tasks = []
    for item in work_items:
        t = ctx.call_activity("process_item", input=item)
        parallel_tasks.append(t)
    
    # Fan-in: Wait for all tasks to complete
    results = yield task.when_all(parallel_tasks)
    
    # Aggregate results
    success_count = sum(1 for r in results if r["success"])
    
    return {
        "total_processed": len(results),
        "success_count": success_count,
        "failure_count": len(results) - success_count,
        "results": results
    }

@task.activity
def process_item(ctx: task.ActivityContext, item: dict) -> dict:
    """Process a single item."""
    
    # Perform processing
    processed_data = transform(item["data"])
    
    return {
        "item_id": item["id"],
        "success": True,
        "data": processed_data
    }
```

### Java

```java
public class BatchProcessingOrchestrator implements TaskOrchestrator {
    @Override
    public BatchResult run(TaskOrchestrationContext ctx) {
        List<WorkItem> workItems = ctx.getInput(new TypeReference<List<WorkItem>>() {});
        
        // Fan-out: Create tasks for all items
        List<Task<ItemResult>> tasks = new ArrayList<>();
        for (WorkItem item : workItems) {
            Task<ItemResult> task = ctx.callActivity(
                "ProcessItem",
                item,
                ItemResult.class
            );
            tasks.add(task);
        }
        
        // Fan-in: Wait for all to complete
        List<ItemResult> results = ctx.allOf(tasks).await();
        
        // Aggregate
        long successCount = results.stream()
            .filter(ItemResult::isSuccess)
            .count();
        
        return new BatchResult(
            results.size(),
            (int) successCount,
            results.size() - (int) successCount,
            results
        );
    }
}
```

---

## Advanced Variations

### Chunked Processing

Process items in batches to control parallelism:

```csharp
public override async Task<BatchResult> RunAsync(
    TaskOrchestrationContext context, 
    List<WorkItem> workItems)
{
    const int chunkSize = 10;
    var allResults = new List<ItemResult>();
    
    // Process in chunks
    foreach (var chunk in workItems.Chunk(chunkSize))
    {
        var tasks = chunk.Select(item =>
            context.CallActivityAsync<ItemResult>(
                nameof(ProcessItemActivity),
                item
            )
        );
        
        var chunkResults = await Task.WhenAll(tasks);
        allResults.AddRange(chunkResults);
    }
    
    return new BatchResult { Results = allResults };
}
```

### With Progress Tracking

```python
@task.orchestrator
def tracked_batch_orchestrator(ctx: task.OrchestrationContext, work_items: list):
    """Process items with progress updates."""
    
    total = len(work_items)
    results = []
    
    for i, item in enumerate(work_items):
        result = yield ctx.call_activity("process_item", input=item)
        results.append(result)
        
        # Set custom status for progress tracking
        progress = (i + 1) / total * 100
        ctx.set_custom_status(f"Processing: {progress:.0f}% complete")
    
    ctx.set_custom_status("Completed")
    return {"results": results}
```

### Partial Failure Handling

```csharp
public override async Task<BatchResult> RunAsync(
    TaskOrchestrationContext context, 
    List<WorkItem> workItems)
{
    var tasks = workItems.Select(async item =>
    {
        try
        {
            return await context.CallActivityAsync<ItemResult>(
                nameof(ProcessItemActivity),
                item
            );
        }
        catch (Exception ex)
        {
            // Return failure result instead of throwing
            return new ItemResult
            {
                ItemId = item.Id,
                Success = false,
                Error = ex.Message
            };
        }
    });
    
    var results = await Task.WhenAll(tasks);
    
    return new BatchResult
    {
        Results = results.ToList(),
        SuccessCount = results.Count(r => r.Success),
        FailureCount = results.Count(r => !r.Success)
    };
}
```

### Dynamic Fan-Out

Determine work items dynamically:

```csharp
public override async Task<BatchResult> RunAsync(
    TaskOrchestrationContext context, 
    BatchRequest request)
{
    // Get work items dynamically
    var workItems = await context.CallActivityAsync<List<WorkItem>>(
        nameof(GetWorkItemsActivity),
        request.BatchId
    );
    
    // Fan-out
    var tasks = workItems.Select(item =>
        context.CallActivityAsync<ItemResult>(nameof(ProcessItemActivity), item)
    );
    
    // Fan-in
    var results = await Task.WhenAll(tasks);
    
    return new BatchResult { Results = results.ToList() };
}
```

---

## Performance Considerations

### Parallelism Limits

```
┌─────────────────────────────────────────────────────────────────┐
│                  PARALLELISM CONSIDERATIONS                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Scenario              Recommendation                            │
│  ─────────             ──────────────                            │
│  100 items             Full parallel (100 concurrent)            │
│  1,000 items           Chunk into batches of 50-100              │
│  10,000+ items         Chunk + use sub-orchestrations           │
│                                                                  │
│  Memory per item       Impact                                    │
│  ───────────────       ──────                                    │
│  < 1 KB                Minimal concern                           │
│  1-10 KB               Monitor memory usage                      │
│  > 10 KB               Consider streaming patterns               │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Optimal Batch Sizes

| Item Size | Recommended Batch | Notes |
|-----------|-------------------|-------|
| Small (< 1KB) | 100-500 | Network overhead dominates |
| Medium (1-10KB) | 50-100 | Balance parallelism and memory |
| Large (> 10KB) | 10-50 | Memory becomes limiting factor |

---

## Best Practices

1. **Set appropriate parallelism limits** - Don't overwhelm downstream services
2. **Handle partial failures gracefully** - Decide if one failure should fail all
3. **Use chunking for large batches** - Prevents memory issues
4. **Consider sub-orchestrations** - For very large workloads
5. **Monitor execution time** - Fan-out scales linearly with activity duration

---

## Related Patterns

- [Function Chaining](function-chaining.md) - Sequential alternative
- [Aggregator](aggregator.md) - For continuous aggregation over time
- [External Events](external-events.md) - For event-driven fan-in

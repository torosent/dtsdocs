---
layout: default
title: Isolated Worker API Reference
parent: .NET SDK
grand_parent: SDKs Overview
nav_order: 5
permalink: /docs/developer-guide/sdks/dotnet/api-reference-isolated/
---

# Isolated Worker SDK API Reference
{: .no_toc }

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

API reference for Microsoft.Azure.Functions.Worker.Extensions.DurableTask (.NET Isolated Worker).
{: .fs-6 .fw-300 }

---

## Package Information

| Package | NuGet |
|---------|-------|
| Microsoft.Azure.Functions.Worker.Extensions.DurableTask | [NuGet](https://www.nuget.org/packages/Microsoft.Azure.Functions.Worker.Extensions.DurableTask) |

```bash
dotnet add package Microsoft.Azure.Functions.Worker.Extensions.DurableTask
```

---

## Triggers

### OrchestrationTriggerAttribute

Defines an orchestrator function.

```csharp
[Function(nameof(MyOrchestrator))]
public async Task<string> MyOrchestrator(
    [OrchestrationTrigger] TaskOrchestrationContext context)
{
    // Orchestration logic
}
```

### ActivityTriggerAttribute

Defines an activity function.

```csharp
[Function(nameof(MyActivity))]
public string MyActivity(
    [ActivityTrigger] string input,
    FunctionContext executionContext)
{
    return $"Processed: {input}";
}
```

### EntityTriggerAttribute

Defines an entity function.

```csharp
[Function(nameof(Counter))]
public static Task Counter(
    [EntityTrigger] TaskEntityDispatcher dispatcher)
{
    return dispatcher.DispatchAsync<Counter>();
}
```

---

## Bindings

### DurableClientAttribute

Binds to a Durable Task client.

```csharp
[Function("HttpStart")]
public async Task<HttpResponseData> HttpStart(
    [HttpTrigger(AuthorizationLevel.Anonymous, "post")] HttpRequestData req,
    [DurableClient] DurableTaskClient client,
    FunctionContext executionContext)
{
    string instanceId = await client.ScheduleNewOrchestrationInstanceAsync(
        "MyOrchestrator");
    return await client.CreateCheckStatusResponseAsync(req, instanceId);
}
```

---

## TaskOrchestrationContext

Context available in orchestrator functions.

### Properties

| Property | Type | Description |
|----------|------|-------------|
| `InstanceId` | `string` | Unique orchestration instance ID |
| `Name` | `TaskName` | Orchestration name |
| `IsReplaying` | `bool` | Whether replaying history |
| `CurrentUtcDateTime` | `DateTime` | Deterministic current time |
| `ParentInstanceId` | `string?` | Parent orchestration ID |

### Activity Methods

#### CallActivityAsync

```csharp
// With return value
var result = await context.CallActivityAsync<string>(
    nameof(MyActivity), "input");

// Without return value
await context.CallActivityAsync(nameof(MyActivity), "input");

// With retry
var options = TaskOptions.FromRetryPolicy(new RetryPolicy(
    maxNumberOfAttempts: 3,
    firstRetryInterval: TimeSpan.FromSeconds(5)));

var result = await context.CallActivityAsync<string>(
    nameof(MyActivity), "input", options);
```

### Sub-orchestration Methods

#### CallSubOrchestratorAsync

```csharp
// Call sub-orchestration
var result = await context.CallSubOrchestratorAsync<OrderResult>(
    nameof(ProcessOrderOrchestrator),
    order);

// With specific instance ID
var result = await context.CallSubOrchestratorAsync<OrderResult>(
    nameof(ProcessOrderOrchestrator),
    order,
    new TaskOptions { InstanceId = $"sub-{context.InstanceId}" });
```

### Timer Methods

#### CreateTimer

```csharp
// Wait for duration
await context.CreateTimer(TimeSpan.FromMinutes(5), CancellationToken.None);

// Wait until specific time
await context.CreateTimer(
    context.CurrentUtcDateTime.AddHours(1),
    CancellationToken.None);
```

### External Event Methods

#### WaitForExternalEvent

```csharp
// Wait for event
var approval = await context.WaitForExternalEvent<bool>("ApprovalEvent");

// With timeout using pattern
var timeout = context.CreateTimer(TimeSpan.FromHours(24), CancellationToken.None);
var approval = context.WaitForExternalEvent<bool>("ApprovalEvent");

var winner = await Task.WhenAny(timeout, approval);
if (winner == approval)
{
    return await approval;
}
throw new TimeoutException();
```

### State Methods

#### GetInput

```csharp
var input = context.GetInput<OrderRequest>();
```

#### SetCustomStatus

```csharp
context.SetCustomStatus(new { Progress = 50, Stage = "Processing" });
```

#### ContinueAsNew

```csharp
context.ContinueAsNew(newInput);
```

### Logging Methods

#### CreateReplaySafeLogger

```csharp
var logger = context.CreateReplaySafeLogger(nameof(MyOrchestrator));
logger.LogInformation("Starting orchestration");
```

---

## DurableTaskClient

Client for managing orchestrations.

### Orchestration Management

#### ScheduleNewOrchestrationInstanceAsync

```csharp
// Start orchestration
string instanceId = await client.ScheduleNewOrchestrationInstanceAsync(
    nameof(MyOrchestrator),
    input);

// With custom instance ID
string instanceId = await client.ScheduleNewOrchestrationInstanceAsync(
    nameof(MyOrchestrator),
    input,
    new StartOrchestrationOptions { InstanceId = "custom-id" });
```

#### GetInstanceAsync

```csharp
var metadata = await client.GetInstanceAsync(instanceId, getInputsAndOutputs: true);
if (metadata?.RuntimeStatus == OrchestrationRuntimeStatus.Completed)
{
    var output = metadata.ReadOutputAs<string>();
}
```

#### WaitForInstanceCompletionAsync

```csharp
var metadata = await client.WaitForInstanceCompletionAsync(
    instanceId,
    getInputsAndOutputs: true);
```

#### RaiseEventAsync

```csharp
await client.RaiseEventAsync(instanceId, "ApprovalEvent", true);
```

#### TerminateInstanceAsync

```csharp
await client.TerminateInstanceAsync(instanceId, "User cancelled");
```

#### SuspendInstanceAsync

```csharp
await client.SuspendInstanceAsync(instanceId, "Maintenance");
```

#### ResumeInstanceAsync

```csharp
await client.ResumeInstanceAsync(instanceId, "Maintenance complete");
```

### HTTP Response Helpers

#### CreateCheckStatusResponseAsync

Creates an HTTP response with status query URLs.

```csharp
return await client.CreateCheckStatusResponseAsync(request, instanceId);
```

Response includes:
- `id` - Instance ID
- `statusQueryGetUri` - URL to check status
- `sendEventPostUri` - URL to send events
- `terminatePostUri` - URL to terminate
- `suspendPostUri` - URL to suspend
- `resumePostUri` - URL to resume

### Query Methods

#### GetAllInstancesAsync

```csharp
var query = new OrchestrationQuery
{
    RuntimeStatus = new[] { OrchestrationRuntimeStatus.Running },
    PageSize = 100
};

await foreach (var page in client.GetAllInstancesAsync(query))
{
    foreach (var instance in page.Instances)
    {
        Console.WriteLine($"{instance.InstanceId}: {instance.RuntimeStatus}");
    }
}
```

### Purge Methods

#### PurgeInstanceAsync

```csharp
await client.PurgeInstanceAsync(instanceId);
```

#### PurgeAllInstancesAsync

```csharp
var filter = new PurgeInstancesFilter
{
    CreatedFrom = DateTime.UtcNow.AddDays(-30),
    RuntimeStatus = new[] { OrchestrationRuntimeStatus.Completed }
};

var result = await client.PurgeAllInstancesAsync(filter);
```

---

## TaskEntityDispatcher

Dispatcher for entity functions.

### DispatchAsync (Class-based)

```csharp
[Function(nameof(Counter))]
public static Task Counter([EntityTrigger] TaskEntityDispatcher dispatcher)
{
    return dispatcher.DispatchAsync<Counter>();
}
```

### DispatchAsync (Function-based)

```csharp
[Function("MyEntity")]
public static Task MyEntity([EntityTrigger] TaskEntityDispatcher dispatcher)
{
    return dispatcher.DispatchAsync(operation =>
    {
        switch (operation.Name)
        {
            case "set":
                operation.State.SetState(operation.GetInput<string>());
                break;
            case "get":
                return operation.State.GetState(typeof(string));
            case "delete":
                operation.State.SetState(null);
                break;
        }
        return null;
    });
}
```

---

## Entity Client Operations

### SignalEntityAsync

```csharp
await client.Entities.SignalEntityAsync(
    new EntityInstanceId("Counter", "myCounter"),
    "Add",
    5);
```

### GetEntityAsync

```csharp
var state = await client.Entities.GetEntityAsync<int>(
    new EntityInstanceId("Counter", "myCounter"));
```

---

## Configuration

### host.json

```json
{
  "version": "2.0",
  "extensions": {
    "durableTask": {
      "hubName": "MyTaskHub",
      "storageProvider": {
        "type": "azurestorage",
        "connectionStringName": "AzureWebJobsStorage"
      },
      "maxConcurrentActivityFunctions": 10,
      "maxConcurrentOrchestratorFunctions": 5
    }
  }
}
```

---

## Common Patterns

### Error Handling

```csharp
[Function(nameof(MyOrchestrator))]
public async Task<string> MyOrchestrator(
    [OrchestrationTrigger] TaskOrchestrationContext context)
{
    try
    {
        return await context.CallActivityAsync<string>(
            nameof(MyActivity), "input");
    }
    catch (TaskFailedException ex)
    {
        // Activity failed
        var logger = context.CreateReplaySafeLogger(nameof(MyOrchestrator));
        logger.LogError(ex, "Activity failed: {Message}", ex.Message);
        
        // Handle or rethrow
        throw;
    }
}
```

### Fan-out/Fan-in

```csharp
[Function(nameof(FanOutFanIn))]
public async Task<int[]> FanOutFanIn(
    [OrchestrationTrigger] TaskOrchestrationContext context)
{
    var items = context.GetInput<string[]>();
    
    // Fan out
    var tasks = items.Select(item =>
        context.CallActivityAsync<int>(nameof(ProcessItem), item));
    
    // Fan in
    var results = await Task.WhenAll(tasks);
    return results;
}
```

### Human Interaction

```csharp
[Function(nameof(ApprovalWorkflow))]
public async Task<string> ApprovalWorkflow(
    [OrchestrationTrigger] TaskOrchestrationContext context)
{
    // Send approval request
    await context.CallActivityAsync(nameof(SendApprovalRequest), context.InstanceId);
    
    // Wait for approval with timeout
    using var cts = new CancellationTokenSource();
    var timeout = context.CreateTimer(TimeSpan.FromDays(3), cts.Token);
    var approval = context.WaitForExternalEvent<ApprovalResult>("Approval");
    
    var winner = await Task.WhenAny(timeout, approval);
    
    if (winner == approval)
    {
        cts.Cancel();
        var result = await approval;
        return result.Approved ? "Approved" : "Rejected";
    }
    
    return "Timeout - Escalated";
}
```

### Eternal Orchestrations

```csharp
[Function(nameof(PeriodicTask))]
public async Task PeriodicTask(
    [OrchestrationTrigger] TaskOrchestrationContext context)
{
    var state = context.GetInput<PeriodicState>() ?? new PeriodicState();
    
    // Do work
    await context.CallActivityAsync(nameof(DoPeriodicWork), state);
    
    // Wait for next interval
    await context.CreateTimer(TimeSpan.FromHours(1), CancellationToken.None);
    
    // Continue as new to prevent history growth
    state.IterationCount++;
    context.ContinueAsNew(state);
}
```

---
layout: default
title: Portable API Reference
parent: .NET SDK
grand_parent: SDKs Overview
nav_order: 4
permalink: /docs/developer-guide/sdks/dotnet/api-reference-portable/
---

# Portable SDK API Reference
{: .no_toc }

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

API reference for Microsoft.DurableTask portable SDK.
{: .fs-6 .fw-300 }

---

## Package Overview

| Package | Purpose |
|---------|---------|
| `Microsoft.DurableTask.Worker` | Worker registration and hosting |
| `Microsoft.DurableTask.Client` | Client for managing orchestrations |
| `Microsoft.DurableTask.Worker.AzureManaged` | Azure Managed integration for workers |
| `Microsoft.DurableTask.Client.AzureManaged` | Azure Managed integration for clients |

---

## Installation

```bash
dotnet add package Microsoft.DurableTask.Worker.AzureManaged
dotnet add package Microsoft.DurableTask.Client.AzureManaged
```

---

## Worker APIs

### AddDurableTaskWorker

Registers the Durable Task worker with dependency injection.

```csharp
services.AddDurableTaskWorker()
    .AddTasks(registry =>
    {
        registry.AddOrchestrator<MyOrchestration>();
        registry.AddActivity<MyActivity>();
    })
    .UseDurableTaskScheduler(connectionString);
```

### TaskRegistry

| Method | Description |
|--------|-------------|
| `AddOrchestrator<T>()` | Register an orchestrator |
| `AddActivity<T>()` | Register an activity |
| `AddAllFrom(assembly)` | Register all tasks from an assembly |

---

## TaskOrchestrator<TInput, TOutput>

Base class for defining orchestrators.

### Properties

| Property | Type | Description |
|----------|------|-------------|
| `Name` | `string` | Orchestrator name (defaults to class name) |

### Methods

| Method | Returns | Description |
|--------|---------|-------------|
| `RunAsync(context, input)` | `Task<TOutput>` | Main orchestration logic |

### Example

```csharp
[DurableTask]
public class ProcessOrderOrchestration : TaskOrchestrator<Order, OrderResult>
{
    public override async Task<OrderResult> RunAsync(
        TaskOrchestrationContext context,
        Order input)
    {
        // Orchestration logic
        var validated = await context.CallActivityAsync<bool>(
            nameof(ValidateOrderActivity), input);
        
        if (!validated)
        {
            return new OrderResult { Success = false };
        }
        
        await context.CallActivityAsync(nameof(ProcessPaymentActivity), input);
        await context.CallActivityAsync(nameof(ShipOrderActivity), input);
        
        return new OrderResult { Success = true, OrderId = input.Id };
    }
}
```

---

## TaskActivity<TInput, TOutput>

Base class for defining activities.

### Properties

| Property | Type | Description |
|----------|------|-------------|
| `Name` | `string` | Activity name (defaults to class name) |

### Methods

| Method | Returns | Description |
|--------|---------|-------------|
| `RunAsync(context, input)` | `Task<TOutput>` | Activity execution logic |

### Example

```csharp
[DurableTask]
public class ValidateOrderActivity : TaskActivity<Order, bool>
{
    private readonly IOrderValidator _validator;
    
    public ValidateOrderActivity(IOrderValidator validator)
    {
        _validator = validator;
    }
    
    public override Task<bool> RunAsync(
        TaskActivityContext context,
        Order input)
    {
        return Task.FromResult(_validator.Validate(input));
    }
}
```

---

## TaskOrchestrationContext

Context object available in orchestrators.

### Properties

| Property | Type | Description |
|----------|------|-------------|
| `InstanceId` | `string` | Unique orchestration instance ID |
| `Name` | `TaskName` | Orchestration name |
| `IsReplaying` | `bool` | Whether currently replaying history |
| `CurrentUtcDateTime` | `DateTime` | Deterministic current time |
| `ParentInstanceId` | `string?` | Parent orchestration ID (for sub-orchestrations) |

### Methods

#### CallActivityAsync

Call an activity function.

```csharp
// With return value
var result = await context.CallActivityAsync<string>(
    activityName: "MyActivity",
    input: "input data");

// Without return value
await context.CallActivityAsync(
    activityName: "MyActivity",
    input: "input data");

// With options
var result = await context.CallActivityAsync<string>(
    activityName: "MyActivity",
    input: "input data",
    options: new TaskOptions
    {
        Retry = new RetryPolicy(
            maxNumberOfAttempts: 3,
            firstRetryInterval: TimeSpan.FromSeconds(5))
    });
```

#### CallSubOrchestratorAsync

Call a sub-orchestration.

```csharp
var result = await context.CallSubOrchestratorAsync<OrderResult>(
    orchestratorName: "ProcessOrderOrchestration",
    input: order,
    options: new SubOrchestrationOptions
    {
        InstanceId = $"suborch-{context.InstanceId}-{Guid.NewGuid()}"
    });
```

#### CreateTimer

Create a durable timer.

```csharp
// Wait for a duration
await context.CreateTimer(TimeSpan.FromHours(1), CancellationToken.None);

// Wait until a specific time
await context.CreateTimer(
    context.CurrentUtcDateTime.AddDays(1), 
    CancellationToken.None);
```

#### WaitForExternalEvent

Wait for an external event.

```csharp
// Wait indefinitely
var approval = await context.WaitForExternalEvent<bool>("ApprovalEvent");

// Wait with timeout
using var cts = new CancellationTokenSource();
var timerTask = context.CreateTimer(TimeSpan.FromHours(24), cts.Token);
var eventTask = context.WaitForExternalEvent<bool>("ApprovalEvent");

var winner = await Task.WhenAny(timerTask, eventTask);
if (winner == eventTask)
{
    cts.Cancel();
    return await eventTask;
}
else
{
    throw new TimeoutException("Approval timeout");
}
```

#### ContinueAsNew

Restart the orchestration with new input.

```csharp
context.ContinueAsNew(newInput, preserveUnprocessedEvents: true);
```

#### CreateReplaySafeLogger

Create a logger that only logs during non-replay executions.

```csharp
var logger = context.CreateReplaySafeLogger(nameof(MyOrchestration));
logger.LogInformation("This only logs once, not during replay");
```

#### GetInput

Get the orchestration input.

```csharp
var input = context.GetInput<Order>();
```

#### SetCustomStatus

Set custom status visible in management APIs.

```csharp
context.SetCustomStatus(new { step = "Processing", progress = 50 });
```

---

## DurableTaskClient

Client for managing orchestrations.

### Construction

```csharp
// Via dependency injection (recommended)
services.AddDurableTaskClient()
    .UseDurableTaskScheduler(connectionString);

// Manual construction
var client = new DurableTaskGrpcClient(connectionString);
```

### Methods

#### ScheduleNewOrchestrationInstanceAsync

Start a new orchestration.

```csharp
// Simple start
string instanceId = await client.ScheduleNewOrchestrationInstanceAsync(
    orchestratorName: "MyOrchestration",
    input: myInput);

// With specific instance ID
string instanceId = await client.ScheduleNewOrchestrationInstanceAsync(
    orchestratorName: "MyOrchestration",
    input: myInput,
    options: new StartOrchestrationOptions
    {
        InstanceId = "my-custom-id"
    });

// Scheduled start
string instanceId = await client.ScheduleNewOrchestrationInstanceAsync(
    orchestratorName: "MyOrchestration",
    input: myInput,
    options: new StartOrchestrationOptions
    {
        StartAt = DateTimeOffset.UtcNow.AddHours(1)
    });
```

#### GetInstanceAsync

Get orchestration metadata.

```csharp
var metadata = await client.GetInstanceAsync(
    instanceId: "my-instance-id",
    getInputsAndOutputs: true);

if (metadata != null)
{
    Console.WriteLine($"Status: {metadata.RuntimeStatus}");
    Console.WriteLine($"Created: {metadata.CreatedAt}");
    Console.WriteLine($"Completed: {metadata.LastUpdatedAt}");
    
    if (metadata.IsCompleted)
    {
        var output = metadata.ReadOutputAs<OrderResult>();
    }
}
```

#### WaitForInstanceCompletionAsync

Wait for an orchestration to complete.

```csharp
var metadata = await client.WaitForInstanceCompletionAsync(
    instanceId: "my-instance-id",
    getInputsAndOutputs: true,
    cancellation: CancellationToken.None);

var result = metadata.ReadOutputAs<OrderResult>();
```

#### WaitForInstanceStartAsync

Wait for an orchestration to start.

```csharp
var metadata = await client.WaitForInstanceStartAsync(
    instanceId: "my-instance-id",
    cancellation: CancellationToken.None);
```

#### RaiseEventAsync

Raise an event to an orchestration.

```csharp
await client.RaiseEventAsync(
    instanceId: "my-instance-id",
    eventName: "ApprovalEvent",
    eventPayload: true);
```

#### TerminateInstanceAsync

Terminate a running orchestration.

```csharp
await client.TerminateInstanceAsync(
    instanceId: "my-instance-id",
    reason: "Cancelled by user");
```

#### SuspendInstanceAsync

Suspend an orchestration.

```csharp
await client.SuspendInstanceAsync(
    instanceId: "my-instance-id",
    reason: "Paused for maintenance");
```

#### ResumeInstanceAsync

Resume a suspended orchestration.

```csharp
await client.ResumeInstanceAsync(
    instanceId: "my-instance-id",
    reason: "Maintenance complete");
```

#### PurgeInstanceAsync

Purge orchestration history.

```csharp
await client.PurgeInstanceAsync(instanceId: "my-instance-id");
```

#### GetAllInstancesAsync

Query orchestration instances.

```csharp
var query = new OrchestrationQuery
{
    RuntimeStatus = new[]
    {
        OrchestrationRuntimeStatus.Running,
        OrchestrationRuntimeStatus.Pending
    },
    CreatedFrom = DateTime.UtcNow.AddDays(-7),
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

#### PurgeAllInstancesAsync

Bulk purge orchestrations.

```csharp
var filter = new PurgeInstancesFilter
{
    CreatedFrom = DateTime.UtcNow.AddDays(-30),
    RuntimeStatus = new[] { OrchestrationRuntimeStatus.Completed }
};

var result = await client.PurgeAllInstancesAsync(filter);
Console.WriteLine($"Purged {result.PurgedInstanceCount} instances");
```

---

## OrchestrationMetadata

Information about an orchestration instance.

### Properties

| Property | Type | Description |
|----------|------|-------------|
| `InstanceId` | `string` | Instance ID |
| `Name` | `string` | Orchestration name |
| `RuntimeStatus` | `OrchestrationRuntimeStatus` | Current status |
| `CreatedAt` | `DateTimeOffset` | Creation time |
| `LastUpdatedAt` | `DateTimeOffset` | Last update time |
| `SerializedInput` | `string` | JSON input |
| `SerializedOutput` | `string` | JSON output |
| `SerializedCustomStatus` | `string` | JSON custom status |
| `FailureDetails` | `TaskFailureDetails?` | Failure information |
| `IsRunning` | `bool` | Whether currently running |
| `IsCompleted` | `bool` | Whether completed (success or failure) |

### Methods

| Method | Returns | Description |
|--------|---------|-------------|
| `ReadInputAs<T>()` | `T` | Deserialize input |
| `ReadOutputAs<T>()` | `T` | Deserialize output |
| `ReadCustomStatusAs<T>()` | `T` | Deserialize custom status |

---

## OrchestrationRuntimeStatus

| Value | Description |
|-------|-------------|
| `Pending` | Waiting to start |
| `Running` | Currently executing |
| `Completed` | Finished successfully |
| `Failed` | Failed with error |
| `Terminated` | Manually terminated |
| `Canceled` | Canceled |
| `Suspended` | Suspended |

---

## TaskOptions

Options for activity and sub-orchestration calls.

### Properties

| Property | Type | Description |
|----------|------|-------------|
| `Retry` | `RetryPolicy?` | Retry configuration |

### RetryPolicy Properties

| Property | Type | Description |
|----------|------|-------------|
| `MaxNumberOfAttempts` | `int` | Maximum retry attempts |
| `FirstRetryInterval` | `TimeSpan` | Initial retry delay |
| `BackoffCoefficient` | `double` | Backoff multiplier |
| `MaxRetryInterval` | `TimeSpan?` | Maximum delay between retries |
| `RetryTimeout` | `TimeSpan?` | Total retry timeout |

### Example

```csharp
var options = new TaskOptions
{
    Retry = new RetryPolicy(
        maxNumberOfAttempts: 5,
        firstRetryInterval: TimeSpan.FromSeconds(1))
    {
        BackoffCoefficient = 2.0,
        MaxRetryInterval = TimeSpan.FromMinutes(1),
        RetryTimeout = TimeSpan.FromMinutes(10)
    }
};

await context.CallActivityAsync<string>("MyActivity", input, options);
```

---

## Attributes

### DurableTaskAttribute

Mark classes for source generation.

```csharp
[DurableTask]
public class MyOrchestration : TaskOrchestrator<string, string>
{
    // ...
}

[DurableTask]
public class MyActivity : TaskActivity<string, string>
{
    // ...
}
```

---

## Entity APIs

### TaskEntity<TState>

Base class for entities.

```csharp
[DurableTask]
public class Counter : TaskEntity<CounterState>
{
    protected override CounterState InitializeState() => new() { Value = 0 };

    public void Add(int amount) => State.Value += amount;
    public void Reset() => State.Value = 0;
    public int Get() => State.Value;
}

public class CounterState
{
    public int Value { get; set; }
}
```

### Entity Operations (Client)

```csharp
// Signal entity (fire-and-forget)
await client.Entities.SignalEntityAsync(
    new EntityInstanceId("Counter", "myCounter"),
    "Add",
    5);

// Get entity state
var state = await client.Entities.GetEntityAsync<CounterState>(
    new EntityInstanceId("Counter", "myCounter"));
```

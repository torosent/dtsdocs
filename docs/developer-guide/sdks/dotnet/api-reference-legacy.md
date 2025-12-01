---
layout: default
title: Legacy In-Process API Reference
parent: .NET SDK
grand_parent: SDKs Overview
nav_order: 6
permalink: /docs/developer-guide/sdks/dotnet/api-reference-legacy/
---

# In-Process SDK API Reference (Legacy)
{: .no_toc }

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

API reference for Microsoft.Azure.WebJobs.Extensions.DurableTask (In-Process model).
{: .fs-6 .fw-300 }

> **Warning:** The in-process model is being deprecated. New projects should use the [isolated worker model](./api-reference-isolated.md) or [portable SDK](./api-reference-portable.md).

---

## Package Information

```bash
dotnet add package Microsoft.Azure.WebJobs.Extensions.DurableTask
```

---

## Triggers

### OrchestrationTrigger

```csharp
[FunctionName("MyOrchestrator")]
public static async Task<string> RunOrchestrator(
    [OrchestrationTrigger] IDurableOrchestrationContext context)
{
    // Orchestration logic
}
```

### ActivityTrigger

```csharp
[FunctionName("MyActivity")]
public static string RunActivity(
    [ActivityTrigger] string input,
    ILogger log)
{
    return $"Processed: {input}";
}

// With IDurableActivityContext
[FunctionName("MyActivity")]
public static string RunActivity(
    [ActivityTrigger] IDurableActivityContext context,
    ILogger log)
{
    var input = context.GetInput<string>();
    return $"Processed: {input}";
}
```

### EntityTrigger

```csharp
[FunctionName("Counter")]
public static Task Counter(
    [EntityTrigger] IDurableEntityContext ctx)
{
    return ctx.DispatchAsync<Counter>();
}
```

---

## Bindings

### DurableClient

```csharp
[FunctionName("HttpStart")]
public static async Task<HttpResponseMessage> HttpStart(
    [HttpTrigger(AuthorizationLevel.Function, "post")] HttpRequestMessage req,
    [DurableClient] IDurableOrchestrationClient starter,
    ILogger log)
{
    string instanceId = await starter.StartNewAsync("MyOrchestrator", null);
    return starter.CreateCheckStatusResponse(req, instanceId);
}
```

---

## IDurableOrchestrationContext

Primary interface for orchestrator functions.

### Properties

| Property | Type | Description |
|----------|------|-------------|
| `InstanceId` | `string` | Unique instance ID |
| `ParentInstanceId` | `string` | Parent orchestration ID |
| `IsReplaying` | `bool` | Whether replaying history |
| `CurrentUtcDateTime` | `DateTime` | Deterministic current time |

### Activity Methods

#### CallActivityAsync

```csharp
// With return value
var result = await context.CallActivityAsync<string>("ActivityName", input);

// Without return value
await context.CallActivityAsync("ActivityName", input);
```

#### CallActivityWithRetryAsync

```csharp
var retryOptions = new RetryOptions(
    firstRetryInterval: TimeSpan.FromSeconds(5),
    maxNumberOfAttempts: 3)
{
    BackoffCoefficient = 2.0,
    MaxRetryInterval = TimeSpan.FromMinutes(1),
    RetryTimeout = TimeSpan.FromMinutes(10)
};

var result = await context.CallActivityWithRetryAsync<string>(
    "ActivityName",
    retryOptions,
    input);
```

### Sub-orchestration Methods

#### CallSubOrchestratorAsync

```csharp
var result = await context.CallSubOrchestratorAsync<OrderResult>(
    "SubOrchestrator",
    input);

// With specific instance ID
var result = await context.CallSubOrchestratorAsync<OrderResult>(
    "SubOrchestrator",
    instanceId: "sub-123",
    input);
```

#### CallSubOrchestratorWithRetryAsync

```csharp
var result = await context.CallSubOrchestratorWithRetryAsync<OrderResult>(
    "SubOrchestrator",
    retryOptions,
    input);
```

### Timer Methods

#### CreateTimer

```csharp
// Wait for duration
await context.CreateTimer(
    context.CurrentUtcDateTime.AddMinutes(5),
    CancellationToken.None);

// Cancelable timer
using var cts = new CancellationTokenSource();
try
{
    await context.CreateTimer(
        context.CurrentUtcDateTime.AddHours(1),
        cts.Token);
}
catch (TaskCanceledException)
{
    // Timer was cancelled
}
```

### External Event Methods

#### WaitForExternalEvent

```csharp
// Wait indefinitely
var approval = await context.WaitForExternalEvent<bool>("ApprovalEvent");

// Wait with timeout
var approval = await context.WaitForExternalEvent<bool>(
    "ApprovalEvent",
    timeout: TimeSpan.FromHours(24),
    defaultValue: false);
```

### HTTP Methods

#### CallHttpAsync

```csharp
var response = await context.CallHttpAsync(
    HttpMethod.Get,
    new Uri("https://api.example.com/data"));

// With managed identity
var request = new DurableHttpRequest(
    HttpMethod.Get,
    new Uri("https://api.example.com/data"),
    tokenSource: new ManagedIdentityTokenSource("https://api.example.com"));

var response = await context.CallHttpAsync(request);
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
context.ContinueAsNew(newInput, preserveUnprocessedEvents: true);
```

### Entity Methods

#### SignalEntity

```csharp
var entityId = new EntityId("Counter", "myCounter");
context.SignalEntity(entityId, "Add", 5);
```

#### CallEntityAsync

```csharp
var entityId = new EntityId("Counter", "myCounter");
var value = await context.CallEntityAsync<int>(entityId, "Get");
```

#### CreateEntityProxy

```csharp
var entityId = new EntityId("Counter", "myCounter");
var proxy = context.CreateEntityProxy<ICounter>(entityId);
proxy.Add(5);
var value = await proxy.Get();
```

---

## IDurableOrchestrationClient

Client interface for managing orchestrations.

### Start Methods

#### StartNewAsync

```csharp
// Simple start
string instanceId = await client.StartNewAsync("OrchestratorName", input);

// With specific instance ID
string instanceId = await client.StartNewAsync(
    orchestratorFunctionName: "OrchestratorName",
    instanceId: "my-custom-id",
    input: input);
```

### Query Methods

#### GetStatusAsync

```csharp
var status = await client.GetStatusAsync(instanceId);
var status = await client.GetStatusAsync(
    instanceId,
    showHistory: true,
    showHistoryOutput: true,
    showInput: true);
```

#### ListInstancesAsync

```csharp
var condition = new OrchestrationStatusQueryCondition
{
    RuntimeStatus = new[]
    {
        OrchestrationRuntimeStatus.Running,
        OrchestrationRuntimeStatus.Pending
    },
    CreatedTimeFrom = DateTime.UtcNow.AddDays(-7),
    CreatedTimeTo = DateTime.UtcNow,
    PageSize = 100
};

var result = await client.ListInstancesAsync(condition, CancellationToken.None);

do
{
    foreach (var instance in result.DurableOrchestrationState)
    {
        Console.WriteLine($"{instance.InstanceId}: {instance.RuntimeStatus}");
    }
    
    if (!string.IsNullOrEmpty(result.ContinuationToken))
    {
        condition.ContinuationToken = result.ContinuationToken;
        result = await client.ListInstancesAsync(condition, CancellationToken.None);
    }
    else
    {
        break;
    }
} while (true);
```

### Event Methods

#### RaiseEventAsync

```csharp
await client.RaiseEventAsync(instanceId, "ApprovalEvent", true);
```

### Lifecycle Methods

#### TerminateAsync

```csharp
await client.TerminateAsync(instanceId, "User requested termination");
```

#### RestartAsync

```csharp
await client.RestartAsync(instanceId, restartWithNewInstanceId: false);
```

#### RewindAsync

```csharp
await client.RewindAsync(instanceId, "Rewind to fix issue");
```

### Suspend/Resume Methods

#### SuspendAsync

```csharp
await client.SuspendAsync(instanceId, "Maintenance");
```

#### ResumeAsync

```csharp
await client.ResumeAsync(instanceId, "Maintenance complete");
```

### Purge Methods

#### PurgeInstanceHistoryAsync

```csharp
// Single instance
await client.PurgeInstanceHistoryAsync(instanceId);

// By date range
var result = await client.PurgeInstanceHistoryAsync(
    createdTimeFrom: DateTime.UtcNow.AddDays(-30),
    createdTimeTo: DateTime.UtcNow.AddDays(-7),
    runtimeStatus: new[]
    {
        OrchestrationRuntimeStatus.Completed,
        OrchestrationRuntimeStatus.Failed
    });

Console.WriteLine($"Purged {result.InstancesDeleted} instances");
```

### HTTP Response Methods

#### CreateCheckStatusResponse

```csharp
return client.CreateCheckStatusResponse(request, instanceId);
return client.CreateCheckStatusResponse(
    request,
    instanceId,
    returnInternalServerErrorOnFailure: true);
```

#### CreateHttpManagementPayload

```csharp
var payload = client.CreateHttpManagementPayload(instanceId);
// Returns URLs for status, send event, terminate, etc.
```

#### WaitForCompletionOrCreateCheckStatusResponseAsync

```csharp
// Wait up to 10 seconds for completion, otherwise return check status
return await client.WaitForCompletionOrCreateCheckStatusResponseAsync(
    request,
    instanceId,
    timeout: TimeSpan.FromSeconds(10),
    retryInterval: TimeSpan.FromSeconds(1));
```

---

## IDurableEntityContext

Context for entity functions.

### Properties

| Property | Type | Description |
|----------|------|-------------|
| `EntityName` | `string` | Entity type name |
| `EntityKey` | `string` | Entity instance key |
| `EntityId` | `EntityId` | Full entity ID |
| `OperationName` | `string` | Current operation |
| `HasState` | `bool` | Whether state exists |

### State Methods

#### GetState

```csharp
var value = ctx.GetState<int>();
```

#### SetState

```csharp
ctx.SetState(newValue);
```

#### DeleteState

```csharp
ctx.DeleteState();
```

### Input/Output Methods

#### GetInput

```csharp
var input = ctx.GetInput<int>();
```

#### Return

```csharp
ctx.Return(result);
```

### Dispatch Methods

#### DispatchAsync

```csharp
await ctx.DispatchAsync<MyEntityClass>();
```

### Communication Methods

#### SignalEntity

```csharp
ctx.SignalEntity(new EntityId("OtherEntity", "key"), "Operation", input);
```

#### StartNewOrchestration

```csharp
ctx.StartNewOrchestration("OrchestratorName", input);
```

---

## IDurableEntityClient

Client for entity operations.

### SignalEntityAsync

```csharp
var entityId = new EntityId("Counter", "myCounter");
await client.SignalEntityAsync(entityId, "Add", 5);

// Type-safe with interface
await client.SignalEntityAsync<ICounter>(entityId, proxy => proxy.Add(5));
```

### ReadEntityStateAsync

```csharp
var entityId = new EntityId("Counter", "myCounter");
var state = await client.ReadEntityStateAsync<Counter>(entityId);

if (state.EntityExists)
{
    Console.WriteLine($"Value: {state.EntityState.CurrentValue}");
}
```

### ListEntitiesAsync

```csharp
var query = new EntityQuery
{
    EntityName = "Counter",
    FetchState = true,
    PageSize = 100
};

var result = await client.ListEntitiesAsync(query, CancellationToken.None);

foreach (var entity in result.Entities)
{
    var counter = entity.State?.ToObject<Counter>();
    Console.WriteLine($"{entity.EntityId}: {counter?.CurrentValue}");
}
```

### CleanEntityStorageAsync

```csharp
var result = await client.CleanEntityStorageAsync(
    removeEmptyEntities: true,
    releaseOrphanedLocks: true,
    CancellationToken.None);

Console.WriteLine($"Removed {result.NumberOfEmptyEntitiesRemoved} empty entities");
Console.WriteLine($"Released {result.NumberOfOrphanedLocksRemoved} orphaned locks");
```

---

## DurableOrchestrationStatus

Status information for an orchestration.

### Properties

| Property | Type | Description |
|----------|------|-------------|
| `InstanceId` | `string` | Instance ID |
| `Name` | `string` | Orchestrator name |
| `RuntimeStatus` | `OrchestrationRuntimeStatus` | Current status |
| `CreatedTime` | `DateTime` | Creation time |
| `LastUpdatedTime` | `DateTime` | Last update time |
| `Input` | `JToken` | Orchestration input |
| `Output` | `JToken` | Orchestration output |
| `CustomStatus` | `JToken` | Custom status |
| `History` | `JArray` | Execution history |

---

## RetryOptions

Configuration for retry behavior.

### Properties

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `FirstRetryInterval` | `TimeSpan` | Required | Initial retry delay |
| `MaxNumberOfAttempts` | `int` | Required | Maximum attempts |
| `BackoffCoefficient` | `double` | 1.0 | Exponential backoff multiplier |
| `MaxRetryInterval` | `TimeSpan?` | null | Maximum delay |
| `RetryTimeout` | `TimeSpan?` | null | Total timeout |
| `Handle` | `Func<Exception, bool>` | null | Exception filter |

### Example

```csharp
var retryOptions = new RetryOptions(
    firstRetryInterval: TimeSpan.FromSeconds(1),
    maxNumberOfAttempts: 5)
{
    BackoffCoefficient = 2.0,
    MaxRetryInterval = TimeSpan.FromMinutes(1),
    RetryTimeout = TimeSpan.FromMinutes(10),
    Handle = ex => ex is HttpRequestException
};
```

---

## OrchestrationRuntimeStatus

| Value | Description |
|-------|-------------|
| `Running` | Currently executing |
| `Completed` | Finished successfully |
| `ContinuedAsNew` | Restarted with new input |
| `Failed` | Failed with error |
| `Canceled` | Canceled |
| `Terminated` | Manually terminated |
| `Pending` | Waiting to start |
| `Suspended` | Suspended |

---

## Migration Notes

When migrating from in-process to isolated worker:

| In-Process | Isolated Worker |
|------------|-----------------|
| `[FunctionName("X")]` | `[Function("X")]` |
| `IDurableOrchestrationContext` | `TaskOrchestrationContext` |
| `IDurableOrchestrationClient` | `DurableTaskClient` |
| `IDurableEntityContext` | `TaskEntityDispatcher` |
| `IDurableEntityClient` | `DurableTaskClient.Entities` |
| `context.CallActivityAsync` | `context.CallActivityAsync` |
| `client.StartNewAsync` | `client.ScheduleNewOrchestrationInstanceAsync` |
| `client.CreateCheckStatusResponse` | `client.CreateCheckStatusResponseAsync` |

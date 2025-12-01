---
layout: default
title: Instance Management
parent: Core Concepts
nav_order: 8
permalink: /docs/concepts/instance-management/
---

# Instance Management
{: .no_toc }

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

Managing orchestration instances is a key part of operating a durable application. You need to be able to start, query, terminate, and purge instances.

---

## Getting the Client

The way you obtain a client differs between Azure Functions and standalone SDK applications.

### Azure Durable Functions

In Azure Functions, inject the client using the `[DurableClient]` attribute:

```csharp
// Azure Functions (Isolated Worker)
[Function("StartOrchestration")]
public static async Task<HttpResponseData> HttpStart(
    [HttpTrigger(AuthorizationLevel.Anonymous, "post")] HttpRequestData req,
    [DurableClient] DurableTaskClient client)
{
    string instanceId = await client.ScheduleNewOrchestrationInstanceAsync(
        "MyOrchestrator", 
        input
    );
    
    return await client.CreateCheckStatusResponseAsync(req, instanceId);
}
```

### Durable Task SDKs

In standalone applications, obtain the client from dependency injection:

```csharp
// Durable Task SDKs (Console app, ASP.NET Core, etc.)
var builder = Host.CreateApplicationBuilder(args);

builder.Services.AddDurableTaskClient()
    .UseDurableTaskScheduler(connectionString, taskHub);

var host = builder.Build();
var client = host.Services.GetRequiredService<DurableTaskClient>();

string instanceId = await client.ScheduleNewOrchestrationInstanceAsync(
    "MyOrchestrator", 
    input
);
```

> **Note**: The remainder of this guide uses SDK-agnostic code. The client APIs are the same regardless of how you obtain the client.

---

## Starting Instances

You can start a new orchestration instance using the client. You must provide the orchestrator name and can optionally provide an instance ID and input.

```csharp
string instanceId = await client.ScheduleNewOrchestrationInstanceAsync("MyOrchestrator", input);
```

If you don't provide an instance ID, one will be auto-generated (GUID).

### With Custom Instance ID

```csharp
string instanceId = await client.ScheduleNewOrchestrationInstanceAsync(
    "MyOrchestrator",
    new StartOrchestrationOptions
    {
        InstanceId = "order-12345"
    },
    input
);
```

### Python

```python
instance_id = await client.schedule_new_orchestration(
    "my_orchestrator",
    input=my_input,
    instance_id="order-12345"  # Optional
)
```

### Java

```java
String instanceId = client.scheduleNewOrchestrationInstance(
    "MyOrchestrator",
    new NewOrchestrationInstanceOptions()
        .setInstanceId("order-12345")
        .setInput(input)
);
```

---

## Querying Instances

You can query the status of an orchestration instance using its ID.

### Get Status

```csharp
var metadata = await client.GetInstanceAsync(instanceId, getInputsAndOutputs: true);

if (metadata != null)
{
    Console.WriteLine($"Status: {metadata.RuntimeStatus}");
    Console.WriteLine($"Created: {metadata.CreatedAt}");
    Console.WriteLine($"Last Updated: {metadata.LastUpdatedAt}");
    
    if (metadata.RuntimeStatus == OrchestrationRuntimeStatus.Completed)
    {
        var output = metadata.ReadOutputAs<MyOutput>();
        Console.WriteLine($"Output: {output}");
    }
}
```

### Python

```python
metadata = await client.get_instance(instance_id, fetch_inputs_and_outputs=True)

if metadata:
    print(f"Status: {metadata.runtime_status}")
    print(f"Output: {metadata.serialized_output}")
```

### Query with Filters

You can also query for multiple instances that match specific criteria (e.g., all failed instances created in the last hour).

```csharp
var query = new OrchestrationQuery
{
    CreatedFrom = DateTimeOffset.UtcNow.AddHours(-1),
    RuntimeStatus = new[] { OrchestrationRuntimeStatus.Failed }
};

await foreach (var instance in client.GetAllInstancesAsync(query))
{
    Console.WriteLine($"Failed: {instance.InstanceId} at {instance.LastUpdatedAt}");
}
```

---

## Terminating Instances

If an orchestration is stuck or running too long, you can forcibly terminate it.

```csharp
await client.TerminateInstanceAsync(instanceId, "Manually terminated by operator");
```

Terminating an instance stops it immediately. It transitions to the `Terminated` state.

### Python

```python
await client.terminate_instance(instance_id, reason="Manually terminated")
```

### Java

```java
client.terminate(instanceId, "Manually terminated");
```

---

## Suspending and Resuming

You can suspend a running orchestration to pause it temporarily without terminating it.

```csharp
// Suspend
await client.SuspendInstanceAsync(instanceId, "Paused for maintenance");

// Resume
await client.ResumeInstanceAsync(instanceId, "Maintenance complete");
```

---

## Purging History

Orchestration history takes up storage space. You can purge the history of completed, failed, or terminated instances to free up space and comply with data retention policies.

### Purge Single Instance

```csharp
await client.PurgeInstanceAsync(instanceId);
```

### Purge by Criteria

```csharp
var filter = new PurgeInstancesFilter
{
    CreatedFrom = DateTimeOffset.MinValue,
    CreatedTo = DateTimeOffset.UtcNow.AddDays(-30),
    RuntimeStatus = new[] { OrchestrationRuntimeStatus.Completed }
};

var result = await client.PurgeAllInstancesAsync(filter);
Console.WriteLine($"Purged {result.PurgedInstanceCount} instances");
```

---

## Sending Events

You can send external events to running orchestrations to trigger logic or unblock them.

```csharp
await client.RaiseEventAsync(instanceId, "ApprovalEvent", new ApprovalData { Approved = true });
```

### Python

```python
await client.raise_event(instance_id, "ApprovalEvent", {"approved": True})
```

### Waiting for Events in Orchestrator

```csharp
// In orchestrator
var approval = await context.WaitForExternalEvent<ApprovalData>("ApprovalEvent");

if (approval.Approved)
{
    await context.CallActivityAsync("ProcessApproval", data);
}
```

---

## HTTP APIs (Durable Functions)

If you are using Durable Functions, many of these operations are also exposed via built-in HTTP APIs.

| Operation | HTTP Method | Endpoint |
|-----------|-------------|----------|
| Start | POST | `/runtime/webhooks/durabletask/orchestrators/{functionName}` |
| Status | GET | `/runtime/webhooks/durabletask/instances/{instanceId}` |
| Terminate | POST | `/runtime/webhooks/durabletask/instances/{instanceId}/terminate` |
| Suspend | POST | `/runtime/webhooks/durabletask/instances/{instanceId}/suspend` |
| Resume | POST | `/runtime/webhooks/durabletask/instances/{instanceId}/resume` |
| Raise Event | POST | `/runtime/webhooks/durabletask/instances/{instanceId}/raiseEvent/{eventName}` |
| Purge | DELETE | `/runtime/webhooks/durabletask/instances/{instanceId}` |

The exact URLs depend on your function app configuration.

### Create Check Status Response

In Azure Functions, use `CreateCheckStatusResponseAsync` to return status URLs to the caller:

```csharp
[Function("StartOrchestration")]
public static async Task<HttpResponseData> HttpStart(
    [HttpTrigger(AuthorizationLevel.Anonymous, "post")] HttpRequestData req,
    [DurableClient] DurableTaskClient client)
{
    string instanceId = await client.ScheduleNewOrchestrationInstanceAsync("MyOrchestrator");
    
    // Returns response with status URLs
    return await client.CreateCheckStatusResponseAsync(req, instanceId);
}
```

Response includes:
- `statusQueryGetUri` — Check orchestration status
- `sendEventPostUri` — Send events to the orchestration
- `terminatePostUri` — Terminate the orchestration
- `purgeHistoryDeleteUri` — Delete orchestration history

---

## Instance Lifecycle

```
                    INSTANCE STATES
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│    ┌─────────┐                                              │
│    │ Pending │ ◀── ScheduleNewOrchestrationInstanceAsync    │
│    └────┬────┘                                              │
│         │                                                   │
│         ▼                                                   │
│    ┌─────────┐                                              │
│    │ Running │ ◀── Orchestrator executing                   │
│    └────┬────┘                                              │
│         │                                                   │
│    ┌────┴─────────────┬────────────────┐                    │
│    │                  │                │                    │
│    ▼                  ▼                ▼                    │
│ ┌──────────┐    ┌──────────┐    ┌────────────┐             │
│ │Completed │    │  Failed  │    │ Suspended  │             │
│ └──────────┘    └──────────┘    └─────┬──────┘             │
│                                       │                     │
│                                       ▼                     │
│                                  ResumeAsync                │
│                                       │                     │
│                                       ▼                     │
│                                  ┌─────────┐                │
│                                  │ Running │                │
│                                  └─────────┘                │
│                                                             │
│    Any state ─── TerminateAsync ───▶ ┌────────────┐        │
│                                      │ Terminated │        │
│                                      └────────────┘        │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Best Practices

### Use Meaningful Instance IDs

```csharp
// ✅ Good: Meaningful and traceable
var instanceId = $"order-{orderId}";
var instanceId = $"user-{userId}-signup-{timestamp}";

// ❌ Bad: Random GUIDs are hard to trace
var instanceId = Guid.NewGuid().ToString();  // Default if not specified
```

### Implement Singleton Orchestrations

Ensure only one instance runs for a given key:

```csharp
var instanceId = $"singleton-{key}";

// Check if already running
var existing = await client.GetInstanceAsync(instanceId);
if (existing?.RuntimeStatus == OrchestrationRuntimeStatus.Running)
{
    return existing;  // Already running
}

// Start new instance
await client.ScheduleNewOrchestrationInstanceAsync(
    "MyOrchestrator",
    new StartOrchestrationOptions { InstanceId = instanceId },
    input
);
```

### Handle Purging Responsibly

- Set up regular purging to manage storage costs
- Keep history for failed instances longer for debugging
- Consider compliance requirements for data retention

---

## Next Steps

- [Learn about External Events →](../patterns/external-events.md)
- [Explore the Dashboard →](../durable-task-scheduler/dashboard.md)
- [Understand Error Handling →](./error-handling.md)

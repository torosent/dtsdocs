---
layout: default
title: Instance Management
parent: Core Concepts
nav_order: 7
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

## Starting Instances

You can start a new orchestration instance using the client SDK. You must provide the orchestrator name and can optionally provide an instance ID and input.

```csharp
string instanceId = await client.ScheduleNewOrchestrationInstanceAsync("MyOrchestrator", input);
```

If you don't provide an instance ID, one will be auto-generated (GUID).

---

## Querying Instances

You can query the status of an orchestration instance using its ID.

### Get Status

```csharp
var metadata = await client.GetInstanceMetadataAsync(instanceId, getInputsAndOutputs: true);

if (metadata != null)
{
    Console.WriteLine($"Status: {metadata.RuntimeStatus}");
    Console.WriteLine($"Input: {metadata.Input}");
    Console.WriteLine($"Output: {metadata.Output}");
}
```

### Query with Filters

You can also query for multiple instances that match specific criteria (e.g., all failed instances created in the last hour).

```csharp
var filter = new OrchestrationQuery
{
    CreatedTimeFrom = DateTime.UtcNow.AddHours(-1),
    RuntimeStatus = new[] { OrchestrationRuntimeStatus.Failed }
};

var result = await client.GetInstancesAsync(filter);
```

---

## Terminating Instances

If an orchestration is stuck or running too long, you can forcibly terminate it.

```csharp
await client.TerminateInstanceAsync(instanceId, "Manually terminated by operator");
```

Terminating an instance stops it immediately. It transitions to the `Terminated` state.

---

## Purging History

Orchestration history takes up storage space. You can purge the history of completed, failed, or terminated instances to free up space and comply with data retention policies.

### Purge Single Instance

```csharp
await client.PurgeInstanceMetadataAsync(instanceId);
```

### Purge by Criteria

```csharp
var filter = new PurgeInstanceFilter
{
    CreatedTimeFrom = DateTime.MinValue,
    CreatedTimeTo = DateTime.UtcNow.AddDays(-30),
    RuntimeStatus = new[] { OrchestrationRuntimeStatus.Completed }
};

await client.PurgeInstancesAsync(filter);
```

---

## Rewinding Instances (Preview)

Rewinding allows you to restart a failed orchestration from a specific point in its history. This is useful for recovering from unexpected failures without re-running the entire workflow.

> **Note**: Rewind support depends on the storage provider and SDK version.

---

## Sending Events

You can send external events to running orchestrations to trigger logic or unblock them.

```csharp
await client.RaiseEventAsync(instanceId, "ApprovalEvent", true);
```

---

## HTTP API

If you are using Durable Functions, many of these operations are also exposed via built-in HTTP APIs.

- **Start**: `POST /runtime/webhooks/durabletask/orchestrators/{functionName}`
- **Status**: `GET /runtime/webhooks/durabletask/instances/{instanceId}`
- **Terminate**: `POST /runtime/webhooks/durabletask/instances/{instanceId}/terminate`
- **Raise Event**: `POST /runtime/webhooks/durabletask/instances/{instanceId}/raiseEvent/{eventName}`

The exact URLs depend on your function app configuration.

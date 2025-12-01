---
layout: default
title: Task Hubs
parent: Core Concepts
nav_order: 6
permalink: /docs/concepts/task-hubs/
---

# Task Hubs
{: .no_toc }

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

A **Task Hub** is a logical container for Azure Durable resources. It groups together orchestrations, entities, and activities that belong to a specific application or environment.

---

## What is a Task Hub?

Think of a task hub as a namespace or a boundary. All storage resources (queues, tables, blobs) used by a durable application are prefixed or tagged with the task hub name.

- **Isolation**: Multiple durable applications can share the same storage account by using different task hub names.
- **Grouping**: All instances within a task hub can interact with each other (e.g., sub-orchestrations).
- **Versioning**: You can use different task hubs to run different versions of your application side-by-side.

```
STORAGE ACCOUNT
┌──────────────────────────────────────────────────────────────────┐
│                                                                  │
│  ┌─────────────────────┐      ┌─────────────────────┐            │
│  │   Task Hub: App1    │      │   Task Hub: App2    │            │
│  │                     │      │                     │            │
│  │  • App1History      │      │  • App2History      │            │
│  │  • App1Instances    │      │  • App2Instances    │            │
│  │  • App1WorkItems    │      │  • App2WorkItems    │            │
│  │                     │      │                     │            │
│  └─────────────────────┘      └─────────────────────┘            │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

---

## Configuring Task Hubs

The task hub name is configured in your application settings or host configuration.

### Durable Functions (host.json)

In Azure Functions, the task hub name is defined in `host.json`.

```json
{
  "extensions": {
    "durableTask": {
      "hubName": "MyTaskHub"
    }
  }
}
```

> **Note**: If you don't specify a hub name, it defaults to `DurableFunctionsHub`.

### Durable Task SDKs

When using the SDKs directly, you specify the task hub when connecting to the backend.

**C# Example:**

```csharp
services.AddDurableTaskClient(builder =>
{
    builder.UseGrpc(options =>
    {
        options.Address = "http://localhost:4001";
        options.TaskHub = "MyTaskHub";
    });
});
```

---

## Task Hub Contents

A task hub typically consists of the following storage resources (depending on the storage provider):

1. **Orchestration History**: Stores the event history for all orchestration instances.
2. **Instance Status**: Stores the runtime status (Running, Completed, Failed) and input/output of instances.
3. **Work Item Queues**: Queues for scheduling activities and orchestrator replays.
4. **Lease Blobs**: Used for partition management and load balancing.

---

## Best Practices

1. **Naming Conventions**: Use alphanumeric characters. Avoid special characters. Keep names short (some storage providers have length limits).
2. **Environment Isolation**: Use different task hub names for Development, Staging, and Production (e.g., `MyAppDev`, `MyAppProd`).
3. **Shared Storage**: It's safe to share a storage account across multiple task hubs, but be mindful of storage limits (IOPS, throughput).
4. **Side-by-Side Deployments**: When deploying a breaking change, consider deploying to a new task hub (`MyAppV2`) to isolate the new version from existing running instances.

---

## Task Hub Management

You generally don't need to manage task hub resources manually. The framework automatically creates the necessary queues, tables, and blobs when the application starts.

However, you may need to delete a task hub to:
- Clean up test data
- Reset the state of an application
- Remove unused resources

This can be done via the [Durable Functions Monitor](../durable-task-scheduler/dashboard.md) or by manually deleting the underlying storage resources.

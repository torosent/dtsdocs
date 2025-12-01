---
layout: default
title: Glossary
nav_order: 10
permalink: /docs/glossary/
---

# Glossary
{: .no_toc }

A comprehensive reference of terms used throughout the Azure Durable documentation.
{: .fs-6 .fw-300 }

---

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## A

### Activity Function
A function that performs the actual work in a durable orchestration. Activities are where you execute business logic, make API calls, access databases, and perform other I/O operations. Unlike orchestrators, activities do not need to be deterministic. Also known as **Activity**.

### Azure Durable
An umbrella term for Microsoft's durable execution platform, encompassing Durable Functions, Durable Task Scheduler, and Durable Task SDKs.

### Azure Durable Functions
An extension of Azure Functions that enables writing stateful workflows in a serverless environment. See [Durable Functions Overview](./durable-functions/overview.md).

---

## B

### Backend Provider
The storage system that persists orchestration state and history. Options include the Durable Task Scheduler (recommended), Azure Storage, MSSQL, and Netherite. See [Storage Providers](./durable-functions/storage-providers.md).

---

## C

### Checkpoint
A point where the orchestration state is saved to durable storage. Checkpoints occur automatically when an orchestrator awaits an activity, timer, or external event. The framework uses checkpoints to recover orchestrations after failures.

### Client Function
A function that starts, queries, or manages orchestration instances. Client functions use the `DurableTaskClient` to interact with the Durable Task framework.

### ContinueAsNew
A method that restarts an orchestration with new input and a fresh history. Use this for eternal/long-running orchestrations to prevent unbounded history growth.

---

## D

### Determinism
The property of producing the same output given the same input. Orchestrator functions must be deterministic because they are replayed to recover state. Avoid using `DateTime.Now`, `Guid.NewGuid()`, `Random`, or direct I/O in orchestrators.

### Durable Entity
A stateful object that manages a small piece of state with explicit operations. Entities process operations one at a time, avoiding concurrency issues. Also known as **Entity Function**. See [Entities](./concepts/entities.md).

### Durable Task Framework (DTFx)
The open-source framework that powers all Azure Durable technologies. It implements event sourcing, replay, and durable timers. See [GitHub](https://github.com/Azure/durabletask).

### Durable Task Scheduler (DTS)
A fully managed Azure service that serves as the orchestration engine and state store for durable workloads. It provides the best performance and includes a built-in monitoring dashboard. See [DTS Overview](./durable-task-scheduler/overview.md).

### Durable Task SDKs
Portable client libraries (.NET, Python, Java) for building durable orchestrations that can run on any compute platform, not just Azure Functions. See [SDKs Overview](./sdks/overview.md).

### Durable Timer
A timer created via the orchestration context that survives process restarts. Unlike `Thread.Sleep()` or `Task.Delay()`, durable timers are persisted and will fire even if the host restarts.

---

## E

### Entity Function
See **Durable Entity**.

### Entity ID
A unique identifier for an entity instance, consisting of an entity name (the function name) and an entity key (a unique string).

### Event Sourcing
A pattern where state changes are stored as a sequence of events rather than as the current state. The Durable Task Framework uses event sourcing to record orchestration history and replay it for recovery.

### External Event
An event sent to a running orchestration from outside the orchestration context. Use `WaitForExternalEvent` in orchestrators and `RaiseEventAsync` from clients. See [External Events Pattern](./patterns/external-events.md).

---

## F

### Fan-Out/Fan-In
An orchestration pattern where multiple activities are executed in parallel (fan-out) and then aggregated (fan-in). See [Fan-Out/Fan-In Pattern](./patterns/fan-out-fan-in.md).

### Function Chaining
An orchestration pattern where activities are executed in sequence, with the output of one becoming the input of the next. See [Function Chaining Pattern](./patterns/function-chaining.md).

---

## G

### gRPC
A high-performance RPC framework used by the Durable Task Scheduler for communication. gRPC provides persistent connections, binary encoding, and lower latency compared to HTTP.

---

## H

### History
The sequence of events recorded for an orchestration instance. The history is used during replay to reconstruct the orchestration state without re-executing completed work.

### Human Interaction
An orchestration pattern that waits for human input (e.g., approval) before continuing. See [Human Interaction Pattern](./patterns/human-interaction.md).

---

## I

### Instance ID
A unique identifier for an orchestration instance. You can specify a custom instance ID or let the framework generate one. Instance IDs are used to query status, send events, and terminate orchestrations.

### Isolated Worker Model
The recommended hosting model for .NET Azure Functions, where the function code runs in a separate process from the Functions host. This model provides better dependency management and .NET version flexibility.

---

## M

### Managed Identity
An Azure Active Directory identity automatically managed by Azure. Used to authenticate to the Durable Task Scheduler without secrets. Can be system-assigned (tied to a resource) or user-assigned (independent resource).

---

## N

### Non-Deterministic Exception
An exception thrown when the orchestrator code produces different results during replay than during the original execution. This typically happens when using non-deterministic APIs like `DateTime.Now` or `Random`.

---

## O

### Orchestration
A workflow defined by an orchestrator function. An orchestration coordinates the execution of activities, sub-orchestrations, timers, and external events.

### Orchestration Context
The context object passed to orchestrator functions (`TaskOrchestrationContext`). It provides methods for calling activities, creating timers, waiting for events, and accessing orchestration metadata.

### Orchestration Status
The current state of an orchestration instance: Pending, Running, Completed, Failed, Suspended, or Terminated.

### Orchestrator Function
A function that defines the workflow logic. Orchestrators coordinate activities and must be deterministic. See [Orchestrators](./concepts/orchestrators.md).

---

## R

### Replay
The process of re-executing orchestrator code using stored history to reconstruct state. During replay, completed activities return their stored results instead of re-executing.

### Retry Policy
Configuration for automatic retries of failed activities. Includes settings for maximum attempts, retry interval, backoff coefficient, and retry timeout.

---

## S

### Saga Pattern
A pattern for managing distributed transactions using compensation. If a step fails, previously completed steps are "undone" by executing compensating actions. See [Error Handling](./concepts/error-handling.md).

### Sub-Orchestration
An orchestration called from within another orchestration. Useful for composing complex workflows from reusable components.

---

## T

### Task Hub
A logical container for orchestration and entity instances. Task hubs provide isolation between different workloads or environments. All instances in a task hub share the same history storage and work queues.

---

## V

### Versioning
Strategies for updating orchestrator code while instances are in flight. Options include side-by-side deployment (new task hub) or in-code versioning (version flag in input). See [Versioning](./concepts/versioning.md).

---

## W

### Worker
A process that hosts and executes orchestrators and activities. In Durable Functions, the Azure Functions host is the worker. With Durable Task SDKs, you create your own worker process.

---

## Related Documentation

- [Core Concepts →](./concepts/index.md)
- [Orchestration Patterns →](./patterns/index.md)
- [When to Use What →](./comparison/when-to-use.md)

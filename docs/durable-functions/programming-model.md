---
layout: default
title: Programming Model
parent: Durable Functions
nav_order: 3
---

# Durable Functions Programming Model

Durable Functions extends Azure Functions with a stateful programming model. This model allows you to write workflows (orchestrations) and stateful entities using code.

## Triggers and Bindings

Durable Functions introduces three new trigger types and one output binding.

### Orchestration Trigger

The **Orchestration Trigger** enables you to author durable orchestrator functions. This trigger supports starting new orchestrator instances and resuming existing orchestrator instances that are "awaiting" a task.

*   **Behavior**: Deterministic replay. The code runs multiple times to rebuild state.
*   **Usage**: Use `[OrchestrationTrigger]` (C#) or `orchestrationTrigger` (JSON) to define an orchestrator function.

### Activity Trigger

The **Activity Trigger** enables you to author activity functions. Activity functions are the basic unit of work in a durable orchestration.

*   **Behavior**: Stateless (idempotent recommended). Can perform I/O, database calls, etc.
*   **Usage**: Use `[ActivityTrigger]` (C#) or `activityTrigger` (JSON).

### Entity Trigger

The **Entity Trigger** enables you to author durable entity functions. These functions manage the state for a specific entity instance.

*   **Behavior**: Operations on a specific entity ID are serialized (executed one at a time).
*   **Usage**: Use `[EntityTrigger]` (C#) or `entityTrigger` (JSON).

### Durable Client Binding

The **Durable Client** binding (formerly "Orchestration Client") enables you to interact with the Durable Task framework. You can use this binding to:

*   Start new orchestration instances.
*   Query the status of orchestration instances.
*   Terminate or rewind instances.
*   Raise events to instances.

*   **Usage**: Use `[DurableClient]` (C#) or `durableClient` (JSON).

## Orchestrator Constraints

Orchestrator code must be **deterministic**. Because the Durable Task Framework replays the code to restore state, you must avoid:

1.  **Non-deterministic APIs**: `DateTime.Now`, `Guid.NewGuid()`, `Random`. Use the context-provided equivalents (e.g., `context.CurrentUtcDateTime`).
2.  **Blocking calls**: `Thread.Sleep`, `Task.Wait`. Use `await` on tasks returned by the context.
3.  **Infinite loops**: Use `context.ContinueAsNew` for eternal orchestrations.

## Activity Functions

Activity functions are where the "real work" happens. They are not restricted like orchestrators. They can make network calls, use standard libraries, and do not need to be deterministic.

## Durable Entities

Entities provide a way to define stateful objects. They are similar to actors in the Actor Model.

*   **State**: Persisted automatically.
*   **Access**: Via signals (one-way) or calls (request-response).

## Next Steps

*   [Learn about Orchestrators](../concepts/orchestrators.md)
*   [Learn about Activities](../concepts/activities.md)
*   [Learn about Entities](../concepts/entities.md)

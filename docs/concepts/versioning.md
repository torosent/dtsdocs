---
layout: default
title: Versioning
parent: Core Concepts
nav_order: 8
permalink: /docs/concepts/versioning/
---

# Versioning
{: .no_toc }

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

Versioning is a critical consideration for durable applications because of the replay mechanism. If you change the code of an orchestrator function while instances are still running (or suspended), the replay logic might fail or produce incorrect results.

---

## The Problem: Breaking Replay

When an orchestrator replays, it expects the code to be deterministic and match the history of events. If you change the code, the replay might diverge.

**Example of a Breaking Change:**

1. **Version 1**: Call Activity A, then Activity B.
2. **Instance Starts**: Runs Activity A, checkpoints, and waits for Activity B.
3. **Deploy Version 2**: Change code to Call Activity C, then Activity B.
4. **Instance Resumes**: Replays. History says "Activity A completed", but code says "Call Activity C".
5. **Error**: `NonDeterministicOrchestrationException`.

---

## Versioning Strategies

There are three main strategies for handling versioning in Durable Functions.

### 1. Do Nothing (Reset)

If you don't care about in-flight instances (e.g., in development), you can simply clear the task hub (delete storage artifacts) and restart. This is obviously not suitable for production.

### 2. Side-by-Side Deployment (New Task Hub)

Deploy the new version of your application to a **new task hub** (e.g., `MyAppV2`).

- **Pros**: Easiest to implement. Complete isolation.
- **Cons**: Requires managing multiple task hubs. Clients need to know which hub to use.
- **Workflow**:
    1. Deploy V2 app pointing to `MyAppV2` task hub.
    2. Keep V1 app running on `MyAppV1` task hub until all V1 instances complete.
    3. Decommission V1 app.

### 3. In-Code Versioning

Modify the orchestrator code to handle multiple versions logic using `context.Name` or a version flag in the input.

**Example:**

```csharp
[Function("ProcessOrder")]
public static async Task Run(
    [OrchestrationTrigger] TaskOrchestrationContext context)
{
    var input = context.GetInput<OrderInput>();
    
    if (input.Version == 2)
    {
        await context.CallActivityAsync("NewActivity", input);
    }
    else
    {
        await context.CallActivityAsync("OldActivity", input);
    }
}
```

This allows you to support existing instances (which have `Version=1` in their input) while new instances use the new logic.

---

## Mitigation

To minimize versioning issues:

- **Keep Orchestrators Simple**: Move complex logic to activities. Activities are not replayed, so you can change their implementation freely (as long as the signature doesn't change).
- **Use Sub-Orchestrators**: If you need to change a part of the workflow, you can version just the sub-orchestrator.
- **Plan for Long-Running Instances**: If your orchestrations run for months, side-by-side deployment is usually the best strategy.

---
layout: default
title: Orchestration Patterns
nav_order: 6
has_children: true
permalink: /docs/patterns/
---

# Orchestration Patterns
{: .no_toc }

Common patterns for building durable workflows with Durable Functions and Durable Task SDKs.
{: .fs-6 .fw-300 }

---

## Overview

| Pattern | Description | Use Case |
|---------|-------------|----------|
| [Function Chaining](function-chaining.md) | Execute activities in sequence | Multi-step workflows |
| [Fan-Out/Fan-In](fan-out-fan-in.md) | Parallel execution with aggregation | Batch processing |
| [Async HTTP APIs](async-http.md) | Long-running operations with polling | API integrations |
| [Human Interaction](human-interaction.md) | Wait for human approval | Approval workflows |
| [Aggregator](aggregator.md) | Stateful entity aggregation | Real-time aggregation |
| [External Events](external-events.md) | React to external signals | Event-driven workflows |

---

## Pattern Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│                       ORCHESTRATION PATTERNS                      │
├──────────────────────────────────────────────────────────────────┤
│                                                                   │
│  ┌─────────────┐   ┌─────────────┐   ┌─────────────────────────┐ │
│  │  Function   │   │  Fan-Out/   │   │    Async HTTP APIs      │ │
│  │  Chaining   │   │   Fan-In    │   │                         │ │
│  │             │   │             │   │  Client ──► Start ──►   │ │
│  │  A ──► B    │   │    ┌─A      │   │              │          │ │
│  │       │     │   │   ╱        ╲│   │              ▼          │ │
│  │       ▼     │   │  S ─B─────► E   │        Poll/Webhook     │ │
│  │       C     │   │   ╲        ╱│   │              │          │ │
│  │             │   │    └─C      │   │              ▼          │ │
│  │             │   │             │   │           Result        │ │
│  └─────────────┘   └─────────────┘   └─────────────────────────┘ │
│                                                                   │
│  ┌─────────────┐   ┌─────────────┐   ┌─────────────────────────┐ │
│  │   Human     │   │  Aggregator │   │   External Events       │ │
│  │ Interaction │   │  (Entity)   │   │                         │ │
│  │             │   │             │   │                         │ │
│  │  Request    │   │  ┌───────┐  │   │  Orchestrator           │ │
│  │     │       │   │  │ State │  │   │       │                 │ │
│  │     ▼       │   │  │  ┌─┐  │  │   │       ▼                 │ │
│  │  Wait for   │   │  │  │+│◄─┼──┼───│  WaitForEvent()         │ │
│  │  Approval   │   │  │  └─┘  │  │   │       │                 │ │
│  │     │       │   │  └───────┘  │   │       ▼                 │ │
│  │     ▼       │   │             │   │   Continue              │ │
│  │  Continue   │   │             │   │                         │ │
│  └─────────────┘   └─────────────┘   └─────────────────────────┘ │
│                                                                   │
└──────────────────────────────────────────────────────────────────┘
```

---

## Quick Examples

### Function Chaining

```csharp
// C# - Sequential execution
public override async Task<string> RunAsync(TaskOrchestrationContext context, string input)
{
    var result1 = await context.CallActivityAsync<string>("Step1", input);
    var result2 = await context.CallActivityAsync<string>("Step2", result1);
    var result3 = await context.CallActivityAsync<string>("Step3", result2);
    return result3;
}
```

### Fan-Out/Fan-In

```python
# Python - Parallel execution
@task.orchestrator
def batch_processor(ctx: task.OrchestrationContext, items: list):
    tasks = [ctx.call_activity("process", input=item) for item in items]
    results = yield task.when_all(tasks)
    return results
```

### Human Interaction

```java
// Java - Wait for approval
public ApprovalResult run(TaskOrchestrationContext ctx) {
    // Send notification
    ctx.callActivity("SendApprovalRequest", request, Void.class).await();
    
    // Wait for human response
    ApprovalEvent approval = ctx.waitForExternalEvent(
        "ApprovalReceived", 
        ApprovalEvent.class
    ).await();
    
    return new ApprovalResult(approval.isApproved());
}
```

---

## Choosing the Right Pattern

```
┌──────────────────────────────────────────────────────────────────┐
│                    PATTERN DECISION GUIDE                         │
├──────────────────────────────────────────────────────────────────┤
│                                                                   │
│  "Need to process steps in order?"                               │
│      │                                                            │
│      ├── Yes ──► Function Chaining                               │
│      │                                                            │
│      └── No ───► "Need to process many items?"                   │
│                       │                                           │
│                       ├── Yes ──► Fan-Out/Fan-In                 │
│                       │                                           │
│                       └── No ───► "Need human approval?"         │
│                                       │                           │
│                                       ├── Yes ──► Human Int.     │
│                                       │                           │
│                                       └── No ───► "Need to       │
│                                                    aggregate?"    │
│                                                        │          │
│                                                        ├── Yes    │
│                                                        │   ▼      │
│                                                        │ Aggregator│
│                                                        │          │
│                                                        └── No     │
│                                                            ▼      │
│                                                     External Events│
│                                                                   │
└──────────────────────────────────────────────────────────────────┘
```

---

## Detailed Guides

- [Function Chaining →](function-chaining.md)
- [Fan-Out/Fan-In →](fan-out-fan-in.md)
- [Async HTTP APIs →](async-http.md)
- [Human Interaction →](human-interaction.md)
- [Aggregator →](aggregator.md)
- [External Events →](external-events.md)

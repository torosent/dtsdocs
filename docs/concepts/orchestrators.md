---
layout: default
title: Orchestrator Functions
parent: Core Concepts
nav_order: 1
permalink: /docs/concepts/orchestrators/
---

# Orchestrator Functions
{: .no_toc }

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

Orchestrator functions are the heart of any durable application. They define the workflow logic that coordinates activities, sub-orchestrations, timers, and external events.

---

## What is an Orchestrator?

An orchestrator function is a special type of function that:

- **Coordinates workflow execution** — Decides what activities to run and in what order
- **Maintains state automatically** — State is persisted and recovered without your intervention
- **Survives failures** — Can resume from the last checkpoint after a crash or restart
- **Supports long-running processes** — Can run for seconds, minutes, days, or even indefinitely

```
                    ORCHESTRATOR FUNCTION
┌────────────────────────────────────────────────────────────┐
│                                                            │
│   Start ──▶ Activity 1 ──▶ Activity 2 ──▶ Activity 3       │
│                  │              │              │           │
│            [checkpoint]   [checkpoint]   [checkpoint]      │
│                                                            │
└────────────────────────────────────────────────────────────┘
```

---

## How Orchestrators Work

### Event Sourcing & Replay

Orchestrator functions use **event sourcing** to maintain their state. Every action (calling an activity, creating a timer, etc.) generates an event that is persisted to the orchestration history.

When an orchestrator needs to resume after a checkpoint, the framework **replays** the function from the beginning, using the stored history to skip already-completed work.

```
History:                      Replay:
┌─────────────────────┐      ┌─────────────────────────────────┐
│ 1. Started          │      │ Start execution                 │
│ 2. Activity1 called │  ──▶ │ Activity1 called → skip (done)  │
│ 3. Activity1 done   │      │ Activity2 called → execute now  │
│ 4. Activity2 called │      │                                 │
└─────────────────────┘      └─────────────────────────────────┘
```

### The Orchestration Cycle

1. **Trigger**: Orchestrator is triggered by an event (new instance, activity completion, timer, etc.)
2. **Replay**: Framework replays history up to the current point
3. **Execute**: Orchestrator runs until it hits an `await` on a new task
4. **Checkpoint**: State is saved; orchestrator can be unloaded from memory
5. **Resume**: When the awaited task completes, the cycle repeats

---

## Code Examples

### C# (Isolated Worker)

```csharp
using Microsoft.Azure.Functions.Worker;
using Microsoft.DurableTask;

public static class OrderProcessingOrchestrator
{
    [Function(nameof(ProcessOrder))]
    public static async Task<OrderResult> ProcessOrder(
        [OrchestrationTrigger] TaskOrchestrationContext context)
    {
        // Get the order from the input
        var order = context.GetInput<Order>();
        
        // Step 1: Validate the order
        var isValid = await context.CallActivityAsync<bool>("ValidateOrder", order);
        if (!isValid)
        {
            return new OrderResult { Success = false, Message = "Invalid order" };
        }
        
        // Step 2: Process payment
        var paymentResult = await context.CallActivityAsync<PaymentResult>(
            "ProcessPayment", 
            order.Payment
        );
        
        // Step 3: Update inventory
        await context.CallActivityAsync("UpdateInventory", order.Items);
        
        // Step 4: Send confirmation
        await context.CallActivityAsync("SendConfirmation", new {
            OrderId = order.Id,
            Email = order.CustomerEmail
        });
        
        return new OrderResult { 
            Success = true, 
            OrderId = order.Id,
            TransactionId = paymentResult.TransactionId
        };
    }
}
```

### Python

```python
import azure.durable_functions as df

def orchestrator_function(context: df.DurableOrchestrationContext):
    # Get the order from the input
    order = context.get_input()
    
    # Step 1: Validate the order
    is_valid = yield context.call_activity("ValidateOrder", order)
    if not is_valid:
        return {"success": False, "message": "Invalid order"}
    
    # Step 2: Process payment
    payment_result = yield context.call_activity("ProcessPayment", order["payment"])
    
    # Step 3: Update inventory
    yield context.call_activity("UpdateInventory", order["items"])
    
    # Step 4: Send confirmation
    yield context.call_activity("SendConfirmation", {
        "order_id": order["id"],
        "email": order["customer_email"]
    })
    
    return {
        "success": True,
        "order_id": order["id"],
        "transaction_id": payment_result["transaction_id"]
    }

main = df.Orchestrator.create(orchestrator_function)
```

### Java

```java
import com.microsoft.durabletask.*;

@FunctionName("ProcessOrder")
public OrderResult processOrder(
    @DurableOrchestrationTrigger(name = "ctx") TaskOrchestrationContext ctx) {
    
    // Get the order from the input
    Order order = ctx.getInput(Order.class);
    
    // Step 1: Validate the order
    boolean isValid = ctx.callActivity("ValidateOrder", order, Boolean.class).await();
    if (!isValid) {
        return new OrderResult(false, "Invalid order", null, null);
    }
    
    // Step 2: Process payment
    PaymentResult paymentResult = ctx.callActivity(
        "ProcessPayment", 
        order.getPayment(), 
        PaymentResult.class
    ).await();
    
    // Step 3: Update inventory
    ctx.callActivity("UpdateInventory", order.getItems(), Void.class).await();
    
    // Step 4: Send confirmation
    ctx.callActivity("SendConfirmation", Map.of(
        "orderId", order.getId(),
        "email", order.getCustomerEmail()
    ), Void.class).await();
    
    return new OrderResult(true, null, order.getId(), paymentResult.getTransactionId());
}
```

---

## Orchestrator Features

### Sub-Orchestrations

Orchestrators can call other orchestrators, enabling composition and reuse:

```csharp
// C#
var subResult = await context.CallSubOrchestratorAsync<SubResult>(
    "SubOrchestrator", 
    subInput
);
```

```python
# Python
sub_result = yield context.call_sub_orchestrator("SubOrchestrator", sub_input)
```

### Durable Timers

Create delays that survive restarts:

```csharp
// C#
var deadline = context.CurrentUtcDateTime.AddHours(1);
await context.CreateTimer(deadline, CancellationToken.None);
```

```python
# Python
deadline = context.current_utc_datetime + timedelta(hours=1)
yield context.create_timer(deadline)
```

### External Events

Wait for events from external systems or users:

```csharp
// C#
var approval = await context.WaitForExternalEvent<bool>("ApprovalEvent");
```

```python
# Python
approval = yield context.wait_for_external_event("ApprovalEvent")
```

### Custom Orchestration Status

You can set a custom status value for your orchestration that can be queried externally. This is useful for providing progress updates (e.g., "Processing item 5 of 10", "Waiting for approval").

```csharp
// C#
context.SetCustomStatus(new { Progress = 50, Message = "Halfway there" });
```

```python
# Python
context.set_custom_status({"progress": 50, "message": "Halfway there"})
```

### Retry Policies

Add automatic retries for activities:

```csharp
// C#
var options = new TaskOptions(new TaskRetryOptions(
    new RetryPolicy(
        maxNumberOfAttempts: 3,
        firstRetryInterval: TimeSpan.FromSeconds(5)
    )
));

var result = await context.CallActivityAsync<string>("UnreliableActivity", input, options);
```

---

## Best Practices

### ✅ Do

- Use activities for all I/O operations
- Use `context.CurrentUtcDateTime` instead of `DateTime.UtcNow`
- Keep orchestrator logic simple and delegate complex work to activities
- Use meaningful names for orchestration instances
- Log at the activity level, not in orchestrators (logs are replayed)

### ❌ Don't

- Call external APIs directly from orchestrators
- Use random number generators
- Access environment variables for non-deterministic values
- Use `Thread.Sleep()` or `Task.Delay()` directly
- Perform heavy computations (use activities instead)

---

## Orchestration Context API

| Method | Description |
|--------|-------------|
| `CallActivityAsync` | Schedules an activity function |
| `CallSubOrchestratorAsync` | Schedules a sub-orchestration |
| `CreateTimer` | Creates a durable timer |
| `WaitForExternalEvent` | Waits for an external event |
| `ContinueAsNew` | Restarts the orchestration with new input |
| `GetInput` | Gets the orchestration input |
| `CurrentUtcDateTime` | Gets the current replay-safe UTC time |
| `InstanceId` | Gets the unique orchestration instance ID |
| `IsReplaying` | Indicates if the orchestrator is replaying |

---

## Next Steps

- [Learn about Activities →](./activities.md)
- [Explore Orchestration Patterns →](../patterns/index.md)
- [View Code Samples →](../samples/index.md)

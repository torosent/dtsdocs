---
layout: default
title: Function Chaining
parent: Orchestration Patterns
nav_order: 1
permalink: /docs/patterns/function-chaining/
---

# Function Chaining Pattern
{: .no_toc }

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

Execute a sequence of functions in a specific order, where the output of one function becomes the input of the next.

---

## Overview

Function chaining is the most fundamental orchestration pattern. It executes activities sequentially, passing data from one step to the next.

```
┌─────────────────────────────────────────────────────────────┐
│                    FUNCTION CHAINING                         │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│    Input                                                     │
│      │                                                       │
│      ▼                                                       │
│  ┌───────────┐                                               │
│  │ Activity  │                                               │
│  │    A      │                                               │
│  └─────┬─────┘                                               │
│        │ Output A                                            │
│        ▼                                                     │
│  ┌───────────┐                                               │
│  │ Activity  │                                               │
│  │    B      │                                               │
│  └─────┬─────┘                                               │
│        │ Output B                                            │
│        ▼                                                     │
│  ┌───────────┐                                               │
│  │ Activity  │                                               │
│  │    C      │                                               │
│  └─────┬─────┘                                               │
│        │                                                     │
│        ▼                                                     │
│     Result                                                   │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Use Cases

- **Order Processing**: Validate → Charge → Ship → Notify
- **Document Processing**: Parse → Transform → Validate → Store
- **ETL Pipelines**: Extract → Transform → Load
- **User Onboarding**: Create Account → Send Email → Provision Resources

---

## Implementation

### C# (.NET)

```csharp
using Microsoft.DurableTask;

[DurableTask(nameof(OrderProcessingOrchestrator))]
public class OrderProcessingOrchestrator : TaskOrchestrator<Order, OrderResult>
{
    public override async Task<OrderResult> RunAsync(
        TaskOrchestrationContext context, 
        Order order)
    {
        // Step 1: Validate the order
        var validation = await context.CallActivityAsync<ValidationResult>(
            nameof(ValidateOrderActivity),
            order
        );
        
        if (!validation.IsValid)
        {
            return new OrderResult { 
                Success = false, 
                Error = validation.ErrorMessage 
            };
        }
        
        // Step 2: Reserve inventory
        var reservation = await context.CallActivityAsync<ReservationResult>(
            nameof(ReserveInventoryActivity),
            order.Items
        );
        
        // Step 3: Process payment
        var payment = await context.CallActivityAsync<PaymentResult>(
            nameof(ProcessPaymentActivity),
            new PaymentRequest
            {
                Amount = order.Total,
                PaymentMethod = order.PaymentMethod
            }
        );
        
        // Step 4: Ship the order
        var shipping = await context.CallActivityAsync<ShippingResult>(
            nameof(ShipOrderActivity),
            new ShippingRequest
            {
                OrderId = order.Id,
                ReservationId = reservation.ReservationId,
                Address = order.ShippingAddress
            }
        );
        
        // Step 5: Send confirmation
        await context.CallActivityAsync(
            nameof(SendConfirmationActivity),
            new ConfirmationEmail
            {
                OrderId = order.Id,
                Email = order.CustomerEmail,
                TrackingNumber = shipping.TrackingNumber
            }
        );
        
        return new OrderResult
        {
            Success = true,
            OrderId = order.Id,
            TrackingNumber = shipping.TrackingNumber,
            TransactionId = payment.TransactionId
        };
    }
}
```

### Python

```python
from durabletask import task

@task.orchestrator
def order_processing_orchestrator(ctx: task.OrchestrationContext, order: dict):
    """Process an order through multiple sequential steps."""
    
    # Step 1: Validate the order
    validation = yield ctx.call_activity("validate_order", input=order)
    
    if not validation["is_valid"]:
        return {"success": False, "error": validation["error_message"]}
    
    # Step 2: Reserve inventory
    reservation = yield ctx.call_activity(
        "reserve_inventory", 
        input=order["items"]
    )
    
    # Step 3: Process payment
    payment = yield ctx.call_activity(
        "process_payment",
        input={
            "amount": order["total"],
            "payment_method": order["payment_method"]
        }
    )
    
    # Step 4: Ship the order
    shipping = yield ctx.call_activity(
        "ship_order",
        input={
            "order_id": order["id"],
            "reservation_id": reservation["reservation_id"],
            "address": order["shipping_address"]
        }
    )
    
    # Step 5: Send confirmation
    yield ctx.call_activity(
        "send_confirmation",
        input={
            "order_id": order["id"],
            "email": order["customer_email"],
            "tracking_number": shipping["tracking_number"]
        }
    )
    
    return {
        "success": True,
        "order_id": order["id"],
        "tracking_number": shipping["tracking_number"],
        "transaction_id": payment["transaction_id"]
    }
```

### Java

```java
public class OrderProcessingOrchestrator implements TaskOrchestrator {
    @Override
    public OrderResult run(TaskOrchestrationContext ctx) {
        Order order = ctx.getInput(Order.class);
        
        // Step 1: Validate
        ValidationResult validation = ctx.callActivity(
            "ValidateOrder",
            order,
            ValidationResult.class
        ).await();
        
        if (!validation.isValid()) {
            return new OrderResult(false, validation.getErrorMessage());
        }
        
        // Step 2: Reserve inventory
        ReservationResult reservation = ctx.callActivity(
            "ReserveInventory",
            order.getItems(),
            ReservationResult.class
        ).await();
        
        // Step 3: Process payment
        PaymentResult payment = ctx.callActivity(
            "ProcessPayment",
            new PaymentRequest(order.getTotal(), order.getPaymentMethod()),
            PaymentResult.class
        ).await();
        
        // Step 4: Ship
        ShippingResult shipping = ctx.callActivity(
            "ShipOrder",
            new ShippingRequest(order.getId(), reservation.getId(), order.getAddress()),
            ShippingResult.class
        ).await();
        
        // Step 5: Confirm
        ctx.callActivity(
            "SendConfirmation",
            new ConfirmationEmail(order.getId(), order.getEmail(), shipping.getTrackingNumber()),
            Void.class
        ).await();
        
        return new OrderResult(true, order.getId(), shipping.getTrackingNumber());
    }
}
```

---

## Error Handling

### With Retry Policies

```csharp
// C# - Add retry for unreliable steps
var options = new TaskOptions(new TaskRetryOptions(
    new RetryPolicy(
        maxNumberOfAttempts: 3,
        firstRetryInterval: TimeSpan.FromSeconds(5),
        backoffCoefficient: 2.0
    )
));

var payment = await context.CallActivityAsync<PaymentResult>(
    nameof(ProcessPaymentActivity),
    paymentRequest,
    options
);
```

### With Compensation (Saga Pattern)

```csharp
public override async Task<OrderResult> RunAsync(
    TaskOrchestrationContext context, 
    Order order)
{
    // Track completed steps for compensation
    var completedSteps = new List<string>();
    
    try
    {
        // Step 1: Reserve inventory
        await context.CallActivityAsync(nameof(ReserveInventory), order);
        completedSteps.Add("inventory");
        
        // Step 2: Charge payment
        await context.CallActivityAsync(nameof(ChargePayment), order);
        completedSteps.Add("payment");
        
        // Step 3: Ship order
        await context.CallActivityAsync(nameof(ShipOrder), order);
        completedSteps.Add("shipping");
        
        return new OrderResult { Success = true };
    }
    catch (Exception ex)
    {
        // Compensate in reverse order
        foreach (var step in completedSteps.AsEnumerable().Reverse())
        {
            switch (step)
            {
                case "shipping":
                    await context.CallActivityAsync(nameof(CancelShipment), order);
                    break;
                case "payment":
                    await context.CallActivityAsync(nameof(RefundPayment), order);
                    break;
                case "inventory":
                    await context.CallActivityAsync(nameof(ReleaseInventory), order);
                    break;
            }
        }
        
        return new OrderResult { Success = false, Error = ex.Message };
    }
}
```

---

## Best Practices

1. **Keep activities focused** - Each activity should do one thing well
2. **Pass minimal data** - Only pass what's needed between steps
3. **Use retry policies** - Add retries for potentially failing operations
4. **Plan for compensation** - Design activities with rollback in mind
5. **Log at each step** - Activities can log; orchestrators should not

---

## Related Patterns

- [Fan-Out/Fan-In](fan-out-fan-in.md) - When steps can run in parallel
- [Human Interaction](human-interaction.md) - When a step requires approval
- [External Events](external-events.md) - When waiting for external systems

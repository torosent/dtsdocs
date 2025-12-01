---
layout: default
title: Activity Functions
parent: Core Concepts
nav_order: 2
permalink: /docs/concepts/activities/
---

# Activity Functions
{: .no_toc }

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

Activity functions are where the "real work" happens in a durable orchestration. They perform the actual business logic, I/O operations, and computations that your workflow needs.

---

## What is an Activity?

An activity function is a regular function that:

- **Performs work** — Database calls, API requests, file operations, computations
- **Is invoked by orchestrators** — Activities are always called from within an orchestrator
- **Can be retried** — The framework can automatically retry failed activities
- **Has no replay constraints** — Unlike orchestrators, activities don't need to be deterministic

```
ORCHESTRATOR                         ACTIVITIES
┌────────────┐                    ┌─────────────┐
│            │──── call ─────────▶│ ValidateData│
│            │◀─── result ────────│             │
│            │                    └─────────────┘
│            │                    ┌─────────────┐
│ Workflow   │──── call ─────────▶│ SaveToDb    │
│ Logic      │◀─── result ────────│             │
│            │                    └─────────────┘
│            │                    ┌─────────────┐
│            │──── call ─────────▶│ SendEmail   │
│            │◀─── result ────────│             │
└────────────┘                    └─────────────┘
```

---

## Activity Guarantees

### At-Least-Once Execution

Activities are guaranteed to execute **at least once**. If an activity fails after completing its work but before reporting success, it may be executed again.

**Implication**: Make your activities **idempotent** when possible — running them multiple times with the same input should produce the same result.

```csharp
// ✅ Idempotent - Uses upsert pattern
public async Task<bool> UpdateOrderStatus(OrderStatusUpdate update)
{
    await _database.UpsertAsync(new {
        OrderId = update.OrderId,
        Status = update.NewStatus,
        UpdatedAt = DateTime.UtcNow
    });
    return true;
}

// ⚠️ Not idempotent - Increments a counter
public async Task<int> IncrementCounter(string counterId)
{
    return await _database.IncrementAsync(counterId); // May double-increment!
}
```

---

## Code Examples

### C# (Isolated Worker)

```csharp
using Microsoft.Azure.Functions.Worker;
using Microsoft.Extensions.Logging;

public class OrderActivities
{
    private readonly ILogger<OrderActivities> _logger;
    private readonly IPaymentService _paymentService;
    
    public OrderActivities(ILogger<OrderActivities> logger, IPaymentService paymentService)
    {
        _logger = logger;
        _paymentService = paymentService;
    }

    [Function(nameof(ValidateOrder))]
    public async Task<bool> ValidateOrder([ActivityTrigger] Order order)
    {
        _logger.LogInformation("Validating order {OrderId}", order.Id);
        
        // Perform validation logic
        if (order.Items == null || !order.Items.Any())
        {
            _logger.LogWarning("Order {OrderId} has no items", order.Id);
            return false;
        }
        
        if (order.TotalAmount <= 0)
        {
            _logger.LogWarning("Order {OrderId} has invalid amount", order.Id);
            return false;
        }
        
        return true;
    }

    [Function(nameof(ProcessPayment))]
    public async Task<PaymentResult> ProcessPayment([ActivityTrigger] PaymentRequest request)
    {
        _logger.LogInformation("Processing payment for {Amount}", request.Amount);
        
        try
        {
            var result = await _paymentService.ChargeAsync(
                request.CardToken,
                request.Amount,
                request.Currency
            );
            
            return new PaymentResult
            {
                Success = true,
                TransactionId = result.TransactionId
            };
        }
        catch (PaymentException ex)
        {
            _logger.LogError(ex, "Payment failed");
            return new PaymentResult
            {
                Success = false,
                ErrorMessage = ex.Message
            };
        }
    }

    [Function(nameof(SendConfirmation))]
    public async Task SendConfirmation([ActivityTrigger] ConfirmationRequest request)
    {
        _logger.LogInformation("Sending confirmation to {Email}", request.Email);
        
        await _emailService.SendAsync(new Email
        {
            To = request.Email,
            Subject = $"Order {request.OrderId} Confirmed",
            Body = $"Thank you for your order!"
        });
    }
}
```

### Python

```python
import azure.durable_functions as df
import logging

myApp = df.DFApp()

@myApp.activity_trigger(input_name="order")
def validate_order(order: dict) -> bool:
    logging.info(f"Validating order {order['id']}")
    
    # Perform validation logic
    if not order.get('items'):
        logging.warning(f"Order {order['id']} has no items")
        return False
    
    if order.get('total_amount', 0) <= 0:
        logging.warning(f"Order {order['id']} has invalid amount")
        return False
    
    return True

@myApp.activity_trigger(input_name="request")
def process_payment(request: dict) -> dict:
    logging.info(f"Processing payment for {request['amount']}")
    
    try:
        # Call payment service
        result = payment_service.charge(
            card_token=request['card_token'],
            amount=request['amount'],
            currency=request['currency']
        )
        
        return {
            "success": True,
            "transaction_id": result.transaction_id
        }
    except PaymentError as e:
        logging.error(f"Payment failed: {e}")
        return {
            "success": False,
            "error_message": str(e)
        }

@myApp.activity_trigger(input_name="request")
def send_confirmation(request: dict) -> None:
    logging.info(f"Sending confirmation to {request['email']}")
    
    email_service.send(
        to=request['email'],
        subject=f"Order {request['order_id']} Confirmed",
        body="Thank you for your order!"
    )
```

### Java

```java
import com.microsoft.azure.functions.*;
import com.microsoft.azure.functions.annotation.*;

public class OrderActivities {
    
    private static final Logger logger = Logger.getLogger(OrderActivities.class.getName());
    
    @FunctionName("ValidateOrder")
    public boolean validateOrder(
        @DurableActivityTrigger(name = "order") Order order) {
        
        logger.info("Validating order " + order.getId());
        
        // Perform validation logic
        if (order.getItems() == null || order.getItems().isEmpty()) {
            logger.warning("Order " + order.getId() + " has no items");
            return false;
        }
        
        if (order.getTotalAmount() <= 0) {
            logger.warning("Order " + order.getId() + " has invalid amount");
            return false;
        }
        
        return true;
    }

    @FunctionName("ProcessPayment")
    public PaymentResult processPayment(
        @DurableActivityTrigger(name = "request") PaymentRequest request) {
        
        logger.info("Processing payment for " + request.getAmount());
        
        try {
            var result = paymentService.charge(
                request.getCardToken(),
                request.getAmount(),
                request.getCurrency()
            );
            
            return new PaymentResult(true, result.getTransactionId(), null);
        } catch (PaymentException e) {
            logger.severe("Payment failed: " + e.getMessage());
            return new PaymentResult(false, null, e.getMessage());
        }
    }

    @FunctionName("SendConfirmation")
    public void sendConfirmation(
        @DurableActivityTrigger(name = "request") ConfirmationRequest request) {
        
        logger.info("Sending confirmation to " + request.getEmail());
        
        emailService.send(
            request.getEmail(),
            "Order " + request.getOrderId() + " Confirmed",
            "Thank you for your order!"
        );
    }
}
```

---

## Activity Best Practices

### ✅ Do

- **Make activities idempotent** — Design so repeated execution is safe
- **Use dependency injection** — Inject services for testability
- **Log extensively** — Activities are the right place for detailed logging
- **Handle exceptions** — Return meaningful error information
- **Keep activities focused** — Each activity should do one thing well

### ❌ Don't

- **Don't call activities from activities** — Only orchestrators can call activities
- **Don't store shared state** — Each activity execution should be independent
- **Don't assume single execution** — Activities may run multiple times

---

## Activity Input/Output

### Single Value Input

```csharp
// Definition
[Function(nameof(Greet))]
public string Greet([ActivityTrigger] string name) => $"Hello, {name}!";

// Call from orchestrator
var greeting = await context.CallActivityAsync<string>("Greet", "World");
```

### Complex Object Input

```csharp
// Definition
[Function(nameof(ProcessOrder))]
public OrderResult ProcessOrder([ActivityTrigger] Order order) { ... }

// Call from orchestrator
var result = await context.CallActivityAsync<OrderResult>("ProcessOrder", order);
```

### Multiple Values (Use Object)

```csharp
// Use a class or record for multiple inputs
public record EmailRequest(string To, string Subject, string Body);

[Function(nameof(SendEmail))]
public void SendEmail([ActivityTrigger] EmailRequest request) { ... }

// Call from orchestrator
await context.CallActivityAsync("SendEmail", new EmailRequest(
    To: "user@example.com",
    Subject: "Hello",
    Body: "World"
));
```

---

## Retry Policies

Configure automatic retries for activities:

### C#

```csharp
var options = TaskOptions.FromRetryPolicy(new RetryPolicy(
    maxNumberOfAttempts: 5,
    firstRetryInterval: TimeSpan.FromSeconds(1),
    backoffCoefficient: 2.0,
    maxRetryInterval: TimeSpan.FromMinutes(1)
));

var result = await context.CallActivityAsync<string>(
    "UnreliableActivity", 
    input, 
    options
);
```

### Python

```python
retry_options = df.RetryOptions(
    first_retry_interval_in_milliseconds=1000,
    max_number_of_attempts=5
)

result = yield context.call_activity_with_retry(
    "UnreliableActivity",
    retry_options,
    input
)
```

---

## Next Steps

- [Learn about Entities →](./entities.md)
- [Explore Fan-Out/Fan-In Pattern →](../patterns/fan-out-fan-in.md)
- [View Code Samples →](../samples/index.md)

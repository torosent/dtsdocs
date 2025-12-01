---
layout: default
title: Error Handling
parent: Core Concepts
nav_order: 5
permalink: /docs/concepts/error-handling/
---

# Error Handling & Compensation
{: .no_toc }

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

Durable orchestrations must be resilient to failures. This guide covers how to handle errors, implement retries, and perform compensation logic when steps fail.

---

## Handling Exceptions

Orchestrator functions can use standard `try/catch` blocks to handle exceptions thrown by activity functions or sub-orchestrations.

### C# Example

```csharp
try
{
    await context.CallActivityAsync("TransferFunds", amount);
}
catch (TaskFailedException ex)
{
    logger.LogError("Transfer failed: {Message}", ex.Message);
    // Handle the error (e.g., call a cleanup activity)
}
```

### Python Example

```python
try:
    yield context.call_activity("TransferFunds", amount)
except Exception as e:
    logging.error(f"Transfer failed: {e}")
    # Handle the error
```

---

## Automatic Retries

Activities and sub-orchestrations often fail due to transient issues (network blips, timeouts). You can configure automatic retry policies to handle these cases without writing custom loop logic.

### Retry Options

| Option | Description |
|--------|-------------|
| **MaxNumberOfAttempts** | Maximum number of retry attempts |
| **FirstRetryInterval** | Wait time before the first retry |
| **BackoffCoefficient** | Multiplier for the wait time on subsequent retries |
| **MaxRetryInterval** | Maximum wait time between retries |
| **RetryTimeout** | Total timeout for all retries |

### C# Example

```csharp
var options = new TaskRetryOptions(
    new RetryPolicy(
        maxNumberOfAttempts: 3,
        firstRetryInterval: TimeSpan.FromSeconds(5),
        backoffCoefficient: 2.0
    )
);

await context.CallActivityAsync("UnreliableOperation", input, new TaskOptions(options));
```

### Python Example

```python
retry_options = df.RetryOptions(
    first_retry_interval_in_milliseconds=5000,
    max_number_of_attempts=3
)

yield context.call_activity_with_retry("UnreliableOperation", retry_options, input)
```

---

## Compensation

Compensation is the process of undoing work that was successfully completed when a later step in the workflow fails. This is critical for maintaining data consistency in distributed transactions (Saga pattern).

### Implementing Compensation

The standard pattern for compensation involves:
1. Tracking successfully completed steps
2. Catching exceptions
3. Executing "undo" activities for the completed steps in reverse order

### Example: Booking a Trip

```csharp
var outputs = new List<object>();

try
{
    // Step 1: Book Flight
    var flight = await context.CallActivityAsync<object>("BookFlight", input);
    outputs.Add(flight);

    // Step 2: Book Hotel
    var hotel = await context.CallActivityAsync<object>("BookHotel", input);
    outputs.Add(hotel);

    // Step 3: Book Car (Fails)
    var car = await context.CallActivityAsync<object>("BookCar", input);
    outputs.Add(car);
}
catch (Exception)
{
    // Compensation Logic
    if (outputs.Count > 1) 
    {
        await context.CallActivityAsync("CancelHotel", outputs[1]);
    }
    if (outputs.Count > 0) 
    {
        await context.CallActivityAsync("CancelFlight", outputs[0]);
    }
    throw;
}
```

---

## Timeout Handling

You can enforce time limits on orchestrations or specific tasks using timers.

### Task Timeout

To timeout a specific activity, use `Task.WhenAny` with a timer.

```csharp
var activityTask = context.CallActivityAsync("LongRunningTask");
var timeoutTask = context.CreateTimer(context.CurrentUtcDateTime.AddMinutes(5), CancellationToken.None);

var winner = await Task.WhenAny(activityTask, timeoutTask);

if (winner == activityTask)
{
    // Task completed successfully
}
else
{
    // Timeout occurred
}
```

---

## Best Practices

1. **Use Retries for Transient Errors**: Configure retries for network calls and external services.
2. **Use Compensation for Business Failures**: If a logical step fails (e.g., "insufficient funds"), undo previous steps.
3. **Idempotency**: Ensure activity functions are idempotent so they can be safely retried.
4. **Log Errors**: Always log exceptions in your orchestrator to aid debugging.
5. **Avoid Infinite Loops**: Be careful with retry policies that could loop indefinitely.

---
layout: default
title: Samples
parent: Durable Functions
nav_order: 20
---

# Durable Functions Samples

This section contains code samples specifically for **Azure Durable Functions** (Serverless).

## Quickstarts

*   [Hello World (C#)](./quickstart.md) - A basic orchestration that calls three activities.

## Common Patterns

### Function Chaining

Execute a sequence of functions in a specific order.

```csharp
[Function("Chaining")]
public static async Task<object> Run(
    [OrchestrationTrigger] TaskOrchestrationContext context)
{
    try
    {
        var x = await context.CallActivityAsync<object>("F1", null);
        var y = await context.CallActivityAsync<object>("F2", x);
        var z = await context.CallActivityAsync<object>("F3", y);
        return  await context.CallActivityAsync<object>("F4", z);
    }
    catch (Exception)
    {
        // Error handling or compensation goes here.
    }
    return null;
}
```

### Fan-out/Fan-in

Execute multiple functions in parallel and then wait for all to finish.

```csharp
[Function("FanOutFanIn")]
public static async Task Run(
    [OrchestrationTrigger] TaskOrchestrationContext context)
{
    var parallelTasks = new List<Task<int>>();

    // Get a list of N work items to process in parallel.
    object[] workBatch = await context.CallActivityAsync<object[]>("F1", null);

    for (int i = 0; i < workBatch.Length; i++)
    {
        Task<int> task = context.CallActivityAsync<int>("F2", workBatch[i]);
        parallelTasks.Add(task);
    }

    await Task.WhenAll(parallelTasks);

    // Aggregate all N outputs and send the result to F3.
    int sum = parallelTasks.Sum(t => t.Result);
    await context.CallActivityAsync("F3", sum);
}
```

### Async HTTP APIs

Implement the [Async HTTP API pattern](../patterns/index.md#async-http-apis) to handle long-running operations.

## External Samples

*   [Azure Durable Functions Samples (GitHub)](https://github.com/Azure/azure-functions-durable-extension/tree/dev/samples)
*   [Serverless Community Library](https://github.com/Azure/azure-functions-durable-extension/tree/dev/samples)

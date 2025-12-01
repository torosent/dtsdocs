---
layout: default
title: .NET SDK
parent: SDKs Overview
grand_parent: Developer Reference
nav_order: 1
has_children: true
permalink: /docs/developer-guide/sdks/dotnet/
---

# .NET SDK Reference
{: .no_toc }

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

Comprehensive reference for Durable Task .NET SDKs.
{: .fs-6 .fw-300 }

---

## SDK Options

The .NET ecosystem offers three SDK options:

| SDK Package | Target Framework | Hosting Model |
|-------------|------------------|---------------|
| **Microsoft.DurableTask** | .NET 6+ | Portable (any host) |
| **Worker.Extensions.DurableTask** | .NET 6+ | Azure Functions (Isolated) |
| **WebJobs.Extensions.DurableTask** | .NET Framework / .NET Core | Azure Functions (In-Process) |

---

## Portable SDK (Microsoft.DurableTask)

The portable SDK is the recommended choice for new applications not hosted on Azure Functions.

### Installation

```bash
# Worker package for orchestrations and activities
dotnet add package Microsoft.DurableTask.Worker.AzureManaged

# Client package for starting and managing orchestrations
dotnet add package Microsoft.DurableTask.Client.AzureManaged
```

### Features

- **Type-safe orchestrations and activities** with strong compile-time validation
- **Source generators** for automatic registration
- **Dependency injection** integration with `Microsoft.Extensions.Hosting`
- **Platform agnostic** - runs on any .NET host

### Basic Setup

```csharp
using Microsoft.Extensions.Hosting;
using Microsoft.DurableTask.Worker;
using Microsoft.DurableTask.Client;

var builder = Host.CreateApplicationBuilder(args);

// Add Durable Task worker
builder.Services.AddDurableTaskWorker()
    .AddTasks(registry =>
    {
        registry.AddOrchestrator<MyOrchestration>();
        registry.AddActivity<MyActivity>();
    })
    .UseDurableTaskScheduler(connectionString);

// Add Durable Task client
builder.Services.AddDurableTaskClient()
    .UseDurableTaskScheduler(connectionString);

var host = builder.Build();
await host.RunAsync();
```

### Defining Orchestrations

```csharp
[DurableTask]
public class MyOrchestration : TaskOrchestrator<string, string>
{
    public override async Task<string> RunAsync(
        TaskOrchestrationContext context, 
        string input)
    {
        var result1 = await context.CallActivityAsync<string>(
            nameof(SayHello), "Tokyo");
        var result2 = await context.CallActivityAsync<string>(
            nameof(SayHello), "Seattle");
        var result3 = await context.CallActivityAsync<string>(
            nameof(SayHello), "London");
        
        return $"{result1}; {result2}; {result3}";
    }
}
```

### Defining Activities

```csharp
[DurableTask]
public class SayHello : TaskActivity<string, string>
{
    private readonly ILogger<SayHello> _logger;
    
    public SayHello(ILogger<SayHello> logger)
    {
        _logger = logger;
    }
    
    public override Task<string> RunAsync(
        TaskActivityContext context,
        string input)
    {
        _logger.LogInformation("Saying hello to {Name}", input);
        return Task.FromResult($"Hello {input}!");
    }
}
```

### Client Operations

```csharp
public class OrchestrationController
{
    private readonly DurableTaskClient _client;
    
    public OrchestrationController(DurableTaskClient client)
    {
        _client = client;
    }
    
    public async Task<string> StartOrchestration(string input)
    {
        // Schedule a new orchestration
        string instanceId = await _client.ScheduleNewOrchestrationInstanceAsync(
            nameof(MyOrchestration),
            input);
        
        return instanceId;
    }
    
    public async Task<string?> GetResult(string instanceId)
    {
        // Wait for completion
        var result = await _client.WaitForInstanceCompletionAsync(
            instanceId,
            getInputsAndOutputs: true);
        
        return result?.ReadOutputAs<string>();
    }
}
```

---

## Azure Functions Isolated Worker SDK

For Azure Functions using the .NET isolated worker model.

### Installation

```bash
dotnet add package Microsoft.Azure.Functions.Worker.Extensions.DurableTask
```

### Basic Setup

```csharp
using Microsoft.Azure.Functions.Worker;
using Microsoft.DurableTask;
using Microsoft.DurableTask.Client;

public class DurableFunctionsOrchestration
{
    [Function(nameof(HelloCities))]
    public async Task<List<string>> HelloCities(
        [OrchestrationTrigger] TaskOrchestrationContext context)
    {
        var outputs = new List<string>();
        
        outputs.Add(await context.CallActivityAsync<string>(
            nameof(SayHello), "Tokyo"));
        outputs.Add(await context.CallActivityAsync<string>(
            nameof(SayHello), "Seattle"));
        outputs.Add(await context.CallActivityAsync<string>(
            nameof(SayHello), "London"));
        
        return outputs;
    }

    [Function(nameof(SayHello))]
    public string SayHello(
        [ActivityTrigger] string name,
        FunctionContext executionContext)
    {
        ILogger logger = executionContext.GetLogger(nameof(SayHello));
        logger.LogInformation("Saying hello to {name}", name);
        return $"Hello {name}!";
    }

    [Function("HelloCities_HttpStart")]
    public async Task<HttpResponseData> HttpStart(
        [HttpTrigger(AuthorizationLevel.Anonymous, "get", "post")] 
        HttpRequestData req,
        [DurableClient] DurableTaskClient client,
        FunctionContext executionContext)
    {
        string instanceId = await client.ScheduleNewOrchestrationInstanceAsync(
            nameof(HelloCities));

        return await client.CreateCheckStatusResponseAsync(req, instanceId);
    }
}
```

### Key Differences from In-Process

| Feature | In-Process | Isolated Worker |
|---------|------------|-----------------|
| Attribute | `[FunctionName]` | `[Function]` |
| Context | `IDurableOrchestrationContext` | `TaskOrchestrationContext` |
| Client | `IDurableClient` | `DurableTaskClient` |
| Binding | `[DurableClient]` | `[DurableClient]` |

---

## In-Process SDK (Legacy)

For Azure Functions using the in-process model (deprecated).

### Installation

```bash
dotnet add package Microsoft.Azure.WebJobs.Extensions.DurableTask
```

### Basic Setup

```csharp
using Microsoft.Azure.WebJobs;
using Microsoft.Azure.WebJobs.Extensions.DurableTask;
using Microsoft.Azure.WebJobs.Extensions.Http;
using Microsoft.Extensions.Logging;

public static class DurableFunctionsOrchestration
{
    [FunctionName("HelloCities")]
    public static async Task<List<string>> RunOrchestrator(
        [OrchestrationTrigger] IDurableOrchestrationContext context)
    {
        var outputs = new List<string>();
        
        outputs.Add(await context.CallActivityAsync<string>(
            "SayHello", "Tokyo"));
        outputs.Add(await context.CallActivityAsync<string>(
            "SayHello", "Seattle"));
        outputs.Add(await context.CallActivityAsync<string>(
            "SayHello", "London"));
        
        return outputs;
    }

    [FunctionName("SayHello")]
    public static string SayHello(
        [ActivityTrigger] string name, 
        ILogger log)
    {
        log.LogInformation($"Saying hello to {name}.");
        return $"Hello {name}!";
    }

    [FunctionName("HelloCities_HttpStart")]
    public static async Task<HttpResponseMessage> HttpStart(
        [HttpTrigger(AuthorizationLevel.Anonymous, "get", "post")] 
        HttpRequestMessage req,
        [DurableClient] IDurableOrchestrationClient starter,
        ILogger log)
    {
        string instanceId = await starter.StartNewAsync("HelloCities", null);
        
        log.LogInformation($"Started orchestration with ID = '{instanceId}'.");
        
        return starter.CreateCheckStatusResponse(req, instanceId);
    }
}
```

> **Warning:** The in-process model is being deprecated. New projects should use the isolated worker model or portable SDK.

---

## Detailed Documentation

- [Unit Testing →](./unit-testing.md) - Testing orchestrations, activities, and trigger functions
- [Durable Entities →](./entities.md) - Stateful entity programming model
- [WebJobs Integration →](./webjobs.md) - Running Durable Functions as WebJobs
- [Schedules →](./schedules.md) - Recurring orchestrations with cron expressions
- [Portable API Reference →](./api-reference-portable.md) - Microsoft.DurableTask API documentation
- [Isolated API Reference →](./api-reference-isolated.md) - Worker.Extensions.DurableTask API documentation
- [Legacy API Reference →](./api-reference-legacy.md) - WebJobs.Extensions.DurableTask API documentation

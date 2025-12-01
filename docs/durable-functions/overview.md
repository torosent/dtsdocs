---
layout: default
title: Durable Functions
nav_order: 3
has_children: true
permalink: /docs/durable-functions/
---

# Azure Durable Functions
{: .no_toc }

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

Azure Durable Functions is an extension of Azure Functions that lets you write **stateful workflows** in a serverless compute environment. It's the easiest way to get started with durable orchestrations if you're already using Azure Functions.

[Get Started →](./quickstart.md){: .btn .btn-primary .fs-5 .mb-4 .mb-md-0 .mr-2 }
[View Samples →](./samples.md){: .btn .fs-5 .mb-4 .mb-md-0 }

---

## What are Durable Functions?

Durable Functions lets you write **orchestrator functions** that coordinate the execution of other functions. Behind the scenes, the extension handles:

- State persistence and recovery
- Automatic checkpointing
- Replay-based execution
- Long-running workflows

```
┌─────────────────────────────────────────────────────────────────┐
│                    AZURE FUNCTIONS HOST                         │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │              DURABLE FUNCTIONS EXTENSION                   │ │
│  │  ┌──────────────────────────────────────────────────────┐ │ │
│  │  │  Your Code                                           │ │ │
│  │  │  ├── Orchestrator Functions                          │ │ │
│  │  │  ├── Activity Functions                              │ │ │
│  │  │  ├── Entity Functions                                │ │ │
│  │  │  └── Client Functions                                │ │ │
│  │  └──────────────────────────────────────────────────────┘ │ │
│  │                          │                                 │ │
│  │                          ▼                                 │ │
│  │  ┌──────────────────────────────────────────────────────┐ │ │
│  │  │  Storage Backend                                     │ │ │
│  │  │  ├── Durable Task Scheduler (recommended)            │ │ │
│  │  │  ├── Azure Storage                                   │ │ │
│  │  │  └── MSSQL                                           │ │ │
│  │  └──────────────────────────────────────────────────────┘ │ │
│  └───────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

---

## Key Benefits

| Benefit | Description |
|---------|-------------|
| **Serverless** | No infrastructure to manage; pay only for execution time |
| **Event-Driven** | Easily integrate with Azure services via triggers and bindings |
| **Familiar Model** | Same programming model as regular Azure Functions |
| **Multi-Language** | Support for C#, JavaScript, TypeScript, Python, Java, PowerShell |
| **Local Development** | Full local debugging with Azure Functions Core Tools |
| **Built-in Patterns** | Pre-built support for common workflow patterns |

---

## Supported Languages

| Language | Worker Model | Status |
|----------|--------------|--------|
| C# | Isolated Process | ✅ Generally Available |
| C# | In-Process | ⚠️ Legacy (end of support Nov 2026) |
| JavaScript | Node.js | ✅ Generally Available |
| TypeScript | Node.js | ✅ Generally Available |
| Python | Python Worker | ✅ Generally Available |
| Java | Java Worker | ✅ Generally Available |
| PowerShell | PowerShell Worker | ✅ Generally Available |

---

## .NET Worker Models: In-Process vs Isolated

For .NET developers, Azure Functions supports two execution models. Understanding the differences is critical for new projects and existing applications.

{: .warning }
> **Important:** Support for the in-process model ends on **November 10, 2026**. We highly recommend migrating to the isolated worker model. See our [Migration Guide](./migration-guide.md) for step-by-step instructions.

### Comparison Table

| Feature | Isolated Worker Model | In-Process Model |
|---------|----------------------|------------------|
| **Support Status** | ✅ Actively supported | ⚠️ End of support Nov 2026 |
| **.NET Versions** | .NET 8, .NET 9, .NET 10, .NET Framework 4.8 | .NET 8 only |
| **Process Isolation** | Separate worker process | Same process as host |
| **Assembly Conflicts** | ✅ No conflicts with host | ❌ Potential version conflicts |
| **Middleware Support** | ✅ Full middleware pipeline | ❌ Not supported |
| **Dependency Injection** | ✅ Standard .NET DI | ⚠️ Limited (via FunctionsStartup) |
| **Startup Control** | ✅ Full control via Program.cs | ❌ Limited |
| **ASP.NET Core Integration** | ✅ Full support | ❌ Not supported |
| **Cold Start** | ⚠️ Slightly higher (configurable) | ✅ Optimized |
| **Flex Consumption Plan** | ✅ Supported | ❌ Not supported |
| **.NET Aspire** | ✅ Preview support | ❌ Not supported |

### In-Process Model Limitations

The in-process model has several limitations that make the isolated worker model the recommended choice:

1. **End of Support**: The in-process model will reach end of support on November 10, 2026. After this date, no security updates or bug fixes will be provided.

2. **Limited .NET Version Support**: Only supports LTS versions of .NET, ending with .NET 8. Cannot use newer .NET versions or .NET Framework.

3. **Assembly Version Conflicts**: Because your code runs in the same process as the Functions host, you may encounter assembly version conflicts with host dependencies.

4. **No Middleware Support**: Cannot use ASP.NET Core middleware patterns for cross-cutting concerns like authentication, logging, or error handling.

5. **Limited Dependency Injection**: Requires using `FunctionsStartup` attribute rather than standard .NET patterns.

6. **No ASP.NET Core Integration**: Cannot use familiar ASP.NET Core types like `HttpRequest` and `IActionResult` with full feature support.

7. **Platform Limitations**: Not supported on Flex Consumption plan and cannot use .NET Aspire integration.

### Benefits of Isolated Worker Model

The isolated worker model provides significant advantages:

- **Fewer Conflicts**: Your code runs in a separate process, eliminating assembly conflicts with the host.
- **Full Process Control**: Control startup, configuration, and middleware through standard `Program.cs`.
- **Standard Dependency Injection**: Use familiar .NET dependency injection patterns.
- **.NET Version Flexibility**: Use any supported .NET version, including STS releases and .NET Framework.
- **Better Observability**: Enhanced logging and telemetry control.
- **Future-Proof**: All new features and improvements target the isolated model.

### Migration Path

If you're currently using the in-process model, we strongly recommend migrating to the isolated worker model. The migration involves:

1. Updating project dependencies
2. Adding a `Program.cs` file
3. Updating function signatures and attributes
4. Updating `local.settings.json`

For detailed migration instructions, see the [Migration Guide](./migration-guide.md).

---

## Function Types

> **Important:** To successfully build durable applications, you must understand the constraints and patterns of the [Programming Model](./programming-model.md).

Durable Functions introduces four special function types:

### 1. Orchestrator Functions

Define the workflow logic:

```csharp
[Function(nameof(OrderWorkflow))]
public static async Task<OrderResult> OrderWorkflow(
    [OrchestrationTrigger] TaskOrchestrationContext context)
{
    var order = context.GetInput<Order>();
    
    // Coordinate activities
    await context.CallActivityAsync("ValidateOrder", order);
    await context.CallActivityAsync("ProcessPayment", order.Payment);
    await context.CallActivityAsync("ShipOrder", order);
    
    return new OrderResult { Success = true };
}
```

### 2. Activity Functions

Perform the actual work:

```csharp
[Function(nameof(ValidateOrder))]
public static bool ValidateOrder([ActivityTrigger] Order order)
{
    // Validation logic here
    return order.Items.Any() && order.TotalAmount > 0;
}
```

### 3. Entity Functions

Manage stateful entities:

```csharp
[Function(nameof(Counter))]
public static Task Counter([EntityTrigger] TaskEntityDispatcher dispatcher)
{
    return dispatcher.DispatchAsync<Counter>();
}
```

### 4. Client Functions

Start and manage orchestrations:

```csharp
[Function("StartOrder")]
public static async Task<HttpResponseData> StartOrder(
    [HttpTrigger(AuthorizationLevel.Anonymous, "post")] HttpRequestData req,
    [DurableClient] DurableTaskClient client)
{
    var order = await req.ReadFromJsonAsync<Order>();
    string instanceId = await client.ScheduleNewOrchestrationInstanceAsync(
        nameof(OrderWorkflow), 
        order
    );
    
    return client.CreateCheckStatusResponse(req, instanceId);
}
```

---

## Storage Backend Options

Durable Functions supports multiple storage backends. [Learn more about Storage Providers](./storage-providers.md).

### Durable Task Scheduler (Recommended)

The fully managed backend with the best performance and monitoring:

```json
// host.json
{
  "extensions": {
    "durableTask": {
      "storageProvider": {
        "type": "durabletaskscheduler",
        "connectionStringName": "DURABLE_TASK_SCHEDULER_CONNECTION_STRING",
        "taskHubName": "myTaskHub"
      }
    }
  }
}
```

### Azure Storage (Default)

The traditional bring-your-own-storage option:

```json
// host.json
{
  "extensions": {
    "durableTask": {
      "storageProvider": {
        "connectionStringName": "AzureWebJobsStorage"
      }
    }
  }
}
```

### MSSQL

For organizations preferring SQL Server:

```json
// host.json
{
  "extensions": {
    "durableTask": {
      "storageProvider": {
        "type": "mssql",
        "connectionStringName": "SqlConnectionString"
      }
    }
  }
}
```

---

## When to Use Durable Functions

### ✅ Great For

- **Serverless workflows** — Event-driven, pay-per-execution
- **Azure integrations** — Leverage Azure Functions triggers and bindings
- **Multi-language teams** — Support for 6+ languages
- **Rapid development** — Familiar Azure Functions model
- **Existing Functions apps** — Add orchestration to existing apps

### ⚠️ Consider Alternatives

- **Container-based workloads** — Consider Durable Task SDKs on ACA/AKS
- **Custom hosting requirements** — Consider Durable Task SDKs
- **Non-Azure deployments** — Consider Durable Task SDKs

---

## Quick Start

### Prerequisites

- Azure Functions Core Tools v4.x
- .NET 8.0 SDK (for C#)
- Docker (for local Durable Task Scheduler emulator)

### Create a New Project

```bash
# Create a new Durable Functions project
func init MyDurableApp --worker-runtime dotnet-isolated
cd MyDurableApp

# Add Durable Functions extension
dotnet add package Microsoft.Azure.Functions.Worker.Extensions.DurableTask
```

### Sample Orchestration

```csharp
using Microsoft.Azure.Functions.Worker;
using Microsoft.Azure.Functions.Worker.Http;
using Microsoft.DurableTask;
using Microsoft.DurableTask.Client;

namespace MyDurableApp;

public static class HelloCities
{
    [Function(nameof(StartHello))]
    public static async Task<HttpResponseData> StartHello(
        [HttpTrigger(AuthorizationLevel.Anonymous, "get")] HttpRequestData req,
        [DurableClient] DurableTaskClient client)
    {
        string instanceId = await client.ScheduleNewOrchestrationInstanceAsync(
            nameof(HelloOrchestrator)
        );
        return client.CreateCheckStatusResponse(req, instanceId);
    }

    [Function(nameof(HelloOrchestrator))]
    public static async Task<string> HelloOrchestrator(
        [OrchestrationTrigger] TaskOrchestrationContext context)
    {
        var result = "";
        result += await context.CallActivityAsync<string>(nameof(SayHello), "Tokyo") + " ";
        result += await context.CallActivityAsync<string>(nameof(SayHello), "London") + " ";
        result += await context.CallActivityAsync<string>(nameof(SayHello), "Seattle");
        return result;
    }

    [Function(nameof(SayHello))]
    public static string SayHello([ActivityTrigger] string cityName)
    {
        return $"Hello, {cityName}!";
    }
}
```

### Run Locally

```bash
# Start the Durable Task Scheduler emulator
docker run -itP mcr.microsoft.com/dts/dts-emulator:latest

# Start the function app
func start
```

---

## Hosting Options

| Hosting Plan | Description | Scale |
|--------------|-------------|-------|
| **Consumption** | Pay-per-execution, auto-scale | 0 → 200 instances |
| **Flex Consumption** | Enhanced consumption with VNet | 0 → 1000 instances |
| **Premium** | Pre-warmed instances, VNet | 0 → 100 instances |
| **Dedicated** | App Service plan | Fixed instances |
| **Container Apps** | Container-based hosting | Dynamic |

---

## Related Documentation

- [Quickstart: Create your first Durable Function](./quickstart.md)
- **[Programming Model](./programming-model.md)**
- [Storage Providers](./storage-providers.md)
- [Migration Guide: In-Process to Isolated](./migration-guide.md)
- [Durable Functions Overview (Microsoft Learn)](https://learn.microsoft.com/azure/azure-functions/durable/)
- [Configure Durable Task Scheduler](../durable-task-scheduler/setup.md)
- [Orchestration Patterns](../patterns/index.md)

---

## Next Steps

- [Create your first Durable Function →](./quickstart.md)
- [Learn about the Programming Model →](./programming-model.md)
- [Choose a Storage Provider →](./storage-providers.md)
- [Migrate from In-Process to Isolated →](./migration-guide.md)
- [Explore Orchestration Patterns →](../patterns/index.md)
- [View Code Samples →](./samples.md)

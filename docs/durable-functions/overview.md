---
layout: default
title: Durable Functions
nav_order: 3
has_children: false
permalink: /docs/durable-functions/
---

# Azure Durable Functions
{: .no_toc }

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

Azure Durable Functions is an extension of Azure Functions that enables you to write stateful workflows in a serverless compute environment. It's the easiest way to get started with durable orchestrations if you're already using Azure Functions.

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

## Function Types

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

- [Durable Functions Overview (Microsoft Learn)](https://learn.microsoft.com/azure/azure-functions/durable/)
- [Quickstart: Create a Durable Functions app](https://learn.microsoft.com/azure/azure-functions/durable/durable-functions-isolated-create-first-csharp)
- [Configure Durable Task Scheduler](../durable-task-scheduler/setup.md)
- [Orchestration Patterns](../patterns/index.md)

---

## Next Steps

- [Set up Durable Task Scheduler →](../durable-task-scheduler/setup.md)
- [Explore Orchestration Patterns →](../patterns/index.md)
- [View Code Samples →](../samples/index.md)

---
layout: default
title: Quickstart
parent: Durable Functions
nav_order: 2
permalink: /docs/durable-functions/quickstart/
---

# Quickstart: Create Your First Durable Function
{: .no_toc }

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

Build your first Durable Functions orchestration in under 10 minutes.
{: .fs-6 .fw-300 }

---

## What You'll Build

A simple "Hello World" orchestration that calls activities in sequence:

```
Start → SayHello("Tokyo") → SayHello("Seattle") → SayHello("London") → Return Results
```

---

## Prerequisites

- [Visual Studio Code](https://code.visualstudio.com/)
- [Azure Functions Extension](https://marketplace.visualstudio.com/items?itemName=ms-azuretools.vscode-azurefunctions)
- [Azure Functions Core Tools](https://learn.microsoft.com/azure/azure-functions/functions-run-local) (v4.x)
- [.NET 8.0 SDK](https://dotnet.microsoft.com/download) (for C# projects)
- [Docker Desktop](https://www.docker.com/products/docker-desktop/) (for the local emulator)

---

## Step 1: Create a New Project

Open VS Code and use the Azure Functions extension:

1. Press `F1` to open the command palette
2. Search for **Azure Functions: Create New Project...**
3. Select a folder for your project
4. Choose **C#** as the language
5. Select **.NET 8.0 Isolated** as the runtime
6. Select **Durable Functions Orchestration** as the template
7. Name your function `HelloOrchestration`

This creates three functions automatically:
- **HttpStart** — HTTP trigger that starts the orchestration
- **HelloOrchestration** — The orchestrator function
- **SayHello** — The activity function

---

## Step 2: Review the Generated Code

Open the generated `HelloOrchestration.cs` file:

```csharp
using Microsoft.Azure.Functions.Worker;
using Microsoft.Azure.Functions.Worker.Http;
using Microsoft.DurableTask;
using Microsoft.DurableTask.Client;
using Microsoft.Extensions.Logging;

namespace MyDurableApp
{
    public static class HelloOrchestration
    {
        [Function(nameof(HelloOrchestration))]
        public static async Task<List<string>> RunOrchestrator(
            [OrchestrationTrigger] TaskOrchestrationContext context)
        {
            ILogger logger = context.CreateReplaySafeLogger(nameof(HelloOrchestration));
            logger.LogInformation("Starting orchestration.");
            
            var outputs = new List<string>();

            // Call activities in sequence
            outputs.Add(await context.CallActivityAsync<string>(nameof(SayHello), "Tokyo"));
            outputs.Add(await context.CallActivityAsync<string>(nameof(SayHello), "Seattle"));
            outputs.Add(await context.CallActivityAsync<string>(nameof(SayHello), "London"));

            return outputs;
        }

        [Function(nameof(SayHello))]
        public static string SayHello([ActivityTrigger] string name, FunctionContext executionContext)
        {
            ILogger logger = executionContext.GetLogger("SayHello");
            logger.LogInformation("Saying hello to {name}.", name);
            return $"Hello {name}!";
        }

        [Function("HelloOrchestration_HttpStart")]
        public static async Task<HttpResponseData> HttpStart(
            [HttpTrigger(AuthorizationLevel.Anonymous, "get", "post")] HttpRequestData req,
            [DurableClient] DurableTaskClient client,
            FunctionContext executionContext)
        {
            ILogger logger = executionContext.GetLogger("HelloOrchestration_HttpStart");

            string instanceId = await client.ScheduleNewOrchestrationInstanceAsync(
                nameof(HelloOrchestration));

            logger.LogInformation("Started orchestration with ID = '{instanceId}'.", instanceId);

            return await client.CreateCheckStatusResponseAsync(req, instanceId);
        }
    }
}
```

---

## Step 3: Start the Local Emulator

The Durable Task Scheduler emulator provides a local development experience:

```bash
# Start the emulator
docker run -d -p 8080:8080 -p 8082:8082 mcr.microsoft.com/dts/dts-emulator:latest
```

The emulator provides:
- **gRPC endpoint** at `http://localhost:8080`
- **Dashboard** at `http://localhost:8082`

---

## Step 4: Configure for Local Development

Update your `local.settings.json`:

```json
{
  "IsEncrypted": false,
  "Values": {
    "AzureWebJobsStorage": "UseDevelopmentStorage=true",
    "FUNCTIONS_WORKER_RUNTIME": "dotnet-isolated",
    "DURABLE_TASK_SCHEDULER_CONNECTION_STRING": "Endpoint=http://localhost:8080;Authentication=None"
  }
}
```

Update your `host.json` to use the Durable Task Scheduler:

```json
{
  "version": "2.0",
  "extensions": {
    "durableTask": {
      "storageProvider": {
        "type": "durabletaskscheduler",
        "connectionStringName": "DURABLE_TASK_SCHEDULER_CONNECTION_STRING"
      }
    }
  }
}
```

---

## Step 5: Run the Function

1. Press `F5` to start debugging
2. The terminal shows the function URL:
   ```
   HelloOrchestration_HttpStart: [GET,POST] http://localhost:7071/api/HelloOrchestration_HttpStart
   ```

---

## Step 6: Test Your Orchestration

### Start the Orchestration

Open a browser or use `curl`:

```bash
curl http://localhost:7071/api/HelloOrchestration_HttpStart
```

You'll receive a response with status check URLs:

```json
{
  "id": "abc123",
  "statusQueryGetUri": "http://localhost:7071/runtime/webhooks/durabletask/instances/abc123",
  "sendEventPostUri": "http://localhost:7071/runtime/webhooks/durabletask/instances/abc123/raiseEvent/{eventName}",
  "terminatePostUri": "http://localhost:7071/runtime/webhooks/durabletask/instances/abc123/terminate",
  "purgeHistoryDeleteUri": "http://localhost:7071/runtime/webhooks/durabletask/instances/abc123"
}
```

### Check the Status

Visit the `statusQueryGetUri` to see the result:

```json
{
  "name": "HelloOrchestration",
  "instanceId": "abc123",
  "runtimeStatus": "Completed",
  "input": null,
  "output": ["Hello Tokyo!", "Hello Seattle!", "Hello London!"],
  "createdTime": "2025-11-30T10:00:00Z",
  "lastUpdatedTime": "2025-11-30T10:00:02Z"
}
```

### View in Dashboard

Open [http://localhost:8082](http://localhost:8082) to see your orchestration in the dashboard.

---

## 🎉 Congratulations!

You've created your first Durable Function! The orchestration:
- Started via HTTP trigger
- Called three activities in sequence
- Returned the combined results

---

## Next Steps

- [Understand the Programming Model →](programming-model.md)
- [Learn about Orchestrator Concepts →](../concepts/orchestrators.md)
- [Explore Orchestration Patterns →](../patterns/index.md)
- [Deploy to Azure →](../architecture/durable-functions-dts.md)

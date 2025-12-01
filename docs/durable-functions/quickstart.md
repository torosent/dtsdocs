---
layout: default
title: Quickstart
parent: Durable Functions
nav_order: 2
---

# Quickstart: Create your first Durable Function

This guide walks you through creating a "Hello World" Durable Function using Visual Studio Code.

## Prerequisites

*   [Visual Studio Code](https://code.visualstudio.com/)
*   [Azure Functions Extension](https://marketplace.visualstudio.com/items?itemName=ms-azuretools.vscode-azurefunctions)
*   [Azure Functions Core Tools](https://learn.microsoft.com/azure/azure-functions/functions-run-local)

## Create a Local Project

1.  Open VS Code.
2.  Press `F1` to open the command palette.
3.  Search for **Azure Functions: Create New Project...**.
4.  Select a folder for your project.
5.  Select a language (e.g., **C#**, **Python**, **JavaScript**, **TypeScript**, **Java**, **PowerShell**).
6.  Select a template. For this guide, we'll start with an empty project and add functions later, or select **Durable Functions Orchestration** if available.

## Add a Durable Function

1.  Press `F1` and search for **Azure Functions: Create Function...**.
2.  Select **Durable Functions Orchestration**.
3.  Provide a function name (e.g., `HelloOrchestration`).

This will generate three functions:
*   **Orchestrator Client**: An HTTP-triggered function that starts the orchestration.
*   **Orchestrator**: The function that manages the workflow logic.
*   **Activity**: The function that performs the actual work (e.g., saying "Hello").

## Code Example (C#)

```csharp
using Microsoft.Azure.Functions.Worker;
using Microsoft.Azure.Functions.Worker.Http;
using Microsoft.DurableTask;
using Microsoft.DurableTask.Client;
using Microsoft.Extensions.Logging;

public class HelloOrchestration
{
    [Function(nameof(HelloOrchestration))]
    public static async Task<List<string>> RunOrchestrator(
        [OrchestrationTrigger] TaskOrchestrationContext context)
    {
        var outputs = new List<string>();

        outputs.Add(await context.CallActivityAsync<string>(nameof(SayHello), "Tokyo"));
        outputs.Add(await context.CallActivityAsync<string>(nameof(SayHello), "Seattle"));
        outputs.Add(await context.CallActivityAsync<string>(nameof(SayHello), "London"));

        return outputs;
    }

    [Function(nameof(SayHello))]
    public static string SayHello([ActivityTrigger] string name, FunctionContext executionContext)
    {
        var logger = executionContext.GetLogger("SayHello");
        logger.LogInformation("Saying hello to {name}.", name);
        return $"Hello {name}!";
    }

    [Function("HelloOrchestration_HttpStart")]
    public static async Task<HttpResponseData> HttpStart(
        [HttpTrigger(AuthorizationLevel.Anonymous, "get", "post")] HttpRequestData req,
        [DurableClient] DurableTaskClient client,
        FunctionContext executionContext)
    {
        var logger = executionContext.GetLogger("HelloOrchestration_HttpStart");

        string instanceId = await client.ScheduleNewOrchestrationInstanceAsync(
            nameof(HelloOrchestration));

        logger.LogInformation("Started orchestration with ID = '{instanceId}'.", instanceId);

        return client.CreateCheckStatusResponse(req, instanceId);
    }
}
```

## Run Locally

1.  Press `F5` to start debugging.
2.  The terminal will show the URL for your HTTP starter function (e.g., `http://localhost:7071/api/HelloOrchestration_HttpStart`).
3.  Open this URL in a browser or use a tool like `curl` or Postman.
4.  You will receive a response containing status query URLs.
5.  Visit the `statusQueryGetUri` to see the output of your orchestration.

## Next Steps

*   [Learn about the Programming Model](programming-model.md)
*   [Explore Orchestrator Concepts](../concepts/orchestrators.md)

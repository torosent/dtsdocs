---
layout: default
title: Quickstart
parent: Durable Task SDKs
nav_order: 2
---

# Quickstart: Use the Durable Task SDKs

This guide demonstrates how to use the Durable Task SDKs (Microsoft.DurableTask) to build workflows that can run independently of Azure Functions, such as in a console app or a container.

## Prerequisites

*   .NET 6.0 or later (for the .NET SDK)
*   A running sidecar (e.g., Azure Storage Emulator or a real Azure Storage account if using the Netherite/MSSQL backend, though typically the SDK connects to a backend via a sidecar or direct connection depending on the language).
*   *Note: The Durable Task SDKs often work in conjunction with a backend provider. For simplicity, this guide assumes a basic setup.*

## .NET Console App Example

1.  Create a new console application:
    ```bash
    dotnet new console -n DurableTaskQuickstart
    cd DurableTaskQuickstart
    ```

2.  Add the NuGet package:
    ```bash
    dotnet add package Microsoft.DurableTask.Worker
    ```

3.  Update `Program.cs`:

```csharp
using Microsoft.DurableTask;
using Microsoft.DurableTask.Worker;
using Microsoft.Extensions.Hosting;
using Microsoft.Extensions.Logging;

// Define the orchestration
class HelloOrchestration : TaskOrchestrator<string, List<string>>
{
    public override async Task<List<string>> RunAsync(TaskOrchestrationContext context, string input)
    {
        var outputs = new List<string>();

        outputs.Add(await context.CallActivityAsync<string>(nameof(SayHello), "Tokyo"));
        outputs.Add(await context.CallActivityAsync<string>(nameof(SayHello), "Seattle"));
        outputs.Add(await context.CallActivityAsync<string>(nameof(SayHello), "London"));

        return outputs;
    }
}

// Define the activity
class SayHello : TaskActivity<string, string>
{
    public override Task<string> RunAsync(TaskActivityContext context, string input)
    {
        return Task.FromResult($"Hello {input}!");
    }
}

// Setup and run the worker
IHost host = Host.CreateDefaultBuilder(args)
    .ConfigureLogging(logging => logging.AddConsole())
    .ConfigureServices(services =>
    {
        services.AddDurableTaskWorker(builder =>
        {
            // Use a storage provider (e.g., Azure Storage)
            // For this example, we assume a local emulator or connection string is configured
            // builder.UseAzureStorage(...); 
            
            builder.AddOrchestrator<HelloOrchestration>();
            builder.AddActivity<SayHello>();
        });
    })
    .Build();

await host.RunAsync();
```

## Running the Worker

1.  Run the application:
    ```bash
    dotnet run
    ```
2.  The worker will start polling for tasks. You will need a separate client to schedule an orchestration instance.

## Scheduling an Instance (Client)

You can use the `DurableTaskClient` to schedule instances.

```csharp
// In a separate client app or part of the same host
var client = host.Services.GetRequiredService<DurableTaskClient>();
string instanceId = await client.ScheduleNewOrchestrationInstanceAsync(
    nameof(HelloOrchestration), 
    "World");
Console.WriteLine($"Started instance: {instanceId}");
```

## Next Steps

*   [Explore the .NET SDK](dotnet.md)
*   [Explore the Java SDK](java.md)
*   [Explore the Python SDK](python.md)

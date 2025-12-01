---
layout: default
title: Quickstart
parent: Durable Task SDKs
nav_order: 2
permalink: /docs/sdks/quickstart/
---

# Quickstart: Build a Portable Durable Worker
{: .no_toc }

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

Build your first portable durable orchestration in under 10 minutes.
{: .fs-6 .fw-300 }

---

## What You'll Build

A console application that runs a "Hello World" orchestration, independent of Azure Functions:

```
Worker starts → Client schedules orchestration → Worker executes → Results returned
```

---

## Prerequisites

- [.NET 8.0 SDK](https://dotnet.microsoft.com/download) (for .NET example)
- [Docker Desktop](https://www.docker.com/products/docker-desktop/) (for the local emulator)

---

## Step 1: Start the Local Emulator

```bash
docker run -d -p 8080:8080 -p 8082:8082 mcr.microsoft.com/dts/dts-emulator:latest
```

The emulator provides:
- **gRPC endpoint** at `http://localhost:8080`  
- **Dashboard** at `http://localhost:8082`

---

## Step 2: Create a New Project

```bash
# Create a new console app
dotnet new console -n DurableWorkerQuickstart
cd DurableWorkerQuickstart

# Add the required packages
dotnet add package Microsoft.DurableTask.Worker.AzureManaged
dotnet add package Microsoft.DurableTask.Client.AzureManaged
dotnet add package Microsoft.Extensions.Hosting
```

---

## Step 3: Define Your Orchestration

Replace the contents of `Program.cs`:

```csharp
using Microsoft.DurableTask;
using Microsoft.DurableTask.Worker;
using Microsoft.DurableTask.Client;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;
using Microsoft.Extensions.Logging;

// Connection settings for local emulator
var connectionString = "Endpoint=http://localhost:8080;Authentication=None";
var taskHub = "default";

// Build the host
var builder = Host.CreateApplicationBuilder(args);

// Configure logging
builder.Logging.AddConsole();

// Add the Durable Task Worker
builder.Services.AddDurableTaskWorker(options =>
{
    options.AddOrchestrator<HelloCitiesOrchestrator>();
    options.AddActivity<SayHelloActivity>();
})
.UseDurableTaskScheduler(connectionString, taskHub);

// Add the Durable Task Client
builder.Services.AddDurableTaskClient()
    .UseDurableTaskScheduler(connectionString, taskHub);

// Build and start
var host = builder.Build();

// Start an orchestration
_ = Task.Run(async () =>
{
    // Wait for worker to initialize
    await Task.Delay(2000);
    
    var client = host.Services.GetRequiredService<DurableTaskClient>();
    
    Console.WriteLine("Starting orchestration...");
    var instanceId = await client.ScheduleNewOrchestrationInstanceAsync(
        nameof(HelloCitiesOrchestrator)
    );
    Console.WriteLine($"Started orchestration with ID: {instanceId}");
    
    // Wait for completion
    var metadata = await client.WaitForInstanceCompletionAsync(
        instanceId,
        getInputsAndOutputs: true
    );
    
    Console.WriteLine($"Status: {metadata.RuntimeStatus}");
    Console.WriteLine($"Output: {metadata.SerializedOutput}");
});

await host.RunAsync();

// Orchestrator - coordinates the workflow
[DurableTask(nameof(HelloCitiesOrchestrator))]
public class HelloCitiesOrchestrator : TaskOrchestrator<string?, string>
{
    public override async Task<string> RunAsync(
        TaskOrchestrationContext context, 
        string? input)
    {
        var results = new List<string>();
        
        results.Add(await context.CallActivityAsync<string>(nameof(SayHelloActivity), "Tokyo"));
        results.Add(await context.CallActivityAsync<string>(nameof(SayHelloActivity), "London"));
        results.Add(await context.CallActivityAsync<string>(nameof(SayHelloActivity), "Seattle"));
        
        return string.Join(" ", results);
    }
}

// Activity - does the actual work
[DurableTask(nameof(SayHelloActivity))]
public class SayHelloActivity : TaskActivity<string, string>
{
    private readonly ILogger<SayHelloActivity> _logger;
    
    public SayHelloActivity(ILogger<SayHelloActivity> logger)
    {
        _logger = logger;
    }
    
    public override Task<string> RunAsync(TaskActivityContext context, string city)
    {
        _logger.LogInformation("Saying hello to {City}", city);
        return Task.FromResult($"Hello, {city}!");
    }
}
```

---

## Step 4: Run the Application

```bash
dotnet run
```

You should see output like:

```
Starting orchestration...
Started orchestration with ID: abc123
info: SayHelloActivity[0]
      Saying hello to Tokyo
info: SayHelloActivity[0]
      Saying hello to London
info: SayHelloActivity[0]
      Saying hello to Seattle
Status: Completed
Output: "Hello, Tokyo! Hello, London! Hello, Seattle!"
```

---

## Step 5: View in Dashboard

Open [http://localhost:8082](http://localhost:8082) to see your orchestration in the dashboard:

- View the orchestration status
- Inspect the execution timeline
- See input/output values

---

## 🎉 Congratulations!

You've built your first portable Durable Task application! This same code can run on:
- Azure Container Apps
- Azure Kubernetes Service (AKS)
- Virtual Machines
- On-premises servers

---

## Python Example

For Python developers:

```bash
pip install durabletask-azure
```

```python
import asyncio
from durabletask import task
from durabletask.azuremanaged.worker import DurableTaskSchedulerWorker, DurableTaskSchedulerWorkerOptions
from durabletask.azuremanaged.client import DurableTaskSchedulerClient

@task.orchestrator
def hello_cities(ctx: task.OrchestrationContext, _: None):
    results = []
    results.append(yield ctx.call_activity("say_hello", input="Tokyo"))
    results.append(yield ctx.call_activity("say_hello", input="London"))
    results.append(yield ctx.call_activity("say_hello", input="Seattle"))
    return " ".join(results)

@task.activity
def say_hello(ctx: task.ActivityContext, city: str) -> str:
    print(f"Saying hello to {city}")
    return f"Hello, {city}!"

async def main():
    options = DurableTaskSchedulerWorkerOptions(
        host="localhost:8080",
        secure_channel=False,
        taskhub="default"
    )
    
    async with DurableTaskSchedulerWorker(
        options=options,
        orchestrators=[hello_cities],
        activities=[say_hello]
    ) as worker:
        await worker.start()
        
        client = DurableTaskSchedulerClient(
            host="localhost:8080",
            secure_channel=False,
            taskhub="default"
        )
        
        instance_id = await client.schedule_new_orchestration(hello_cities)
        print(f"Started: {instance_id}")
        
        result = await client.wait_for_orchestration_completion(instance_id)
        print(f"Result: {result.serialized_output}")

asyncio.run(main())
```

---

## Next Steps

- [Explore the .NET SDK →](./dotnet.md)
- [Explore the Python SDK →](./python.md)  
- [Explore the Java SDK →](./java.md)
- [Deploy to Azure Container Apps →](../architecture/aca-dts.md)
- [Explore Orchestration Patterns →](../patterns/index.md)

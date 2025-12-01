---
layout: default
title: Samples & Examples
nav_order: 9
has_children: false
permalink: /docs/samples/
---

# Samples & Examples
{: .no_toc }

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

Practical code samples for common durable orchestration scenarios.

---

## Sample Categories

| Category | Description |
|----------|-------------|
| [Quick Starts](#quick-starts) | Minimal examples to get started |
| [Patterns](#pattern-examples) | Implementation of common patterns |
| [Real-World Scenarios](#real-world-scenarios) | Production-like examples |
| [Language-Specific](#language-specific) | Per-language samples |

---

## Quick Starts

### Hello World (.NET)

```csharp
using Microsoft.DurableTask;
using Microsoft.DurableTask.Worker;
using Microsoft.DurableTask.Client;
using Microsoft.Extensions.Hosting;

var builder = Host.CreateApplicationBuilder(args);

builder.Services.AddDurableTaskWorker(options =>
{
    options.AddOrchestrator<HelloOrchestrator>();
    options.AddActivity<SayHelloActivity>();
})
.UseDurableTaskScheduler("Endpoint=http://localhost:8080;Authentication=None", "default");

builder.Services.AddDurableTaskClient()
    .UseDurableTaskScheduler("Endpoint=http://localhost:8080;Authentication=None", "default");

var host = builder.Build();

// Start orchestration
var client = host.Services.GetRequiredService<DurableTaskClient>();
var instanceId = await client.ScheduleNewOrchestrationInstanceAsync("HelloOrchestrator", "World");
Console.WriteLine($"Started: {instanceId}");

// Wait for completion
var result = await client.WaitForInstanceCompletionAsync(instanceId);
Console.WriteLine($"Result: {result.ReadOutputAs<string>()}");

await host.RunAsync();

// Orchestrator
[DurableTask("HelloOrchestrator")]
public class HelloOrchestrator : TaskOrchestrator<string, string>
{
    public override async Task<string> RunAsync(TaskOrchestrationContext context, string name)
    {
        return await context.CallActivityAsync<string>("SayHello", name);
    }
}

// Activity
[DurableTask("SayHello")]
public class SayHelloActivity : TaskActivity<string, string>
{
    public override Task<string> RunAsync(TaskActivityContext context, string name)
    {
        return Task.FromResult($"Hello, {name}!");
    }
}
```

### Hello World (Python)

```python
import asyncio
from durabletask import task
from durabletask.azuremanaged.worker import DurableTaskSchedulerWorker, DurableTaskSchedulerWorkerOptions
from durabletask.azuremanaged.client import DurableTaskSchedulerClient

@task.orchestrator
def hello_orchestrator(ctx: task.OrchestrationContext, name: str):
    result = yield ctx.call_activity("say_hello", input=name)
    return result

@task.activity
def say_hello(ctx: task.ActivityContext, name: str) -> str:
    return f"Hello, {name}!"

async def main():
    options = DurableTaskSchedulerWorkerOptions(
        host="localhost:8080",
        secure_channel=False,
        taskhub="default"
    )
    
    async with DurableTaskSchedulerWorker(
        options=options,
        orchestrators=[hello_orchestrator],
        activities=[say_hello]
    ) as worker:
        await worker.start()
        
        client = DurableTaskSchedulerClient(
            host="localhost:8080",
            secure_channel=False,
            taskhub="default"
        )
        
        instance_id = await client.schedule_new_orchestration(
            orchestrator=hello_orchestrator,
            input="World"
        )
        
        result = await client.wait_for_orchestration_completion(instance_id)
        print(f"Result: {result.serialized_output}")

asyncio.run(main())
```

---

## Pattern Examples

### Order Processing (Function Chaining)

```csharp
[DurableTask(nameof(OrderProcessingOrchestrator))]
public class OrderProcessingOrchestrator : TaskOrchestrator<Order, OrderResult>
{
    public override async Task<OrderResult> RunAsync(
        TaskOrchestrationContext context, 
        Order order)
    {
        // Sequential steps
        var validated = await context.CallActivityAsync<bool>("ValidateOrder", order);
        if (!validated) return new OrderResult(false, "Validation failed");
        
        var reserved = await context.CallActivityAsync<Reservation>("ReserveInventory", order);
        var payment = await context.CallActivityAsync<Payment>("ProcessPayment", order);
        var shipment = await context.CallActivityAsync<Shipment>("CreateShipment", order);
        
        await context.CallActivityAsync("SendConfirmation", new {
            Order = order,
            Reservation = reserved,
            Payment = payment,
            Shipment = shipment
        });
        
        return new OrderResult(true, shipment.TrackingNumber);
    }
}
```

### Image Processing (Fan-Out/Fan-In)

```python
@task.orchestrator
def image_processing_orchestrator(ctx: task.OrchestrationContext, images: list):
    # Fan-out: Process all images in parallel
    tasks = []
    for image in images:
        t = ctx.call_activity("process_image", input=image)
        tasks.append(t)
    
    # Fan-in: Wait for all
    processed = yield task.when_all(tasks)
    
    # Create thumbnail gallery
    gallery = yield ctx.call_activity("create_gallery", input=processed)
    
    return {"gallery_url": gallery["url"], "count": len(processed)}

@task.activity
def process_image(ctx: task.ActivityContext, image: dict) -> dict:
    # Resize, compress, add watermark
    return {
        "original": image["url"],
        "thumbnail": f"{image['url']}_thumb",
        "processed": True
    }
```

### Document Approval (Human Interaction)

```csharp
[DurableTask(nameof(DocumentApprovalOrchestrator))]
public class DocumentApprovalOrchestrator : TaskOrchestrator<Document, ApprovalResult>
{
    public override async Task<ApprovalResult> RunAsync(
        TaskOrchestrationContext context, 
        Document document)
    {
        // Notify approver
        await context.CallActivityAsync("SendApprovalRequest", new {
            Document = document,
            ApprovalUrl = $"https://app.example.com/approve/{context.InstanceId}"
        });
        
        // Wait for approval with 7-day timeout
        using var cts = new CancellationTokenSource();
        var approvalTask = context.WaitForExternalEvent<ApprovalResponse>("Approved");
        var timeoutTask = context.CreateTimer(
            context.CurrentUtcDateTime.AddDays(7), 
            cts.Token
        );
        
        var winner = await Task.WhenAny(approvalTask, timeoutTask);
        
        if (winner == approvalTask)
        {
            cts.Cancel();
            var response = await approvalTask;
            
            if (response.IsApproved)
            {
                await context.CallActivityAsync("PublishDocument", document);
                return new ApprovalResult(true, "Published");
            }
            return new ApprovalResult(false, response.Reason);
        }
        
        return new ApprovalResult(false, "Approval timeout");
    }
}
```

---

## Real-World Scenarios

### E-Commerce Order Saga

Complete order processing with compensation:

```csharp
[DurableTask(nameof(OrderSagaOrchestrator))]
public class OrderSagaOrchestrator : TaskOrchestrator<Order, OrderResult>
{
    public override async Task<OrderResult> RunAsync(
        TaskOrchestrationContext context, 
        Order order)
    {
        var completedSteps = new List<string>();
        
        try
        {
            // Step 1: Reserve inventory
            await context.CallActivityAsync("ReserveInventory", order);
            completedSteps.Add("inventory");
            
            // Step 2: Process payment
            await context.CallActivityAsync("ProcessPayment", order);
            completedSteps.Add("payment");
            
            // Step 3: Create shipment
            var shipment = await context.CallActivityAsync<Shipment>("CreateShipment", order);
            completedSteps.Add("shipment");
            
            // Step 4: Send notifications
            await context.CallActivityAsync("SendOrderConfirmation", order);
            
            return new OrderResult(true, shipment.TrackingNumber);
        }
        catch (Exception ex)
        {
            // Compensate in reverse order
            foreach (var step in completedSteps.AsEnumerable().Reverse())
            {
                await CompensateStep(context, step, order);
            }
            
            return new OrderResult(false, ex.Message);
        }
    }
    
    private async Task CompensateStep(
        TaskOrchestrationContext context, 
        string step, 
        Order order)
    {
        switch (step)
        {
            case "shipment":
                await context.CallActivityAsync("CancelShipment", order);
                break;
            case "payment":
                await context.CallActivityAsync("RefundPayment", order);
                break;
            case "inventory":
                await context.CallActivityAsync("ReleaseInventory", order);
                break;
        }
    }
}
```

### Video Transcoding Pipeline

```python
@task.orchestrator
def video_pipeline_orchestrator(ctx: task.OrchestrationContext, video: dict):
    # Step 1: Validate and analyze
    analysis = yield ctx.call_activity("analyze_video", input=video)
    
    # Step 2: Generate multiple resolutions in parallel
    resolutions = ["1080p", "720p", "480p", "360p"]
    transcode_tasks = []
    
    for res in resolutions:
        t = ctx.call_activity("transcode", input={
            "video": video,
            "resolution": res
        })
        transcode_tasks.append(t)
    
    transcoded = yield task.when_all(transcode_tasks)
    
    # Step 3: Generate thumbnails
    thumbnails = yield ctx.call_activity("generate_thumbnails", input={
        "video": video,
        "count": 10
    })
    
    # Step 4: Create manifest
    manifest = yield ctx.call_activity("create_streaming_manifest", input={
        "videos": transcoded,
        "thumbnails": thumbnails
    })
    
    # Step 5: Publish
    publish_result = yield ctx.call_activity("publish_to_cdn", input=manifest)
    
    return {
        "original": video["id"],
        "manifest_url": publish_result["url"],
        "resolutions": [t["resolution"] for t in transcoded],
        "thumbnail_count": len(thumbnails)
    }
```

### IoT Device Provisioning

```csharp
[DurableTask(nameof(DeviceProvisioningOrchestrator))]
public class DeviceProvisioningOrchestrator : TaskOrchestrator<DeviceRequest, ProvisioningResult>
{
    public override async Task<ProvisioningResult> RunAsync(
        TaskOrchestrationContext context, 
        DeviceRequest request)
    {
        // Register device in IoT Hub
        var registration = await context.CallActivityAsync<DeviceRegistration>(
            "RegisterDevice", 
            request
        );
        
        // Generate certificates
        var certificates = await context.CallActivityAsync<Certificates>(
            "GenerateCertificates",
            registration.DeviceId
        );
        
        // Wait for device to connect (with timeout)
        context.SetCustomStatus(new { stage = "WaitingForConnection" });
        
        var connectionTask = context.WaitForExternalEvent<DeviceConnection>("DeviceConnected");
        var timeoutTask = context.CreateTimer(
            context.CurrentUtcDateTime.AddMinutes(30),
            CancellationToken.None
        );
        
        var winner = await Task.WhenAny(connectionTask, timeoutTask);
        
        if (winner == timeoutTask)
        {
            // Cleanup on timeout
            await context.CallActivityAsync("RevokeDevice", registration.DeviceId);
            return new ProvisioningResult(false, "Connection timeout");
        }
        
        // Configure device
        await context.CallActivityAsync("ConfigureDevice", new {
            DeviceId = registration.DeviceId,
            Configuration = request.Configuration
        });
        
        // Run diagnostics
        var diagnostics = await context.CallActivityAsync<DiagnosticResult>(
            "RunDiagnostics",
            registration.DeviceId
        );
        
        if (!diagnostics.Passed)
        {
            return new ProvisioningResult(false, $"Diagnostics failed: {diagnostics.Message}");
        }
        
        return new ProvisioningResult(true, registration.DeviceId);
    }
}
```

---

## Language-Specific

### .NET Samples

| Sample | Description |
|--------|-------------|
| [Console App](csharp/console-app.md) | Basic console application |
| [ASP.NET Core API](csharp/aspnet-api.md) | Web API with orchestrations |
| [Background Worker](csharp/background-worker.md) | Long-running worker service |

### Python Samples

| Sample | Description |
|--------|-------------|
| [Basic Worker](python/basic-worker.md) | Simple Python worker |
| [FastAPI Integration](python/fastapi.md) | FastAPI with orchestrations |
| [Data Processing](python/data-processing.md) | Data pipeline example |

### Java Samples

| Sample | Description |
|--------|-------------|
| [Spring Boot](java/spring-boot.md) | Spring Boot integration |
| [Batch Processing](java/batch.md) | Batch job orchestration |

---

## Next Steps

- [Explore Patterns →](../patterns/index.md)
- [View Architecture Guides →](../architecture/index.md)
- [Read the Concepts →](../concepts/index.md)

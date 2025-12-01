---
layout: default
title: .NET SDK
parent: Durable Task SDKs
nav_order: 2
permalink: /docs/sdks/dotnet/
---

# .NET Durable Task SDK
{: .no_toc }

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

The .NET Durable Task SDK provides a modern, type-safe way to build durable orchestrations that can run on any .NET platform.

---

## Installation

### NuGet Packages

```bash
# Core packages for Durable Task Scheduler
dotnet add package Microsoft.DurableTask.Worker.AzureManaged
dotnet add package Microsoft.DurableTask.Client.AzureManaged

# Optional: Source generators for type-safe APIs
dotnet add package Microsoft.DurableTask.Generators
```

### Package Reference

```xml
<ItemGroup>
  <PackageReference Include="Microsoft.DurableTask.Worker.AzureManaged" Version="1.17.1" />
  <PackageReference Include="Microsoft.DurableTask.Client.AzureManaged" Version="1.17.1" />
  <PackageReference Include="Microsoft.DurableTask.Generators" Version="1.0.0-preview.1" 
                    OutputItemType="Analyzer" />
</ItemGroup>
```

---

## Basic Setup

### Host Configuration

```csharp
using Microsoft.Extensions.Hosting;
using Microsoft.DurableTask.Worker;
using Microsoft.DurableTask.Client;

var builder = Host.CreateApplicationBuilder(args);

// Get connection settings
var connectionString = Environment.GetEnvironmentVariable("DTS_CONNECTION_STRING") 
    ?? "Endpoint=http://localhost:8080;Authentication=None";
var taskHub = Environment.GetEnvironmentVariable("TASKHUB_NAME") ?? "default";

// Configure the Durable Task Worker
builder.Services.AddDurableTaskWorker(options =>
{
    options.AddOrchestrator<MyOrchestrator>();
    options.AddActivity<MyActivity>();
})
.UseDurableTaskScheduler(connectionString, taskHub);

// Configure the Durable Task Client
builder.Services.AddDurableTaskClient()
    .UseDurableTaskScheduler(connectionString, taskHub);

var host = builder.Build();
await host.RunAsync();
```

---

## Defining Orchestrators

### Class-Based Syntax

```csharp
using Microsoft.DurableTask;

[DurableTask(nameof(OrderProcessingOrchestrator))]
public class OrderProcessingOrchestrator : TaskOrchestrator<Order, OrderResult>
{
    public override async Task<OrderResult> RunAsync(
        TaskOrchestrationContext context, 
        Order input)
    {
        // Step 1: Validate
        var isValid = await context.CallActivityAsync<bool>(
            nameof(ValidateOrderActivity), 
            input
        );
        
        if (!isValid)
        {
            return new OrderResult { Success = false, Message = "Invalid order" };
        }
        
        // Step 2: Process payment
        var payment = await context.CallActivityAsync<PaymentResult>(
            nameof(ProcessPaymentActivity),
            input.Payment
        );
        
        // Step 3: Ship
        await context.CallActivityAsync(
            nameof(ShipOrderActivity),
            input
        );
        
        return new OrderResult 
        { 
            Success = true, 
            OrderId = input.Id,
            TransactionId = payment.TransactionId
        };
    }
}
```

### Function-Based Syntax

```csharp
using Microsoft.DurableTask;

public static class Orchestrators
{
    public static async Task<string> HelloCities(TaskOrchestrationContext context, string input)
    {
        var results = new List<string>();
        
        results.Add(await context.CallActivityAsync<string>("SayHello", "Tokyo"));
        results.Add(await context.CallActivityAsync<string>("SayHello", "London"));
        results.Add(await context.CallActivityAsync<string>("SayHello", "Seattle"));
        
        return string.Join(" ", results);
    }
}

// Register in startup
builder.Services.AddDurableTaskWorker(options =>
{
    options.AddOrchestrator("HelloCities", Orchestrators.HelloCities);
    options.AddActivity("SayHello", Activities.SayHello);
});
```

---

## Defining Activities

### Class-Based Syntax

```csharp
using Microsoft.DurableTask;

[DurableTask(nameof(ValidateOrderActivity))]
public class ValidateOrderActivity : TaskActivity<Order, bool>
{
    private readonly ILogger<ValidateOrderActivity> _logger;
    private readonly IOrderValidator _validator;
    
    public ValidateOrderActivity(
        ILogger<ValidateOrderActivity> logger,
        IOrderValidator validator)
    {
        _logger = logger;
        _validator = validator;
    }
    
    public override async Task<bool> RunAsync(
        TaskActivityContext context, 
        Order input)
    {
        _logger.LogInformation("Validating order {OrderId}", input.Id);
        return await _validator.ValidateAsync(input);
    }
}
```

### Function-Based Syntax

```csharp
public static class Activities
{
    public static string SayHello(TaskActivityContext context, string cityName)
    {
        return $"Hello, {cityName}!";
    }
}

// Register in startup
options.AddActivity("SayHello", Activities.SayHello);
```

---

## Using the Client

### Schedule an Orchestration

```csharp
using Microsoft.DurableTask.Client;

public class OrderController : ControllerBase
{
    private readonly DurableTaskClient _client;
    
    public OrderController(DurableTaskClient client)
    {
        _client = client;
    }
    
    [HttpPost]
    public async Task<IActionResult> CreateOrder([FromBody] Order order)
    {
        // Schedule the orchestration
        var instanceId = await _client.ScheduleNewOrchestrationInstanceAsync(
            nameof(OrderProcessingOrchestrator),
            order
        );
        
        return Accepted(new { instanceId });
    }
}
```

### Wait for Completion

```csharp
// Schedule and wait
var instanceId = await _client.ScheduleNewOrchestrationInstanceAsync(
    nameof(OrderProcessingOrchestrator),
    order
);

// Wait for completion with timeout
var metadata = await _client.WaitForInstanceCompletionAsync(
    instanceId,
    cancellationToken,
    getInputsAndOutputs: true
);

if (metadata.RuntimeStatus == OrchestrationRuntimeStatus.Completed)
{
    var result = metadata.ReadOutputAs<OrderResult>();
    return Ok(result);
}
```

### Query Status

```csharp
// Get orchestration status
var metadata = await _client.GetInstanceAsync(instanceId, getInputsAndOutputs: true);

Console.WriteLine($"Status: {metadata.RuntimeStatus}");
Console.WriteLine($"Created: {metadata.CreatedAt}");
Console.WriteLine($"Last Updated: {metadata.LastUpdatedAt}");
```

### Terminate an Orchestration

```csharp
await _client.TerminateInstanceAsync(instanceId, "Cancelled by user");
```

### Raise an Event

```csharp
await _client.RaiseEventAsync(instanceId, "ApprovalReceived", new { 
    Approved = true, 
    ApprovedBy = "manager@example.com" 
});
```

---

## Advanced Patterns

### Fan-Out/Fan-In

```csharp
public class FanOutOrchestrator : TaskOrchestrator<List<WorkItem>, List<WorkResult>>
{
    public override async Task<List<WorkResult>> RunAsync(
        TaskOrchestrationContext context, 
        List<WorkItem> workItems)
    {
        // Fan out - schedule all tasks
        var tasks = workItems.Select(item =>
            context.CallActivityAsync<WorkResult>(nameof(ProcessWorkItem), item)
        ).ToList();
        
        // Fan in - wait for all to complete
        var results = await Task.WhenAll(tasks);
        
        return results.ToList();
    }
}
```

### Sub-Orchestrations

```csharp
public override async Task<OrderResult> RunAsync(
    TaskOrchestrationContext context, 
    Order input)
{
    // Call a sub-orchestration
    var shippingResult = await context.CallSubOrchestratorAsync<ShippingResult>(
        nameof(ShippingOrchestrator),
        new ShippingRequest { OrderId = input.Id, Address = input.ShippingAddress }
    );
    
    return new OrderResult { ShippingTrackingNumber = shippingResult.TrackingNumber };
}
```

### Durable Timers

```csharp
public override async Task<string> RunAsync(
    TaskOrchestrationContext context, 
    Order input)
{
    // Wait for 24 hours
    var deadline = context.CurrentUtcDateTime.AddHours(24);
    await context.CreateTimer(deadline, CancellationToken.None);
    
    // Continue after the delay
    return "Timer completed";
}
```

### External Events

```csharp
public override async Task<ApprovalResult> RunAsync(
    TaskOrchestrationContext context, 
    ApprovalRequest input)
{
    // Send notification
    await context.CallActivityAsync(nameof(SendApprovalRequest), input);
    
    // Wait for approval with timeout
    using var cts = new CancellationTokenSource();
    var approvalTask = context.WaitForExternalEvent<Approval>("ApprovalReceived");
    var timeoutTask = context.CreateTimer(
        context.CurrentUtcDateTime.AddHours(24), 
        cts.Token
    );
    
    var winner = await Task.WhenAny(approvalTask, timeoutTask);
    
    if (winner == approvalTask)
    {
        cts.Cancel();
        var approval = await approvalTask;
        return new ApprovalResult { Approved = approval.Approved };
    }
    else
    {
        return new ApprovalResult { Approved = false, Reason = "Timeout" };
    }
}
```

### Retry Policies

```csharp
var options = new TaskOptions(new TaskRetryOptions(
    new RetryPolicy(
        maxNumberOfAttempts: 5,
        firstRetryInterval: TimeSpan.FromSeconds(1),
        backoffCoefficient: 2.0,
        maxRetryInterval: TimeSpan.FromMinutes(1)
    )
));

var result = await context.CallActivityAsync<string>(
    nameof(UnreliableActivity), 
    input, 
    options
);
```

---

## Type-Safe APIs with Source Generators

When using `Microsoft.DurableTask.Generators`, you get generated extension methods:

```csharp
// Generated extension method
var instanceId = await client.ScheduleNewOrderProcessingOrchestratorInstanceAsync(order);

// Instead of
var instanceId = await client.ScheduleNewOrchestrationInstanceAsync(
    nameof(OrderProcessingOrchestrator), 
    order
);
```

---

## Dependency Injection

### Registering Services

```csharp
builder.Services.AddScoped<IOrderValidator, OrderValidator>();
builder.Services.AddScoped<IPaymentService, PaymentService>();

builder.Services.AddDurableTaskWorker(options =>
{
    // Activities can use dependency injection
    options.AddActivity<ValidateOrderActivity>();
    options.AddActivity<ProcessPaymentActivity>();
});
```

### Activity with DI

```csharp
public class ValidateOrderActivity : TaskActivity<Order, bool>
{
    private readonly IOrderValidator _validator;
    
    // Dependencies are injected
    public ValidateOrderActivity(IOrderValidator validator)
    {
        _validator = validator;
    }
    
    public override async Task<bool> RunAsync(TaskActivityContext context, Order input)
    {
        return await _validator.ValidateAsync(input);
    }
}
```

---

## ASP.NET Core Integration

### WebAPI Example

```csharp
var builder = WebApplication.CreateBuilder(args);

// Add Durable Task services
builder.Services.AddDurableTaskWorker(options =>
{
    options.AddOrchestrator<OrderProcessingOrchestrator>();
    options.AddActivity<ValidateOrderActivity>();
})
.UseDurableTaskScheduler(connectionString, taskHub);

builder.Services.AddDurableTaskClient()
    .UseDurableTaskScheduler(connectionString, taskHub);

var app = builder.Build();

app.MapPost("/orders", async (Order order, DurableTaskClient client) =>
{
    var instanceId = await client.ScheduleNewOrchestrationInstanceAsync(
        nameof(OrderProcessingOrchestrator), 
        order
    );
    
    return Results.Accepted($"/orders/{instanceId}", new { instanceId });
});

app.MapGet("/orders/{instanceId}", async (string instanceId, DurableTaskClient client) =>
{
    var metadata = await client.GetInstanceAsync(instanceId);
    return Results.Ok(new
    {
        instanceId,
        status = metadata.RuntimeStatus.ToString(),
        createdAt = metadata.CreatedAt,
        output = metadata.SerializedOutput
    });
});

app.Run();
```

---

## Next Steps

- [View .NET Samples →](../samples/csharp/index.md)
- [Deploy to Azure Container Apps →](../architecture/aca-dts.md)
- [Explore Patterns →](../patterns/index.md)

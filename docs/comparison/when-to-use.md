---
layout: default
title: When to Use What
parent: Comparison
nav_order: 1
permalink: /docs/comparison/when-to-use/
---

# When to Use What
{: .no_toc }

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

Guidance for choosing between Durable Functions and Durable Task SDKs.

---

## Quick Decision Guide

```
┌─────────────────────────────────────────────────────────────────┐
│                    DECISION FLOWCHART                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  "Do you need serverless with scale-to-zero?"                   │
│      │                                                           │
│      ├── Yes ──► Durable Functions                              │
│      │                                                           │
│      └── No ───► "Do you need to run outside Azure Functions?" │
│                       │                                          │
│                       ├── Yes ──► Durable Task SDKs             │
│                       │           (Container Apps, AKS, VMs)    │
│                       │                                          │
│                       └── No ───► "Do you prefer containers?"   │
│                                       │                          │
│                                       ├── Yes ──► Durable Task  │
│                                       │           SDKs + ACA    │
│                                       │                          │
│                                       └── No ──► Durable        │
│                                                  Functions      │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Feature Comparison

| Feature | Durable Functions | Durable Task SDKs |
|---------|-------------------|-------------------|
| **Compute Platform** | Azure Functions only | Any (.NET, Python, Java apps) |
| **Scale to Zero** | ✅ Yes | ❌ Minimum 1 instance |
| **Trigger Types** | HTTP, Queue, Timer, etc. | Custom (you implement) |
| **Language Support** | C#, JS, Python, Java, PS | .NET, Python, Java (Preview) |
| **Entity Functions** | ✅ Full support | ✅ .NET only |
| **Deployment** | Function App | Container, VM, any host |
| **Cold Start** | Yes (mitigated with Premium) | No (always running) |
| **Pricing Model** | Pay-per-execution | Pay for compute time |

---

## When to Choose Durable Functions

Choose Durable Functions when:

### ✅ Ideal Scenarios

1. **Event-Driven Workloads**
   - React to HTTP requests, queue messages, or events
   - Trigger orchestrations from various sources
   
2. **Sporadic or Unpredictable Load**
   - Scale to zero when idle
   - Pay only for actual executions
   
3. **Existing Azure Functions Investment**
   - Already using Azure Functions
   - Want to add orchestration to existing functions
   
4. **Quick Prototyping**
   - Fast development cycle
   - Built-in triggers and bindings
   
5. **Multi-Language Teams**
   - Need JavaScript/TypeScript, Python, or PowerShell
   - Consistent experience across languages

### Code Example

```csharp
// Durable Functions style - with triggers
public class OrderFunctions
{
    [Function("StartOrder")]
    public async Task<IActionResult> StartOrder(
        [HttpTrigger(AuthorizationLevel.Function, "post")] HttpRequest req,
        [DurableClient] DurableTaskClient client)
    {
        var order = await req.ReadFromJsonAsync<Order>();
        var instanceId = await client.ScheduleNewOrchestrationInstanceAsync(
            nameof(ProcessOrder), order);
        return new OkObjectResult(new { instanceId });
    }
    
    [Function(nameof(ProcessOrder))]
    public async Task<OrderResult> ProcessOrder(
        [OrchestrationTrigger] TaskOrchestrationContext context)
    {
        // Orchestration logic
    }
}
```

---

## When to Choose Durable Task SDKs

Choose Durable Task SDKs when:

### ✅ Ideal Scenarios

1. **Container-Based Deployments**
   - Running on Kubernetes (AKS)
   - Using Azure Container Apps
   - Existing containerized microservices

2. **Long-Running Workers**
   - Background processing services
   - Workers that should always be available
   - Avoiding cold starts

3. **Non-Azure Functions Environments**
   - On-premises deployments
   - Multi-cloud strategies
   - Custom hosting requirements

4. **Full Control Over Host**
   - Custom dependency injection
   - Specific runtime configurations
   - Integration with existing app frameworks

5. **High-Throughput Scenarios**
   - Consistent performance requirements
   - Predictable scaling patterns

### Code Example

```csharp
// Durable Task SDK style - as a background service
var builder = Host.CreateApplicationBuilder(args);

builder.Services.AddDurableTaskWorker(options =>
{
    options.AddOrchestrator<OrderProcessingOrchestrator>();
    options.AddActivity<ValidateOrderActivity>();
})
.UseDurableTaskScheduler(endpoint, taskHub);

// Also add your API layer
builder.Services.AddControllers();

var host = builder.Build();
await host.RunAsync();
```

---

## Hybrid Scenarios

You can use both in the same solution:

```
┌─────────────────────────────────────────────────────────────────┐
│                    HYBRID ARCHITECTURE                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │              Durable Task Scheduler                      │   │
│   │                   (Shared Backend)                       │   │
│   └───────────────────────┬─────────────────────────────────┘   │
│                           │                                      │
│           ┌───────────────┼───────────────┐                     │
│           │               │               │                      │
│           ▼               ▼               ▼                      │
│   ┌───────────────┐ ┌───────────────┐ ┌───────────────┐         │
│   │    Azure      │ │   Container   │ │     AKS       │         │
│   │   Functions   │ │     Apps      │ │    Workers    │         │
│   │               │ │               │ │               │         │
│   │ Event-driven  │ │ Long-running  │ │ High-volume   │         │
│   │ triggers      │ │ workers       │ │ processing    │         │
│   └───────────────┘ └───────────────┘ └───────────────┘         │
│                                                                  │
│   Use Case:           Use Case:         Use Case:                │
│   - HTTP APIs         - Background      - Batch jobs            │
│   - Queue triggers    - Always-on       - ML pipelines          │
│   - Event Grid        - Low latency     - Complex flows         │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Migration Paths

### From Durable Functions to Durable Task SDK

```csharp
// Before (Durable Functions)
[Function(nameof(ProcessOrder))]
public async Task<OrderResult> ProcessOrder(
    [OrchestrationTrigger] TaskOrchestrationContext context)
{
    var order = context.GetInput<Order>();
    await context.CallActivityAsync("Validate", order);
    return new OrderResult();
}

// After (Durable Task SDK) - Very similar!
public class ProcessOrderOrchestrator : TaskOrchestrator<Order, OrderResult>
{
    public override async Task<OrderResult> RunAsync(
        TaskOrchestrationContext context, Order order)
    {
        await context.CallActivityAsync("Validate", order);
        return new OrderResult();
    }
}
```

Key changes:
1. Move from trigger-based to class-based syntax
2. Register orchestrators/activities explicitly
3. Create your own hosting (console app, ASP.NET, etc.)

---

## Cost Considerations

| Aspect | Durable Functions | Durable Task SDKs |
|--------|-------------------|-------------------|
| **Idle Cost** | $0 (Consumption) | Compute costs |
| **Per-Execution** | Small cost per execution | Included in compute |
| **Predictable Load** | May cost more | Usually cheaper |
| **Burst Traffic** | Scales automatically | Need over-provision |
| **Durable Task Scheduler** | Same pricing | Same pricing |

### Cost Optimization Tips

**Durable Functions:**
- Use Consumption plan for sporadic workloads
- Use Premium plan to avoid cold starts
- Monitor execution count and duration

**Durable Task SDKs:**
- Right-size your containers
- Use autoscaling effectively
- Consider spot instances for batch workloads

---

## Summary Table

| Requirement | Recommendation |
|-------------|----------------|
| Serverless, pay-per-use | Durable Functions |
| Always-on worker | Durable Task SDKs |
| Container-based | Durable Task SDKs |
| Quick start/prototype | Durable Functions |
| Full host control | Durable Task SDKs |
| Event Grid/Queue triggers | Durable Functions |
| Kubernetes deployment | Durable Task SDKs |
| Both options work | Choose based on team skills |

---

## Next Steps

- [Durable Functions Setup →](../architecture/durable-functions-dts.md)
- [Container Apps Setup →](../architecture/aca-dts.md)
- [AKS Setup →](../architecture/aks-dts.md)
- [Advantages of Durable Task Scheduler →](advantages.md)

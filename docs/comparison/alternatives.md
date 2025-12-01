---
layout: default
title: Comparison with Alternatives
parent: Comparison
nav_order: 3
permalink: /docs/comparison/alternatives/
---

# Azure Durable vs. Alternative Solutions
{: .no_toc }

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

How does Azure Durable compare to other Azure workflow and messaging solutions?
{: .fs-6 .fw-300 }

---

## Overview

When building workflows and distributed applications on Azure, you have several options. This guide compares Azure Durable to other Azure-native alternatives to help you choose the right tool for your scenario.

---

## Quick Comparison

| Feature | Azure Durable | Azure Logic Apps | Azure Service Bus | Orleans |
|---------|--------------|------------------|-------------------|---------|
| **Primary Use Case** | Code-first workflows | Low-code integrations | Messaging & queuing | Virtual actor model |
| **Programming Model** | Code (C#, Python, JS) | Visual designer / JSON | SDK / REST | C# grains |
| **State Management** | Automatic, durable | Workflow state | No built-in state | Actor state |
| **Serverless Option** | ✅ Durable Functions | ✅ Consumption plan | ✅ | ❌ |
| **Long-running Support** | ✅ Built-in | ✅ Built-in | ❌ Manual | ✅ Built-in |
| **Connectors** | Custom code | 400+ pre-built | N/A | Custom code |
| **Learning Curve** | Low-Medium | Low | Low | Medium-High |

---

## Azure Durable vs. Azure Logic Apps

### Similarities
- Both are Azure-native
- Both support long-running workflows
- Both offer monitoring and diagnostics

### Key Differences

| Aspect | Azure Durable | Azure Logic Apps |
|--------|--------------|------------------|
| **Programming Model** | Code-first | Visual designer / JSON |
| **Target Audience** | Developers | Developers + Citizen integrators |
| **Connectors** | Custom code | 400+ pre-built connectors |
| **Performance** | High throughput | Optimized for integration |
| **Testing** | Standard unit tests | Logic Apps Test Framework |
| **Complexity** | Better for complex logic | Better for integrations |

### When to Choose Azure Durable
- You need complex business logic
- You want maximum control over execution
- Performance is critical
- You prefer code over visual design

### When to Choose Azure Logic Apps
- You need many pre-built connectors
- Non-developers need to create workflows
- You're building integration scenarios
- You prefer low-code development

---

## Azure Durable vs. Azure Service Bus

### What is Azure Service Bus?
Azure Service Bus is a fully managed enterprise message broker with queues and publish-subscribe topics. It excels at decoupling applications and services through asynchronous messaging.

### Key Differences

| Aspect | Azure Durable | Azure Service Bus |
|--------|--------------|-------------------|
| **Primary Purpose** | Workflow orchestration | Message delivery |
| **State Management** | Built-in, automatic | None (stateless messaging) |
| **Workflow Coordination** | Native support | Manual implementation |
| **Long-running Processes** | Built-in timers & waits | Requires external state store |
| **Message Patterns** | Orchestrated sequences | Pub/sub, queues, topics |
| **Retry Logic** | Declarative retry policies | Dead-letter queues |
| **Visibility** | Execution history & dashboard | Message metrics only |

### When to Choose Azure Durable
- You need to coordinate multiple steps in sequence
- You require automatic state persistence
- You want visibility into workflow execution
- You need human-in-the-loop workflows
- Your process has complex branching logic

### When to Choose Azure Service Bus
- You need simple message delivery between services
- You want to decouple producers and consumers
- You need pub/sub or competing consumers patterns
- Message ordering and delivery guarantees are key
- You're building event-driven microservices without orchestration

### Using Them Together

Azure Durable and Service Bus complement each other well:

```csharp
[Function(nameof(ProcessOrder))]
public async Task<OrderResult> ProcessOrder(
    [OrchestrationTrigger] TaskOrchestrationContext context)
{
    var order = context.GetInput<Order>();
    
    // Use Durable for orchestration
    await context.CallActivityAsync(nameof(ValidateOrder), order);
    await context.CallActivityAsync(nameof(ProcessPayment), order);
    
    // Use Service Bus to notify downstream systems
    await context.CallActivityAsync(nameof(PublishToServiceBus), new OrderEvent {
        Type = "OrderCompleted",
        OrderId = order.Id
    });
    
    return new OrderResult { Success = true };
}

[Function(nameof(PublishToServiceBus))]
[ServiceBusOutput("orders-topic", Connection = "ServiceBusConnection")]
public OrderEvent PublishToServiceBus([ActivityTrigger] OrderEvent orderEvent)
{
    return orderEvent; // Automatically published to Service Bus
}
```

---

## Azure Durable vs. Orleans

### What is Orleans?
Orleans is a cross-platform framework for building distributed applications using the virtual actor model. It was developed by Microsoft Research and powers services like Xbox Live, Halo, and Azure PlayFab.

### Key Differences

| Aspect | Azure Durable | Orleans |
|--------|--------------|---------|
| **Programming Model** | Orchestrator + Activities | Virtual Actors (Grains) |
| **State Persistence** | Event sourcing (replay) | Grain state snapshots |
| **Execution Model** | Workflow-centric | Actor-centric |
| **Hosting** | Serverless or containers | Self-hosted clusters |
| **Scalability** | Automatic (managed) | Manual cluster management |
| **Use Case** | Sequential workflows | Stateful distributed objects |
| **Managed Option** | ✅ Durable Task Scheduler | ❌ Self-host only |

### Programming Model Comparison

**Azure Durable (Workflow-centric):**
```csharp
[Function(nameof(ProcessOrder))]
public async Task<OrderResult> ProcessOrder(
    [OrchestrationTrigger] TaskOrchestrationContext context)
{
    var order = context.GetInput<Order>();
    
    // Sequential workflow steps
    await context.CallActivityAsync(nameof(ValidateOrder), order);
    await context.CallActivityAsync(nameof(ChargePayment), order);
    await context.CallActivityAsync(nameof(ShipOrder), order);
    
    return new OrderResult { Success = true };
}
```

**Orleans (Actor-centric):**
```csharp
public interface IOrderGrain : IGrainWithStringKey
{
    Task<OrderResult> ProcessAsync(Order order);
}

public class OrderGrain : Grain, IOrderGrain
{
    private readonly IPersistentState<OrderState> _state;
    
    public async Task<OrderResult> ProcessAsync(Order order)
    {
        // Actor manages its own state
        _state.State.Order = order;
        _state.State.Status = "Processing";
        await _state.WriteStateAsync();
        
        // Call other actors
        var paymentGrain = GrainFactory.GetGrain<IPaymentGrain>(order.Id);
        await paymentGrain.ChargeAsync(order.Total);
        
        var shippingGrain = GrainFactory.GetGrain<IShippingGrain>(order.Id);
        await shippingGrain.ShipAsync(order);
        
        return new OrderResult { Success = true };
    }
}
```

### When to Choose Azure Durable
- You're building sequential, long-running workflows
- You want serverless execution with pay-per-use
- You need built-in workflow patterns (fan-out, human interaction, timers)
- You prefer managed infrastructure
- You need visibility into workflow execution history

### When to Choose Orleans
- You're modeling domain entities as stateful actors
- You need fine-grained concurrency control per entity
- You're building real-time, interactive systems (games, IoT)
- You have experience with actor model programming
- You need direct actor-to-actor communication patterns

### Using Them Together

For complex systems, you might use both:

- **Orleans** for real-time, stateful entity management (e.g., game sessions, IoT devices)
- **Azure Durable** for background workflows that span those entities (e.g., batch processing, scheduled jobs)

---

## Decision Matrix

Use this matrix to help choose the right solution:

| Your Requirement | Recommended Solution |
|-----------------|---------------------|
| Code-first, long-running workflows | **Azure Durable** |
| Low-code integrations with 400+ connectors | **Azure Logic Apps** |
| Simple message delivery & pub/sub | **Azure Service Bus** |
| Stateful virtual actors for real-time systems | **Orleans** |
| Serverless workflow execution | **Azure Durable Functions** |
| Visual workflow design for citizen developers | **Azure Logic Apps** |
| Event-driven microservices decoupling | **Azure Service Bus** |
| Containerized workflow workers | **Azure Durable + Durable Task SDKs** |

---

## Summary

Each Azure solution has its strengths:

| Solution | Best For |
|----------|----------|
| **Azure Durable** | Code-first workflows with automatic state, retries, and orchestration |
| **Azure Logic Apps** | Low-code integrations with pre-built connectors |
| **Azure Service Bus** | Reliable message delivery and decoupling services |
| **Orleans** | Stateful distributed objects with the virtual actor model |

### Azure Durable excels when you need:

- ✅ **Code-first development** with full IDE support and unit testing
- ✅ **Automatic state management** without external databases
- ✅ **Long-running workflows** with timers and human interaction
- ✅ **Workflow patterns** like fan-out/fan-in, chaining, and sub-orchestrations
- ✅ **Managed infrastructure** with the Durable Task Scheduler

### Choose alternatives when:

- You need 400+ pre-built connectors → **Logic Apps**
- You need simple message pub/sub → **Service Bus**
- You're modeling stateful actors → **Orleans**
- Non-developers create workflows → **Logic Apps**

---

## Next Steps

- [When to Use What →](./when-to-use.md)
- [Advantages of Durable Task Scheduler →](./advantages.md)
- [Get Started with Durable Functions →](../durable-functions/quickstart.md)
- [Get Started with Durable Task SDKs →](../sdks/quickstart.md)

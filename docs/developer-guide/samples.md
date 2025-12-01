---
layout: default
title: Samples
parent: Developer Guide
nav_order: 20
permalink: /docs/developer-guide/samples/
---

# Durable Task SDK Samples
{: .no_toc }

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

Code samples for the Durable Task SDKs, designed to run on any compute platform (Azure Container Apps, AKS, etc.) with the Durable Task Scheduler.

---

## Language-Specific Samples

### .NET SDK

| Sample | Description |
|--------|-------------|
| [Function Chaining](https://github.com/Azure-Samples/Durable-Task-Scheduler/tree/main/samples/durable-task-sdks/dotnet/function-chaining) | Basic sequence of functions |
| [Fan-Out/Fan-In](https://github.com/Azure-Samples/Durable-Task-Scheduler/tree/main/samples/durable-task-sdks/dotnet/fan-out-fan-in) | Parallel execution |
| [Eternal Orchestration](https://github.com/Azure-Samples/Durable-Task-Scheduler/tree/main/samples/durable-task-sdks/dotnet/eternal-orchestrations) | Infinite loops |

### Python SDK

| Sample | Description |
|--------|-------------|
| [Function Chaining](https://github.com/Azure-Samples/Durable-Task-Scheduler/tree/main/samples/durable-task-sdks/python/function-chaining) | Basic sequence of functions |
| [Fan-Out/Fan-In](https://github.com/Azure-Samples/Durable-Task-Scheduler/tree/main/samples/durable-task-sdks/python/fan-out-fan-in) | Parallel execution |
| [Async HTTP](https://github.com/Azure-Samples/Durable-Task-Scheduler/tree/main/samples/durable-task-sdks/python/async-http-api) | HTTP API integration |

### Java SDK

| Sample | Description |
|--------|-------------|
| [Function Chaining](https://github.com/Azure-Samples/Durable-Task-Scheduler/tree/main/samples/durable-task-sdks/java/function-chaining) | Basic sequence of functions |
| [Fan-Out/Fan-In](https://github.com/Azure-Samples/Durable-Task-Scheduler/tree/main/samples/durable-task-sdks/java/fan-out-fan-in) | Parallel execution |
| [Monitoring](https://github.com/Azure-Samples/Durable-Task-Scheduler/tree/main/samples/durable-task-sdks/java/monitoring) | Orchestration monitoring |

---

## Common Patterns

### Hello World (.NET)

```csharp
using Microsoft.DurableTask;
using Microsoft.DurableTask.Worker;
using Microsoft.DurableTask.Client;

// Orchestrator
[DurableTask(nameof(HelloOrchestrator))]
public class HelloOrchestrator : TaskOrchestrator<string, string>
{
    public override async Task<string> RunAsync(
        TaskOrchestrationContext context, 
        string name)
    {
        return await context.CallActivityAsync<string>(
            nameof(SayHelloActivity), 
            name
        );
    }
}

// Activity
[DurableTask(nameof(SayHelloActivity))]
public class SayHelloActivity : TaskActivity<string, string>
{
    public override Task<string> RunAsync(
        TaskActivityContext context, 
        string name)
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

### Hello World (Java)

```java
import com.microsoft.durabletask.*;

public class HelloOrchestrator implements TaskOrchestrator {
    @Override
    public Object run(TaskOrchestrationContext ctx) {
        String name = ctx.getInput(String.class);
        return ctx.callActivity("SayHello", name, String.class).await();
    }
}

public class SayHelloActivity implements TaskActivity<String, String> {
    @Override
    public String run(TaskActivityContext ctx, String name) {
        return "Hello, " + name + "!";
    }
}
```

---

## Fan-Out/Fan-In (.NET)

```csharp
[DurableTask(nameof(FanOutOrchestrator))]
public class FanOutOrchestrator : TaskOrchestrator<List<string>, List<string>>
{
    public override async Task<List<string>> RunAsync(
        TaskOrchestrationContext context, 
        List<string> items)
    {
        // Fan out - schedule all tasks in parallel
        var tasks = items.Select(item =>
            context.CallActivityAsync<string>(nameof(ProcessItem), item)
        ).ToList();
        
        // Fan in - wait for all to complete
        var results = await Task.WhenAll(tasks);
        
        return results.ToList();
    }
}

[DurableTask(nameof(ProcessItem))]
public class ProcessItem : TaskActivity<string, string>
{
    public override Task<string> RunAsync(
        TaskActivityContext context, 
        string item)
    {
        return Task.FromResult($"Processed: {item}");
    }
}
```

---

## External Events (.NET)

```csharp
[DurableTask(nameof(ApprovalOrchestrator))]
public class ApprovalOrchestrator : TaskOrchestrator<ApprovalRequest, ApprovalResult>
{
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
}
```

---

## Sub-Orchestrations (Python)

```python
@task.orchestrator
def parent_orchestrator(ctx: task.OrchestrationContext, order: dict):
    # Process payment in sub-orchestration
    payment_result = yield ctx.call_sub_orchestrator(
        orchestrator=payment_orchestrator,
        input=order["payment"]
    )
    
    # Process shipping in sub-orchestration
    shipping_result = yield ctx.call_sub_orchestrator(
        orchestrator=shipping_orchestrator,
        input={"order_id": order["id"], "address": order["address"]}
    )
    
    return {
        "order_id": order["id"],
        "payment_id": payment_result["transaction_id"],
        "tracking": shipping_result["tracking_number"]
    }

@task.orchestrator
def payment_orchestrator(ctx: task.OrchestrationContext, payment: dict):
    result = yield ctx.call_activity("process_payment", input=payment)
    return result

@task.orchestrator
def shipping_orchestrator(ctx: task.OrchestrationContext, shipping: dict):
    result = yield ctx.call_activity("create_shipment", input=shipping)
    return result
```

---

## Durable Timers (Java)

```java
public class ReminderOrchestrator implements TaskOrchestrator {
    @Override
    public Object run(TaskOrchestrationContext ctx) {
        String userId = ctx.getInput(String.class);
        
        // Send initial notification
        ctx.callActivity("SendReminder", userId, Void.class).await();
        
        // Wait 24 hours
        ctx.createTimer(Duration.ofHours(24)).await();
        
        // Send follow-up
        ctx.callActivity("SendFollowUp", userId, Void.class).await();
        
        // Wait another 24 hours
        ctx.createTimer(Duration.ofHours(24)).await();
        
        // Final reminder
        ctx.callActivity("SendFinalReminder", userId, Void.class).await();
        
        return "Reminder sequence completed";
    }
}
```

---

## End-to-End Sample Repository

For complete, runnable samples across all languages, visit the official sample repository:

[**Azure-Samples/Durable-Task-Scheduler**](https://github.com/Azure-Samples/Durable-Task-Scheduler)

The repository includes:
- Complete project setup
- Docker Compose for local development
- Deployment scripts for Azure
- Integration with Durable Task Scheduler

---

## Next Steps

- [.NET SDK Reference →](./dotnet.md)
- [Python SDK Reference →](./python.md)
- [Java SDK Reference →](./java.md)
- [Orchestration Patterns →](../patterns/index.md)
- [Deploy to Azure Container Apps →](../hosting-options/container-apps/index.md)

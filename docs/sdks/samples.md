---
layout: default
title: Samples
parent: Durable Task SDKs
nav_order: 20
---

# Durable Task SDK Samples

This section contains code samples for the **Durable Task SDKs** (Portable), designed to run on any compute platform (ACA, AKS, etc.) with the Durable Task Scheduler.

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

## Common Patterns

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

---
layout: default
title: Python SDK
parent: SDKs Overview
grand_parent: Developer Reference
nav_order: 2
has_children: true
permalink: /docs/developer-guide/sdks/python/
---

# Python SDK Reference
{: .no_toc }

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

Comprehensive reference for Durable Task Python SDKs.
{: .fs-6 .fw-300 }

---

## SDK Options

Python has two SDK options for building durable orchestrations:

| SDK Package | Hosting | Description |
|-------------|---------|-------------|
| **azure-functions-durable** | Azure Functions | Full-featured Durable Functions SDK |
| **durabletask** | Portable | Platform-agnostic SDK for any host |

---

## Azure Functions SDK

### Installation

```bash
pip install azure-functions-durable
```

### Programming Models

Python Durable Functions supports two programming models:

| Model | Minimum Version | Description |
|-------|----------------|-------------|
| **v2** (Decorators) | Python 3.7+, Functions 4.0+ | Code-centric, decorator-based |
| **v1** (function.json) | Python 3.7+, Functions 2.0+ | JSON configuration files |

The v2 model is recommended for new projects.

### Project Structure (v2 Model)

```
my-function-app/
├── function_app.py      # Main application file
├── requirements.txt     # Python dependencies
├── host.json            # Host configuration
└── local.settings.json  # Local settings
```

### Basic Setup (v2 Model)

```python
import azure.functions as func
import azure.durable_functions as df

# Create the Durable Functions app
app = df.DFApp(http_auth_level=func.AuthLevel.ANONYMOUS)

# Define orchestrator
@app.orchestration_trigger(context_name="context")
def hello_orchestrator(context: df.DurableOrchestrationContext):
    result1 = yield context.call_activity("say_hello", "Tokyo")
    result2 = yield context.call_activity("say_hello", "Seattle")
    result3 = yield context.call_activity("say_hello", "London")
    return [result1, result2, result3]

# Define activity
@app.activity_trigger(input_name="city")
def say_hello(city: str) -> str:
    return f"Hello {city}!"

# Define HTTP starter
@app.route(route="orchestrators/{function_name}")
@app.durable_client_input(client_name="client")
async def http_start(req: func.HttpRequest, client):
    function_name = req.route_params.get("function_name")
    instance_id = await client.start_new(function_name)
    response = client.create_check_status_response(req, instance_id)
    return response
```

### Dependencies

```
# requirements.txt
azure-functions
azure-functions-durable
```

---

## Portable SDK (durabletask)

### Installation

```bash
pip install durabletask-azuremanaged
```

### Basic Setup

```python
import asyncio
from durabletask import task
from durabletask.azuremanaged import DurableTaskSchedulerClient, DurableTaskSchedulerWorker

# Define orchestration
@task.orchestration
def process_order(ctx: task.OrchestrationContext, order: dict):
    # Validate order
    is_valid = yield ctx.call_activity(validate_order, input=order)
    if not is_valid:
        return {"success": False, "reason": "Invalid order"}
    
    # Process payment
    payment_result = yield ctx.call_activity(process_payment, input=order)
    
    # Ship order
    yield ctx.call_activity(ship_order, input=order)
    
    return {"success": True, "payment_id": payment_result["id"]}

# Define activities
@task.activity
def validate_order(ctx: task.ActivityContext, order: dict) -> bool:
    return "items" in order and len(order["items"]) > 0

@task.activity
def process_payment(ctx: task.ActivityContext, order: dict) -> dict:
    return {"id": f"payment-{order['id']}", "status": "completed"}

@task.activity
def ship_order(ctx: task.ActivityContext, order: dict):
    print(f"Shipping order {order['id']}")

async def main():
    connection_string = "your-connection-string"
    
    # Create worker
    async with DurableTaskSchedulerWorker(connection_string) as worker:
        # Register tasks
        worker.add_orchestration(process_order)
        worker.add_activity(validate_order)
        worker.add_activity(process_payment)
        worker.add_activity(ship_order)
        
        # Start worker
        await worker.start()
        
        # Create client
        client = DurableTaskSchedulerClient(connection_string)
        
        # Start orchestration
        instance_id = await client.schedule_new_orchestration(
            process_order,
            input={"id": "order-123", "items": ["item1", "item2"]}
        )
        
        # Wait for completion
        result = await client.wait_for_orchestration_completion(instance_id)
        print(f"Result: {result.result}")

if __name__ == "__main__":
    asyncio.run(main())
```

---

## Orchestrations

### Generator-Based Syntax

Python orchestrations use generator functions with `yield`:

```python
@app.orchestration_trigger(context_name="context")
def my_orchestrator(context: df.DurableOrchestrationContext):
    # Each activity call must be yielded
    result1 = yield context.call_activity("activity1", input1)
    result2 = yield context.call_activity("activity2", input2)
    return result1 + result2
```

### Async Orchestrations

For the portable SDK, async/await syntax is also supported:

```python
@task.orchestration
async def async_orchestrator(ctx: task.OrchestrationContext, input: str):
    result = await ctx.call_activity(my_activity, input=input)
    return result
```

---

## Activities

### Azure Functions Activities

```python
@app.activity_trigger(input_name="data")
def process_data(data: dict) -> dict:
    # Perform work
    processed = transform(data)
    return processed

# With complex input
@app.activity_trigger(input_name="request")
def complex_activity(request: ProcessRequest) -> ProcessResult:
    return ProcessResult(status="completed")
```

### Portable SDK Activities

```python
@task.activity
def my_activity(ctx: task.ActivityContext, input: str) -> str:
    return f"Processed: {input}"

# Async activity
@task.activity
async def async_activity(ctx: task.ActivityContext, data: dict) -> dict:
    result = await external_api_call(data)
    return result
```

---

## Sub-Orchestrations

```python
@app.orchestration_trigger(context_name="context")
def parent_orchestrator(context: df.DurableOrchestrationContext):
    # Call sub-orchestration
    result = yield context.call_sub_orchestrator(
        "child_orchestrator",
        input_data
    )
    return result

@app.orchestration_trigger(context_name="context")
def child_orchestrator(context: df.DurableOrchestrationContext):
    # Child orchestration logic
    return "child result"
```

---

## Timers

### Durable Timer

```python
@app.orchestration_trigger(context_name="context")
def timer_orchestrator(context: df.DurableOrchestrationContext):
    import datetime
    
    # Wait for 1 hour
    fire_at = context.current_utc_datetime + datetime.timedelta(hours=1)
    yield context.create_timer(fire_at)
    
    # Continue after timer
    return "Timer completed"
```

### Timer with Cancellation

```python
@app.orchestration_trigger(context_name="context")
def timer_with_cancellation(context: df.DurableOrchestrationContext):
    import datetime
    
    fire_at = context.current_utc_datetime + datetime.timedelta(hours=24)
    timer_task = context.create_timer(fire_at)
    event_task = context.wait_for_external_event("ApprovalEvent")
    
    winner = yield context.task_any([timer_task, event_task])
    
    if winner == event_task:
        timer_task.cancel()
        return "Approved"
    else:
        return "Timeout"
```

---

## External Events

### Waiting for Events

```python
@app.orchestration_trigger(context_name="context")
def approval_workflow(context: df.DurableOrchestrationContext):
    # Wait for approval event
    approval = yield context.wait_for_external_event("ApprovalReceived")
    
    if approval["approved"]:
        yield context.call_activity("process_approved", approval)
    else:
        yield context.call_activity("process_rejected", approval)
```

### Raising Events (Client)

```python
@app.route(route="approve/{instance_id}")
@app.durable_client_input(client_name="client")
async def approve(req: func.HttpRequest, client):
    instance_id = req.route_params.get("instance_id")
    body = req.get_json()
    
    await client.raise_event(
        instance_id,
        "ApprovalReceived",
        {"approved": True, "approver": body["approver"]}
    )
    
    return func.HttpResponse("Event raised", status_code=202)
```

---

## Entities (Azure Functions)

### Class-Based Entity

```python
from azure.durable_functions import EntityContext

class Counter:
    def __init__(self):
        self.value = 0
    
    def add(self, amount: int):
        self.value += amount
    
    def reset(self):
        self.value = 0
    
    def get(self) -> int:
        return self.value

@app.entity_trigger(context_name="context")
def counter_entity(context: EntityContext):
    state = context.get_state(Counter) or Counter()
    operation = context.operation_name
    
    if operation == "add":
        state.add(context.get_input())
    elif operation == "reset":
        state.reset()
    elif operation == "get":
        context.set_result(state.get())
    
    context.set_state(state)
```

### Signaling Entities

```python
@app.orchestration_trigger(context_name="context")
def entity_orchestrator(context: df.DurableOrchestrationContext):
    entity_id = df.EntityId("counter_entity", "myCounter")
    
    # Signal (fire-and-forget)
    context.signal_entity(entity_id, "add", 5)
    
    # Call (with return value)
    value = yield context.call_entity(entity_id, "get")
    
    return value
```

---

## Error Handling

### Retry Policies

```python
@app.orchestration_trigger(context_name="context")
def retry_orchestrator(context: df.DurableOrchestrationContext):
    retry_options = df.RetryOptions(
        first_retry_interval_in_milliseconds=5000,
        max_number_of_attempts=3
    )
    retry_options.backoff_coefficient = 2.0
    retry_options.max_retry_interval_in_milliseconds=60000
    
    result = yield context.call_activity_with_retry(
        "unreliable_activity",
        retry_options,
        input_data
    )
    return result
```

### Exception Handling

```python
@app.orchestration_trigger(context_name="context")
def error_handling_orchestrator(context: df.DurableOrchestrationContext):
    try:
        result = yield context.call_activity("risky_activity", input_data)
        return {"success": True, "result": result}
    except Exception as e:
        # Handle the error
        yield context.call_activity("log_error", str(e))
        return {"success": False, "error": str(e)}
```

---

## Fan-Out/Fan-In

```python
@app.orchestration_trigger(context_name="context")
def fan_out_fan_in(context: df.DurableOrchestrationContext):
    items = context.get_input()
    
    # Fan-out: create parallel tasks
    tasks = []
    for item in items:
        task = context.call_activity("process_item", item)
        tasks.append(task)
    
    # Fan-in: wait for all tasks
    results = yield context.task_all(tasks)
    
    # Aggregate results
    total = sum(results)
    return total
```

---

## Eternal Orchestrations

```python
@app.orchestration_trigger(context_name="context")
def eternal_orchestrator(context: df.DurableOrchestrationContext):
    import datetime
    
    # Get current state or initialize
    iteration = context.get_input() or 0
    
    # Do periodic work
    yield context.call_activity("do_periodic_work", iteration)
    
    # Wait for next interval
    next_run = context.current_utc_datetime + datetime.timedelta(hours=1)
    yield context.create_timer(next_run)
    
    # Continue as new to prevent history growth
    context.continue_as_new(iteration + 1)
```

---

## Client Operations

### Starting Orchestrations

```python
@app.route(route="start")
@app.durable_client_input(client_name="client")
async def start_orchestration(req: func.HttpRequest, client):
    # Start with auto-generated ID
    instance_id = await client.start_new("my_orchestrator", None, input_data)
    
    # Start with specific ID
    instance_id = await client.start_new(
        "my_orchestrator",
        instance_id="my-custom-id",
        client_input=input_data
    )
    
    return client.create_check_status_response(req, instance_id)
```

### Querying Status

```python
@app.route(route="status/{instance_id}")
@app.durable_client_input(client_name="client")
async def get_status(req: func.HttpRequest, client):
    instance_id = req.route_params.get("instance_id")
    
    status = await client.get_status(instance_id)
    
    return func.HttpResponse(
        json.dumps({
            "instanceId": status.instance_id,
            "runtimeStatus": status.runtime_status.name,
            "output": status.output
        }),
        mimetype="application/json"
    )
```

### Terminating Orchestrations

```python
@app.route(route="terminate/{instance_id}")
@app.durable_client_input(client_name="client")
async def terminate(req: func.HttpRequest, client):
    instance_id = req.route_params.get("instance_id")
    
    await client.terminate(instance_id, "Terminated by user")
    
    return func.HttpResponse("Terminated", status_code=202)
```

---

## Detailed Documentation

- [Azure Functions SDK →](./durable-functions.md) - Complete azure-functions-durable reference
- [Portable SDK →](./durabletask-sdk.md) - durabletask SDK reference

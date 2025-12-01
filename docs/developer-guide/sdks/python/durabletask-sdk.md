---
layout: default
title: Portable durabletask SDK
parent: Python SDK
grand_parent: SDKs Overview
nav_order: 2
permalink: /docs/developer-guide/sdks/python/durabletask-sdk/
---

# Portable durabletask Python SDK
{: .no_toc }

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

Reference for the platform-agnostic durabletask Python SDK.
{: .fs-6 .fw-300 }

---

## Overview

The `durabletask` package provides a portable SDK for building durable orchestrations outside of Azure Functions. It works with the Durable Task Scheduler and can run on any Python hosting platform.

---

## Installation

```bash
# Core package with Azure Managed backend
pip install durabletask-azuremanaged
```

---

## Quick Start

```python
import asyncio
from durabletask import task
from durabletask.azuremanaged import DurableTaskSchedulerClient, DurableTaskSchedulerWorker

# Define orchestration
@task.orchestration
def hello_cities(ctx: task.OrchestrationContext, _):
    result1 = yield ctx.call_activity(say_hello, input="Tokyo")
    result2 = yield ctx.call_activity(say_hello, input="Seattle")
    result3 = yield ctx.call_activity(say_hello, input="London")
    return [result1, result2, result3]

# Define activity
@task.activity
def say_hello(ctx: task.ActivityContext, city: str) -> str:
    return f"Hello {city}!"

async def main():
    connection_string = "your-connection-string"
    
    # Create and start worker
    async with DurableTaskSchedulerWorker(connection_string) as worker:
        worker.add_orchestration(hello_cities)
        worker.add_activity(say_hello)
        await worker.start()
        
        # Create client and start orchestration
        client = DurableTaskSchedulerClient(connection_string)
        instance_id = await client.schedule_new_orchestration(hello_cities)
        
        # Wait for result
        result = await client.wait_for_orchestration_completion(instance_id)
        print(f"Result: {result.result}")

if __name__ == "__main__":
    asyncio.run(main())
```

---

## Orchestration Decorator

### @task.orchestration

Defines an orchestration function.

```python
@task.orchestration
def my_orchestration(ctx: task.OrchestrationContext, input: dict):
    # Orchestration logic using yield
    result = yield ctx.call_activity(my_activity, input=input["data"])
    return result
```

**Generator Syntax:**

Orchestrations must use generator syntax with `yield` for all async operations:

```python
@task.orchestration
def orchestration_example(ctx: task.OrchestrationContext, order: Order):
    # Each call_activity must be yielded
    validated = yield ctx.call_activity(validate_order, input=order)
    
    if not validated:
        return {"success": False}
    
    # Multiple sequential activities
    payment = yield ctx.call_activity(process_payment, input=order)
    shipping = yield ctx.call_activity(arrange_shipping, input=order)
    
    return {"success": True, "payment": payment, "shipping": shipping}
```

---

## Activity Decorator

### @task.activity

Defines an activity function.

```python
@task.activity
def my_activity(ctx: task.ActivityContext, input: str) -> str:
    # Activity logic
    return f"Processed: {input}"
```

**Async Activities:**

```python
@task.activity
async def async_activity(ctx: task.ActivityContext, data: dict) -> dict:
    result = await external_api_call(data)
    return result
```

---

## OrchestrationContext

Context object available in orchestration functions.

### Properties

| Property | Type | Description |
|----------|------|-------------|
| `instance_id` | `str` | Orchestration instance ID |
| `is_replaying` | `bool` | Whether replaying history |
| `current_utc_datetime` | `datetime` | Deterministic current time |

### Methods

#### call_activity()

Call an activity function.

```python
result = yield ctx.call_activity(
    activity=my_activity,
    input=input_data
)
```

#### call_sub_orchestration()

Call a sub-orchestration.

```python
result = yield ctx.call_sub_orchestration(
    orchestration=child_orchestration,
    input=input_data,
    instance_id="child-123"  # Optional
)
```

#### create_timer()

Create a durable timer.

```python
from datetime import timedelta

# Wait for duration
yield ctx.create_timer(timedelta(hours=1))

# Wait until specific time
fire_at = ctx.current_utc_datetime + timedelta(days=1)
yield ctx.create_timer(fire_at)
```

#### wait_for_external_event()

Wait for an external event.

```python
event_data = yield ctx.wait_for_external_event("ApprovalEvent")
```

#### continue_as_new()

Restart the orchestration.

```python
ctx.continue_as_new(new_input, save_events=True)
```

---

## ActivityContext

Context object available in activity functions.

### Properties

| Property | Type | Description |
|----------|------|-------------|
| `orchestration_id` | `str` | Parent orchestration ID |
| `task_id` | `int` | Activity task ID |

---

## DurableTaskSchedulerWorker

Worker that executes orchestrations and activities.

### Constructor

```python
from durabletask.azuremanaged import DurableTaskSchedulerWorker

worker = DurableTaskSchedulerWorker(
    connection_string="your-connection-string",
    task_hub_name="default"  # Optional
)
```

### Methods

#### add_orchestration()

Register an orchestration.

```python
worker.add_orchestration(my_orchestration)
```

#### add_activity()

Register an activity.

```python
worker.add_activity(my_activity)
```

#### start()

Start the worker.

```python
await worker.start()
```

#### stop()

Stop the worker.

```python
await worker.stop()
```

### Context Manager

```python
async with DurableTaskSchedulerWorker(connection_string) as worker:
    worker.add_orchestration(my_orchestration)
    worker.add_activity(my_activity)
    await worker.start()
    
    # Worker runs until context exits
```

---

## DurableTaskSchedulerClient

Client for managing orchestrations.

### Constructor

```python
from durabletask.azuremanaged import DurableTaskSchedulerClient

client = DurableTaskSchedulerClient(
    connection_string="your-connection-string",
    task_hub_name="default"  # Optional
)
```

### Methods

#### schedule_new_orchestration()

Start a new orchestration.

```python
# With auto-generated ID
instance_id = await client.schedule_new_orchestration(
    orchestration=my_orchestration,
    input=input_data
)

# With specific ID
instance_id = await client.schedule_new_orchestration(
    orchestration=my_orchestration,
    input=input_data,
    instance_id="my-custom-id"
)

# Scheduled start
from datetime import datetime, timedelta
instance_id = await client.schedule_new_orchestration(
    orchestration=my_orchestration,
    input=input_data,
    start_at=datetime.utcnow() + timedelta(hours=1)
)
```

#### get_orchestration_state()

Get orchestration status.

```python
state = await client.get_orchestration_state(instance_id)

if state:
    print(f"Status: {state.runtime_status}")
    print(f"Output: {state.result}")
```

#### wait_for_orchestration_completion()

Wait for an orchestration to complete.

```python
state = await client.wait_for_orchestration_completion(
    instance_id,
    timeout=timedelta(minutes=5)  # Optional
)

if state.runtime_status == task.OrchestrationStatus.COMPLETED:
    print(f"Result: {state.result}")
```

#### wait_for_orchestration_start()

Wait for an orchestration to start.

```python
state = await client.wait_for_orchestration_start(instance_id)
```

#### raise_event()

Raise an event to an orchestration.

```python
await client.raise_event(
    instance_id=instance_id,
    event_name="ApprovalEvent",
    data={"approved": True}
)
```

#### terminate_orchestration()

Terminate a running orchestration.

```python
await client.terminate_orchestration(
    instance_id=instance_id,
    reason="Cancelled by user"
)
```

#### suspend_orchestration()

Suspend an orchestration.

```python
await client.suspend_orchestration(
    instance_id=instance_id,
    reason="Maintenance"
)
```

#### resume_orchestration()

Resume a suspended orchestration.

```python
await client.resume_orchestration(
    instance_id=instance_id,
    reason="Maintenance complete"
)
```

#### purge_orchestration()

Purge orchestration history.

```python
await client.purge_orchestration(instance_id)
```

---

## OrchestrationState

Status information for an orchestration.

### Properties

| Property | Type | Description |
|----------|------|-------------|
| `instance_id` | `str` | Instance ID |
| `name` | `str` | Orchestration name |
| `runtime_status` | `OrchestrationStatus` | Current status |
| `created_at` | `datetime` | Creation time |
| `last_updated_at` | `datetime` | Last update time |
| `input` | `any` | Orchestration input |
| `result` | `any` | Orchestration output |
| `failure_details` | `FailureDetails` | Error information |

---

## OrchestrationStatus

| Value | Description |
|-------|-------------|
| `PENDING` | Waiting to start |
| `RUNNING` | Currently executing |
| `COMPLETED` | Finished successfully |
| `FAILED` | Failed with error |
| `TERMINATED` | Manually terminated |
| `SUSPENDED` | Suspended |

---

## Patterns

### Fan-Out/Fan-In

```python
@task.orchestration
def fan_out_fan_in(ctx: task.OrchestrationContext, items: list):
    # Create parallel tasks
    tasks = []
    for item in items:
        t = ctx.call_activity(process_item, input=item)
        tasks.append(t)
    
    # Wait for all tasks
    results = yield task.when_all(tasks)
    
    # Aggregate results
    total = yield ctx.call_activity(aggregate_results, input=results)
    return total

@task.activity
def process_item(ctx: task.ActivityContext, item: str) -> int:
    import random
    import time
    time.sleep(random.uniform(0.5, 2.0))  # Simulate work
    return len(item) * 2

@task.activity
def aggregate_results(ctx: task.ActivityContext, results: list) -> dict:
    return {
        "count": len(results),
        "sum": sum(results),
        "average": sum(results) / len(results) if results else 0
    }
```

### Approval Workflow

```python
@task.orchestration
def approval_workflow(ctx: task.OrchestrationContext, request: dict):
    from datetime import timedelta
    
    # Send approval request
    yield ctx.call_activity(send_approval_request, input=request)
    
    # Wait for approval with timeout
    timeout_at = ctx.current_utc_datetime + timedelta(days=3)
    
    # Create competing tasks
    timer_task = ctx.create_timer(timeout_at)
    approval_task = ctx.wait_for_external_event("ApprovalReceived")
    
    # Wait for first to complete
    winner = yield task.when_any([timer_task, approval_task])
    
    if winner == approval_task:
        approval = yield approval_task
        if approval.get("approved"):
            yield ctx.call_activity(process_approval, input=request)
            return {"status": "approved"}
        else:
            yield ctx.call_activity(process_rejection, input=request)
            return {"status": "rejected"}
    else:
        yield ctx.call_activity(escalate_request, input=request)
        return {"status": "escalated"}
```

### Eternal Orchestration

```python
@task.orchestration
def periodic_task(ctx: task.OrchestrationContext, state: dict):
    from datetime import timedelta
    
    state = state or {"iteration": 0}
    
    # Do periodic work
    yield ctx.call_activity(do_work, input=state)
    
    # Wait for next interval
    yield ctx.create_timer(timedelta(hours=1))
    
    # Continue as new to prevent history growth
    state["iteration"] += 1
    ctx.continue_as_new(state)
```

### Retry with Custom Logic

```python
@task.orchestration
def retry_orchestration(ctx: task.OrchestrationContext, input: dict):
    from datetime import timedelta
    
    max_attempts = 5
    attempt = 0
    
    while attempt < max_attempts:
        try:
            result = yield ctx.call_activity(unreliable_activity, input=input)
            return {"success": True, "result": result}
        except Exception as e:
            attempt += 1
            if attempt >= max_attempts:
                return {"success": False, "error": str(e)}
            
            # Exponential backoff
            delay = timedelta(seconds=2 ** attempt)
            yield ctx.create_timer(delay)
```

---

## Running with Docker (Emulator)

For local development, use the Durable Task Scheduler emulator:

```bash
# Pull and run the emulator
docker pull mcr.microsoft.com/dts/dts-emulator:latest
docker run -d -p 8080:8080 -p 8082:8082 mcr.microsoft.com/dts/dts-emulator:latest
```

Use local connection settings:

```python
# For local emulator
connection_string = None  # SDK defaults to localhost:8080

# Or explicitly
worker = DurableTaskSchedulerWorker(
    endpoint="http://localhost:8080",
    task_hub_name="default"
)

client = DurableTaskSchedulerClient(
    endpoint="http://localhost:8080",
    task_hub_name="default"
)
```

---

## Complete Example

```python
import asyncio
from datetime import timedelta
from durabletask import task
from durabletask.azuremanaged import DurableTaskSchedulerClient, DurableTaskSchedulerWorker

# Data classes
class Order:
    def __init__(self, id: str, items: list, total: float):
        self.id = id
        self.items = items
        self.total = total

# Orchestration
@task.orchestration
def process_order(ctx: task.OrchestrationContext, order_dict: dict):
    order = Order(**order_dict)
    
    # Validate order
    is_valid = yield ctx.call_activity(validate_order, input=order_dict)
    if not is_valid:
        return {"success": False, "reason": "Invalid order"}
    
    # Process items in parallel
    item_tasks = []
    for item in order.items:
        t = ctx.call_activity(process_item, input=item)
        item_tasks.append(t)
    
    item_results = yield task.when_all(item_tasks)
    
    # Process payment
    payment = yield ctx.call_activity(process_payment, input={
        "order_id": order.id,
        "total": order.total
    })
    
    # Schedule shipping
    yield ctx.call_activity(schedule_shipping, input={
        "order_id": order.id,
        "items": item_results
    })
    
    return {
        "success": True,
        "order_id": order.id,
        "payment_id": payment["id"]
    }

# Activities
@task.activity
def validate_order(ctx: task.ActivityContext, order: dict) -> bool:
    return len(order.get("items", [])) > 0

@task.activity
def process_item(ctx: task.ActivityContext, item: dict) -> dict:
    return {"item_id": item["id"], "status": "processed"}

@task.activity
def process_payment(ctx: task.ActivityContext, data: dict) -> dict:
    return {"id": f"payment-{data['order_id']}", "status": "completed"}

@task.activity
def schedule_shipping(ctx: task.ActivityContext, data: dict):
    print(f"Shipping scheduled for order {data['order_id']}")

# Main
async def main():
    connection_string = None  # Use local emulator
    
    async with DurableTaskSchedulerWorker(connection_string) as worker:
        # Register all tasks
        worker.add_orchestration(process_order)
        worker.add_activity(validate_order)
        worker.add_activity(process_item)
        worker.add_activity(process_payment)
        worker.add_activity(schedule_shipping)
        
        await worker.start()
        
        # Create client
        client = DurableTaskSchedulerClient(connection_string)
        
        # Start orchestration
        order = {
            "id": "order-123",
            "items": [
                {"id": "item-1", "name": "Widget"},
                {"id": "item-2", "name": "Gadget"}
            ],
            "total": 99.99
        }
        
        instance_id = await client.schedule_new_orchestration(
            process_order,
            input=order
        )
        print(f"Started orchestration: {instance_id}")
        
        # Wait for completion
        result = await client.wait_for_orchestration_completion(instance_id)
        print(f"Result: {result.result}")

if __name__ == "__main__":
    asyncio.run(main())
```

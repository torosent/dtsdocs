---
layout: default
title: Python SDK
parent: Durable Task SDKs
nav_order: 3
permalink: /docs/sdks/python/
---

# Python Durable Task SDK
{: .no_toc }

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

The Python Durable Task SDK enables building durable, reliable orchestrations using familiar Python syntax and async/await patterns.

---

## Installation

### pip

```bash
pip install durabletask-azuremanaged
```

### Requirements

- Python 3.10 or later
- asyncio support

---

## Quickstart: Function Chaining

This example demonstrates the "Function Chaining" pattern, where a sequence of functions executes in a specific order.

### 1. Define the Worker (`worker.py`)

The worker hosts the orchestrator and activity functions.

```python
import asyncio
import logging
import os
from azure.identity import DefaultAzureCredential, ManagedIdentityCredential
from durabletask.azuremanaged.worker import DurableTaskSchedulerWorker

# Configure logging
logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

# Activity functions
def say_hello(ctx, name: str) -> str:
    """First activity that greets the user."""
    logger.info(f"Activity say_hello called with name: {name}")
    return f"Hello {name}!"

def process_greeting(ctx, greeting: str) -> str:
    """Second activity that processes the greeting."""
    logger.info(f"Activity process_greeting called with greeting: {greeting}")
    return f"{greeting} How are you today?"

def finalize_response(ctx, response: str) -> str:
    """Third activity that finalizes the response."""
    logger.info(f"Activity finalize_response called with response: {response}")
    return f"{response} I hope you're doing well!"

# Orchestrator function
def function_chaining_orchestrator(ctx, name: str) -> str:
    """Orchestrator that demonstrates function chaining pattern."""
    logger.info(f"Starting function chaining orchestration for {name}")
    
    # Call first activity - passing input directly without named parameter
    greeting = yield ctx.call_activity('say_hello', input=name)
    
    # Call second activity with the result from first activity
    processed_greeting = yield ctx.call_activity('process_greeting', input=greeting)
    
    # Call third activity with the result from second activity
    final_response = yield ctx.call_activity('finalize_response', input=processed_greeting)
    
    return final_response

async def main():
    """Main entry point for the worker process."""
    logger.info("Starting Function Chaining pattern worker...")
    
    # Get environment variables for taskhub and endpoint with defaults
    taskhub_name = os.getenv("TASKHUB", "default")
    endpoint = os.getenv("ENDPOINT", "http://localhost:8080")

    print(f"Using taskhub: {taskhub_name}")
    print(f"Using endpoint: {endpoint}")
    
    # Credential handling
    credential = None
    if endpoint != "http://localhost:8080":
        credential = DefaultAzureCredential()
    
    with DurableTaskSchedulerWorker(
        host_address=endpoint, 
        secure_channel=endpoint != "http://localhost:8080",
        taskhub=taskhub_name, 
        token_credential=credential
    ) as worker:
        
        # Register activities and orchestrators
        worker.add_activity(say_hello)
        worker.add_activity(process_greeting)
        worker.add_activity(finalize_response)
        worker.add_orchestrator(function_chaining_orchestrator)
        
        # Start the worker
        worker.start()
        
        try:
            # Keep the worker running
            while True:
                await asyncio.sleep(1)
        except KeyboardInterrupt:
            logger.info("Worker shutdown initiated")

if __name__ == "__main__":
    asyncio.run(main())
```

### 2. Define the Client (`client.py`)

The client schedules the orchestration.

```python
import asyncio
import logging
import sys
import os
from azure.identity import DefaultAzureCredential
from durabletask.azuremanaged.client import DurableTaskSchedulerClient

# Configure logging
logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

async def main():
    """Main entry point for the client application."""
    logger.info("Starting Function Chaining pattern client...")
    
    # Get environment variables
    taskhub_name = os.getenv("TASKHUB", "default")
    endpoint = os.getenv("ENDPOINT", "http://localhost:8080")

    # Credential handling
    credential = None
    if endpoint != "http://localhost:8080":
        credential = DefaultAzureCredential()
    
    # Create a client
    client = DurableTaskSchedulerClient(
        host_address=endpoint, 
        secure_channel=endpoint != "http://localhost:8080",
        taskhub=taskhub_name, 
        token_credential=credential
    )
    
    # Get user input or use default name
    name = sys.argv[1] if len(sys.argv) > 1 else "User"
    
    # Schedule the orchestration
    instance_id = client.schedule_new_orchestration(
        "function_chaining_orchestrator",
        input=name
    )
    
    logger.info(f"Orchestration scheduled with ID: {instance_id}")
    
    # Wait for completion
    result = client.wait_for_orchestration_completion(
        instance_id,
        timeout=30
    )
    
    logger.info(f"Orchestration completed with result: {result.serialized_output}")

if __name__ == "__main__":
    asyncio.run(main())
```

### 3. Run the Example

1.  **Start the Emulator** (if running locally):
    ```bash
    docker run -p 8080:8080 mcr.microsoft.com/dts/dts-emulator:latest
    ```

2.  **Start the Worker**:
    ```bash
    python worker.py
    ```

3.  **Run the Client**:
    ```bash
    python client.py "World"
    ```

---

## Defining Orchestrators

### Basic Orchestrator

```python
from durabletask import task

@task.orchestrator
def order_processing_orchestrator(ctx: task.OrchestrationContext, order: dict):
    """Process an order through multiple steps."""
    
    # Step 1: Validate the order
    is_valid = yield ctx.call_activity("validate_order", input=order)
    
    if not is_valid:
        return {"success": False, "message": "Invalid order"}
    
    # Step 2: Process payment
    payment_result = yield ctx.call_activity(
        "process_payment", 
        input=order["payment"]
    )
    
    # Step 3: Ship the order
    yield ctx.call_activity("ship_order", input=order)
    
    return {
        "success": True,
        "order_id": order["id"],
        "transaction_id": payment_result["transaction_id"]
    }
```

### Generator Syntax

Orchestrators in Python use generator syntax with `yield`:

```python
@task.orchestrator
def hello_cities_orchestrator(ctx: task.OrchestrationContext, _):
    """Say hello to multiple cities."""
    
    results = []
    
    # Sequential execution
    results.append((yield ctx.call_activity("say_hello", input="Tokyo")))
    results.append((yield ctx.call_activity("say_hello", input="London")))
    results.append((yield ctx.call_activity("say_hello", input="Seattle")))
    
    return " ".join(results)
```

---

## Defining Activities

### Basic Activity

```python
from durabletask import task

@task.activity
def validate_order(ctx: task.ActivityContext, order: dict) -> bool:
    """Validate an order."""
    
    # Perform validation logic
    if not order.get("items"):
        return False
    
    if order.get("total", 0) <= 0:
        return False
    
    return True
```

### Async Activity

```python
@task.activity
async def process_payment(ctx: task.ActivityContext, payment: dict) -> dict:
    """Process payment asynchronously."""
    
    # Simulate async payment processing
    await asyncio.sleep(1)
    
    return {
        "transaction_id": f"txn_{payment['amount']}",
        "status": "completed"
    }
```

### Activity with Logging

```python
import logging

logger = logging.getLogger(__name__)

@task.activity
def ship_order(ctx: task.ActivityContext, order: dict) -> dict:
    """Ship an order and return tracking info."""
    
    logger.info(f"Shipping order {order['id']} to {order['address']}")
    
    # Perform shipping logic
    tracking_number = f"TRACK-{order['id']}-{ctx.task_id}"
    
    logger.info(f"Order shipped with tracking: {tracking_number}")
    
    return {
        "tracking_number": tracking_number,
        "carrier": "FastShip"
    }
```

---

## Using the Client

### Create a Client

```python
from durabletask.azuremanaged.client import DurableTaskSchedulerClient

client = DurableTaskSchedulerClient(
    host="your-scheduler.centralus.durabletask.io",
    secure_channel=True,
    taskhub="default"
)
```

### Schedule an Orchestration

```python
# Start a new orchestration
instance_id = await client.schedule_new_orchestration(
    orchestrator=order_processing_orchestrator,
    input={"id": "order-123", "items": [...], "total": 99.99}
)

print(f"Started orchestration: {instance_id}")
```

### Wait for Completion

```python
from datetime import timedelta

# Wait for completion with timeout
result = await client.wait_for_orchestration_completion(
    instance_id,
    timeout=timedelta(seconds=60)
)

print(f"Status: {result.runtime_status}")
print(f"Output: {result.serialized_output}")
```

### Query Status

```python
# Get orchestration status
metadata = await client.get_orchestration_metadata(instance_id)

print(f"Instance ID: {metadata.instance_id}")
print(f"Status: {metadata.runtime_status}")
print(f"Created: {metadata.created_at}")
print(f"Last Updated: {metadata.last_updated_at}")
```

### Raise an Event

```python
# Send an event to a waiting orchestration
await client.raise_orchestration_event(
    instance_id,
    event_name="ApprovalReceived",
    data={"approved": True, "approved_by": "manager@example.com"}
)
```

### Terminate an Orchestration

```python
await client.terminate_orchestration(instance_id, reason="Cancelled by user")
```

---

## Advanced Patterns

### Fan-Out/Fan-In

```python
@task.orchestrator
def fan_out_fan_in_orchestrator(ctx: task.OrchestrationContext, work_items: list):
    """Process multiple work items in parallel."""
    
    # Fan out - create tasks for all items
    parallel_tasks = []
    for item in work_items:
        task = ctx.call_activity("process_item", input=item)
        parallel_tasks.append(task)
    
    # Fan in - wait for all to complete
    results = yield task.when_all(parallel_tasks)
    
    return results
```

### Sub-Orchestrations

```python
@task.orchestrator
def parent_orchestrator(ctx: task.OrchestrationContext, order: dict):
    """Parent orchestration that calls child orchestrators."""
    
    # Call a sub-orchestration
    shipping_result = yield ctx.call_sub_orchestrator(
        orchestrator=shipping_orchestrator,
        input={"order_id": order["id"], "address": order["address"]}
    )
    
    return {
        "order_id": order["id"],
        "tracking": shipping_result["tracking_number"]
    }
```

### Durable Timers

```python
from datetime import timedelta

@task.orchestrator
def timer_orchestrator(ctx: task.OrchestrationContext, _):
    """Orchestration with a timer delay."""
    
    # Wait for 24 hours
    yield ctx.create_timer(timedelta(hours=24))
    
    # Continue after the delay
    yield ctx.call_activity("send_reminder", input="Timer completed!")
    
    return "Done"
```

### External Events

```python
from datetime import timedelta

@task.orchestrator
def approval_orchestrator(ctx: task.OrchestrationContext, request: dict):
    """Wait for human approval with timeout."""
    
    # Send approval request
    yield ctx.call_activity("send_approval_request", input=request)
    
    # Wait for approval event with 24 hour timeout
    approval_event = ctx.wait_for_external_event("ApprovalReceived")
    timeout_event = ctx.create_timer(timedelta(hours=24))
    
    # Wait for either approval or timeout
    winner = yield task.when_any([approval_event, timeout_event])
    
    if winner == approval_event:
        approval = yield approval_event
        return {"approved": approval.get("approved", False)}
    else:
        return {"approved": False, "reason": "Timeout"}
```

### Retry Policies

```python
from durabletask import task

@task.orchestrator
def retry_orchestrator(ctx: task.OrchestrationContext, input_data: dict):
    """Orchestration with retry logic."""
    
    retry_policy = task.RetryPolicy(
        max_number_of_attempts=5,
        first_retry_interval=timedelta(seconds=1),
        backoff_coefficient=2.0,
        max_retry_interval=timedelta(minutes=1)
    )
    
    result = yield ctx.call_activity(
        "unreliable_activity",
        input=input_data,
        retry_policy=retry_policy
    )
    
    return result
```

---

## Error Handling

### Try/Except in Orchestrators

```python
@task.orchestrator
def error_handling_orchestrator(ctx: task.OrchestrationContext, order: dict):
    """Handle errors gracefully."""
    
    try:
        result = yield ctx.call_activity("risky_operation", input=order)
        return {"success": True, "result": result}
    except Exception as e:
        # Log or handle the error
        yield ctx.call_activity("log_error", input=str(e))
        return {"success": False, "error": str(e)}
```

### Activity Error Handling

```python
@task.activity
def risky_operation(ctx: task.ActivityContext, data: dict):
    """Activity that might fail."""
    
    if not data.get("required_field"):
        raise ValueError("Missing required field")
    
    # Perform operation
    return {"processed": True}
```

---

## Complete Example

```python
import asyncio
from datetime import timedelta
from durabletask import task
from durabletask.azuremanaged.worker import DurableTaskSchedulerWorker, DurableTaskSchedulerWorkerOptions
from durabletask.azuremanaged.client import DurableTaskSchedulerClient

# Define activities
@task.activity
def get_work_items(ctx: task.ActivityContext, batch_id: str) -> list:
    return [
        {"id": f"{batch_id}-1", "data": "item1"},
        {"id": f"{batch_id}-2", "data": "item2"},
        {"id": f"{batch_id}-3", "data": "item3"},
    ]

@task.activity
def process_item(ctx: task.ActivityContext, item: dict) -> dict:
    return {"id": item["id"], "status": "processed"}

@task.activity
def aggregate_results(ctx: task.ActivityContext, results: list) -> dict:
    return {"total": len(results), "items": results}

# Define orchestrator
@task.orchestrator
def batch_processing_orchestrator(ctx: task.OrchestrationContext, batch_id: str):
    # Get work items
    items = yield ctx.call_activity("get_work_items", input=batch_id)
    
    # Process items in parallel (fan-out)
    tasks = [ctx.call_activity("process_item", input=item) for item in items]
    results = yield task.when_all(tasks)
    
    # Aggregate results (fan-in)
    summary = yield ctx.call_activity("aggregate_results", input=results)
    
    return summary

# Main function
async def main():
    options = DurableTaskSchedulerWorkerOptions(
        host="your-scheduler.centralus.durabletask.io",
        secure_channel=True,
        taskhub="default"
    )
    
    # Start worker in background
    async with DurableTaskSchedulerWorker(
        options=options,
        orchestrators=[batch_processing_orchestrator],
        activities=[get_work_items, process_item, aggregate_results]
    ) as worker:
        await worker.start()
        
        # Create client and start orchestration
        client = DurableTaskSchedulerClient(
            host="your-scheduler.centralus.durabletask.io",
            secure_channel=True,
            taskhub="default"
        )
        
        instance_id = await client.schedule_new_orchestration(
            orchestrator=batch_processing_orchestrator,
            input="batch-001"
        )
        
        print(f"Started: {instance_id}")
        
        # Wait for completion
        result = await client.wait_for_orchestration_completion(
            instance_id,
            timeout=timedelta(seconds=60)
        )
        
        print(f"Result: {result.serialized_output}")

if __name__ == "__main__":
    asyncio.run(main())
```

---

## Authentication

### Using Managed Identity

```python
from azure.identity import DefaultAzureCredential

credential = DefaultAzureCredential()

worker = DurableTaskSchedulerWorker(
    options=DurableTaskSchedulerWorkerOptions(
        host="your-scheduler.centralus.durabletask.io",
        secure_channel=True,
        taskhub="default",
        credential=credential
    ),
    orchestrators=[...],
    activities=[...]
)
```

### Using User-Assigned Managed Identity

```python
from azure.identity import ManagedIdentityCredential

credential = ManagedIdentityCredential(
    client_id="your-managed-identity-client-id"
)

client = DurableTaskSchedulerClient(
    host="your-scheduler.centralus.durabletask.io",
    secure_channel=True,
    taskhub="default",
    credential=credential
)
```

---

## Next Steps

- [View Python Samples →](../samples/python/index.md)
- [Deploy to Azure Container Apps →](../architecture/aca-dts.md)
- [Explore Patterns →](../patterns/index.md)

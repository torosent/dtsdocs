---
layout: default
title: Azure Functions SDK
parent: Python SDK
grand_parent: SDKs Overview
nav_order: 1
permalink: /docs/developer-guide/sdks/python/durable-functions/
---

# Azure Functions Durable Python SDK
{: .no_toc }

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

Complete reference for azure-functions-durable package.
{: .fs-6 .fw-300 }

---

## Installation

```bash
pip install azure-functions-durable
```

**requirements.txt:**

```
azure-functions
azure-functions-durable
```

---

## DFApp Class

The main application class for Durable Functions.

### Constructor

```python
from azure.durable_functions import DFApp
import azure.functions as func

app = DFApp(http_auth_level=func.AuthLevel.ANONYMOUS)
```

### Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `http_auth_level` | `func.AuthLevel` | Default auth level for HTTP triggers |

---

## Decorators

### @orchestration_trigger

Defines an orchestrator function.

```python
@app.orchestration_trigger(context_name="context")
def my_orchestrator(context: df.DurableOrchestrationContext):
    result = yield context.call_activity("activity_name", input)
    return result
```

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `context_name` | `str` | `"context"` | Name of context parameter |

### @activity_trigger

Defines an activity function.

```python
@app.activity_trigger(input_name="data")
def my_activity(data: dict) -> dict:
    return process(data)
```

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `input_name` | `str` | `"input"` | Name of input parameter |

### @entity_trigger

Defines an entity function.

```python
@app.entity_trigger(context_name="context")
def my_entity(context: df.EntityContext):
    # Entity logic
    pass
```

### @durable_client_input

Binds a durable client to an HTTP function.

```python
@app.route(route="start")
@app.durable_client_input(client_name="client")
async def http_start(req: func.HttpRequest, client):
    instance_id = await client.start_new("orchestrator")
    return client.create_check_status_response(req, instance_id)
```

| Parameter | Type | Description |
|-----------|------|-------------|
| `client_name` | `str` | Name of client parameter |
| `task_hub` | `str` | Optional task hub name |
| `connection_name` | `str` | Optional connection string name |

---

## DurableOrchestrationContext

Context object available in orchestrator functions.

### Properties

| Property | Type | Description |
|----------|------|-------------|
| `instance_id` | `str` | Orchestration instance ID |
| `parent_instance_id` | `str` | Parent orchestration ID |
| `is_replaying` | `bool` | Whether replaying history |
| `current_utc_datetime` | `datetime` | Deterministic current time |

### Methods

#### get_input()

Get the orchestration input.

```python
def my_orchestrator(context):
    input_data = context.get_input()
```

#### call_activity()

Call an activity function.

```python
# With return value
result = yield context.call_activity("activity_name", input_data)

# Without input
result = yield context.call_activity("activity_name")
```

#### call_activity_with_retry()

Call an activity with retry policy.

```python
retry_options = df.RetryOptions(
    first_retry_interval_in_milliseconds=5000,
    max_number_of_attempts=3
)

result = yield context.call_activity_with_retry(
    "activity_name",
    retry_options,
    input_data
)
```

#### call_sub_orchestrator()

Call a sub-orchestration.

```python
result = yield context.call_sub_orchestrator(
    "sub_orchestrator_name",
    input_data
)

# With specific instance ID
result = yield context.call_sub_orchestrator(
    "sub_orchestrator_name",
    input_data,
    instance_id="sub-123"
)
```

#### call_sub_orchestrator_with_retry()

Call a sub-orchestration with retry.

```python
result = yield context.call_sub_orchestrator_with_retry(
    "sub_orchestrator_name",
    retry_options,
    input_data
)
```

#### create_timer()

Create a durable timer.

```python
import datetime

fire_at = context.current_utc_datetime + datetime.timedelta(hours=1)
yield context.create_timer(fire_at)
```

#### wait_for_external_event()

Wait for an external event.

```python
# Wait indefinitely
event_data = yield context.wait_for_external_event("EventName")

# The event name is case-insensitive
```

#### task_all()

Wait for all tasks to complete.

```python
tasks = [
    context.call_activity("process", item)
    for item in items
]
results = yield context.task_all(tasks)
```

#### task_any()

Wait for any task to complete.

```python
timeout_task = context.create_timer(fire_at)
event_task = context.wait_for_external_event("ApprovalEvent")

winner = yield context.task_any([timeout_task, event_task])

if winner == event_task:
    # Event received
    timeout_task.cancel()
```

#### continue_as_new()

Restart the orchestration with new input.

```python
context.continue_as_new(new_input)
```

#### set_custom_status()

Set custom status visible in management APIs.

```python
context.set_custom_status({"progress": 50, "stage": "processing"})
```

#### call_entity()

Call an entity and wait for response.

```python
entity_id = df.EntityId("Counter", "myCounter")
value = yield context.call_entity(entity_id, "get")
```

#### signal_entity()

Signal an entity (fire-and-forget).

```python
entity_id = df.EntityId("Counter", "myCounter")
context.signal_entity(entity_id, "add", 5)
```

#### call_http()

Make a durable HTTP call.

```python
response = yield context.call_http(
    method="GET",
    uri="https://api.example.com/data"
)

# With headers and body
response = yield context.call_http(
    method="POST",
    uri="https://api.example.com/data",
    headers={"Content-Type": "application/json"},
    content=json.dumps({"key": "value"})
)
```

---

## DurableOrchestrationClient

Client for managing orchestrations.

### Methods

#### start_new()

Start a new orchestration.

```python
# With auto-generated ID
instance_id = await client.start_new("orchestrator_name", None, input_data)

# With specific ID
instance_id = await client.start_new(
    "orchestrator_name",
    instance_id="my-id",
    client_input=input_data
)
```

#### get_status()

Get orchestration status.

```python
status = await client.get_status(instance_id)

# With history
status = await client.get_status(
    instance_id,
    show_history=True,
    show_history_output=True
)
```

#### get_status_all()

Get status of all orchestrations.

```python
statuses = await client.get_status_all()
```

#### get_status_by()

Query orchestrations by filter.

```python
from datetime import datetime, timedelta

statuses = await client.get_status_by(
    created_time_from=datetime.utcnow() - timedelta(days=7),
    created_time_to=datetime.utcnow(),
    runtime_status=[
        df.OrchestrationRuntimeStatus.Running,
        df.OrchestrationRuntimeStatus.Pending
    ]
)
```

#### raise_event()

Raise an event to an orchestration.

```python
await client.raise_event(
    instance_id,
    "EventName",
    event_data
)
```

#### terminate()

Terminate an orchestration.

```python
await client.terminate(instance_id, "Reason for termination")
```

#### suspend()

Suspend an orchestration.

```python
await client.suspend(instance_id, "Reason for suspension")
```

#### resume()

Resume a suspended orchestration.

```python
await client.resume(instance_id, "Reason for resumption")
```

#### rewind()

Rewind a failed orchestration.

```python
await client.rewind(instance_id, "Reason for rewind")
```

#### purge_instance_history()

Purge orchestration history.

```python
# Single instance
await client.purge_instance_history(instance_id)
```

#### purge_instance_history_by()

Purge multiple orchestrations.

```python
result = await client.purge_instance_history_by(
    created_time_from=datetime.utcnow() - timedelta(days=30),
    runtime_status=[df.OrchestrationRuntimeStatus.Completed]
)
print(f"Purged {result.instances_deleted} instances")
```

#### create_check_status_response()

Create HTTP response with status endpoints.

```python
return client.create_check_status_response(request, instance_id)
```

#### create_http_management_payload()

Get management URLs for an orchestration.

```python
payload = client.create_http_management_payload(instance_id)
# Returns dict with statusQueryGetUri, sendEventPostUri, etc.
```

#### wait_for_completion_or_create_check_status_response()

Wait for completion or return status response.

```python
import datetime

response = await client.wait_for_completion_or_create_check_status_response(
    request,
    instance_id,
    timeout=datetime.timedelta(seconds=10)
)
```

---

## EntityContext

Context for entity functions.

### Properties

| Property | Type | Description |
|----------|------|-------------|
| `entity_name` | `str` | Entity type name |
| `entity_key` | `str` | Entity instance key |
| `entity_id` | `EntityId` | Full entity ID |
| `operation_name` | `str` | Current operation |

### Methods

#### get_state()

Get entity state.

```python
state = context.get_state(lambda: default_value)
```

#### set_state()

Set entity state.

```python
context.set_state(new_state)
```

#### get_input()

Get operation input.

```python
input_value = context.get_input()
```

#### set_result()

Set operation result.

```python
context.set_result(result_value)
```

#### signal_entity()

Signal another entity.

```python
context.signal_entity(
    df.EntityId("OtherEntity", "key"),
    "operation",
    input_data
)
```

---

## RetryOptions

Configuration for retry behavior.

### Constructor

```python
retry_options = df.RetryOptions(
    first_retry_interval_in_milliseconds=5000,
    max_number_of_attempts=3
)
```

### Properties

| Property | Type | Description |
|----------|------|-------------|
| `first_retry_interval_in_milliseconds` | `int` | Initial retry delay |
| `max_number_of_attempts` | `int` | Maximum attempts |
| `backoff_coefficient` | `float` | Exponential backoff multiplier |
| `max_retry_interval_in_milliseconds` | `int` | Maximum delay |
| `retry_timeout_in_milliseconds` | `int` | Total retry timeout |

---

## OrchestrationRuntimeStatus

| Value | Description |
|-------|-------------|
| `Running` | Currently executing |
| `Completed` | Finished successfully |
| `ContinuedAsNew` | Restarted with new input |
| `Failed` | Failed with error |
| `Canceled` | Canceled |
| `Terminated` | Manually terminated |
| `Pending` | Waiting to start |
| `Suspended` | Suspended |

---

## DurableOrchestrationStatus

Status information returned by get_status().

### Properties

| Property | Type | Description |
|----------|------|-------------|
| `instance_id` | `str` | Instance ID |
| `name` | `str` | Orchestrator name |
| `runtime_status` | `OrchestrationRuntimeStatus` | Current status |
| `created_time` | `datetime` | Creation time |
| `last_updated_time` | `datetime` | Last update time |
| `input` | `any` | Orchestration input |
| `output` | `any` | Orchestration output |
| `custom_status` | `any` | Custom status |
| `history` | `list` | Execution history |

---

## EntityId

Identifier for entity instances.

```python
entity_id = df.EntityId("EntityName", "entityKey")

# Properties
entity_id.name  # "EntityName"
entity_id.key   # "entityKey"
```

---

## Common Patterns

### Approval Workflow

```python
@app.orchestration_trigger(context_name="context")
def approval_workflow(context: df.DurableOrchestrationContext):
    request = context.get_input()
    
    # Send approval request
    yield context.call_activity("send_approval_request", request)
    
    # Wait for approval with timeout
    import datetime
    expiration = context.current_utc_datetime + datetime.timedelta(days=3)
    
    timeout_task = context.create_timer(expiration)
    approval_task = context.wait_for_external_event("ApprovalReceived")
    
    winner = yield context.task_any([timeout_task, approval_task])
    
    if winner == approval_task:
        timeout_task.cancel()
        approval = yield approval_task
        if approval["approved"]:
            yield context.call_activity("process_approval", request)
            return "Approved"
        else:
            yield context.call_activity("process_rejection", request)
            return "Rejected"
    else:
        yield context.call_activity("escalate", request)
        return "Escalated"
```

### Parallel Processing

```python
@app.orchestration_trigger(context_name="context")
def parallel_processor(context: df.DurableOrchestrationContext):
    items = context.get_input()
    
    # Create parallel tasks
    tasks = []
    for item in items:
        task = context.call_activity("process_item", item)
        tasks.append(task)
    
    # Wait for all to complete
    results = yield context.task_all(tasks)
    
    # Aggregate
    return sum(results)
```

### Monitor Pattern

```python
@app.orchestration_trigger(context_name="context")
def monitor_workflow(context: df.DurableOrchestrationContext):
    import datetime
    
    config = context.get_input()
    expiry_time = context.current_utc_datetime + datetime.timedelta(
        hours=config.get("duration_hours", 24)
    )
    
    while context.current_utc_datetime < expiry_time:
        # Check condition
        result = yield context.call_activity("check_condition", config)
        
        if result["met"]:
            yield context.call_activity("condition_met", result)
            return "Condition met"
        
        # Wait before next check
        next_check = context.current_utc_datetime + datetime.timedelta(
            minutes=config.get("polling_interval", 5)
        )
        if next_check < expiry_time:
            yield context.create_timer(next_check)
    
    return "Expired"
```

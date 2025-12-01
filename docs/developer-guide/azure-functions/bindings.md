---
layout: default
title: Bindings
parent: Azure Functions
grand_parent: Developer Reference
nav_order: 2
permalink: /docs/developer-guide/azure-functions/bindings/
---

# Durable Functions Bindings
{: .no_toc }

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

The Durable Functions extension provides trigger and client bindings for orchestrators, activities, entities, and the durable client.

---

## Overview

Durable Functions introduces four bindings:

| Binding | Type | Description |
|---------|------|-------------|
| **Orchestration Trigger** | Trigger | Executes orchestrator functions |
| **Activity Trigger** | Trigger | Executes activity functions |
| **Entity Trigger** | Trigger | Executes entity functions |
| **Durable Client** | Input | Interacts with orchestrations and entities |

---

## Orchestration Trigger

The orchestration trigger executes orchestrator functions when new instances are scheduled or existing instances receive events.

### .NET (In-Process)

```csharp
[FunctionName("HelloOrchestrator")]
public static async Task<List<string>> RunOrchestrator(
    [OrchestrationTrigger] IDurableOrchestrationContext context)
{
    var outputs = new List<string>();
    outputs.Add(await context.CallActivityAsync<string>("SayHello", "Tokyo"));
    outputs.Add(await context.CallActivityAsync<string>("SayHello", "Seattle"));
    outputs.Add(await context.CallActivityAsync<string>("SayHello", "London"));
    return outputs;
}
```

### .NET (Isolated Worker)

```csharp
[Function("HelloOrchestrator")]
public static async Task<List<string>> RunOrchestrator(
    [OrchestrationTrigger] TaskOrchestrationContext context)
{
    var outputs = new List<string>();
    outputs.Add(await context.CallActivityAsync<string>("SayHello", "Tokyo"));
    outputs.Add(await context.CallActivityAsync<string>("SayHello", "Seattle"));
    outputs.Add(await context.CallActivityAsync<string>("SayHello", "London"));
    return outputs;
}
```

### JavaScript (v4 Model)

```javascript
const df = require('durable-functions');

df.app.orchestration('helloOrchestrator', function* (context) {
    const outputs = [];
    outputs.push(yield context.df.callActivity('sayHello', 'Tokyo'));
    outputs.push(yield context.df.callActivity('sayHello', 'Seattle'));
    outputs.push(yield context.df.callActivity('sayHello', 'London'));
    return outputs;
});
```

### Python (v2 Model)

```python
import azure.functions as func
import azure.durable_functions as df

myApp = df.DFApp(http_auth_level=func.AuthLevel.ANONYMOUS)

@myApp.orchestration_trigger(context_name="context")
def hello_orchestrator(context: df.DurableOrchestrationContext):
    result1 = yield context.call_activity("say_hello", "Tokyo")
    result2 = yield context.call_activity("say_hello", "Seattle")
    result3 = yield context.call_activity("say_hello", "London")
    return [result1, result2, result3]
```

### Java

```java
@FunctionName("HelloOrchestrator")
public List<String> helloOrchestrator(
        @DurableOrchestrationTrigger(name = "ctx") TaskOrchestrationContext ctx) {
    List<String> outputs = new ArrayList<>();
    outputs.add(ctx.callActivity("SayHello", "Tokyo", String.class).await());
    outputs.add(ctx.callActivity("SayHello", "Seattle", String.class).await());
    outputs.add(ctx.callActivity("SayHello", "London", String.class).await());
    return outputs;
}
```

### PowerShell

**function.json**
```json
{
  "bindings": [
    {
      "name": "Context",
      "type": "orchestrationTrigger",
      "direction": "in"
    }
  ]
}
```

**run.ps1**
```powershell
param($Context)

$output = @()
$output += Invoke-DurableActivity -FunctionName 'SayHello' -Input 'Tokyo'
$output += Invoke-DurableActivity -FunctionName 'SayHello' -Input 'Seattle'
$output += Invoke-DurableActivity -FunctionName 'SayHello' -Input 'London'

$output
```

### Trigger Behavior

| Behavior | Description |
|----------|-------------|
| **Single-threading** | One dispatcher thread per host instance for all orchestrator executions |
| **No poison messages** | Poison message handling is not supported |
| **Message visibility** | Messages are dequeued and kept invisible; visibility renewed automatically |
| **Return values** | Serialized to JSON and stored in orchestration history |

> **Warning:** Orchestrator functions must never use any input/output bindings other than the orchestration trigger. This can cause issues with the Durable Task extension's replay behavior.

---

## Activity Trigger

The activity trigger executes activity functions called by orchestrators.

### .NET (In-Process)

```csharp
[FunctionName("SayHello")]
public static string SayHello([ActivityTrigger] string name)
{
    return $"Hello {name}!";
}
```

### .NET (Isolated Worker)

```csharp
[Function("SayHello")]
public static string SayHello([ActivityTrigger] string name)
{
    return $"Hello {name}!";
}
```

### JavaScript (v4 Model)

```javascript
const df = require('durable-functions');

df.app.activity('sayHello', {
    handler: (input) => {
        return `Hello, ${input}!`;
    },
});
```

### Python (v2 Model)

```python
@myApp.activity_trigger(input_name="name")
def say_hello(name: str) -> str:
    return f"Hello {name}!"
```

### Java

```java
@FunctionName("SayHello")
public String sayHello(@DurableActivityTrigger(name = "name") String name) {
    return String.format("Hello %s!", name);
}
```

### PowerShell

**function.json**
```json
{
  "bindings": [
    {
      "name": "name",
      "type": "activityTrigger",
      "direction": "in"
    }
  ]
}
```

**run.ps1**
```powershell
param($name)
"Hello $name!"
```

### Trigger Behavior

| Behavior | Description |
|----------|-------------|
| **Threading** | No restrictions—activities can perform any I/O operations |
| **No poison messages** | Poison message handling is not supported |
| **Message visibility** | Messages kept invisible and renewed automatically |
| **Return values** | Serialized to JSON and returned to the orchestrator |

---

## Entity Trigger

The entity trigger executes entity functions (available in Durable Functions 2.x+).

### .NET (In-Process)

```csharp
[FunctionName("Counter")]
public static void Counter([EntityTrigger] IDurableEntityContext ctx)
{
    switch (ctx.OperationName.ToLowerInvariant())
    {
        case "add":
            ctx.SetState(ctx.GetState<int>() + ctx.GetInput<int>());
            break;
        case "reset":
            ctx.SetState(0);
            break;
        case "get":
            ctx.Return(ctx.GetState<int>());
            break;
    }
}
```

### JavaScript (v4 Model)

```javascript
const df = require('durable-functions');

df.app.entity('counter', (context) => {
    const currentValue = context.df.getState(() => 0);
    switch (context.df.operationName) {
        case 'add':
            context.df.setState(currentValue + context.df.getInput());
            break;
        case 'reset':
            context.df.setState(0);
            break;
        case 'get':
            context.df.return(currentValue);
            break;
    }
});
```

### Python (v2 Model)

```python
@myApp.entity_trigger(context_name="context")
def counter_entity(context: df.DurableEntityContext):
    current_value = context.get_state(lambda: 0)
    operation = context.operation_name
    
    if operation == "add":
        amount = context.get_input()
        context.set_state(current_value + amount)
    elif operation == "reset":
        context.set_state(0)
    elif operation == "get":
        context.set_result(current_value)
```

### Trigger Behavior

| Behavior | Description |
|----------|-------------|
| **Single-threading** | Operations for a single entity are processed one at a time |
| **No poison messages** | Poison message handling is not supported |
| **State persistence** | State changes are automatically persisted after execution |
| **No return values** | Use specific APIs to save state or pass values to orchestrations |

> **Note:** Entity triggers are not yet supported in Java and PowerShell.

---

## Durable Client

The durable client binding interacts with orchestrations and entities from any function type.

### .NET (In-Process)

```csharp
[FunctionName("HttpStart")]
public static async Task<HttpResponseMessage> HttpStart(
    [HttpTrigger(AuthorizationLevel.Anonymous, "post")] HttpRequestMessage req,
    [DurableClient] IDurableOrchestrationClient starter,
    ILogger log)
{
    string instanceId = await starter.StartNewAsync("HelloOrchestrator", null);
    log.LogInformation($"Started orchestration with ID = '{instanceId}'.");
    return starter.CreateCheckStatusResponse(req, instanceId);
}
```

### .NET (Isolated Worker)

```csharp
[Function("HttpStart")]
public static async Task<HttpResponseData> HttpStart(
    [HttpTrigger(AuthorizationLevel.Anonymous, "post")] HttpRequestData req,
    [DurableClient] DurableTaskClient client,
    FunctionContext executionContext)
{
    string instanceId = await client.ScheduleNewOrchestrationInstanceAsync("HelloOrchestrator");
    return await client.CreateCheckStatusResponseAsync(req, instanceId);
}
```

### JavaScript (v4 Model)

```javascript
const { app } = require('@azure/functions');
const df = require('durable-functions');

app.http('httpStart', {
    route: 'orchestrators/{orchestratorName}',
    extraInputs: [df.input.durableClient()],
    handler: async (request, context) => {
        const client = df.getClient(context);
        const instanceId = await client.startNew(request.params.orchestratorName);
        context.log(`Started orchestration with ID = '${instanceId}'.`);
        return client.createCheckStatusResponse(request, instanceId);
    },
});
```

### Python (v2 Model)

```python
@myApp.route(route="orchestrators/{functionName}")
@myApp.durable_client_input(client_name="client")
async def http_start(req: func.HttpRequest, client: df.DurableOrchestrationClient):
    function_name = req.route_params.get('functionName')
    instance_id = await client.start_new(function_name)
    return client.create_check_status_response(req, instance_id)
```

### Java

```java
@FunctionName("HttpStart")
public HttpResponseMessage httpStart(
        @HttpTrigger(name = "req", methods = {HttpMethod.POST}) HttpRequestMessage<Optional<String>> request,
        @DurableClientInput(name = "durableContext") DurableClientContext durableContext,
        final ExecutionContext context) {
    
    DurableTaskClient client = durableContext.getClient();
    String instanceId = client.scheduleNewOrchestrationInstance("HelloOrchestrator");
    context.getLogger().info("Started orchestration with ID = " + instanceId);
    return durableContext.createCheckStatusResponse(request, instanceId);
}
```

### PowerShell

**function.json**
```json
{
  "bindings": [
    {
      "name": "Request",
      "type": "httpTrigger",
      "direction": "in",
      "methods": ["post"]
    },
    {
      "name": "starter",
      "type": "durableClient",
      "direction": "in"
    },
    {
      "name": "Response",
      "type": "http",
      "direction": "out"
    }
  ]
}
```

**run.ps1**
```powershell
param($Request, $TriggerMetadata)

$InstanceId = Start-DurableOrchestration -FunctionName 'HelloOrchestrator'
Write-Host "Started orchestration with ID = '$InstanceId'"

$Response = New-DurableOrchestrationCheckStatusResponse -Request $Request -InstanceId $InstanceId
Push-OutputBinding -Name Response -Value $Response
```

### Client Operations

| Operation | Description |
|-----------|-------------|
| `StartNew` / `ScheduleNewOrchestrationInstance` | Start a new orchestration |
| `GetStatus` / `GetInstance` | Query orchestration status |
| `Terminate` / `TerminateInstance` | Terminate a running orchestration |
| `RaiseEvent` / `RaiseEventAsync` | Send an event to a waiting orchestration |
| `Purge` / `PurgeInstanceHistory` | Delete orchestration history |
| `CreateCheckStatusResponse` | Create HTTP response with management URLs |
| `CreateHttpManagementPayload` | Get management URLs as a payload object |

---

## Binding Configuration

### function.json (v1-style)

For languages using function.json:

**Orchestration Trigger**
```json
{
  "name": "context",
  "type": "orchestrationTrigger",
  "direction": "in"
}
```

**Activity Trigger**
```json
{
  "name": "input",
  "type": "activityTrigger",
  "direction": "in"
}
```

**Entity Trigger**
```json
{
  "name": "context",
  "type": "entityTrigger",
  "direction": "in"
}
```

**Durable Client**
```json
{
  "name": "client",
  "type": "durableClient",
  "direction": "in",
  "taskHub": "MyTaskHub",
  "connectionName": "Storage"
}
```

### Optional Properties

| Property | Description |
|----------|-------------|
| `taskHub` | Override the task hub name from host.json |
| `connectionName` | App setting name for storage connection (client only) |

---

## Extension Bundle Requirements

Ensure your `host.json` includes the extension bundle:

```json
{
  "version": "2.0",
  "extensionBundle": {
    "id": "Microsoft.Azure.Functions.ExtensionBundle",
    "version": "[4.*, 5.0.0)"
  }
}
```

---

## Next Steps

- [host.json Settings →](./host-json.md)
- [HTTP API Reference →](./http-api.md)
- [Orchestrator Constraints →](../../concepts/code-constraints.md)

---
layout: default
title: Azure Functions SDK
nav_order: 1
parent: Node.js SDK
grand_parent: SDKs
permalink: /developer-guide/sdks/nodejs/durable-functions/
---

# durable-functions SDK Reference

The **durable-functions** npm package provides Durable Functions support for JavaScript and TypeScript Azure Functions applications.

## Installation

```bash
npm install durable-functions
```

For TypeScript support:
```bash
npm install --save-dev typescript @types/node
```

## Package Information

| Property | Value |
|----------|-------|
| npm package | [durable-functions](https://www.npmjs.com/package/durable-functions) |
| Current version | 3.x (for v4 programming model) |
| GitHub | [Azure/azure-functions-durable-js](https://github.com/Azure/azure-functions-durable-js) |

---

## App Namespace (df.app)

The `df.app` namespace provides methods for registering durable functions in the v4 programming model.

### df.app.orchestration()

Registers an orchestrator function.

```typescript
df.app.orchestration(name: string, handler: OrchestrationHandler): void
df.app.orchestration(name: string, options: OrchestrationOptions): void
```

**Parameters:**
- `name` - The name of the orchestrator function
- `handler` - The orchestrator generator function
- `options` - Configuration options including the handler

**Example:**
```typescript
import * as df from "durable-functions";

const myOrchestrator: df.OrchestrationHandler = function* (context) {
    const result1 = yield context.df.callActivity("Activity1", "input1");
    const result2 = yield context.df.callActivity("Activity2", result1);
    return result2;
};

df.app.orchestration("myOrchestrator", myOrchestrator);

// Or with options
df.app.orchestration("myOrchestrator", {
    handler: myOrchestrator
});
```

### df.app.activity()

Registers an activity function.

```typescript
df.app.activity(name: string, handler: ActivityHandler): void
df.app.activity(name: string, options: ActivityOptions): void
```

**Parameters:**
- `name` - The name of the activity function
- `handler` - The activity function implementation
- `options` - Configuration options including the handler

**Example:**
```typescript
import * as df from "durable-functions";

const processData: df.ActivityHandler = (input: any): any => {
    // Perform some processing
    return { processed: input, timestamp: new Date() };
};

df.app.activity("processData", { handler: processData });
```

### df.app.entity()

Registers an entity function.

```typescript
df.app.entity(name: string, handler: EntityHandler): void
df.app.entity(name: string, options: EntityOptions): void
```

**Parameters:**
- `name` - The name of the entity function
- `handler` - The entity function implementation
- `options` - Configuration options including the handler

**Example:**
```typescript
import * as df from "durable-functions";

const counterEntity: df.EntityHandler<number> = (context) => {
    const currentValue = context.df.getState(() => 0);
    
    switch (context.df.operationName) {
        case "add":
            context.df.setState(currentValue + context.df.getInput<number>());
            break;
        case "reset":
            context.df.setState(0);
            break;
        case "get":
            context.df.return(currentValue);
            break;
    }
};

df.app.entity("Counter", { handler: counterEntity });
```

---

## Input Bindings (df.input)

### df.input.durableClient()

Creates a durable client input binding for use with HTTP triggers or other client functions.

```typescript
df.input.durableClient(): DurableClientInput
```

**Example:**
```typescript
import * as df from "durable-functions";
import { app, HttpRequest, HttpResponseInit, InvocationContext } from "@azure/functions";

app.http("startOrchestration", {
    route: "start/{orchestratorName}",
    extraInputs: [df.input.durableClient()],
    handler: async (req: HttpRequest, context: InvocationContext): Promise<HttpResponseInit> => {
        const client = df.getClient(context);
        const instanceId = await client.startNew(req.params.orchestratorName);
        return client.createCheckStatusResponse(req, instanceId);
    }
});
```

---

## Orchestration Context

### OrchestrationContext

The context object passed to orchestrator functions, providing access to the `df` property with orchestration operations.

### context.df.callActivity()

Schedules an activity function for execution.

```typescript
callActivity(name: string, input?: unknown): Task<T>
```

**Parameters:**
- `name` - The name of the activity function to call
- `input` - Optional input to pass to the activity

**Returns:** A Task that resolves to the activity's return value

**Example:**
```typescript
const orchestrator: df.OrchestrationHandler = function* (context) {
    const result = yield context.df.callActivity("ProcessOrder", { orderId: "12345" });
    return result;
};
```

### context.df.callActivityWithRetry()

Schedules an activity function with automatic retry on failure.

```typescript
callActivityWithRetry(name: string, retryOptions: RetryOptions, input?: unknown): Task<T>
```

**Parameters:**
- `name` - The name of the activity function
- `retryOptions` - Retry configuration
- `input` - Optional input to pass to the activity

**Example:**
```typescript
const orchestrator: df.OrchestrationHandler = function* (context) {
    const retryOptions = new df.RetryOptions(5000, 3); // 5 second interval, 3 attempts
    retryOptions.backoffCoefficient = 2.0;
    
    const result = yield context.df.callActivityWithRetry(
        "UnreliableActivity",
        retryOptions,
        "input"
    );
    return result;
};
```

### context.df.callSubOrchestrator()

Calls another orchestrator function as a sub-orchestration.

```typescript
callSubOrchestrator(name: string, input?: unknown, instanceId?: string): Task<T>
```

**Parameters:**
- `name` - The name of the sub-orchestrator
- `input` - Optional input for the sub-orchestrator
- `instanceId` - Optional specific instance ID for the sub-orchestration

**Example:**
```typescript
const mainOrchestrator: df.OrchestrationHandler = function* (context) {
    const result = yield context.df.callSubOrchestrator("ChildOrchestrator", { data: "value" });
    return result;
};
```

### context.df.callSubOrchestratorWithRetry()

Calls a sub-orchestrator with automatic retry on failure.

```typescript
callSubOrchestratorWithRetry(
    name: string,
    retryOptions: RetryOptions,
    input?: unknown,
    instanceId?: string
): Task<T>
```

### context.df.callHttp()

Makes an HTTP call from the orchestrator with automatic polling support.

```typescript
callHttp(options: CallHttpOptions): Task<DurableHttpResponse>
```

**Parameters (v4 model options object):**
- `method` - HTTP method (GET, POST, etc.)
- `url` - The URL to call
- `body` - Optional request body
- `headers` - Optional request headers
- `enablePolling` - Whether to enable automatic polling (default: true)
- `tokenSource` - Optional managed identity token source

**Example:**
```typescript
const orchestrator: df.OrchestrationHandler = function* (context) {
    const response = yield context.df.callHttp({
        method: "POST",
        url: "https://api.example.com/process",
        body: JSON.stringify({ data: "value" }),
        headers: {
            "Content-Type": "application/json"
        },
        enablePolling: false
    });
    
    return response.content;
};
```

### context.df.waitForExternalEvent()

Waits for an external event to be raised.

```typescript
waitForExternalEvent(name: string, timeout?: number): Task<T>
```

**Parameters:**
- `name` - The name of the event to wait for
- `timeout` - Optional timeout in milliseconds

**Example:**
```typescript
const orchestrator: df.OrchestrationHandler = function* (context) {
    // Wait for approval event with 24 hour timeout
    const approvalResult = yield context.df.waitForExternalEvent("ApprovalEvent", 86400000);
    
    if (approvalResult.approved) {
        yield context.df.callActivity("ProcessApproval");
    }
    
    return approvalResult;
};
```

### context.df.createTimer()

Creates a durable timer that fires at a specified time.

```typescript
createTimer(fireAt: Date): Task<void>
```

**Parameters:**
- `fireAt` - The Date when the timer should fire

**Example:**
```typescript
const orchestrator: df.OrchestrationHandler = function* (context) {
    const deadline = new Date(context.df.currentUtcDateTime);
    deadline.setHours(deadline.getHours() + 1);
    
    yield context.df.createTimer(deadline);
    
    yield context.df.callActivity("TimedActivity");
};
```

### context.df.Task.all()

Creates a task that completes when all tasks in the array complete (fan-out/fan-in).

```typescript
Task.all(tasks: Task[]): Task<any[]>
```

**Example:**
```typescript
const orchestrator: df.OrchestrationHandler = function* (context) {
    const cities = ["Tokyo", "Seattle", "London", "Sydney"];
    
    // Fan-out: schedule all activities in parallel
    const tasks = cities.map(city => context.df.callActivity("GetWeather", city));
    
    // Fan-in: wait for all to complete
    const results = yield context.df.Task.all(tasks);
    
    return results;
};
```

### context.df.Task.any()

Creates a task that completes when any task in the array completes.

```typescript
Task.any(tasks: Task[]): Task<TaskSet>
```

**Example:**
```typescript
const orchestrator: df.OrchestrationHandler = function* (context) {
    // Create a timer for timeout
    const timeout = new Date(context.df.currentUtcDateTime);
    timeout.setMinutes(timeout.getMinutes() + 5);
    
    const timerTask = context.df.createTimer(timeout);
    const eventTask = context.df.waitForExternalEvent("Response");
    
    const winner = yield context.df.Task.any([timerTask, eventTask]);
    
    if (winner === timerTask) {
        return { status: "timeout" };
    } else {
        return { status: "received", data: eventTask.result };
    }
};
```

### context.df.callEntity()

Calls an entity operation and waits for the response.

```typescript
callEntity(entityId: EntityId, operationName: string, operationInput?: unknown): Task<T>
```

**Example:**
```typescript
const orchestrator: df.OrchestrationHandler = function* (context) {
    const entityId = new df.EntityId("Counter", "myCounter");
    
    // Call entity operation
    const currentValue = yield context.df.callEntity(entityId, "get");
    
    return currentValue;
};
```

### context.df.signalEntity()

Sends a one-way signal to an entity (fire and forget).

```typescript
signalEntity(entityId: EntityId, operationName: string, operationInput?: unknown): void
```

**Example:**
```typescript
const orchestrator: df.OrchestrationHandler = function* (context) {
    const entityId = new df.EntityId("Counter", "myCounter");
    
    // Signal entity (doesn't wait for response)
    context.df.signalEntity(entityId, "add", 5);
    
    yield context.df.callActivity("SomeActivity");
};
```

### context.df.currentUtcDateTime

Gets the current date/time in a replay-safe manner.

```typescript
currentUtcDateTime: Date
```

{: .warning }
> Always use `context.df.currentUtcDateTime` instead of `new Date()` in orchestrators for deterministic replay.

### context.df.instanceId

Gets the instance ID of the current orchestration.

```typescript
instanceId: string
```

### context.df.isReplaying

Indicates whether the orchestrator is currently replaying.

```typescript
isReplaying: boolean
```

**Example:**
```typescript
const orchestrator: df.OrchestrationHandler = function* (context) {
    if (!context.df.isReplaying) {
        console.log("First execution - not replay");
    }
    
    yield context.df.callActivity("MyActivity");
};
```

### context.df.continueAsNew()

Restarts the orchestration with new input (eternal orchestrations).

```typescript
continueAsNew(input?: unknown): void
```

**Example:**
```typescript
const eternalOrchestrator: df.OrchestrationHandler = function* (context) {
    const input = context.df.getInput<{ counter: number }>();
    
    yield context.df.callActivity("DoWork", input.counter);
    
    // Wait 1 hour before next iteration
    const nextTime = new Date(context.df.currentUtcDateTime);
    nextTime.setHours(nextTime.getHours() + 1);
    yield context.df.createTimer(nextTime);
    
    // Restart with incremented counter
    context.df.continueAsNew({ counter: input.counter + 1 });
};
```

### context.df.setCustomStatus()

Sets a custom status for the orchestration visible via status queries.

```typescript
setCustomStatus(status: unknown): void
```

**Example:**
```typescript
const orchestrator: df.OrchestrationHandler = function* (context) {
    context.df.setCustomStatus({ stage: "processing", progress: 0 });
    
    yield context.df.callActivity("Step1");
    context.df.setCustomStatus({ stage: "processing", progress: 50 });
    
    yield context.df.callActivity("Step2");
    context.df.setCustomStatus({ stage: "complete", progress: 100 });
};
```

---

## DurableClient

The client class for managing orchestration instances.

### getClient()

Gets a DurableClient instance from the function context.

```typescript
df.getClient(context: InvocationContext): DurableClient
```

### client.startNew()

Starts a new orchestration instance.

```typescript
startNew(orchestratorName: string, options?: StartNewOptions): Promise<string>
```

**Parameters:**
- `orchestratorName` - Name of the orchestrator function
- `options` - Optional configuration:
  - `instanceId` - Specific instance ID (auto-generated if not provided)
  - `input` - Input data for the orchestration

**Returns:** The instance ID of the started orchestration

**Example:**
```typescript
const client = df.getClient(context);

// Start with auto-generated instance ID
const instanceId1 = await client.startNew("MyOrchestrator", { input: { data: "value" } });

// Start with specific instance ID
const instanceId2 = await client.startNew("MyOrchestrator", {
    instanceId: "custom-id-123",
    input: { data: "value" }
});
```

### client.getStatus()

Gets the status of an orchestration instance.

```typescript
getStatus(instanceId: string, options?: GetStatusOptions): Promise<DurableOrchestrationStatus>
```

**Parameters:**
- `instanceId` - The instance ID to query
- `options` - Query options:
  - `showHistory` - Include execution history
  - `showHistoryOutput` - Include outputs in history
  - `showInput` - Include the orchestration input

**Example:**
```typescript
const client = df.getClient(context);

const status = await client.getStatus("instance-123", {
    showHistory: true,
    showHistoryOutput: true,
    showInput: true
});

console.log(`Status: ${status.runtimeStatus}`);
console.log(`Output: ${JSON.stringify(status.output)}`);
```

### client.getStatusBy()

Gets the status of multiple orchestrations matching a filter.

```typescript
getStatusBy(options: OrchestrationFilter): Promise<DurableOrchestrationStatus[]>
```

**Parameters:**
- `options` - Filter options:
  - `createdTimeFrom` - Filter by creation time (from)
  - `createdTimeTo` - Filter by creation time (to)
  - `runtimeStatus` - Filter by status (Running, Completed, Failed, etc.)

**Example:**
```typescript
const client = df.getClient(context);

const runningInstances = await client.getStatusBy({
    runtimeStatus: [df.OrchestrationRuntimeStatus.Running],
    createdTimeFrom: new Date(Date.now() - 86400000) // Last 24 hours
});
```

### client.terminate()

Terminates a running orchestration.

```typescript
terminate(instanceId: string, reason?: string): Promise<void>
```

**Example:**
```typescript
const client = df.getClient(context);
await client.terminate("instance-123", "Cancelled by user");
```

### client.suspend()

Suspends a running orchestration.

```typescript
suspend(instanceId: string, reason?: string): Promise<void>
```

### client.resume()

Resumes a suspended orchestration.

```typescript
resume(instanceId: string, reason?: string): Promise<void>
```

### client.rewind()

Rewinds a failed orchestration to retry from the last checkpoint.

```typescript
rewind(instanceId: string, reason?: string): Promise<void>
```

### client.raiseEvent()

Raises an event to a waiting orchestration.

```typescript
raiseEvent(instanceId: string, eventName: string, eventData?: unknown): Promise<void>
```

**Example:**
```typescript
const client = df.getClient(context);

await client.raiseEvent("instance-123", "ApprovalEvent", {
    approved: true,
    approvedBy: "user@example.com"
});
```

### client.purgeInstanceHistory()

Purges the history of a specific orchestration instance.

```typescript
purgeInstanceHistory(instanceId: string): Promise<PurgeHistoryResult>
```

### client.purgeInstanceHistoryBy()

Purges history for multiple orchestrations matching a filter.

```typescript
purgeInstanceHistoryBy(options: OrchestrationFilter): Promise<PurgeHistoryResult>
```

**Example:**
```typescript
const client = df.getClient(context);

// Purge completed orchestrations older than 30 days
const result = await client.purgeInstanceHistoryBy({
    createdTimeTo: new Date(Date.now() - 30 * 86400000),
    runtimeStatus: [df.OrchestrationRuntimeStatus.Completed]
});

console.log(`Purged ${result.instancesDeleted} instances`);
```

### client.createCheckStatusResponse()

Creates an HTTP response with status query URLs.

```typescript
createCheckStatusResponse(request: HttpRequest, instanceId: string): HttpResponseInit
```

**Example:**
```typescript
app.http("startOrchestration", {
    extraInputs: [df.input.durableClient()],
    handler: async (req, context) => {
        const client = df.getClient(context);
        const instanceId = await client.startNew("MyOrchestrator");
        return client.createCheckStatusResponse(req, instanceId);
    }
});
```

### client.waitForCompletionOrCreateCheckStatusResponse()

Waits for completion or returns status URLs if timeout is reached.

```typescript
waitForCompletionOrCreateCheckStatusResponse(
    request: HttpRequest,
    instanceId: string,
    timeout?: number,
    retryInterval?: number
): Promise<HttpResponseInit>
```

---

## Entity Context

### EntityContext

The context object passed to entity functions.

### context.df.getState()

Gets the current state of the entity.

```typescript
getState<T>(initializer?: () => T): T
```

### context.df.setState()

Sets the state of the entity.

```typescript
setState(state: unknown): void
```

### context.df.operationName

Gets the name of the current operation being executed.

```typescript
operationName: string
```

### context.df.getInput()

Gets the input for the current operation.

```typescript
getInput<T>(): T
```

### context.df.return()

Sets the return value for the current operation.

```typescript
return(value: unknown): void
```

### context.df.destructOnExit()

Marks the entity for deletion after the current operation.

```typescript
destructOnExit(): void
```

**Complete Entity Example:**
```typescript
interface Counter {
    value: number;
    lastModified: Date;
}

const counterEntity: df.EntityHandler<Counter> = (context) => {
    const state = context.df.getState<Counter>(() => ({ value: 0, lastModified: new Date() }));
    
    switch (context.df.operationName) {
        case "add":
            state.value += context.df.getInput<number>();
            state.lastModified = new Date();
            context.df.setState(state);
            break;
            
        case "subtract":
            state.value -= context.df.getInput<number>();
            state.lastModified = new Date();
            context.df.setState(state);
            break;
            
        case "reset":
            context.df.setState({ value: 0, lastModified: new Date() });
            break;
            
        case "get":
            context.df.return(state);
            break;
            
        case "delete":
            context.df.destructOnExit();
            break;
    }
};

df.app.entity("Counter", { handler: counterEntity });
```

---

## Types

### RetryOptions

Configuration for retry behavior.

```typescript
class RetryOptions {
    constructor(firstRetryIntervalInMilliseconds: number, maxNumberOfAttempts: number);
    
    firstRetryIntervalInMilliseconds: number;
    maxNumberOfAttempts: number;
    backoffCoefficient?: number;
    maxRetryIntervalInMilliseconds?: number;
    retryTimeoutInMilliseconds?: number;
}
```

### EntityId

Identifies a specific entity instance.

```typescript
class EntityId {
    constructor(name: string, key: string);
    
    name: string;
    key: string;
}
```

### OrchestrationRuntimeStatus

Enum representing orchestration states.

```typescript
enum OrchestrationRuntimeStatus {
    Running = "Running",
    Completed = "Completed",
    ContinuedAsNew = "ContinuedAsNew",
    Failed = "Failed",
    Canceled = "Canceled",
    Terminated = "Terminated",
    Pending = "Pending",
    Suspended = "Suspended"
}
```

### DurableOrchestrationStatus

Status information for an orchestration instance.

```typescript
interface DurableOrchestrationStatus {
    name: string;
    instanceId: string;
    createdTime: Date;
    lastUpdatedTime: Date;
    input: unknown;
    output: unknown;
    runtimeStatus: OrchestrationRuntimeStatus;
    customStatus: unknown;
    history?: HistoryEvent[];
}
```

### DurableHttpResponse

Response from an HTTP call made via `callHttp`.

```typescript
interface DurableHttpResponse {
    statusCode: number;
    content: string;
    headers: Record<string, string>;
}
```

---

## Handler Types

### OrchestrationHandler

Type for orchestrator function handlers.

```typescript
type OrchestrationHandler = (context: OrchestrationContext) => Generator<Task, unknown, unknown>;
```

### ActivityHandler

Type for activity function handlers.

```typescript
type ActivityHandler = (input: unknown) => unknown | Promise<unknown>;
```

### EntityHandler

Type for entity function handlers.

```typescript
type EntityHandler<TState = unknown> = (context: EntityContext<TState>) => void;
```

---

## Patterns

### Function Chaining

```typescript
const chainOrchestrator: df.OrchestrationHandler = function* (context) {
    const x = yield context.df.callActivity("F1", null);
    const y = yield context.df.callActivity("F2", x);
    const z = yield context.df.callActivity("F3", y);
    return yield context.df.callActivity("F4", z);
};
```

### Fan-Out/Fan-In

```typescript
const fanOutOrchestrator: df.OrchestrationHandler = function* (context) {
    const workBatch = yield context.df.callActivity("GetWorkItems", null);
    
    const tasks = workBatch.map((item: any) => 
        context.df.callActivity("ProcessItem", item)
    );
    
    const results = yield context.df.Task.all(tasks);
    
    return yield context.df.callActivity("Aggregate", results);
};
```

### Async HTTP APIs

```typescript
// Starter function
app.http("startLongRunning", {
    route: "start",
    extraInputs: [df.input.durableClient()],
    handler: async (req, context) => {
        const client = df.getClient(context);
        const instanceId = await client.startNew("LongRunningOrchestrator", { input: await req.json() });
        return client.createCheckStatusResponse(req, instanceId);
    }
});
```

### Human Interaction with Timeout

```typescript
const approvalOrchestrator: df.OrchestrationHandler = function* (context) {
    yield context.df.callActivity("RequestApproval");
    
    const timeout = new Date(context.df.currentUtcDateTime);
    timeout.setDate(timeout.getDate() + 3); // 3 day timeout
    
    const timerTask = context.df.createTimer(timeout);
    const approvalTask = context.df.waitForExternalEvent<boolean>("ApprovalResponse");
    
    const winner = yield context.df.Task.any([timerTask, approvalTask]);
    
    if (winner === approvalTask && approvalTask.result) {
        yield context.df.callActivity("ProcessApproval");
    } else {
        yield context.df.callActivity("Escalate");
    }
};
```

### Eternal Orchestration (Monitor Pattern)

```typescript
const monitorOrchestrator: df.OrchestrationHandler = function* (context) {
    const jobId = context.df.getInput<string>();
    
    const status = yield context.df.callActivity("CheckJobStatus", jobId);
    
    if (status === "complete") {
        yield context.df.callActivity("SendNotification", jobId);
        return;
    }
    
    // Wait 1 minute before checking again
    const nextCheck = new Date(context.df.currentUtcDateTime);
    nextCheck.setMinutes(nextCheck.getMinutes() + 1);
    yield context.df.createTimer(nextCheck);
    
    context.df.continueAsNew(jobId);
};
```

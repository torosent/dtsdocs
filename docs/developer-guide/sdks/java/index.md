---
layout: default
title: Java SDK
nav_order: 4
parent: SDKs
grand_parent: Developer Guide
has_children: false
permalink: /developer-guide/sdks/java/
---

# Java SDK for Durable Task Framework

Build durable workflows in Java using the Durable Task Framework SDKs. Java supports both the Azure Functions extension and the portable SDK.

## SDK Options

### Azure Functions - durabletask-azure-functions

The **durabletask-azure-functions** Maven package provides Durable Functions support for Java Azure Functions applications.

| Property | Details |
|----------|---------|
| Package | [durabletask-azure-functions](https://mvnrepository.com/artifact/com.microsoft/durabletask-azure-functions) |
| Runtime | Azure Functions 4.x |
| Java Version | 8+ |
| Extension Bundle | 4.x bundles |

### Portable SDK - durabletask-client

The **durabletask-client** Maven package provides a portable SDK that can run outside of Azure Functions with any supported backend provider.

| Property | Details |
|----------|---------|
| Package | [durabletask-client](https://mvnrepository.com/artifact/com.microsoft/durabletask-client) |
| Backend Support | Durable Task Scheduler, Azure Storage |
| Hosting | Any Java environment |

---

## Quick Comparison

| Feature | durabletask-azure-functions | durabletask-client |
|---------|----------------------------|-------------------|
| Hosting | Azure Functions only | Any Java environment |
| Backend | Via Functions | Direct configuration |
| Triggers | HTTP, Timer, Queue | Custom |
| Entities | Yes | Planned |
| Complexity | Lower (managed) | Higher (flexible) |

---

## Getting Started with Azure Functions

### Prerequisites

- Java Development Kit (JDK) 8 or later
- Apache Maven 3.0 or later
- Azure Functions Core Tools 4.0.4915 or later
- An Azure Storage account

### Maven Project Setup

Add the dependency to your `pom.xml`:

```xml
<dependencies>
    <dependency>
        <groupId>com.microsoft.azure.functions</groupId>
        <artifactId>azure-functions-java-library</artifactId>
        <version>3.0.0</version>
    </dependency>
    <dependency>
        <groupId>com.microsoft</groupId>
        <artifactId>durabletask-azure-functions</artifactId>
        <version>1.0.0</version>
    </dependency>
</dependencies>

<build>
    <plugins>
        <plugin>
            <groupId>com.microsoft.azure</groupId>
            <artifactId>azure-functions-maven-plugin</artifactId>
            <version>1.29.0</version>
        </plugin>
    </plugins>
</build>
```

### Generate Project with Maven Archetype

```bash
mvn archetype:generate \
  -DarchetypeGroupId=com.microsoft.azure \
  -DarchetypeArtifactId=azure-functions-archetype \
  -DarchetypeVersion=1.62 \
  -Dtrigger=durablefunctions
```

### Configure Storage Backend

**local.settings.json**:
```json
{
  "IsEncrypted": false,
  "Values": {
    "AzureWebJobsStorage": "<your-storage-connection-string>",
    "FUNCTIONS_WORKER_RUNTIME": "java"
  }
}
```

---

## Azure Functions API Reference

### Orchestrator Functions

Orchestrators define the workflow logic using the `@DurableOrchestrationTrigger` annotation.

```java
import com.microsoft.azure.functions.*;
import com.microsoft.azure.functions.annotation.*;
import com.microsoft.durabletask.*;
import com.microsoft.durabletask.azurefunctions.*;

public class DurableFunctions {
    
    @FunctionName("HelloOrchestrator")
    public String helloOrchestrator(
            @DurableOrchestrationTrigger(name = "ctx") TaskOrchestrationContext ctx) {
        
        String result = "";
        result += ctx.callActivity("SayHello", "Tokyo", String.class).await() + " ";
        result += ctx.callActivity("SayHello", "Seattle", String.class).await() + " ";
        result += ctx.callActivity("SayHello", "London", String.class).await();
        
        return result;
    }
}
```

### TaskOrchestrationContext Methods

| Method | Description |
|--------|-------------|
| `callActivity(name, input, returnType)` | Calls an activity function |
| `callSubOrchestrator(name, input, instanceId, returnType)` | Calls a sub-orchestrator |
| `createTimer(fireAt)` | Creates a durable timer |
| `waitForExternalEvent(name, timeout, type)` | Waits for an external event |
| `getInput(type)` | Gets the orchestration input |
| `getInstanceId()` | Gets the current instance ID |
| `getCurrentInstant()` | Gets current time (replay-safe) |
| `continueAsNew(input)` | Restarts the orchestration |
| `setCustomStatus(status)` | Sets custom status |
| `allOf(tasks)` | Waits for all tasks (fan-in) |
| `anyOf(tasks)` | Waits for any task |

### Activity Functions

Activities perform the actual work and are defined with `@DurableActivityTrigger`.

```java
@FunctionName("SayHello")
public String sayHello(
        @DurableActivityTrigger(name = "name") String name,
        final ExecutionContext context) {
    
    context.getLogger().info("Saying hello to " + name);
    return "Hello, " + name + "!";
}
```

### Client Functions (HTTP Starter)

Client functions start and manage orchestrations.

```java
@FunctionName("StartOrchestration")
public HttpResponseMessage startOrchestration(
        @HttpTrigger(name = "req", methods = {HttpMethod.POST}, 
                     authLevel = AuthorizationLevel.ANONYMOUS,
                     route = "orchestrators/{orchestratorName}")
        HttpRequestMessage<Optional<String>> request,
        @DurableClientInput(name = "durableContext") DurableClientContext durableContext,
        @BindingName("orchestratorName") String orchestratorName,
        final ExecutionContext context) {
    
    DurableTaskClient client = durableContext.getClient();
    String instanceId = client.scheduleNewOrchestrationInstance(orchestratorName);
    
    context.getLogger().info("Started orchestration with ID = " + instanceId);
    
    return durableContext.createCheckStatusResponse(request, instanceId);
}
```

### DurableTaskClient Methods

| Method | Description |
|--------|-------------|
| `scheduleNewOrchestrationInstance(name)` | Starts a new orchestration |
| `scheduleNewOrchestrationInstance(name, input)` | Starts with input |
| `scheduleNewOrchestrationInstance(name, options)` | Starts with options |
| `getInstanceMetadata(instanceId, getInputsAndOutputs)` | Gets instance status |
| `waitForInstanceStart(instanceId, timeout)` | Waits for start |
| `waitForInstanceCompletion(instanceId, timeout, getInputsAndOutputs)` | Waits for completion |
| `terminateInstance(instanceId, reason)` | Terminates instance |
| `suspendInstance(instanceId, reason)` | Suspends instance |
| `resumeInstance(instanceId, reason)` | Resumes instance |
| `raiseEvent(instanceId, eventName, eventData)` | Raises an event |
| `purgeInstance(instanceId)` | Purges history |

---

## Common Patterns

### Function Chaining

```java
@FunctionName("ChainOrchestrator")
public String chainOrchestrator(
        @DurableOrchestrationTrigger(name = "ctx") TaskOrchestrationContext ctx) {
    
    String result = ctx.callActivity("Step1", null, String.class).await();
    result = ctx.callActivity("Step2", result, String.class).await();
    result = ctx.callActivity("Step3", result, String.class).await();
    result = ctx.callActivity("Step4", result, String.class).await();
    
    return result;
}
```

### Fan-Out/Fan-In

```java
@FunctionName("FanOutFanInOrchestrator")
public List<String> fanOutFanIn(
        @DurableOrchestrationTrigger(name = "ctx") TaskOrchestrationContext ctx) {
    
    // Get work items
    List<String> workItems = ctx.callActivity("GetWorkItems", null, 
            new TypeReference<List<String>>() {}).await();
    
    // Fan-out: create parallel tasks
    List<Task<String>> parallelTasks = new ArrayList<>();
    for (String item : workItems) {
        Task<String> task = ctx.callActivity("ProcessItem", item, String.class);
        parallelTasks.add(task);
    }
    
    // Fan-in: wait for all tasks
    List<String> results = ctx.allOf(parallelTasks).await();
    
    return results;
}
```

### Timer and Timeout

```java
@FunctionName("TimerOrchestrator")
public String timerOrchestrator(
        @DurableOrchestrationTrigger(name = "ctx") TaskOrchestrationContext ctx) {
    
    // Wait for 1 hour
    Instant fireAt = ctx.getCurrentInstant().plus(Duration.ofHours(1));
    ctx.createTimer(fireAt).await();
    
    return ctx.callActivity("DelayedActivity", null, String.class).await();
}
```

### External Events with Timeout

```java
@FunctionName("ApprovalOrchestrator")
public String approvalOrchestrator(
        @DurableOrchestrationTrigger(name = "ctx") TaskOrchestrationContext ctx) {
    
    // Send approval request
    ctx.callActivity("SendApprovalRequest", null, Void.class).await();
    
    // Wait for approval with 3-day timeout
    Duration timeout = Duration.ofDays(3);
    Task<Boolean> approvalTask = ctx.waitForExternalEvent("ApprovalEvent", 
            timeout, Boolean.class);
    Task<Void> timeoutTask = ctx.createTimer(ctx.getCurrentInstant().plus(timeout));
    
    Task<?> winner = ctx.anyOf(approvalTask, timeoutTask).await();
    
    if (winner == approvalTask && approvalTask.await()) {
        ctx.callActivity("ProcessApproval", null, Void.class).await();
        return "Approved";
    } else {
        ctx.callActivity("Escalate", null, Void.class).await();
        return "Escalated";
    }
}
```

### Sub-Orchestrations

```java
@FunctionName("ParentOrchestrator")
public String parentOrchestrator(
        @DurableOrchestrationTrigger(name = "ctx") TaskOrchestrationContext ctx) {
    
    String input = ctx.getInput(String.class);
    
    // Call sub-orchestrator
    String result = ctx.callSubOrchestrator("ChildOrchestrator", input, 
            String.class).await();
    
    return "Parent received: " + result;
}

@FunctionName("ChildOrchestrator")
public String childOrchestrator(
        @DurableOrchestrationTrigger(name = "ctx") TaskOrchestrationContext ctx) {
    
    String input = ctx.getInput(String.class);
    return ctx.callActivity("ProcessInChild", input, String.class).await();
}
```

### Eternal Orchestration (Monitor Pattern)

```java
@FunctionName("MonitorOrchestrator")
public void monitorOrchestrator(
        @DurableOrchestrationTrigger(name = "ctx") TaskOrchestrationContext ctx) {
    
    String jobId = ctx.getInput(String.class);
    int pollingInterval = 60; // seconds
    Instant expiryTime = ctx.getCurrentInstant().plus(Duration.ofHours(6));
    
    while (ctx.getCurrentInstant().isBefore(expiryTime)) {
        String status = ctx.callActivity("CheckJobStatus", jobId, String.class).await();
        
        if ("Completed".equals(status)) {
            ctx.callActivity("SendNotification", jobId, Void.class).await();
            return;
        }
        
        // Wait before next check
        Instant nextCheck = ctx.getCurrentInstant().plus(Duration.ofSeconds(pollingInterval));
        ctx.createTimer(nextCheck).await();
    }
    
    ctx.callActivity("SendTimeoutNotification", jobId, Void.class).await();
}
```

---

## Portable SDK (durabletask-client)

The portable SDK allows running durable workflows outside of Azure Functions.

### Maven Dependency

```xml
<dependency>
    <groupId>com.microsoft</groupId>
    <artifactId>durabletask-client</artifactId>
    <version>1.0.0</version>
</dependency>
```

### Worker Setup

```java
import com.microsoft.durabletask.*;
import com.microsoft.durabletask.DurableTaskGrpcWorker;
import com.microsoft.durabletask.DurableTaskGrpcWorkerBuilder;

public class WorkerApp {
    public static void main(String[] args) throws Exception {
        // Create the worker
        DurableTaskGrpcWorkerBuilder builder = new DurableTaskGrpcWorkerBuilder()
                .grpcChannel(/* gRPC channel to Durable Task Scheduler */);
        
        // Register orchestrators
        builder.addOrchestrator("HelloOrchestrator", ctx -> {
            String name = ctx.getInput(String.class);
            String greeting = ctx.callActivity("SayHello", name, String.class).await();
            return greeting;
        });
        
        // Register activities
        builder.addActivity("SayHello", ctx -> {
            String name = ctx.getInput(String.class);
            return "Hello, " + name + "!";
        });
        
        // Start the worker
        DurableTaskGrpcWorker worker = builder.build();
        worker.start();
        
        System.out.println("Worker started. Press Enter to stop.");
        System.in.read();
        
        worker.stop();
    }
}
```

### Client Usage

```java
import com.microsoft.durabletask.*;
import com.microsoft.durabletask.DurableTaskClient;

public class ClientApp {
    public static void main(String[] args) throws Exception {
        // Create the client
        DurableTaskClient client = new DurableTaskGrpcClientBuilder()
                .grpcChannel(/* gRPC channel to Durable Task Scheduler */)
                .build();
        
        // Start a new orchestration
        String instanceId = client.scheduleNewOrchestrationInstance(
                "HelloOrchestrator", "World");
        
        System.out.println("Started orchestration: " + instanceId);
        
        // Wait for completion
        OrchestrationMetadata result = client.waitForInstanceCompletion(
                instanceId, Duration.ofMinutes(5), true);
        
        System.out.println("Status: " + result.getRuntimeStatus());
        System.out.println("Output: " + result.readOutputAs(String.class));
        
        client.close();
    }
}
```

---

## Entity Functions

Entity functions provide a stateful programming model for managing durable state.

```java
@FunctionName("Counter")
public void counter(
        @DurableEntityTrigger(name = "ctx") DurableEntityContext ctx) {
    
    String operation = ctx.getOperationName();
    
    switch (operation) {
        case "add":
            int addValue = ctx.getInput(Integer.class);
            int current = ctx.getState(Integer.class);
            ctx.setState(current + addValue);
            break;
            
        case "reset":
            ctx.setState(0);
            break;
            
        case "get":
            ctx.setResult(ctx.getState(Integer.class));
            break;
    }
}
```

### Calling Entities from Orchestrators

```java
@FunctionName("EntityOrchestrator")
public int entityOrchestrator(
        @DurableOrchestrationTrigger(name = "ctx") TaskOrchestrationContext ctx) {
    
    EntityInstanceId entityId = new EntityInstanceId("Counter", "myCounter");
    
    // Signal entity (fire-and-forget)
    ctx.signalEntity(entityId, "add", 5);
    
    // Call entity and wait for response
    int value = ctx.callEntity(entityId, "get", null, Integer.class).await();
    
    return value;
}
```

---

## Error Handling

### Activity Retry

```java
@FunctionName("RetryOrchestrator")
public String retryOrchestrator(
        @DurableOrchestrationTrigger(name = "ctx") TaskOrchestrationContext ctx) {
    
    RetryPolicy retryPolicy = RetryPolicy.newBuilder(3, Duration.ofSeconds(5))
            .setBackoffCoefficient(2.0)
            .setMaxRetryInterval(Duration.ofMinutes(1))
            .build();
    
    TaskOptions options = new TaskOptions(retryPolicy);
    
    return ctx.callActivity("UnreliableActivity", null, options, String.class).await();
}
```

### Exception Handling

```java
@FunctionName("ExceptionHandlingOrchestrator")
public String exceptionHandling(
        @DurableOrchestrationTrigger(name = "ctx") TaskOrchestrationContext ctx) {
    
    try {
        return ctx.callActivity("RiskyActivity", null, String.class).await();
    } catch (TaskFailedException ex) {
        // Handle the failure
        ctx.callActivity("HandleFailure", ex.getMessage(), Void.class).await();
        return "Handled failure: " + ex.getMessage();
    }
}
```

---

## Testing

### Unit Testing Activities

```java
import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;
import static org.mockito.Mockito.*;

public class ActivityTests {
    
    @Test
    public void testSayHello() {
        DurableFunctions functions = new DurableFunctions();
        ExecutionContext context = mock(ExecutionContext.class);
        when(context.getLogger()).thenReturn(mock(Logger.class));
        
        String result = functions.sayHello("World", context);
        
        assertEquals("Hello, World!", result);
    }
}
```

### Integration Testing

```java
import com.microsoft.durabletask.IntegrationTestBase;
import org.junit.jupiter.api.Test;

public class OrchestratorIntegrationTests extends IntegrationTestBase {
    
    @Test
    public void testHelloOrchestrator() throws Exception {
        // Start orchestration
        String instanceId = client.scheduleNewOrchestrationInstance("HelloOrchestrator");
        
        // Wait for completion
        OrchestrationMetadata result = client.waitForInstanceCompletion(
                instanceId, Duration.ofSeconds(30), true);
        
        assertEquals(OrchestrationRuntimeStatus.COMPLETED, result.getRuntimeStatus());
    }
}
```

---

## Build and Run

### Build the Project

```bash
mvn clean package
```

### Run Locally

```bash
mvn azure-functions:run
```

### Deploy to Azure

```bash
mvn azure-functions:deploy
```

---

## Resources

- [Maven Repository - durabletask-azure-functions](https://mvnrepository.com/artifact/com.microsoft/durabletask-azure-functions)
- [Maven Repository - durabletask-client](https://mvnrepository.com/artifact/com.microsoft/durabletask-client)
- [Java Quickstart](https://learn.microsoft.com/azure/azure-functions/durable/quickstart-java)
- [GitHub - Durable Task Java](https://github.com/microsoft/durabletask-java)

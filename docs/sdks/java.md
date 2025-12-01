---
layout: default
title: Java SDK (Preview)
parent: Durable Task SDKs
nav_order: 4
permalink: /docs/sdks/java/
---

# Java Durable Task SDK (Preview)
{: .no_toc }

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

The Java Durable Task SDK brings durable orchestration capabilities to Java applications. Note that Java SDK support is currently in preview.

---

## Status: Preview

> ⚠️ The Java SDK is in **preview** and subject to change. Use in production with caution.

---

## Installation

### Maven

```xml
<dependency>
    <groupId>com.microsoft</groupId>
    <artifactId>durabletask-client</artifactId>
    <version>1.5.1</version>
</dependency>
<dependency>
    <groupId>com.microsoft</groupId>
    <artifactId>durabletask-azure-managed</artifactId>
    <version>1.5.1</version>
</dependency>
```

### Gradle

```gradle
implementation 'com.microsoft:durabletask-client:1.5.1'
implementation 'com.microsoft:durabletask-azure-managed:1.5.1'
```

---

## Quickstart: Function Chaining

This example demonstrates the "Function Chaining" pattern using the Java SDK.

### 1. Define the Logic (`ChainingPattern.java`)

```java
package io.durabletask.samples;

import com.microsoft.durabletask.*;
import com.microsoft.durabletask.azuremanaged.DurableTaskSchedulerClientExtensions;
import com.microsoft.durabletask.azuremanaged.DurableTaskSchedulerWorkerExtensions;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.io.IOException;
import java.time.Duration;
import java.util.concurrent.TimeoutException;

public class ChainingPattern {
    private static final Logger logger = LoggerFactory.getLogger(ChainingPattern.class);

    public static void main(String[] args) throws IOException, InterruptedException, TimeoutException {
        // Configuration
        String connectionString = "Endpoint=http://localhost:8080;TaskHub=default;Authentication=None";

        // Create worker
        DurableTaskGrpcWorker worker = DurableTaskSchedulerWorkerExtensions.createWorkerBuilder(connectionString)
            .addOrchestration(new TaskOrchestrationFactory() {
                @Override
                public String getName() { return "ActivityChaining"; }

                @Override
                public TaskOrchestration create() {
                    return ctx -> {
                        String input = ctx.getInput(String.class);
                        String x = ctx.callActivity("Reverse", input, String.class).await();
                        String y = ctx.callActivity("Capitalize", x, String.class).await();
                        String z = ctx.callActivity("ReplaceWhitespace", y, String.class).await();
                        ctx.complete(z);
                    };
                }
            })
            .addActivity(new TaskActivityFactory() {
                @Override
                public String getName() { return "Reverse"; }

                @Override
                public TaskActivity create() {
                    return ctx -> {
                        String input = ctx.getInput(String.class);
                        return new StringBuilder(input).reverse().toString();
                    };
                }
            })
            .addActivity(new TaskActivityFactory() {
                @Override
                public String getName() { return "Capitalize"; }

                @Override
                public TaskActivity create() {
                    return ctx -> ctx.getInput(String.class).toUpperCase();
                }
            })
            .addActivity(new TaskActivityFactory() {
                @Override
                public String getName() { return "ReplaceWhitespace"; }

                @Override
                public TaskActivity create() {
                    return ctx -> ctx.getInput(String.class).trim().replaceAll("\\s", "-");
                }
            })
            .build();

        // Start the worker
        worker.start();

        // Create client
        DurableTaskClient client = DurableTaskSchedulerClientExtensions.createClientBuilder(connectionString).build();

        // Start orchestration
        String instanceId = client.scheduleNewOrchestrationInstance(
                "ActivityChaining",
                new NewOrchestrationInstanceOptions().setInput("Hello, world!"));
        logger.info("Started new orchestration instance: {}", instanceId);

        // Wait for completion
        OrchestrationMetadata completedInstance = client.waitForInstanceCompletion(
                instanceId,
                Duration.ofSeconds(30),
                true);
        logger.info("Orchestration completed: {}", completedInstance);
        logger.info("Output: {}", completedInstance.readOutputAs(String.class));

        // Cleanup
        worker.stop();
        System.exit(0);
    }
}
```

### 2. Run the Example

1.  **Start the Emulator**:
    ```bash
    docker run -p 8080:8080 mcr.microsoft.com/dts/dts-emulator:latest
    ```

2.  **Run the Java App**:
    ```bash
    ./gradlew run
    ```

---

## Defining Orchestrators

### Basic Orchestrator

```java
import com.microsoft.durabletask.*;

public class OrderProcessingOrchestrator implements TaskOrchestrator {
    @Override
    public Object run(TaskOrchestrationContext ctx) {
        // Get input
        Order order = ctx.getInput(Order.class);
        
        // Step 1: Validate
        boolean isValid = ctx.callActivity(
            "ValidateOrder", 
            order, 
            Boolean.class
        ).await();
        
        if (!isValid) {
            return new OrderResult(false, "Invalid order", null);
        }
        
        // Step 2: Process payment
        PaymentResult payment = ctx.callActivity(
            "ProcessPayment",
            order.getPayment(),
            PaymentResult.class
        ).await();
        
        // Step 3: Ship
        ctx.callActivity("ShipOrder", order, Void.class).await();
        
        return new OrderResult(true, order.getId(), payment.getTransactionId());
    }
}
```

### Orchestrator with Generics

```java
import com.microsoft.durabletask.*;

public class TypedOrchestrator implements TaskOrchestrator {
    @Override
    public OrderResult run(TaskOrchestrationContext ctx) {
        Order order = ctx.getInput(Order.class);
        
        // Process with type safety
        ValidationResult validation = ctx.callActivity(
            "ValidateOrder",
            order,
            ValidationResult.class
        ).await();
        
        return new OrderResult(validation.isValid(), order.getId(), null);
    }
}
```

---

## Defining Activities

### Basic Activity

```java
import com.microsoft.durabletask.*;
import java.util.logging.Logger;

public class ValidateOrderActivity implements TaskActivity<Order, Boolean> {
    private static final Logger logger = Logger.getLogger(ValidateOrderActivity.class.getName());
    
    @Override
    public Boolean run(TaskActivityContext ctx, Order order) {
        logger.info("Validating order: " + order.getId());
        
        // Validation logic
        if (order.getItems() == null || order.getItems().isEmpty()) {
            return false;
        }
        
        if (order.getTotal() <= 0) {
            return false;
        }
        
        return true;
    }
}
```

### Activity with External Services

```java
import com.microsoft.durabletask.*;

public class ProcessPaymentActivity implements TaskActivity<Payment, PaymentResult> {
    private final PaymentService paymentService;
    
    public ProcessPaymentActivity(PaymentService paymentService) {
        this.paymentService = paymentService;
    }
    
    @Override
    public PaymentResult run(TaskActivityContext ctx, Payment payment) {
        // Call external payment service
        String transactionId = paymentService.process(payment);
        
        return new PaymentResult(transactionId, "completed");
    }
}
```

---

## Using the Client

### Create a Client

```java
import com.microsoft.durabletask.*;
import com.microsoft.durabletask.azuremanaged.*;

DurableTaskSchedulerClientOptions options = new DurableTaskSchedulerClientOptions.Builder()
    .setEndpoint("your-scheduler.centralus.durabletask.io")
    .setTaskHubName("default")
    .setSecureChannel(true)
    .build();

DurableTaskClient client = new DurableTaskGrpcClientBuilder()
    .useDurableTaskScheduler(options)
    .build();
```

### Schedule an Orchestration

```java
// Start a new orchestration
Order order = new Order("order-123", items, 99.99);

String instanceId = client.scheduleNewOrchestrationInstance(
    "OrderProcessing",
    order
);

System.out.println("Started orchestration: " + instanceId);
```

### Wait for Completion

```java
import java.time.Duration;

// Wait for completion with timeout
OrchestrationMetadata metadata = client.waitForInstanceCompletion(
    instanceId,
    Duration.ofSeconds(60),
    true  // getInputsAndOutputs
);

if (metadata.getRuntimeStatus() == OrchestrationRuntimeStatus.COMPLETED) {
    OrderResult result = metadata.readOutputAs(OrderResult.class);
    System.out.println("Order completed: " + result);
}
```

### Query Status

```java
// Get orchestration status
OrchestrationMetadata metadata = client.getInstanceMetadata(instanceId, true);

System.out.println("Status: " + metadata.getRuntimeStatus());
System.out.println("Created: " + metadata.getCreatedAt());
System.out.println("Last Updated: " + metadata.getLastUpdatedAt());
```

### Raise an Event

```java
// Send an event to a waiting orchestration
client.raiseEvent(instanceId, "ApprovalReceived", new ApprovalEvent(true, "manager@example.com"));
```

### Terminate an Orchestration

```java
client.terminate(instanceId, "Cancelled by user");
```

---

## Advanced Patterns

### Fan-Out/Fan-In

```java
public class FanOutFanInOrchestrator implements TaskOrchestrator {
    @Override
    public List<WorkResult> run(TaskOrchestrationContext ctx) {
        List<WorkItem> items = ctx.getInput(new TypeReference<List<WorkItem>>() {});
        
        // Fan out - create tasks for all items
        List<Task<WorkResult>> tasks = new ArrayList<>();
        for (WorkItem item : items) {
            tasks.add(ctx.callActivity("ProcessItem", item, WorkResult.class));
        }
        
        // Fan in - wait for all to complete
        List<WorkResult> results = ctx.allOf(tasks).await();
        
        return results;
    }
}
```

### Sub-Orchestrations

```java
public class ParentOrchestrator implements TaskOrchestrator {
    @Override
    public OrderResult run(TaskOrchestrationContext ctx) {
        Order order = ctx.getInput(Order.class);
        
        // Call a sub-orchestration
        ShippingResult shippingResult = ctx.callSubOrchestrator(
            "ShippingOrchestrator",
            new ShippingRequest(order.getId(), order.getAddress()),
            ShippingResult.class
        ).await();
        
        return new OrderResult(true, order.getId(), shippingResult.getTrackingNumber());
    }
}
```

### Durable Timers

```java
import java.time.Duration;

public class TimerOrchestrator implements TaskOrchestrator {
    @Override
    public String run(TaskOrchestrationContext ctx) {
        // Wait for 24 hours
        ctx.createTimer(Duration.ofHours(24)).await();
        
        // Continue after the delay
        ctx.callActivity("SendReminder", "Timer completed!", Void.class).await();
        
        return "Done";
    }
}
```

### External Events with Timeout

```java
import java.time.Duration;

public class ApprovalOrchestrator implements TaskOrchestrator {
    @Override
    public ApprovalResult run(TaskOrchestrationContext ctx) {
        ApprovalRequest request = ctx.getInput(ApprovalRequest.class);
        
        // Send approval request
        ctx.callActivity("SendApprovalRequest", request, Void.class).await();
        
        // Wait for approval event with timeout
        Task<ApprovalEvent> approvalTask = ctx.waitForExternalEvent("ApprovalReceived", ApprovalEvent.class);
        Task<Void> timeoutTask = ctx.createTimer(Duration.ofHours(24));
        
        // Wait for either
        Task<?> winner = ctx.anyOf(approvalTask, timeoutTask).await();
        
        if (winner == approvalTask) {
            ApprovalEvent approval = approvalTask.await();
            return new ApprovalResult(approval.isApproved(), null);
        } else {
            return new ApprovalResult(false, "Timeout");
        }
    }
}
```

### Retry Policies

```java
public class RetryOrchestrator implements TaskOrchestrator {
    @Override
    public String run(TaskOrchestrationContext ctx) {
        String input = ctx.getInput(String.class);
        
        TaskOptions options = new TaskOptions(
            new RetryPolicy(
                5,  // maxNumberOfAttempts
                Duration.ofSeconds(1),  // firstRetryInterval
                2.0,  // backoffCoefficient
                Duration.ofMinutes(1),  // maxRetryInterval
                Duration.ofMinutes(5)   // retryTimeout
            )
        );
        
        String result = ctx.callActivity(
            "UnreliableActivity",
            input,
            options,
            String.class
        ).await();
        
        return result;
    }
}
```

---

## Data Models

### Define POJOs for Serialization

```java
import com.fasterxml.jackson.annotation.JsonCreator;
import com.fasterxml.jackson.annotation.JsonProperty;

public class Order {
    private final String id;
    private final List<OrderItem> items;
    private final double total;
    private final Payment payment;
    private final String address;
    
    @JsonCreator
    public Order(
        @JsonProperty("id") String id,
        @JsonProperty("items") List<OrderItem> items,
        @JsonProperty("total") double total,
        @JsonProperty("payment") Payment payment,
        @JsonProperty("address") String address
    ) {
        this.id = id;
        this.items = items;
        this.total = total;
        this.payment = payment;
        this.address = address;
    }
    
    // Getters
    public String getId() { return id; }
    public List<OrderItem> getItems() { return items; }
    public double getTotal() { return total; }
    public Payment getPayment() { return payment; }
    public String getAddress() { return address; }
}
```

---

## Complete Example

```java
import com.microsoft.durabletask.*;
import com.microsoft.durabletask.azuremanaged.*;
import java.time.Duration;
import java.util.*;

public class BatchProcessingExample {
    public static void main(String[] args) throws Exception {
        DurableTaskSchedulerClientOptions options = new DurableTaskSchedulerClientOptions.Builder()
            .setEndpoint("your-scheduler.centralus.durabletask.io")
            .setTaskHubName("default")
            .setSecureChannel(true)
            .build();
        
        // Create worker
        DurableTaskGrpcWorker worker = new DurableTaskGrpcWorkerBuilder()
            .useDurableTaskScheduler(options)
            .addOrchestrator("BatchProcessing", new BatchProcessingOrchestrator())
            .addActivity("GetWorkItems", new GetWorkItemsActivity())
            .addActivity("ProcessItem", new ProcessItemActivity())
            .addActivity("AggregateResults", new AggregateResultsActivity())
            .build();
        
        // Start worker
        worker.start();
        
        // Create client
        DurableTaskClient client = new DurableTaskGrpcClientBuilder()
            .useDurableTaskScheduler(options)
            .build();
        
        // Start orchestration
        String instanceId = client.scheduleNewOrchestrationInstance(
            "BatchProcessing",
            "batch-001"
        );
        
        System.out.println("Started: " + instanceId);
        
        // Wait for completion
        OrchestrationMetadata result = client.waitForInstanceCompletion(
            instanceId,
            Duration.ofSeconds(60),
            true
        );
        
        System.out.println("Status: " + result.getRuntimeStatus());
        System.out.println("Output: " + result.getSerializedOutput());
        
        worker.close();
    }
}

// Orchestrator
class BatchProcessingOrchestrator implements TaskOrchestrator {
    @Override
    public BatchResult run(TaskOrchestrationContext ctx) {
        String batchId = ctx.getInput(String.class);
        
        // Get work items
        List<WorkItem> items = ctx.callActivity(
            "GetWorkItems", 
            batchId, 
            new TypeReference<List<WorkItem>>() {}
        ).await();
        
        // Process in parallel
        List<Task<ProcessResult>> tasks = new ArrayList<>();
        for (WorkItem item : items) {
            tasks.add(ctx.callActivity("ProcessItem", item, ProcessResult.class));
        }
        List<ProcessResult> results = ctx.allOf(tasks).await();
        
        // Aggregate
        BatchResult summary = ctx.callActivity(
            "AggregateResults", 
            results, 
            BatchResult.class
        ).await();
        
        return summary;
    }
}

// Activities
class GetWorkItemsActivity implements TaskActivity<String, List<WorkItem>> {
    @Override
    public List<WorkItem> run(TaskActivityContext ctx, String batchId) {
        return Arrays.asList(
            new WorkItem(batchId + "-1", "data1"),
            new WorkItem(batchId + "-2", "data2"),
            new WorkItem(batchId + "-3", "data3")
        );
    }
}

class ProcessItemActivity implements TaskActivity<WorkItem, ProcessResult> {
    @Override
    public ProcessResult run(TaskActivityContext ctx, WorkItem item) {
        return new ProcessResult(item.getId(), "processed");
    }
}

class AggregateResultsActivity implements TaskActivity<List<ProcessResult>, BatchResult> {
    @Override
    public BatchResult run(TaskActivityContext ctx, List<ProcessResult> results) {
        return new BatchResult(results.size(), results);
    }
}
```

---

## Authentication

### Using Azure Identity

```java
import com.azure.identity.DefaultAzureCredentialBuilder;
import com.azure.core.credential.TokenCredential;

TokenCredential credential = new DefaultAzureCredentialBuilder().build();

DurableTaskSchedulerClientOptions options = new DurableTaskSchedulerClientOptions.Builder()
    .setEndpoint("your-scheduler.centralus.durabletask.io")
    .setTaskHubName("default")
    .setSecureChannel(true)
    .setCredential(credential)
    .build();
```

### Using Managed Identity

```java
import com.azure.identity.ManagedIdentityCredentialBuilder;

TokenCredential credential = new ManagedIdentityCredentialBuilder()
    .clientId("your-managed-identity-client-id")
    .build();

DurableTaskSchedulerClientOptions options = new DurableTaskSchedulerClientOptions.Builder()
    .setEndpoint("your-scheduler.centralus.durabletask.io")
    .setTaskHubName("default")
    .setSecureChannel(true)
    .setCredential(credential)
    .build();
```

---

## Known Limitations (Preview)

1. **Entity Functions** - Not yet supported in Java SDK
2. **Suspend/Resume** - Limited support
3. **Query APIs** - Some advanced query features may not be available
4. **Error Types** - Exception handling differs from .NET/Python

---

## Next Steps

- [View Java Samples →](../samples/java/index.md)
- [Deploy to Azure Container Apps →](../architecture/aca-dts.md)
- [Explore Patterns →](../patterns/index.md)

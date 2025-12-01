---
layout: default
title: Unit Testing
parent: .NET SDK
grand_parent: SDKs Overview
nav_order: 1
permalink: /docs/developer-guide/sdks/dotnet/unit-testing/
---

# Unit Testing Durable Functions (.NET)
{: .no_toc }

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

Comprehensive guide for unit testing Durable Functions in .NET.
{: .fs-6 .fw-300 }

---

## Overview

Unit testing is an important part of modern software development practices. Durable Functions can easily grow in complexity, so introducing unit tests helps avoid breaking changes. This guide covers testing strategies for:

- **Trigger Functions** - HTTP triggers that start orchestrations
- **Orchestrator Functions** - The workflow logic
- **Activity Functions** - Individual work items
- **Entity Functions** - Stateful entities

---

## Prerequisites

The examples in this guide use the following frameworks:

| Framework | Purpose |
|-----------|---------|
| **xUnit** | Testing framework |
| **Moq** | Mocking framework |

Install the required packages:

```bash
dotnet add package xunit
dotnet add package Moq
dotnet add package Microsoft.NET.Test.Sdk
```

---

## Interfaces for Mocking

### In-Process Model

| Interface | Purpose |
|-----------|---------|
| `IDurableOrchestrationContext` | Orchestrator function execution |
| `IDurableActivityContext` | Activity function execution |
| `IDurableEntityContext` | Entity function execution |
| `IDurableClient` | Client operations (start, status, events) |
| `IDurableOrchestrationClient` | Orchestration-specific client |
| `IDurableEntityClient` | Entity-specific client |

### Isolated Worker Model

| Class | Purpose |
|-------|---------|
| `TaskOrchestrationContext` | Orchestrator function execution |
| `FunctionContext` | Function execution context |
| `DurableTaskClient` | Client operations |
| `HttpRequestData` / `HttpResponseData` | HTTP trigger functions |

---

## Testing Orchestrator Functions

### In-Process Model

```csharp
public static class HelloSequence
{
    [FunctionName("HelloSequence")]
    public static async Task<List<string>> Run(
        [OrchestrationTrigger] IDurableOrchestrationContext context)
    {
        var outputs = new List<string>();
        
        outputs.Add(await context.CallActivityAsync<string>("SayHello", "Tokyo"));
        outputs.Add(await context.CallActivityAsync<string>("SayHello", "Seattle"));
        outputs.Add(await context.CallActivityAsync<string>("SayHello", "London"));
        
        return outputs;
    }
}
```

**Unit Test:**

```csharp
[Fact]
public async Task HelloSequence_ReturnsMultipleGreetings()
{
    // Arrange
    var contextMock = new Mock<IDurableOrchestrationContext>();
    
    contextMock
        .Setup(x => x.CallActivityAsync<string>("SayHello", "Tokyo"))
        .ReturnsAsync("Hello Tokyo!");
    contextMock
        .Setup(x => x.CallActivityAsync<string>("SayHello", "Seattle"))
        .ReturnsAsync("Hello Seattle!");
    contextMock
        .Setup(x => x.CallActivityAsync<string>("SayHello", "London"))
        .ReturnsAsync("Hello London!");
    
    // Act
    var result = await HelloSequence.Run(contextMock.Object);
    
    // Assert
    Assert.Equal(3, result.Count);
    Assert.Equal("Hello Tokyo!", result[0]);
    Assert.Equal("Hello Seattle!", result[1]);
    Assert.Equal("Hello London!", result[2]);
}
```

### Isolated Worker Model

```csharp
public class HelloCitiesOrchestration
{
    [Function(nameof(HelloCities))]
    public async Task<List<string>> HelloCities(
        [OrchestrationTrigger] TaskOrchestrationContext context)
    {
        ILogger logger = context.CreateReplaySafeLogger(nameof(HelloCities));
        logger.LogInformation("Starting orchestration");
        
        var outputs = new List<string>();
        
        outputs.Add(await context.CallActivityAsync<string>(nameof(SayHello), "Tokyo"));
        outputs.Add(await context.CallActivityAsync<string>(nameof(SayHello), "Seattle"));
        outputs.Add(await context.CallActivityAsync<string>(nameof(SayHello), "London"));
        
        return outputs;
    }
    
    [Function(nameof(SayHello))]
    public static string SayHello([ActivityTrigger] string name, FunctionContext context)
    {
        return $"Hello {name}!";
    }
}
```

**Unit Test:**

```csharp
[Fact]
public async Task HelloCities_ReturnsMultipleGreetings()
{
    // Arrange
    var contextMock = new Mock<TaskOrchestrationContext>();
    var loggerMock = new Mock<ILogger>();
    
    // Mock the replay-safe logger
    contextMock
        .Setup(x => x.CreateReplaySafeLogger(It.IsAny<string>()))
        .Returns(loggerMock.Object);
    
    // Mock activity calls
    contextMock
        .Setup(x => x.CallActivityAsync<string>(
            nameof(HelloCitiesOrchestration.SayHello), 
            "Tokyo",
            It.IsAny<TaskOptions>()))
        .ReturnsAsync("Hello Tokyo!");
    contextMock
        .Setup(x => x.CallActivityAsync<string>(
            nameof(HelloCitiesOrchestration.SayHello), 
            "Seattle",
            It.IsAny<TaskOptions>()))
        .ReturnsAsync("Hello Seattle!");
    contextMock
        .Setup(x => x.CallActivityAsync<string>(
            nameof(HelloCitiesOrchestration.SayHello), 
            "London",
            It.IsAny<TaskOptions>()))
        .ReturnsAsync("Hello London!");
    
    var orchestration = new HelloCitiesOrchestration();
    
    // Act
    var result = await orchestration.HelloCities(contextMock.Object);
    
    // Assert
    Assert.Equal(3, result.Count);
    Assert.Equal("Hello Tokyo!", result[0]);
    Assert.Equal("Hello Seattle!", result[1]);
    Assert.Equal("Hello London!", result[2]);
}
```

---

## Testing Activity Functions

Activity functions require no Durable-specific modifications to be tested since they're essentially regular functions.

### In-Process Model

```csharp
public static class SayHelloActivity
{
    [FunctionName("SayHello")]
    public static string SayHello([ActivityTrigger] string name, ILogger log)
    {
        log.LogInformation($"Saying hello to {name}.");
        return $"Hello {name}!";
    }
}
```

**Unit Test:**

```csharp
[Fact]
public void SayHello_ReturnsExpectedGreeting()
{
    // Arrange
    var loggerMock = new Mock<ILogger>();
    
    // Act
    var result = SayHelloActivity.SayHello("Tokyo", loggerMock.Object);
    
    // Assert
    Assert.Equal("Hello Tokyo!", result);
}

[Theory]
[InlineData("Tokyo", "Hello Tokyo!")]
[InlineData("Seattle", "Hello Seattle!")]
[InlineData("London", "Hello London!")]
public void SayHello_ReturnsCorrectGreeting_ForVariousInputs(
    string name, string expected)
{
    // Arrange
    var loggerMock = new Mock<ILogger>();
    
    // Act
    var result = SayHelloActivity.SayHello(name, loggerMock.Object);
    
    // Assert
    Assert.Equal(expected, result);
}
```

### Isolated Worker Model

```csharp
[Fact]
public void SayHello_ReturnsExpectedGreeting()
{
    // Arrange
    var contextMock = new Mock<FunctionContext>();
    
    // Act
    var result = HelloCitiesOrchestration.SayHello("Tokyo", contextMock.Object);
    
    // Assert
    Assert.Equal("Hello Tokyo!", result);
}
```

> **Note:** Loggers created via `FunctionContext` in activity functions aren't currently supported for mocking in unit tests.

---

## Testing Trigger Functions

### In-Process Model

```csharp
public static class HttpStart
{
    [FunctionName("HttpStart")]
    public static async Task<HttpResponseMessage> Run(
        [HttpTrigger(AuthorizationLevel.Anonymous, "post")] HttpRequestMessage req,
        [DurableClient] IDurableClient client,
        string functionName,
        ILogger log)
    {
        string instanceId = await client.StartNewAsync(functionName, null);
        log.LogInformation($"Started orchestration with ID = '{instanceId}'.");
        return client.CreateCheckStatusResponse(req, instanceId);
    }
}
```

**Unit Test:**

```csharp
[Fact]
public async Task HttpStart_ReturnsCheckStatusResponse()
{
    // Arrange
    var clientMock = new Mock<IDurableClient>();
    var loggerMock = new Mock<ILogger>();
    
    string functionName = "HelloSequence";
    string instanceId = "test-instance-id";
    
    clientMock
        .Setup(x => x.StartNewAsync(functionName, It.IsAny<object>()))
        .ReturnsAsync(instanceId);
    
    clientMock
        .Setup(x => x.CreateCheckStatusResponse(
            It.IsAny<HttpRequestMessage>(), 
            instanceId, 
            false))
        .Returns(new HttpResponseMessage(HttpStatusCode.Accepted)
        {
            Headers = { RetryAfter = new RetryConditionHeaderValue(TimeSpan.FromSeconds(10)) }
        });
    
    var request = new HttpRequestMessage
    {
        Content = new StringContent("{}", Encoding.UTF8, "application/json"),
        RequestUri = new Uri("http://localhost:7071/orchestrators/HelloSequence")
    };
    
    // Act
    var result = await HttpStart.Run(request, clientMock.Object, functionName, loggerMock.Object);
    
    // Assert
    Assert.NotNull(result.Headers.RetryAfter);
    Assert.Equal(TimeSpan.FromSeconds(10), result.Headers.RetryAfter.Delta);
}
```

### Isolated Worker Model

```csharp
public class HttpStartFunction
{
    [Function("HttpStart")]
    public async Task<HttpResponseData> Run(
        [HttpTrigger(AuthorizationLevel.Anonymous, "post", Route = "orchestrators/{functionName}")] 
        HttpRequestData req,
        [DurableClient] DurableTaskClient client,
        FunctionContext context,
        string functionName)
    {
        string instanceId = await client.ScheduleNewOrchestrationInstanceAsync(functionName);
        return await client.CreateCheckStatusResponseAsync(req, instanceId);
    }
}
```

**Unit Test:**

```csharp
[Fact]
public async Task HttpStart_ReturnsAcceptedResponse()
{
    // Arrange
    var clientMock = new Mock<DurableTaskClient>("testClient");
    var contextMock = new Mock<FunctionContext>();
    
    string functionName = "HelloCities";
    string instanceId = Guid.NewGuid().ToString();
    
    clientMock
        .Setup(x => x.ScheduleNewOrchestrationInstanceAsync(
            It.IsAny<TaskName>(),
            It.IsAny<object>(),
            It.IsAny<StartOrchestrationOptions>(),
            It.IsAny<CancellationToken>()))
        .ReturnsAsync(instanceId);
    
    var mockRequest = CreateMockHttpRequestData(contextMock.Object);
    var mockResponse = CreateMockHttpResponseData(contextMock.Object, HttpStatusCode.Accepted);
    
    clientMock
        .Setup(x => x.CreateCheckStatusResponseAsync(
            It.IsAny<HttpRequestData>(),
            instanceId,
            It.IsAny<CancellationToken>()))
        .ReturnsAsync(mockResponse);
    
    var function = new HttpStartFunction();
    
    // Act
    var result = await function.Run(mockRequest, clientMock.Object, contextMock.Object, functionName);
    
    // Assert
    Assert.Equal(HttpStatusCode.Accepted, result.StatusCode);
}

private static HttpRequestData CreateMockHttpRequestData(FunctionContext context)
{
    var mockRequest = new Mock<HttpRequestData>(context);
    mockRequest.Setup(r => r.Url).Returns(new Uri("http://localhost/api/orchestrators/test"));
    mockRequest.Setup(r => r.Headers).Returns(new HttpHeadersCollection());
    return mockRequest.Object;
}

private static HttpResponseData CreateMockHttpResponseData(
    FunctionContext context, 
    HttpStatusCode statusCode)
{
    var mockResponse = new Mock<HttpResponseData>(context);
    mockResponse.SetupGet(r => r.StatusCode).Returns(statusCode);
    mockResponse.SetupProperty(r => r.Body, new MemoryStream());
    return mockResponse.Object;
}
```

---

## Testing Entity Functions

### In-Process Model

```csharp
public class Counter
{
    [JsonProperty("value")]
    public int CurrentValue { get; set; }

    public void Add(int amount) => CurrentValue += amount;
    public void Reset() => CurrentValue = 0;
    public int Get() => CurrentValue;

    [FunctionName(nameof(Counter))]
    public static Task Run([EntityTrigger] IDurableEntityContext ctx)
        => ctx.DispatchAsync<Counter>();
}
```

**Unit Test:**

```csharp
[Fact]
public void Counter_Add_IncreasesValue()
{
    // Arrange
    var counter = new Counter { CurrentValue = 5 };
    
    // Act
    counter.Add(3);
    
    // Assert
    Assert.Equal(8, counter.CurrentValue);
}

[Fact]
public void Counter_Reset_SetsValueToZero()
{
    // Arrange
    var counter = new Counter { CurrentValue = 10 };
    
    // Act
    counter.Reset();
    
    // Assert
    Assert.Equal(0, counter.CurrentValue);
}

[Fact]
public void Counter_Get_ReturnsCurrentValue()
{
    // Arrange
    var counter = new Counter { CurrentValue = 42 };
    
    // Act
    var result = counter.Get();
    
    // Assert
    Assert.Equal(42, result);
}
```

### Testing Entity Dispatch

```csharp
[Fact]
public async Task Counter_Entity_DispatchesAddOperation()
{
    // Arrange
    var contextMock = new Mock<IDurableEntityContext>();
    
    contextMock.Setup(x => x.OperationName).Returns("Add");
    contextMock.Setup(x => x.GetInput<int>()).Returns(5);
    contextMock.Setup(x => x.GetState<int>()).Returns(10);
    
    int? capturedState = null;
    contextMock
        .Setup(x => x.SetState(It.IsAny<object>()))
        .Callback<object>(state => capturedState = (int)state);
    
    // Act - Testing function-based entity
    await CounterEntity.Run(contextMock.Object);
    
    // Assert
    Assert.Equal(15, capturedState);
}
```

---

## Mocking External Events

```csharp
[Fact]
public async Task Orchestration_WaitsForExternalEvent()
{
    // Arrange
    var contextMock = new Mock<IDurableOrchestrationContext>();
    
    // Mock waiting for approval event
    contextMock
        .Setup(x => x.WaitForExternalEvent<bool>("ApprovalEvent"))
        .ReturnsAsync(true);
    
    // Act
    var result = await ApprovalWorkflow.Run(contextMock.Object);
    
    // Assert
    Assert.True(result);
}
```

---

## Mocking Timers

```csharp
[Fact]
public async Task Orchestration_WaitsForTimer()
{
    // Arrange
    var contextMock = new Mock<IDurableOrchestrationContext>();
    var fireTime = DateTime.UtcNow.AddHours(1);
    
    contextMock
        .Setup(x => x.CurrentUtcDateTime)
        .Returns(DateTime.UtcNow);
    
    contextMock
        .Setup(x => x.CreateTimer(fireTime, It.IsAny<CancellationToken>()))
        .Returns(Task.CompletedTask);
    
    // Act & Assert - verify timer was created
    await TimerOrchestration.Run(contextMock.Object);
    
    contextMock.Verify(
        x => x.CreateTimer(fireTime, It.IsAny<CancellationToken>()), 
        Times.Once);
}
```

---

## Testing Retry Policies

```csharp
[Fact]
public async Task Orchestration_RetriesActivity()
{
    // Arrange
    var contextMock = new Mock<IDurableOrchestrationContext>();
    
    var retryOptions = new RetryOptions(
        firstRetryInterval: TimeSpan.FromSeconds(5),
        maxNumberOfAttempts: 3);
    
    contextMock
        .Setup(x => x.CallActivityWithRetryAsync<string>(
            "UnreliableActivity",
            retryOptions,
            It.IsAny<object>()))
        .ReturnsAsync("Success");
    
    // Act
    var result = await RetryOrchestration.Run(contextMock.Object);
    
    // Assert
    Assert.Equal("Success", result);
}
```

---

## Best Practices

### 1. Test Business Logic Separately

Extract complex business logic into separate classes that can be unit tested independently:

```csharp
public class OrderProcessor
{
    public decimal CalculateTotal(IEnumerable<OrderItem> items)
    {
        return items.Sum(i => i.Price * i.Quantity);
    }
}

// Unit test without any Durable Functions dependencies
[Fact]
public void CalculateTotal_SumsAllItems()
{
    var processor = new OrderProcessor();
    var items = new[]
    {
        new OrderItem { Price = 10, Quantity = 2 },
        new OrderItem { Price = 5, Quantity = 3 }
    };
    
    var result = processor.CalculateTotal(items);
    
    Assert.Equal(35, result);
}
```

### 2. Use Test Data Builders

```csharp
public class OrderBuilder
{
    private readonly Order _order = new Order();
    
    public OrderBuilder WithId(string id)
    {
        _order.Id = id;
        return this;
    }
    
    public OrderBuilder WithItems(params OrderItem[] items)
    {
        _order.Items = items.ToList();
        return this;
    }
    
    public Order Build() => _order;
}

[Fact]
public async Task ProcessOrder_ValidOrder_Succeeds()
{
    var order = new OrderBuilder()
        .WithId("order-123")
        .WithItems(new OrderItem { ProductId = "prod-1", Quantity = 2 })
        .Build();
    
    // ... test with order
}
```

### 3. Verify Method Calls

```csharp
[Fact]
public async Task Orchestration_CallsActivitiesInOrder()
{
    var contextMock = new Mock<IDurableOrchestrationContext>();
    var sequence = new MockSequence();
    
    contextMock.InSequence(sequence)
        .Setup(x => x.CallActivityAsync<string>("Step1", It.IsAny<object>()))
        .ReturnsAsync("Result1");
    
    contextMock.InSequence(sequence)
        .Setup(x => x.CallActivityAsync<string>("Step2", It.IsAny<object>()))
        .ReturnsAsync("Result2");
    
    await MyOrchestration.Run(contextMock.Object);
    
    contextMock.Verify(x => x.CallActivityAsync<string>("Step1", It.IsAny<object>()), Times.Once);
    contextMock.Verify(x => x.CallActivityAsync<string>("Step2", It.IsAny<object>()), Times.Once);
}
```

---

## Sample Code Repository

Complete sample code for unit testing is available in the official repositories:

- [In-Process Unit Tests](https://github.com/Azure/azure-functions-durable-extension/tree/main/samples/unit-tests)
- [Isolated Worker Unit Tests](https://github.com/Azure/azure-functions-durable-extension/tree/dev/samples/isolated-unit-tests)

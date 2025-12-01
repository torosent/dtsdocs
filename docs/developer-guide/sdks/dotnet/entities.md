---
layout: default
title: Durable Entities
parent: .NET SDK
grand_parent: SDKs Overview
nav_order: 2
permalink: /docs/developer-guide/sdks/dotnet/entities/
---

# Durable Entities (.NET)
{: .no_toc }

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

Developer guide for building stateful entities with Durable Functions in .NET.
{: .fs-6 .fw-300 }

---

## Overview

Durable Entities (also known as entity functions) define operations for reading and updating small pieces of state. They're ideal for the **aggregator pattern** where data needs to be aggregated from multiple sources over time.

Key characteristics:
- **Addressable** - Entities are accessed via a unique identifier (entity ID)
- **Single-threaded** - Operations on an entity are processed one at a time
- **Durable** - State is automatically persisted after each operation
- **Virtual** - Entities are created on-demand when first signaled

---

## Defining Entities

.NET supports two syntaxes for defining entities:

| Syntax | Description | Best For |
|--------|-------------|----------|
| **Class-based** | Entity state and operations in a class | Fixed operations, type safety |
| **Function-based** | Explicit operation dispatch | Dynamic operations, genericity |

---

## Class-Based Syntax

### In-Process Model

```csharp
public class Counter
{
    [JsonProperty("value")]
    public int CurrentValue { get; set; }

    public void Add(int amount) => CurrentValue += amount;
    public void Reset() => CurrentValue = 0;
    public int Get() => CurrentValue;
    public void Delete() => Entity.Current.DeleteState();

    [FunctionName(nameof(Counter))]
    public static Task Run([EntityTrigger] IDurableEntityContext ctx)
        => ctx.DispatchAsync<Counter>();
}
```

### Isolated Worker Model

```csharp
public class Counter
{
    public int CurrentValue { get; set; }

    public void Add(int amount) => CurrentValue += amount;
    public void Reset() => CurrentValue = 0;
    public int Get() => CurrentValue;

    [Function(nameof(Counter))]
    public static Task RunEntityAsync([EntityTrigger] TaskEntityDispatcher dispatcher)
    {
        return dispatcher.DispatchAsync<Counter>();
    }
}
```

### Portable SDK

```csharp
[DurableTask]
public class Counter : TaskEntity<CounterState>
{
    protected override CounterState InitializeState() => new() { Value = 0 };

    public void Add(int amount) => State.Value += amount;
    public void Reset() => State.Value = 0;
    public int Get() => State.Value;
}

public class CounterState
{
    public int Value { get; set; }
}
```

---

## Function-Based Syntax

Function-based syntax provides more control over operation dispatch:

### In-Process Model

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
        case "delete":
            ctx.DeleteState();
            break;
    }
}
```

### Isolated Worker Model

```csharp
[Function("Counter")]
public static Task Counter([EntityTrigger] TaskEntityDispatcher dispatcher)
{
    return dispatcher.DispatchAsync(operation =>
    {
        var state = operation.State.GetState(typeof(int)) as int? ?? 0;
        
        switch (operation.Name.ToLowerInvariant())
        {
            case "add":
                var amount = operation.GetInput<int>();
                operation.State.SetState(state + amount);
                break;
            case "reset":
                operation.State.SetState(0);
                break;
            case "get":
                return state;
            case "delete":
                operation.State.SetState(null);
                break;
        }
        return null;
    });
}
```

---

## Entity Context API

The entity context provides access to entity-specific functionality:

### IDurableEntityContext (In-Process)

| Member | Description |
|--------|-------------|
| `EntityName` | Name of the entity |
| `EntityKey` | Key identifying this instance |
| `EntityId` | Full entity identifier (name + key) |
| `OperationName` | Current operation being executed |
| `HasState` | Whether the entity has state |
| `GetState<T>()` | Get the current state |
| `SetState(value)` | Set the entity state |
| `DeleteState()` | Delete the entity state |
| `GetInput<T>()` | Get the operation input |
| `Return(value)` | Return a value to the caller |
| `SignalEntity(id, operation, input)` | Send a message to another entity |
| `StartNewOrchestration(name, input)` | Start an orchestration |

---

## Accessing Entities

### From Client Code

#### Signaling (Fire-and-forget)

**In-Process:**

```csharp
[FunctionName("AddToCounter")]
public static async Task AddToCounter(
    [HttpTrigger(AuthorizationLevel.Function, "post")] HttpRequest req,
    [DurableClient] IDurableEntityClient client)
{
    var entityId = new EntityId(nameof(Counter), "myCounter");
    await client.SignalEntityAsync(entityId, "Add", 5);
}
```

**Isolated Worker:**

```csharp
[Function("AddToCounter")]
public static async Task AddToCounter(
    [HttpTrigger(AuthorizationLevel.Function, "post")] HttpRequestData req,
    [DurableClient] DurableTaskClient client)
{
    var entityId = new EntityInstanceId(nameof(Counter), "myCounter");
    await client.Entities.SignalEntityAsync(entityId, "Add", 5);
}
```

#### Reading State

**In-Process:**

```csharp
[FunctionName("GetCounter")]
public static async Task<IActionResult> GetCounter(
    [HttpTrigger(AuthorizationLevel.Function, "get")] HttpRequest req,
    [DurableClient] IDurableEntityClient client,
    string counterId)
{
    var entityId = new EntityId(nameof(Counter), counterId);
    var state = await client.ReadEntityStateAsync<Counter>(entityId);
    
    if (!state.EntityExists)
    {
        return new NotFoundResult();
    }
    
    return new OkObjectResult(state.EntityState.CurrentValue);
}
```

### From Orchestrations

#### Signaling from Orchestrations

```csharp
[FunctionName("UpdateCounters")]
public static async Task UpdateCounters(
    [OrchestrationTrigger] IDurableOrchestrationContext context)
{
    var counterIds = new[] { "counter1", "counter2", "counter3" };
    
    foreach (var id in counterIds)
    {
        var entityId = new EntityId(nameof(Counter), id);
        context.SignalEntity(entityId, "Add", 1);
    }
}
```

#### Calling Entities (with return value)

```csharp
[FunctionName("SumCounters")]
public static async Task<int> SumCounters(
    [OrchestrationTrigger] IDurableOrchestrationContext context)
{
    var counterIds = new[] { "counter1", "counter2", "counter3" };
    var total = 0;
    
    foreach (var id in counterIds)
    {
        var entityId = new EntityId(nameof(Counter), id);
        var value = await context.CallEntityAsync<int>(entityId, "Get");
        total += value;
    }
    
    return total;
}
```

---

## Type-Safe Entity Access with Interfaces

Interfaces provide compile-time type checking for entity operations:

### Define Interface

```csharp
public interface ICounter
{
    void Add(int amount);
    Task Reset();
    Task<int> Get();
    void Delete();
}
```

### Implement Interface

```csharp
public class Counter : ICounter
{
    [JsonProperty("value")]
    public int CurrentValue { get; set; }

    public void Add(int amount) => CurrentValue += amount;
    public Task Reset()
    {
        CurrentValue = 0;
        return Task.CompletedTask;
    }
    public Task<int> Get() => Task.FromResult(CurrentValue);
    public void Delete() => Entity.Current.DeleteState();

    [FunctionName(nameof(Counter))]
    public static Task Run([EntityTrigger] IDurableEntityContext ctx)
        => ctx.DispatchAsync<Counter>();
}
```

### Use Interface in Client Code

```csharp
[FunctionName("ResetCounter")]
public static async Task ResetCounter(
    [HttpTrigger] HttpRequest req,
    [DurableClient] IDurableEntityClient client)
{
    var entityId = new EntityId(nameof(Counter), "myCounter");
    
    // Type-safe signal through interface proxy
    await client.SignalEntityAsync<ICounter>(entityId, proxy => proxy.Reset());
}
```

### Use Interface in Orchestrations

```csharp
[FunctionName("TypeSafeOrchestration")]
public static async Task<int> TypeSafeOrchestration(
    [OrchestrationTrigger] IDurableOrchestrationContext context)
{
    var entityId = new EntityId(nameof(Counter), "myCounter");
    
    // Create type-safe proxy
    var proxy = context.CreateEntityProxy<ICounter>(entityId);
    
    // Call operations with compile-time type safety
    proxy.Add(10);
    await proxy.Reset();
    return await proxy.Get();
}
```

---

## Entity Patterns

### Aggregator Pattern

Aggregate data from multiple sources:

```csharp
public class StockAggregator
{
    public Dictionary<string, decimal> Prices { get; set; } = new();
    public DateTime LastUpdated { get; set; }

    public void UpdatePrice(StockUpdate update)
    {
        Prices[update.Symbol] = update.Price;
        LastUpdated = DateTime.UtcNow;
    }

    public decimal GetPrice(string symbol)
    {
        return Prices.TryGetValue(symbol, out var price) ? price : 0;
    }

    public Dictionary<string, decimal> GetAllPrices() => Prices;

    [FunctionName(nameof(StockAggregator))]
    public static Task Run([EntityTrigger] IDurableEntityContext ctx)
        => ctx.DispatchAsync<StockAggregator>();
}
```

### Distributed Locking

Use entities as distributed locks:

```csharp
public class Lock
{
    public string Owner { get; set; }
    public DateTime? AcquiredAt { get; set; }

    public bool TryAcquire(string owner)
    {
        if (string.IsNullOrEmpty(Owner))
        {
            Owner = owner;
            AcquiredAt = DateTime.UtcNow;
            return true;
        }
        return Owner == owner;
    }

    public bool Release(string owner)
    {
        if (Owner == owner)
        {
            Owner = null;
            AcquiredAt = null;
            Entity.Current.DeleteState();
            return true;
        }
        return false;
    }

    [FunctionName(nameof(Lock))]
    public static Task Run([EntityTrigger] IDurableEntityContext ctx)
        => ctx.DispatchAsync<Lock>();
}
```

### Session Management

```csharp
public class UserSession
{
    public string UserId { get; set; }
    public Dictionary<string, object> Data { get; set; } = new();
    public DateTime CreatedAt { get; set; }
    public DateTime LastAccessedAt { get; set; }

    public void Create(string userId)
    {
        UserId = userId;
        CreatedAt = DateTime.UtcNow;
        LastAccessedAt = DateTime.UtcNow;
    }

    public void SetValue(KeyValuePair<string, object> item)
    {
        Data[item.Key] = item.Value;
        LastAccessedAt = DateTime.UtcNow;
    }

    public object GetValue(string key)
    {
        LastAccessedAt = DateTime.UtcNow;
        return Data.TryGetValue(key, out var value) ? value : null;
    }

    public void Expire() => Entity.Current.DeleteState();

    [FunctionName(nameof(UserSession))]
    public static Task Run([EntityTrigger] IDurableEntityContext ctx)
        => ctx.DispatchAsync<UserSession>();
}
```

---

## Signaling Entities from Entities

Entities can signal other entities:

```csharp
public class OrderEntity
{
    public string Status { get; set; }
    public List<string> Items { get; set; } = new();

    public void Complete()
    {
        Status = "Completed";
        
        // Signal inventory entity to update stock
        var inventoryId = new EntityId("Inventory", "main");
        Entity.Current.SignalEntity(inventoryId, "DecrementStock", Items);
        
        // Start a notification orchestration
        Entity.Current.StartNewOrchestration("SendOrderNotification", 
            Entity.Current.EntityId.EntityKey);
    }

    [FunctionName(nameof(OrderEntity))]
    public static Task Run([EntityTrigger] IDurableEntityContext ctx)
        => ctx.DispatchAsync<OrderEntity>();
}
```

---

## Entity Serialization

### Custom Serialization

Entities use JSON.NET for serialization. Customize with attributes:

```csharp
public class CustomEntity
{
    [JsonProperty("val")]
    public int Value { get; set; }
    
    [JsonIgnore]
    public string TemporaryData { get; set; }
    
    [JsonProperty(NullValueHandling = NullValueHandling.Ignore)]
    public string OptionalField { get; set; }
}
```

### Complex State

```csharp
public class ShoppingCart
{
    public List<CartItem> Items { get; set; } = new();
    public decimal Total => Items.Sum(i => i.Price * i.Quantity);
    
    [JsonConverter(typeof(StringEnumConverter))]
    public CartStatus Status { get; set; }
}

public class CartItem
{
    public string ProductId { get; set; }
    public string Name { get; set; }
    public decimal Price { get; set; }
    public int Quantity { get; set; }
}

public enum CartStatus
{
    Active,
    CheckedOut,
    Abandoned
}
```

---

## Querying Entities

### List All Entities

```csharp
[FunctionName("ListCounters")]
public static async Task<IActionResult> ListCounters(
    [HttpTrigger] HttpRequest req,
    [DurableClient] IDurableEntityClient client)
{
    var query = new EntityQuery
    {
        EntityName = nameof(Counter),
        FetchState = true,
        PageSize = 100
    };
    
    var result = await client.ListEntitiesAsync(query, CancellationToken.None);
    
    var counters = result.Entities.Select(e => new
    {
        Id = e.EntityId.EntityKey,
        Value = e.State?.ToObject<Counter>()?.CurrentValue
    });
    
    return new OkObjectResult(counters);
}
```

### Clean Up Entities

```csharp
[FunctionName("CleanupEntities")]
public static async Task CleanupEntities(
    [TimerTrigger("0 0 * * * *")] TimerInfo timer,
    [DurableClient] IDurableEntityClient client)
{
    var query = new EntityQuery
    {
        EntityName = nameof(UserSession),
        FetchState = true,
        LastOperationFrom = DateTime.UtcNow.AddHours(-24)
    };
    
    var result = await client.ListEntitiesAsync(query, CancellationToken.None);
    
    foreach (var entity in result.Entities)
    {
        var session = entity.State?.ToObject<UserSession>();
        if (session?.LastAccessedAt < DateTime.UtcNow.AddHours(-1))
        {
            await client.SignalEntityAsync(entity.EntityId, "Expire");
        }
    }
}
```

---

## Best Practices

### 1. Keep Entities Small

- Limit state size to improve performance
- Split large entities into multiple smaller ones
- Consider using external storage for large data

### 2. Design for Eventual Consistency

- Entities are eventually consistent
- Use orchestrations for transactional workflows
- Consider compensation patterns

### 3. Handle Missing State

```csharp
public class SafeCounter
{
    public int CurrentValue { get; set; }

    public int Get()
    {
        // State is automatically initialized if missing
        return CurrentValue;
    }

    public void Add(int amount)
    {
        CurrentValue += amount;
    }
}
```

### 4. Avoid Long-Running Operations

- Entity operations should complete quickly
- Offload heavy work to activities
- Use signaling for async communication

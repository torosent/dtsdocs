---
layout: default
title: Aggregator (Entity)
parent: Orchestration Patterns
nav_order: 6
permalink: /docs/patterns/aggregator/
---

# Aggregator Pattern (Entity Pattern)
{: .no_toc }

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

Use durable entities to maintain state that can be updated by multiple external events over time.

---

## Overview

The aggregator pattern uses durable entities to collect and aggregate data from multiple sources. Unlike orchestrations that complete after a workflow, entities persist indefinitely and can receive signals at any time.

```
┌──────────────────────────────────────────────────────────────────┐
│                       AGGREGATOR PATTERN                          │
├──────────────────────────────────────────────────────────────────┤
│                                                                   │
│   Event 1 ──────┐                                                │
│                 │                                                 │
│   Event 2 ──────┤         ┌─────────────────────┐                │
│                 ├────────►│    Durable Entity    │               │
│   Event 3 ──────┤         │                     │                │
│                 │         │  ┌───────────────┐  │                │
│   Event N ──────┘         │  │    State      │  │                │
│                           │  │               │  │                │
│                           │  │  count: 42    │  │                │
│                           │  │  sum: 1250.5  │  │                │
│      Query ──────────────►│  │  items: [...]│  │───► Response   │
│                           │  │               │  │                │
│                           │  └───────────────┘  │                │
│                           └─────────────────────┘                │
│                                                                   │
└──────────────────────────────────────────────────────────────────┘
```

---

## Use Cases

- **Shopping Cart**: Aggregate items as user adds/removes
- **Leaderboard**: Track scores from multiple players
- **IoT Aggregation**: Collect sensor readings over time
- **Order Status**: Aggregate shipment updates
- **Session State**: Maintain user session across requests

---

## Implementation

### C# (.NET) - Entity Definition

```csharp
using Microsoft.DurableTask.Entities;

public class CounterEntity : TaskEntity<int>
{
    public int CurrentValue => State;
    
    public void Add(int amount)
    {
        State += amount;
    }
    
    public void Subtract(int amount)
    {
        State -= amount;
    }
    
    public void Reset()
    {
        State = 0;
    }
    
    protected override int InitializeState()
    {
        return 0;
    }
}
```

### More Complex Entity

```csharp
public class ShoppingCartEntity : TaskEntity<ShoppingCart>
{
    public List<CartItem> Items => State.Items;
    public decimal Total => State.Total;
    
    public void AddItem(CartItem item)
    {
        State.Items.Add(item);
        State.Total += item.Price * item.Quantity;
    }
    
    public void RemoveItem(string productId)
    {
        var item = State.Items.FirstOrDefault(i => i.ProductId == productId);
        if (item != null)
        {
            State.Items.Remove(item);
            State.Total -= item.Price * item.Quantity;
        }
    }
    
    public void UpdateQuantity(string productId, int quantity)
    {
        var item = State.Items.FirstOrDefault(i => i.ProductId == productId);
        if (item != null)
        {
            State.Total -= item.Price * item.Quantity;
            item.Quantity = quantity;
            State.Total += item.Price * item.Quantity;
        }
    }
    
    public void Clear()
    {
        State = new ShoppingCart();
    }
    
    protected override ShoppingCart InitializeState()
    {
        return new ShoppingCart
        {
            Items = new List<CartItem>(),
            Total = 0
        };
    }
}

public class ShoppingCart
{
    public List<CartItem> Items { get; set; } = new();
    public decimal Total { get; set; }
}

public class CartItem
{
    public string ProductId { get; set; }
    public string Name { get; set; }
    public decimal Price { get; set; }
    public int Quantity { get; set; }
}
```

### Registering Entities

```csharp
builder.Services.AddDurableTaskWorker(options =>
{
    options.AddEntity<CounterEntity>();
    options.AddEntity<ShoppingCartEntity>();
});
```

---

## Interacting with Entities

### From an Orchestrator

```csharp
public override async Task<OrderResult> RunAsync(
    TaskOrchestrationContext context, 
    Order order)
{
    // Create entity ID
    var entityId = new EntityInstanceId(nameof(ShoppingCartEntity), order.UserId);
    
    // Get current state
    var cart = await context.Entities.GetAsync<ShoppingCartEntity>(entityId);
    
    // Signal the entity (fire-and-forget)
    await context.Entities.SignalAsync(entityId, "Clear");
    
    // Call the entity and wait for result
    var total = await context.Entities.CallAsync<decimal>(
        entityId, 
        nameof(ShoppingCartEntity.Total)
    );
    
    return new OrderResult { Total = total };
}
```

### From Client Code

```csharp
public class CartController : ControllerBase
{
    private readonly DurableTaskClient _client;
    
    [HttpPost("{userId}/items")]
    public async Task<IActionResult> AddItem(string userId, [FromBody] CartItem item)
    {
        var entityId = new EntityInstanceId(nameof(ShoppingCartEntity), userId);
        
        // Signal the entity
        await _client.Entities.SignalAsync(
            entityId,
            nameof(ShoppingCartEntity.AddItem),
            item
        );
        
        return Accepted();
    }
    
    [HttpGet("{userId}")]
    public async Task<IActionResult> GetCart(string userId)
    {
        var entityId = new EntityInstanceId(nameof(ShoppingCartEntity), userId);
        
        // Read entity state
        var state = await _client.Entities.GetAsync<ShoppingCart>(entityId);
        
        return Ok(state);
    }
    
    [HttpDelete("{userId}/items/{productId}")]
    public async Task<IActionResult> RemoveItem(string userId, string productId)
    {
        var entityId = new EntityInstanceId(nameof(ShoppingCartEntity), userId);
        
        await _client.Entities.SignalAsync(
            entityId,
            nameof(ShoppingCartEntity.RemoveItem),
            productId
        );
        
        return NoContent();
    }
}
```

---

## IoT Aggregation Example

```csharp
public class SensorAggregatorEntity : TaskEntity<SensorAggregateState>
{
    public void RecordReading(SensorReading reading)
    {
        State.ReadingCount++;
        State.Sum += reading.Value;
        State.LastReading = reading;
        State.LastUpdated = DateTime.UtcNow;
        
        if (reading.Value > State.MaxValue)
            State.MaxValue = reading.Value;
        if (reading.Value < State.MinValue)
            State.MinValue = reading.Value;
    }
    
    public SensorAggregateStats GetStats()
    {
        return new SensorAggregateStats
        {
            Count = State.ReadingCount,
            Average = State.ReadingCount > 0 ? State.Sum / State.ReadingCount : 0,
            Min = State.MinValue,
            Max = State.MaxValue,
            LastReading = State.LastReading,
            LastUpdated = State.LastUpdated
        };
    }
    
    public void Reset()
    {
        State = new SensorAggregateState();
    }
    
    protected override SensorAggregateState InitializeState()
    {
        return new SensorAggregateState
        {
            MinValue = double.MaxValue,
            MaxValue = double.MinValue
        };
    }
}

public class SensorAggregateState
{
    public int ReadingCount { get; set; }
    public double Sum { get; set; }
    public double MinValue { get; set; }
    public double MaxValue { get; set; }
    public SensorReading LastReading { get; set; }
    public DateTime LastUpdated { get; set; }
}
```

### Processing IoT Events

```csharp
// Event processor (e.g., from Event Hub)
public class IoTEventProcessor
{
    private readonly DurableTaskClient _client;
    
    public async Task ProcessEventAsync(SensorReading reading)
    {
        var entityId = new EntityInstanceId(
            nameof(SensorAggregatorEntity), 
            reading.SensorId
        );
        
        await _client.Entities.SignalAsync(
            entityId,
            nameof(SensorAggregatorEntity.RecordReading),
            reading
        );
    }
}
```

---

## Leaderboard Example

```csharp
public class LeaderboardEntity : TaskEntity<LeaderboardState>
{
    private const int MaxEntries = 100;
    
    public void UpdateScore(PlayerScore score)
    {
        var existing = State.Scores.FirstOrDefault(s => s.PlayerId == score.PlayerId);
        
        if (existing != null)
        {
            if (score.Score > existing.Score)
            {
                existing.Score = score.Score;
                existing.UpdatedAt = DateTime.UtcNow;
            }
        }
        else
        {
            State.Scores.Add(new LeaderboardEntry
            {
                PlayerId = score.PlayerId,
                PlayerName = score.PlayerName,
                Score = score.Score,
                UpdatedAt = DateTime.UtcNow
            });
        }
        
        // Keep only top N
        State.Scores = State.Scores
            .OrderByDescending(s => s.Score)
            .Take(MaxEntries)
            .ToList();
    }
    
    public List<LeaderboardEntry> GetTopScores(int count = 10)
    {
        return State.Scores.Take(count).ToList();
    }
    
    public int? GetPlayerRank(string playerId)
    {
        var index = State.Scores.FindIndex(s => s.PlayerId == playerId);
        return index >= 0 ? index + 1 : null;
    }
    
    protected override LeaderboardState InitializeState()
    {
        return new LeaderboardState { Scores = new List<LeaderboardEntry>() };
    }
}
```

---

## Entity Coordination

### Cross-Entity Transactions

```csharp
public override async Task<TransferResult> RunAsync(
    TaskOrchestrationContext context, 
    TransferRequest request)
{
    var sourceEntity = new EntityInstanceId(nameof(AccountEntity), request.SourceAccountId);
    var targetEntity = new EntityInstanceId(nameof(AccountEntity), request.TargetAccountId);
    
    // Lock both entities for atomic operation
    using (await context.Entities.LockAsync(sourceEntity, targetEntity))
    {
        // Check source balance
        var sourceBalance = await context.Entities.CallAsync<decimal>(
            sourceEntity, 
            nameof(AccountEntity.GetBalance)
        );
        
        if (sourceBalance < request.Amount)
        {
            return new TransferResult { Success = false, Error = "Insufficient funds" };
        }
        
        // Perform transfer
        await context.Entities.SignalAsync(sourceEntity, "Withdraw", request.Amount);
        await context.Entities.SignalAsync(targetEntity, "Deposit", request.Amount);
        
        return new TransferResult { Success = true };
    }
}
```

---

## Entity Lifecycle

```
┌─────────────────────────────────────────────────────────────────┐
│                      ENTITY LIFECYCLE                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   ┌─────────────┐   First Signal   ┌──────────────┐             │
│   │  Not Exists │ ─────────────────► │   Created   │             │
│   └─────────────┘                   └──────┬───────┘             │
│                                            │                     │
│                                            ▼                     │
│                                     ┌──────────────┐             │
│                     ┌───────────────│    Active    │◄──────┐     │
│                     │               └──────┬───────┘       │     │
│                     │                      │               │     │
│                Signal/Call           No Activity          Signal │
│                     │                      │               │     │
│                     │                      ▼               │     │
│                     │               ┌──────────────┐       │     │
│                     └───────────────│    Idle      │───────┘     │
│                                     └──────┬───────┘             │
│                                            │                     │
│                                       Delete/TTL                 │
│                                            │                     │
│                                            ▼                     │
│                                     ┌──────────────┐             │
│                                     │   Deleted    │             │
│                                     └──────────────┘             │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Best Practices

1. **Keep entities focused** - One responsibility per entity type
2. **Minimize state size** - Entities are serialized; keep state compact
3. **Use signals for fire-and-forget** - Calls wait for response
4. **Consider cleanup** - Implement TTL or deletion for unused entities
5. **Handle concurrency** - Entities process one operation at a time
6. **Test thoroughly** - Entity state management can be complex

---

## Related Patterns

- [External Events](external-events.md) - For orchestration-based event handling
- [Fan-Out/Fan-In](fan-out-fan-in.md) - For batch processing with aggregation
- [Function Chaining](function-chaining.md) - For stateless workflows

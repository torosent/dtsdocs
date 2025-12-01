---
layout: default
title: Data Persistence
parent: Core Concepts
nav_order: 13
permalink: /docs/concepts/data-persistence/
---

# Data Persistence & Serialization
{: .no_toc }

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

The Durable Task Framework automatically serializes and persists all data that flows through your orchestrations. Understanding how this works is essential for designing efficient and reliable workflows.

---

## What Gets Persisted

The framework automatically stores:

| Data Type | When Stored | How Used |
|-----------|-------------|----------|
| **Orchestration Input** | When instance starts | Passed to orchestrator function |
| **Orchestration Output** | When instance completes | Returned to callers |
| **Activity Inputs** | When activity scheduled | Passed to activity function |
| **Activity Outputs** | When activity completes | Used during replay |
| **Timer Data** | When timer created | Fire time and optional data |
| **External Event Data** | When event raised | Passed to waiting orchestrator |
| **Custom Status** | When set by orchestrator | Available for monitoring |
| **Exception Details** | When exception thrown | For error handling and debugging |

```
┌────────────────────────────────────────────────────────────────────────────┐
│                        DATA PERSISTENCE FLOW                               │
├────────────────────────────────────────────────────────────────────────────┤
│                                                                            │
│  Client                   Orchestrator                   Activity          │
│    │                          │                             │              │
│    │  Start(input) ─────────▶│                             │              │
│    │       └── [Serialized, stored as OrchestratorStarted]  │              │
│    │                          │                             │              │
│    │                          │  CallActivity(actInput) ───▶│              │
│    │                          │       └── [Serialized, stored as TaskScheduled]
│    │                          │                             │              │
│    │                          │◀──────── result ────────────│              │
│    │                          │       └── [Serialized, stored as TaskCompleted]
│    │                          │                             │              │
│    │◀──────── output ─────────│                             │              │
│    │       └── [Serialized, stored as OrchestratorCompleted]│              │
│    │                                                        │              │
└────────────────────────────────────────────────────────────────────────────┘
```

---

## Serialization Format

### Default: JSON

By default, the framework uses **JSON serialization** for all data:

```csharp
// This object...
var order = new Order 
{ 
    Id = "ORD-123", 
    Amount = 99.99m,
    Items = new[] { "Widget", "Gadget" }
};

// ...is serialized as:
// {"Id":"ORD-123","Amount":99.99,"Items":["Widget","Gadget"]}
```

### Serialization Libraries

| Platform | Default Library |
|----------|----------------|
| **.NET** | `System.Text.Json` |
| **Python** | `json` module |
| **Java** | Jackson |

### .NET Serialization Options

In .NET, you can customize JSON serialization:

```csharp
builder.Services.Configure<JsonSerializerOptions>(options =>
{
    options.PropertyNamingPolicy = JsonNamingPolicy.CamelCase;
    options.DefaultIgnoreCondition = JsonIgnoreCondition.WhenWritingNull;
});
```

---

## Data Types and Serialization

### ✅ Supported Types

| Type | Notes |
|------|-------|
| Primitives | `int`, `string`, `bool`, `double`, etc. |
| Collections | `List<T>`, `Dictionary<K,V>`, arrays |
| Custom Classes | Must be serializable (see below) |
| Records (C#) | Fully supported |
| Enums | Serialized as strings or integers |
| Nullable Types | Supported |
| DateTimeOffset | Recommended for time values |

### ❌ Unsupported Types

| Type | Issue | Solution |
|------|-------|----------|
| Circular References | Infinite serialization loop | Restructure data |
| Non-serializable Types | Cannot be JSON serialized | Create DTO |
| Streams | Cannot be serialized | Store in blob, pass URL |
| Database Connections | Resource handles | Recreate in activity |
| HttpClient | Resource handles | Inject via DI |

### Custom Type Example

```csharp
// ✅ Good: Simple, serializable class
public class Order
{
    public string Id { get; set; }
    public decimal Amount { get; set; }
    public List<OrderItem> Items { get; set; }
    public DateTimeOffset CreatedAt { get; set; }
}

public class OrderItem
{
    public string ProductId { get; set; }
    public int Quantity { get; set; }
    public decimal Price { get; set; }
}

// ✅ Good: Record type (C# 9+)
public record OrderInput(string CustomerId, List<string> ProductIds);
```

---

## Payload Size Considerations

### Size Limits

| Backend | Inline Limit | Large Message Handling |
|---------|--------------|----------------------|
| **Durable Task Scheduler** | ~1 MB | Auto blob offload |
| **Azure Storage** | 64 KB | Auto blob offload |
| **MSSQL** | Configurable | Stored in database |

### Large Payload Handling

When payloads exceed the inline limit, the framework automatically:

1. **Serializes** the data to JSON
2. **Stores** it in blob storage (or equivalent)
3. **Saves a reference** in the history
4. **Retrieves** it automatically during replay

```
Small Payload:                          Large Payload:
┌─────────────────────────┐             ┌─────────────────────────┐
│ History Event           │             │ History Event           │
│ └─ Data: {...json...}   │             │ └─ BlobRef: /blobs/abc  │
└─────────────────────────┘             └───────────┬─────────────┘
                                                    │
                                                    ▼
                                        ┌─────────────────────────┐
                                        │ Blob Storage            │
                                        │ └─ abc: {...large json} │
                                        └─────────────────────────┘
```

### Best Practices for Payload Size

```csharp
// ❌ Bad: Large data as input/output
public async Task<byte[]> ProcessFile(TaskOrchestrationContext context)
{
    var fileContent = context.GetInput<byte[]>();  // Could be MB!
    var result = await context.CallActivityAsync<byte[]>("Process", fileContent);
    return result;  // More MB!
}

// ✅ Good: Pass references instead
public async Task<string> ProcessFile(TaskOrchestrationContext context)
{
    var input = context.GetInput<FileProcessingRequest>();
    
    // Activities read/write to blob storage directly
    var resultBlobUrl = await context.CallActivityAsync<string>(
        "ProcessFile", 
        input.SourceBlobUrl
    );
    
    return resultBlobUrl;  // Just a URL string
}
```

---

## Versioning Serialized Types

Changing the structure of serialized types can break running orchestrations.

### ❌ Breaking Changes

| Change | Problem |
|--------|---------|
| Removing properties | Existing data has property, new code ignores it |
| Renaming properties | Old data uses old name, new code expects new name |
| Changing types | `int` → `string` breaks deserialization |
| Adding required properties | Old data doesn't have the property |

### ✅ Safe Changes

| Change | Why It's Safe |
|--------|---------------|
| Adding optional properties | Old data simply won't have the value |
| Adding default values | Missing properties get defaults |
| Making required → optional | Old data still has the value |

### Versioning Example

```csharp
// Version 1
public class OrderInput
{
    public string CustomerId { get; set; }
    public decimal Amount { get; set; }
}

// Version 2 - Safe evolution
public class OrderInput
{
    public string CustomerId { get; set; }
    public decimal Amount { get; set; }
    
    // New optional property with default
    public string? CouponCode { get; set; } = null;
    
    // New property with default value
    public string Currency { get; set; } = "USD";
}
```

---

## Exception Serialization

When activities throw exceptions, the exception details are serialized and stored:

```csharp
// Activity throws exception
[DurableTask(nameof(ProcessPayment))]
public class ProcessPayment : TaskActivity<PaymentRequest, PaymentResult>
{
    public override Task<PaymentResult> RunAsync(
        TaskActivityContext context, 
        PaymentRequest request)
    {
        throw new PaymentException("Card declined");
    }
}

// Orchestrator catches it
try
{
    await context.CallActivityAsync<PaymentResult>("ProcessPayment", request);
}
catch (TaskFailedException ex)
{
    // ex.Message contains the serialized exception message
    // ex.FailureDetails contains additional details
    logger.LogError("Payment failed: {Message}", ex.Message);
}
```

### Exception Details Stored

| Property | Description |
|----------|-------------|
| `Message` | Exception message |
| `ExceptionType` | Full type name |
| `StackTrace` | Stack trace (if available) |
| `InnerException` | Nested exception (if any) |

---

## Custom Serialization

### .NET: Custom Converters

For types that need special handling:

```csharp
public class DateOnlyConverter : JsonConverter<DateOnly>
{
    public override DateOnly Read(
        ref Utf8JsonReader reader, 
        Type typeToConvert, 
        JsonSerializerOptions options)
    {
        return DateOnly.Parse(reader.GetString()!);
    }

    public override void Write(
        Utf8JsonWriter writer, 
        DateOnly value, 
        JsonSerializerOptions options)
    {
        writer.WriteStringValue(value.ToString("yyyy-MM-dd"));
    }
}

// Register in configuration
builder.Services.Configure<JsonSerializerOptions>(options =>
{
    options.Converters.Add(new DateOnlyConverter());
});
```

### Python: Custom Serialization

```python
import json

class OrderEncoder(json.JSONEncoder):
    def default(self, obj):
        if isinstance(obj, Order):
            return {
                'id': obj.id,
                'amount': str(obj.amount),  # Decimal to string
                'created_at': obj.created_at.isoformat()
            }
        return super().default(obj)
```

---

## Performance Optimization

### 1. Keep Payloads Small

| Size | Impact |
|------|--------|
| < 1 KB | Optimal |
| 1-10 KB | Good |
| 10-100 KB | Consider optimization |
| > 100 KB | Use external storage |

### 2. Avoid Redundant Data

```csharp
// ❌ Bad: Redundant data
await context.CallActivityAsync("SendEmail", new {
    Order = fullOrder,           // Large object
    Email = fullOrder.Email,     // Already in Order
    Name = fullOrder.Name        // Already in Order
});

// ✅ Good: Minimal data
await context.CallActivityAsync("SendEmail", new {
    Email = fullOrder.Email,
    Name = fullOrder.Name,
    OrderId = fullOrder.Id       // Reference, not full object
});
```

### 3. Use Compression for Large Data

If you must pass large data, consider compression:

```csharp
// In activity - compress before returning
public byte[] CompressData(byte[] data)
{
    using var output = new MemoryStream();
    using (var gzip = new GZipStream(output, CompressionLevel.Optimal))
    {
        gzip.Write(data, 0, data.Length);
    }
    return output.ToArray();
}
```

---

## Debugging Serialization Issues

### Common Errors

| Error | Cause | Solution |
|-------|-------|----------|
| `JsonException` | Invalid JSON or type mismatch | Check type compatibility |
| `NotSupportedException` | Unsupported type | Create serializable DTO |
| Circular reference | Object references itself | Restructure data model |
| Missing data | Property removed/renamed | Use versioning strategy |

### Debugging Tips

1. **Log serialized data**: Temporarily log JSON to see what's being stored
2. **Test serialization**: Unit test your types serialize/deserialize correctly
3. **Check history**: Use dashboard to inspect stored event data
4. **Compare types**: Ensure input/output types match between deployments

```csharp
// Debug helper - log what's being serialized
var json = JsonSerializer.Serialize(myObject);
logger.LogDebug("Serialized data: {Json}", json);
```

---

## Next Steps

- [Understand State Management →](./state-management.md)
- [Learn about Code Constraints →](./code-constraints.md)
- [Explore Error Handling →](./error-handling.md)

---
layout: default
title: Code Constraints
parent: Core Concepts
nav_order: 4
permalink: /docs/concepts/code-constraints/
---

# Code Constraints
{: .no_toc }

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

Orchestrator functions have special constraints that you **must** understand to build reliable durable applications. Violating these constraints can cause failures, data corruption, or unexpected behavior.
{: .fs-6 .fw-300 }

---

## Why Constraints Matter

Orchestrator functions use **event sourcing** and **replay** to maintain their state. When an orchestrator resumes after a checkpoint (e.g., after an activity completes), the framework replays the entire function from the beginning, using stored history to skip already-completed work.

```
First Execution:                          Replay:
┌─────────────────────────────────┐      ┌─────────────────────────────────┐
│ var x = await Activity1();      │      │ var x = await Activity1();      │
│   → Actually runs Activity1     │      │   → Returns stored result       │
│ var y = await Activity2(x);     │      │ var y = await Activity2(x);     │
│   → Actually runs Activity2     │      │   → Actually runs Activity2     │
└─────────────────────────────────┘      └─────────────────────────────────┘
```

**For replay to work correctly, the orchestrator must produce the same sequence of operations every time it runs.**

---

## The Golden Rule

> **Orchestrator code must be DETERMINISTIC.**

This means: given the same inputs and history, the orchestrator must make the same decisions and call the same activities in the same order.

---

## Constraint #1: No Non-Deterministic APIs

### ❌ Forbidden APIs

| API | Problem | Solution |
|-----|---------|----------|
| `DateTime.Now` / `DateTime.UtcNow` | Returns different value on replay | Use `context.CurrentUtcDateTime` |
| `Guid.NewGuid()` | Returns different value on replay | Use `context.NewGuid()` or activity |
| `Random` | Returns different value on replay | Use activity for random values |
| `Environment.GetEnvironmentVariable()` | May change between replays | Use activity or pass as input |
| `HttpClient.GetAsync()` | Network call may return different results | Use activity |
| `File.ReadAllText()` | File contents may change | Use activity |

### Examples

```csharp
// ❌ WRONG - Non-deterministic
public async Task<string> BadOrchestrator(TaskOrchestrationContext context)
{
    var now = DateTime.UtcNow;           // Different on replay!
    var id = Guid.NewGuid();             // Different on replay!
    var random = new Random().Next();    // Different on replay!
    var config = Environment.GetEnvironmentVariable("MY_CONFIG");  // May change!
    
    // ...
}

// ✅ CORRECT - Deterministic
public async Task<string> GoodOrchestrator(TaskOrchestrationContext context)
{
    var now = context.CurrentUtcDateTime;  // Replay-safe
    var id = context.NewGuid();            // Replay-safe (if supported)
    var random = await context.CallActivityAsync<int>("GetRandomNumber", null);
    var config = context.GetInput<MyInput>().Config;  // Passed as input
    
    // ...
}
```

### Python Example

```python
# ❌ WRONG
def bad_orchestrator(context):
    now = datetime.now()  # Non-deterministic!
    id = str(uuid.uuid4())  # Non-deterministic!
    
# ✅ CORRECT
def good_orchestrator(context):
    now = context.current_utc_datetime  # Replay-safe
    id = yield context.call_activity("generate_id")  # Use activity
```

---

## Constraint #2: No Blocking Calls

### ❌ Forbidden Patterns

| Pattern | Problem | Solution |
|---------|---------|----------|
| `Thread.Sleep()` | Blocks thread, not tracked | Use `context.CreateTimer()` |
| `Task.Delay()` | Not persisted, resets on replay | Use `context.CreateTimer()` |
| `Task.Wait()` | Blocks thread | Use `await` |
| `Task.Result` | Blocks thread | Use `await` |
| `lock` statements | May cause deadlocks | Avoid shared state |

### Examples

```csharp
// ❌ WRONG - Blocking
public async Task<string> BadOrchestrator(TaskOrchestrationContext context)
{
    Thread.Sleep(5000);        // Blocks thread, not durable!
    await Task.Delay(5000);    // Not tracked, resets on replay!
    var result = SomeTask.Result;  // Blocks thread!
}

// ✅ CORRECT - Non-blocking
public async Task<string> GoodOrchestrator(TaskOrchestrationContext context)
{
    // Durable timer - survives restarts
    await context.CreateTimer(context.CurrentUtcDateTime.AddSeconds(5), CancellationToken.None);
    
    // Proper awaiting
    var result = await context.CallActivityAsync<string>("MyActivity", null);
}
```

### Python Example

```python
# ❌ WRONG
def bad_orchestrator(context):
    import time
    time.sleep(5)  # Blocks, not durable!
    
# ✅ CORRECT
def good_orchestrator(context):
    deadline = context.current_utc_datetime + timedelta(seconds=5)
    yield context.create_timer(deadline)  # Durable timer
```

---

## Constraint #3: No Direct I/O

### ❌ Forbidden Operations

| Operation | Problem | Solution |
|-----------|---------|----------|
| HTTP calls | Non-deterministic, may fail differently | Use activity |
| Database queries | Data may change | Use activity |
| File system access | Files may change | Use activity |
| Message queue operations | Non-deterministic | Use activity |
| Logging with side effects | Logs duplicate on replay | Use `context.IsReplaying` |

### Examples

```csharp
// ❌ WRONG - Direct I/O in orchestrator
public async Task<string> BadOrchestrator(TaskOrchestrationContext context)
{
    // Don't do these in an orchestrator!
    var response = await httpClient.GetAsync("https://api.example.com");
    var data = await database.QueryAsync("SELECT * FROM orders");
    var content = File.ReadAllText("config.json");
}

// ✅ CORRECT - I/O in activities
public async Task<string> GoodOrchestrator(TaskOrchestrationContext context)
{
    var apiResult = await context.CallActivityAsync<string>("CallExternalApi", null);
    var dbResult = await context.CallActivityAsync<List<Order>>("QueryOrders", null);
    var config = await context.CallActivityAsync<Config>("LoadConfig", null);
}
```

---

## Constraint #4: No Infinite Loops Without ContinueAsNew

### Problem

Long-running or eternal orchestrations accumulate history. If an orchestration runs forever with a normal loop, the history grows unbounded, causing:
- Slower replay times
- Higher storage costs
- Eventually, memory issues

### Solution: ContinueAsNew

Use `ContinueAsNew` to restart the orchestration with fresh history:

```csharp
// ❌ WRONG - Infinite history growth
public async Task EternalOrchestrator(TaskOrchestrationContext context)
{
    while (true)  // History grows forever!
    {
        await context.CallActivityAsync("DoWork", null);
        await context.CreateTimer(context.CurrentUtcDateTime.AddMinutes(5), default);
    }
}

// ✅ CORRECT - Fresh history each iteration
public async Task EternalOrchestrator(TaskOrchestrationContext context)
{
    await context.CallActivityAsync("DoWork", null);
    await context.CreateTimer(context.CurrentUtcDateTime.AddMinutes(5), default);
    
    // Restart with fresh history
    context.ContinueAsNew(null);
}
```

### Python Example

```python
# ✅ CORRECT - Using continue_as_new
def eternal_orchestrator(context):
    yield context.call_activity("do_work")
    deadline = context.current_utc_datetime + timedelta(minutes=5)
    yield context.create_timer(deadline)
    
    context.continue_as_new(None)  # Restart fresh
```

---

## Constraint #5: Consistent Orchestrator Code During Execution

### Problem

If you change orchestrator code while instances are running, the replay may fail because the code no longer matches the history.

```
History says:                    But new code says:
1. Called Activity "A"           1. Called Activity "B"  ← Mismatch!
2. Activity "A" completed        
```

This causes a `NonDeterministicOrchestrationException`.

### Solutions

1. **Wait for completion**: Let all in-flight instances finish before deploying new code
2. **Side-by-side deployment**: Deploy to a new task hub
3. **Version your code**: Use version flags in inputs

See [Versioning](./versioning.md) for detailed strategies.

---

## Safe Operations Summary

### ✅ Always Safe in Orchestrators

| Operation | API |
|-----------|-----|
| Call an activity | `context.CallActivityAsync()` |
| Call a sub-orchestration | `context.CallSubOrchestratorAsync()` |
| Create a durable timer | `context.CreateTimer()` |
| Wait for external event | `context.WaitForExternalEvent()` |
| Get current time | `context.CurrentUtcDateTime` |
| Get instance ID | `context.InstanceId` |
| Check if replaying | `context.IsReplaying` |
| Get input | `context.GetInput<T>()` |
| Set custom status | `context.SetCustomStatus()` |
| Continue as new | `context.ContinueAsNew()` |
| Generate GUID | `context.NewGuid()` (where available) |

### ❌ Never Safe in Orchestrators

| Operation | Reason |
|-----------|--------|
| Network calls | Non-deterministic |
| Database access | Non-deterministic |
| File system access | Non-deterministic |
| `DateTime.Now/UtcNow` | Non-deterministic |
| `Guid.NewGuid()` | Non-deterministic |
| `Random` | Non-deterministic |
| `Thread.Sleep()` | Blocking, not tracked |
| `Task.Delay()` | Not persisted |
| Environment variables | May change |
| Static mutable state | Shared across replays |

---

## Debugging Constraint Violations

### Common Errors

| Error | Likely Cause |
|-------|--------------|
| `NonDeterministicOrchestrationException` | Code changed during execution, or non-deterministic API used |
| Orchestration stuck | Blocking call (`Thread.Sleep`, `Task.Wait`) |
| Duplicate activity executions | Non-deterministic branching |
| Inconsistent results | Using `DateTime.Now` or `Random` |

### Detection Tips

1. **Enable replay logging**: Check if code paths differ between first run and replay
2. **Review code changes**: Compare deployed code with code at orchestration start
3. **Check for non-deterministic APIs**: Search for `DateTime.Now`, `Guid.NewGuid()`, `Random`
4. **Monitor history size**: Large histories may indicate missing `ContinueAsNew`

---

## Replay-Safe Logging

Logs in orchestrators will be repeated on every replay. To avoid duplicate log entries:

```csharp
// ✅ Only log on first execution
if (!context.IsReplaying)
{
    logger.LogInformation("Processing order {OrderId}", orderId);
}

// Or use a replay-safe logger if available
var replaySafeLogger = context.CreateReplaySafeLogger(logger);
replaySafeLogger.LogInformation("This only logs once");
```

---

## Quick Reference Card

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    ORCHESTRATOR CODE CONSTRAINTS                            │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   ✅ DO                              ❌ DON'T                               │
│   ─────────────────                  ──────────────────                     │
│   • Call activities                  • Use DateTime.Now                     │
│   • Call sub-orchestrations          • Use Guid.NewGuid()                   │
│   • Create durable timers            • Use Random                           │
│   • Wait for external events         • Make HTTP calls                      │
│   • Use context.CurrentUtcDateTime   • Access databases                     │
│   • Use context.NewGuid()            • Read files                           │
│   • Use context.IsReplaying          • Use Thread.Sleep()                   │
│   • Use ContinueAsNew for loops      • Use Task.Delay()                     │
│                                      • Use infinite loops                   │
│                                                                             │
│   REMEMBER: Orchestrator code must produce the same results on replay!     │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Next Steps

- [Learn about Orchestrators →](./orchestrators.md)
- [Understand Versioning →](./versioning.md)
- [Explore Error Handling →](./error-handling.md)

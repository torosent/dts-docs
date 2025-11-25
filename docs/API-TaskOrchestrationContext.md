---
layout: default
title: TaskOrchestrationContext API
parent: API Reference
nav_order: 2
description: "Complete reference for the TaskOrchestrationContext class"
---

# API Reference: TaskOrchestrationContext

The `TaskOrchestrationContext` provides the execution context for orchestrations, enabling interaction with activities, sub-orchestrations, timers, events, and entities.

## Overview

```csharp
public abstract class TaskOrchestrationContext
```

`TaskOrchestrationContext` is passed to orchestration `RunAsync` methods and provides:
- Activity and sub-orchestration invocation
- Timer creation
- External event handling
- Entity operations
- Deterministic utilities

## Properties

### InstanceId

Gets the unique identifier for this orchestration instance.

```csharp
public abstract string InstanceId { get; }
```

**Example:**
```csharp
var logger = context.CreateReplaySafeLogger("MyOrchestration");
logger.LogInformation("Processing instance {InstanceId}", context.InstanceId);
```

### Name

Gets the name of this orchestration.

```csharp
public abstract string Name { get; }
```

### CurrentUtcDateTime

Gets the current UTC date/time in a replay-safe manner.

```csharp
public abstract DateTime CurrentUtcDateTime { get; }
```

**Example:**
```csharp
// ✅ Use this for time-based logic
var now = context.CurrentUtcDateTime;
var deadline = now.AddHours(24);

// ❌ Don't use DateTime.Now/UtcNow
// var now = DateTime.UtcNow;  // Non-deterministic!
```

### IsReplaying

Indicates whether the orchestration is currently replaying.

```csharp
public abstract bool IsReplaying { get; }
```

**Example:**
```csharp
if (!context.IsReplaying)
{
    // Only execute during actual execution, not replays
    metrics.RecordExecution(context.InstanceId);
}
```

### ParentInstanceId

Gets the parent orchestration's instance ID (for sub-orchestrations).

```csharp
public abstract string? ParentInstanceId { get; }
```

## Methods Reference

### CallActivityAsync

Calls an activity and waits for the result.

```csharp
// With return value
Task<TOutput> CallActivityAsync<TOutput>(
    string activityName,
    object? input = null,
    TaskOptions? options = null)

// Without return value
Task CallActivityAsync(
    string activityName,
    object? input = null,
    TaskOptions? options = null)
```

**Example:**
```csharp
// Basic call
var result = await context.CallActivityAsync<PaymentResult>("ProcessPayment", payment);

// With retry
var options = new TaskOptions
{
    Retry = new RetryPolicy(
        maxNumberOfAttempts: 3,
        firstRetryInterval: TimeSpan.FromSeconds(5))
};
var result = await context.CallActivityAsync<PaymentResult>("ProcessPayment", payment, options);

// Fire and forget
await context.CallActivityAsync("SendNotification", notification);
```

### CallSubOrchestratorAsync

Calls a sub-orchestration and waits for completion.

```csharp
// With return value
Task<TOutput> CallSubOrchestratorAsync<TOutput>(
    string orchestratorName,
    object? input = null,
    TaskOptions? options = null)

Task<TOutput> CallSubOrchestratorAsync<TOutput>(
    string orchestratorName,
    object? input,
    SubOrchestrationOptions? options)

// Without return value
Task CallSubOrchestratorAsync(
    string orchestratorName,
    object? input = null,
    TaskOptions? options = null)
```

**Example:**
```csharp
// Basic call
var result = await context.CallSubOrchestratorAsync<OrderResult>("ProcessOrder", order);

// With custom instance ID
var options = new SubOrchestrationOptions
{
    InstanceId = $"{context.InstanceId}-payment-{paymentId}"
};
var result = await context.CallSubOrchestratorAsync<PaymentResult>(
    "PaymentOrchestration",
    payment,
    options);
```

### CreateTimer

Creates a durable timer.

```csharp
Task CreateTimer(
    DateTime fireAt,
    CancellationToken cancellation = default)
```

**Example:**
```csharp
// Simple delay
await context.CreateTimer(
    context.CurrentUtcDateTime.AddMinutes(30),
    CancellationToken.None);

// Cancellable timer
using var cts = new CancellationTokenSource();

Task timerTask = context.CreateTimer(
    context.CurrentUtcDateTime.AddHours(1),
    cts.Token);

Task eventTask = context.WaitForExternalEvent<bool>("CancelTimer");

Task winner = await Task.WhenAny(timerTask, eventTask);

if (winner == eventTask)
{
    cts.Cancel();
}
```

### WaitForExternalEvent

Waits for an external event.

```csharp
Task<T> WaitForExternalEvent<T>(string eventName)

Task WaitForExternalEvent(string eventName)
```

**Example:**
```csharp
// Wait for typed event
var approval = await context.WaitForExternalEvent<ApprovalResponse>("ApprovalEvent");

// Wait for signal (no data)
await context.WaitForExternalEvent("StartSignal");

// With timeout
using var cts = new CancellationTokenSource();

Task<bool> eventTask = context.WaitForExternalEvent<bool>("ApprovalEvent");
Task timeoutTask = context.CreateTimer(
    context.CurrentUtcDateTime.AddDays(7),
    cts.Token);

Task winner = await Task.WhenAny(eventTask, timeoutTask);

if (winner == eventTask)
{
    cts.Cancel();
    bool approved = await eventTask;
}
else
{
    // Timed out
}
```

### CallEntityAsync

Calls an entity operation.

```csharp
Task<TOutput> CallEntityAsync<TOutput>(
    EntityInstanceId entityId,
    string operationName,
    object? input = null)

Task CallEntityAsync(
    EntityInstanceId entityId,
    string operationName,
    object? input = null)
```

**Example:**
```csharp
var entityId = new EntityInstanceId("Counter", "my-counter");

// Call operation with return value
int value = await context.CallEntityAsync<int>(entityId, "Get");

// Call operation without return
await context.CallEntityAsync(entityId, "Add", 10);

// Multiple operations
await context.CallEntityAsync(entityId, "Increment");
await context.CallEntityAsync(entityId, "Increment");
int finalValue = await context.CallEntityAsync<int>(entityId, "Get");
```

### SignalEntityAsync

Signals an entity without waiting (fire-and-forget).

```csharp
Task SignalEntityAsync(
    EntityInstanceId entityId,
    string operationName,
    object? input = null)
```

**Example:**
```csharp
var entityId = new EntityInstanceId("Counter", "my-counter");

// Signal without waiting
await context.SignalEntityAsync(entityId, "Increment");
await context.SignalEntityAsync(entityId, "Add", 5);
```

### LockEntitiesAsync

Acquires exclusive locks on entities.

```csharp
Task<EntityLock> LockEntitiesAsync(params EntityInstanceId[] entityIds)
```

**Example:**
```csharp
var account1 = new EntityInstanceId("Account", "account-1");
var account2 = new EntityInstanceId("Account", "account-2");

// Acquire locks
using (await context.LockEntitiesAsync(account1, account2))
{
    // Operations within lock
    var balance1 = await context.CallEntityAsync<decimal>(account1, "GetBalance");
    var balance2 = await context.CallEntityAsync<decimal>(account2, "GetBalance");
    
    await context.CallEntityAsync(account1, "Withdraw", 100);
    await context.CallEntityAsync(account2, "Deposit", 100);
}
// Locks released when disposed
```

### ContinueAsNew

Restarts the orchestration with new input.

```csharp
void ContinueAsNew(
    object? newInput = null,
    bool preserveUnprocessedEvents = false)
```

**Example:**
```csharp
public override async Task RunAsync(
    TaskOrchestrationContext context,
    MonitorState state)
{
    // Do work
    await context.CallActivityAsync("CheckStatus", state.Target);
    
    // Wait
    await context.CreateTimer(
        context.CurrentUtcDateTime.AddMinutes(5),
        CancellationToken.None);
    
    // Continue forever
    state.CheckCount++;
    context.ContinueAsNew(state);
}
```

### NewGuid

Creates a deterministic GUID.

```csharp
Guid NewGuid()
```

**Example:**
```csharp
// ✅ Deterministic - same on replay
var id = context.NewGuid();

// ❌ Non-deterministic - different on replay
// var id = Guid.NewGuid();
```

### GetInput

Gets the orchestration input.

```csharp
T GetInput<T>()
```

**Example:**
```csharp
public override async Task<OrderResult> RunAsync(
    TaskOrchestrationContext context,
    OrderInput input)
{
    // Input is passed as parameter, but can also use:
    var sameInput = context.GetInput<OrderInput>();
}
```

### CreateReplaySafeLogger

Creates a logger that doesn't duplicate logs during replay.

```csharp
ILogger CreateReplaySafeLogger(string categoryName)
```

**Example:**
```csharp
public override async Task<string> RunAsync(
    TaskOrchestrationContext context,
    string input)
{
    var logger = context.CreateReplaySafeLogger("MyOrchestration");
    
    // These logs appear only once, not during replays
    logger.LogInformation("Starting orchestration for {Input}", input);
    
    var result = await context.CallActivityAsync<string>("ProcessData", input);
    
    logger.LogInformation("Completed with result: {Result}", result);
    
    return result;
}
```

### SetCustomStatus

Sets a custom status that can be queried.

```csharp
void SetCustomStatus(object? customStatus)
```

**Example:**
```csharp
public override async Task<OrderResult> RunAsync(
    TaskOrchestrationContext context,
    OrderInput input)
{
    context.SetCustomStatus(new { Stage = "Validating" });
    await context.CallActivityAsync("Validate", input);
    
    context.SetCustomStatus(new { Stage = "Processing", Progress = 50 });
    await context.CallActivityAsync("Process", input);
    
    context.SetCustomStatus(new { Stage = "Completing", Progress = 90 });
    await context.CallActivityAsync("Complete", input);
    
    context.SetCustomStatus(new { Stage = "Done", Progress = 100 });
    return new OrderResult { Success = true };
}
```

## Supporting Types

### TaskOptions

```csharp
public class TaskOptions
{
    public RetryPolicy? Retry { get; set; }
}
```

### SubOrchestrationOptions

```csharp
public class SubOrchestrationOptions
{
    public string? InstanceId { get; set; }
}
```

### RetryPolicy

```csharp
public class RetryPolicy
{
    public RetryPolicy(
        int maxNumberOfAttempts,
        TimeSpan firstRetryInterval,
        double backoffCoefficient = 1.0,
        TimeSpan? maxRetryInterval = null,
        TimeSpan? retryTimeout = null)
}
```

### EntityInstanceId

```csharp
public readonly struct EntityInstanceId
{
    public EntityInstanceId(string name, string key)
    
    public string Name { get; }
    public string Key { get; }
}
```

## Next Steps

- [API Reference: DurableTaskClient](API-DurableTaskClient.md)
- [Writing Task Orchestrations](Writing-Task-Orchestrations.md)
- [Orchestration Constraints](Orchestration-Constraints.md)

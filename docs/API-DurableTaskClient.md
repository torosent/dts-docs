---
layout: default
title: DurableTaskClient API
parent: API Reference
nav_order: 1
description: "Complete reference for the DurableTaskClient class"
---

# API Reference: DurableTaskClient

The `DurableTaskClient` class is the primary interface for managing orchestration instances from external code. This page provides a comprehensive reference for all client methods.

## Overview

```csharp
public class DurableTaskClient : IAsyncDisposable
```

Use `DurableTaskClient` to:
- Start orchestration instances
- Query orchestration status
- Raise events to orchestrations
- Terminate, suspend, and resume orchestrations
- Purge orchestration history

## Obtaining a Client

### Dependency Injection

```csharp
// Connection string for Durable Task Scheduler
string connectionString = "Endpoint=http://localhost:8080;TaskHub=default;Authentication=None";

services.AddDurableTaskClient(builder =>
{
    builder.UseDurableTaskScheduler(connectionString);
});

// Inject in your class
public class MyService
{
    private readonly DurableTaskClient _client;
    
    public MyService(DurableTaskClient client)
    {
        _client = client;
    }
}
```

### Azure Functions

```csharp
[Function("MyFunction")]
public async Task Run(
    [HttpTrigger] HttpRequestData req,
    [DurableClient] DurableTaskClient client)
{
    // Use client
}
```

## Methods Reference

### ScheduleNewOrchestrationInstanceAsync

Starts a new orchestration instance.

```csharp
Task<string> ScheduleNewOrchestrationInstanceAsync(
    string orchestratorName,
    object? input = null,
    StartOrchestrationOptions? options = null,
    CancellationToken cancellation = default)
```

**Parameters:**
- `orchestratorName`: Name of the orchestration to start
- `input`: Input data for the orchestration
- `options`: Optional configuration including instance ID and start time
- `cancellation`: Cancellation token

**Returns:** Instance ID of the started orchestration

**Example:**
```csharp
// Basic start
string instanceId = await client.ScheduleNewOrchestrationInstanceAsync(
    "MyOrchestration",
    new { OrderId = "123" });

// With options
string instanceId = await client.ScheduleNewOrchestrationInstanceAsync(
    "MyOrchestration",
    input,
    new StartOrchestrationOptions
    {
        InstanceId = "custom-id-123",
        StartAt = DateTimeOffset.UtcNow.AddHours(1)
    });
```

### GetInstanceAsync

Gets the status of an orchestration instance.

```csharp
Task<OrchestrationMetadata?> GetInstanceAsync(
    string instanceId,
    bool getInputsAndOutputs = false,
    CancellationToken cancellation = default)
```

**Parameters:**
- `instanceId`: The instance ID to query
- `getInputsAndOutputs`: Whether to include serialized input/output
- `cancellation`: Cancellation token

**Returns:** `OrchestrationMetadata` or `null` if not found

**Example:**
```csharp
var metadata = await client.GetInstanceAsync(instanceId, getInputsAndOutputs: true);

if (metadata != null)
{
    Console.WriteLine($"Status: {metadata.RuntimeStatus}");
    Console.WriteLine($"Created: {metadata.CreatedAt}");
    
    if (metadata.RuntimeStatus == OrchestrationRuntimeStatus.Completed)
    {
        var output = metadata.ReadOutputAs<MyOutput>();
    }
}
```

### WaitForInstanceStartAsync

Waits for an orchestration to start running.

```csharp
Task<OrchestrationMetadata?> WaitForInstanceStartAsync(
    string instanceId,
    bool getInputsAndOutputs = false,
    CancellationToken cancellation = default)
```

**Example:**
```csharp
string instanceId = await client.ScheduleNewOrchestrationInstanceAsync("MyOrchestration", input);

// Wait for it to start
var metadata = await client.WaitForInstanceStartAsync(instanceId);
Console.WriteLine($"Orchestration started at {metadata?.CreatedAt}");
```

### WaitForInstanceCompletionAsync

Waits for an orchestration to complete.

```csharp
Task<OrchestrationMetadata?> WaitForInstanceCompletionAsync(
    string instanceId,
    bool getInputsAndOutputs = false,
    CancellationToken cancellation = default)
```

**Example:**
```csharp
using var cts = new CancellationTokenSource(TimeSpan.FromMinutes(5));

var metadata = await client.WaitForInstanceCompletionAsync(
    instanceId,
    getInputsAndOutputs: true,
    cts.Token);

if (metadata?.RuntimeStatus == OrchestrationRuntimeStatus.Completed)
{
    var result = metadata.ReadOutputAs<MyResult>();
}
```

### GetAllInstancesAsync

Queries multiple orchestration instances.

```csharp
Task<OrchestrationQueryResult> GetAllInstancesAsync(
    OrchestrationQuery? query = null,
    CancellationToken cancellation = default)
```

**Example:**
```csharp
var query = new OrchestrationQuery
{
    RuntimeStatus = new[] { OrchestrationRuntimeStatus.Running },
    CreatedFrom = DateTime.UtcNow.AddDays(-7),
    PageSize = 100
};

var result = await client.GetAllInstancesAsync(query);

foreach (var instance in result.Instances)
{
    Console.WriteLine($"{instance.InstanceId}: {instance.RuntimeStatus}");
}

// Handle pagination
if (result.ContinuationToken != null)
{
    query.ContinuationToken = result.ContinuationToken;
    var nextPage = await client.GetAllInstancesAsync(query);
}
```

### RaiseEventAsync

Raises an event to a waiting orchestration.

```csharp
Task RaiseEventAsync(
    string instanceId,
    string eventName,
    object? eventData = null,
    CancellationToken cancellation = default)
```

**Example:**
```csharp
// Simple event
await client.RaiseEventAsync(instanceId, "ApprovalEvent", true);

// With complex data
await client.RaiseEventAsync(instanceId, "OrderUpdate", new OrderUpdate
{
    Status = "Shipped",
    TrackingNumber = "1234567890"
});
```

### TerminateInstanceAsync

Terminates a running orchestration.

```csharp
Task TerminateInstanceAsync(
    string instanceId,
    object? output = null,
    CancellationToken cancellation = default)
```

**Example:**
```csharp
// Simple termination
await client.TerminateInstanceAsync(instanceId, "Terminated by admin");

// With structured output
await client.TerminateInstanceAsync(instanceId, new TerminationInfo
{
    Reason = "User request",
    TerminatedBy = "admin@company.com",
    Timestamp = DateTime.UtcNow
});
```

### SuspendInstanceAsync

Suspends a running orchestration.

```csharp
Task SuspendInstanceAsync(
    string instanceId,
    string? reason = null,
    CancellationToken cancellation = default)
```

**Example:**
```csharp
await client.SuspendInstanceAsync(instanceId, "Paused for maintenance");
```

### ResumeInstanceAsync

Resumes a suspended orchestration.

```csharp
Task ResumeInstanceAsync(
    string instanceId,
    string? reason = null,
    CancellationToken cancellation = default)
```

**Example:**
```csharp
await client.ResumeInstanceAsync(instanceId, "Maintenance complete");
```

### PurgeInstanceAsync

Purges a single orchestration instance.

```csharp
Task<PurgeResult> PurgeInstanceAsync(
    string instanceId,
    CancellationToken cancellation = default)
```

**Example:**
```csharp
var result = await client.PurgeInstanceAsync(instanceId);
Console.WriteLine($"Purged: {result.PurgedInstanceCount}");
```

### PurgeAllInstancesAsync

Purges multiple orchestration instances.

```csharp
Task<PurgeResult> PurgeAllInstancesAsync(
    PurgeInstancesFilter filter,
    CancellationToken cancellation = default)
```

**Example:**
```csharp
var result = await client.PurgeAllInstancesAsync(
    new PurgeInstancesFilter
    {
        CreatedTo = DateTime.UtcNow.AddDays(-30),
        RuntimeStatus = new[]
        {
            OrchestrationRuntimeStatus.Completed,
            OrchestrationRuntimeStatus.Failed,
            OrchestrationRuntimeStatus.Terminated
        }
    });

Console.WriteLine($"Purged {result.PurgedInstanceCount} instances");
```

## Supporting Types

### StartOrchestrationOptions

```csharp
public class StartOrchestrationOptions
{
    // Custom instance ID (auto-generated if not specified)
    public string? InstanceId { get; set; }
    
    // Scheduled start time
    public DateTimeOffset? StartAt { get; set; }
}
```

### OrchestrationQuery

```csharp
public class OrchestrationQuery
{
    // Filter by status
    public OrchestrationRuntimeStatus[]? RuntimeStatus { get; set; }
    
    // Filter by creation time
    public DateTime? CreatedFrom { get; set; }
    public DateTime? CreatedTo { get; set; }
    
    // Filter by instance ID prefix
    public string? InstanceIdPrefix { get; set; }
    
    // Filter by task hub names
    public string[]? TaskHubNames { get; set; }
    
    // Pagination
    public int PageSize { get; set; } = 100;
    public string? ContinuationToken { get; set; }
}
```

### OrchestrationMetadata

```csharp
public class OrchestrationMetadata
{
    public string InstanceId { get; }
    public string? Name { get; }
    public OrchestrationRuntimeStatus RuntimeStatus { get; }
    public DateTimeOffset CreatedAt { get; }
    public DateTimeOffset LastUpdatedAt { get; }
    
    // Available when getInputsAndOutputs = true
    public string? SerializedInput { get; }
    public string? SerializedOutput { get; }
    public string? SerializedCustomStatus { get; }
    public TaskFailureDetails? FailureDetails { get; }
    
    // Helper method
    public T? ReadOutputAs<T>();
    public T? ReadInputAs<T>();
}
```

### OrchestrationRuntimeStatus

```csharp
public enum OrchestrationRuntimeStatus
{
    Running,
    Completed,
    Failed,
    Terminated,
    Pending,
    Suspended
}
```

### PurgeInstancesFilter

```csharp
public class PurgeInstancesFilter
{
    public DateTime? CreatedFrom { get; set; }
    public DateTime? CreatedTo { get; set; }
    public OrchestrationRuntimeStatus[]? RuntimeStatus { get; set; }
}
```

## Next Steps

- [API Reference: TaskOrchestrationContext](API-TaskOrchestrationContext.md)
- [Orchestration Instance Management](Orchestration-Instance-Management.md)
- [Instance Queries](Instance-Queries.md)

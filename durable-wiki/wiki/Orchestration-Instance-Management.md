# Orchestration Instance Management

The Durable Task .NET SDK provides comprehensive APIs for managing orchestration instances through the `DurableTaskClient` class with **Durable Task Scheduler**. This includes starting, querying, terminating, suspending, and purging orchestrations.

## Overview

`DurableTaskClient` is the primary interface for managing orchestration instances from external code (outside of orchestrations).

## Getting a Client Instance

### Dependency Injection

```csharp
// Register in DI with Durable Task Scheduler
services.AddDurableTaskClient(builder =>
{
    builder.UseDurableTaskScheduler(connectionString);
});

// Inject in controllers/services
public class OrchestrationController : Controller
{
    private readonly DurableTaskClient _client;
    
    public OrchestrationController(DurableTaskClient client)
    {
        _client = client;
    }
}
```

### Azure Functions

```csharp
[Function("HttpStart")]
public static async Task<HttpResponseData> HttpStart(
    [HttpTrigger(AuthorizationLevel.Function, "post")] HttpRequestData req,
    [DurableClient] DurableTaskClient client)
{
    // Use client here
}
```

## Starting Orchestrations

### Basic Start

```csharp
string instanceId = await _client.ScheduleNewOrchestrationInstanceAsync(
    "MyOrchestration", 
    input);
```

### With Custom Instance ID

```csharp
string instanceId = await _client.ScheduleNewOrchestrationInstanceAsync(
    "MyOrchestration",
    input,
    new StartOrchestrationOptions
    {
        InstanceId = "my-custom-id-123"
    });
```

### With Scheduled Start Time

```csharp
string instanceId = await _client.ScheduleNewOrchestrationInstanceAsync(
    "MyOrchestration",
    input,
    new StartOrchestrationOptions
    {
        InstanceId = "scheduled-task",
        StartAt = DateTimeOffset.UtcNow.AddHours(1)
    });
```

## Querying Instance Status

### Get Single Instance

```csharp
OrchestrationMetadata? metadata = await _client.GetInstanceAsync(
    instanceId, 
    getInputsAndOutputs: true);

if (metadata != null)
{
    Console.WriteLine($"Status: {metadata.RuntimeStatus}");
    Console.WriteLine($"Created: {metadata.CreatedAt}");
    Console.WriteLine($"Last Updated: {metadata.LastUpdatedAt}");
    
    if (metadata.RuntimeStatus == OrchestrationRuntimeStatus.Completed)
    {
        var output = metadata.ReadOutputAs<MyOutput>();
        Console.WriteLine($"Output: {output}");
    }
}
```

### Runtime Status Values

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

### Wait for Completion

```csharp
// Wait indefinitely
OrchestrationMetadata? metadata = await _client.WaitForInstanceCompletionAsync(
    instanceId,
    getInputsAndOutputs: true);

// Wait with timeout
using var cts = new CancellationTokenSource(TimeSpan.FromMinutes(5));
OrchestrationMetadata? metadata = await _client.WaitForInstanceCompletionAsync(
    instanceId,
    getInputsAndOutputs: true,
    cts.Token);
```

### Wait for Specific State

```csharp
// Wait until running (started)
OrchestrationMetadata? metadata = await _client.WaitForInstanceStartAsync(
    instanceId,
    getInputsAndOutputs: false);
```

## Querying Multiple Instances

### Query with Filters

```csharp
var query = new OrchestrationQuery
{
    RuntimeStatus = new[]
    {
        OrchestrationRuntimeStatus.Running,
        OrchestrationRuntimeStatus.Pending
    },
    CreatedFrom = DateTime.UtcNow.AddDays(-7),
    CreatedTo = DateTime.UtcNow,
    PageSize = 100
};

OrchestrationQueryResult result = await _client.GetAllInstancesAsync(query);

foreach (var instance in result.Instances)
{
    Console.WriteLine($"Instance: {instance.InstanceId}, Status: {instance.RuntimeStatus}");
}
```

### Pagination

```csharp
string? continuationToken = null;
var allInstances = new List<OrchestrationMetadata>();

do
{
    var query = new OrchestrationQuery
    {
        PageSize = 100,
        ContinuationToken = continuationToken
    };
    
    var result = await _client.GetAllInstancesAsync(query);
    allInstances.AddRange(result.Instances);
    continuationToken = result.ContinuationToken;
    
} while (continuationToken != null);
```

### Filter by Instance ID Prefix

```csharp
var query = new OrchestrationQuery
{
    InstanceIdPrefix = "order-",
    RuntimeStatus = new[] { OrchestrationRuntimeStatus.Running }
};

var result = await _client.GetAllInstancesAsync(query);
```

### Get Instances by Name

```csharp
var query = new OrchestrationQuery
{
    TaskHubNames = new[] { "OrderOrchestration", "PaymentOrchestration" }
};

var result = await _client.GetAllInstancesAsync(query);
```

## Terminating Orchestrations

### Terminate Single Instance

```csharp
await _client.TerminateInstanceAsync(
    instanceId, 
    "Terminated by administrator");
```

### Terminate with Output

```csharp
await _client.TerminateInstanceAsync(
    instanceId,
    new TerminationOutput
    {
        Reason = "Manual termination",
        TerminatedBy = "admin@company.com"
    });
```

### Terminate Multiple Instances

```csharp
// Find all running instances of a specific orchestration
var query = new OrchestrationQuery
{
    RuntimeStatus = new[] { OrchestrationRuntimeStatus.Running },
    InstanceIdPrefix = "batch-job-"
};

var result = await _client.GetAllInstancesAsync(query);

foreach (var instance in result.Instances)
{
    await _client.TerminateInstanceAsync(
        instance.InstanceId, 
        "Batch termination");
}
```

## Suspending and Resuming

### Suspend Instance

```csharp
await _client.SuspendInstanceAsync(instanceId, "Paused for maintenance");
```

### Resume Instance

```csharp
await _client.ResumeInstanceAsync(instanceId, "Maintenance complete");
```

### Suspend Pattern

```csharp
public class MaintenanceController
{
    private readonly DurableTaskClient _client;
    
    public async Task<IActionResult> StartMaintenance()
    {
        // Suspend all running orchestrations
        var query = new OrchestrationQuery
        {
            RuntimeStatus = new[] { OrchestrationRuntimeStatus.Running }
        };
        
        var result = await _client.GetAllInstancesAsync(query);
        var suspendedIds = new List<string>();
        
        foreach (var instance in result.Instances)
        {
            await _client.SuspendInstanceAsync(
                instance.InstanceId, 
                "System maintenance");
            suspendedIds.Add(instance.InstanceId);
        }
        
        return Ok(new { SuspendedCount = suspendedIds.Count, InstanceIds = suspendedIds });
    }
    
    public async Task<IActionResult> EndMaintenance([FromBody] List<string> instanceIds)
    {
        foreach (var instanceId in instanceIds)
        {
            await _client.ResumeInstanceAsync(instanceId, "Maintenance complete");
        }
        
        return Ok(new { ResumedCount = instanceIds.Count });
    }
}
```

## Raising Events

### Raise Event to Instance

```csharp
await _client.RaiseEventAsync(
    instanceId, 
    "ApprovalEvent", 
    new ApprovalData { Approved = true, ApproverEmail = "manager@company.com" });
```

### Raise Event with Type

```csharp
await _client.RaiseEventAsync<ApprovalDecision>(
    instanceId,
    "ManagerApproval",
    new ApprovalDecision
    {
        Decision = "Approved",
        Comments = "Looks good",
        DecisionDate = DateTime.UtcNow
    });
```

## Purging Instance History

### Purge Single Instance

```csharp
PurgeResult result = await _client.PurgeInstanceAsync(instanceId);

if (result.PurgedInstanceCount > 0)
{
    Console.WriteLine($"Purged instance {instanceId}");
}
```

### Purge with Filter

```csharp
// Purge all completed instances older than 30 days
PurgeResult result = await _client.PurgeAllInstancesAsync(
    new PurgeInstancesFilter
    {
        CreatedFrom = DateTime.MinValue,
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

### Scheduled Purge

```csharp
[Function("PurgeOldInstances")]
public async Task PurgeOldInstances(
    [TimerTrigger("0 0 * * *")] TimerInfo timerInfo,  // Daily at midnight
    [DurableClient] DurableTaskClient client,
    ILogger log)
{
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
    
    log.LogInformation("Purged {Count} instances", result.PurgedInstanceCount);
}
```

## HTTP Management Endpoints

### ASP.NET Core Controller

```csharp
[ApiController]
[Route("api/orchestrations")]
public class OrchestrationManagementController : ControllerBase
{
    private readonly DurableTaskClient _client;
    
    public OrchestrationManagementController(DurableTaskClient client)
    {
        _client = client;
    }
    
    [HttpPost]
    public async Task<IActionResult> StartOrchestration([FromBody] StartRequest request)
    {
        string instanceId = await _client.ScheduleNewOrchestrationInstanceAsync(
            request.OrchestrationName,
            request.Input,
            new StartOrchestrationOptions { InstanceId = request.InstanceId });
        
        return Accepted(new { InstanceId = instanceId });
    }
    
    [HttpGet("{instanceId}")]
    public async Task<IActionResult> GetStatus(string instanceId)
    {
        var metadata = await _client.GetInstanceAsync(instanceId, getInputsAndOutputs: true);
        
        if (metadata == null)
        {
            return NotFound();
        }
        
        return Ok(new
        {
            metadata.InstanceId,
            metadata.Name,
            Status = metadata.RuntimeStatus.ToString(),
            metadata.CreatedAt,
            metadata.LastUpdatedAt,
            Input = metadata.SerializedInput,
            Output = metadata.SerializedOutput
        });
    }
    
    [HttpPost("{instanceId}/terminate")]
    public async Task<IActionResult> Terminate(string instanceId, [FromBody] TerminateRequest request)
    {
        await _client.TerminateInstanceAsync(instanceId, request.Reason);
        return Ok();
    }
    
    [HttpPost("{instanceId}/suspend")]
    public async Task<IActionResult> Suspend(string instanceId, [FromBody] SuspendRequest request)
    {
        await _client.SuspendInstanceAsync(instanceId, request.Reason);
        return Ok();
    }
    
    [HttpPost("{instanceId}/resume")]
    public async Task<IActionResult> Resume(string instanceId, [FromBody] ResumeRequest request)
    {
        await _client.ResumeInstanceAsync(instanceId, request.Reason);
        return Ok();
    }
    
    [HttpPost("{instanceId}/events/{eventName}")]
    public async Task<IActionResult> RaiseEvent(
        string instanceId, 
        string eventName, 
        [FromBody] object eventData)
    {
        await _client.RaiseEventAsync(instanceId, eventName, eventData);
        return Ok();
    }
    
    [HttpDelete("{instanceId}")]
    public async Task<IActionResult> Purge(string instanceId)
    {
        var result = await _client.PurgeInstanceAsync(instanceId);
        return Ok(new { PurgedCount = result.PurgedInstanceCount });
    }
}
```

### Azure Functions HTTP Endpoints

```csharp
public static class OrchestrationManagement
{
    [Function("GetStatus")]
    public static async Task<HttpResponseData> GetStatus(
        [HttpTrigger(AuthorizationLevel.Function, "get", Route = "orchestrations/{instanceId}")] 
        HttpRequestData req,
        string instanceId,
        [DurableClient] DurableTaskClient client)
    {
        var metadata = await client.GetInstanceAsync(instanceId, getInputsAndOutputs: true);
        
        if (metadata == null)
        {
            return req.CreateResponse(HttpStatusCode.NotFound);
        }
        
        var response = req.CreateResponse(HttpStatusCode.OK);
        await response.WriteAsJsonAsync(new
        {
            metadata.InstanceId,
            Status = metadata.RuntimeStatus.ToString(),
            metadata.CreatedAt,
            metadata.LastUpdatedAt
        });
        
        return response;
    }
}
```

## Best Practices

### 1. Use Idempotent Instance IDs

```csharp
// ✅ Deterministic, idempotent ID
string instanceId = $"order-{order.Id}";

// Check if already exists
var existing = await _client.GetInstanceAsync(instanceId);
if (existing != null && 
    existing.RuntimeStatus == OrchestrationRuntimeStatus.Running)
{
    return existing.InstanceId;
}

await _client.ScheduleNewOrchestrationInstanceAsync(
    "OrderOrchestration",
    order,
    new StartOrchestrationOptions { InstanceId = instanceId });
```

### 2. Handle Race Conditions

```csharp
try
{
    await _client.ScheduleNewOrchestrationInstanceAsync(
        "MyOrchestration",
        input,
        new StartOrchestrationOptions { InstanceId = instanceId });
}
catch (InvalidOperationException ex) when (ex.Message.Contains("already exists"))
{
    // Instance already running - could be a retry
    var existing = await _client.GetInstanceAsync(instanceId);
    return existing?.InstanceId;
}
```

### 3. Poll with Backoff

```csharp
async Task<OrchestrationMetadata?> WaitForCompletionWithBackoff(
    string instanceId, 
    TimeSpan timeout)
{
    var stopwatch = Stopwatch.StartNew();
    var delay = TimeSpan.FromMilliseconds(500);
    
    while (stopwatch.Elapsed < timeout)
    {
        var metadata = await _client.GetInstanceAsync(instanceId);
        
        if (metadata?.RuntimeStatus is 
            OrchestrationRuntimeStatus.Completed or
            OrchestrationRuntimeStatus.Failed or
            OrchestrationRuntimeStatus.Terminated)
        {
            return metadata;
        }
        
        await Task.Delay(delay);
        delay = TimeSpan.FromMilliseconds(Math.Min(delay.TotalMilliseconds * 2, 10000));
    }
    
    return null;
}
```

## Next Steps

- [Instance Queries](Instance-Queries.md) - Advanced querying
- [Orchestration Versioning](Orchestration-Versioning.md) - Version management
- [External Events](External-Events.md) - Raise events

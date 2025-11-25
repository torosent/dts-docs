# Eternal Orchestrations

Eternal orchestrations run indefinitely by using `ContinueAsNew` to reset their state while preserving execution. This pattern is useful for monitoring, aggregation, and long-running background processes.

## Overview

Regular orchestrations maintain their entire execution history, which can grow unbounded for long-running processes. Eternal orchestrations solve this by periodically resetting their state while continuing execution.

## The ContinueAsNew Method

`ContinueAsNew` restarts the orchestration with a new history, preserving only the input you specify:

```csharp
public override async Task RunAsync(
    TaskOrchestrationContext context, 
    MonitorState state)
{
    // Do some work
    var result = await context.CallActivityAsync<HealthCheckResult>("CheckHealth", state.Target);
    
    // Update state for next iteration
    state.LastCheckResult = result;
    state.CheckCount++;
    
    // Wait before next iteration
    await context.CreateTimer(context.CurrentUtcDateTime.AddMinutes(5), CancellationToken.None);
    
    // Restart with new state - history is cleared
    context.ContinueAsNew(state);
}
```

## Patterns

### Periodic Monitor

Monitor a resource at regular intervals:

```csharp
[DurableTask(nameof(HealthMonitorOrchestration))]
public class HealthMonitorOrchestration : TaskOrchestrator<MonitorConfig, object?>
{
    public override async Task<object?> RunAsync(
        TaskOrchestrationContext context, 
        MonitorConfig config)
    {
        var logger = context.CreateReplaySafeLogger("HealthMonitor");
        
        // Perform health check
        var status = await context.CallActivityAsync<HealthStatus>(
            "CheckServiceHealth", 
            config.ServiceEndpoint);
        
        logger.LogInformation(
            "Health check for {Service}: {Status}",
            config.ServiceName,
            status.IsHealthy ? "Healthy" : "Unhealthy");
        
        // Take action based on status
        if (!status.IsHealthy)
        {
            await context.CallActivityAsync("SendAlert", 
                new AlertData 
                { 
                    Service = config.ServiceName,
                    Status = status,
                    Timestamp = context.CurrentUtcDateTime
                });
            
            // Could also attempt recovery
            await context.CallActivityAsync("AttemptServiceRecovery", config);
        }
        
        // Wait for next check interval
        await context.CreateTimer(
            context.CurrentUtcDateTime.Add(config.CheckInterval), 
            CancellationToken.None);
        
        // Continue forever with same config
        context.ContinueAsNew(config);
        
        // Return value is only used if orchestration completes (which it won't)
        return null;
    }
}

public record MonitorConfig
{
    public string ServiceName { get; init; } = "";
    public string ServiceEndpoint { get; init; } = "";
    public TimeSpan CheckInterval { get; init; } = TimeSpan.FromMinutes(5);
}
```

### Event Aggregator

Collect and aggregate events over time:

```csharp
[DurableTask(nameof(EventAggregatorOrchestration))]
public class EventAggregatorOrchestration : TaskOrchestrator<AggregatorState, object?>
{
    public override async Task<object?> RunAsync(
        TaskOrchestrationContext context, 
        AggregatorState state)
    {
        var logger = context.CreateReplaySafeLogger("EventAggregator");
        
        // Initialize state if first run
        state ??= new AggregatorState();
        
        using var cts = new CancellationTokenSource();
        
        // Wait for either an event or the aggregation window
        Task<EventData> eventTask = context.WaitForExternalEvent<EventData>("NewEvent");
        Task timerTask = context.CreateTimer(
            context.CurrentUtcDateTime.Add(state.AggregationWindow), 
            cts.Token);
        
        Task winner = await Task.WhenAny(eventTask, timerTask);
        
        if (winner == eventTask)
        {
            // Cancel timer
            cts.Cancel();
            
            // Add event to collection
            EventData newEvent = await eventTask;
            state.Events.Add(newEvent);
            
            logger.LogInformation(
                "Collected event: {EventType}, total events: {Count}",
                newEvent.Type,
                state.Events.Count);
        }
        else
        {
            // Aggregation window expired - process events
            if (state.Events.Any())
            {
                logger.LogInformation(
                    "Processing {Count} aggregated events",
                    state.Events.Count);
                
                await context.CallActivityAsync("ProcessAggregatedEvents", state.Events);
            }
            
            // Clear events for next window
            state.Events.Clear();
        }
        
        // Continue with preserved unprocessed events
        context.ContinueAsNew(state, preserveUnprocessedEvents: true);
        
        return null;
    }
}

public class AggregatorState
{
    public List<EventData> Events { get; set; } = new();
    public TimeSpan AggregationWindow { get; set; } = TimeSpan.FromMinutes(10);
}
```

### Counter with Periodic Save

Maintain a counter with periodic persistence:

```csharp
[DurableTask(nameof(CounterOrchestration))]
public class CounterOrchestration : TaskOrchestrator<CounterState, object?>
{
    private const int MaxEventsPerIteration = 100;
    
    public override async Task<object?> RunAsync(
        TaskOrchestrationContext context, 
        CounterState state)
    {
        state ??= new CounterState();
        
        int eventsThisIteration = 0;
        
        while (eventsThisIteration < MaxEventsPerIteration)
        {
            using var cts = new CancellationTokenSource();
            
            Task<int> incrementTask = context.WaitForExternalEvent<int>("Increment");
            Task saveTask = context.CreateTimer(
                context.CurrentUtcDateTime.AddMinutes(1), 
                cts.Token);
            
            Task winner = await Task.WhenAny(incrementTask, saveTask);
            
            if (winner == incrementTask)
            {
                cts.Cancel();
                state.Value += await incrementTask;
                eventsThisIteration++;
            }
            else
            {
                // Save periodically
                await context.CallActivityAsync("SaveCounter", state);
                break;
            }
        }
        
        // Reset history but keep state
        context.ContinueAsNew(state, preserveUnprocessedEvents: true);
        
        return null;
    }
}

public class CounterState
{
    public long Value { get; set; }
    public DateTime LastSaved { get; set; }
}
```

### Job Processor with Graceful Shutdown

Process jobs until shutdown signal:

```csharp
[DurableTask(nameof(JobProcessorOrchestration))]
public class JobProcessorOrchestration : TaskOrchestrator<ProcessorState, ProcessorResult>
{
    private const int MaxJobsPerCycle = 10;
    
    public override async Task<ProcessorResult> RunAsync(
        TaskOrchestrationContext context, 
        ProcessorState state)
    {
        var logger = context.CreateReplaySafeLogger("JobProcessor");
        state ??= new ProcessorState();
        
        // Check for shutdown signal
        Task shutdownTask = context.WaitForExternalEvent<object?>("Shutdown");
        
        // Process jobs in this cycle
        for (int i = 0; i < MaxJobsPerCycle; i++)
        {
            if (shutdownTask.IsCompleted)
            {
                logger.LogInformation("Shutdown requested, completing orchestration");
                return new ProcessorResult
                {
                    TotalJobsProcessed = state.ProcessedCount,
                    Status = "Shutdown"
                };
            }
            
            // Check for new jobs
            var jobs = await context.CallActivityAsync<List<Job>>(
                "GetPendingJobs", 
                new { Limit = 1 });
            
            if (jobs.Any())
            {
                var job = jobs.First();
                await context.CallActivityAsync("ProcessJob", job);
                state.ProcessedCount++;
                
                logger.LogInformation(
                    "Processed job {JobId}, total: {Count}",
                    job.Id,
                    state.ProcessedCount);
            }
            else
            {
                // No jobs, wait before checking again
                await context.CreateTimer(
                    context.CurrentUtcDateTime.AddSeconds(30), 
                    CancellationToken.None);
            }
        }
        
        // Continue processing
        context.ContinueAsNew(state, preserveUnprocessedEvents: true);
        
        // Only returned if shutdown
        return new ProcessorResult
        {
            TotalJobsProcessed = state.ProcessedCount,
            Status = "Running"
        };
    }
}
```

### Rate-Limited Processor

Process items with rate limiting:

```csharp
[DurableTask(nameof(RateLimitedOrchestration))]
public class RateLimitedOrchestration : TaskOrchestrator<RateLimitState, object?>
{
    public override async Task<object?> RunAsync(
        TaskOrchestrationContext context, 
        RateLimitState state)
    {
        state ??= new RateLimitState
        {
            ProcessedThisWindow = 0,
            WindowStart = context.CurrentUtcDateTime,
            RateLimitPerMinute = 60
        };
        
        var logger = context.CreateReplaySafeLogger("RateLimited");
        
        // Check if we need to reset the window
        if (context.CurrentUtcDateTime - state.WindowStart >= TimeSpan.FromMinutes(1))
        {
            state.ProcessedThisWindow = 0;
            state.WindowStart = context.CurrentUtcDateTime;
        }
        
        // Wait for work item
        using var cts = new CancellationTokenSource();
        
        Task<WorkItem> workTask = context.WaitForExternalEvent<WorkItem>("NewWorkItem");
        Task windowEndTask = context.CreateTimer(
            state.WindowStart.AddMinutes(1), 
            cts.Token);
        
        Task winner = await Task.WhenAny(workTask, windowEndTask);
        
        if (winner == workTask)
        {
            cts.Cancel();
            var workItem = await workTask;
            
            // Check rate limit
            if (state.ProcessedThisWindow < state.RateLimitPerMinute)
            {
                await context.CallActivityAsync("ProcessWorkItem", workItem);
                state.ProcessedThisWindow++;
                
                logger.LogInformation(
                    "Processed item, {Count}/{Limit} this window",
                    state.ProcessedThisWindow,
                    state.RateLimitPerMinute);
            }
            else
            {
                // Queue for next window
                state.PendingItems.Add(workItem);
                logger.LogWarning("Rate limit hit, queued item for next window");
            }
        }
        else
        {
            // Window ended - reset and process pending
            state.WindowStart = context.CurrentUtcDateTime;
            state.ProcessedThisWindow = 0;
            
            // Process pending items
            while (state.PendingItems.Any() && 
                   state.ProcessedThisWindow < state.RateLimitPerMinute)
            {
                var item = state.PendingItems.First();
                state.PendingItems.RemoveAt(0);
                
                await context.CallActivityAsync("ProcessWorkItem", item);
                state.ProcessedThisWindow++;
            }
        }
        
        context.ContinueAsNew(state, preserveUnprocessedEvents: true);
        return null;
    }
}
```

## ContinueAsNew Options

### Preserve Unprocessed Events

Keep external events that haven't been consumed:

```csharp
// Events raised but not yet awaited will be preserved
context.ContinueAsNew(state, preserveUnprocessedEvents: true);
```

### New Input

Pass updated state or configuration:

```csharp
// Update configuration before continuing
state.Iteration++;
state.LastRunTime = context.CurrentUtcDateTime;
context.ContinueAsNew(state);
```

## Best Practices

### 1. Limit Work Per Iteration

```csharp
// ✅ Limit iterations to prevent history buildup
const int MaxEventsPerCycle = 100;
int processed = 0;

while (processed < MaxEventsPerCycle)
{
    // Process events
    processed++;
}

context.ContinueAsNew(state);
```

### 2. Include Termination Condition

```csharp
// ✅ Provide a way to stop
Task terminateTask = context.WaitForExternalEvent<object?>("Terminate");

if (terminateTask.IsCompleted)
{
    return new Result { Status = "Terminated" };
}

context.ContinueAsNew(state, preserveUnprocessedEvents: true);
```

### 3. Persist State Before ContinueAsNew

```csharp
// ✅ Save state before resetting
if (state.IsDirty)
{
    await context.CallActivityAsync("PersistState", state);
    state.IsDirty = false;
}

context.ContinueAsNew(state);
```

### 4. Use Replay-Safe Logging

```csharp
var logger = context.CreateReplaySafeLogger("EternalOrchestration");

// Logs won't be duplicated during replay
logger.LogInformation("Starting iteration {Iteration}", state.Iteration);
```

## Considerations

1. **History Reset**: Each `ContinueAsNew` creates a new history, so very old events/timers are discarded
2. **State Size**: Keep the state object small as it's passed between iterations
3. **Debugging**: Shorter histories are easier to debug but lose historical context
4. **Termination**: Always provide a way to gracefully terminate eternal orchestrations

## Next Steps

- [External Events](External-Events.md) - Wait for external signals
- [Durable Timers](Durable-Timers.md) - Create delays
- [Orchestration Instance Management](Orchestration-Instance-Management.md) - Terminate instances

# Diagnostics and Logging

The Durable Task .NET SDK provides comprehensive diagnostics and logging capabilities to help you monitor, debug, and troubleshoot your orchestrations, activities, and entities.

## Overview

Effective diagnostics are crucial for:
- Understanding orchestration execution flow
- Debugging failures and unexpected behavior
- Monitoring performance and health
- Auditing workflow execution

## Logging

### Replay-Safe Logging

Orchestrations replay from history, which means code runs multiple times. Use replay-safe logging to avoid duplicate log entries:

```csharp
public override async Task<string> RunAsync(
    TaskOrchestrationContext context, 
    string input)
{
    // Create a replay-safe logger
    ILogger logger = context.CreateReplaySafeLogger("MyOrchestration");
    
    // This will only log once, not during replays
    logger.LogInformation("Starting orchestration with input: {Input}", input);
    
    var result = await context.CallActivityAsync<string>("ProcessData", input);
    
    logger.LogInformation("Processing complete with result: {Result}", result);
    
    return result;
}
```

### Standard Logging in Activities

Activities don't replay, so use standard logging:

```csharp
[DurableTask(nameof(ProcessDataActivity))]
public class ProcessDataActivity : TaskActivity<string, string>
{
    private readonly ILogger<ProcessDataActivity> _logger;
    
    public ProcessDataActivity(ILogger<ProcessDataActivity> logger)
    {
        _logger = logger;
    }
    
    public override async Task<string> RunAsync(
        TaskActivityContext context, 
        string input)
    {
        _logger.LogInformation(
            "Processing data for instance {InstanceId}: {Input}",
            context.InstanceId,
            input);
        
        try
        {
            var result = await ProcessInternalAsync(input);
            
            _logger.LogInformation(
                "Successfully processed data for {InstanceId}",
                context.InstanceId);
            
            return result;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, 
                "Failed to process data for {InstanceId}",
                context.InstanceId);
            throw;
        }
    }
}
```

### Structured Logging

Use structured logging for better searchability:

```csharp
public override async Task<OrderResult> RunAsync(
    TaskOrchestrationContext context, 
    OrderRequest request)
{
    var logger = context.CreateReplaySafeLogger("OrderOrchestration");
    
    // Use structured logging with named parameters
    logger.LogInformation(
        "Processing order {OrderId} for customer {CustomerId} with amount {Amount:C}",
        request.OrderId,
        request.CustomerId,
        request.Amount);
    
    // Log at appropriate levels
    logger.LogDebug("Order details: {@Order}", request);
    
    var paymentResult = await context.CallActivityAsync<PaymentResult>(
        "ProcessPayment", 
        request.Payment);
    
    if (!paymentResult.Success)
    {
        logger.LogWarning(
            "Payment failed for order {OrderId}: {Error}",
            request.OrderId,
            paymentResult.Error);
    }
    
    return new OrderResult { OrderId = request.OrderId };
}
```

## Distributed Tracing

### OpenTelemetry Integration

Configure OpenTelemetry for distributed tracing:

```csharp
var builder = WebApplication.CreateBuilder(args);

// Add OpenTelemetry
builder.Services.AddOpenTelemetry()
    .WithTracing(tracing =>
    {
        tracing
            .AddSource("Microsoft.DurableTask")
            .AddAspNetCoreInstrumentation()
            .AddHttpClientInstrumentation()
            .AddOtlpExporter(options =>
            {
                options.Endpoint = new Uri("http://localhost:4317");
            });
    });

// Add Durable Task with tracing
string connectionString = "Endpoint=http://localhost:8080;TaskHub=default;Authentication=None";
builder.Services.AddDurableTaskWorker(worker =>
{
    worker.UseDurableTaskScheduler(connectionString);
    worker.AddAllGeneratedTasks();
});
```

### Correlation IDs

Use correlation IDs to track requests across services:

```csharp
public override async Task<ProcessingResult> RunAsync(
    TaskOrchestrationContext context, 
    ProcessingRequest request)
{
    var logger = context.CreateReplaySafeLogger("ProcessingOrchestration");
    
    // Use instance ID as correlation ID
    string correlationId = context.InstanceId;
    
    logger.LogInformation(
        "Starting processing with CorrelationId: {CorrelationId}",
        correlationId);
    
    // Pass correlation ID to activities
    var activityInput = new ActivityInput
    {
        Data = request.Data,
        CorrelationId = correlationId
    };
    
    var result = await context.CallActivityAsync<ProcessingResult>(
        "ProcessData", 
        activityInput);
    
    return result;
}

// Activity with correlation
public override async Task<ProcessingResult> RunAsync(
    TaskActivityContext context, 
    ActivityInput input)
{
    using var scope = _logger.BeginScope(
        new Dictionary<string, object>
        {
            ["CorrelationId"] = input.CorrelationId,
            ["InstanceId"] = context.InstanceId
        });
    
    _logger.LogInformation("Processing data with correlation");
    
    // Process...
    return new ProcessingResult();
}
```

## Application Insights Integration

### Azure Functions with App Insights

```csharp
// In host.json
{
  "version": "2.0",
  "logging": {
    "applicationInsights": {
      "samplingSettings": {
        "isEnabled": true,
        "excludedTypes": "Request"
      }
    },
    "logLevel": {
      "Microsoft.DurableTask": "Information",
      "Host.Triggers.DurableTask": "Information"
    }
  }
}
```

### Custom Telemetry

```csharp
public class TelemetryActivity : TaskActivity<TelemetryInput, TelemetryResult>
{
    private readonly TelemetryClient _telemetryClient;
    
    public TelemetryActivity(TelemetryClient telemetryClient)
    {
        _telemetryClient = telemetryClient;
    }
    
    public override async Task<TelemetryResult> RunAsync(
        TaskActivityContext context, 
        TelemetryInput input)
    {
        var startTime = DateTime.UtcNow;
        var stopwatch = Stopwatch.StartNew();
        
        try
        {
            var result = await ProcessAsync(input);
            
            // Track successful operation
            _telemetryClient.TrackEvent("ActivityCompleted", 
                new Dictionary<string, string>
                {
                    ["InstanceId"] = context.InstanceId,
                    ["ActivityName"] = nameof(TelemetryActivity),
                    ["InputType"] = input.Type
                },
                new Dictionary<string, double>
                {
                    ["DurationMs"] = stopwatch.ElapsedMilliseconds
                });
            
            return result;
        }
        catch (Exception ex)
        {
            // Track failure
            _telemetryClient.TrackException(ex, 
                new Dictionary<string, string>
                {
                    ["InstanceId"] = context.InstanceId,
                    ["ActivityName"] = nameof(TelemetryActivity)
                });
            throw;
        }
    }
}
```

## Health Monitoring

### Health Check Implementation

```csharp
public class DurableTaskHealthCheck : IHealthCheck
{
    private readonly DurableTaskClient _client;
    
    public DurableTaskHealthCheck(DurableTaskClient client)
    {
        _client = client;
    }
    
    public async Task<HealthCheckResult> CheckHealthAsync(
        HealthCheckContext context,
        CancellationToken cancellationToken = default)
    {
        try
        {
            // Try to query instances as a health check
            var query = new OrchestrationQuery
            {
                PageSize = 1,
                RuntimeStatus = new[] { OrchestrationRuntimeStatus.Running }
            };
            
            await _client.GetAllInstancesAsync(query);
            
            return HealthCheckResult.Healthy("Durable Task service is responsive");
        }
        catch (Exception ex)
        {
            return HealthCheckResult.Unhealthy(
                "Durable Task service is not responsive",
                ex);
        }
    }
}

// Registration
builder.Services.AddHealthChecks()
    .AddCheck<DurableTaskHealthCheck>("durable-task");
```

### Metrics Collection

```csharp
public class OrchestrationMetrics
{
    private readonly DurableTaskClient _client;
    private readonly ILogger<OrchestrationMetrics> _logger;
    
    public async Task<MetricsSnapshot> CollectMetrics()
    {
        var runningQuery = new OrchestrationQuery
        {
            RuntimeStatus = new[] { OrchestrationRuntimeStatus.Running }
        };
        
        var pendingQuery = new OrchestrationQuery
        {
            RuntimeStatus = new[] { OrchestrationRuntimeStatus.Pending }
        };
        
        var failedQuery = new OrchestrationQuery
        {
            RuntimeStatus = new[] { OrchestrationRuntimeStatus.Failed },
            CreatedFrom = DateTime.UtcNow.AddHours(-1)
        };
        
        var running = await _client.GetAllInstancesAsync(runningQuery);
        var pending = await _client.GetAllInstancesAsync(pendingQuery);
        var failed = await _client.GetAllInstancesAsync(failedQuery);
        
        var snapshot = new MetricsSnapshot
        {
            Timestamp = DateTime.UtcNow,
            RunningCount = running.Instances.Count(),
            PendingCount = pending.Instances.Count(),
            FailedLastHour = failed.Instances.Count()
        };
        
        _logger.LogInformation(
            "Metrics: Running={Running}, Pending={Pending}, Failed(1h)={Failed}",
            snapshot.RunningCount,
            snapshot.PendingCount,
            snapshot.FailedLastHour);
        
        return snapshot;
    }
}
```

## Debugging Techniques

### Verbose Logging for Debugging

```csharp
// In appsettings.Development.json
{
  "Logging": {
    "LogLevel": {
      "Default": "Debug",
      "Microsoft.DurableTask": "Trace",
      "Microsoft.DurableTask.Worker": "Debug"
    }
  }
}
```

### Execution History Inspection

```csharp
public async Task InspectOrchestrationHistory(string instanceId)
{
    var metadata = await _client.GetInstanceAsync(
        instanceId, 
        getInputsAndOutputs: true);
    
    if (metadata == null)
    {
        Console.WriteLine($"Instance {instanceId} not found");
        return;
    }
    
    Console.WriteLine($"Instance: {metadata.InstanceId}");
    Console.WriteLine($"Name: {metadata.Name}");
    Console.WriteLine($"Status: {metadata.RuntimeStatus}");
    Console.WriteLine($"Created: {metadata.CreatedAt}");
    Console.WriteLine($"Last Updated: {metadata.LastUpdatedAt}");
    
    if (metadata.SerializedInput != null)
    {
        Console.WriteLine($"Input: {metadata.SerializedInput}");
    }
    
    if (metadata.SerializedOutput != null)
    {
        Console.WriteLine($"Output: {metadata.SerializedOutput}");
    }
    
    if (metadata.RuntimeStatus == OrchestrationRuntimeStatus.Failed)
    {
        Console.WriteLine($"Failure Details: {metadata.FailureDetails}");
    }
}
```

### Diagnostic Orchestration

Create an orchestration that logs detailed diagnostic information:

```csharp
[DurableTask(nameof(DiagnosticOrchestration))]
public class DiagnosticOrchestration : TaskOrchestrator<DiagnosticInput, DiagnosticReport>
{
    public override async Task<DiagnosticReport> RunAsync(
        TaskOrchestrationContext context, 
        DiagnosticInput input)
    {
        var logger = context.CreateReplaySafeLogger("DiagnosticOrchestration");
        var report = new DiagnosticReport
        {
            InstanceId = context.InstanceId,
            StartTime = context.CurrentUtcDateTime
        };
        
        logger.LogInformation("=== Diagnostic Run Started ===");
        logger.LogInformation("Instance ID: {InstanceId}", context.InstanceId);
        logger.LogInformation("Is Replaying: {IsReplaying}", context.IsReplaying);
        
        // Test activity execution
        var activityStart = context.CurrentUtcDateTime;
        try
        {
            var activityResult = await context.CallActivityAsync<string>(
                "DiagnosticActivity", 
                "test");
            report.ActivityStatus = "Success";
            logger.LogInformation("Activity completed: {Result}", activityResult);
        }
        catch (Exception ex)
        {
            report.ActivityStatus = $"Failed: {ex.Message}";
            logger.LogError(ex, "Activity failed");
        }
        
        // Test timer
        var timerStart = context.CurrentUtcDateTime;
        await context.CreateTimer(
            context.CurrentUtcDateTime.AddSeconds(1), 
            CancellationToken.None);
        report.TimerDuration = context.CurrentUtcDateTime - timerStart;
        logger.LogInformation("Timer completed in {Duration}", report.TimerDuration);
        
        report.EndTime = context.CurrentUtcDateTime;
        report.TotalDuration = report.EndTime - report.StartTime;
        
        logger.LogInformation("=== Diagnostic Run Completed ===");
        logger.LogInformation("Total Duration: {Duration}", report.TotalDuration);
        
        return report;
    }
}
```

## Performance Monitoring

### Activity Duration Tracking

```csharp
public override async Task<ProcessingResult> RunAsync(
    TaskOrchestrationContext context, 
    ProcessingInput input)
{
    var logger = context.CreateReplaySafeLogger("ProcessingOrchestration");
    var timings = new Dictionary<string, TimeSpan>();
    
    // Track each activity duration
    var start = context.CurrentUtcDateTime;
    await context.CallActivityAsync("Step1", input);
    timings["Step1"] = context.CurrentUtcDateTime - start;
    
    start = context.CurrentUtcDateTime;
    await context.CallActivityAsync("Step2", input);
    timings["Step2"] = context.CurrentUtcDateTime - start;
    
    start = context.CurrentUtcDateTime;
    await context.CallActivityAsync("Step3", input);
    timings["Step3"] = context.CurrentUtcDateTime - start;
    
    // Log timing summary
    foreach (var (step, duration) in timings)
    {
        logger.LogInformation(
            "Step {Step} duration: {Duration}",
            step,
            duration);
    }
    
    return new ProcessingResult
    {
        Success = true,
        ActivityTimings = timings
    };
}
```

### Custom Metrics Emission

```csharp
public class MetricsEmittingActivity : TaskActivity<MetricInput, MetricResult>
{
    private readonly IMetricRecorder _metrics;
    
    public MetricsEmittingActivity(IMetricRecorder metrics)
    {
        _metrics = metrics;
    }
    
    public override async Task<MetricResult> RunAsync(
        TaskActivityContext context, 
        MetricInput input)
    {
        var stopwatch = Stopwatch.StartNew();
        
        try
        {
            var result = await ProcessAsync(input);
            
            stopwatch.Stop();
            
            // Record metrics
            _metrics.Record("activity.duration.ms", stopwatch.ElapsedMilliseconds, 
                new[] { $"activity:{nameof(MetricsEmittingActivity)}" });
            _metrics.Increment("activity.success", 
                new[] { $"activity:{nameof(MetricsEmittingActivity)}" });
            
            return result;
        }
        catch
        {
            _metrics.Increment("activity.failure", 
                new[] { $"activity:{nameof(MetricsEmittingActivity)}" });
            throw;
        }
    }
}
```

## Best Practices

### 1. Always Use Replay-Safe Logging in Orchestrations

```csharp
// ❌ Will log on every replay
_logger.LogInformation("Message");

// ✅ Logs only once
var logger = context.CreateReplaySafeLogger("Orchestration");
logger.LogInformation("Message");
```

### 2. Include Context in Log Messages

```csharp
// ❌ Missing context
logger.LogError("Failed to process");

// ✅ With context
logger.LogError(
    "Failed to process order {OrderId} in orchestration {InstanceId}",
    order.Id,
    context.InstanceId);
```

### 3. Use Appropriate Log Levels

```csharp
logger.LogTrace("Detailed debug info");      // Very verbose
logger.LogDebug("Debug information");         // Development
logger.LogInformation("Normal operation");    // Production info
logger.LogWarning("Potential issue");         // Warnings
logger.LogError(ex, "Error occurred");        // Errors
logger.LogCritical("System failure");         // Critical
```

### 4. Configure Log Retention

```json
{
  "Serilog": {
    "WriteTo": [
      {
        "Name": "File",
        "Args": {
          "path": "logs/orchestrations-.log",
          "rollingInterval": "Day",
          "retainedFileCountLimit": 30
        }
      }
    ]
  }
}
```

## Next Steps

- [Unit Testing](Unit-Testing.md) - Test your orchestrations
- [Analyzers](Analyzers.md) - Static analysis
- [Error Handling and Compensation](Error-Handling-and-Compensation.md) - Handle errors

---
layout: default
title: Azure Functions Integration
parent: Hosting
nav_order: 3
description: "Use Durable Task SDK with Azure Functions isolated worker"
---

# Azure Functions Integration

The Durable Task .NET SDK integrates seamlessly with Azure Functions using the .NET isolated worker model. This guide covers setup, triggers, bindings, and best practices.

## Overview

Azure Functions provides a serverless hosting environment for Durable Task orchestrations with:
- Automatic scaling
- Built-in monitoring
- Pay-per-use pricing
- Integration with Azure services

## Getting Started

### Required Packages

```xml
<PackageReference Include="Microsoft.Azure.Functions.Worker" Version="1.20.0" />
<PackageReference Include="Microsoft.Azure.Functions.Worker.Sdk" Version="1.16.2" />
<PackageReference Include="Microsoft.Azure.Functions.Worker.Extensions.DurableTask" Version="1.1.0" />
```

### Project Configuration

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>net8.0</TargetFramework>
    <AzureFunctionsVersion>v4</AzureFunctionsVersion>
    <OutputType>Exe</OutputType>
  </PropertyGroup>
</Project>
```

### Program.cs Setup

```csharp
using Microsoft.Azure.Functions.Worker;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;

var host = new HostBuilder()
    .ConfigureFunctionsWorkerDefaults()
    .ConfigureServices(services =>
    {
        // Add Application Insights
        services.AddApplicationInsightsTelemetryWorkerService();
        services.ConfigureFunctionsApplicationInsights();
        
        // Add your services
        services.AddSingleton<IPaymentService, PaymentService>();
    })
    .Build();

host.Run();
```

### local.settings.json

```json
{
  "IsEncrypted": false,
  "Values": {
    "AzureWebJobsStorage": "UseDevelopmentStorage=true",
    "FUNCTIONS_WORKER_RUNTIME": "dotnet-isolated"
  }
}
```

## Durable Client Binding

### HTTP Starter Function

```csharp
public static class HttpStarters
{
    [Function("StartOrchestration")]
    public static async Task<HttpResponseData> StartOrchestration(
        [HttpTrigger(AuthorizationLevel.Function, "post")] HttpRequestData req,
        [DurableClient] DurableTaskClient client,
        FunctionContext executionContext)
    {
        var logger = executionContext.GetLogger("StartOrchestration");
        
        // Read input from request body
        var input = await req.ReadFromJsonAsync<OrchestrationInput>();
        
        // Schedule the orchestration
        string instanceId = await client.ScheduleNewOrchestrationInstanceAsync(
            nameof(MyOrchestration),
            input);
        
        logger.LogInformation("Started orchestration with ID = '{instanceId}'", instanceId);
        
        // Return check status response
        return client.CreateCheckStatusResponse(req, instanceId);
    }
}
```

### Status Query Function

```csharp
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
        metadata.Name,
        Status = metadata.RuntimeStatus.ToString(),
        metadata.CreatedAt,
        metadata.LastUpdatedAt,
        Input = metadata.SerializedInput,
        Output = metadata.SerializedOutput
    });
    
    return response;
}
```

### Terminate Function

```csharp
[Function("TerminateOrchestration")]
public static async Task<HttpResponseData> Terminate(
    [HttpTrigger(AuthorizationLevel.Function, "post", Route = "orchestrations/{instanceId}/terminate")] 
    HttpRequestData req,
    string instanceId,
    [DurableClient] DurableTaskClient client)
{
    var body = await req.ReadFromJsonAsync<TerminateRequest>();
    await client.TerminateInstanceAsync(instanceId, body?.Reason ?? "Terminated via API");
    
    return req.CreateResponse(HttpStatusCode.OK);
}
```

### Raise Event Function

```csharp
[Function("RaiseEvent")]
public static async Task<HttpResponseData> RaiseEvent(
    [HttpTrigger(AuthorizationLevel.Function, "post", Route = "orchestrations/{instanceId}/events/{eventName}")] 
    HttpRequestData req,
    string instanceId,
    string eventName,
    [DurableClient] DurableTaskClient client)
{
    string body = await new StreamReader(req.Body).ReadToEndAsync();
    await client.RaiseEventAsync(instanceId, eventName, body);
    
    return req.CreateResponse(HttpStatusCode.OK);
}
```

## Orchestration Function

### Basic Orchestration

```csharp
[Function(nameof(OrderProcessingOrchestration))]
public static async Task<OrderResult> OrderProcessingOrchestration(
    [OrchestrationTrigger] TaskOrchestrationContext context)
{
    var logger = context.CreateReplaySafeLogger(nameof(OrderProcessingOrchestration));
    var input = context.GetInput<OrderInput>();
    
    logger.LogInformation("Processing order {OrderId}", input.OrderId);
    
    // Call activities
    var validation = await context.CallActivityAsync<ValidationResult>(
        nameof(ValidateOrderActivity), 
        input);
    
    if (!validation.IsValid)
    {
        return new OrderResult { Status = "ValidationFailed", Error = validation.Error };
    }
    
    var payment = await context.CallActivityAsync<PaymentResult>(
        nameof(ProcessPaymentActivity), 
        input.Payment);
    
    var shipping = await context.CallActivityAsync<ShippingResult>(
        nameof(CreateShipmentActivity), 
        input.Shipping);
    
    return new OrderResult
    {
        OrderId = input.OrderId,
        Status = "Completed",
        TrackingNumber = shipping.TrackingNumber
    };
}
```

### Orchestration with Retry

```csharp
[Function(nameof(ReliableOrchestration))]
public static async Task<string> ReliableOrchestration(
    [OrchestrationTrigger] TaskOrchestrationContext context)
{
    var input = context.GetInput<string>();
    
    var retryOptions = new TaskOptions
    {
        Retry = new RetryPolicy(
            maxNumberOfAttempts: 3,
            firstRetryInterval: TimeSpan.FromSeconds(5))
    };
    
    return await context.CallActivityAsync<string>(
        nameof(UnreliableActivity), 
        input, 
        retryOptions);
}
```

## Activity Functions

### Basic Activity

```csharp
[Function(nameof(ValidateOrderActivity))]
public static ValidationResult ValidateOrderActivity(
    [ActivityTrigger] OrderInput input,
    FunctionContext executionContext)
{
    var logger = executionContext.GetLogger(nameof(ValidateOrderActivity));
    logger.LogInformation("Validating order {OrderId}", input.OrderId);
    
    // Validation logic
    if (input.Amount <= 0)
    {
        return new ValidationResult { IsValid = false, Error = "Invalid amount" };
    }
    
    return new ValidationResult { IsValid = true };
}
```

### Activity with Dependency Injection

```csharp
public class PaymentActivities
{
    private readonly IPaymentService _paymentService;
    private readonly ILogger<PaymentActivities> _logger;
    
    public PaymentActivities(
        IPaymentService paymentService, 
        ILogger<PaymentActivities> logger)
    {
        _paymentService = paymentService;
        _logger = logger;
    }
    
    [Function(nameof(ProcessPaymentActivity))]
    public async Task<PaymentResult> ProcessPaymentActivity(
        [ActivityTrigger] PaymentInput input)
    {
        _logger.LogInformation("Processing payment for amount {Amount}", input.Amount);
        
        return await _paymentService.ProcessAsync(input);
    }
}
```

### Activity with Context

```csharp
[Function(nameof(ContextAwareActivity))]
public static async Task<ActivityResult> ContextAwareActivity(
    [ActivityTrigger] string input,
    FunctionContext executionContext)
{
    var logger = executionContext.GetLogger(nameof(ContextAwareActivity));
    
    // Get orchestration context info
    var instanceId = executionContext.BindingContext.BindingData["instanceId"]?.ToString();
    
    logger.LogInformation(
        "Activity running for orchestration {InstanceId}", 
        instanceId);
    
    return new ActivityResult { Success = true };
}
```

## Entity Functions

### Basic Entity

```csharp
[Function(nameof(CounterEntity))]
public static Task CounterEntity(
    [EntityTrigger] TaskEntityDispatcher dispatcher)
{
    return dispatcher.DispatchAsync<CounterState>(operation =>
    {
        return operation.Name switch
        {
            "add" => () => operation.State.Value += operation.GetInput<int>(),
            "get" => () => operation.State.Value,
            "reset" => () => operation.State.Value = 0,
            _ => throw new InvalidOperationException($"Unknown operation: {operation.Name}")
        };
    });
}

public class CounterState
{
    public int Value { get; set; }
}
```

### Class-Based Entity

```csharp
public class ShoppingCart : TaskEntity<CartState>
{
    public void AddItem(CartItem item)
    {
        State.Items.Add(item);
    }
    
    public void RemoveItem(string productId)
    {
        State.Items.RemoveAll(i => i.ProductId == productId);
    }
    
    public CartState GetCart() => State;
    
    public void Clear()
    {
        State.Items.Clear();
    }
    
    [Function(nameof(ShoppingCart))]
    public static Task RunEntityAsync(
        [EntityTrigger] TaskEntityDispatcher dispatcher)
    {
        return dispatcher.DispatchAsync<ShoppingCart>();
    }
}

public class CartState
{
    public List<CartItem> Items { get; set; } = new();
}
```

## Timer-Triggered Orchestrations

### Scheduled Orchestration

```csharp
[Function("ScheduledBatchProcessor")]
public static async Task ScheduledBatchProcessor(
    [TimerTrigger("0 0 * * * *")] TimerInfo timerInfo,  // Every hour
    [DurableClient] DurableTaskClient client,
    FunctionContext executionContext)
{
    var logger = executionContext.GetLogger("ScheduledBatchProcessor");
    
    // Use timestamp-based instance ID to prevent duplicates
    string instanceId = $"batch-{DateTime.UtcNow:yyyyMMddHH}";
    
    try
    {
        await client.ScheduleNewOrchestrationInstanceAsync(
            nameof(BatchProcessingOrchestration),
            new BatchInput { Timestamp = DateTime.UtcNow },
            new StartOrchestrationOptions { InstanceId = instanceId });
        
        logger.LogInformation("Started batch processing {InstanceId}", instanceId);
    }
    catch (InvalidOperationException ex) when (ex.Message.Contains("already exists"))
    {
        logger.LogInformation("Batch {InstanceId} already running", instanceId);
    }
}
```

### Maintenance Job

```csharp
[Function("DailyCleanup")]
public static async Task DailyCleanup(
    [TimerTrigger("0 0 2 * * *")] TimerInfo timerInfo,  // 2 AM daily
    [DurableClient] DurableTaskClient client,
    FunctionContext executionContext)
{
    var logger = executionContext.GetLogger("DailyCleanup");
    
    // Purge completed instances older than 30 days
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
    
    logger.LogInformation("Purged {Count} instances", result.PurgedInstanceCount);
}
```

## Queue-Triggered Orchestrations

### Start from Queue Message

```csharp
[Function("QueueTriggeredStarter")]
public static async Task QueueTriggeredStarter(
    [QueueTrigger("orders")] OrderMessage message,
    [DurableClient] DurableTaskClient client,
    FunctionContext executionContext)
{
    var logger = executionContext.GetLogger("QueueTriggeredStarter");
    
    string instanceId = $"order-{message.OrderId}";
    
    await client.ScheduleNewOrchestrationInstanceAsync(
        nameof(OrderProcessingOrchestration),
        message.ToOrderInput(),
        new StartOrchestrationOptions { InstanceId = instanceId });
    
    logger.LogInformation("Started order processing for {OrderId}", message.OrderId);
}
```

## Configuration

### host.json Settings

```json
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
      "DurableTask.Core": "Warning",
      "DurableTask.AzureStorage": "Warning"
    }
  },
  "extensions": {
    "durableTask": {
      "hubName": "MyTaskHub",
      "storageProvider": {
        "partitionCount": 4
      },
      "maxConcurrentActivityFunctions": 10,
      "maxConcurrentOrchestratorFunctions": 5
    }
  }
}
```

### Azure Storage Configuration

```json
{
  "extensions": {
    "durableTask": {
      "storageProvider": {
        "connectionStringName": "AzureWebJobsStorage",
        "controlQueueBatchSize": 32,
        "partitionCount": 4,
        "trackingStoreConnectionStringName": "TrackingStorage"
      }
    }
  }
}
```

## Monitoring and Diagnostics

### Custom Telemetry

```csharp
[Function(nameof(TelemetryOrchestration))]
public static async Task<string> TelemetryOrchestration(
    [OrchestrationTrigger] TaskOrchestrationContext context)
{
    var logger = context.CreateReplaySafeLogger(nameof(TelemetryOrchestration));
    var input = context.GetInput<string>();
    
    var startTime = context.CurrentUtcDateTime;
    
    logger.LogInformation(
        "Starting orchestration {InstanceId} at {StartTime}",
        context.InstanceId,
        startTime);
    
    var result = await context.CallActivityAsync<string>(
        nameof(ProcessActivity), 
        input);
    
    var duration = context.CurrentUtcDateTime - startTime;
    
    logger.LogInformation(
        "Completed orchestration {InstanceId} in {Duration}ms",
        context.InstanceId,
        duration.TotalMilliseconds);
    
    return result;
}
```

### Health Check Endpoint

```csharp
[Function("HealthCheck")]
public static async Task<HttpResponseData> HealthCheck(
    [HttpTrigger(AuthorizationLevel.Anonymous, "get", Route = "health")] 
    HttpRequestData req,
    [DurableClient] DurableTaskClient client)
{
    try
    {
        // Test client connectivity
        var query = new OrchestrationQuery { PageSize = 1 };
        await client.GetAllInstancesAsync(query);
        
        var response = req.CreateResponse(HttpStatusCode.OK);
        await response.WriteAsJsonAsync(new { status = "healthy" });
        return response;
    }
    catch (Exception ex)
    {
        var response = req.CreateResponse(HttpStatusCode.ServiceUnavailable);
        await response.WriteAsJsonAsync(new { status = "unhealthy", error = ex.Message });
        return response;
    }
}
```

## Best Practices

### 1. Use Instance ID for Idempotency

```csharp
// ✅ Deterministic instance ID prevents duplicates
string instanceId = $"order-{orderId}";
await client.ScheduleNewOrchestrationInstanceAsync(
    nameof(OrderOrchestration),
    input,
    new StartOrchestrationOptions { InstanceId = instanceId });
```

### 2. Handle Singleton Orchestrations

```csharp
[Function("StartSingletonOrchestration")]
public static async Task<HttpResponseData> StartSingleton(
    [HttpTrigger(AuthorizationLevel.Function, "post")] HttpRequestData req,
    [DurableClient] DurableTaskClient client)
{
    const string instanceId = "singleton-orchestration";
    
    var existing = await client.GetInstanceAsync(instanceId);
    
    if (existing?.RuntimeStatus == OrchestrationRuntimeStatus.Running)
    {
        var response = req.CreateResponse(HttpStatusCode.Conflict);
        await response.WriteAsJsonAsync(new { error = "Already running" });
        return response;
    }
    
    await client.ScheduleNewOrchestrationInstanceAsync(
        nameof(SingletonOrchestration),
        null,
        new StartOrchestrationOptions { InstanceId = instanceId });
    
    return client.CreateCheckStatusResponse(req, instanceId);
}
```

### 3. Use Replay-Safe Logging

```csharp
// ✅ Won't duplicate logs during replay
var logger = context.CreateReplaySafeLogger("MyOrchestration");
logger.LogInformation("Processing {Input}", input);
```

## Next Steps

- [ASP.NET Core Integration](AspNetCore-Integration.md) - Host in ASP.NET Core
- [Source Generators](Source-Generators.md) - Generate boilerplate code
- [Diagnostics and Logging](Diagnostics-and-Logging.md) - Monitoring

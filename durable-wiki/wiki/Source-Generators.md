# Source Generators

The Durable Task .NET SDK includes source generators that reduce boilerplate code and provide compile-time validation for your orchestrations, activities, and entities with **Durable Task Scheduler**.

## Overview

Source generators provide:
- Automatic task registration with `AddAllGeneratedTasks()`
- Type-safe orchestration/activity calls
- Compile-time validation
- Reduced boilerplate code

## Installation

Include the generators package:

```xml
<PackageReference Include="Microsoft.DurableTask.Generators" Version="1.0.0" PrivateAssets="all" />
```

## Automatic Registration

With source generators, use `AddAllGeneratedTasks()` for automatic registration:

```csharp
builder.Services.AddDurableTaskWorker(worker =>
{
    worker.UseDurableTaskScheduler(connectionString);
    
    // Automatically register all tasks with [DurableTask] attribute
    worker.AddAllGeneratedTasks();
});
```

## The DurableTask Attribute

The `[DurableTask]` attribute marks classes for source generation:

```csharp
[DurableTask(nameof(MyOrchestration))]
public class MyOrchestration : TaskOrchestrator<string, string>
{
    public override async Task<string> RunAsync(
        TaskOrchestrationContext context, 
        string input)
    {
        return await context.CallActivityAsync<string>(
            nameof(MyActivity), 
            input);
    }
}

[DurableTask(nameof(MyActivity))]
public class MyActivity : TaskActivity<string, string>
{
    public override Task<string> RunAsync(
        TaskActivityContext context, 
        string input)
    {
        return Task.FromResult(input.ToUpper());
    }
}
```

## Generated Code

### Task Registration Extensions

The generator creates extension methods for registration:

```csharp
// Generated code (simplified)
public static class DurableTaskWorkerBuilderExtensions
{
    public static DurableTaskWorkerBuilder AddMyOrchestration(
        this DurableTaskWorkerBuilder builder)
    {
        return builder.AddTasks<MyOrchestration>();
    }
    
    public static DurableTaskWorkerBuilder AddMyActivity(
        this DurableTaskWorkerBuilder builder)
    {
        return builder.AddTasks<MyActivity>();
    }
    
    public static DurableTaskWorkerBuilder AddAllGeneratedTasks(
        this DurableTaskWorkerBuilder builder)
    {
        builder.AddTasks<MyOrchestration>();
        builder.AddTasks<MyActivity>();
        // ... all other generated tasks
        return builder;
    }
}
```

### Usage

```csharp
string connectionString = "Endpoint=http://localhost:8080;TaskHub=default;Authentication=None";

builder.Services.AddDurableTaskWorker(workerBuilder =>
{
    workerBuilder.UseDurableTaskScheduler(connectionString);
    
    // Add individual tasks
    workerBuilder.AddMyOrchestration();
    workerBuilder.AddMyActivity();
    
    // Or add all generated tasks at once (recommended)
    workerBuilder.AddAllGeneratedTasks();
});
```

## Type-Safe Calls

### Generated Task Names

Source generators create constants for task names:

```csharp
// Generated code
public static class DurableTaskNames
{
    public const string MyOrchestration = "MyOrchestration";
    public const string MyActivity = "MyActivity";
    public const string PaymentOrchestration = "PaymentOrchestration";
    public const string ProcessPaymentActivity = "ProcessPaymentActivity";
}
```

### Usage in Orchestrations

```csharp
[DurableTask(nameof(OrderOrchestration))]
public class OrderOrchestration : TaskOrchestrator<OrderInput, OrderResult>
{
    public override async Task<OrderResult> RunAsync(
        TaskOrchestrationContext context, 
        OrderInput input)
    {
        // Use generated names for type safety
        var paymentResult = await context.CallActivityAsync<PaymentResult>(
            DurableTaskNames.ProcessPaymentActivity, 
            input.Payment);
        
        var shippingResult = await context.CallSubOrchestratorAsync<ShippingResult>(
            DurableTaskNames.ShippingOrchestration, 
            input.Shipping);
        
        return new OrderResult
        {
            PaymentId = paymentResult.TransactionId,
            TrackingNumber = shippingResult.TrackingNumber
        };
    }
}
```

## Generated Type-Safe Extensions

### Activity Call Extensions

```csharp
// Generated extension methods
public static class TaskOrchestrationContextExtensions
{
    public static Task<PaymentResult> CallProcessPaymentActivityAsync(
        this TaskOrchestrationContext context,
        PaymentInput input,
        TaskOptions? options = null)
    {
        return context.CallActivityAsync<PaymentResult>(
            "ProcessPaymentActivity",
            input,
            options);
    }
    
    public static Task<ShippingResult> CallShippingOrchestrationAsync(
        this TaskOrchestrationContext context,
        ShippingInput input,
        TaskOptions? options = null)
    {
        return context.CallSubOrchestratorAsync<ShippingResult>(
            "ShippingOrchestration",
            input,
            options);
    }
}
```

### Usage

```csharp
[DurableTask(nameof(OrderOrchestration))]
public class OrderOrchestration : TaskOrchestrator<OrderInput, OrderResult>
{
    public override async Task<OrderResult> RunAsync(
        TaskOrchestrationContext context, 
        OrderInput input)
    {
        // Type-safe generated extension methods
        var paymentResult = await context.CallProcessPaymentActivityAsync(input.Payment);
        var shippingResult = await context.CallShippingOrchestrationAsync(input.Shipping);
        
        return new OrderResult
        {
            PaymentId = paymentResult.TransactionId,
            TrackingNumber = shippingResult.TrackingNumber
        };
    }
}
```

## Client Extensions

### Generated Client Methods

```csharp
// Generated extensions for DurableTaskClient
public static class DurableTaskClientExtensions
{
    public static Task<string> ScheduleOrderOrchestrationAsync(
        this DurableTaskClient client,
        OrderInput input,
        StartOrchestrationOptions? options = null)
    {
        return client.ScheduleNewOrchestrationInstanceAsync(
            "OrderOrchestration",
            input,
            options);
    }
    
    public static Task<string> SchedulePaymentOrchestrationAsync(
        this DurableTaskClient client,
        PaymentInput input,
        StartOrchestrationOptions? options = null)
    {
        return client.ScheduleNewOrchestrationInstanceAsync(
            "PaymentOrchestration",
            input,
            options);
    }
}
```

### Usage in Controllers

```csharp
[ApiController]
[Route("api/orders")]
public class OrdersController : ControllerBase
{
    private readonly DurableTaskClient _client;

    public OrdersController(DurableTaskClient client)
    {
        _client = client;
    }

    [HttpPost]
    public async Task<ActionResult> CreateOrder([FromBody] OrderInput input)
    {
        // Use generated type-safe method
        string instanceId = await _client.ScheduleOrderOrchestrationAsync(
            input,
            new StartOrchestrationOptions
            {
                InstanceId = $"order-{Guid.NewGuid():N}"
            });

        return Accepted(new { InstanceId = instanceId });
    }
}
```

## Entity Generators

### Entity Operations

```csharp
[DurableTask(nameof(CounterEntity))]
public class CounterEntity : TaskEntity<int>
{
    public void Increment() => State++;
    
    public void Add(int amount) => State += amount;
    
    public int Get() => State;
    
    public void Reset() => State = 0;
}

// Generated extension methods
public static class CounterEntityExtensions
{
    public static Task IncrementCounterAsync(
        this TaskOrchestrationContext context,
        EntityInstanceId entityId)
    {
        return context.CallEntityAsync(entityId, "Increment");
    }
    
    public static Task AddToCounterAsync(
        this TaskOrchestrationContext context,
        EntityInstanceId entityId,
        int amount)
    {
        return context.CallEntityAsync(entityId, "Add", amount);
    }
    
    public static Task<int> GetCounterAsync(
        this TaskOrchestrationContext context,
        EntityInstanceId entityId)
    {
        return context.CallEntityAsync<int>(entityId, "Get");
    }
}
```

### Usage

```csharp
[DurableTask(nameof(CounterOrchestration))]
public class CounterOrchestration : TaskOrchestrator<string, int>
{
    public override async Task<int> RunAsync(
        TaskOrchestrationContext context, 
        string counterId)
    {
        var entityId = new EntityInstanceId(nameof(CounterEntity), counterId);
        
        // Use generated type-safe methods
        await context.IncrementCounterAsync(entityId);
        await context.AddToCounterAsync(entityId, 5);
        
        return await context.GetCounterAsync(entityId);
    }
}
```

## Configuration

### Generator Options

Configure generator behavior in your project file:

```xml
<PropertyGroup>
  <!-- Enable/disable specific generators -->
  <DurableTaskGenerateExtensions>true</DurableTaskGenerateExtensions>
  <DurableTaskGenerateNames>true</DurableTaskGenerateNames>
</PropertyGroup>
```

### Namespace Configuration

```csharp
// Generated code uses the same namespace as your tasks
namespace MyApp.Workflows
{
    [DurableTask(nameof(MyOrchestration))]
    public class MyOrchestration : TaskOrchestrator<string, string>
    {
        // ...
    }
}

// Generated extensions in same namespace
namespace MyApp.Workflows
{
    public static class DurableTaskExtensions
    {
        // ...
    }
}
```

## Compile-Time Validation

### Type Checking

The generator validates:
- Input/output types are serializable
- Method signatures match expected patterns
- Task names are unique

```csharp
// ❌ Compile error: Task is not serializable
[DurableTask(nameof(BadOrchestration))]
public class BadOrchestration : TaskOrchestrator<Task, string>  // Error!
{
    // ...
}

// ❌ Compile error: Duplicate task name
[DurableTask("MyTask")]
public class MyOrchestration1 : TaskOrchestrator<string, string> { }

[DurableTask("MyTask")]  // Error: duplicate name
public class MyOrchestration2 : TaskOrchestrator<string, string> { }
```

### Missing Attribute Warning

```csharp
// ⚠️ Warning: Class implements TaskOrchestrator but missing attribute
public class UnmarkedOrchestration : TaskOrchestrator<string, string>
{
    // Generator will warn about missing [DurableTask] attribute
}
```

## Best Practices

### 1. Use nameof() for Task Names

```csharp
// ✅ Refactoring-safe
[DurableTask(nameof(PaymentOrchestration))]
public class PaymentOrchestration : TaskOrchestrator<PaymentInput, PaymentResult>
{
}

// ❌ Magic string - can get out of sync
[DurableTask("PaymentOrchestration")]
public class PaymentOrchestration : TaskOrchestrator<PaymentInput, PaymentResult>
{
}
```

### 2. Keep Tasks in Organized Namespaces

```csharp
namespace MyApp.Workflows.Orders
{
    [DurableTask(nameof(OrderOrchestration))]
    public class OrderOrchestration : TaskOrchestrator<OrderInput, OrderResult> { }
}

namespace MyApp.Workflows.Payments
{
    [DurableTask(nameof(PaymentOrchestration))]
    public class PaymentOrchestration : TaskOrchestrator<PaymentInput, PaymentResult> { }
}
```

### 3. Use Generated Extensions Consistently

```csharp
// ✅ Use generated type-safe methods
var result = await context.CallProcessPaymentActivityAsync(input);

// ❌ Avoid string-based calls when generators are available
var result = await context.CallActivityAsync<PaymentResult>("ProcessPaymentActivity", input);
```

### 4. Register All Tasks at Startup

```csharp
// ✅ Use generated AddAllGeneratedTasks
string connectionString = "Endpoint=http://localhost:8080;TaskHub=default;Authentication=None";

builder.Services.AddDurableTaskWorker(workerBuilder =>
{
    workerBuilder.UseDurableTaskScheduler(connectionString);
    workerBuilder.AddAllGeneratedTasks();
});
```

## Viewing Generated Code

### In Visual Studio

1. Expand Dependencies > Analyzers > Microsoft.DurableTask.Generators
2. View generated files

### In Project

Add to project file to emit generated files:

```xml
<PropertyGroup>
  <EmitCompilerGeneratedFiles>true</EmitCompilerGeneratedFiles>
  <CompilerGeneratedFilesOutputPath>Generated</CompilerGeneratedFilesOutputPath>
</PropertyGroup>
```

## Troubleshooting

### Generator Not Running

1. Ensure package is referenced correctly
2. Rebuild the solution
3. Check for analyzer errors

### Missing Extensions

1. Verify `[DurableTask]` attribute is applied
2. Check namespace imports
3. Rebuild to trigger generation

### Type Mismatch Errors

1. Ensure input/output types match between definition and usage
2. Check for nullable reference type mismatches
3. Verify serialization compatibility

## Next Steps

- [Analyzers](Analyzers.md) - Static analysis rules
- [Azure Functions Integration](Azure-Functions-Integration.md) - Use with Functions
- [ASP.NET Core Integration](AspNetCore-Integration.md) - Use with ASP.NET Core

---
layout: default
title: Orchestration Constraints
nav_order: 4
description: "Understanding determinism and constraints in durable orchestrations"
---

# Orchestration Constraints

Orchestrations have specific constraints due to their replay-based execution model. Understanding and following these constraints is essential for building reliable durable workflows.

## The Replay Model

When an orchestration resumes after waiting (for activities, timers, events, etc.), it replays from the beginning. The framework:

1. Replays all code up to the awaited point
2. Uses cached results from history for completed operations
3. Continues execution from where it left off

This means orchestration code **must be deterministic** - it must produce the same sequence of operations every time it runs.

## Constraint Categories

### 1. No Direct I/O

Orchestrations must not perform I/O operations directly.

**❌ Violations:**
```csharp
public override async Task<string> RunAsync(
    TaskOrchestrationContext context, 
    string input)
{
    // ❌ File I/O
    var data = await File.ReadAllTextAsync("config.json");
    
    // ❌ Network calls
    using var client = new HttpClient();
    var response = await client.GetAsync("https://api.example.com");
    
    // ❌ Database access
    using var connection = new SqlConnection(connectionString);
    await connection.OpenAsync();
    
    return data;
}
```

**✅ Correct Approach:**
```csharp
public override async Task<string> RunAsync(
    TaskOrchestrationContext context, 
    string input)
{
    // ✅ Use activities for I/O
    var data = await context.CallActivityAsync<string>("ReadConfig", "config.json");
    var apiData = await context.CallActivityAsync<string>("CallApi", "https://api.example.com");
    var dbData = await context.CallActivityAsync<string>("QueryDatabase", input);
    
    return data;
}
```

### 2. No Non-Deterministic Operations

Avoid operations that produce different results on each execution.

**❌ Violations:**
```csharp
public override async Task<string> RunAsync(
    TaskOrchestrationContext context, 
    string input)
{
    // ❌ Current time
    var now = DateTime.Now;
    var utcNow = DateTime.UtcNow;
    var offset = DateTimeOffset.Now;
    
    // ❌ Random numbers
    var random = new Random().Next();
    
    // ❌ GUIDs
    var id = Guid.NewGuid();
    
    // ❌ Environment variables (can change)
    var config = Environment.GetEnvironmentVariable("CONFIG");
    
    return $"{now}-{random}-{id}";
}
```

**✅ Correct Approach:**
```csharp
public override async Task<string> RunAsync(
    TaskOrchestrationContext context, 
    string input)
{
    // ✅ Use context for deterministic time
    var now = context.CurrentUtcDateTime;
    
    // ✅ Use context for deterministic GUIDs
    var id = context.NewGuid();
    
    // ✅ Get random/config from activities
    var random = await context.CallActivityAsync<int>("GenerateRandom", null);
    var config = await context.CallActivityAsync<string>("GetConfig", "CONFIG");
    
    return $"{now}-{random}-{id}";
}
```

### 3. No Threading Operations

Orchestrations run on a single thread and must not create additional threads.

**❌ Violations:**
```csharp
public override async Task<string> RunAsync(
    TaskOrchestrationContext context, 
    string input)
{
    // ❌ Thread.Sleep blocks and isn't durable
    Thread.Sleep(1000);
    
    // ❌ Creating new threads
    var thread = new Thread(() => DoWork());
    thread.Start();
    
    // ❌ Task.Run starts work on thread pool
    var result = await Task.Run(() => ExpensiveComputation());
    
    // ❌ Parallel operations
    Parallel.ForEach(items, item => Process(item));
    
    return result;
}
```

**✅ Correct Approach:**
```csharp
public override async Task<string> RunAsync(
    TaskOrchestrationContext context, 
    string input)
{
    // ✅ Use durable timer for delays
    await context.CreateTimer(
        context.CurrentUtcDateTime.AddSeconds(1), 
        CancellationToken.None);
    
    // ✅ Use activity for CPU-intensive work
    var result = await context.CallActivityAsync<string>(
        "ExpensiveComputation", 
        input);
    
    // ✅ Use fan-out for parallel work
    var tasks = items.Select(item =>
        context.CallActivityAsync<string>("ProcessItem", item));
    var results = await Task.WhenAll(tasks);
    
    return result;
}
```

### 4. No Blocking Operations

Never block the orchestration thread.

**❌ Violations:**
```csharp
public override async Task<string> RunAsync(
    TaskOrchestrationContext context, 
    string input)
{
    // ❌ .Wait() blocks
    var task = SomeAsyncMethod();
    task.Wait();
    
    // ❌ .Result blocks
    var result = task.Result;
    
    // ❌ .GetAwaiter().GetResult() blocks
    var value = task.GetAwaiter().GetResult();
    
    // ❌ ConfigureAwait(false) can cause issues
    var data = await SomeMethod().ConfigureAwait(false);
    
    return result;
}
```

**✅ Correct Approach:**
```csharp
public override async Task<string> RunAsync(
    TaskOrchestrationContext context, 
    string input)
{
    // ✅ Always use await
    var result = await SomeAsyncMethod();
    var data = await SomeMethod();
    
    return result;
}
```

### 5. No Static State Modification

Static state is shared and non-deterministic.

**❌ Violations:**
```csharp
public class BadOrchestration : TaskOrchestrator<string, string>
{
    private static int _counter = 0;  // ❌ Static field
    private static List<string> _cache = new();  // ❌ Static collection
    
    public override async Task<string> RunAsync(
        TaskOrchestrationContext context, 
        string input)
    {
        // ❌ Modifying static state
        _counter++;
        _cache.Add(input);
        
        return _counter.ToString();
    }
}
```

**✅ Correct Approach:**
```csharp
public class GoodOrchestration : TaskOrchestrator<OrchestrationInput, string>
{
    public override async Task<string> RunAsync(
        TaskOrchestrationContext context, 
        OrchestrationInput input)
    {
        // ✅ Pass state through input
        var currentCount = input.Counter;
        
        // ✅ Or use entities for shared state
        var count = await context.CallEntityAsync<int>(
            new EntityInstanceId("Counter", "global"),
            "Increment");
        
        return count.ToString();
    }
}
```

### 6. No Dynamic Code Execution

Avoid patterns that can change behavior between runs.

**❌ Violations:**
```csharp
public override async Task<string> RunAsync(
    TaskOrchestrationContext context, 
    string input)
{
    // ❌ Reflection-based dynamic invocation
    var methodName = GetMethodNameFromDatabase();
    var method = this.GetType().GetMethod(methodName);
    var result = method.Invoke(this, new object[] { input });
    
    // ❌ Configuration that changes
    var config = ConfigurationManager.AppSettings["WorkflowVersion"];
    if (config == "2")
    {
        await DoV2Stuff();
    }
    
    return result.ToString();
}
```

**✅ Correct Approach:**
```csharp
public override async Task<string> RunAsync(
    TaskOrchestrationContext context, 
    string input)
{
    // ✅ Use version from input (stored in history)
    var version = input.Version ?? 
        await context.CallActivityAsync<int>("GetCurrentVersion", null);
    
    // ✅ Deterministic branching based on stored version
    if (version >= 2)
    {
        await DoV2Stuff(context);
    }
    else
    {
        await DoV1Stuff(context);
    }
    
    return "done";
}
```

## Safe Patterns

### Deterministic Workflow Logic

```csharp
public override async Task<ProcessingResult> RunAsync(
    TaskOrchestrationContext context, 
    ProcessingInput input)
{
    var logger = context.CreateReplaySafeLogger("MyOrchestration");
    
    // ✅ Deterministic time
    var startTime = context.CurrentUtcDateTime;
    logger.LogInformation("Started at {Time}", startTime);
    
    // ✅ Activities for external operations
    var data = await context.CallActivityAsync<DataResult>("FetchData", input);
    
    // ✅ Pure computation in orchestration (deterministic)
    var processedItems = data.Items
        .Where(i => i.Value > input.Threshold)
        .OrderBy(i => i.Name)
        .ToList();
    
    // ✅ Parallel activities with fan-out
    var tasks = processedItems.Select(item =>
        context.CallActivityAsync<ItemResult>("ProcessItem", item));
    var results = await Task.WhenAll(tasks);
    
    // ✅ Durable timer
    await context.CreateTimer(
        context.CurrentUtcDateTime.AddMinutes(5),
        CancellationToken.None);
    
    // ✅ External event
    var approval = await context.WaitForExternalEvent<bool>("Approval");
    
    return new ProcessingResult
    {
        ProcessedCount = results.Length,
        ApprovalReceived = approval
    };
}
```

### Idempotent Activities

```csharp
[DurableTask(nameof(ProcessPaymentActivity))]
public class ProcessPaymentActivity : TaskActivity<PaymentInput, PaymentResult>
{
    private readonly IPaymentService _paymentService;
    
    public ProcessPaymentActivity(IPaymentService paymentService)
    {
        _paymentService = paymentService;
    }
    
    public override async Task<PaymentResult> RunAsync(
        TaskActivityContext context, 
        PaymentInput input)
    {
        // ✅ Use idempotency key based on orchestration instance
        var idempotencyKey = $"{context.InstanceId}-payment-{input.PaymentId}";
        
        // Check if already processed
        var existing = await _paymentService.GetByIdempotencyKeyAsync(idempotencyKey);
        if (existing != null)
        {
            return existing;
        }
        
        // Process and store with idempotency key
        return await _paymentService.ProcessAsync(input, idempotencyKey);
    }
}
```

## Detection and Prevention

### Code Analyzers

Enable built-in analyzers to catch violations:

```xml
<!-- In .csproj -->
<PropertyGroup>
  <TreatWarningsAsErrors>true</TreatWarningsAsErrors>
</PropertyGroup>

<ItemGroup>
  <PackageReference Include="Microsoft.DurableTask.Generators" Version="1.0.0" />
</ItemGroup>
```

### Testing for Determinism

```csharp
[Fact]
public async Task Orchestration_IsDeterministic()
{
    var input = new TestInput { Value = "test" };
    
    // Run twice
    var result1 = await RunOrchestrationWithMocks(input);
    var result2 = await RunOrchestrationWithMocks(input);
    
    // Should produce identical sequences
    Assert.Equal(result1.ActivityCalls, result2.ActivityCalls);
    Assert.Equal(result1.Output, result2.Output);
}
```

### Replay-Safe Logging

```csharp
public override async Task<string> RunAsync(
    TaskOrchestrationContext context, 
    string input)
{
    // ✅ Use replay-safe logger
    var logger = context.CreateReplaySafeLogger("MyOrchestration");
    
    // Won't duplicate during replays
    logger.LogInformation("Processing started");
    
    // ❌ Don't inject ILogger directly - will log on every replay
    // _logger.LogInformation("This logs multiple times!");
    
    return "result";
}
```

## Summary Table

| Operation | Allowed | Alternative |
|-----------|---------|-------------|
| `DateTime.Now` | ❌ | `context.CurrentUtcDateTime` |
| `Guid.NewGuid()` | ❌ | `context.NewGuid()` |
| `Random.Next()` | ❌ | Activity |
| `Thread.Sleep()` | ❌ | `context.CreateTimer()` |
| File I/O | ❌ | Activity |
| HTTP calls | ❌ | Activity |
| Database access | ❌ | Activity |
| `.Wait()` / `.Result` | ❌ | `await` |
| `Task.Run()` | ❌ | Activity |
| Static fields | ❌ | Entity or Activity |
| Environment variables | ❌ | Activity |
| `ILogger` (injected) | ❌ | `context.CreateReplaySafeLogger()` |

## Next Steps

- [Analyzers](Analyzers.md) - Static analysis for constraints
- [Writing Task Orchestrations](Writing-Task-Orchestrations.md) - Orchestration patterns
- [Writing Task Activities](Writing-Task-Activities.md) - Activity patterns

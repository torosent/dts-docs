# ASP.NET Core Integration

The Durable Task .NET SDK can be hosted in ASP.NET Core applications with **Durable Task Scheduler**, enabling you to build web applications and APIs with durable workflow capabilities.

## Overview

ASP.NET Core integration provides:
- Self-hosted durable task workers
- Web API endpoints for workflow management
- Dependency injection integration
- Background service hosting

## Getting Started

### Required Packages

```xml
<PackageReference Include="Microsoft.DurableTask.Worker.AzureManaged" Version="1.3.0" />
<PackageReference Include="Microsoft.DurableTask.Client.AzureManaged" Version="1.3.0" />
<PackageReference Include="Microsoft.DurableTask.Generators" Version="1.0.0" PrivateAssets="all" />
```

### Basic Setup with Durable Task Scheduler

```csharp
var builder = WebApplication.CreateBuilder(args);

// Connection string for Durable Task Scheduler
// Local emulator: "Endpoint=http://localhost:8080;TaskHub=default;Authentication=None"
// Azure: "Endpoint=https://<scheduler>.westus2.durabletask.io;TaskHub=<hub>;Authentication=DefaultAzure"
string connectionString = builder.Configuration["DurableTaskScheduler:ConnectionString"]
    ?? "Endpoint=http://localhost:8080;TaskHub=default;Authentication=None";

// Add Durable Task services
builder.Services.AddDurableTaskClient(clientBuilder =>
{
    clientBuilder.UseDurableTaskScheduler(connectionString);
});

builder.Services.AddDurableTaskWorker(workerBuilder =>
{
    workerBuilder.UseDurableTaskScheduler(connectionString);
    
    // Use source generators for automatic registration
    workerBuilder.AddAllGeneratedTasks();
});

// Add controllers
builder.Services.AddControllers();

var app = builder.Build();

app.MapControllers();

app.Run();
```

## Service Registration

### Adding Orchestrations and Activities

```csharp
builder.Services.AddDurableTaskWorker(workerBuilder =>
{
    workerBuilder.UseDurableTaskScheduler(connectionString);
    
    // Option 1: Add individual tasks
    workerBuilder.AddTasks(tasks =>
    {
        tasks.AddOrchestrator<OrderOrchestration>();
        tasks.AddActivity<PaymentActivity>();
        tasks.AddActivity<ShippingActivity>();
    });
    
    // Option 2: Use source generators (recommended)
    workerBuilder.AddAllGeneratedTasks();
});
```

### With Dependency Injection

```csharp
// Register services
builder.Services.AddScoped<IPaymentService, PaymentService>();
builder.Services.AddScoped<IShippingService, ShippingService>();
builder.Services.AddSingleton<IEmailService, EmailService>();

// Activities can use DI
[DurableTask(nameof(PaymentActivity))]
public class PaymentActivity : TaskActivity<PaymentInput, PaymentResult>
{
    private readonly IPaymentService _paymentService;
    private readonly ILogger<PaymentActivity> _logger;
    
    public PaymentActivity(
        IPaymentService paymentService, 
        ILogger<PaymentActivity> logger)
    {
        _paymentService = paymentService;
        _logger = logger;
    }
    
    public override async Task<PaymentResult> RunAsync(
        TaskActivityContext context, 
        PaymentInput input)
    {
        _logger.LogInformation("Processing payment for {Amount}", input.Amount);
        return await _paymentService.ProcessAsync(input);
    }
}
```

## Controller Integration

### Orchestration Management Controller

```csharp
[ApiController]
[Route("api/[controller]")]
public class OrchestrationController : ControllerBase
{
    private readonly DurableTaskClient _client;
    private readonly ILogger<OrchestrationController> _logger;

    public OrchestrationController(
        DurableTaskClient client,
        ILogger<OrchestrationController> logger)
    {
        _client = client;
        _logger = logger;
    }

    [HttpPost("start")]
    public async Task<ActionResult<StartResponse>> Start(
        [FromBody] StartOrchestrationRequest request)
    {
        string instanceId = await _client.ScheduleNewOrchestrationInstanceAsync(
            request.OrchestrationName,
            request.Input,
            new StartOrchestrationOptions
            {
                InstanceId = request.InstanceId
            });

        _logger.LogInformation(
            "Started orchestration {Name} with instance {InstanceId}",
            request.OrchestrationName,
            instanceId);

        return Accepted(new StartResponse { InstanceId = instanceId });
    }

    [HttpGet("{instanceId}")]
    public async Task<ActionResult<StatusResponse>> GetStatus(string instanceId)
    {
        var metadata = await _client.GetInstanceAsync(
            instanceId, 
            getInputsAndOutputs: true);

        if (metadata == null)
        {
            return NotFound();
        }

        return Ok(new StatusResponse
        {
            InstanceId = metadata.InstanceId,
            Name = metadata.Name,
            RuntimeStatus = metadata.RuntimeStatus.ToString(),
            CreatedAt = metadata.CreatedAt,
            LastUpdatedAt = metadata.LastUpdatedAt,
            Input = metadata.SerializedInput,
            Output = metadata.SerializedOutput
        });
    }

    [HttpPost("{instanceId}/terminate")]
    public async Task<IActionResult> Terminate(
        string instanceId, 
        [FromBody] TerminateRequest request)
    {
        await _client.TerminateInstanceAsync(instanceId, request.Reason);
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

    [HttpPost("{instanceId}/suspend")]
    public async Task<IActionResult> Suspend(
        string instanceId,
        [FromBody] SuspendRequest request)
    {
        await _client.SuspendInstanceAsync(instanceId, request.Reason);
        return Ok();
    }

    [HttpPost("{instanceId}/resume")]
    public async Task<IActionResult> Resume(
        string instanceId,
        [FromBody] ResumeRequest request)
    {
        await _client.ResumeInstanceAsync(instanceId, request.Reason);
        return Ok();
    }

    [HttpGet]
    public async Task<ActionResult<QueryResponse>> Query(
        [FromQuery] OrchestrationRuntimeStatus? status,
        [FromQuery] string? prefix,
        [FromQuery] DateTime? from,
        [FromQuery] DateTime? to,
        [FromQuery] int pageSize = 50,
        [FromQuery] string? continuationToken = null)
    {
        var query = new OrchestrationQuery
        {
            PageSize = Math.Min(pageSize, 100),
            ContinuationToken = continuationToken
        };

        if (status.HasValue)
        {
            query.RuntimeStatus = new[] { status.Value };
        }

        if (!string.IsNullOrEmpty(prefix))
        {
            query.InstanceIdPrefix = prefix;
        }

        if (from.HasValue)
        {
            query.CreatedFrom = from.Value;
        }

        if (to.HasValue)
        {
            query.CreatedTo = to.Value;
        }

        var result = await _client.GetAllInstancesAsync(query);

        return Ok(new QueryResponse
        {
            Instances = result.Instances.Select(i => new InstanceSummary
            {
                InstanceId = i.InstanceId,
                Name = i.Name,
                Status = i.RuntimeStatus.ToString(),
                CreatedAt = i.CreatedAt,
                LastUpdatedAt = i.LastUpdatedAt
            }).ToList(),
            ContinuationToken = result.ContinuationToken
        });
    }
}
```

### Domain-Specific Controller

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
    public async Task<ActionResult<OrderResponse>> CreateOrder(
        [FromBody] CreateOrderRequest request)
    {
        string instanceId = $"order-{Guid.NewGuid():N}";

        await _client.ScheduleNewOrchestrationInstanceAsync(
            "OrderProcessingOrchestration",
            request,
            new StartOrchestrationOptions { InstanceId = instanceId });

        return Accepted(new OrderResponse
        {
            OrderId = instanceId,
            Status = "Processing",
            StatusUrl = Url.Action(nameof(GetOrderStatus), new { orderId = instanceId })
        });
    }

    [HttpGet("{orderId}")]
    public async Task<ActionResult<OrderStatusResponse>> GetOrderStatus(string orderId)
    {
        var metadata = await _client.GetInstanceAsync(orderId, getInputsAndOutputs: true);

        if (metadata == null)
        {
            return NotFound();
        }

        var response = new OrderStatusResponse
        {
            OrderId = orderId,
            Status = metadata.RuntimeStatus.ToString(),
            CreatedAt = metadata.CreatedAt
        };

        if (metadata.RuntimeStatus == OrchestrationRuntimeStatus.Completed)
        {
            response.Result = metadata.ReadOutputAs<OrderResult>();
        }

        return Ok(response);
    }

    [HttpPost("{orderId}/cancel")]
    public async Task<IActionResult> CancelOrder(string orderId)
    {
        await _client.RaiseEventAsync(orderId, "CancelRequested", null);
        return Ok();
    }
}
```

## Background Worker Hosting

### Hosted Service Setup

```csharp
builder.Services.AddHostedService<DurableTaskBackgroundService>();

public class DurableTaskBackgroundService : BackgroundService
{
    private readonly IServiceProvider _serviceProvider;
    private readonly ILogger<DurableTaskBackgroundService> _logger;

    public DurableTaskBackgroundService(
        IServiceProvider serviceProvider,
        ILogger<DurableTaskBackgroundService> logger)
    {
        _serviceProvider = serviceProvider;
        _logger = logger;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        _logger.LogInformation("Durable Task worker starting...");

        using var scope = _serviceProvider.CreateScope();
        var worker = scope.ServiceProvider.GetRequiredService<DurableTaskWorker>();

        await worker.StartAsync(stoppingToken);

        _logger.LogInformation("Durable Task worker started");

        // Wait until cancellation
        await Task.Delay(Timeout.Infinite, stoppingToken);
    }
}
```

## Minimal API Integration

### Using Minimal APIs

```csharp
var builder = WebApplication.CreateBuilder(args);

string connectionString = builder.Configuration["DURABLE_TASK_SCHEDULER_CONNECTION_STRING"]
    ?? "Endpoint=http://localhost:8080;TaskHub=default;Authentication=None";

builder.Services.AddDurableTaskClient(b => b.UseDurableTaskScheduler(connectionString));
builder.Services.AddDurableTaskWorker(b =>
{
    b.UseDurableTaskScheduler(connectionString);
    b.AddAllGeneratedTasks();
});

var app = builder.Build();

// Start orchestration
app.MapPost("/api/workflows", async (
    DurableTaskClient client,
    StartWorkflowRequest request) =>
{
    string instanceId = await client.ScheduleNewOrchestrationInstanceAsync(
        request.WorkflowName,
        request.Input);
    
    return Results.Accepted($"/api/workflows/{instanceId}", new { InstanceId = instanceId });
});

// Get status
app.MapGet("/api/workflows/{instanceId}", async (
    DurableTaskClient client,
    string instanceId) =>
{
    var metadata = await client.GetInstanceAsync(instanceId, getInputsAndOutputs: true);
    
    if (metadata == null)
    {
        return Results.NotFound();
    }
    
    return Results.Ok(new
    {
        metadata.InstanceId,
        metadata.Name,
        Status = metadata.RuntimeStatus.ToString(),
        metadata.CreatedAt,
        Output = metadata.SerializedOutput
    });
});

// Raise event
app.MapPost("/api/workflows/{instanceId}/events/{eventName}", async (
    DurableTaskClient client,
    string instanceId,
    string eventName,
    object eventData) =>
{
    await client.RaiseEventAsync(instanceId, eventName, eventData);
    return Results.Ok();
});

// Terminate
app.MapDelete("/api/workflows/{instanceId}", async (
    DurableTaskClient client,
    string instanceId,
    string? reason) =>
{
    await client.TerminateInstanceAsync(instanceId, reason ?? "Terminated via API");
    return Results.Ok();
});

app.Run();
```

## Configuration

### App Settings

```json
{
  "DurableTask": {
    "GrpcEndpoint": "http://localhost:5001",
    "TaskHubName": "MyTaskHub"
  },
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.DurableTask": "Debug"
    }
  }
}
```

### Configuration Binding

```csharp
builder.Services.Configure<DurableTaskOptions>(
    builder.Configuration.GetSection("DurableTask"));

builder.Services.AddDurableTaskClient((services, clientBuilder) =>
{
    var options = services.GetRequiredService<IOptions<DurableTaskOptions>>().Value;
    clientBuilder.UseDurableTaskScheduler(options.ConnectionString);
});
```

## Health Checks

### Health Check Implementation

```csharp
builder.Services.AddHealthChecks()
    .AddCheck<DurableTaskHealthCheck>("durable-task");

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
            // Simple connectivity check
            var query = new OrchestrationQuery { PageSize = 1 };
            await _client.GetAllInstancesAsync(query);

            return HealthCheckResult.Healthy("Durable Task service is available");
        }
        catch (Exception ex)
        {
            return HealthCheckResult.Unhealthy(
                "Durable Task service is unavailable",
                ex);
        }
    }
}

// Map health endpoint
app.MapHealthChecks("/health");
```

## Error Handling

### Global Exception Handler

```csharp
builder.Services.AddExceptionHandler<GlobalExceptionHandler>();

public class GlobalExceptionHandler : IExceptionHandler
{
    private readonly ILogger<GlobalExceptionHandler> _logger;

    public GlobalExceptionHandler(ILogger<GlobalExceptionHandler> logger)
    {
        _logger = logger;
    }

    public async ValueTask<bool> TryHandleAsync(
        HttpContext context,
        Exception exception,
        CancellationToken cancellationToken)
    {
        if (exception is InvalidOperationException ex && 
            ex.Message.Contains("already exists"))
        {
            _logger.LogWarning("Attempted to start duplicate orchestration");
            
            context.Response.StatusCode = StatusCodes.Status409Conflict;
            await context.Response.WriteAsJsonAsync(new
            {
                Error = "Orchestration instance already exists"
            }, cancellationToken);
            
            return true;
        }

        return false;
    }
}
```

## OpenAPI/Swagger Integration

### Configure Swagger

```csharp
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen(c =>
{
    c.SwaggerDoc("v1", new OpenApiInfo
    {
        Title = "Durable Task API",
        Version = "v1",
        Description = "API for managing durable orchestrations"
    });
});

var app = builder.Build();

if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}
```

## Authentication and Authorization

### Secure Endpoints

```csharp
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.Authority = builder.Configuration["Auth:Authority"];
        options.Audience = builder.Configuration["Auth:Audience"];
    });

builder.Services.AddAuthorization(options =>
{
    options.AddPolicy("WorkflowAdmin", policy =>
        policy.RequireRole("admin", "workflow-manager"));
});

// Apply to controller
[Authorize(Policy = "WorkflowAdmin")]
[ApiController]
[Route("api/orchestrations")]
public class OrchestrationController : ControllerBase
{
    // ...
}
```

## Best Practices

### 1. Use Scoped Services Carefully

```csharp
// Activities should use scoped services carefully
[DurableTask(nameof(ScopedActivity))]
public class ScopedActivity : TaskActivity<string, string>
{
    private readonly IServiceProvider _serviceProvider;

    public ScopedActivity(IServiceProvider serviceProvider)
    {
        _serviceProvider = serviceProvider;
    }

    public override async Task<string> RunAsync(
        TaskActivityContext context, 
        string input)
    {
        // Create scope for database context
        using var scope = _serviceProvider.CreateScope();
        var dbContext = scope.ServiceProvider.GetRequiredService<MyDbContext>();
        
        // Use dbContext...
        return "result";
    }
}
```

### 2. Handle Graceful Shutdown

```csharp
app.Lifetime.ApplicationStopping.Register(() =>
{
    // Give orchestrations time to complete current work
    Task.Delay(TimeSpan.FromSeconds(30)).Wait();
});
```

### 3. Use Response Compression

```csharp
builder.Services.AddResponseCompression(options =>
{
    options.EnableForHttps = true;
});

app.UseResponseCompression();
```

## Next Steps

- [Azure Functions Integration](Azure-Functions-Integration.md) - Serverless hosting
- [Source Generators](Source-Generators.md) - Reduce boilerplate
- [Diagnostics and Logging](Diagnostics-and-Logging.md) - Monitoring

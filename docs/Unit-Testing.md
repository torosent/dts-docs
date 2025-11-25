---
layout: default
title: Unit Testing
parent: Testing & Diagnostics
nav_order: 1
description: "Write comprehensive tests for orchestrations and activities"
---

# Unit Testing

The Durable Task .NET SDK provides testing utilities to help you write comprehensive tests for your orchestrations, activities, and entities.

## Overview

Testing durable functions requires different approaches than typical unit testing because:

1. **Orchestrations** replay from history and must be deterministic
2. **Activities** are the actual work units and can be tested traditionally
3. **Entities** maintain state and respond to operations

## Setting Up Test Projects

### Required Packages

```xml
<PackageReference Include="Microsoft.DurableTask.Testing" Version="1.0.0" />
<PackageReference Include="Microsoft.NET.Test.Sdk" Version="17.8.0" />
<PackageReference Include="xunit" Version="2.6.1" />
<PackageReference Include="xunit.runner.visualstudio" Version="2.5.3" />
<PackageReference Include="Moq" Version="4.20.69" />
```

## Testing Activities

Activities are the simplest to test since they're regular async methods.

### Basic Activity Test

```csharp
public class ActivityTests
{
    [Fact]
    public async Task ProcessPayment_ValidInput_ReturnsSuccess()
    {
        // Arrange
        var activity = new PaymentActivity();
        var input = new PaymentRequest
        {
            Amount = 100.00m,
            CardNumber = "4111111111111111"
        };
        
        // Create mock context
        var context = new Mock<TaskActivityContext>();
        context.Setup(c => c.InstanceId).Returns("test-instance");
        
        // Act
        var result = await activity.RunAsync(context.Object, input);
        
        // Assert
        Assert.True(result.Success);
        Assert.NotEmpty(result.TransactionId);
    }
    
    [Fact]
    public async Task ProcessPayment_InvalidAmount_ThrowsException()
    {
        // Arrange
        var activity = new PaymentActivity();
        var input = new PaymentRequest { Amount = -100.00m };
        var context = new Mock<TaskActivityContext>();
        
        // Act & Assert
        await Assert.ThrowsAsync<ArgumentException>(
            () => activity.RunAsync(context.Object, input));
    }
}
```

### Activity with Dependencies

```csharp
public class ActivityWithDependenciesTests
{
    private readonly Mock<IPaymentService> _paymentServiceMock;
    private readonly Mock<ILogger<PaymentActivity>> _loggerMock;
    private readonly PaymentActivity _activity;
    
    public ActivityWithDependenciesTests()
    {
        _paymentServiceMock = new Mock<IPaymentService>();
        _loggerMock = new Mock<ILogger<PaymentActivity>>();
        _activity = new PaymentActivity(_paymentServiceMock.Object, _loggerMock.Object);
    }
    
    [Fact]
    public async Task ProcessPayment_CallsPaymentService()
    {
        // Arrange
        var request = new PaymentRequest { Amount = 50.00m };
        var expectedResult = new PaymentResult 
        { 
            Success = true, 
            TransactionId = "txn-123" 
        };
        
        _paymentServiceMock
            .Setup(s => s.ProcessAsync(It.IsAny<PaymentRequest>()))
            .ReturnsAsync(expectedResult);
        
        var context = new Mock<TaskActivityContext>();
        
        // Act
        var result = await _activity.RunAsync(context.Object, request);
        
        // Assert
        Assert.Equal("txn-123", result.TransactionId);
        _paymentServiceMock.Verify(
            s => s.ProcessAsync(It.Is<PaymentRequest>(r => r.Amount == 50.00m)), 
            Times.Once);
    }
}
```

## Testing Orchestrations

Orchestrations require mocking the `TaskOrchestrationContext` to simulate activity calls.

### Using DurableTaskTestingExtensions

```csharp
using Microsoft.DurableTask.Testing;

public class OrchestrationTests
{
    [Fact]
    public async Task OrderOrchestration_HappyPath_Succeeds()
    {
        // Arrange
        var input = new OrderInput { OrderId = "order-123", Amount = 99.99m };
        
        // Create a mock context using testing extensions
        var context = new MockTaskOrchestrationContext();
        
        // Setup activity mocks
        context.SetupActivityResult<ValidationResult>(
            "ValidateOrder",
            new ValidationResult { IsValid = true });
        
        context.SetupActivityResult<PaymentResult>(
            "ProcessPayment",
            new PaymentResult { Success = true, TransactionId = "txn-123" });
        
        context.SetupActivityResult<ShippingResult>(
            "CreateShipment",
            new ShippingResult { TrackingNumber = "TRACK123" });
        
        var orchestration = new OrderOrchestration();
        
        // Act
        var result = await orchestration.RunAsync(context, input);
        
        // Assert
        Assert.Equal("Completed", result.Status);
        Assert.Equal("txn-123", result.PaymentTransactionId);
        Assert.Equal("TRACK123", result.TrackingNumber);
    }
}
```

### Manual Mock Setup

```csharp
public class ManualOrchestrationTests
{
    [Fact]
    public async Task OrderOrchestration_PaymentFails_ReturnsFailure()
    {
        // Arrange
        var input = new OrderInput { OrderId = "order-123", Amount = 99.99m };
        
        var contextMock = new Mock<TaskOrchestrationContext>();
        
        // Setup instance ID
        contextMock.Setup(c => c.InstanceId).Returns("test-instance");
        contextMock.Setup(c => c.CurrentUtcDateTime).Returns(DateTime.UtcNow);
        
        // Setup validation to succeed
        contextMock
            .Setup(c => c.CallActivityAsync<ValidationResult>(
                "ValidateOrder",
                input,
                It.IsAny<TaskOptions>()))
            .ReturnsAsync(new ValidationResult { IsValid = true });
        
        // Setup payment to fail
        contextMock
            .Setup(c => c.CallActivityAsync<PaymentResult>(
                "ProcessPayment",
                It.IsAny<object>(),
                It.IsAny<TaskOptions>()))
            .ReturnsAsync(new PaymentResult { Success = false, Error = "Declined" });
        
        var orchestration = new OrderOrchestration();
        
        // Act
        var result = await orchestration.RunAsync(contextMock.Object, input);
        
        // Assert
        Assert.Equal("PaymentFailed", result.Status);
        Assert.Equal("Declined", result.Error);
    }
}
```

### Testing External Events

```csharp
[Fact]
public async Task ApprovalOrchestration_EventReceived_Proceeds()
{
    // Arrange
    var input = new ApprovalRequest { RequestId = "req-123" };
    
    var contextMock = new Mock<TaskOrchestrationContext>();
    contextMock.Setup(c => c.InstanceId).Returns("test-instance");
    
    // Setup send notification
    contextMock
        .Setup(c => c.CallActivityAsync(
            "SendApprovalRequest",
            It.IsAny<object>(),
            It.IsAny<TaskOptions>()))
        .Returns(Task.CompletedTask);
    
    // Setup external event - returns immediately with approval
    contextMock
        .Setup(c => c.WaitForExternalEvent<bool>("ApprovalEvent"))
        .ReturnsAsync(true);
    
    // Setup final activity
    contextMock
        .Setup(c => c.CallActivityAsync<ProcessingResult>(
            "ProcessApprovedRequest",
            input,
            It.IsAny<TaskOptions>()))
        .ReturnsAsync(new ProcessingResult { Success = true });
    
    var orchestration = new ApprovalOrchestration();
    
    // Act
    var result = await orchestration.RunAsync(contextMock.Object, input);
    
    // Assert
    Assert.Equal("Approved", result.Status);
}
```

### Testing Timers

```csharp
[Fact]
public async Task TimerOrchestration_WaitsForTimer_ThenProceeds()
{
    // Arrange
    var baseTime = new DateTime(2024, 1, 1, 12, 0, 0, DateTimeKind.Utc);
    var input = new TimerInput { DelayMinutes = 30 };
    
    var contextMock = new Mock<TaskOrchestrationContext>();
    contextMock.Setup(c => c.CurrentUtcDateTime).Returns(baseTime);
    
    // Setup timer - completes immediately in test
    contextMock
        .Setup(c => c.CreateTimer(
            It.Is<DateTime>(d => d == baseTime.AddMinutes(30)),
            It.IsAny<CancellationToken>()))
        .Returns(Task.CompletedTask);
    
    // Setup activity after timer
    contextMock
        .Setup(c => c.CallActivityAsync<string>(
            "ScheduledTask",
            It.IsAny<object>(),
            It.IsAny<TaskOptions>()))
        .ReturnsAsync("Done");
    
    var orchestration = new TimerOrchestration();
    
    // Act
    var result = await orchestration.RunAsync(contextMock.Object, input);
    
    // Assert
    Assert.Equal("Done", result);
    
    // Verify timer was created with correct time
    contextMock.Verify(
        c => c.CreateTimer(baseTime.AddMinutes(30), It.IsAny<CancellationToken>()),
        Times.Once);
}
```

### Testing Sub-Orchestrations

```csharp
[Fact]
public async Task ParentOrchestration_CallsSubOrchestration()
{
    // Arrange
    var input = new ParentInput { Data = "test" };
    
    var contextMock = new Mock<TaskOrchestrationContext>();
    
    // Setup sub-orchestration call
    contextMock
        .Setup(c => c.CallSubOrchestratorAsync<ChildResult>(
            "ChildOrchestration",
            It.IsAny<ChildInput>(),
            It.IsAny<TaskOptions>()))
        .ReturnsAsync(new ChildResult { Processed = true });
    
    var orchestration = new ParentOrchestration();
    
    // Act
    var result = await orchestration.RunAsync(contextMock.Object, input);
    
    // Assert
    Assert.True(result.ChildCompleted);
}
```

## Testing Entities

### Basic Entity Test

```csharp
public class EntityTests
{
    [Fact]
    public async Task CounterEntity_Increment_IncreasesValue()
    {
        // Arrange
        var entity = new CounterEntity();
        var contextMock = new Mock<TaskEntityContext>();
        
        // Setup state
        int currentState = 5;
        contextMock.Setup(c => c.GetState<int>(It.IsAny<Func<int>>()))
            .Returns(currentState);
        
        // Capture state setter
        int? savedState = null;
        contextMock
            .Setup(c => c.SetState(It.IsAny<int>()))
            .Callback<int>(s => savedState = s);
        
        // Act
        entity.Increment(contextMock.Object);
        
        // Assert
        Assert.Equal(6, savedState);
    }
    
    [Fact]
    public async Task CounterEntity_Add_AddsSpecifiedAmount()
    {
        // Arrange
        var entity = new CounterEntity();
        var contextMock = new Mock<TaskEntityContext>();
        
        contextMock.Setup(c => c.GetState<int>(It.IsAny<Func<int>>()))
            .Returns(10);
        
        int? savedState = null;
        contextMock
            .Setup(c => c.SetState(It.IsAny<int>()))
            .Callback<int>(s => savedState = s);
        
        // Act
        entity.Add(contextMock.Object, 5);
        
        // Assert
        Assert.Equal(15, savedState);
    }
}
```

### Entity State Tests

```csharp
public class ShoppingCartEntityTests
{
    [Fact]
    public void AddItem_EmptyCart_AddsItem()
    {
        // Arrange
        var entity = new ShoppingCartEntity();
        var contextMock = new Mock<TaskEntityContext>();
        
        var emptyCart = new CartState { Items = new List<CartItem>() };
        contextMock.Setup(c => c.GetState(It.IsAny<Func<CartState>>()))
            .Returns(emptyCart);
        
        CartState? savedState = null;
        contextMock
            .Setup(c => c.SetState(It.IsAny<CartState>()))
            .Callback<CartState>(s => savedState = s);
        
        var newItem = new CartItem { ProductId = "prod-1", Quantity = 2 };
        
        // Act
        entity.AddItem(contextMock.Object, newItem);
        
        // Assert
        Assert.NotNull(savedState);
        Assert.Single(savedState!.Items);
        Assert.Equal("prod-1", savedState.Items[0].ProductId);
    }
    
    [Fact]
    public void RemoveItem_ExistingItem_RemovesItem()
    {
        // Arrange
        var entity = new ShoppingCartEntity();
        var contextMock = new Mock<TaskEntityContext>();
        
        var cart = new CartState
        {
            Items = new List<CartItem>
            {
                new() { ProductId = "prod-1", Quantity = 2 },
                new() { ProductId = "prod-2", Quantity = 1 }
            }
        };
        
        contextMock.Setup(c => c.GetState(It.IsAny<Func<CartState>>()))
            .Returns(cart);
        
        CartState? savedState = null;
        contextMock
            .Setup(c => c.SetState(It.IsAny<CartState>()))
            .Callback<CartState>(s => savedState = s);
        
        // Act
        entity.RemoveItem(contextMock.Object, "prod-1");
        
        // Assert
        Assert.NotNull(savedState);
        Assert.Single(savedState!.Items);
        Assert.Equal("prod-2", savedState.Items[0].ProductId);
    }
}
```

## Integration Testing

### In-Memory Worker Tests

```csharp
public class IntegrationTests : IAsyncLifetime
{
    private DurableTaskGrpcWorker? _worker;
    private DurableTaskClient? _client;
    
    public async Task InitializeAsync()
    {
        // Setup in-memory backend for testing
        var services = new ServiceCollection();
        
        services.AddDurableTaskWorker(builder =>
        {
            builder.UseInMemory();
            builder.AddTasks<OrderOrchestration>();
            builder.AddTasks<PaymentActivity>();
            builder.AddTasks<ShippingActivity>();
        });
        
        services.AddDurableTaskClient(builder =>
        {
            builder.UseInMemory();
        });
        
        var provider = services.BuildServiceProvider();
        
        _worker = provider.GetRequiredService<DurableTaskGrpcWorker>();
        _client = provider.GetRequiredService<DurableTaskClient>();
        
        await _worker.StartAsync(CancellationToken.None);
    }
    
    public async Task DisposeAsync()
    {
        if (_worker != null)
        {
            await _worker.StopAsync(CancellationToken.None);
        }
    }
    
    [Fact]
    public async Task OrderOrchestration_EndToEnd_Succeeds()
    {
        // Arrange
        var input = new OrderInput
        {
            OrderId = "order-123",
            Amount = 99.99m
        };
        
        // Act
        string instanceId = await _client!.ScheduleNewOrchestrationInstanceAsync(
            "OrderOrchestration",
            input);
        
        var result = await _client.WaitForInstanceCompletionAsync(
            instanceId,
            getInputsAndOutputs: true,
            new CancellationTokenSource(TimeSpan.FromSeconds(30)).Token);
        
        // Assert
        Assert.NotNull(result);
        Assert.Equal(OrchestrationRuntimeStatus.Completed, result.RuntimeStatus);
        
        var output = result.ReadOutputAs<OrderResult>();
        Assert.Equal("Completed", output.Status);
    }
}
```

## Testing Best Practices

### 1. Test Determinism

```csharp
[Fact]
public async Task Orchestration_IsDeterministic_ProducesSameResult()
{
    // Run orchestration twice with same input
    var input = new TestInput { Value = "test" };
    
    var result1 = await RunOrchestration(input);
    var result2 = await RunOrchestration(input);
    
    // Should produce identical results
    Assert.Equal(result1.Output, result2.Output);
}
```

### 2. Test Edge Cases

```csharp
[Theory]
[InlineData(0)]
[InlineData(-1)]
[InlineData(int.MaxValue)]
public async Task Orchestration_HandlesEdgeCases(int value)
{
    var input = new NumberInput { Value = value };
    var context = CreateMockContext();
    
    var orchestration = new NumberOrchestration();
    var result = await orchestration.RunAsync(context, input);
    
    // Verify proper handling
    Assert.NotNull(result);
}
```

### 3. Test Error Scenarios

```csharp
[Fact]
public async Task Orchestration_ActivityFails_HandlesGracefully()
{
    var contextMock = new Mock<TaskOrchestrationContext>();
    
    // Setup activity to throw
    contextMock
        .Setup(c => c.CallActivityAsync<PaymentResult>(
            "ProcessPayment",
            It.IsAny<object>(),
            It.IsAny<TaskOptions>()))
        .ThrowsAsync(new TaskFailedException("Payment service unavailable"));
    
    var orchestration = new OrderOrchestration();
    
    var result = await orchestration.RunAsync(contextMock.Object, new OrderInput());
    
    // Verify error handling
    Assert.Equal("Failed", result.Status);
    Assert.Contains("unavailable", result.Error);
}
```

### 4. Use Test Fixtures

```csharp
public class OrchestrationTestFixture : IDisposable
{
    public Mock<TaskOrchestrationContext> Context { get; }
    public ILogger<OrderOrchestration> Logger { get; }
    
    public OrchestrationTestFixture()
    {
        Context = new Mock<TaskOrchestrationContext>();
        Context.Setup(c => c.InstanceId).Returns("test-instance");
        Context.Setup(c => c.CurrentUtcDateTime).Returns(DateTime.UtcNow);
        
        Logger = new Mock<ILogger<OrderOrchestration>>().Object;
    }
    
    public void SetupSuccessfulActivities()
    {
        Context
            .Setup(c => c.CallActivityAsync<ValidationResult>(
                "ValidateOrder",
                It.IsAny<object>(),
                It.IsAny<TaskOptions>()))
            .ReturnsAsync(new ValidationResult { IsValid = true });
        
        // ... more setups
    }
    
    public void Dispose()
    {
        // Cleanup if needed
    }
}
```

## Next Steps

- [Diagnostics and Logging](Diagnostics-and-Logging.md) - Tracing and debugging
- [Analyzers](Analyzers.md) - Static analysis
- [Error Handling and Compensation](Error-Handling-and-Compensation.md) - Error patterns

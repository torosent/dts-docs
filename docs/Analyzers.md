---
layout: default
title: Analyzers
parent: Testing & Diagnostics
nav_order: 3
description: "Roslyn analyzers for detecting common mistakes"
---

# Analyzers

The Durable Task .NET SDK includes Roslyn analyzers that help detect common mistakes and enforce best practices at compile time. These analyzers catch issues that would otherwise cause runtime failures.

## Overview

Analyzers check your code for:
- Determinism violations in orchestrations
- Improper use of context methods
- Serialization issues
- Best practice violations

## Installation

Analyzers are included in the SDK packages:

```xml
<PackageReference Include="Microsoft.DurableTask.Generators" Version="1.0.0" />
```

## Analyzer Rules

### DURABLE001: Non-Deterministic API Usage

Orchestrations must be deterministic. This analyzer detects usage of non-deterministic APIs.

**Severity:** Error

**Example (Violation):**
```csharp
public override async Task<string> RunAsync(
    TaskOrchestrationContext context, 
    string input)
{
    // ❌ DURABLE001: DateTime.Now is non-deterministic
    var now = DateTime.Now;
    
    // ❌ DURABLE001: Guid.NewGuid() is non-deterministic
    var id = Guid.NewGuid();
    
    // ❌ DURABLE001: Random is non-deterministic
    var random = new Random().Next();
    
    return $"{now}_{id}_{random}";
}
```

**Fix:**
```csharp
public override async Task<string> RunAsync(
    TaskOrchestrationContext context, 
    string input)
{
    // ✅ Use context.CurrentUtcDateTime
    var now = context.CurrentUtcDateTime;
    
    // ✅ Use context.NewGuid()
    var id = context.NewGuid();
    
    // ✅ Generate random in activity or use deterministic seed
    var random = await context.CallActivityAsync<int>("GenerateRandom", null);
    
    return $"{now}_{id}_{random}";
}
```

### DURABLE002: Direct I/O Operations

Orchestrations should not perform direct I/O operations.

**Severity:** Warning

**Example (Violation):**
```csharp
public override async Task<string> RunAsync(
    TaskOrchestrationContext context, 
    string input)
{
    // ❌ DURABLE002: Direct file I/O
    var content = await File.ReadAllTextAsync("config.json");
    
    // ❌ DURABLE002: Direct HTTP call
    using var client = new HttpClient();
    var response = await client.GetAsync("https://api.example.com");
    
    return content;
}
```

**Fix:**
```csharp
public override async Task<string> RunAsync(
    TaskOrchestrationContext context, 
    string input)
{
    // ✅ Use activities for I/O
    var content = await context.CallActivityAsync<string>("ReadConfigFile", "config.json");
    var apiResult = await context.CallActivityAsync<string>("CallApi", "https://api.example.com");
    
    return content;
}
```

### DURABLE003: Thread-Blocking Operations

Orchestrations should not use blocking operations.

**Severity:** Error

**Example (Violation):**
```csharp
public override async Task<string> RunAsync(
    TaskOrchestrationContext context, 
    string input)
{
    // ❌ DURABLE003: Thread.Sleep blocks the thread
    Thread.Sleep(1000);
    
    // ❌ DURABLE003: Task.Wait() is blocking
    var task = SomeAsyncMethod();
    task.Wait();
    
    // ❌ DURABLE003: .Result blocks
    var result = task.Result;
    
    return result;
}
```

**Fix:**
```csharp
public override async Task<string> RunAsync(
    TaskOrchestrationContext context, 
    string input)
{
    // ✅ Use durable timer for delays
    await context.CreateTimer(
        context.CurrentUtcDateTime.AddSeconds(1), 
        CancellationToken.None);
    
    // ✅ Use await
    var result = await SomeAsyncMethod();
    
    return result;
}
```

### DURABLE004: Environment Variable Access

Direct environment variable access is non-deterministic.

**Severity:** Warning

**Example (Violation):**
```csharp
public override async Task<string> RunAsync(
    TaskOrchestrationContext context, 
    string input)
{
    // ❌ DURABLE004: Environment variables can change between replays
    var connectionString = Environment.GetEnvironmentVariable("DB_CONNECTION");
    
    return connectionString ?? "default";
}
```

**Fix:**
```csharp
public override async Task<string> RunAsync(
    TaskOrchestrationContext context, 
    string input)
{
    // ✅ Read config in activity (cached in history)
    var connectionString = await context.CallActivityAsync<string>(
        "GetConfiguration", 
        "DB_CONNECTION");
    
    return connectionString ?? "default";
}
```

### DURABLE005: Incorrect TaskOptions Usage

Ensures TaskOptions are used correctly.

**Severity:** Warning

**Example (Violation):**
```csharp
public override async Task<string> RunAsync(
    TaskOrchestrationContext context, 
    string input)
{
    // ❌ DURABLE005: Invalid retry configuration
    var options = new TaskOptions
    {
        Retry = new RetryPolicy(
            maxNumberOfAttempts: 0,  // Must be >= 1
            firstRetryInterval: TimeSpan.Zero)  // Must be > 0
    };
    
    return await context.CallActivityAsync<string>("MyActivity", input, options);
}
```

**Fix:**
```csharp
public override async Task<string> RunAsync(
    TaskOrchestrationContext context, 
    string input)
{
    // ✅ Valid retry configuration
    var options = new TaskOptions
    {
        Retry = new RetryPolicy(
            maxNumberOfAttempts: 3,
            firstRetryInterval: TimeSpan.FromSeconds(1))
    };
    
    return await context.CallActivityAsync<string>("MyActivity", input, options);
}
```

### DURABLE006: Missing DurableTask Attribute

Classes implementing orchestrator/activity interfaces should have the attribute.

**Severity:** Info

**Example (Violation):**
```csharp
// ❌ DURABLE006: Missing [DurableTask] attribute
public class MyOrchestration : TaskOrchestrator<string, string>
{
    public override async Task<string> RunAsync(
        TaskOrchestrationContext context, 
        string input)
    {
        return input;
    }
}
```

**Fix:**
```csharp
// ✅ Has [DurableTask] attribute
[DurableTask(nameof(MyOrchestration))]
public class MyOrchestration : TaskOrchestrator<string, string>
{
    public override async Task<string> RunAsync(
        TaskOrchestrationContext context, 
        string input)
    {
        return input;
    }
}
```

### DURABLE007: Non-Serializable Types

Input and output types must be serializable.

**Severity:** Error

**Example (Violation):**
```csharp
// ❌ DURABLE007: Task is not serializable
public class MyOrchestration : TaskOrchestrator<Task, string>
{
    // ...
}

// ❌ DURABLE007: Delegate is not serializable
public class BadActivity : TaskActivity<Action, string>
{
    // ...
}
```

**Fix:**
```csharp
// ✅ Use serializable types
public class MyOrchestration : TaskOrchestrator<MyInput, string>
{
    // ...
}

public class MyInput
{
    public string Data { get; set; } = "";
}
```

### DURABLE008: Static State Modification

Orchestrations should not modify static state.

**Severity:** Warning

**Example (Violation):**
```csharp
public class MyOrchestration : TaskOrchestrator<string, string>
{
    private static int _counter = 0;  // ❌ Static state
    
    public override async Task<string> RunAsync(
        TaskOrchestrationContext context, 
        string input)
    {
        // ❌ DURABLE008: Modifying static state
        _counter++;
        
        return _counter.ToString();
    }
}
```

**Fix:**
```csharp
public class MyOrchestration : TaskOrchestrator<CounterInput, string>
{
    public override async Task<string> RunAsync(
        TaskOrchestrationContext context, 
        CounterInput input)
    {
        // ✅ Use entities for state
        var counter = await context.CallEntityAsync<int>(
            new EntityInstanceId("Counter", "global"),
            "Increment");
        
        return counter.ToString();
    }
}
```

## Configuring Analyzers

### Severity Levels

Configure analyzer severity in `.editorconfig`:

```ini
# .editorconfig

[*.cs]
# Treat determinism violations as errors
dotnet_diagnostic.DURABLE001.severity = error

# Treat I/O warnings as errors in production
dotnet_diagnostic.DURABLE002.severity = error

# Downgrade missing attribute to suggestion
dotnet_diagnostic.DURABLE006.severity = suggestion
```

### Suppressing Warnings

Suppress specific warnings when intentional:

```csharp
public override async Task<string> RunAsync(
    TaskOrchestrationContext context, 
    string input)
{
    // Suppress for intentional non-determinism
    #pragma warning disable DURABLE001
    var randomForLogging = Guid.NewGuid().ToString();
    #pragma warning restore DURABLE001
    
    // Or use attribute
    [System.Diagnostics.CodeAnalysis.SuppressMessage(
        "DurableTask", 
        "DURABLE001",
        Justification = "Used only for logging, not affecting orchestration flow")]
    void LogWithId()
    {
        Console.WriteLine($"Log ID: {Guid.NewGuid()}");
    }
    
    return "result";
}
```

### Global Suppressions

Add global suppressions in `GlobalSuppressions.cs`:

```csharp
using System.Diagnostics.CodeAnalysis;

[assembly: SuppressMessage(
    "DurableTask",
    "DURABLE002",
    Scope = "type",
    Target = "~T:MyNamespace.LegacyOrchestration",
    Justification = "Legacy code pending migration")]
```

## Code Fixes

Many analyzers provide automatic code fixes:

### Quick Actions

In Visual Studio or VS Code, hover over the warning and use the lightbulb menu:

1. **DURABLE001** - Replace `DateTime.Now` → `context.CurrentUtcDateTime`
2. **DURABLE003** - Replace `Thread.Sleep` → `context.CreateTimer`
3. **DURABLE006** - Add `[DurableTask]` attribute

### Applying Fixes

```csharp
// Before fix
var now = DateTime.Now;

// After applying quick fix
var now = context.CurrentUtcDateTime;
```

## Custom Analyzers

Create custom analyzers for project-specific rules:

```csharp
[DiagnosticAnalyzer(LanguageNames.CSharp)]
public class CustomDurableAnalyzer : DiagnosticAnalyzer
{
    public const string DiagnosticId = "MYCUSTOM001";
    
    private static readonly DiagnosticDescriptor Rule = new(
        DiagnosticId,
        "Custom Rule",
        "Custom orchestration rule violation: {0}",
        "DurableTask.Custom",
        DiagnosticSeverity.Warning,
        isEnabledByDefault: true);
    
    public override ImmutableArray<DiagnosticDescriptor> SupportedDiagnostics 
        => ImmutableArray.Create(Rule);
    
    public override void Initialize(AnalysisContext context)
    {
        context.ConfigureGeneratedCodeAnalysis(
            GeneratedCodeAnalysisFlags.None);
        context.EnableConcurrentExecution();
        context.RegisterSyntaxNodeAction(
            AnalyzeMethodInvocation, 
            SyntaxKind.InvocationExpression);
    }
    
    private void AnalyzeMethodInvocation(SyntaxNodeAnalysisContext context)
    {
        // Custom analysis logic
    }
}
```

## Best Practices

### 1. Enable All Analyzers in CI/CD

```xml
<!-- Directory.Build.props -->
<PropertyGroup>
    <TreatWarningsAsErrors>true</TreatWarningsAsErrors>
    <WarningsAsErrors>DURABLE001;DURABLE002;DURABLE003</WarningsAsErrors>
</PropertyGroup>
```

### 2. Review Warnings Early

Address analyzer warnings during development, not after deployment.

### 3. Document Suppressions

Always include justification when suppressing:

```csharp
#pragma warning disable DURABLE001 // Justification: Only used for correlation ID in logs
```

### 4. Keep Analyzers Updated

Update the SDK package to get the latest analyzer rules:

```bash
dotnet add package Microsoft.DurableTask.Generators --version latest
```

## Troubleshooting

### Analyzers Not Running

1. Ensure package is referenced correctly
2. Restart Visual Studio / VS Code
3. Rebuild the solution

### False Positives

If you encounter false positives:

1. Check if it's actually safe
2. Suppress with justification if intentional
3. Report issues on GitHub

### Performance Issues

If analyzers slow down your IDE:

```xml
<!-- Disable live analysis, run only on build -->
<PropertyGroup>
    <RunAnalyzersDuringBuild>true</RunAnalyzersDuringBuild>
    <RunAnalyzersDuringLiveAnalysis>false</RunAnalyzersDuringLiveAnalysis>
</PropertyGroup>
```

## Next Steps

- [Writing Task Orchestrations](Writing-Task-Orchestrations.md) - Orchestration patterns
- [Orchestration Constraints](Orchestration-Constraints.md) - Determinism rules
- [Unit Testing](Unit-Testing.md) - Test your orchestrations

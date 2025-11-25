# Instance Queries

The Durable Task .NET SDK with **Durable Task Scheduler** provides powerful querying capabilities to find and filter orchestration instances. This enables building dashboards, management tools, and monitoring solutions.

## Overview

Instance queries allow you to:
- Find orchestrations by status, name, or time range
- Paginate through large result sets
- Build management and monitoring UIs
- Implement cleanup and maintenance tasks

## Basic Queries

### Query All Instances

```csharp
var query = new OrchestrationQuery();
OrchestrationQueryResult result = await _client.GetAllInstancesAsync(query);

foreach (var instance in result.Instances)
{
    Console.WriteLine($"{instance.InstanceId}: {instance.RuntimeStatus}");
}
```

### Query by Status

```csharp
var query = new OrchestrationQuery
{
    RuntimeStatus = new[]
    {
        OrchestrationRuntimeStatus.Running,
        OrchestrationRuntimeStatus.Pending
    }
};

var result = await _client.GetAllInstancesAsync(query);
Console.WriteLine($"Active instances: {result.Instances.Count()}");
```

### Query by Instance ID Prefix

```csharp
var query = new OrchestrationQuery
{
    InstanceIdPrefix = "order-2024-"
};

var result = await _client.GetAllInstancesAsync(query);
```

### Query by Time Range

```csharp
var query = new OrchestrationQuery
{
    CreatedFrom = DateTime.UtcNow.AddDays(-7),
    CreatedTo = DateTime.UtcNow
};

var result = await _client.GetAllInstancesAsync(query);
Console.WriteLine($"Instances created in last 7 days: {result.Instances.Count()}");
```

## Pagination

### Basic Pagination

```csharp
var query = new OrchestrationQuery
{
    PageSize = 50
};

var result = await _client.GetAllInstancesAsync(query);

// Check if there are more results
if (result.ContinuationToken != null)
{
    Console.WriteLine("More results available");
}
```

### Iterate Through All Pages

```csharp
public async IAsyncEnumerable<OrchestrationMetadata> GetAllInstances()
{
    string? continuationToken = null;
    
    do
    {
        var query = new OrchestrationQuery
        {
            PageSize = 100,
            ContinuationToken = continuationToken
        };
        
        var result = await _client.GetAllInstancesAsync(query);
        
        foreach (var instance in result.Instances)
        {
            yield return instance;
        }
        
        continuationToken = result.ContinuationToken;
        
    } while (continuationToken != null);
}

// Usage
await foreach (var instance in GetAllInstances())
{
    Console.WriteLine(instance.InstanceId);
}
```

### Limit Total Results

```csharp
public async Task<List<OrchestrationMetadata>> GetInstances(int maxResults)
{
    var instances = new List<OrchestrationMetadata>();
    string? continuationToken = null;
    
    while (instances.Count < maxResults)
    {
        int remaining = maxResults - instances.Count;
        
        var query = new OrchestrationQuery
        {
            PageSize = Math.Min(remaining, 100),
            ContinuationToken = continuationToken
        };
        
        var result = await _client.GetAllInstancesAsync(query);
        instances.AddRange(result.Instances);
        
        continuationToken = result.ContinuationToken;
        if (continuationToken == null)
            break;
    }
    
    return instances;
}
```

## Advanced Filtering

### Combined Filters

```csharp
var query = new OrchestrationQuery
{
    RuntimeStatus = new[] { OrchestrationRuntimeStatus.Failed },
    CreatedFrom = DateTime.UtcNow.AddDays(-1),
    CreatedTo = DateTime.UtcNow,
    InstanceIdPrefix = "order-",
    PageSize = 50
};

var result = await _client.GetAllInstancesAsync(query);
```

### Get Instances with Input/Output

```csharp
// By default, inputs/outputs are not fetched for performance
var query = new OrchestrationQuery
{
    RuntimeStatus = new[] { OrchestrationRuntimeStatus.Completed },
    PageSize = 10
};

var result = await _client.GetAllInstancesAsync(query);

// Fetch details for specific instances
foreach (var instance in result.Instances)
{
    var details = await _client.GetInstanceAsync(
        instance.InstanceId, 
        getInputsAndOutputs: true);
    
    if (details != null)
    {
        Console.WriteLine($"Input: {details.SerializedInput}");
        Console.WriteLine($"Output: {details.SerializedOutput}");
    }
}
```

## Query Patterns

### Dashboard Metrics

```csharp
public class DashboardMetrics
{
    private readonly DurableTaskClient _client;
    
    public async Task<DashboardData> GetDashboardData()
    {
        var tasks = new[]
        {
            GetCountByStatus(OrchestrationRuntimeStatus.Running),
            GetCountByStatus(OrchestrationRuntimeStatus.Pending),
            GetCountByStatus(OrchestrationRuntimeStatus.Completed),
            GetCountByStatus(OrchestrationRuntimeStatus.Failed),
            GetRecentFailures()
        };
        
        await Task.WhenAll(tasks);
        
        return new DashboardData
        {
            RunningCount = await tasks[0],
            PendingCount = await tasks[1],
            CompletedCount = await tasks[2],
            FailedCount = await tasks[3],
            RecentFailures = await tasks[4]
        };
    }
    
    private async Task<int> GetCountByStatus(OrchestrationRuntimeStatus status)
    {
        var query = new OrchestrationQuery
        {
            RuntimeStatus = new[] { status },
            PageSize = 1000
        };
        
        int count = 0;
        string? token = null;
        
        do
        {
            query.ContinuationToken = token;
            var result = await _client.GetAllInstancesAsync(query);
            count += result.Instances.Count();
            token = result.ContinuationToken;
        } while (token != null);
        
        return count;
    }
    
    private async Task<List<FailureInfo>> GetRecentFailures()
    {
        var query = new OrchestrationQuery
        {
            RuntimeStatus = new[] { OrchestrationRuntimeStatus.Failed },
            CreatedFrom = DateTime.UtcNow.AddHours(-24),
            PageSize = 10
        };
        
        var result = await _client.GetAllInstancesAsync(query);
        
        return result.Instances.Select(i => new FailureInfo
        {
            InstanceId = i.InstanceId,
            Name = i.Name,
            FailedAt = i.LastUpdatedAt,
            Error = i.FailureDetails?.ErrorMessage
        }).ToList();
    }
}
```

### Find Stuck Orchestrations

```csharp
public async Task<List<StuckOrchestration>> FindStuckOrchestrations(
    TimeSpan stuckThreshold)
{
    var query = new OrchestrationQuery
    {
        RuntimeStatus = new[] { OrchestrationRuntimeStatus.Running },
        CreatedTo = DateTime.UtcNow - stuckThreshold,
        PageSize = 100
    };
    
    var stuck = new List<StuckOrchestration>();
    
    await foreach (var instance in QueryAllPages(query))
    {
        stuck.Add(new StuckOrchestration
        {
            InstanceId = instance.InstanceId,
            Name = instance.Name,
            RunningFor = DateTime.UtcNow - instance.CreatedAt,
            LastUpdated = instance.LastUpdatedAt
        });
    }
    
    return stuck;
}
```

### Cleanup Old Instances

```csharp
public async Task<CleanupResult> CleanupOldInstances(
    TimeSpan retentionPeriod,
    int batchSize = 100)
{
    var cutoffDate = DateTime.UtcNow - retentionPeriod;
    var result = new CleanupResult();
    
    // Query completed instances older than retention period
    var query = new OrchestrationQuery
    {
        RuntimeStatus = new[]
        {
            OrchestrationRuntimeStatus.Completed,
            OrchestrationRuntimeStatus.Failed,
            OrchestrationRuntimeStatus.Terminated
        },
        CreatedTo = cutoffDate,
        PageSize = batchSize
    };
    
    await foreach (var instance in QueryAllPages(query))
    {
        try
        {
            var purgeResult = await _client.PurgeInstanceAsync(instance.InstanceId);
            result.PurgedCount += purgeResult.PurgedInstanceCount;
        }
        catch (Exception ex)
        {
            result.Errors.Add($"{instance.InstanceId}: {ex.Message}");
        }
    }
    
    return result;
}
```

### Search by Custom Criteria

```csharp
public async Task<List<OrchestrationMetadata>> SearchOrchestrations(
    SearchCriteria criteria)
{
    var query = new OrchestrationQuery
    {
        PageSize = criteria.PageSize ?? 50
    };
    
    // Apply filters
    if (criteria.Statuses?.Any() == true)
    {
        query.RuntimeStatus = criteria.Statuses.ToArray();
    }
    
    if (!string.IsNullOrEmpty(criteria.InstanceIdPrefix))
    {
        query.InstanceIdPrefix = criteria.InstanceIdPrefix;
    }
    
    if (criteria.CreatedAfter.HasValue)
    {
        query.CreatedFrom = criteria.CreatedAfter.Value;
    }
    
    if (criteria.CreatedBefore.HasValue)
    {
        query.CreatedTo = criteria.CreatedBefore.Value;
    }
    
    var results = new List<OrchestrationMetadata>();
    var maxResults = criteria.MaxResults ?? 1000;
    
    await foreach (var instance in QueryAllPages(query))
    {
        // Post-filter by orchestration name if specified
        if (!string.IsNullOrEmpty(criteria.OrchestrationName) &&
            instance.Name != criteria.OrchestrationName)
        {
            continue;
        }
        
        results.Add(instance);
        
        if (results.Count >= maxResults)
            break;
    }
    
    return results;
}
```

## HTTP API for Queries

### Query Controller

```csharp
[ApiController]
[Route("api/orchestrations")]
public class QueryController : ControllerBase
{
    private readonly DurableTaskClient _client;
    
    public QueryController(DurableTaskClient client)
    {
        _client = client;
    }
    
    [HttpGet]
    public async Task<ActionResult<QueryResponse>> Query(
        [FromQuery] string? status,
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
        
        if (!string.IsNullOrEmpty(status))
        {
            if (Enum.TryParse<OrchestrationRuntimeStatus>(status, true, out var runtimeStatus))
            {
                query.RuntimeStatus = new[] { runtimeStatus };
            }
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
            Instances = result.Instances.Select(MapToDto).ToList(),
            ContinuationToken = result.ContinuationToken,
            HasMore = result.ContinuationToken != null
        });
    }
    
    private static InstanceDto MapToDto(OrchestrationMetadata metadata)
    {
        return new InstanceDto
        {
            InstanceId = metadata.InstanceId,
            Name = metadata.Name,
            Status = metadata.RuntimeStatus.ToString(),
            CreatedAt = metadata.CreatedAt,
            LastUpdatedAt = metadata.LastUpdatedAt
        };
    }
}
```

### Query Models

```csharp
public class QueryResponse
{
    public List<InstanceDto> Instances { get; set; } = new();
    public string? ContinuationToken { get; set; }
    public bool HasMore { get; set; }
}

public class InstanceDto
{
    public string InstanceId { get; set; } = "";
    public string? Name { get; set; }
    public string Status { get; set; } = "";
    public DateTimeOffset CreatedAt { get; set; }
    public DateTimeOffset LastUpdatedAt { get; set; }
}

public class SearchCriteria
{
    public List<OrchestrationRuntimeStatus>? Statuses { get; set; }
    public string? InstanceIdPrefix { get; set; }
    public string? OrchestrationName { get; set; }
    public DateTime? CreatedAfter { get; set; }
    public DateTime? CreatedBefore { get; set; }
    public int? PageSize { get; set; }
    public int? MaxResults { get; set; }
}
```

## Helper Methods

### Query All Pages

```csharp
private async IAsyncEnumerable<OrchestrationMetadata> QueryAllPages(
    OrchestrationQuery query,
    [EnumeratorCancellation] CancellationToken cancellationToken = default)
{
    string? continuationToken = null;
    
    do
    {
        query.ContinuationToken = continuationToken;
        var result = await _client.GetAllInstancesAsync(query);
        
        foreach (var instance in result.Instances)
        {
            cancellationToken.ThrowIfCancellationRequested();
            yield return instance;
        }
        
        continuationToken = result.ContinuationToken;
        
    } while (continuationToken != null);
}
```

## Best Practices

### 1. Use Appropriate Page Sizes

```csharp
// ❌ Too large - memory issues
var query = new OrchestrationQuery { PageSize = 10000 };

// ✅ Reasonable page size
var query = new OrchestrationQuery { PageSize = 100 };
```

### 2. Filter as Much as Possible

```csharp
// ❌ Get all, then filter in memory
var all = await GetAllInstances();
var failed = all.Where(i => i.RuntimeStatus == OrchestrationRuntimeStatus.Failed);

// ✅ Filter in query
var query = new OrchestrationQuery
{
    RuntimeStatus = new[] { OrchestrationRuntimeStatus.Failed }
};
```

### 3. Use Cancellation Tokens

```csharp
public async Task<List<OrchestrationMetadata>> SearchWithTimeout(
    OrchestrationQuery query,
    TimeSpan timeout)
{
    using var cts = new CancellationTokenSource(timeout);
    
    var results = new List<OrchestrationMetadata>();
    
    try
    {
        await foreach (var instance in QueryAllPages(query, cts.Token))
        {
            results.Add(instance);
        }
    }
    catch (OperationCanceledException)
    {
        // Return partial results on timeout
    }
    
    return results;
}
```

### 4. Cache Results When Appropriate

```csharp
public class CachedOrchestrationStats
{
    private readonly DurableTaskClient _client;
    private readonly IMemoryCache _cache;
    
    public async Task<OrchestrationStats> GetStats()
    {
        return await _cache.GetOrCreateAsync("orchestration-stats", async entry =>
        {
            entry.AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(1);
            return await CalculateStats();
        });
    }
}
```

## Next Steps

- [Orchestration Instance Management](Orchestration-Instance-Management.md) - Manage instances
- [Diagnostics and Logging](Diagnostics-and-Logging.md) - Monitoring
- [Orchestration Versioning](Orchestration-Versioning.md) - Version queries

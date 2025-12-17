# Performance Profiling and Optimization

> Measure before optimizing, use profiling tools to find bottlenecks, and apply targeted optimizations based on data.

## Problem

Premature optimization wastes time on code that doesn't matter, while performance issues in critical paths go unfixed.

## Example

### ❌ Before (Guessing at Optimizations)

```csharp
public class OrderService
{
    // Optimizing the wrong thing
    public decimal CalculateTotal(Order order)
    {
        // Micro-optimization that doesn't matter
        decimal total = 0m;
        var count = order.Items.Count;
        for (int i = 0; i < count; i++)  // Avoiding foreach
        {
            total += order.Items[i].Price;
        }
        return total;
    }

    // Real bottleneck ignored
    public List<Order> GetOrders()
    {
        var orders = _context.Orders.ToList();  // Loads everything!

        foreach (var order in orders)  // N+1 query
        {
            order.Customer = _context.Customers.Find(order.CustomerId);
        }

        return orders;
    }
}
```

### ✅ After (Measured Optimizations)

```csharp
public class OrderService
{
    // Simple readable code for non-bottleneck
    public decimal CalculateTotal(Order order)
    {
        return order.Items.Sum(i => i.Price);
    }

    // Optimized actual bottleneck (measured with profiler)
    public List<Order> GetOrders()
    {
        return _context.Orders
            .Include(o => o.Customer)  // Single query
            .AsNoTracking()  // Read-only, no tracking overhead
            .ToList();
    }
}
```

## Best Practices

### 1. Measure First, Optimize Second

```csharp
// ❌ Premature optimization
public int CalculateSum(List<int> numbers)
{
    // Trying to be clever without measuring
    int sum = 0;
    unsafe
    {
        fixed (int* ptr = numbers.ToArray())
        {
            for (int i = 0; i < numbers.Count; i++)
            {
                sum += ptr[i];
            }
        }
    }
    return sum;
}

// ✅ Simple, readable code first
public int CalculateSum(List<int> numbers)
{
    return numbers.Sum();
}

// ✅ If profiler shows this is a bottleneck, then optimize
public int CalculateSumOptimized(List<int> numbers)
{
    return numbers.AsSpan().Sum();  // Measured improvement
}
```

### 2. Use BenchmarkDotNet for Micro-Benchmarks

```csharp
// ✅ Measure performance differences
[MemoryDiagnoser]
public class StringConcatBenchmark
{
    private readonly List<string> items = Enumerable.Range(0, 100)
        .Select(i => $"Item {i}")
        .ToList();

    [Benchmark(Baseline = true)]
    public string UsingPlusOperator()
    {
        string result = "";
        foreach (var item in items)
        {
            result += item;
        }
        return result;
    }

    [Benchmark]
    public string UsingStringBuilder()
    {
        var sb = new StringBuilder();
        foreach (var item in items)
        {
            sb.Append(item);
        }
        return sb.ToString();
    }

    [Benchmark]
    public string UsingStringJoin()
    {
        return string.Join("", items);
    }
}
```

### 3. Profile with Visual Studio Profiler

```csharp
// ✅ Use profiling tools to find real bottlenecks
// Tools -> Performance Profiler
// - CPU Usage
// - Memory Usage
// - .NET Object Allocation
// - Database
// - Instrumentation

public class OrderProcessor
{
    // Profiler showed this as bottleneck
    public List<OrderSummary> GetOrderSummaries()
    {
        return _context.Orders
            .Select(o => new OrderSummary  // Projection in DB
            {
                Id = o.Id,
                Total = o.Total,
                CustomerName = o.Customer.Name
            })
            .ToList();
    }
}
```

### 4. Use Diagnostic Tools

```csharp
// ✅ Add diagnostic logging for performance
public async Task<Order> ProcessOrderAsync(OrderId orderId)
{
    using var activity = DiagnosticSource.StartActivity("ProcessOrder");
    activity?.AddTag("OrderId", orderId);

    var sw = Stopwatch.StartNew();
    try
    {
        var order = await _repository.GetOrderAsync(orderId);

        sw.Stop();
        _logger.LogInformation(
            "Retrieved order in {ElapsedMs}ms",
            sw.ElapsedMilliseconds);

        return order;
    }
    catch (Exception ex)
    {
        activity?.SetStatus(ActivityStatusCode.Error, ex.Message);
        throw;
    }
}
```

### 5. Use MiniProfiler for Web Applications

```csharp
// ✅ Profile database queries in development
public IActionResult GetOrders()
{
    using (MiniProfiler.Current.Step("Get Orders"))
    {
        using (MiniProfiler.Current.Step("Database Query"))
        {
            var orders = _context.Orders
                .Include(o => o.Customer)
                .ToList();
        }

        using (MiniProfiler.Current.Step("Map to DTO"))
        {
            return Ok(orders.Select(o => new OrderDto(o)));
        }
    }
}
```

### 6. Identify Allocations with Allocation Profiler

```csharp
// ❌ Hidden allocations
public string FormatOrders(List<Order> orders)
{
    return string.Join(", ", orders.Select(o => o.Id.ToString()));
    // Allocates: IEnumerable, array for Join, strings
}

// ✅ Measured and reduced allocations
public string FormatOrders(List<Order> orders)
{
    var sb = new StringBuilder(orders.Count * 10);
    for (int i = 0; i < orders.Count; i++)
    {
        if (i > 0) sb.Append(", ");
        sb.Append(orders[i].Id);
    }
    return sb.ToString();
}
```

### 7. Use PerfView for Deep Analysis

```csharp
// ✅ Analyze with PerfView (Windows)
// - CPU sampling
// - Memory allocation
// - GC collections
// - JIT compilation
// - Thread time

// Example: Find why GC is running frequently
public void ProcessLargeDataSet()
{
    // PerfView shows allocations here
    var data = LoadAllData();  // 1 GB of data!

    // After profiling, use streaming instead
    ProcessDataInChunks();
}
```

### 8. Monitor Production with Application Insights

```csharp
// ✅ Track performance in production
public class OrderController : Controller
{
    private readonly TelemetryClient telemetry;

    [HttpPost]
    public async Task<IActionResult> PlaceOrder(PlaceOrderCommand command)
    {
        var sw = Stopwatch.StartNew();
        try
        {
            var result = await _mediator.Send(command);
            sw.Stop();

            telemetry.TrackMetric(
                "OrderProcessingTime",
                sw.ElapsedMilliseconds,
                new Dictionary<string, string>
                {
                    ["CustomerId"] = command.CustomerId.ToString()
                });

            return Ok(result);
        }
        catch (Exception ex)
        {
            telemetry.TrackException(ex);
            throw;
        }
    }
}
```

### 9. Use Performance Counters

```csharp
// ✅ Track custom metrics
public class OrderMetrics
{
    private static readonly Counter<int> OrdersProcessed =
        Meter.CreateCounter<int>("orders.processed");

    private static readonly Histogram<double> OrderProcessingTime =
        Meter.CreateHistogram<double>("orders.processing_time");

    private static readonly Meter Meter = new("OrderService", "1.0");

    public async Task ProcessOrderAsync(Order order)
    {
        var sw = Stopwatch.StartNew();
        try
        {
            await ProcessOrderInternal(order);
            OrdersProcessed.Add(1);
        }
        finally
        {
            sw.Stop();
            OrderProcessingTime.Record(sw.Elapsed.TotalMilliseconds);
        }
    }
}
```

### 10. Identify Database Query Issues

```csharp
// ✅ Log slow queries
public class SlowQueryLogger : DbCommandInterceptor
{
    private readonly ILogger<SlowQueryLogger> logger;
    private const int SlowQueryThresholdMs = 100;

    public override DbDataReader ReaderExecuted(
        DbCommand command,
        CommandExecutedEventData eventData,
        DbDataReader result)
    {
        if (eventData.Duration.TotalMilliseconds > SlowQueryThresholdMs)
        {
            logger.LogWarning(
                "Slow query detected: {Query} took {ElapsedMs}ms",
                command.CommandText,
                eventData.Duration.TotalMilliseconds);
        }

        return base.ReaderExecuted(command, eventData, result);
    }
}
```

### 11. Use Caching for Expensive Operations

```csharp
// ❌ No caching, repeated expensive calculation
public decimal GetTaxRate(string region)
{
    return _externalApi.GetTaxRate(region);  // 500ms API call
}

// ✅ Cache expensive results (after profiling)
private readonly MemoryCache cache = new(new MemoryCacheOptions());

public decimal GetTaxRate(string region)
{
    return cache.GetOrCreate(
        $"tax-rate-{region}",
        entry =>
        {
            entry.AbsoluteExpirationRelativeToNow = TimeSpan.FromHours(24);
            return _externalApi.GetTaxRate(region);
        });
}
```

### 12. Test Under Load

```csharp
// ✅ Load test to find bottlenecks
[Test]
public async Task LoadTest_ProcessOrders()
{
    var tasks = new List<Task>();

    // Simulate 100 concurrent requests
    for (int i = 0; i < 100; i++)
    {
        var orderId = new OrderId(Guid.NewGuid());
        tasks.Add(Task.Run(() => _service.ProcessOrderAsync(orderId)));
    }

    var sw = Stopwatch.StartNew();
    await Task.WhenAll(tasks);
    sw.Stop();

    _output.WriteLine($"100 orders processed in {sw.ElapsedMilliseconds}ms");
    Assert.That(sw.ElapsedMilliseconds, Is.LessThan(5000));
}
```

## Profiling Tools

- **BenchmarkDotNet** - Micro-benchmarks
- **Visual Studio Profiler** - CPU, memory, allocations
- **PerfView** - Windows performance analysis
- **dotnet-trace** - Cross-platform profiling
- **dotnet-counters** - Performance counters
- **MiniProfiler** - Web application profiling
- **Application Insights** - Production monitoring
- **Glimpse** - ASP.NET debugging

## Symptoms

- Slow application response times
- High CPU usage
- Memory leaks
- Frequent GC pauses
- Database timeout errors
- Poor user experience

## Benefits

- **Targeted optimizations** based on real bottlenecks
- **Measurable improvements** with before/after metrics
- **No wasted effort** on non-issues
- **Production insights** with monitoring
- **Objective decisions** based on data

## See Also

- [Allocation Budget](./allocation-budget.md) — Zero-allocation patterns
- [LINQ Query Optimization](./linq-query-optimization.md) — Database performance
- [Memory Safety](./memory-safety-span.md) — Span<T> for performance
- [Struct Layout](./struct-layout.md) — Memory optimization

# Batch Operations (Type-Safe Bulk Processing)

> Processing multiple items with loops and mixed results in a single array—use typed batch operations with explicit success/failure tracking.

## Problem

APIs often need to process multiple items at once (bulk create, update, delete). Naive implementations either fail atomically (all or nothing) or return inconsistent results. Clients need to know which items succeeded, which failed, and why—without parsing error messages.

## Example

### ❌ Before

```csharp
[ApiController]
[Route("api/products")]
public class ProductsController : ControllerBase
{
    [HttpPost("batch")]
    public async Task<IActionResult> CreateProducts([FromBody] List<CreateProductDto> products)
    {
        // All or nothing—one failure fails everything
        var results = new List<Product>();
        
        foreach (var dto in products)
        {
            try
            {
                var product = await _service.CreateAsync(dto);
                results.Add(product);
            }
            catch (Exception ex)
            {
                // What about already-created products?
                return BadRequest($"Failed to create product: {ex.Message}");
            }
        }
        
        return Ok(results);
    }
    
    [HttpPut("batch")]
    public async Task<IActionResult> UpdateProducts([FromBody] List<UpdateProductDto> products)
    {
        // Mixed results with no structure
        var results = new List<object>();
        
        foreach (var dto in products)
        {
            try
            {
                var product = await _service.UpdateAsync(dto);
                results.Add(new { success = true, product });
            }
            catch (Exception ex)
            {
                // Mixing success and error objects—no type safety
                results.Add(new { success = false, error = ex.Message, id = dto.Id });
            }
        }
        
        return Ok(results);  // Client must inspect each object
    }
}
```

**Problems:**
- All-or-nothing atomicity not appropriate for batch operations
- Mixed success/failure results without type safety
- No correlation between input and output
- Client must parse unstructured error messages
- No partial retry capability
- Lost context about which item failed

### ✅ After

```csharp
/// <summary>
/// Result for a single item in a batch operation.
/// Maintains correlation with input.
/// </summary>
public abstract record BatchItemResult<TInput, TSuccess>
{
    public required int Index { get; init; }
    public required TInput Input { get; init; }
    
    private BatchItemResult() { }
    
    public sealed record Success(TSuccess Value) : BatchItemResult<TInput, TSuccess>;
    
    public sealed record Failure(
        string ErrorCode,
        string Message,
        Option<IReadOnlyDictionary<string, string>> ValidationErrors = default) 
        : BatchItemResult<TInput, TSuccess>;
}

/// <summary>
/// Result of batch operation with summary statistics.
/// </summary>
public sealed record BatchResult<TInput, TSuccess>
{
    public required IReadOnlyList<BatchItemResult<TInput, TSuccess>> Results { get; init; }
    public required int TotalCount { get; init; }
    public required int SuccessCount { get; init; }
    public required int FailureCount { get; init; }
    public required TimeSpan Duration { get; init; }
    
    /// <summary>
    /// Gets all successful results.
    /// </summary>
    public IEnumerable<TSuccess> Successes => 
        Results.OfType<BatchItemResult<TInput, TSuccess>.Success>()
               .Select(s => s.Value);
    
    /// <summary>
    /// Gets all failures.
    /// </summary>
    public IEnumerable<BatchItemResult<TInput, TSuccess>.Failure> Failures => 
        Results.OfType<BatchItemResult<TInput, TSuccess>.Failure>();
    
    /// <summary>
    /// Indicates if all operations succeeded.
    /// </summary>
    public bool IsFullSuccess => FailureCount == 0;
    
    /// <summary>
    /// Indicates if all operations failed.
    /// </summary>
    public bool IsFullFailure => SuccessCount == 0;
    
    /// <summary>
    /// Indicates partial success (some succeeded, some failed).
    /// </summary>
    public bool IsPartialSuccess => SuccessCount > 0 && FailureCount > 0;
}

/// <summary>
/// Batch operation configuration.
/// </summary>
public sealed record BatchConfig
{
    /// <summary>
    /// Maximum number of items allowed in a single batch.
    /// </summary>
    public int MaxBatchSize { get; init; } = 100;
    
    /// <summary>
    /// Whether to stop processing on first failure.
    /// </summary>
    public bool StopOnFirstFailure { get; init; } = false;
    
    /// <summary>
    /// Maximum degree of parallelism (1 = sequential).
    /// </summary>
    public int MaxDegreeOfParallelism { get; init; } = 1;
}

/// <summary>
/// Service for processing batch operations.
/// </summary>
public interface IBatchProcessor
{
    Task<BatchResult<TInput, TSuccess>> ProcessAsync<TInput, TSuccess>(
        IReadOnlyList<TInput> inputs,
        Func<TInput, Task<Result<TSuccess, BatchError>>> processor,
        BatchConfig config);
}

public sealed record BatchError(
    string ErrorCode,
    string Message,
    Option<IReadOnlyDictionary<string, string>> ValidationErrors = default);

public class BatchProcessor : IBatchProcessor
{
    public async Task<BatchResult<TInput, TSuccess>> ProcessAsync<TInput, TSuccess>(
        IReadOnlyList<TInput> inputs,
        Func<TInput, Task<Result<TSuccess, BatchError>>> processor,
        BatchConfig config)
    {
        var stopwatch = System.Diagnostics.Stopwatch.StartNew();
        
        // Validate batch size
        if (inputs.Count > config.MaxBatchSize)
        {
            throw new ArgumentException(
                $"Batch size {inputs.Count} exceeds maximum {config.MaxBatchSize}");
        }
        
        var results = new List<BatchItemResult<TInput, TSuccess>>();
        var successCount = 0;
        var failureCount = 0;
        
        if (config.MaxDegreeOfParallelism == 1)
        {
            // Sequential processing
            for (int i = 0; i < inputs.Count; i++)
            {
                var result = await ProcessItemAsync(inputs[i], i, processor);
                results.Add(result);
                
                if (result is BatchItemResult<TInput, TSuccess>.Success)
                {
                    successCount++;
                }
                else
                {
                    failureCount++;
                    
                    if (config.StopOnFirstFailure)
                        break;
                }
            }
        }
        else
        {
            // Parallel processing
            var tasks = inputs.Select((input, index) => 
                ProcessItemAsync(input, index, processor)).ToList();
            
            var processedResults = await Task.WhenAll(tasks);
            results.AddRange(processedResults);
            
            successCount = results.Count(r => r is BatchItemResult<TInput, TSuccess>.Success);
            failureCount = results.Count - successCount;
        }
        
        stopwatch.Stop();
        
        return new BatchResult<TInput, TSuccess>
        {
            Results = results,
            TotalCount = inputs.Count,
            SuccessCount = successCount,
            FailureCount = failureCount,
            Duration = stopwatch.Elapsed
        };
    }
    
    private async Task<BatchItemResult<TInput, TSuccess>> ProcessItemAsync<TInput, TSuccess>(
        TInput input,
        int index,
        Func<TInput, Task<Result<TSuccess, BatchError>>> processor)
    {
        try
        {
            var result = await processor(input);
            
            return result.Match(
                onSuccess: value => new BatchItemResult<TInput, TSuccess>.Success(value)
                {
                    Index = index,
                    Input = input
                } as BatchItemResult<TInput, TSuccess>,
                onFailure: error => new BatchItemResult<TInput, TSuccess>.Failure(
                    error.ErrorCode,
                    error.Message,
                    error.ValidationErrors)
                {
                    Index = index,
                    Input = input
                });
        }
        catch (Exception ex)
        {
            return new BatchItemResult<TInput, TSuccess>.Failure(
                "unexpected_error",
                ex.Message,
                Option<IReadOnlyDictionary<string, string>>.None)
            {
                Index = index,
                Input = input
            };
        }
    }
}

// Service layer
/// <summary>
/// Unit type representing "no value" or void as a value.
/// </summary>
public readonly record struct Unit
{
    public static Unit Value { get; } = new();
}

public interface IProductService
{
    Task<Result<Product, BatchError>> CreateProductAsync(CreateProductDto dto);
    Task<Result<Product, BatchError>> UpdateProductAsync(UpdateProductDto dto);
    Task<Result<Unit, BatchError>> DeleteProductAsync(ProductId id);
}

public class ProductService : IProductService
{
    public async Task<Result<Product, BatchError>> CreateProductAsync(CreateProductDto dto)
    {
        // Validate
        if (string.IsNullOrWhiteSpace(dto.Name))
        {
            return Result<Product, BatchError>.Failure(new BatchError(
                "validation_error",
                "Product name is required",
                Option<IReadOnlyDictionary<string, string>>.Some(
                    new Dictionary<string, string> { ["name"] = "Required" })));
        }
        
        // Create
        var product = new Product
        {
            Id = ProductId.New(),
            Name = dto.Name,
            Price = dto.Price
        };
        
        await _repository.SaveAsync(product);
        
        return Result<Product, BatchError>.Success(product);
    }
    
    public async Task<Result<Product, BatchError>> UpdateProductAsync(UpdateProductDto dto)
    {
        var existing = await _repository.GetByIdAsync(dto.Id);
        
        if (existing == null)
        {
            return Result<Product, BatchError>.Failure(new BatchError(
                "not_found",
                $"Product {dto.Id} not found"));
        }
        
        existing.Name = dto.Name;
        existing.Price = dto.Price;
        
        await _repository.SaveAsync(existing);
        
        return Result<Product, BatchError>.Success(existing);
    }
    
    public async Task<Result<Unit, BatchError>> DeleteProductAsync(ProductId id)
    {
        var existing = await _repository.GetByIdAsync(id);
        
        if (existing == null)
        {
            return Result<Unit, BatchError>.Failure(new BatchError(
                "not_found",
                $"Product {id} not found"));
        }
        
        await _repository.DeleteAsync(id);
        
        return Result<Unit, BatchError>.Success(Unit.Value);
    }
}

// Controller
[ApiController]
[Route("api/products")]
public class ProductsController : ControllerBase
{
    private readonly IProductService _service;
    private readonly IBatchProcessor _batchProcessor;
    
    [HttpPost("batch")]
    public async Task<IActionResult> CreateProducts(
        [FromBody] IReadOnlyList<CreateProductDto> products)
    {
        var config = new BatchConfig
        {
            MaxBatchSize = 100,
            StopOnFirstFailure = false,
            MaxDegreeOfParallelism = 10  // Process 10 at a time
        };
        
        var result = await _batchProcessor.ProcessAsync(
            products,
            dto => _service.CreateProductAsync(dto),
            config);
        
        // Return 207 Multi-Status for partial success
        var statusCode = result.IsFullSuccess ? 200 :
                        result.IsFullFailure ? 400 :
                        207;  // Multi-Status
        
        return StatusCode(statusCode, result);
    }
    
    [HttpPut("batch")]
    public async Task<IActionResult> UpdateProducts(
        [FromBody] IReadOnlyList<UpdateProductDto> products)
    {
        var config = new BatchConfig
        {
            MaxBatchSize = 100,
            StopOnFirstFailure = false
        };
        
        var result = await _batchProcessor.ProcessAsync(
            products,
            dto => _service.UpdateProductAsync(dto),
            config);
        
        var statusCode = result.IsFullSuccess ? 200 :
                        result.IsFullFailure ? 400 :
                        207;
        
        return StatusCode(statusCode, result);
    }
    
    [HttpDelete("batch")]
    public async Task<IActionResult> DeleteProducts(
        [FromBody] IReadOnlyList<int> productIds)
    {
        var config = new BatchConfig { MaxBatchSize = 100 };
        
        var result = await _batchProcessor.ProcessAsync(
            productIds.Select(id => ProductId.From(id)).ToList(),
            id => _service.DeleteProductAsync(id),
            config);
        
        var statusCode = result.IsFullSuccess ? 204 :
                        result.IsFullFailure ? 400 :
                        207;
        
        return StatusCode(statusCode, result);
    }
}
```

## Retry Failed Items

```csharp
/// <summary>
/// Request to retry failed items from previous batch.
/// </summary>
public sealed record BatchRetryRequest<TInput>
{
    public required IReadOnlyList<RetryItem<TInput>> Items { get; init; }
}

public sealed record RetryItem<TInput>
{
    public required int OriginalIndex { get; init; }
    public required TInput Input { get; init; }
}

[HttpPost("batch/retry")]
public async Task<IActionResult> RetryFailedProducts(
    [FromBody] BatchRetryRequest<CreateProductDto> request)
{
    var config = new BatchConfig { MaxBatchSize = 100 };
    
    var result = await _batchProcessor.ProcessAsync(
        request.Items.Select(i => i.Input).ToList(),
        dto => _service.CreateProductAsync(dto),
        config);
    
    return StatusCode(result.IsFullSuccess ? 200 : 207, result);
}
```

## Transactional Batches

```csharp
/// <summary>
/// Batch processor with transaction support.
/// </summary>
public class TransactionalBatchProcessor : IBatchProcessor
{
    private readonly IDbContext _dbContext;
    
    public async Task<BatchResult<TInput, TSuccess>> ProcessAsync<TInput, TSuccess>(
        IReadOnlyList<TInput> inputs,
        Func<TInput, Task<Result<TSuccess, BatchError>>> processor,
        BatchConfig config)
    {
        using var transaction = await _dbContext.BeginTransactionAsync();
        
        try
        {
            var result = await ProcessBatchAsync(inputs, processor, config);
            
            // Commit only if all succeeded
            if (result.IsFullSuccess)
            {
                await transaction.CommitAsync();
            }
            else
            {
                await transaction.RollbackAsync();
            }
            
            return result;
        }
        catch
        {
            await transaction.RollbackAsync();
            throw;
        }
    }
}
```

## Testing

```csharp
public class BatchOperationTests
{
    [Fact]
    public async Task BatchCreate_AllSucceed_Returns200()
    {
        var dtos = new List<CreateProductDto>
        {
            new() { Name = "Product 1", Price = 10 },
            new() { Name = "Product 2", Price = 20 }
        };
        
        var result = await _batchProcessor.ProcessAsync(
            dtos,
            dto => _service.CreateProductAsync(dto),
            new BatchConfig());
        
        Assert.True(result.IsFullSuccess);
        Assert.Equal(2, result.SuccessCount);
        Assert.Equal(0, result.FailureCount);
    }
    
    [Fact]
    public async Task BatchCreate_PartialFailure_Returns207()
    {
        var dtos = new List<CreateProductDto>
        {
            new() { Name = "Product 1", Price = 10 },
            new() { Name = "", Price = 20 }  // Invalid
        };
        
        var result = await _batchProcessor.ProcessAsync(
            dtos,
            dto => _service.CreateProductAsync(dto),
            new BatchConfig());
        
        Assert.True(result.IsPartialSuccess);
        Assert.Equal(1, result.SuccessCount);
        Assert.Equal(1, result.FailureCount);
    }
    
    [Fact]
    public async Task BatchResult_PreservesInputCorrelation()
    {
        var dtos = new List<CreateProductDto>
        {
            new() { Name = "Product 1", Price = 10 },
            new() { Name = "Product 2", Price = 20 }
        };
        
        var result = await _batchProcessor.ProcessAsync(
            dtos,
            dto => _service.CreateProductAsync(dto),
            new BatchConfig());
        
        // Can correlate results back to input
        Assert.Equal(dtos[0].Name, result.Results[0].Input.Name);
        Assert.Equal(dtos[1].Name, result.Results[1].Input.Name);
    }
}
```

## Why It's a Problem

1. **All-or-nothing**: One failure aborts entire batch
2. **Lost context**: Can't tell which items failed
3. **No retry**: Must resend entire batch
4. **Type unsafety**: Mixed success/error results
5. **Poor performance**: Sequential processing only

## Symptoms

- Batch operations fail completely or succeed completely
- Mixed anonymous objects in results
- Client must parse error messages
- No way to retry only failed items
- Performance issues with large batches

## Benefits

- **Partial success**: Process all items independently
- **Type safety**: Structured success/failure results
- **Correlation**: Maintain relationship between input and output
- **Retry support**: Client can retry only failed items
- **Performance**: Configurable parallelism

## See Also

- [Honest Functions](./honest-functions.md) — explicit results
- [Idempotency Keys](./idempotency-keys.md) — safe retries
- [Pagination Cursors](./pagination-cursors.md) — handling large datasets
- [Rate Limiting](./rate-limiting-tokens.md) — throttling batch operations

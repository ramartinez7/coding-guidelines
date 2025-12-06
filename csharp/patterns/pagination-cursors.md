# Pagination Cursors (Opaque Token Types)

> Using page numbers or offsets for pagination exposes implementation details and breaks when data changes—use opaque cursor types for stable, efficient pagination.

## Problem

Traditional offset-based pagination (`page=2`, `limit=50`) has problems: items can be skipped or duplicated when data changes between requests, and it requires scanning all previous rows (inefficient for large offsets). Exposing the cursor format (e.g., timestamps, IDs) leaks implementation details.

## Example

### ❌ Before

```csharp
[ApiController]
[Route("api/products")]
public class ProductsController : ControllerBase
{
    [HttpGet]
    public IActionResult GetProducts(int page = 1, int pageSize = 50)
    {
        // Offset-based pagination
        var offset = (page - 1) * pageSize;
        var products = _repository.GetProducts(offset, pageSize);
        
        return Ok(new
        {
            page,
            pageSize,
            data = products,
            nextPage = page + 1  // What if items were added/removed?
        });
    }
}

// Repository
public List<Product> GetProducts(int offset, int limit)
{
    // Inefficient: scans all rows up to offset
    return _db.Products
        .OrderBy(p => p.Id)
        .Skip(offset)
        .Take(limit)
        .ToList();
}
```

**Problems:**
- Skips or duplicates items when data changes
- Inefficient for large offsets (must scan all previous rows)
- Page numbers meaningless if items added/removed
- Exposes implementation details

### ✅ After

```csharp
/// <summary>
/// Opaque pagination cursor that encodes position in result set.
/// Cannot be constructed directly—must come from pagination response.
/// </summary>
public sealed record PaginationCursor
{
    private readonly string _encodedValue;
    
    private PaginationCursor(string encodedValue)
    {
        _encodedValue = encodedValue;
    }
    
    /// <summary>
    /// Creates cursor from internal state (e.g., last ID + timestamp).
    /// </summary>
    internal static PaginationCursor Create(long lastId, DateTime lastCreated)
    {
        // Encode as base64 to hide implementation
        var json = JsonSerializer.Serialize(new { lastId, lastCreated });
        var encoded = Convert.ToBase64String(Encoding.UTF8.GetBytes(json));
        return new PaginationCursor(encoded);
    }
    
    /// <summary>
    /// Decodes cursor back to internal state.
    /// </summary>
    internal Result<(long LastId, DateTime LastCreated), string> Decode()
    {
        try
        {
            var json = Encoding.UTF8.GetString(Convert.FromBase64String(_encodedValue));
            var data = JsonSerializer.Deserialize<CursorData>(json);
            
            if (data == null)
                return Result<(long, DateTime), string>.Failure("Invalid cursor format");
            
            return Result<(long, DateTime), string>.Success((data.LastId, data.LastCreated));
        }
        catch (Exception ex)
        {
            return Result<(long, DateTime), string>.Failure($"Failed to decode cursor: {ex.Message}");
        }
    }
    
    public override string ToString() => _encodedValue;
    
    public static Result<PaginationCursor, string> Parse(string encoded)
    {
        if (string.IsNullOrWhiteSpace(encoded))
            return Result<PaginationCursor, string>.Failure("Cursor cannot be empty");
        
        var cursor = new PaginationCursor(encoded);
        
        // Validate by attempting decode
        return cursor.Decode().Match(
            onSuccess: _ => Result<PaginationCursor, string>.Success(cursor),
            onFailure: error => Result<PaginationCursor, string>.Failure(error)
        );
    }
    
    private record CursorData(long LastId, DateTime LastCreated);
}

/// <summary>
/// Paginated response with opaque cursors.
/// </summary>
public sealed record PaginatedResponse<T>
{
    public required IReadOnlyList<T> Data { get; init; }
    public required int PageSize { get; init; }
    public PaginationCursor? NextCursor { get; init; }
    public PaginationCursor? PreviousCursor { get; init; }
    public required bool HasMore { get; init; }
}

[ApiController]
[Route("api/products")]
public class ProductsController : ControllerBase
{
    [HttpGet]
    public IActionResult GetProducts([FromQuery] string? cursor = null, [FromQuery] int pageSize = 50)
    {
        // Parse cursor if provided
        var parsedCursor = string.IsNullOrEmpty(cursor)
            ? Option<PaginationCursor>.None
            : PaginationCursor.Parse(cursor).Match(
                onSuccess: c => Option<PaginationCursor>.Some(c),
                onFailure: _ => Option<PaginationCursor>.None
            );
        
        var response = _repository.GetProducts(parsedCursor, pageSize);
        
        return Ok(response);
    }
}

// Repository with cursor-based pagination
public class ProductRepository
{
    public PaginatedResponse<Product> GetProducts(
        Option<PaginationCursor> cursor,
        int pageSize)
    {
        // Build query
        var query = _db.Products.OrderBy(p => p.CreatedAt).ThenBy(p => p.Id);
        
        // Apply cursor if present
        query = cursor.Match(
            onSome: c =>
            {
                var decoded = c.Decode().Value;  // We know it's valid from controller
                return query.Where(p => 
                    p.CreatedAt > decoded.LastCreated ||
                    (p.CreatedAt == decoded.LastCreated && p.Id > decoded.LastId));
            },
            onNone: () => query
        );
        
        // Fetch one extra to check if there's more
        var products = query.Take(pageSize + 1).ToList();
        var hasMore = products.Count > pageSize;
        
        if (hasMore)
            products.RemoveAt(products.Count - 1);
        
        // Create next cursor from last item
        PaginationCursor? nextCursor = null;
        if (hasMore && products.Count > 0)
        {
            var last = products[^1];
            nextCursor = PaginationCursor.Create(last.Id, last.CreatedAt);
        }
        
        return new PaginatedResponse<Product>
        {
            Data = products,
            PageSize = pageSize,
            NextCursor = nextCursor,
            HasMore = hasMore,
            PreviousCursor = null  // Backward pagination optional
        };
    }
}
```

## Bidirectional Pagination

```csharp
public sealed record PaginationCursor
{
    // ... existing members ...
    
    public PaginationDirection Direction { get; }
    
    internal static PaginationCursor CreateForward(long lastId, DateTime lastCreated)
    {
        var json = JsonSerializer.Serialize(new 
        { 
            lastId, 
            lastCreated, 
            direction = "forward" 
        });
        var encoded = Convert.ToBase64String(Encoding.UTF8.GetBytes(json));
        return new PaginationCursor(encoded) { Direction = PaginationDirection.Forward };
    }
    
    internal static PaginationCursor CreateBackward(long firstId, DateTime firstCreated)
    {
        var json = JsonSerializer.Serialize(new 
        { 
            firstId, 
            firstCreated, 
            direction = "backward" 
        });
        var encoded = Convert.ToBase64String(Encoding.UTF8.GetBytes(json));
        return new PaginationCursor(encoded) { Direction = PaginationDirection.Backward };
    }
}

public enum PaginationDirection
{
    Forward,
    Backward
}
```

## Stable Sorting

```csharp
/// <summary>
/// Cursor that includes sort key to maintain stable ordering.
/// </summary>
public sealed record SortedPaginationCursor
{
    internal static SortedPaginationCursor Create(
        long id,
        string sortField,
        object sortValue)
    {
        var json = JsonSerializer.Serialize(new 
        { 
            id, 
            sortField, 
            sortValue = sortValue.ToString() 
        });
        var encoded = Convert.ToBase64String(Encoding.UTF8.GetBytes(json));
        return new SortedPaginationCursor(encoded);
    }
    
    private SortedPaginationCursor(string encodedValue)
    {
        _encodedValue = encodedValue;
    }
    
    private readonly string _encodedValue;
    public override string ToString() => _encodedValue;
}

// Usage: sort by price, then by ID for stability
public PaginatedResponse<Product> GetProductsByPrice(
    Option<SortedPaginationCursor> cursor,
    int pageSize)
{
    var query = _db.Products.OrderBy(p => p.Price).ThenBy(p => p.Id);
    
    query = cursor.Match(
        onSome: c =>
        {
            var decoded = c.Decode();
            return query.Where(p => 
                p.Price > decoded.SortValue ||
                (p.Price == decoded.SortValue && p.Id > decoded.Id));
        },
        onNone: () => query
    );
    
    // ... rest of implementation
}
```

## Testing

```csharp
public class PaginationTests
{
    [Fact]
    public void GetProducts_WithoutCursor_ReturnsFirstPage()
    {
        var response = _repository.GetProducts(Option<PaginationCursor>.None, 10);
        
        Assert.Equal(10, response.Data.Count);
        Assert.True(response.HasMore);
        Assert.NotNull(response.NextCursor);
    }
    
    [Fact]
    public void GetProducts_WithCursor_ReturnsNextPage()
    {
        var firstPage = _repository.GetProducts(Option<PaginationCursor>.None, 10);
        var secondPage = _repository.GetProducts(
            Option<PaginationCursor>.Some(firstPage.NextCursor!), 10);
        
        Assert.Equal(10, secondPage.Data.Count);
        Assert.NotEqual(firstPage.Data[0].Id, secondPage.Data[0].Id);
    }
    
    [Fact]
    public void GetProducts_LastPage_HasNoNextCursor()
    {
        // Navigate to last page
        var cursor = Option<PaginationCursor>.None;
        PaginatedResponse<Product> response;
        
        do
        {
            response = _repository.GetProducts(cursor, 10);
            cursor = response.NextCursor != null 
                ? Option<PaginationCursor>.Some(response.NextCursor) 
                : Option<PaginationCursor>.None;
        } while (response.HasMore);
        
        Assert.False(response.HasMore);
        Assert.Null(response.NextCursor);
    }
}
```

## Why It's a Problem

1. **Unstable results**: Offset pagination skips/duplicates items when data changes
2. **Inefficient**: Large offsets require scanning many rows
3. **Implementation leakage**: Exposing IDs or timestamps as cursors
4. **No consistency**: Page numbers meaningless across requests

## Symptoms

- `Skip(offset).Take(limit)` in queries
- Page numbers in URLs
- Duplicate or missing items in paginated results
- Slow performance with large page numbers
- Raw IDs or timestamps used as cursors

## Benefits

- **Stable results**: Same items regardless of concurrent changes
- **Efficient**: Index-based queries, no scanning
- **Opaque**: Implementation hidden from clients
- **Flexible**: Can change cursor format without breaking clients

## See Also

- [Strongly Typed IDs](./strongly-typed-ids.md) — type safety for IDs
- [Honest Functions](./honest-functions.md) — explicit error handling
- [Secret Types](./secret-types.md) — opaque values

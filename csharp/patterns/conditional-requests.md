# Conditional Requests (Type-Safe ETags and Preconditions)

> Checking ETags and If-Match headers with strings—use typed entity tags to prevent lost updates and enable optimistic concurrency.

## Problem

In REST APIs, concurrent updates can cause lost updates: User A reads a resource, User B reads the same resource, B updates it, then A updates it—A's update overwrites B's changes without A even knowing. ETags and conditional requests solve this, but string-based implementations are error-prone.

## Example

### ❌ Before

```csharp
[ApiController]
[Route("api/products")]
public class ProductsController : ControllerBase
{
    [HttpGet("{id}")]
    public IActionResult GetProduct(int id)
    {
        var product = _repository.GetById(id);
        
        // Manually compute ETag as string
        var etag = $"\"{product.Version}\"";
        Response.Headers.Add("ETag", etag);
        
        return Ok(product);
    }
    
    [HttpPut("{id}")]
    public IActionResult UpdateProduct(int id, [FromBody] ProductDto dto)
    {
        // String-based If-Match check
        if (Request.Headers.TryGetValue("If-Match", out var ifMatch))
        {
            var currentProduct = _repository.GetById(id);
            var currentETag = $"\"{currentProduct.Version}\"";
            
            // String comparison—easy to mess up format
            if (ifMatch != currentETag)
            {
                return StatusCode(412, "Precondition failed");
            }
        }
        
        // Update without version check—race condition!
        var product = _repository.GetById(id);
        product.Name = dto.Name;
        product.Price = dto.Price;
        _repository.Save(product);
        
        return Ok(product);
    }
}

// Lost update scenario:
// 1. User A: GET /products/1 → ETag: "v1"
// 2. User B: GET /products/1 → ETag: "v1"
// 3. User B: PUT /products/1 (If-Match: "v1") → Success, now version "v2"
// 4. User A: PUT /products/1 (If-Match: "v1") → Should fail but might not!
```

**Problems:**
- String-based ETag comparison
- Manual ETag generation and parsing
- Easy to forget If-Match check
- No compile-time guarantees
- Race conditions on updates
- Inconsistent ETag format

### ✅ After

```csharp
/// <summary>
/// Entity tag for optimistic concurrency control.
/// Represents the version of a resource.
/// </summary>
public readonly record struct EntityTag
{
    public string Value { get; }
    public bool IsWeak { get; }
    
    private EntityTag(string value, bool isWeak)
    {
        Value = value;
        IsWeak = isWeak;
    }
    
    /// <summary>
    /// Creates a strong ETag from version number.
    /// </summary>
    public static EntityTag Strong(long version) => 
        new($"{version}", isWeak: false);
    
    /// <summary>
    /// Creates a weak ETag (for approximate comparison).
    /// </summary>
    public static EntityTag Weak(string value) => 
        new(value, isWeak: true);
    
    /// <summary>
    /// Computes ETag from content hash.
    /// </summary>
    public static EntityTag FromContent(byte[] content)
    {
        var hash = System.Security.Cryptography.SHA256.HashData(content);
        var hashString = Convert.ToBase64String(hash);
        return new(hashString, isWeak: false);
    }
    
    /// <summary>
    /// Parses ETag from HTTP header value.
    /// Format: "value" for strong ETags, W/"value" for weak ETags.
    /// </summary>
    public static Result<EntityTag, string> Parse(string? headerValue)
    {
        if (string.IsNullOrWhiteSpace(headerValue))
            return Result<EntityTag, string>.Failure("ETag header is empty");
        
        var trimmed = headerValue.Trim();
        var isWeak = trimmed.StartsWith("W/", StringComparison.Ordinal);
        
        // Extract value: W/"value" -> "value" or "value" -> "value"
        var value = isWeak
            ? trimmed.Length > 4 ? trimmed.Substring(3, trimmed.Length - 4) : ""
            : trimmed.Length > 2 ? trimmed.Substring(1, trimmed.Length - 2) : "";
        
        if (string.IsNullOrWhiteSpace(value))
            return Result<EntityTag, string>.Failure("Invalid ETag format");
        
        return Result<EntityTag, string>.Success(new EntityTag(value, isWeak));
    }
    
    /// <summary>
    /// Formats ETag for HTTP header.
    /// </summary>
    public string ToHeaderValue() => 
        IsWeak ? $"W/\"{Value}\"" : $"\"{Value}\"";
    
    /// <summary>
    /// Checks if this ETag matches another (respecting weak/strong semantics).
    /// </summary>
    public bool Matches(EntityTag other, bool allowWeak = true)
    {
        if (!allowWeak && (IsWeak || other.IsWeak))
            return false;
        
        return Value == other.Value;
    }
}

/// <summary>
/// Precondition result for conditional requests.
/// </summary>
public abstract record PreconditionResult
{
    private PreconditionResult() { }
    
    public sealed record Satisfied : PreconditionResult;
    public sealed record Failed(string Reason) : PreconditionResult;
    public sealed record NotProvided : PreconditionResult;
}

/// <summary>
/// Versioned resource with ETag.
/// </summary>
public sealed record VersionedResource<T>
{
    public required T Data { get; init; }
    public required EntityTag ETag { get; init; }
    public required DateTime LastModified { get; init; }
}

/// <summary>
/// Request with conditional headers.
/// </summary>
public sealed record ConditionalRequest
{
    public Option<EntityTag> IfMatch { get; init; } = Option<EntityTag>.None;
    public Option<EntityTag> IfNoneMatch { get; init; } = Option<EntityTag>.None;
    public Option<DateTime> IfModifiedSince { get; init; } = Option<DateTime>.None;
    public Option<DateTime> IfUnmodifiedSince { get; init; } = Option<DateTime>.None;
    
    /// <summary>
    /// Evaluates preconditions against current resource state.
    /// </summary>
    public PreconditionResult Evaluate<T>(VersionedResource<T> resource)
    {
        // If-Match: only proceed if ETag matches (for updates)
        var ifMatchResult = IfMatch.Match(
            onSome: requestedETag =>
            {
                if (!requestedETag.Matches(resource.ETag, allowWeak: false))
                    return new PreconditionResult.Failed("If-Match precondition failed");
                return new PreconditionResult.Satisfied();
            },
            onNone: () => new PreconditionResult.NotProvided() as PreconditionResult
        );
        
        if (ifMatchResult is PreconditionResult.Failed)
            return ifMatchResult;
        
        // If-None-Match: only proceed if ETag doesn't match (for conditional GET)
        var ifNoneMatchResult = IfNoneMatch.Match(
            onSome: requestedETag =>
            {
                if (requestedETag.Matches(resource.ETag, allowWeak: true))
                    return new PreconditionResult.Failed("If-None-Match precondition failed");
                return new PreconditionResult.Satisfied();
            },
            onNone: () => new PreconditionResult.NotProvided() as PreconditionResult
        );
        
        if (ifNoneMatchResult is PreconditionResult.Failed)
            return ifNoneMatchResult;
        
        // If-Unmodified-Since: for updates
        var ifUnmodifiedResult = IfUnmodifiedSince.Match(
            onSome: since =>
            {
                if (resource.LastModified > since)
                    return new PreconditionResult.Failed("Resource modified since specified time");
                return new PreconditionResult.Satisfied();
            },
            onNone: () => new PreconditionResult.NotProvided() as PreconditionResult
        );
        
        if (ifUnmodifiedResult is PreconditionResult.Failed)
            return ifUnmodifiedResult;
        
        // If-Modified-Since: for conditional GETs
        var ifModifiedResult = IfModifiedSince.Match(
            onSome: since =>
            {
                if (resource.LastModified <= since)
                    return new PreconditionResult.Failed("Resource not modified");
                return new PreconditionResult.Satisfied();
            },
            onNone: () => new PreconditionResult.NotProvided() as PreconditionResult
        );
        
        return ifModifiedResult;
    }
}

// Repository returns versioned resources
public interface IProductRepository
{
    Task<VersionedResource<Product>> GetByIdAsync(ProductId id);
    Task<Result<VersionedResource<Product>, string>> UpdateAsync(
        ProductId id,
        EntityTag expectedVersion,
        Product updatedProduct);
}

public class ProductRepository : IProductRepository
{
    public async Task<Result<VersionedResource<Product>, string>> UpdateAsync(
        ProductId id,
        EntityTag expectedVersion,
        Product updatedProduct)
    {
        var current = await GetByIdAsync(id);
        
        // Check version at database level
        if (!expectedVersion.Matches(current.ETag, allowWeak: false))
        {
            return Result<VersionedResource<Product>, string>.Failure(
                $"Optimistic concurrency failure: expected {expectedVersion.ToHeaderValue()}, " +
                $"current is {current.ETag.ToHeaderValue()}");
        }
        
        // Update with new version
        updatedProduct.Version = current.Data.Version + 1;
        await _db.SaveAsync(updatedProduct);
        
        return Result<VersionedResource<Product>, string>.Success(
            new VersionedResource<Product>
            {
                Data = updatedProduct,
                ETag = EntityTag.Strong(updatedProduct.Version),
                LastModified = DateTime.UtcNow
            });
    }
}

[ApiController]
[Route("api/products")]
public class ProductsController : ControllerBase
{
    private readonly IProductRepository _repository;
    
    [HttpGet("{id}")]
    public async Task<IActionResult> GetProduct(int id)
    {
        var productId = ProductId.From(id);
        var resource = await _repository.GetByIdAsync(productId);
        
        // Parse conditional headers
        var conditionalRequest = ParseConditionalRequest();
        
        // Evaluate preconditions
        var precondition = conditionalRequest.Evaluate(resource);
        
        if (precondition is PreconditionResult.Failed failed)
        {
            // Resource hasn't changed—return 304 Not Modified
            if (failed.Reason.Contains("not modified"))
            {
                Response.Headers.Add("ETag", resource.ETag.ToHeaderValue());
                return StatusCode(304);
            }
        }
        
        // Add caching headers
        Response.Headers.Add("ETag", resource.ETag.ToHeaderValue());
        Response.Headers.Add("Last-Modified", resource.LastModified.ToString("R"));
        Response.Headers.Add("Cache-Control", "private, must-revalidate");
        
        return Ok(resource.Data);
    }
    
    [HttpPut("{id}")]
    public async Task<IActionResult> UpdateProduct(
        int id,
        [FromBody] UpdateProductDto dto)
    {
        var productId = ProductId.From(id);
        
        // Extract If-Match header
        var ifMatch = Request.Headers.TryGetValue("If-Match", out var ifMatchValue)
            ? EntityTag.Parse(ifMatchValue.ToString()).Match(
                onSuccess: etag => Option<EntityTag>.Some(etag),
                onFailure: _ => Option<EntityTag>.None)
            : Option<EntityTag>.None;
        
        // Require If-Match for updates (prevent lost updates)
        if (!ifMatch.HasValue)
        {
            return StatusCode(428, new
            {
                error = "precondition_required",
                message = "If-Match header is required for updates"
            });
        }
        
        var updatedProduct = new Product
        {
            Id = productId,
            Name = dto.Name,
            Price = dto.Price
        };
        
        var result = await _repository.UpdateAsync(
            productId,
            ifMatch.Value,
            updatedProduct);
        
        return result.Match(
            onSuccess: resource =>
            {
                Response.Headers.Add("ETag", resource.ETag.ToHeaderValue());
                return Ok(resource.Data);
            },
            onFailure: error => StatusCode(412, new
            {
                error = "precondition_failed",
                message = error
            })
        );
    }
    
    [HttpDelete("{id}")]
    public async Task<IActionResult> DeleteProduct(int id)
    {
        var productId = ProductId.From(id);
        
        // Extract If-Match
        var ifMatch = Request.Headers.TryGetValue("If-Match", out var ifMatchValue)
            ? EntityTag.Parse(ifMatchValue.ToString()).Match(
                onSuccess: etag => Option<EntityTag>.Some(etag),
                onFailure: _ => Option<EntityTag>.None)
            : Option<EntityTag>.None;
        
        if (!ifMatch.HasValue)
        {
            return StatusCode(428, "If-Match required for delete");
        }
        
        var result = await _repository.DeleteAsync(productId, ifMatch.Value);
        
        return result.Match(
            onSuccess: () => NoContent(),
            onFailure: error => StatusCode(412, error)
        );
    }
    
    private ConditionalRequest ParseConditionalRequest()
    {
        var ifMatch = Request.Headers.TryGetValue("If-Match", out var ifMatchValue)
            ? EntityTag.Parse(ifMatchValue.ToString()).ToOption()
            : Option<EntityTag>.None;
        
        var ifNoneMatch = Request.Headers.TryGetValue("If-None-Match", out var ifNoneMatchValue)
            ? EntityTag.Parse(ifNoneMatchValue.ToString()).ToOption()
            : Option<EntityTag>.None;
        
        var ifModifiedSince = Request.Headers.TryGetValue("If-Modified-Since", out var imsValue)
            ? DateTime.TryParse(imsValue.ToString(), out var ims) 
                ? Option<DateTime>.Some(ims) 
                : Option<DateTime>.None
            : Option<DateTime>.None;
        
        var ifUnmodifiedSince = Request.Headers.TryGetValue("If-Unmodified-Since", out var iusValue)
            ? DateTime.TryParse(iusValue.ToString(), out var ius) 
                ? Option<DateTime>.Some(ius) 
                : Option<DateTime>.None
            : Option<DateTime>.None;
        
        return new ConditionalRequest
        {
            IfMatch = ifMatch,
            IfNoneMatch = ifNoneMatch,
            IfModifiedSince = ifModifiedSince,
            IfUnmodifiedSince = ifUnmodifiedSince
        };
    }
}
```

## Content-Based ETags

```csharp
/// <summary>
/// Generates ETag from resource content (for immutable resources).
/// </summary>
public class ContentBasedETagGenerator
{
    public EntityTag GenerateETag<T>(T resource)
    {
        var json = JsonSerializer.Serialize(resource);
        var bytes = Encoding.UTF8.GetBytes(json);
        return EntityTag.FromContent(bytes);
    }
}

// For static content like images
[HttpGet("images/{filename}")]
public IActionResult GetImage(string filename)
{
    var imageBytes = _storage.ReadFile(filename);
    var etag = EntityTag.FromContent(imageBytes);
    
    // Check If-None-Match
    if (Request.Headers.TryGetValue("If-None-Match", out var ifNoneMatch))
    {
        var requestedETag = EntityTag.Parse(ifNoneMatch.ToString());
        
        if (requestedETag.IsSuccess && requestedETag.Value.Matches(etag, allowWeak: true))
        {
            return StatusCode(304);  // Not Modified
        }
    }
    
    Response.Headers.Add("ETag", etag.ToHeaderValue());
    Response.Headers.Add("Cache-Control", "public, max-age=31536000");
    
    return File(imageBytes, "image/jpeg");
}
```

## Testing

```csharp
public class ConditionalRequestTests
{
    [Fact]
    public async Task UpdateProduct_WithoutIfMatch_Returns428()
    {
        var response = await _client.PutAsync("/api/products/1", 
            new StringContent("{}"));
        
        Assert.Equal(HttpStatusCode.PreconditionRequired, response.StatusCode);
    }
    
    [Fact]
    public async Task UpdateProduct_WithWrongETag_Returns412()
    {
        var request = new HttpRequestMessage(HttpMethod.Put, "/api/products/1");
        request.Headers.Add("If-Match", "\"wrong-version\"");
        request.Content = new StringContent("{\"name\":\"New Name\"}");
        
        var response = await _client.SendAsync(request);
        
        Assert.Equal(HttpStatusCode.PreconditionFailed, response.StatusCode);
    }
    
    [Fact]
    public async Task GetProduct_WithIfNoneMatch_Returns304()
    {
        // First request—get ETag
        var response1 = await _client.GetAsync("/api/products/1");
        var etag = response1.Headers.ETag.Tag;
        
        // Second request with If-None-Match
        var request = new HttpRequestMessage(HttpMethod.Get, "/api/products/1");
        request.Headers.Add("If-None-Match", etag);
        
        var response2 = await _client.SendAsync(request);
        
        Assert.Equal(HttpStatusCode.NotModified, response2.StatusCode);
    }
    
    [Fact]
    public void EntityTag_StrongMatch_RequiresBothStrong()
    {
        var etag1 = EntityTag.Strong(1);
        var etag2 = EntityTag.Weak("1");
        
        // Strong comparison requires both strong
        Assert.False(etag1.Matches(etag2, allowWeak: false));
        
        // Weak comparison allows weak tags
        Assert.True(etag1.Matches(etag2, allowWeak: true));
    }
}
```

## Why It's a Problem

1. **Lost updates**: Concurrent modifications overwrite each other
2. **Manual checks**: Easy to forget precondition validation
3. **String comparison**: Error-prone ETag formatting
4. **No type safety**: ETags are just strings
5. **Inconsistent behavior**: Some endpoints check, others don't

## Symptoms

- Concurrent update bugs in production
- Manual ETag string formatting
- If-Match checks missing on some endpoints
- Lost updates when multiple users edit same resource
- No caching strategy

## Benefits

- **Prevents lost updates**: Optimistic concurrency enforced
- **Type safety**: Cannot create invalid ETags
- **Consistent behavior**: All updates require version check
- **Better caching**: Conditional GETs reduce bandwidth
- **Compile-time checks**: Repository API requires version

## See Also

- [Value Semantics](./value-semantics.md) — immutable resources
- [Versioned Endpoints](./versioned-endpoints.md) — API versioning
- [Honest Functions](./honest-functions.md) — explicit results
- [Strongly Typed IDs](./strongly-typed-ids.md) — resource IDs

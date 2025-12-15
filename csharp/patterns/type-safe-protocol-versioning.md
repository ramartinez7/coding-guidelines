# Type-Safe Protocol Versioning (API Protocol Evolution)

> API protocol versions tracked with integers or strings allow version mismatches—use typed protocol versions to enforce compatibility at compile time.

## Problem

API versioning schemes using version numbers in URLs or headers don't enforce protocol compatibility. Clients and servers can use incompatible protocol versions, leading to runtime serialization errors or silent data loss.

## Example

### ❌ Before

```csharp
public class ApiController
{
    [HttpGet("api/v1/users/{id}")]
    public IActionResult GetUserV1(int id)
    {
        // Returns V1 format
        return Ok(new { Id = id, Name = "John" });
    }
    
    [HttpGet("api/v2/users/{id}")]
    public IActionResult GetUserV2(int id)
    {
        // Returns V2 format—no type safety between versions
        return Ok(new { Id = id, FullName = "John Doe", Email = "john@example.com" });
    }
}
```

### ✅ After

```csharp
public interface IApiVersion { }
public interface IV1 : IApiVersion { }
public interface IV2 : IApiVersion { }
public interface IV3 : IApiVersion { }

public interface IProtocolHandler<TVersion, TRequest, TResponse> 
    where TVersion : IApiVersion
{
    Task<TResponse> HandleAsync(TRequest request);
}

public sealed record GetUserRequestV1(int UserId);
public sealed record GetUserResponseV1(int Id, string Name);

public sealed record GetUserRequestV2(string UserId);
public sealed record GetUserResponseV2(string Id, string FullName, string Email);

public sealed class GetUserHandlerV1 
    : IProtocolHandler<IV1, GetUserRequestV1, GetUserResponseV1>
{
    public async Task<GetUserResponseV1> HandleAsync(GetUserRequestV1 request)
    {
        var user = await _repository.GetByIdAsync(new UserId(request.UserId));
        return new GetUserResponseV1(user.Id.Value, user.Name.Value);
    }
}

public sealed class GetUserHandlerV2 
    : IProtocolHandler<IV2, GetUserRequestV2, GetUserResponseV2>
{
    public async Task<GetUserResponseV2> HandleAsync(GetUserRequestV2 request)
    {
        var user = await _repository.GetByIdAsync(UserId.Parse(request.UserId).Value);
        return new GetUserResponseV2(
            user.Id.Value.ToString(),
            user.Name.Value,
            user.Email.Value);
    }
}

// Type-safe version routing
public class VersionedApiController
{
    [HttpGet("api/v1/users/{id}")]
    public async Task<IActionResult> GetUserV1(
        [FromRoute] int id,
        [FromServices] IProtocolHandler<IV1, GetUserRequestV1, GetUserResponseV1> handler)
    {
        var request = new GetUserRequestV1(id);
        var response = await handler.HandleAsync(request);
        return Ok(response);
    }
    
    [HttpGet("api/v2/users/{id}")]
    public async Task<IActionResult> GetUserV2(
        [FromRoute] string id,
        [FromServices] IProtocolHandler<IV2, GetUserRequestV2, GetUserResponseV2> handler)
    {
        var request = new GetUserRequestV2(id);
        var response = await handler.HandleAsync(request);
        return Ok(response);
    }
}
```

## See Also

- [Versioned Endpoints](./versioned-endpoints.md)
- [Type-Safe Serialization Contracts](./type-safe-serialization-contracts.md)
- [Phantom Types](./phantom-types.md)

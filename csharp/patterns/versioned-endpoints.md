# Versioned Endpoints (Type-Safe API Versioning)

> API versioning with string paths and runtime route matching—use types to represent versions at compile time.

## Problem

API versioning typically relies on strings in URLs (`/v1/users`, `/v2/users`) or headers (`api-version: 2.0`). This makes it easy to accidentally call the wrong version, return incompatible responses, or forget to implement a version of an endpoint.

## Example

### ❌ Before

```csharp
[ApiController]
[Route("api/v{version:apiVersion}/users")]
public class UsersController : ControllerBase
{
    [HttpGet("{id}")]
    public IActionResult GetUser(string version, int id)
    {
        // Runtime string matching—easy to get wrong
        return version switch
        {
            "1" => GetUserV1(id),
            "2" => GetUserV2(id),
            _ => BadRequest("Unsupported API version")
        };
    }
    
    private IActionResult GetUserV1(int id)
    {
        var user = _repository.GetUser(id);
        return Ok(new
        {
            id = user.Id,
            name = user.Name
            // V1 response format
        });
    }
    
    private IActionResult GetUserV2(int id)
    {
        var user = _repository.GetUser(id);
        return Ok(new
        {
            userId = user.Id,  // Different field name in V2
            fullName = user.Name,
            email = user.Email  // New field in V2
        });
    }
}
```

**Problems:**
- String matching at runtime
- Easy to typo version strings
- No compile-time verification version exists
- Response types aren't tracked per version
- Unclear which versions exist

### ✅ After

```csharp
/// <summary>
/// Marker interface for all API versions.
/// </summary>
public interface IApiVersion { }

/// <summary>
/// API Version 1.
/// </summary>
public sealed record V1 : IApiVersion
{
    public static V1 Instance { get; } = new();
    private V1() { }
    public override string ToString() => "1";
}

/// <summary>
/// API Version 2.
/// </summary>
public sealed record V2 : IApiVersion
{
    public static V2 Instance { get; } = new();
    private V2() { }
    public override string ToString() => "2";
}

/// <summary>
/// Represents a versioned request for a specific API version.
/// </summary>
public sealed record VersionedRequest<TVersion> where TVersion : IApiVersion
{
    public TVersion Version { get; }
    public required HttpRequest HttpRequest { get; init; }
    
    public VersionedRequest(TVersion version)
    {
        Version = version;
    }
}

// Version-specific response types
public sealed record UserResponseV1
{
    public int Id { get; init; }
    public string Name { get; init; } = string.Empty;
}

public sealed record UserResponseV2
{
    public int UserId { get; init; }
    public string FullName { get; init; } = string.Empty;
    public string Email { get; init; } = string.Empty;
}

// Service layer with version-specific methods
public class UserService
{
    private readonly IUserRepository _repository;
    
    public UserService(IUserRepository repository)
    {
        _repository = repository;
    }
    
    // Compile-time guarantee this handles V1
    public UserResponseV1 GetUser(VersionedRequest<V1> request, int id)
    {
        var user = _repository.GetUser(id);
        return new UserResponseV1
        {
            Id = user.Id,
            Name = user.Name
        };
    }
    
    // Compile-time guarantee this handles V2
    public UserResponseV2 GetUser(VersionedRequest<V2> request, int id)
    {
        var user = _repository.GetUser(id);
        return new UserResponseV2
        {
            UserId = user.Id,
            FullName = user.Name,
            Email = user.Email.Value
        };
    }
}

// Controllers now specific to versions
[ApiController]
[Route("api/v1/users")]
public class UsersV1Controller : ControllerBase
{
    private readonly UserService _userService;
    
    public UsersV1Controller(UserService userService)
    {
        _userService = userService;
    }
    
    [HttpGet("{id}")]
    public IActionResult GetUser(int id)
    {
        var request = new VersionedRequest<V1>(V1.Instance) { HttpRequest = Request };
        var response = _userService.GetUser(request, id);
        return Ok(response);
    }
}

[ApiController]
[Route("api/v2/users")]
public class UsersV2Controller : ControllerBase
{
    private readonly UserService _userService;
    
    public UsersV2Controller(UserService userService)
    {
        _userService = userService;
    }
    
    [HttpGet("{id}")]
    public IActionResult GetUser(int id)
    {
        var request = new VersionedRequest<V2>(V2.Instance) { HttpRequest = Request };
        var response = _userService.GetUser(request, id);
        return Ok(response);
    }
}
```

## Deprecation Tracking

```csharp
/// <summary>
/// An API version with deprecation information.
/// </summary>
public abstract record ApiVersion : IApiVersion
{
    public abstract string VersionNumber { get; }
    public abstract bool IsDeprecated { get; }
    public abstract DateTime? SunsetDate { get; }
}

public sealed record V1 : ApiVersion
{
    public static V1 Instance { get; } = new();
    private V1() { }
    
    public override string VersionNumber => "1";
    public override bool IsDeprecated => true;
    public override DateTime? SunsetDate => new DateTime(2025, 12, 31);
}

public sealed record V2 : ApiVersion
{
    public static V2 Instance { get; } = new();
    private V2() { }
    
    public override string VersionNumber => "2";
    public override bool IsDeprecated => false;
    public override DateTime? SunsetDate => null;
}

public sealed record V3 : ApiVersion
{
    public static V3 Instance { get; } = new();
    private V3() { }
    
    public override string VersionNumber => "3";
    public override bool IsDeprecated => false;
    public override DateTime? SunsetDate => null;
}

// Middleware to add deprecation headers
public class DeprecationHeaderMiddleware
{
    private readonly RequestDelegate _next;
    
    public async Task InvokeAsync(HttpContext context)
    {
        await _next(context);
        
        // If request used deprecated version, add header
        if (context.Items.TryGetValue("ApiVersion", out var version) && version is ApiVersion apiVersion)
        {
            if (apiVersion.IsDeprecated)
            {
                context.Response.Headers.Add("Deprecation", "true");
                
                if (apiVersion.SunsetDate.HasValue)
                {
                    context.Response.Headers.Add("Sunset", 
                        apiVersion.SunsetDate.Value.ToString("R"));
                }
            }
        }
    }
}
```

## Version Negotiation

```csharp
/// <summary>
/// Resolves API version from request.
/// </summary>
public interface IVersionResolver
{
    Option<IApiVersion> ResolveVersion(HttpRequest request);
}

public class HeaderVersionResolver : IVersionResolver
{
    public Option<IApiVersion> ResolveVersion(HttpRequest request)
    {
        if (!request.Headers.TryGetValue("api-version", out var versionHeader))
            return Option<IApiVersion>.None;
        
        return versionHeader.ToString() switch
        {
            "1" => Option<IApiVersion>.Some(V1.Instance),
            "2" => Option<IApiVersion>.Some(V2.Instance),
            "3" => Option<IApiVersion>.Some(V3.Instance),
            _ => Option<IApiVersion>.None
        };
    }
}

public class UrlVersionResolver : IVersionResolver
{
    public Option<IApiVersion> ResolveVersion(HttpRequest request)
    {
        var path = request.Path.Value;
        
        if (path == null)
            return Option<IApiVersion>.None;
        
        if (path.Contains("/v1/"))
            return Option<IApiVersion>.Some(V1.Instance);
        if (path.Contains("/v2/"))
            return Option<IApiVersion>.Some(V2.Instance);
        if (path.Contains("/v3/"))
            return Option<IApiVersion>.Some(V3.Instance);
        
        return Option<IApiVersion>.None;
    }
}

// Composite resolver: try multiple strategies
public class CompositeVersionResolver : IVersionResolver
{
    private readonly IVersionResolver[] _resolvers;
    
    public CompositeVersionResolver(params IVersionResolver[] resolvers)
    {
        _resolvers = resolvers;
    }
    
    public Option<IApiVersion> ResolveVersion(HttpRequest request)
    {
        foreach (var resolver in _resolvers)
        {
            var result = resolver.ResolveVersion(request);
            if (result.HasValue)
                return result;
        }
        
        return Option<IApiVersion>.None;
    }
}
```

## Exhaustive Version Handling

```csharp
/// <summary>
/// Handler that must implement all API versions.
/// </summary>
public interface IVersionedHandler<TRequest, TResponse>
{
    TResponse HandleV1(TRequest request);
    TResponse HandleV2(TRequest request);
    TResponse HandleV3(TRequest request);
}

public class GetUserHandler : IVersionedHandler<GetUserRequest, IActionResult>
{
    private readonly IUserRepository _repository;
    
    public GetUserHandler(IUserRepository repository)
    {
        _repository = repository;
    }
    
    public IActionResult HandleV1(GetUserRequest request)
    {
        var user = _repository.GetUser(request.UserId);
        return new OkObjectResult(new UserResponseV1
        {
            Id = user.Id,
            Name = user.Name
        });
    }
    
    public IActionResult HandleV2(GetUserRequest request)
    {
        var user = _repository.GetUser(request.UserId);
        return new OkObjectResult(new UserResponseV2
        {
            UserId = user.Id,
            FullName = user.Name,
            Email = user.Email.Value
        });
    }
    
    public IActionResult HandleV3(GetUserRequest request)
    {
        var user = _repository.GetUser(request.UserId);
        return new OkObjectResult(new UserResponseV3
        {
            UserId = user.Id,
            FullName = user.Name,
            EmailAddress = user.Email.Value,
            PhoneNumber = user.PhoneNumber?.Value,
            CreatedAt = user.CreatedAt
        });
    }
}

// Dispatcher ensures version is handled
public class VersionedEndpoint
{
    private readonly GetUserHandler _handler;
    
    public IActionResult GetUser(IApiVersion version, GetUserRequest request)
    {
        return version switch
        {
            V1 => _handler.HandleV1(request),
            V2 => _handler.HandleV2(request),
            V3 => _handler.HandleV3(request),
            _ => throw new NotSupportedException($"API version {version} not supported")
        };
    }
}
```

## Version-Specific Validation

```csharp
public abstract record CreateUserCommand
{
    public required string Name { get; init; }
}

public sealed record CreateUserCommandV1 : CreateUserCommand
{
    // V1: Only name required
}

public sealed record CreateUserCommandV2 : CreateUserCommand
{
    // V2: Email now required
    public required string Email { get; init; }
}

public sealed record CreateUserCommandV3 : CreateUserCommand
{
    // V3: Email and phone required
    public required string Email { get; init; }
    public required string PhoneNumber { get; init; }
}

// Validators are version-specific
public class CreateUserCommandV1Validator : AbstractValidator<CreateUserCommandV1>
{
    public CreateUserCommandV1Validator()
    {
        RuleFor(x => x.Name).NotEmpty().MaximumLength(100);
    }
}

public class CreateUserCommandV2Validator : AbstractValidator<CreateUserCommandV2>
{
    public CreateUserCommandV2Validator()
    {
        RuleFor(x => x.Name).NotEmpty().MaximumLength(100);
        RuleFor(x => x.Email).NotEmpty().EmailAddress();
    }
}

public class CreateUserCommandV3Validator : AbstractValidator<CreateUserCommandV3>
{
    public CreateUserCommandV3Validator()
    {
        RuleFor(x => x.Name).NotEmpty().MaximumLength(100);
        RuleFor(x => x.Email).NotEmpty().EmailAddress();
        RuleFor(x => x.PhoneNumber).NotEmpty().Matches(@"^\+?\d{10,15}$");
    }
}
```

## Testing

```csharp
public class UserEndpointTests
{
    [Fact]
    public void GetUser_V1_ReturnsV1Response()
    {
        // Arrange
        var service = new UserService(_mockRepository.Object);
        var request = new VersionedRequest<V1>(V1.Instance);
        
        // Act
        var response = service.GetUser(request, 123);
        
        // Assert
        Assert.IsType<UserResponseV1>(response);
        Assert.Equal(123, response.Id);
    }
    
    [Fact]
    public void GetUser_V2_ReturnsV2Response()
    {
        // Arrange
        var service = new UserService(_mockRepository.Object);
        var request = new VersionedRequest<V2>(V2.Instance);
        
        // Act
        var response = service.GetUser(request, 123);
        
        // Assert
        Assert.IsType<UserResponseV2>(response);
        Assert.Equal(123, response.UserId);
        Assert.NotNull(response.Email);
    }
}
```

## Why It's a Problem

1. **Runtime errors**: Wrong version strings don't cause compile errors
2. **Unclear versions**: Hard to discover which versions exist
3. **Missing implementations**: Easy to forget to implement a version
4. **Inconsistent responses**: No type safety for version-specific responses
5. **Refactoring hazards**: Renaming fields in one version affects others

## Symptoms

- String constants for version numbers
- Switch statements on version strings
- Runtime errors for unsupported versions
- Comments listing supported versions
- Inconsistent response shapes across versions

## Benefits

- **Compile-time safety**: Cannot reference non-existent versions
- **Type-safe responses**: Each version has its own response type
- **Exhaustive handling**: Compiler enforces all versions handled
- **Self-documenting**: Version types make API surface clear
- **Refactoring-safe**: Changing one version doesn't affect others

## See Also

- [Enum to Class Hierarchy](./enum-to-class-hierarchy.md) — representing variants
- [Honest Functions](./honest-functions.md) — explicit signatures
- [DTO vs. Domain Boundary](./dto-domain-boundary.md) — separating contracts
- [Strongly Typed IDs](./strongly-typed-ids.md) — type safety

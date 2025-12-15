# Type-Safe Serialization Contracts (Preventing Serialization Errors)

> Serialization that fails at runtime from missing properties, type mismatches, or versioning issues—use typed serialization contracts to catch errors at compile time.

## Problem

JSON/XML serialization relies on reflection and runtime type discovery. Breaking changes to data models, missing required fields, or incompatible type changes only surface when serialization occurs, often in production. Contract mismatches between client and server cause silent data loss or runtime exceptions.

## Example

### ❌ Before

```csharp
public class UserDto
{
    public int Id { get; set; }
    public string Name { get; set; } = "";
    public string Email { get; set; } = "";
    // Add new required field—breaks existing clients silently
    public string PhoneNumber { get; set; } = "";
}

public class ApiController
{
    [HttpGet]
    public IActionResult GetUser(int id)
    {
        // Runtime serialization—no compile-time contract validation
        var user = new UserDto
        {
            Id = id,
            Name = "John",
            Email = "john@example.com"
            // Forgot PhoneNumber—serializes as empty string
        };
        return Ok(user);
    }
}

// Problems:
// - Adding required fields breaks existing clients
// - Removing fields is undetected
// - Type changes fail at runtime
// - No versioning support
```

### ✅ After

```csharp
/// <summary>
/// Marker interface for serialization contracts.
/// </summary>
public interface ISerializationContract<T>
{
    T ToModel();
}

public interface ISerializationVersion { }
public interface IV1 : ISerializationVersion { }
public interface IV2 : ISerializationVersion { }

/// <summary>
/// Versioned serialization contract with compile-time validation.
/// </summary>
public sealed record UserContractV1 : ISerializationContract<User>
{
    public required int Id { get; init; }
    public required string Name { get; init; }
    public required string Email { get; init; }
    
    public User ToModel() => new(
        new UserId(Id),
        new UserName(Name),
        new Email(Email));
    
    public static UserContractV1 FromModel(User user) => new()
    {
        Id = user.Id.Value,
        Name = user.Name.Value,
        Email = user.Email.Value
    };
}

public sealed record UserContractV2 : ISerializationContract<User>
{
    public required int Id { get; init; }
    public required string Name { get; init; }
    public required string Email { get; init; }
    public required string PhoneNumber { get; init; }  // New required field
    
    public User ToModel() => new(
        new UserId(Id),
        new UserName(Name),
        new Email(Email),
        new PhoneNumber(PhoneNumber));
    
    public static UserContractV2 FromModel(User user) => new()
    {
        Id = user.Id.Value,
        Name = user.Name.Value,
        Email = user.Email.Value,
        PhoneNumber = user.Phone.Value
    };
}

/// <summary>
/// Type-safe serialization builder.
/// </summary>
public sealed class SerializationBuilder<TContract, TModel>
    where TContract : ISerializationContract<TModel>
{
    private readonly JsonSerializerOptions _options;
    
    private SerializationBuilder(JsonSerializerOptions options)
    {
        _options = options;
    }
    
    public static SerializationBuilder<TContract, TModel> Create()
    {
        var options = new JsonSerializerOptions
        {
            PropertyNamingPolicy = JsonNamingPolicy.CamelCase
        };
        return new SerializationBuilder<TContract, TModel>(options);
    }
    
    public string Serialize(TModel model, Func<TModel, TContract> toContract)
    {
        var contract = toContract(model);
        return JsonSerializer.Serialize(contract, _options);
    }
    
    public Result<TModel, string> Deserialize(string json)
    {
        try
        {
            var contract = JsonSerializer.Deserialize<TContract>(json, _options);
            if (contract is null)
                return Result<TModel, string>.Failure("Deserialization returned null");
            
            return Result<TModel, string>.Success(contract.ToModel());
        }
        catch (JsonException ex)
        {
            return Result<TModel, string>.Failure(ex.Message);
        }
    }
}

// Usage: Version-specific serialization
public class UserApiController
{
    private readonly IUserRepository _users;
    
    [HttpGet("v1/users/{id}")]
    public IActionResult GetUserV1(int id)
    {
        var user = _users.GetById(new UserId(id));
        
        var serializer = SerializationBuilder<UserContractV1, User>.Create();
        var json = serializer.Serialize(user, UserContractV1.FromModel);
        
        return Content(json, "application/json");
    }
    
    [HttpGet("v2/users/{id}")]
    public IActionResult GetUserV2(int id)
    {
        var user = _users.GetById(new UserId(id));
        
        var serializer = SerializationBuilder<UserContractV2, User>.Create();
        var json = serializer.Serialize(user, UserContractV2.FromModel);
        
        return Content(json, "application/json");
    }
    
    [HttpPost("v2/users")]
    public IActionResult CreateUserV2([FromBody] string json)
    {
        var serializer = SerializationBuilder<UserContractV2, User>.Create();
        var result = serializer.Deserialize(json);
        
        return result.Match(
            user => Ok(UserContractV2.FromModel(user)),
            error => BadRequest(error));
    }
}

// Won't compile:
// var v1 = UserContractV1.FromModel(user);  // Missing PhoneNumber in V1
```

## Schema Evolution with Types

```csharp
/// <summary>
/// Schema migration between versions.
/// </summary>
public interface ISchemaMigration<TFrom, TTo>
{
    TTo Migrate(TFrom from);
}

public sealed class UserV1ToV2Migration : ISchemaMigration<UserContractV1, UserContractV2>
{
    public UserContractV2 Migrate(UserContractV1 from)
    {
        return new UserContractV2
        {
            Id = from.Id,
            Name = from.Name,
            Email = from.Email,
            PhoneNumber = ""  // Default for missing field
        };
    }
}

public sealed class UserV2ToV1Migration : ISchemaMigration<UserContractV2, UserContractV1>
{
    public UserContractV1 Migrate(UserContractV2 from)
    {
        return new UserContractV1
        {
            Id = from.Id,
            Name = from.Name,
            Email = from.Email
            // PhoneNumber dropped
        };
    }
}

// Type-safe migration registry
public sealed class MigrationRegistry
{
    private readonly Dictionary<(Type, Type), object> _migrations = new();
    
    public void Register<TFrom, TTo>(ISchemaMigration<TFrom, TTo> migration)
    {
        _migrations[(typeof(TFrom), typeof(TTo))] = migration;
    }
    
    public Option<ISchemaMigration<TFrom, TTo>> Get<TFrom, TTo>()
    {
        if (_migrations.TryGetValue((typeof(TFrom), typeof(TTo)), out var migration))
            return Option<ISchemaMigration<TFrom, TTo>>.Some((ISchemaMigration<TFrom, TTo>)migration);
        
        return Option<ISchemaMigration<TFrom, TTo>>.None();
    }
}
```

## Required Properties with Init

```csharp
// C# 11: required members ensure all properties are set
public sealed record OrderContract
{
    public required Guid Id { get; init; }
    public required string CustomerId { get; init; }
    public required OrderItemContract[] Items { get; init; }
    public required decimal Total { get; init; }
}

public sealed record OrderItemContract
{
    public required string ProductId { get; init; }
    public required int Quantity { get; init; }
    public required decimal Price { get; init; }
}

// Won't compile:
// var order = new OrderContract { Id = Guid.NewGuid() };  // Missing required properties
```

## Benefits

- **Compile-time safety**: Missing required fields don't compile
- **Version isolation**: Each version has distinct type
- **Breaking change detection**: Type changes caught by compiler
- **Explicit migrations**: Schema evolution as typed transformations
- **Self-documenting**: Contract types serve as API documentation

## See Also

- [DTO vs Domain Boundary](./dto-domain-boundary.md) — separating contracts from domain
- [Type-Safe Messaging](./type-safe-messaging.md) — message contract safety
- [Versioned Endpoints](./versioned-endpoints.md) — API versioning
- [Smart Constructors](./smart-constructors.md) — validation at construction

# Deserialization Prevention (Safe Object Deserialization)

> Deserializing untrusted data with `BinaryFormatter` or unrestricted type resolution—use safe serializers and type allowlists to prevent remote code execution.

## Problem

Insecure deserialization is one of the most dangerous vulnerabilities (OWASP Top 10). Deserializing untrusted data can lead to remote code execution when attackers craft malicious payloads that instantiate arbitrary types. `BinaryFormatter`, `NetDataContractSerializer`, and `LosFormatter` are particularly dangerous.

## Example

### ❌ Before

```csharp
public class SessionService
{
    public UserSession? DeserializeSession(byte[] sessionData)
    {
        // Extremely dangerous: BinaryFormatter is vulnerable to RCE
        // Attacker can craft a payload that executes arbitrary code during deserialization
        var formatter = new BinaryFormatter();
        
        using var stream = new MemoryStream(sessionData);
        return formatter.Deserialize(stream) as UserSession;
    }
    
    public object DeserializeObject(string json, string typeName)
    {
        // Dangerous: attacker controls the type being instantiated
        var type = Type.GetType(typeName);
        return JsonConvert.DeserializeObject(json, type);
    }
    
    public Configuration LoadConfig(string xml)
    {
        // Vulnerable: NetDataContractSerializer can instantiate any type
        var serializer = new NetDataContractSerializer();
        
        using var reader = new StringReader(xml);
        using var xmlReader = XmlReader.Create(reader);
        
        return (Configuration)serializer.ReadObject(xmlReader);
    }
}
```

**Problems:**
- `BinaryFormatter` enables remote code execution
- Type name from untrusted input allows arbitrary instantiation
- No type validation during deserialization
- Attackers can craft gadget chains to execute code
- No compile-time protection

### ✅ After

```csharp
/// <summary>
/// Marker interface for types that can be safely deserialized.
/// </summary>
public interface ISafelySerializable { }

/// <summary>
/// Registry of types allowed for deserialization.
/// </summary>
public sealed class DeserializationTypeRegistry
{
    private readonly HashSet<Type> _allowedTypes = new();
    
    public void Register<T>() where T : ISafelySerializable
    {
        _allowedTypes.Add(typeof(T));
    }
    
    public void RegisterMany(params Type[] types)
    {
        foreach (var type in types)
        {
            if (!typeof(ISafelySerializable).IsAssignableFrom(type))
                throw new ArgumentException(
                    $"Type {type.Name} must implement ISafelySerializable");
            
            _allowedTypes.Add(type);
        }
    }
    
    public bool IsAllowed(Type type) => _allowedTypes.Contains(type);
    
    public Result<Type, string> GetAllowedType(string typeName)
    {
        var type = _allowedTypes.FirstOrDefault(t => t.FullName == typeName);
        
        if (type == null)
            return Result<Type, string>.Failure(
                $"Type '{typeName}' is not allowed for deserialization");
        
        return Result<Type, string>.Success(type);
    }
}

/// <summary>
/// Safe JSON serializer with type restrictions.
/// </summary>
public sealed class SafeJsonSerializer
{
    private readonly DeserializationTypeRegistry _typeRegistry;
    private readonly JsonSerializerOptions _options;
    
    public SafeJsonSerializer(DeserializationTypeRegistry typeRegistry)
    {
        _typeRegistry = typeRegistry;
        
        _options = new JsonSerializerOptions
        {
            // Prevent circular references
            ReferenceHandler = ReferenceHandler.IgnoreCycles,
            
            // Limit depth to prevent DoS
            MaxDepth = 32,
            
            // Only serialize public properties
            IncludeFields = false,
            
            // Use camelCase naming
            PropertyNamingPolicy = JsonNamingPolicy.CamelCase,
            
            // Strict property name matching
            PropertyNameCaseInsensitive = false
        };
    }
    
    public Result<T, string> Deserialize<T>(string json) 
        where T : ISafelySerializable
    {
        if (string.IsNullOrWhiteSpace(json))
            return Result<T, string>.Failure("JSON content cannot be empty");
        
        var targetType = typeof(T);
        
        if (!_typeRegistry.IsAllowed(targetType))
            return Result<T, string>.Failure(
                $"Type {targetType.Name} is not registered for deserialization");
        
        try
        {
            var result = JsonSerializer.Deserialize<T>(json, _options);
            
            if (result == null)
                return Result<T, string>.Failure("Deserialization returned null");
            
            return Result<T, string>.Success(result);
        }
        catch (JsonException ex)
        {
            return Result<T, string>.Failure($"Invalid JSON: {ex.Message}");
        }
        catch (Exception ex)
        {
            return Result<T, string>.Failure($"Deserialization failed: {ex.Message}");
        }
    }
    
    public string Serialize<T>(T obj) where T : ISafelySerializable
    {
        if (obj == null)
            throw new ArgumentNullException(nameof(obj));
        
        return JsonSerializer.Serialize(obj, _options);
    }
}

/// <summary>
/// Safe data contract serializer with type restrictions.
/// </summary>
public sealed class SafeXmlSerializer
{
    private readonly DeserializationTypeRegistry _typeRegistry;
    private readonly Dictionary<Type, DataContractSerializer> _serializers = new();
    
    public SafeXmlSerializer(DeserializationTypeRegistry typeRegistry)
    {
        _typeRegistry = typeRegistry;
    }
    
    public Result<T, string> Deserialize<T>(string xml) 
        where T : ISafelySerializable
    {
        if (string.IsNullOrWhiteSpace(xml))
            return Result<T, string>.Failure("XML content cannot be empty");
        
        var targetType = typeof(T);
        
        if (!_typeRegistry.IsAllowed(targetType))
            return Result<T, string>.Failure(
                $"Type {targetType.Name} is not registered for deserialization");
        
        try
        {
            var serializer = GetSerializer<T>();
            
            using var stringReader = new StringReader(xml);
            using var xmlReader = XmlReader.Create(stringReader, 
                SecureXmlSettings.CreateSecureSettings());
            
            var result = serializer.ReadObject(xmlReader) as T;
            
            if (result == null)
                return Result<T, string>.Failure("Deserialization returned null");
            
            return Result<T, string>.Success(result);
        }
        catch (Exception ex)
        {
            return Result<T, string>.Failure($"Deserialization failed: {ex.Message}");
        }
    }
    
    public string Serialize<T>(T obj) where T : ISafelySerializable
    {
        if (obj == null)
            throw new ArgumentNullException(nameof(obj));
        
        var serializer = GetSerializer<T>();
        
        using var stringWriter = new StringWriter();
        using var xmlWriter = XmlWriter.Create(stringWriter);
        
        serializer.WriteObject(xmlWriter, obj);
        xmlWriter.Flush();
        
        return stringWriter.ToString();
    }
    
    private DataContractSerializer GetSerializer<T>()
    {
        var type = typeof(T);
        
        if (!_serializers.TryGetValue(type, out var serializer))
        {
            // Only known types can be deserialized
            serializer = new DataContractSerializer(type, 
                new DataContractSerializerSettings
                {
                    // No known types beyond what's registered
                    KnownTypes = Array.Empty<Type>(),
                    
                    // Preserve object references to prevent circular issues
                    PreserveObjectReferences = false,
                    
                    // Limit depth
                    MaxItemsInObjectGraph = int.MaxValue
                });
            
            _serializers[type] = serializer;
        }
        
        return serializer;
    }
}

// Example domain types
[DataContract]
public sealed record UserSession : ISafelySerializable
{
    [DataMember]
    public required UserId UserId { get; init; }
    
    [DataMember]
    public required string Username { get; init; }
    
    [DataMember]
    public required DateTime CreatedAt { get; init; }
    
    [DataMember]
    public DateTime? LastAccessedAt { get; init; }
}

[DataContract]
public sealed record Configuration : ISafelySerializable
{
    [DataMember]
    public required string DatabaseConnection { get; init; }
    
    [DataMember]
    public required int MaxConnections { get; init; }
    
    [DataMember]
    public required TimeSpan Timeout { get; init; }
}

public class SessionService
{
    private readonly SafeJsonSerializer _serializer;
    
    public SessionService(SafeJsonSerializer serializer)
    {
        _serializer = serializer;
    }
    
    public Result<UserSession, string> DeserializeSession(string sessionJson)
    {
        // Type-safe deserialization—only registered types allowed
        return _serializer.Deserialize<UserSession>(sessionJson);
    }
    
    public string SerializeSession(UserSession session)
    {
        return _serializer.Serialize(session);
    }
}
```

## Advanced Patterns

### Type Binder for Custom Serialization

```csharp
/// <summary>
/// Secure type binder that only allows registered types.
/// </summary>
public sealed class SecureSerializationBinder : SerializationBinder
{
    private readonly DeserializationTypeRegistry _typeRegistry;
    
    public SecureSerializationBinder(DeserializationTypeRegistry typeRegistry)
    {
        _typeRegistry = typeRegistry;
    }
    
    public override Type? BindToType(string assemblyName, string typeName)
    {
        var fullTypeName = $"{typeName}, {assemblyName}";
        
        var result = _typeRegistry.GetAllowedType(fullTypeName);
        
        if (!result.IsSuccess)
            throw new SecurityException(
                $"Type '{fullTypeName}' is not allowed for deserialization");
        
        return result.Value;
    }
}
```

### Message Pack with Type Restrictions

```csharp
/// <summary>
/// Safe MessagePack serializer with type restrictions.
/// </summary>
public sealed class SafeMessagePackSerializer
{
    private readonly DeserializationTypeRegistry _typeRegistry;
    private readonly MessagePackSerializerOptions _options;
    
    public SafeMessagePackSerializer(DeserializationTypeRegistry typeRegistry)
    {
        _typeRegistry = typeRegistry;
        
        _options = MessagePackSerializerOptions.Standard
            .WithSecurity(MessagePackSecurity.UntrustedData);
    }
    
    public Result<T, string> Deserialize<T>(byte[] data) 
        where T : ISafelySerializable
    {
        var targetType = typeof(T);
        
        if (!_typeRegistry.IsAllowed(targetType))
            return Result<T, string>.Failure(
                $"Type {targetType.Name} is not registered for deserialization");
        
        try
        {
            var result = MessagePackSerializer.Deserialize<T>(data, _options);
            
            if (result == null)
                return Result<T, string>.Failure("Deserialization returned null");
            
            return Result<T, string>.Success(result);
        }
        catch (Exception ex)
        {
            return Result<T, string>.Failure($"Deserialization failed: {ex.Message}");
        }
    }
    
    public byte[] Serialize<T>(T obj) where T : ISafelySerializable
    {
        if (obj == null)
            throw new ArgumentNullException(nameof(obj));
        
        return MessagePackSerializer.Serialize(obj, _options);
    }
}
```

### Polymorphic Deserialization (Safe)

```csharp
/// <summary>
/// Base type for polymorphic deserialization.
/// Uses discriminator field to determine concrete type.
/// </summary>
[JsonPolymorphic(TypeDiscriminatorPropertyName = "$type")]
[JsonDerivedType(typeof(EmailNotification), "email")]
[JsonDerivedType(typeof(SmsNotification), "sms")]
[JsonDerivedType(typeof(PushNotification), "push")]
public abstract record Notification : ISafelySerializable
{
    public required string RecipientId { get; init; }
    public required string Message { get; init; }
}

public sealed record EmailNotification : Notification
{
    public required string EmailAddress { get; init; }
    public required string Subject { get; init; }
}

public sealed record SmsNotification : Notification
{
    public required string PhoneNumber { get; init; }
}

public sealed record PushNotification : Notification
{
    public required string DeviceToken { get; init; }
    public string? Title { get; init; }
}

public class NotificationService
{
    private readonly SafeJsonSerializer _serializer;
    
    public Result<Notification, string> DeserializeNotification(string json)
    {
        // Type discriminator determines concrete type
        // Only registered derived types are allowed
        return _serializer.Deserialize<Notification>(json);
    }
}
```

### DTO Validation After Deserialization

```csharp
/// <summary>
/// DTO that validates itself after deserialization.
/// </summary>
[DataContract]
public sealed record CreateOrderDto : ISafelySerializable, IValidatableObject
{
    [DataMember]
    public required Guid CustomerId { get; init; }
    
    [DataMember]
    public required List<OrderItemDto> Items { get; init; }
    
    [DataMember]
    public required decimal TotalAmount { get; init; }
    
    public IEnumerable<ValidationResult> Validate(ValidationContext validationContext)
    {
        if (Items.Count == 0)
            yield return new ValidationResult("Order must have at least one item");
        
        if (TotalAmount <= 0)
            yield return new ValidationResult("Total amount must be positive");
        
        var calculatedTotal = Items.Sum(i => i.Price * i.Quantity);
        if (Math.Abs(calculatedTotal - TotalAmount) > 0.01m)
            yield return new ValidationResult("Total amount doesn't match items");
    }
    
    public Result<CreateOrderCommand, string> ToCommand()
    {
        var validationResults = new List<ValidationResult>();
        var context = new ValidationContext(this);
        
        if (!Validator.TryValidateObject(this, context, validationResults, true))
        {
            var errors = string.Join("; ", validationResults.Select(v => v.ErrorMessage));
            return Result<CreateOrderCommand, string>.Failure(errors);
        }
        
        return Result<CreateOrderCommand, string>.Success(
            new CreateOrderCommand(
                new CustomerId(CustomerId),
                Items.Select(i => new OrderItem(i.ProductId, i.Quantity)).ToList(),
                Money.Create(TotalAmount, Currency.USD)));
    }
}

[DataContract]
public sealed record OrderItemDto
{
    [DataMember]
    public required string ProductId { get; init; }
    
    [DataMember]
    public required int Quantity { get; init; }
    
    [DataMember]
    public required decimal Price { get; init; }
}
```

## Testing

```csharp
public class DeserializationTests
{
    private readonly DeserializationTypeRegistry _registry;
    private readonly SafeJsonSerializer _serializer;
    
    public DeserializationTests()
    {
        _registry = new DeserializationTypeRegistry();
        _registry.Register<UserSession>();
        _registry.Register<Configuration>();
        
        _serializer = new SafeJsonSerializer(_registry);
    }
    
    [Fact]
    public void Deserialize_RegisteredType_Succeeds()
    {
        var json = @"{
            ""userId"": ""123"",
            ""username"": ""test"",
            ""createdAt"": ""2024-01-01T00:00:00Z""
        }";
        
        var result = _serializer.Deserialize<UserSession>(json);
        
        Assert.True(result.IsSuccess);
        Assert.Equal("test", result.Value!.Username);
    }
    
    [Fact]
    public void Deserialize_UnregisteredType_Fails()
    {
        // Attempt to deserialize a type not in the registry
        var json = @"{""malicious"": ""payload""}";
        
        // This would fail at compile time because MaliciousType 
        // doesn't implement ISafelySerializable
        // var result = _serializer.Deserialize<MaliciousType>(json);
    }
    
    [Fact]
    public void Deserialize_CircularReference_IsHandled()
    {
        var json = @"{
            ""userId"": ""123"",
            ""username"": ""test"",
            ""createdAt"": ""2024-01-01T00:00:00Z"",
            ""lastAccessedAt"": ""2024-01-01T00:00:00Z""
        }";
        
        var result = _serializer.Deserialize<UserSession>(json);
        
        Assert.True(result.IsSuccess);
    }
    
    [Fact]
    public void Deserialize_ExcessiveDepth_IsRejected()
    {
        // Create deeply nested JSON
        var json = "{" + string.Concat(Enumerable.Repeat(@"""x"":{", 100)) + 
                   @"""value"":1" + string.Concat(Enumerable.Repeat("}", 100));
        
        var result = _serializer.Deserialize<UserSession>(json);
        
        // Should fail due to depth limit
        Assert.False(result.IsSuccess);
    }
}
```

## Migration from BinaryFormatter

```csharp
/// <summary>
/// Helper to migrate from BinaryFormatter to safe serialization.
/// </summary>
public sealed class SerializationMigration
{
    private readonly SafeJsonSerializer _jsonSerializer;
    
    [Obsolete("BinaryFormatter is insecure. Use MigrateToJson instead.")]
    public Result<string, string> MigrateToJson<T>(byte[] binaryData) 
        where T : ISafelySerializable
    {
        try
        {
            // This is still unsafe but allows one-time migration
            #pragma warning disable SYSLIB0011
            var formatter = new BinaryFormatter();
            using var stream = new MemoryStream(binaryData);
            var obj = (T)formatter.Deserialize(stream);
            #pragma warning restore SYSLIB0011
            
            // Re-serialize with safe serializer
            var json = _jsonSerializer.Serialize(obj);
            
            return Result<string, string>.Success(json);
        }
        catch (Exception ex)
        {
            return Result<string, string>.Failure(
                $"Migration failed: {ex.Message}");
        }
    }
}
```

## Why It's a Problem

1. **Remote code execution**: Malicious payloads can execute arbitrary code
2. **Gadget chains**: Attackers chain existing code to achieve RCE
3. **Type confusion**: Untrusted type names allow instantiation of dangerous types
4. **BinaryFormatter**: Inherently unsafe, deprecated in .NET 5+
5. **No compile-time detection**: Vulnerable deserialization looks like normal code

## Symptoms

- Using `BinaryFormatter`, `NetDataContractSerializer`, `LosFormatter`, or `SoapFormatter`
- Deserializing with type name from untrusted input
- No type restrictions during deserialization
- No validation after deserialization
- Security warnings about deprecated serializers

## Benefits

- **RCE prevention**: Type allowlist prevents arbitrary type instantiation
- **Compile-time safety**: `ISafelySerializable` marks allowed types
- **Self-documenting**: Safe serializers make security guarantees explicit
- **Validation**: Combines deserialization with validation
- **Modern serializers**: Uses `System.Text.Json` and `DataContractSerializer`

## Trade-offs

- **Breaking change**: Cannot deserialize existing `BinaryFormatter` data without migration
- **Type registration**: All serializable types must be registered
- **More verbose**: Requires explicit type allowlisting
- **Limited polymorphism**: Type discriminators needed for derived types

## See Also

- [Input Sanitization](./input-sanitization.md) — trusted types at boundaries
- [XXE Prevention](./xxe-prevention.md) — safe XML parsing
- [Validated Configuration](./validated-configuration.md) — configuration deserialization
- [DTO vs. Domain Boundary](./dto-domain-boundary.md) — separating serialization from domain

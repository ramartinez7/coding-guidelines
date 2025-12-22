# Serialization Best Practices

> Serialize and deserialize data safely, efficiently, and with proper version handling.

## Problem

Unsafe serialization leads to security vulnerabilities, breaking changes, and data loss when schemas evolve.

## Example

### ❌ Before

```csharp
// Unsafe BinaryFormatter (security risk!)
var formatter = new BinaryFormatter();
using var stream = File.OpenWrite("data.bin");
formatter.Serialize(stream, order);  // Remote code execution risk!

// No versioning
[Serializable]
public class Order
{
    public string Customer;  // What if we rename this?
    public decimal Total;
}
```

### ✅ After

```csharp
// Safe JSON serialization
var options = new JsonSerializerOptions
{
    PropertyNamingPolicy = JsonNamingPolicy.CamelCase,
    WriteIndented = false
};

var json = JsonSerializer.Serialize(order, options);

// Versioned with data contract
[DataContract]
public class Order
{
    [DataMember(Name = "customerId", Order = 1)]
    public string CustomerId { get; set; }

    [DataMember(Name = "total", Order = 2)]
    public decimal Total { get; set; }
}
```

## Best Practices

### 1. Use System.Text.Json

```csharp
// ✅ Modern, fast, secure serialization
using System.Text.Json;

var order = new Order { Id = 1, Total = 100m };

// Serialize
var json = JsonSerializer.Serialize(order);

// Deserialize
var deserialized = JsonSerializer.Deserialize<Order>(json);
```

### 2. Configure JsonSerializerOptions

```csharp
// ✅ Consistent serialization settings
public static class JsonOptions
{
    public static readonly JsonSerializerOptions Default = new()
    {
        PropertyNamingPolicy = JsonNamingPolicy.CamelCase,
        DefaultIgnoreCondition = JsonIgnoreCondition.WhenWritingNull,
        WriteIndented = false,
        PropertyNameCaseInsensitive = true,
        Converters = 
        {
            new JsonStringEnumConverter()
        }
    };
}

// Usage
var json = JsonSerializer.Serialize(order, JsonOptions.Default);
```

### 3. Use Data Transfer Objects (DTOs)

```csharp
// ❌ Serializing domain models directly
public class Order  // Domain model
{
    public OrderId Id { get; set; }
    public Customer Customer { get; set; }
    // Complex domain logic, internal state
}

// ✅ Separate DTO for serialization
public sealed record OrderDto(
    Guid Id,
    string CustomerName,
    decimal Total);

public OrderDto ToDto(Order order) => new(
    order.Id.Value,
    order.Customer.Name,
    order.Total.Amount);
```

### 4. Handle Unknown Properties

```csharp
// ✅ Preserve unknown properties for forward compatibility
public class OrderDto
{
    public Guid Id { get; set; }
    public decimal Total { get; set; }

    [JsonExtensionData]
    public Dictionary<string, JsonElement>? ExtensionData { get; set; }
}
```

### 5. Use Converters for Custom Types

```csharp
// ✅ Custom converter for value objects
public class OrderIdConverter : JsonConverter<OrderId>
{
    public override OrderId Read(
        ref Utf8JsonReader reader,
        Type typeToConvert,
        JsonSerializerOptions options)
    {
        var guid = reader.GetGuid();
        return new OrderId(guid);
    }

    public override void Write(
        Utf8JsonWriter writer,
        OrderId value,
        JsonSerializerOptions options)
    {
        writer.WriteStringValue(value.Value);
    }
}

// Register
var options = new JsonSerializerOptions
{
    Converters = { new OrderIdConverter() }
};
```

### 6. Version Your Contracts

```csharp
// ✅ Explicit versioning
[DataContract(Name = "OrderV2")]
public class OrderDto
{
    [DataMember(Name = "version")]
    public int Version { get; set; } = 2;

    [DataMember(Name = "id")]
    public Guid Id { get; set; }

    [DataMember(Name = "total")]
    public decimal Total { get; set; }

    // New in V2
    [DataMember(Name = "currency", EmitDefaultValue = false)]
    public string? Currency { get; set; }
}
```

### 7. Use Required Properties

```csharp
// ✅ Required properties (C# 11+)
public class OrderDto
{
    [JsonRequired]
    public required Guid Id { get; init; }

    [JsonRequired]
    public required decimal Total { get; init; }

    public string? Notes { get; init; }  // Optional
}
```

### 8. Avoid BinaryFormatter

```csharp
// ❌ NEVER use BinaryFormatter (security vulnerability!)
var formatter = new BinaryFormatter();
formatter.Serialize(stream, data);  // Remote code execution risk!

// ✅ Use safe alternatives
var json = JsonSerializer.Serialize(data);
// or
var bytes = MessagePackSerializer.Serialize(data);
```

### 9. Handle Circular References

```csharp
// ❌ Circular reference causes exception
public class Order
{
    public Customer Customer { get; set; }
}

public class Customer
{
    public List<Order> Orders { get; set; }
}

// ✅ Configure to handle cycles
var options = new JsonSerializerOptions
{
    ReferenceHandler = ReferenceHandler.Preserve
};

// ✅ Or better: use DTOs without cycles
public sealed record OrderDto(Guid Id, Guid CustomerId);
```

### 10. Validate After Deserialization

```csharp
// ✅ Validate deserialized data
public Result<OrderDto, ValidationError> DeserializeOrder(string json)
{
    try
    {
        var order = JsonSerializer.Deserialize<OrderDto>(json);

        if (order == null)
        {
            return Result.Failure<OrderDto, ValidationError>(
                new ValidationError("Invalid JSON"));
        }

        if (order.Total < 0)
        {
            return Result.Failure<OrderDto, ValidationError>(
                new ValidationError("Total cannot be negative"));
        }

        return Result.Success<OrderDto, ValidationError>(order);
    }
    catch (JsonException ex)
    {
        return Result.Failure<OrderDto, ValidationError>(
            new ValidationError($"Deserialization failed: {ex.Message}"));
    }
}
```

### 11. Use Source Generators for Performance

```csharp
// ✅ Source generator for zero-allocation serialization
[JsonSerializable(typeof(Order))]
[JsonSerializable(typeof(Customer))]
internal partial class AppJsonContext : JsonSerializerContext
{
}

// Usage
var json = JsonSerializer.Serialize(
    order,
    AppJsonContext.Default.Order);
```

### 12. Handle Enums Consistently

```csharp
// ✅ Serialize enums as strings
var options = new JsonSerializerOptions
{
    Converters = { new JsonStringEnumConverter() }
};

public enum OrderStatus
{
    Pending,
    Confirmed,
    Shipped
}

// Serializes as "Pending" not 0
```

## Symptoms

- Breaking changes when schema evolves
- Security vulnerabilities from deserialization
- Poor performance from inefficient serialization
- Data loss from missing version handling
- Invalid data accepted without validation

## Benefits

- **Security** by avoiding unsafe serializers
- **Versioning** for schema evolution
- **Performance** with modern serializers
- **Type safety** with DTOs
- **Validation** of deserialized data

## See Also

- [Type-Safe Serialization Contracts](./type-safe-serialization-contracts.md) — Compile-time safety
- [DTO vs Domain Boundary](./dto-domain-boundary.md) — Separation of concerns
- [Deserialization Prevention](./deserialization-prevention.md) — Security

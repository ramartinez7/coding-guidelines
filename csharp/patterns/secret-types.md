# Secret Types (Preventing Accidental Exposure)

> Secrets logged or serialized accidentally—use dedicated types that prevent rendering in logs, exceptions, and JSON.

## Problem

Sensitive data (API keys, passwords, tokens) are stored as plain strings, making them easy to accidentally expose in logs, exception messages, database dumps, or API responses. There's no compile-time protection against leaking secrets.

## Example

### ❌ Before

```csharp
public class PaymentService
{
    public void ProcessPayment(string apiKey, string cardNumber)
    {
        try
        {
            _logger.LogInformation($"Processing payment with API key: {apiKey}");  // 💥 Logged!
            
            var result = _paymentGateway.Charge(cardNumber, apiKey);
            
            if (!result.Success)
            {
                throw new Exception($"Payment failed for card {cardNumber}");  // 💥 In exception!
            }
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, $"Failed with API key {apiKey}");  // 💥 Logged again!
            throw;
        }
    }
}

public class ApiController
{
    public IActionResult GetStatus(string apiKey)
    {
        var status = _service.GetStatus(apiKey);
        
        // Accidentally returned in JSON response
        return Ok(new { ApiKey = apiKey, Status = status });  // 💥 Exposed!
    }
}

// Even ToString() leaks secrets
var config = new Config { ApiKey = "sk_live_abc123" };
Console.WriteLine(config);  // 💥 "{ ApiKey = sk_live_abc123 }" printed
```

**Problems:**
- Secrets logged in plain text
- Secrets in exception messages
- Secrets serialized to JSON
- `ToString()` exposes secrets
- No compile-time protection

### ✅ After

```csharp
/// <summary>
/// A secret value that cannot be accidentally exposed through logging, 
/// serialization, or string conversion.
/// </summary>
public sealed class Secret<T>
{
    private readonly T _value;
    
    private Secret(T value) => _value = value;
    
    public static Secret<T> Create(T value) => new(value);
    
    /// <summary>
    /// Explicitly access the secret value.
    /// Use of this method should be auditable.
    /// </summary>
    public T Expose() => _value;
    
    /// <summary>
    /// Use the secret without exposing it.
    /// </summary>
    public TResult Use<TResult>(Func<T, TResult> func) => func(_value);
    
    // Prevents accidental logging
    public override string ToString() => "[REDACTED]";
    
    // Prevents accidental equality comparisons that might leak timing info
    public override bool Equals(object? obj) => false;
    public override int GetHashCode() => 0;
}

// Specific secret types
public sealed record ApiKey(Secret<string> Value)
{
    public static ApiKey Create(string key) => new(Secret<string>.Create(key));
    public override string ToString() => "[REDACTED API KEY]";
}

public sealed record CreditCardNumber(Secret<string> Value)
{
    private CreditCardNumber(Secret<string> value) : this() => Value = value;
    
    public static Result<CreditCardNumber, string> Create(string cardNumber)
    {
        if (string.IsNullOrWhiteSpace(cardNumber))
            return Result<CreditCardNumber, string>.Failure("Card number required");
            
        // Remove spaces and dashes
        var cleaned = cardNumber.Replace(" ", "").Replace("-", "");
        
        if (!Regex.IsMatch(cleaned, @"^\d{13,19}$"))
            return Result<CreditCardNumber, string>.Failure("Invalid card number format");
            
        return Result<CreditCardNumber, string>.Success(
            new CreditCardNumber(Secret<string>.Create(cleaned)));
    }
    
    // Show only last 4 digits for display
    public string MaskedDisplay => Value.Use(v => $"****-****-****-{v[^4..]}");
    
    public override string ToString() => MaskedDisplay;
}

public sealed record DatabasePassword(Secret<string> Value)
{
    public static DatabasePassword Create(string password) => 
        new(Secret<string>.Create(password));
        
    public override string ToString() => "[REDACTED PASSWORD]";
}

public class PaymentService
{
    public void ProcessPayment(ApiKey apiKey, CreditCardNumber cardNumber)
    {
        // Logging doesn't expose secrets
        _logger.LogInformation($"Processing payment with API key: {apiKey}");  // Logs "[REDACTED API KEY]"
        
        try
        {
            // Explicitly expose only when necessary
            var result = apiKey.Value.Use(key => 
                cardNumber.Value.Use(card => 
                    _paymentGateway.Charge(card, key)));
            
            if (!result.Success)
            {
                // Safe to include cardNumber—it's masked
                throw new Exception($"Payment failed for card {cardNumber}");  // Shows "****-****-****-1234"
            }
        }
        catch (Exception ex)
        {
            // No secret in log even if we try
            _logger.LogError(ex, $"Failed with API key {apiKey}");  // Logs "[REDACTED API KEY]"
            throw;
        }
    }
}

public class ApiController
{
    public IActionResult GetStatus(ApiKey apiKey)
    {
        var status = _service.GetStatus(apiKey);
        
        // JSON serialization doesn't expose the secret
        return Ok(new { ApiKey = apiKey.ToString(), Status = status });  // Returns "[REDACTED API KEY]"
    }
}
```

## JSON Serialization

Prevent secrets from appearing in JSON:

```csharp
public class SecretJsonConverter<T> : JsonConverter<Secret<T>>
{
    public override Secret<T> Read(ref Utf8JsonReader reader, Type typeToConvert, JsonSerializerOptions options)
    {
        var value = JsonSerializer.Deserialize<T>(ref reader, options);
        return Secret<T>.Create(value!);
    }

    public override void Write(Utf8JsonWriter writer, Secret<T> value, JsonSerializerOptions options)
    {
        // Never serialize the actual value
        writer.WriteStringValue("[REDACTED]");
    }
}

// Register in Startup
services.AddControllers()
    .AddJsonOptions(options =>
    {
        options.JsonSerializerOptions.Converters.Add(new SecretJsonConverter<string>());
    });

// Usage
public class Config
{
    public ApiKey ApiKey { get; set; }
    public DatabasePassword DbPassword { get; set; }
}

var config = new Config 
{ 
    ApiKey = ApiKey.Create("sk_live_abc123"),
    DbPassword = DatabasePassword.Create("super_secret")
};

var json = JsonSerializer.Serialize(config);
// Result: { "ApiKey": "[REDACTED API KEY]", "DbPassword": "[REDACTED PASSWORD]" }
```

## Auditable Access

Track when secrets are exposed:

```csharp
public sealed class Secret<T>
{
    private readonly T _value;
    private readonly List<SecretAccess> _accessLog = new();
    
    private Secret(T value) => _value = value;
    
    public static Secret<T> Create(T value) => new(value);
    
    public T Expose([CallerMemberName] string caller = "", 
                    [CallerFilePath] string file = "", 
                    [CallerLineNumber] int line = 0)
    {
        _accessLog.Add(new SecretAccess(caller, file, line, DateTime.UtcNow));
        return _value;
    }
    
    public IReadOnlyList<SecretAccess> GetAccessLog() => _accessLog.AsReadOnly();
    
    public override string ToString() => "[REDACTED]";
}

public sealed record SecretAccess(
    string Caller, 
    string FilePath, 
    int LineNumber, 
    DateTime AccessedAt);

// Usage
var apiKey = Secret<string>.Create("sk_live_abc123");
var key1 = apiKey.Expose();  // Tracked: who, where, when
var key2 = apiKey.Expose();  // Tracked again

foreach (var access in apiKey.GetAccessLog())
{
    Console.WriteLine($"Secret accessed by {access.Caller} at {access.AccessedAt}");
}
```

## Configuration Integration

```csharp
public class DatabaseConfig
{
    public string Host { get; set; } = "localhost";
    public int Port { get; set; } = 5432;
    public string Database { get; set; } = "myapp";
    public string Username { get; set; } = "admin";
    
    // Password is a secret—won't be logged or serialized
    public DatabasePassword Password { get; set; } = DatabasePassword.Create("");
    
    public string ConnectionString => Password.Value.Use(pwd =>
        $"Host={Host};Port={Port};Database={Database};Username={Username};Password={pwd}");
}

// Load from configuration
public class Startup
{
    public void ConfigureServices(IServiceCollection services)
    {
        services.Configure<DatabaseConfig>(options =>
        {
            options.Host = Configuration["Database:Host"];
            options.Port = int.Parse(Configuration["Database:Port"]);
            options.Database = Configuration["Database:Database"];
            options.Username = Configuration["Database:Username"];
            
            // Wrap secret value
            options.Password = DatabasePassword.Create(Configuration["Database:Password"]);
        });
    }
}

// Usage
public class DatabaseService
{
    private readonly DatabaseConfig _config;
    
    public DatabaseService(IOptions<DatabaseConfig> config)
    {
        _config = config.Value;
        
        // Safe to log—password is redacted
        _logger.LogInformation($"Connecting to {_config.Host}:{_config.Port} with config: {JsonSerializer.Serialize(_config)}");
    }
    
    public void Connect()
    {
        // Use connection string without exposing password
        var connection = new NpgsqlConnection(_config.ConnectionString);
        connection.Open();
    }
}
```

## Time-Safe Comparison

Prevent timing attacks when comparing secrets:

```csharp
public sealed class Secret<T>
{
    private readonly T _value;
    
    // ... other members ...
    
    /// <summary>
    /// Constant-time equality check to prevent timing attacks.
    /// Only works with byte arrays or strings.
    /// </summary>
    public bool SecureEquals(Secret<T> other)
    {
        if (_value is string s1 && other._value is string s2)
        {
            return CryptographicOperations.FixedTimeEquals(
                Encoding.UTF8.GetBytes(s1),
                Encoding.UTF8.GetBytes(s2));
        }
        
        if (_value is byte[] b1 && other._value is byte[] b2)
        {
            return CryptographicOperations.FixedTimeEquals(b1, b2);
        }
        
        throw new InvalidOperationException("SecureEquals only supports strings and byte arrays");
    }
}

// Usage for API key validation
public bool ValidateApiKey(ApiKey provided, ApiKey expected)
{
    // Constant-time comparison prevents timing attacks
    return provided.Value.SecureEquals(expected.Value);
}
```

## Why It's a Problem

1. **Accidental logging**: `Logger.LogInformation($"API Key: {key}")` exposes secrets
2. **Exception messages**: Secrets in exceptions appear in log aggregators
3. **JSON responses**: Accidentally returning config objects with secrets
4. **ToString() calls**: Debugging or logging objects exposes secrets
5. **Timing attacks**: `==` operator leaks information through timing

## Symptoms

- API keys found in application logs
- Passwords in exception stack traces
- Secrets in JSON API responses
- Database credentials in configuration dumps
- Security incidents from exposed secrets

## Benefits

- **Prevents accidental exposure**: `ToString()` never reveals secrets
- **Compile-time safety**: Cannot pass plain strings where secrets are required
- **Auditable access**: Track where and when secrets are exposed
- **Self-documenting**: `ProcessPayment(ApiKey key)` makes it clear the parameter is sensitive
- **Serialization-safe**: JSON/XML serializers don't expose secrets

## Trade-offs

- **More verbose**: Must call `.Expose()` or `.Use()` to access the value
- **Boxing overhead**: Wrapping primitive values in objects adds memory overhead
- **Cannot use in LINQ**: Cannot directly query over secret values
- **Requires discipline**: Team must consistently use secret types

## See Also

- [Input Sanitization](./input-sanitization.md) — trusted input types
- [Capability Security](./capability-security.md) — authorization tokens
- [Strongly Typed IDs](./strongly-typed-ids.md) — preventing ID confusion
- [Validated Configuration](./validated-configuration.md) — startup validation

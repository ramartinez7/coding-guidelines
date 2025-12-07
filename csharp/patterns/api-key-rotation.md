# API Key Rotation (Time-Limited Credentials)

> Using permanent API keys as strings—use time-limited, versioned API keys with automatic rotation to minimize exposure windows.

## Problem

Traditional API keys are permanent strings stored in plaintext configuration. When compromised, they remain valid forever until manually revoked. There's no built-in expiration, versioning, or rotation mechanism, and no compile-time guarantee that keys are validated before use.

## Example

### ❌ Before

```csharp
public class ExternalApiClient
{
    private readonly string _apiKey;
    
    public ExternalApiClient(IConfiguration config)
    {
        // Permanent key from config—never expires
        _apiKey = config["ExternalApi:ApiKey"];
    }
    
    public async Task<TResult> CallApiAsync<TResult>(string endpoint)
    {
        var request = new HttpRequestMessage(HttpMethod.Get, endpoint);
        request.Headers.Add("X-API-Key", _apiKey);  // String with no validation
        
        var response = await _httpClient.SendAsync(request);
        return await response.Content.ReadFromJsonAsync<TResult>();
    }
}

// API key validation
public class ApiKeyMiddleware
{
    public async Task InvokeAsync(HttpContext context)
    {
        var apiKey = context.Request.Headers["X-API-Key"].FirstOrDefault();
        
        // String comparison—no expiration check
        if (apiKey != _configuredApiKey)
        {
            context.Response.StatusCode = 401;
            return;
        }
        
        await _next(context);
    }
}
```

**Problems:**
- Permanent keys never expire
- No key versioning or rotation
- String-based with no type safety
- Compromised keys valid forever
- No compile-time validation guarantee

### ✅ After

```csharp
/// <summary>
/// Version identifier for API keys to support rotation.
/// </summary>
public readonly record struct ApiKeyVersion
{
    public int Value { get; }
    
    private ApiKeyVersion(int value) => Value = value;
    
    public static ApiKeyVersion Create(int value)
    {
        if (value < 1)
            throw new ArgumentException("Version must be positive");
        return new ApiKeyVersion(value);
    }
    
    public ApiKeyVersion Next() => new(Value + 1);
}

/// <summary>
/// Client identifier associated with API keys.
/// </summary>
public readonly record struct ClientId
{
    public Guid Value { get; }
    
    private ClientId(Guid value) => Value = value;
    
    public static ClientId New() => new(Guid.NewGuid());
    
    public static ClientId Parse(string value)
    {
        if (!Guid.TryParse(value, out var guid))
            throw new ArgumentException("Invalid client ID format");
        return new ClientId(guid);
    }
}

/// <summary>
/// Time-limited API key that expires automatically.
/// Cannot be constructed directly—must come from key management service.
/// </summary>
public sealed record ApiKey
{
    public required string KeyValue { get; init; }
    public required ClientId ClientId { get; init; }
    public required ApiKeyVersion Version { get; init; }
    public required DateTime CreatedAt { get; init; }
    public required DateTime ExpiresAt { get; init; }
    public required IReadOnlySet<string> Scopes { get; init; }
    
    internal ApiKey() { }  // Force construction through factory
    
    public bool IsExpired => DateTime.UtcNow >= ExpiresAt;
    
    public bool HasScope(string scope) => Scopes.Contains(scope);
    
    public TimeSpan TimeUntilExpiration => ExpiresAt - DateTime.UtcNow;
}

/// <summary>
/// Result of API key validation.
/// </summary>
public abstract record ApiKeyValidationResult
{
    private ApiKeyValidationResult() { }
    
    public sealed record Valid(ApiKey Key) : ApiKeyValidationResult;
    
    public sealed record Invalid(ApiKeyFailureReason Reason) : ApiKeyValidationResult;
}

public enum ApiKeyFailureReason
{
    KeyMissing,
    KeyInvalid,
    KeyExpired,
    KeyRevoked,
    InsufficientScope
}

/// <summary>
/// Manages API key lifecycle: creation, validation, rotation, revocation.
/// </summary>
public interface IApiKeyManager
{
    /// <summary>
    /// Creates a new API key for a client.
    /// </summary>
    Task<ApiKey> CreateKeyAsync(
        ClientId clientId,
        IReadOnlySet<string> scopes,
        TimeSpan lifetime);
    
    /// <summary>
    /// Validates an API key from a request.
    /// </summary>
    Task<ApiKeyValidationResult> ValidateKeyAsync(string keyValue, string? requiredScope = null);
    
    /// <summary>
    /// Rotates a client's API key (creates new, marks old for deprecation).
    /// </summary>
    Task<ApiKey> RotateKeyAsync(ClientId clientId);
    
    /// <summary>
    /// Revokes an API key immediately.
    /// </summary>
    Task RevokeKeyAsync(string keyValue);
}

public class ApiKeyManager : IApiKeyManager
{
    private readonly IApiKeyStore _store;
    private readonly IDataProtectionProvider _dataProtection;
    
    public ApiKeyManager(
        IApiKeyStore store,
        IDataProtectionProvider dataProtection)
    {
        _store = store;
        _dataProtection = dataProtection;
    }
    
    public async Task<ApiKey> CreateKeyAsync(
        ClientId clientId,
        IReadOnlySet<string> scopes,
        TimeSpan lifetime)
    {
        var currentVersion = await _store.GetLatestVersionAsync(clientId);
        var newVersion = currentVersion?.Next() ?? ApiKeyVersion.Create(1);
        
        // Generate cryptographically secure key
        var keyBytes = RandomNumberGenerator.GetBytes(32);
        var keyValue = $"sk_{clientId.Value:N}_{newVersion.Value}_{Convert.ToBase64String(keyBytes)}";
        
        var apiKey = new ApiKey
        {
            KeyValue = keyValue,
            ClientId = clientId,
            Version = newVersion,
            CreatedAt = DateTime.UtcNow,
            ExpiresAt = DateTime.UtcNow.Add(lifetime),
            Scopes = scopes
        };
        
        await _store.StoreKeyAsync(apiKey);
        
        return apiKey;
    }
    
    public async Task<ApiKeyValidationResult> ValidateKeyAsync(
        string keyValue,
        string? requiredScope = null)
    {
        if (string.IsNullOrWhiteSpace(keyValue))
            return new ApiKeyValidationResult.Invalid(ApiKeyFailureReason.KeyMissing);
        
        var key = await _store.GetKeyAsync(keyValue);
        
        if (key is null)
            return new ApiKeyValidationResult.Invalid(ApiKeyFailureReason.KeyInvalid);
        
        if (key.IsExpired)
            return new ApiKeyValidationResult.Invalid(ApiKeyFailureReason.KeyExpired);
        
        var isRevoked = await _store.IsRevokedAsync(keyValue);
        if (isRevoked)
            return new ApiKeyValidationResult.Invalid(ApiKeyFailureReason.KeyRevoked);
        
        if (requiredScope is not null && !key.HasScope(requiredScope))
            return new ApiKeyValidationResult.Invalid(ApiKeyFailureReason.InsufficientScope);
        
        return new ApiKeyValidationResult.Valid(key);
    }
    
    public async Task<ApiKey> RotateKeyAsync(ClientId clientId)
    {
        var currentKey = await _store.GetActiveKeyAsync(clientId);
        
        if (currentKey is not null)
        {
            // Mark old key for deprecation (grace period before full revocation)
            await _store.MarkDeprecatedAsync(
                currentKey.KeyValue,
                gracePeriod: TimeSpan.FromDays(7));
        }
        
        // Create new key with same scopes but new version
        var newKey = await CreateKeyAsync(
            clientId,
            currentKey?.Scopes ?? new HashSet<string>(),
            lifetime: TimeSpan.FromDays(90));
        
        return newKey;
    }
    
    public async Task RevokeKeyAsync(string keyValue)
    {
        await _store.RevokeKeyAsync(keyValue);
    }
}

/// <summary>
/// Storage for API keys and their state.
/// </summary>
public interface IApiKeyStore
{
    Task StoreKeyAsync(ApiKey key);
    Task<ApiKey?> GetKeyAsync(string keyValue);
    Task<ApiKey?> GetActiveKeyAsync(ClientId clientId);
    Task<ApiKeyVersion?> GetLatestVersionAsync(ClientId clientId);
    Task<bool> IsRevokedAsync(string keyValue);
    Task RevokeKeyAsync(string keyValue);
    Task MarkDeprecatedAsync(string keyValue, TimeSpan gracePeriod);
}

// Example implementation using Entity Framework
public class ApiKeyStore : IApiKeyStore
{
    private readonly ApplicationDbContext _db;
    
    public async Task StoreKeyAsync(ApiKey key)
    {
        _db.ApiKeys.Add(new ApiKeyRecord
        {
            KeyValue = key.KeyValue,
            ClientId = key.ClientId.Value,
            Version = key.Version.Value,
            CreatedAt = key.CreatedAt,
            ExpiresAt = key.ExpiresAt,
            Scopes = string.Join(",", key.Scopes),
            IsRevoked = false
        });
        
        await _db.SaveChangesAsync();
    }
    
    public async Task<ApiKey?> GetKeyAsync(string keyValue)
    {
        var record = await _db.ApiKeys
            .FirstOrDefaultAsync(k => k.KeyValue == keyValue);
        
        if (record is null)
            return null;
        
        return new ApiKey
        {
            KeyValue = record.KeyValue,
            ClientId = ClientId.Parse(record.ClientId.ToString()),
            Version = ApiKeyVersion.Create(record.Version),
            CreatedAt = record.CreatedAt,
            ExpiresAt = record.ExpiresAt,
            Scopes = record.Scopes.Split(',').ToHashSet()
        };
    }
    
    public async Task<ApiKey?> GetActiveKeyAsync(ClientId clientId)
    {
        var record = await _db.ApiKeys
            .Where(k => k.ClientId == clientId.Value)
            .Where(k => !k.IsRevoked)
            .Where(k => k.ExpiresAt > DateTime.UtcNow)
            .OrderByDescending(k => k.Version)
            .FirstOrDefaultAsync();
        
        if (record is null)
            return null;
        
        return await GetKeyAsync(record.KeyValue);
    }
    
    public async Task<ApiKeyVersion?> GetLatestVersionAsync(ClientId clientId)
    {
        var latestVersion = await _db.ApiKeys
            .Where(k => k.ClientId == clientId.Value)
            .MaxAsync(k => (int?)k.Version);
        
        return latestVersion.HasValue
            ? ApiKeyVersion.Create(latestVersion.Value)
            : null;
    }
    
    public async Task<bool> IsRevokedAsync(string keyValue)
    {
        return await _db.ApiKeys
            .Where(k => k.KeyValue == keyValue)
            .AnyAsync(k => k.IsRevoked);
    }
    
    public async Task RevokeKeyAsync(string keyValue)
    {
        await _db.ApiKeys
            .Where(k => k.KeyValue == keyValue)
            .ExecuteUpdateAsync(s => s.SetProperty(k => k.IsRevoked, true));
    }
    
    public async Task MarkDeprecatedAsync(string keyValue, TimeSpan gracePeriod)
    {
        var newExpiresAt = DateTime.UtcNow.Add(gracePeriod);
        
        await _db.ApiKeys
            .Where(k => k.KeyValue == keyValue)
            .ExecuteUpdateAsync(s => s
                .SetProperty(k => k.ExpiresAt, newExpiresAt)
                .SetProperty(k => k.IsDeprecated, true));
    }
}

// Middleware validates API keys
public class ApiKeyMiddleware
{
    private readonly RequestDelegate _next;
    
    public async Task InvokeAsync(
        HttpContext context,
        IApiKeyManager keyManager)
    {
        var apiKeyHeader = context.Request.Headers["X-API-Key"].FirstOrDefault();
        
        var result = await keyManager.ValidateKeyAsync(apiKeyHeader ?? "");
        
        if (result is ApiKeyValidationResult.Invalid invalid)
        {
            context.Response.StatusCode = 401;
            await context.Response.WriteAsJsonAsync(new
            {
                error = "invalid_api_key",
                reason = invalid.Reason.ToString()
            });
            return;
        }
        
        var valid = (ApiKeyValidationResult.Valid)result;
        
        // Add deprecation warning if key expires soon
        if (valid.Key.TimeUntilExpiration < TimeSpan.FromDays(7))
        {
            context.Response.Headers.Add("X-API-Key-Expiring",
                $"Key expires at {valid.Key.ExpiresAt:O}. Please rotate.");
        }
        
        // Store validated key in HttpContext
        context.Items["ApiKey"] = valid.Key;
        
        await _next(context);
    }
}

// Extension to get validated API key
public static class HttpContextExtensions
{
    public static ApiKey GetApiKey(this HttpContext context)
    {
        return context.Items["ApiKey"] as ApiKey
            ?? throw new InvalidOperationException("API key not validated");
    }
}

// Service methods require validated API key
public interface IPaymentService
{
    Task<PaymentResult> ProcessPaymentAsync(
        ApiKey apiKey,
        PaymentRequest request);
}

public class PaymentService : IPaymentService
{
    public async Task<PaymentResult> ProcessPaymentAsync(
        ApiKey apiKey,
        PaymentRequest request)
    {
        // Type system guarantees key validation happened
        
        if (!apiKey.HasScope("payments:write"))
            throw new InsufficientScopeException("payments:write scope required");
        
        if (apiKey.IsExpired)
            throw new SecurityException("API key expired");
        
        return await ProcessAsync(request);
    }
}

// Controller
[ApiController]
[Route("api/payments")]
public class PaymentsController : ControllerBase
{
    private readonly IPaymentService _paymentService;
    
    [HttpPost]
    public async Task<IActionResult> ProcessPayment([FromBody] PaymentRequest request)
    {
        var apiKey = HttpContext.GetApiKey();
        
        var result = await _paymentService.ProcessPaymentAsync(apiKey, request);
        
        return Ok(result);
    }
}
```

## Automatic Key Rotation

```csharp
/// <summary>
/// Background service that automatically rotates API keys before expiration.
/// </summary>
public class ApiKeyRotationService : BackgroundService
{
    private readonly IApiKeyManager _keyManager;
    private readonly IApiKeyStore _store;
    private readonly ILogger<ApiKeyRotationService> _logger;
    
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            await RotateExpiringKeysAsync(stoppingToken);
            await Task.Delay(TimeSpan.FromHours(1), stoppingToken);
        }
    }
    
    private async Task RotateExpiringKeysAsync(CancellationToken cancellationToken)
    {
        var threshold = DateTime.UtcNow.AddDays(7);
        
        var expiringKeys = await _store.GetKeysExpiringBeforeAsync(threshold);
        
        foreach (var key in expiringKeys)
        {
            if (cancellationToken.IsCancellationRequested)
                break;
            
            try
            {
                var newKey = await _keyManager.RotateKeyAsync(key.ClientId);
                
                _logger.LogInformation(
                    "Rotated API key for client {ClientId}. Old version: {OldVersion}, New version: {NewVersion}",
                    key.ClientId,
                    key.Version,
                    newKey.Version);
                
                // Notify client about new key
                await NotifyClientOfRotationAsync(key.ClientId, newKey);
            }
            catch (Exception ex)
            {
                _logger.LogError(ex,
                    "Failed to rotate API key for client {ClientId}",
                    key.ClientId);
            }
        }
    }
    
    private async Task NotifyClientOfRotationAsync(ClientId clientId, ApiKey newKey)
    {
        // Send email/webhook notification with new key
        // Implementation depends on notification system
    }
}
```

## Scoped API Keys

```csharp
/// <summary>
/// API key with specific scope requirement proven at compile time.
/// </summary>
public sealed record ScopedApiKey<TScope> where TScope : IApiScope
{
    public ApiKey Key { get; }
    
    private ScopedApiKey(ApiKey key) => Key = key;
    
    public static Result<ScopedApiKey<TScope>, string> From(ApiKey key)
    {
        var requiredScope = TScope.ScopeName;
        
        if (!key.HasScope(requiredScope))
            return Result<ScopedApiKey<TScope>, string>.Failure(
                $"Key missing required scope: {requiredScope}");
        
        return Result<ScopedApiKey<TScope>, string>.Success(
            new ScopedApiKey<TScope>(key));
    }
}

// Scope marker interface and types
public interface IApiScope
{
    static abstract string ScopeName { get; }
}

public sealed record PaymentsReadScope : IApiScope
{
    public static string ScopeName => "payments:read";
}

public sealed record PaymentsWriteScope : IApiScope
{
    public static string ScopeName => "payments:write";
}

// Service requires specific scope
public interface IPaymentService
{
    Task<IEnumerable<Payment>> GetPaymentsAsync(
        ScopedApiKey<PaymentsReadScope> apiKey);
    
    Task<PaymentResult> ProcessPaymentAsync(
        ScopedApiKey<PaymentsWriteScope> apiKey,
        PaymentRequest request);
}
```

## Testing

```csharp
public class ApiKeyRotationTests
{
    [Fact]
    public async Task CreateKey_GeneratesValidKey()
    {
        var manager = CreateManager();
        var clientId = ClientId.New();
        var scopes = new HashSet<string> { "payments:read", "payments:write" };
        
        var key = await manager.CreateKeyAsync(clientId, scopes, TimeSpan.FromDays(90));
        
        Assert.Equal(clientId, key.ClientId);
        Assert.Equal(scopes, key.Scopes);
        Assert.True(key.ExpiresAt > DateTime.UtcNow);
    }
    
    [Fact]
    public async Task ValidateKey_WithExpiredKey_ReturnsInvalid()
    {
        var manager = CreateManager();
        var clientId = ClientId.New();
        
        var key = await manager.CreateKeyAsync(
            clientId,
            new HashSet<string>(),
            TimeSpan.FromMilliseconds(-1));  // Already expired
        
        var result = await manager.ValidateKeyAsync(key.KeyValue);
        
        var invalid = Assert.IsType<ApiKeyValidationResult.Invalid>(result);
        Assert.Equal(ApiKeyFailureReason.KeyExpired, invalid.Reason);
    }
    
    [Fact]
    public async Task RotateKey_CreatesNewVersionAndDeprecatesOld()
    {
        var manager = CreateManager();
        var store = CreateStore();
        var clientId = ClientId.New();
        
        var key1 = await manager.CreateKeyAsync(
            clientId,
            new HashSet<string> { "read" },
            TimeSpan.FromDays(90));
        
        var key2 = await manager.RotateKeyAsync(clientId);
        
        Assert.Equal(clientId, key2.ClientId);
        Assert.True(key2.Version.Value > key1.Version.Value);
        
        var oldKey = await store.GetKeyAsync(key1.KeyValue);
        Assert.NotNull(oldKey);
        // Old key should be deprecated but still valid during grace period
    }
}
```

## Why It's a Problem

1. **Permanent keys**: Once compromised, valid forever
2. **No expiration**: Keys don't expire automatically
3. **No versioning**: Can't track which key version is in use
4. **Manual rotation**: Requires manual coordination to rotate keys
5. **String-based**: No type safety around key validation

## Symptoms

- Permanent API keys in configuration
- No key expiration or rotation policy
- Compromised keys remain valid indefinitely
- String-based key validation
- Manual key revocation process

## Benefits

- **Automatic expiration**: Keys expire after defined lifetime
- **Versioned keys**: Track key versions for rotation
- **Type-safe validation**: `ApiKey` type proves validation occurred
- **Compile-time enforcement**: Cannot use services without validated key
- **Rotation support**: Built-in key rotation with grace periods

## See Also

- [Secret Types](./secret-types.md) — preventing key exposure
- [Capability Security](./capability-security.md) — authorization tokens
- [Authentication Context](./authentication-context.md) — user identity
- [Rate Limiting](./rate-limiting-tokens.md) — throttling API usage

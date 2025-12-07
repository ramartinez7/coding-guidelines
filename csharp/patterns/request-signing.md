# Request Signing (Message Authentication)

> Trusting API requests at face value—use HMAC-based request signatures to verify authenticity and prevent tampering.

## Problem

API requests can be intercepted and modified in transit. Without cryptographic signatures, there's no way to verify that a request came from a legitimate client or that its contents haven't been tampered with. Traditional approaches rely on HTTPS alone, which doesn't protect against compromised credentials or man-in-the-middle attacks at the application layer.

## Example

### ❌ Before

```csharp
public class WebhookController : ControllerBase
{
    [HttpPost("webhook")]
    public IActionResult ProcessWebhook([FromBody] WebhookPayload payload)
    {
        // How do we know this came from the real service?
        // Anyone who knows the endpoint can send fake webhooks!
        
        _orderService.ProcessOrder(payload.OrderId);
        return Ok();
    }
}

// Client sending request
public class PaymentClient
{
    public async Task NotifyPaymentAsync(PaymentNotification notification)
    {
        var json = JsonSerializer.Serialize(notification);
        var content = new StringContent(json, Encoding.UTF8, "application/json");
        
        // No signature—recipient can't verify this came from us
        await _httpClient.PostAsync("/webhook", content);
    }
}
```

**Problems:**
- No proof request came from legitimate source
- Request body can be modified in transit
- Replay attacks possible
- No timestamp validation
- String-based with no type safety

### ✅ After

```csharp
/// <summary>
/// Cryptographic signature proving request authenticity.
/// Cannot be constructed directly—must come from signing service.
/// </summary>
public sealed record RequestSignature
{
    public required string SignatureValue { get; init; }
    public required SigningAlgorithm Algorithm { get; init; }
    public required DateTime Timestamp { get; init; }
    public required string Nonce { get; init; }
    
    internal RequestSignature() { }
}

public enum SigningAlgorithm
{
    HmacSha256,
    HmacSha512
}

/// <summary>
/// Secret key used for signing requests.
/// </summary>
public sealed record SigningKey(Secret<byte[]> KeyBytes, SigningAlgorithm Algorithm)
{
    public static SigningKey Generate(SigningAlgorithm algorithm = SigningAlgorithm.HmacSha256)
    {
        var keySize = algorithm switch
        {
            SigningAlgorithm.HmacSha256 => 32,
            SigningAlgorithm.HmacSha512 => 64,
            _ => throw new ArgumentException("Unknown algorithm")
        };
        
        var keyBytes = RandomNumberGenerator.GetBytes(keySize);
        return new SigningKey(Secret<byte[]>.Create(keyBytes), algorithm);
    }
    
    public string ToBase64() => KeyBytes.Use(bytes => Convert.ToBase64String(bytes));
    
    public static SigningKey FromBase64(string base64, SigningAlgorithm algorithm)
    {
        var bytes = Convert.FromBase64String(base64);
        return new SigningKey(Secret<byte[]>.Create(bytes), algorithm);
    }
}

/// <summary>
/// Result of signature validation.
/// </summary>
public abstract record SignatureValidationResult
{
    private SignatureValidationResult() { }
    
    public sealed record Valid(RequestSignature Signature) : SignatureValidationResult;
    
    public sealed record Invalid(SignatureFailureReason Reason) : SignatureValidationResult;
}

public enum SignatureFailureReason
{
    SignatureMissing,
    SignatureInvalid,
    TimestampExpired,
    NonceReused,
    AlgorithmMismatch
}

/// <summary>
/// Service for signing and verifying HTTP requests.
/// </summary>
public interface IRequestSigningService
{
    /// <summary>
    /// Signs a request body with the signing key.
    /// </summary>
    RequestSignature SignRequest(
        SigningKey key,
        string httpMethod,
        string path,
        string body,
        DateTime timestamp);
    
    /// <summary>
    /// Validates a request signature.
    /// </summary>
    Task<SignatureValidationResult> ValidateSignatureAsync(
        SigningKey key,
        string httpMethod,
        string path,
        string body,
        string signatureHeader,
        string timestampHeader,
        string nonceHeader);
}

public class RequestSigningService : IRequestSigningService
{
    private readonly INonceStore _nonceStore;
    private readonly TimeSpan _timestampTolerance;
    
    public RequestSigningService(
        INonceStore nonceStore,
        TimeSpan? timestampTolerance = null)
    {
        _nonceStore = nonceStore;
        _timestampTolerance = timestampTolerance ?? TimeSpan.FromMinutes(5);
    }
    
    public RequestSignature SignRequest(
        SigningKey key,
        string httpMethod,
        string path,
        string body,
        DateTime timestamp)
    {
        var nonce = Guid.NewGuid().ToString("N");
        
        var stringToSign = BuildStringToSign(
            httpMethod,
            path,
            body,
            timestamp.ToString("O"),
            nonce);
        
        var signatureBytes = key.KeyBytes.Use(keyBytes =>
        {
            using var hmac = key.Algorithm switch
            {
                SigningAlgorithm.HmacSha256 => new HMACSHA256(keyBytes),
                SigningAlgorithm.HmacSha512 => new HMACSHA512(keyBytes),
                _ => throw new ArgumentException("Unknown algorithm")
            };
            
            return hmac.ComputeHash(Encoding.UTF8.GetBytes(stringToSign));
        });
        
        return new RequestSignature
        {
            SignatureValue = Convert.ToBase64String(signatureBytes),
            Algorithm = key.Algorithm,
            Timestamp = timestamp,
            Nonce = nonce
        };
    }
    
    public async Task<SignatureValidationResult> ValidateSignatureAsync(
        SigningKey key,
        string httpMethod,
        string path,
        string body,
        string signatureHeader,
        string timestampHeader,
        string nonceHeader)
    {
        if (string.IsNullOrWhiteSpace(signatureHeader))
            return new SignatureValidationResult.Invalid(SignatureFailureReason.SignatureMissing);
        
        if (!DateTime.TryParse(timestampHeader, out var timestamp))
            return new SignatureValidationResult.Invalid(SignatureFailureReason.TimestampExpired);
        
        // Check timestamp is within tolerance
        var age = DateTime.UtcNow - timestamp;
        if (Math.Abs(age.TotalSeconds) > _timestampTolerance.TotalSeconds)
            return new SignatureValidationResult.Invalid(SignatureFailureReason.TimestampExpired);
        
        // Check nonce hasn't been used before (prevents replay attacks)
        if (await _nonceStore.HasBeenUsedAsync(nonceHeader))
            return new SignatureValidationResult.Invalid(SignatureFailureReason.NonceReused);
        
        // Compute expected signature
        var stringToSign = BuildStringToSign(
            httpMethod,
            path,
            body,
            timestampHeader,
            nonceHeader);
        
        var expectedSignatureBytes = key.KeyBytes.Use(keyBytes =>
        {
            using var hmac = key.Algorithm switch
            {
                SigningAlgorithm.HmacSha256 => new HMACSHA256(keyBytes),
                SigningAlgorithm.HmacSha512 => new HMACSHA512(keyBytes),
                _ => throw new ArgumentException("Unknown algorithm")
            };
            
            return hmac.ComputeHash(Encoding.UTF8.GetBytes(stringToSign));
        });
        
        var expectedSignature = Convert.ToBase64String(expectedSignatureBytes);
        
        // Constant-time comparison to prevent timing attacks
        if (!CryptographicOperations.FixedTimeEquals(
            Encoding.UTF8.GetBytes(expectedSignature),
            Encoding.UTF8.GetBytes(signatureHeader)))
        {
            return new SignatureValidationResult.Invalid(SignatureFailureReason.SignatureInvalid);
        }
        
        // Store nonce to prevent reuse
        await _nonceStore.StoreNonceAsync(nonceHeader, timestamp.Add(_timestampTolerance));
        
        return new SignatureValidationResult.Valid(new RequestSignature
        {
            SignatureValue = signatureHeader,
            Algorithm = key.Algorithm,
            Timestamp = timestamp,
            Nonce = nonceHeader
        });
    }
    
    private string BuildStringToSign(
        string httpMethod,
        string path,
        string body,
        string timestamp,
        string nonce)
    {
        // Canonical format: METHOD\nPATH\nTIMESTAMP\nNONCE\nBODY
        return $"{httpMethod.ToUpperInvariant()}\n{path}\n{timestamp}\n{nonce}\n{body}";
    }
}

/// <summary>
/// Storage for nonces to prevent replay attacks.
/// </summary>
public interface INonceStore
{
    Task<bool> HasBeenUsedAsync(string nonce);
    Task StoreNonceAsync(string nonce, DateTime expiresAt);
}

// In-memory implementation (use Redis or database in production)
public class InMemoryNonceStore : INonceStore
{
    private readonly ConcurrentDictionary<string, DateTime> _nonces = new();
    
    public Task<bool> HasBeenUsedAsync(string nonce)
    {
        CleanupExpired();
        return Task.FromResult(_nonces.ContainsKey(nonce));
    }
    
    public Task StoreNonceAsync(string nonce, DateTime expiresAt)
    {
        _nonces.TryAdd(nonce, expiresAt);
        return Task.CompletedTask;
    }
    
    private void CleanupExpired()
    {
        var now = DateTime.UtcNow;
        var expired = _nonces.Where(kvp => kvp.Value < now).Select(kvp => kvp.Key).ToList();
        
        foreach (var key in expired)
            _nonces.TryRemove(key, out _);
    }
}

// Middleware validates request signatures
public class RequestSignatureMiddleware
{
    private readonly RequestDelegate _next;
    
    public async Task InvokeAsync(
        HttpContext context,
        IRequestSigningService signingService,
        ISigningKeyProvider keyProvider)
    {
        // Only validate POST/PUT/PATCH/DELETE
        if (context.Request.Method is not ("POST" or "PUT" or "PATCH" or "DELETE"))
        {
            await _next(context);
            return;
        }
        
        // Read request body
        context.Request.EnableBuffering();
        using var reader = new StreamReader(
            context.Request.Body,
            Encoding.UTF8,
            leaveOpen: true);
        var body = await reader.ReadToEndAsync();
        context.Request.Body.Position = 0;
        
        // Extract signature headers
        var signatureHeader = context.Request.Headers["X-Signature"].FirstOrDefault() ?? "";
        var timestampHeader = context.Request.Headers["X-Signature-Timestamp"].FirstOrDefault() ?? "";
        var nonceHeader = context.Request.Headers["X-Signature-Nonce"].FirstOrDefault() ?? "";
        var keyIdHeader = context.Request.Headers["X-Key-Id"].FirstOrDefault() ?? "";
        
        // Get signing key for this client
        var key = await keyProvider.GetKeyAsync(keyIdHeader);
        if (key is null)
        {
            context.Response.StatusCode = 401;
            await context.Response.WriteAsJsonAsync(new { error = "invalid_key_id" });
            return;
        }
        
        // Validate signature
        var result = await signingService.ValidateSignatureAsync(
            key,
            context.Request.Method,
            context.Request.Path,
            body,
            signatureHeader,
            timestampHeader,
            nonceHeader);
        
        if (result is SignatureValidationResult.Invalid invalid)
        {
            context.Response.StatusCode = 401;
            await context.Response.WriteAsJsonAsync(new
            {
                error = "invalid_signature",
                reason = invalid.Reason.ToString()
            });
            return;
        }
        
        var valid = (SignatureValidationResult.Valid)result;
        context.Items["RequestSignature"] = valid.Signature;
        
        await _next(context);
    }
}

// Extension to get validated signature
public static class HttpContextExtensions
{
    public static RequestSignature GetRequestSignature(this HttpContext context)
    {
        return context.Items["RequestSignature"] as RequestSignature
            ?? throw new InvalidOperationException("Request signature not validated");
    }
}

// Client sending signed requests
public class SignedHttpClient
{
    private readonly HttpClient _httpClient;
    private readonly IRequestSigningService _signingService;
    private readonly SigningKey _signingKey;
    private readonly string _keyId;
    
    public SignedHttpClient(
        HttpClient httpClient,
        IRequestSigningService signingService,
        SigningKey signingKey,
        string keyId)
    {
        _httpClient = httpClient;
        _signingService = signingService;
        _signingKey = signingKey;
        _keyId = keyId;
    }
    
    public async Task<HttpResponseMessage> PostAsync<T>(string path, T payload)
    {
        var body = JsonSerializer.Serialize(payload);
        var timestamp = DateTime.UtcNow;
        
        var signature = _signingService.SignRequest(
            _signingKey,
            "POST",
            path,
            body,
            timestamp);
        
        var request = new HttpRequestMessage(HttpMethod.Post, path)
        {
            Content = new StringContent(body, Encoding.UTF8, "application/json")
        };
        
        request.Headers.Add("X-Signature", signature.SignatureValue);
        request.Headers.Add("X-Signature-Timestamp", timestamp.ToString("O"));
        request.Headers.Add("X-Signature-Nonce", signature.Nonce);
        request.Headers.Add("X-Key-Id", _keyId);
        
        return await _httpClient.SendAsync(request);
    }
}

// Service methods require validated signature
public interface IWebhookService
{
    Task ProcessWebhookAsync(
        RequestSignature signature,
        WebhookPayload payload);
}

public class WebhookService : IWebhookService
{
    public async Task ProcessWebhookAsync(
        RequestSignature signature,
        WebhookPayload payload)
    {
        // Type system guarantees signature validation happened
        
        if (DateTime.UtcNow - signature.Timestamp > TimeSpan.FromMinutes(5))
            throw new SecurityException("Signature timestamp too old");
        
        await ProcessAsync(payload);
    }
}

// Controller
[ApiController]
[Route("api/webhooks")]
public class WebhooksController : ControllerBase
{
    private readonly IWebhookService _webhookService;
    
    [HttpPost]
    public async Task<IActionResult> ProcessWebhook([FromBody] WebhookPayload payload)
    {
        var signature = HttpContext.GetRequestSignature();
        
        await _webhookService.ProcessWebhookAsync(signature, payload);
        
        return Ok();
    }
}
```

## AWS Signature Version 4 Style

For more complex scenarios, use AWS-style canonical request signing:

```csharp
public class CanonicalRequestSigner : IRequestSigningService
{
    public RequestSignature SignRequest(
        SigningKey key,
        string httpMethod,
        string path,
        string body,
        DateTime timestamp)
    {
        // Build canonical request
        var canonicalRequest = BuildCanonicalRequest(
            httpMethod,
            path,
            queryString: "",
            canonicalHeaders: "host:api.example.com\nx-timestamp:" + timestamp.ToString("O"),
            signedHeaders: "host;x-timestamp",
            body);
        
        // Hash canonical request
        var canonicalRequestHash = ComputeSha256Hash(canonicalRequest);
        
        // Build string to sign
        var stringToSign = $"HMAC-SHA256\n{timestamp:O}\n{canonicalRequestHash}";
        
        // Sign
        var signatureBytes = key.KeyBytes.Use(keyBytes =>
        {
            using var hmac = new HMACSHA256(keyBytes);
            return hmac.ComputeHash(Encoding.UTF8.GetBytes(stringToSign));
        });
        
        return new RequestSignature
        {
            SignatureValue = Convert.ToBase64String(signatureBytes),
            Algorithm = SigningAlgorithm.HmacSha256,
            Timestamp = timestamp,
            Nonce = Guid.NewGuid().ToString("N")
        };
    }
    
    private string BuildCanonicalRequest(
        string method,
        string path,
        string queryString,
        string canonicalHeaders,
        string signedHeaders,
        string body)
    {
        var bodyHash = ComputeSha256Hash(body);
        
        return $"{method}\n{path}\n{queryString}\n{canonicalHeaders}\n\n{signedHeaders}\n{bodyHash}";
    }
    
    private string ComputeSha256Hash(string input)
    {
        var bytes = SHA256.HashData(Encoding.UTF8.GetBytes(input));
        return Convert.ToHexString(bytes).ToLowerInvariant();
    }
}
```

## Webhook Signature Verification

Common pattern for webhook providers (Stripe, GitHub, etc.):

```csharp
/// <summary>
/// Validates webhook signatures from external providers.
/// </summary>
public class WebhookSignatureValidator
{
    public static SignatureValidationResult ValidateStripeSignature(
        string payload,
        string signatureHeader,
        string webhookSecret,
        long toleranceSeconds = 300)
    {
        // Stripe format: t=timestamp,v1=signature
        var parts = signatureHeader.Split(',');
        var timestamp = long.Parse(parts[0].Split('=')[1]);
        var signature = parts[1].Split('=')[1];
        
        // Check timestamp
        var now = DateTimeOffset.UtcNow.ToUnixTimeSeconds();
        if (Math.Abs(now - timestamp) > toleranceSeconds)
            return new SignatureValidationResult.Invalid(
                SignatureFailureReason.TimestampExpired);
        
        // Compute expected signature
        var signedPayload = $"{timestamp}.{payload}";
        var secretBytes = Encoding.UTF8.GetBytes(webhookSecret);
        
        using var hmac = new HMACSHA256(secretBytes);
        var hash = hmac.ComputeHash(Encoding.UTF8.GetBytes(signedPayload));
        var expectedSignature = Convert.ToHexString(hash).ToLowerInvariant();
        
        // Compare
        if (!CryptographicOperations.FixedTimeEquals(
            Encoding.UTF8.GetBytes(expectedSignature),
            Encoding.UTF8.GetBytes(signature)))
        {
            return new SignatureValidationResult.Invalid(
                SignatureFailureReason.SignatureInvalid);
        }
        
        return new SignatureValidationResult.Valid(new RequestSignature
        {
            SignatureValue = signature,
            Algorithm = SigningAlgorithm.HmacSha256,
            Timestamp = DateTimeOffset.FromUnixTimeSeconds(timestamp).UtcDateTime,
            Nonce = ""
        });
    }
}
```

## Testing

```csharp
public class RequestSigningTests
{
    [Fact]
    public void SignRequest_ProducesValidSignature()
    {
        var signingService = new RequestSigningService(new InMemoryNonceStore());
        var key = SigningKey.Generate();
        var timestamp = DateTime.UtcNow;
        
        var signature = signingService.SignRequest(
            key,
            "POST",
            "/api/webhook",
            "{\"test\":true}",
            timestamp);
        
        Assert.NotNull(signature.SignatureValue);
        Assert.Equal(timestamp, signature.Timestamp);
    }
    
    [Fact]
    public async Task ValidateSignature_WithValidSignature_ReturnsValid()
    {
        var signingService = new RequestSigningService(new InMemoryNonceStore());
        var key = SigningKey.Generate();
        var timestamp = DateTime.UtcNow;
        var body = "{\"test\":true}";
        
        var signature = signingService.SignRequest(key, "POST", "/api/webhook", body, timestamp);
        
        var result = await signingService.ValidateSignatureAsync(
            key,
            "POST",
            "/api/webhook",
            body,
            signature.SignatureValue,
            timestamp.ToString("O"),
            signature.Nonce);
        
        Assert.IsType<SignatureValidationResult.Valid>(result);
    }
    
    [Fact]
    public async Task ValidateSignature_WithModifiedBody_ReturnsInvalid()
    {
        var signingService = new RequestSigningService(new InMemoryNonceStore());
        var key = SigningKey.Generate();
        var timestamp = DateTime.UtcNow;
        var originalBody = "{\"amount\":100}";
        var tamperedBody = "{\"amount\":999}";
        
        var signature = signingService.SignRequest(key, "POST", "/api/payment", originalBody, timestamp);
        
        // Try to validate with tampered body
        var result = await signingService.ValidateSignatureAsync(
            key,
            "POST",
            "/api/payment",
            tamperedBody,
            signature.SignatureValue,
            timestamp.ToString("O"),
            signature.Nonce);
        
        var invalid = Assert.IsType<SignatureValidationResult.Invalid>(result);
        Assert.Equal(SignatureFailureReason.SignatureInvalid, invalid.Reason);
    }
    
    [Fact]
    public async Task ValidateSignature_WithReusedNonce_ReturnsInvalid()
    {
        var nonceStore = new InMemoryNonceStore();
        var signingService = new RequestSigningService(nonceStore);
        var key = SigningKey.Generate();
        var timestamp = DateTime.UtcNow;
        var body = "{\"test\":true}";
        
        var signature = signingService.SignRequest(key, "POST", "/api/webhook", body, timestamp);
        
        // First validation succeeds
        await signingService.ValidateSignatureAsync(
            key, "POST", "/api/webhook", body,
            signature.SignatureValue, timestamp.ToString("O"), signature.Nonce);
        
        // Second validation with same nonce fails (replay attack)
        var result = await signingService.ValidateSignatureAsync(
            key, "POST", "/api/webhook", body,
            signature.SignatureValue, timestamp.ToString("O"), signature.Nonce);
        
        var invalid = Assert.IsType<SignatureValidationResult.Invalid>(result);
        Assert.Equal(SignatureFailureReason.NonceReused, invalid.Reason);
    }
}
```

## Why It's a Problem

1. **No authenticity proof**: Can't verify request came from legitimate source
2. **Tampering possible**: Request body can be modified in transit
3. **Replay attacks**: Old requests can be replayed
4. **No timestamp validation**: Stale requests accepted
5. **String-based**: No type safety around signature validation

## Symptoms

- Webhook endpoints vulnerable to spoofing
- No way to verify request hasn't been tampered with
- Replay attacks possible
- String-based signature validation
- Manual HMAC computation scattered in code

## Benefits

- **Cryptographic proof**: Signatures prove request authenticity
- **Tamper detection**: Modified requests fail validation
- **Replay prevention**: Nonces prevent replay attacks
- **Type-safe**: `RequestSignature` type proves validation occurred
- **Compile-time enforcement**: Cannot use services without validated signature

## See Also

- [Secret Types](./secret-types.md) — protecting signing keys
- [API Key Rotation](./api-key-rotation.md) — managing API credentials
- [Input Sanitization](./input-sanitization.md) — trusted types
- [Capability Security](./capability-security.md) — authorization tokens

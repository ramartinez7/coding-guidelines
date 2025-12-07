# CSRF Protection (Anti-Forgery Tokens)

> Using string-based tokens or forgetting CSRF checks—use typed anti-forgery tokens to prevent cross-site request forgery at compile time.

## Problem

Cross-Site Request Forgery (CSRF) attacks trick authenticated users into executing unwanted actions. Traditional CSRF protection relies on middleware or manual token validation, both error-prone. Developers might forget to add the `[ValidateAntiForgeryToken]` attribute, or tokens might be passed as plain strings without validation.

## Example

### ❌ Before

```csharp
public class TransferController : Controller
{
    [HttpPost]
    public IActionResult Transfer(Guid toAccount, decimal amount)
    {
        // No CSRF protection—attacker can forge this request!
        // If user is authenticated and visits attacker's site:
        // <form action="https://bank.com/transfer" method="post">
        //   <input name="toAccount" value="attacker-account" />
        //   <input name="amount" value="10000" />
        // </form>
        
        var user = GetCurrentUser();
        _bankService.Transfer(user.AccountId, toAccount, amount);
        return Ok();
    }
}

// Manual token validation is error-prone
public class PaymentController : Controller
{
    [HttpPost]
    public IActionResult Pay(string csrfToken, PaymentRequest request)
    {
        // Did we validate the token? Type doesn't tell us.
        if (csrfToken != GetExpectedToken())
            return BadRequest();
        
        _paymentService.Process(request);
        return Ok();
    }
}
```

**Problems:**
- Easy to forget `[ValidateAntiForgeryToken]` attribute
- No compile-time guarantee CSRF check happened
- String-based tokens provide no type safety
- Manual validation is scattered and inconsistent

### ✅ After

```csharp
/// <summary>
/// Proof that CSRF validation has succeeded for this request.
/// Cannot be constructed outside the anti-forgery service.
/// </summary>
public sealed record CsrfToken
{
    public required string TokenValue { get; init; }
    public required DateTime GeneratedAt { get; init; }
    public required DateTime ExpiresAt { get; init; }
    public required SessionId SessionId { get; init; }
    
    internal CsrfToken() { }  // Force construction through factory
}

/// <summary>
/// Result of CSRF token validation.
/// </summary>
public abstract record CsrfValidationResult
{
    private CsrfValidationResult() { }
    
    public sealed record Valid(CsrfToken Token) : CsrfValidationResult;
    
    public sealed record Invalid(CsrfFailureReason Reason) : CsrfValidationResult;
}

public enum CsrfFailureReason
{
    TokenMissing,
    TokenExpired,
    TokenInvalid,
    SessionMismatch
}

/// <summary>
/// Session identifier for binding CSRF tokens to sessions.
/// </summary>
public readonly record struct SessionId
{
    public string Value { get; }
    
    private SessionId(string value) => Value = value;
    
    public static SessionId New() => new(Guid.NewGuid().ToString("N"));
    
    public static SessionId Parse(string value)
    {
        if (string.IsNullOrWhiteSpace(value))
            throw new ArgumentException("Session ID cannot be empty");
        return new SessionId(value);
    }
}

public interface ICsrfTokenService
{
    /// <summary>
    /// Generates a new CSRF token for the session.
    /// </summary>
    CsrfToken GenerateToken(SessionId sessionId);
    
    /// <summary>
    /// Validates a CSRF token from a request.
    /// </summary>
    CsrfValidationResult ValidateToken(string tokenValue, SessionId sessionId);
}

public class CsrfTokenService : ICsrfTokenService
{
    private readonly IDataProtectionProvider _dataProtection;
    private readonly TimeSpan _tokenLifetime;
    
    public CsrfTokenService(
        IDataProtectionProvider dataProtection,
        TimeSpan? tokenLifetime = null)
    {
        _dataProtection = dataProtection;
        _tokenLifetime = tokenLifetime ?? TimeSpan.FromHours(2);
    }
    
    public CsrfToken GenerateToken(SessionId sessionId)
    {
        var protector = _dataProtection.CreateProtector("CsrfToken");
        
        var payload = new CsrfTokenPayload
        {
            SessionId = sessionId.Value,
            GeneratedAt = DateTime.UtcNow,
            Nonce = Guid.NewGuid().ToString("N")
        };
        
        var serialized = JsonSerializer.Serialize(payload);
        var tokenValue = protector.Protect(serialized);
        
        return new CsrfToken
        {
            TokenValue = tokenValue,
            GeneratedAt = payload.GeneratedAt,
            ExpiresAt = payload.GeneratedAt.Add(_tokenLifetime),
            SessionId = sessionId
        };
    }
    
    public CsrfValidationResult ValidateToken(string tokenValue, SessionId sessionId)
    {
        if (string.IsNullOrWhiteSpace(tokenValue))
            return new CsrfValidationResult.Invalid(CsrfFailureReason.TokenMissing);
        
        try
        {
            var protector = _dataProtection.CreateProtector("CsrfToken");
            var decrypted = protector.Unprotect(tokenValue);
            var payload = JsonSerializer.Deserialize<CsrfTokenPayload>(decrypted)!;
            
            if (payload.SessionId != sessionId.Value)
                return new CsrfValidationResult.Invalid(CsrfFailureReason.SessionMismatch);
            
            var expiresAt = payload.GeneratedAt.Add(_tokenLifetime);
            if (DateTime.UtcNow > expiresAt)
                return new CsrfValidationResult.Invalid(CsrfFailureReason.TokenExpired);
            
            return new CsrfValidationResult.Valid(new CsrfToken
            {
                TokenValue = tokenValue,
                GeneratedAt = payload.GeneratedAt,
                ExpiresAt = expiresAt,
                SessionId = sessionId
            });
        }
        catch (CryptographicException)
        {
            return new CsrfValidationResult.Invalid(CsrfFailureReason.TokenInvalid);
        }
    }
    
    private sealed record CsrfTokenPayload
    {
        public required string SessionId { get; init; }
        public required DateTime GeneratedAt { get; init; }
        public required string Nonce { get; init; }
    }
}

// Middleware validates CSRF tokens for state-changing requests
public class CsrfMiddleware
{
    private readonly RequestDelegate _next;
    
    public async Task InvokeAsync(
        HttpContext context,
        ICsrfTokenService csrfService)
    {
        // Only validate state-changing methods
        if (context.Request.Method is "POST" or "PUT" or "DELETE" or "PATCH")
        {
            var sessionId = GetOrCreateSessionId(context);
            
            // Try header first (for AJAX), then form
            var tokenValue = context.Request.Headers["X-CSRF-Token"].FirstOrDefault()
                ?? context.Request.Form["__RequestVerificationToken"].FirstOrDefault();
            
            var result = csrfService.ValidateToken(tokenValue ?? "", sessionId);
            
            if (result is CsrfValidationResult.Invalid invalid)
            {
                context.Response.StatusCode = 403;
                await context.Response.WriteAsJsonAsync(new
                {
                    error = "csrf_validation_failed",
                    reason = invalid.Reason.ToString()
                });
                return;
            }
            
            // Store validated token in HttpContext for controller access
            var valid = (CsrfValidationResult.Valid)result;
            context.Items["CsrfToken"] = valid.Token;
        }
        
        await _next(context);
    }
    
    private SessionId GetOrCreateSessionId(HttpContext context)
    {
        const string SessionIdKey = "SessionId";
        
        if (context.Session.TryGetValue(SessionIdKey, out var sessionBytes))
        {
            var sessionIdString = Encoding.UTF8.GetString(sessionBytes);
            return SessionId.Parse(sessionIdString);
        }
        
        var newSessionId = SessionId.New();
        context.Session.Set(SessionIdKey, Encoding.UTF8.GetBytes(newSessionId.Value));
        return newSessionId;
    }
}

// Extension to get validated token
public static class HttpContextExtensions
{
    public static CsrfToken GetCsrfToken(this HttpContext context)
    {
        return context.Items["CsrfToken"] as CsrfToken
            ?? throw new InvalidOperationException("CSRF token not validated");
    }
}

// Service methods require CSRF token as proof
public interface IBankService
{
    Task TransferAsync(
        CsrfToken csrfToken,
        AccountId from,
        AccountId to,
        Money amount);
}

public class BankService : IBankService
{
    public async Task TransferAsync(
        CsrfToken csrfToken,
        AccountId from,
        AccountId to,
        Money amount)
    {
        // Type system guarantees CSRF validation happened
        // Token proves this is not a forged request
        
        if (DateTime.UtcNow > csrfToken.ExpiresAt)
            throw new SecurityException("CSRF token expired");
        
        await PerformTransferAsync(from, to, amount);
    }
}

// Controller must acquire and pass CSRF token
[ApiController]
[Route("api/transfer")]
public class TransferController : ControllerBase
{
    private readonly IBankService _bankService;
    
    [HttpPost]
    public async Task<IActionResult> Transfer([FromBody] TransferRequest request)
    {
        // Get validated token from middleware
        var csrfToken = HttpContext.GetCsrfToken();
        
        var user = GetCurrentUser();
        await _bankService.TransferAsync(
            csrfToken,
            user.AccountId,
            AccountId.Parse(request.ToAccount),
            Money.Create(request.Amount, Currency.USD));
        
        return Ok();
    }
}

// Razor Pages integration
public class TransferModel : PageModel
{
    private readonly ICsrfTokenService _csrfService;
    
    public string CsrfToken { get; private set; } = "";
    
    public void OnGet()
    {
        var sessionId = GetSessionId();
        var token = _csrfService.GenerateToken(sessionId);
        CsrfToken = token.TokenValue;
    }
}

// In the view:
// <form method="post">
//   <input type="hidden" name="__RequestVerificationToken" value="@Model.CsrfToken" />
//   <!-- form fields -->
// </form>
```

## Double-Submit Cookie Pattern

An alternative approach using cookies:

```csharp
public class DoubleSubmitCsrfService : ICsrfTokenService
{
    private const string CookieName = "__Host-CSRF-TOKEN";
    
    public CsrfToken GenerateToken(SessionId sessionId)
    {
        var tokenValue = Convert.ToBase64String(
            RandomNumberGenerator.GetBytes(32));
        
        return new CsrfToken
        {
            TokenValue = tokenValue,
            GeneratedAt = DateTime.UtcNow,
            ExpiresAt = DateTime.UtcNow.AddHours(2),
            SessionId = sessionId
        };
    }
    
    public CsrfValidationResult ValidateToken(string tokenValue, SessionId sessionId)
    {
        if (string.IsNullOrWhiteSpace(tokenValue))
            return new CsrfValidationResult.Invalid(CsrfFailureReason.TokenMissing);
        
        // In middleware: compare cookie value with header/form value
        // They must match (both set by the client)
        return new CsrfValidationResult.Valid(new CsrfToken
        {
            TokenValue = tokenValue,
            GeneratedAt = DateTime.UtcNow,
            ExpiresAt = DateTime.UtcNow.AddHours(2),
            SessionId = sessionId
        });
    }
}

public class DoubleSubmitCsrfMiddleware
{
    public async Task InvokeAsync(HttpContext context, ICsrfTokenService csrfService)
    {
        // Set cookie on first request
        if (!context.Request.Cookies.ContainsKey(CookieName))
        {
            var sessionId = GetOrCreateSessionId(context);
            var token = csrfService.GenerateToken(sessionId);
            
            context.Response.Cookies.Append(CookieName, token.TokenValue, new CookieOptions
            {
                HttpOnly = true,
                Secure = true,
                SameSite = SameSiteMode.Strict,
                IsEssential = true
            });
        }
        
        // Validate state-changing requests
        if (context.Request.Method is "POST" or "PUT" or "DELETE" or "PATCH")
        {
            var cookieToken = context.Request.Cookies[CookieName];
            var headerToken = context.Request.Headers["X-CSRF-Token"].FirstOrDefault();
            
            if (cookieToken != headerToken || string.IsNullOrEmpty(cookieToken))
            {
                context.Response.StatusCode = 403;
                await context.Response.WriteAsJsonAsync(new { error = "csrf_validation_failed" });
                return;
            }
            
            var sessionId = GetOrCreateSessionId(context);
            var result = csrfService.ValidateToken(cookieToken, sessionId);
            
            if (result is CsrfValidationResult.Valid valid)
                context.Items["CsrfToken"] = valid.Token;
            else
            {
                context.Response.StatusCode = 403;
                return;
            }
        }
        
        await _next(context);
    }
}
```

## SPA/AJAX Integration

```csharp
// API endpoint to get CSRF token for JavaScript apps
[ApiController]
[Route("api/csrf")]
public class CsrfController : ControllerBase
{
    private readonly ICsrfTokenService _csrfService;
    
    [HttpGet("token")]
    public IActionResult GetToken()
    {
        var sessionId = GetOrCreateSessionId();
        var token = _csrfService.GenerateToken(sessionId);
        
        return Ok(new
        {
            token = token.TokenValue,
            expiresAt = token.ExpiresAt
        });
    }
}

// JavaScript usage:
// const response = await fetch('/api/csrf/token');
// const { token } = await response.json();
// 
// await fetch('/api/transfer', {
//   method: 'POST',
//   headers: {
//     'X-CSRF-Token': token,
//     'Content-Type': 'application/json'
//   },
//   body: JSON.stringify(data)
// });
```

## Testing

```csharp
public class CsrfProtectionTests
{
    [Fact]
    public void GenerateToken_CreatesValidToken()
    {
        var service = CreateService();
        var sessionId = SessionId.New();
        
        var token = service.GenerateToken(sessionId);
        
        Assert.NotNull(token.TokenValue);
        Assert.Equal(sessionId, token.SessionId);
        Assert.True(token.ExpiresAt > DateTime.UtcNow);
    }
    
    [Fact]
    public void ValidateToken_WithValidToken_ReturnsValid()
    {
        var service = CreateService();
        var sessionId = SessionId.New();
        var token = service.GenerateToken(sessionId);
        
        var result = service.ValidateToken(token.TokenValue, sessionId);
        
        Assert.IsType<CsrfValidationResult.Valid>(result);
    }
    
    [Fact]
    public void ValidateToken_WithWrongSession_ReturnsInvalid()
    {
        var service = CreateService();
        var sessionId1 = SessionId.New();
        var sessionId2 = SessionId.New();
        var token = service.GenerateToken(sessionId1);
        
        var result = service.ValidateToken(token.TokenValue, sessionId2);
        
        var invalid = Assert.IsType<CsrfValidationResult.Invalid>(result);
        Assert.Equal(CsrfFailureReason.SessionMismatch, invalid.Reason);
    }
    
    [Fact]
    public void BankService_WithoutToken_DoesNotCompile()
    {
        var service = new BankService();
        
        // This won't compile—CSRF token required!
        // await service.TransferAsync(from, to, amount);
        
        // Must have a validated token
        var csrfService = CreateService();
        var sessionId = SessionId.New();
        var token = csrfService.GenerateToken(sessionId);
        
        await service.TransferAsync(token, from, to, amount);
    }
}
```

## Why It's a Problem

1. **Easy to forget**: Developers must remember to add CSRF protection to every state-changing endpoint
2. **String-based tokens**: No type safety around token validation
3. **Runtime-only**: No compile-time guarantee CSRF check happened
4. **Inconsistent application**: Some endpoints protected, others forgotten
5. **Testing gaps**: Tests often skip CSRF validation

## Symptoms

- Security audits find unprotected endpoints
- `[ValidateAntiForgeryToken]` attribute forgotten on new endpoints
- Manual token validation scattered across controllers
- String-based token passing with no validation
- CSRF attacks discovered in production

## Benefits

- **Compile-time enforcement**: Cannot call sensitive methods without CSRF token
- **Type-safe tokens**: `CsrfToken` type proves validation occurred
- **Self-documenting**: Method signatures show CSRF requirement
- **Consistent protection**: Middleware ensures all state-changing requests are validated
- **Testable**: Easy to test CSRF validation logic in isolation

## Trade-offs

- **More ceremony**: Must pass CSRF token explicitly
- **Session required**: CSRF tokens are typically bound to sessions
- **SameSite cookies**: Modern browsers provide some CSRF protection via SameSite cookies
- **Complexity**: More complex than simple attribute-based validation

## See Also

- [Capability Security](./capability-security.md) — authorization tokens
- [Authentication Context](./authentication-context.md) — user identity tokens
- [Secret Types](./secret-types.md) — preventing token exposure
- [Input Sanitization](./input-sanitization.md) — trusted types at boundaries

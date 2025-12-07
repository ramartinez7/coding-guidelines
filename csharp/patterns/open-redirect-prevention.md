# Open Redirect Prevention (Safe URL Redirects)

> Redirecting to URLs from untrusted input—use validated URL types to prevent open redirect attacks that enable phishing.

## Problem

Open redirect vulnerabilities occur when applications redirect users to URLs controlled by attackers. This enables phishing attacks where victims think they're visiting a legitimate site. String-based URL handling provides no compile-time protection against redirect attacks.

## Example

### ❌ Before

```csharp
public class AuthController : Controller
{
    public IActionResult Login(string returnUrl)
    {
        // Dangerous: open redirect vulnerability!
        // Attacker URL: /login?returnUrl=https://evil.com/fake-login
        // User logs in, gets redirected to attacker's site
        
        if (User.Identity?.IsAuthenticated == true)
        {
            return Redirect(returnUrl);  // 💥 Redirects to evil.com
        }
        
        return View();
    }
    
    public IActionResult Logout(string redirectTo)
    {
        // Vulnerable: redirectTo could be any URL
        SignOut();
        
        if (!string.IsNullOrEmpty(redirectTo))
        {
            return Redirect(redirectTo);
        }
        
        return RedirectToAction("Index", "Home");
    }
    
    public IActionResult ExternalLink(string url)
    {
        // Dangerous: no validation
        return Redirect(url);
    }
}
```

**Problems:**
- User input directly used in redirects
- No URL validation
- Enables phishing attacks
- Can bypass security controls
- No compile-time detection

### ✅ After

```csharp
/// <summary>
/// A validated redirect URL that has been checked for safety.
/// Cannot be constructed from untrusted strings without validation.
/// </summary>
public abstract record RedirectUrl
{
    private RedirectUrl() { }
    
    /// <summary>
    /// Redirect to a local path within the application.
    /// </summary>
    public sealed record Local(string Path) : RedirectUrl
    {
        public static Result<Local, string> Create(string path)
        {
            if (string.IsNullOrWhiteSpace(path))
                return Result<Local, string>.Failure("Path cannot be empty");
            
            // Must start with / for local paths
            if (!path.StartsWith("/"))
                return Result<Local, string>.Failure(
                    "Local path must start with /");
            
            // Prevent protocol-relative URLs (//evil.com)
            if (path.StartsWith("//"))
                return Result<Local, string>.Failure(
                    "Protocol-relative URLs not allowed");
            
            // Prevent data URLs
            if (path.StartsWith("data:", StringComparison.OrdinalIgnoreCase))
                return Result<Local, string>.Failure(
                    "Data URLs not allowed");
            
            // Prevent javascript URLs
            if (path.StartsWith("javascript:", StringComparison.OrdinalIgnoreCase))
                return Result<Local, string>.Failure(
                    "JavaScript URLs not allowed");
            
            try
            {
                // Validate it's a valid URI path
                var uri = new Uri(path, UriKind.Relative);
                
                return Result<Local, string>.Success(new Local(path));
            }
            catch (Exception ex)
            {
                return Result<Local, string>.Failure($"Invalid path: {ex.Message}");
            }
        }
    }
    
    /// <summary>
    /// Redirect to an allowed external domain.
    /// </summary>
    public sealed record External(string Url, AllowedDomain Domain) : RedirectUrl
    {
        public static Result<External, string> Create(
            string url,
            AllowedDomainRegistry registry)
        {
            if (string.IsNullOrWhiteSpace(url))
                return Result<External, string>.Failure("URL cannot be empty");
            
            if (!Uri.TryCreate(url, UriKind.Absolute, out var uri))
                return Result<External, string>.Failure("Invalid URL format");
            
            // Only allow HTTP/HTTPS
            if (uri.Scheme != Uri.UriSchemeHttp && uri.Scheme != Uri.UriSchemeHttps)
                return Result<External, string>.Failure(
                    $"Scheme '{uri.Scheme}' not allowed. Use http or https.");
            
            // Check if domain is allowed
            var domainResult = registry.GetAllowedDomain(uri.Host);
            
            if (!domainResult.IsSuccess)
                return Result<External, string>.Failure(domainResult.Error!);
            
            return Result<External, string>.Success(
                new External(url, domainResult.Value!));
        }
    }
    
    public string GetUrl() => this switch
    {
        Local local => local.Path,
        External external => external.Url,
        _ => throw new UnreachableException()
    };
}

/// <summary>
/// An allowed external domain for redirects.
/// </summary>
public sealed record AllowedDomain
{
    public string Domain { get; }
    public string Description { get; }
    
    private AllowedDomain(string domain, string description)
    {
        Domain = domain;
        Description = description;
    }
    
    public static Result<AllowedDomain, string> Create(string domain, string description)
    {
        if (string.IsNullOrWhiteSpace(domain))
            return Result<AllowedDomain, string>.Failure("Domain cannot be empty");
        
        // Validate domain format
        if (!Uri.CheckHostName(domain).HasFlag(UriHostNameType.Dns))
            return Result<AllowedDomain, string>.Failure("Invalid domain format");
        
        return Result<AllowedDomain, string>.Success(
            new AllowedDomain(domain.ToLowerInvariant(), description));
    }
    
    public bool Matches(string hostname)
    {
        var normalizedHost = hostname.ToLowerInvariant();
        
        // Exact match
        if (normalizedHost == Domain)
            return true;
        
        // Subdomain match (e.g., *.example.com)
        if (Domain.StartsWith("*."))
        {
            var domainSuffix = Domain[2..]; // Remove "*."
            return normalizedHost == domainSuffix || 
                   normalizedHost.EndsWith("." + domainSuffix);
        }
        
        return false;
    }
}

/// <summary>
/// Registry of allowed external domains for redirects.
/// </summary>
public sealed class AllowedDomainRegistry
{
    private readonly List<AllowedDomain> _allowedDomains = new();
    
    public void Register(AllowedDomain domain)
    {
        _allowedDomains.Add(domain);
    }
    
    public Result<AllowedDomain, string> GetAllowedDomain(string hostname)
    {
        var domain = _allowedDomains.FirstOrDefault(d => d.Matches(hostname));
        
        if (domain == null)
            return Result<AllowedDomain, string>.Failure(
                $"Domain '{hostname}' is not in the allowlist");
        
        return Result<AllowedDomain, string>.Success(domain);
    }
    
    public static AllowedDomainRegistry FromConfiguration(IConfiguration configuration)
    {
        var registry = new AllowedDomainRegistry();
        
        var domains = configuration.GetSection("AllowedRedirectDomains").Get<string[]>() 
            ?? Array.Empty<string>();
        
        foreach (var domain in domains)
        {
            var domainResult = AllowedDomain.Create(domain, $"Configured domain: {domain}");
            
            if (domainResult.IsSuccess)
                registry.Register(domainResult.Value!);
        }
        
        return registry;
    }
}

public class AuthController : Controller
{
    private readonly AllowedDomainRegistry _domainRegistry;
    
    public AuthController(AllowedDomainRegistry domainRegistry)
    {
        _domainRegistry = domainRegistry;
    }
    
    public IActionResult Login(string? returnUrl)
    {
        if (User.Identity?.IsAuthenticated == true)
        {
            // Validate redirect URL before using it
            if (!string.IsNullOrEmpty(returnUrl))
            {
                var redirectResult = ValidateRedirectUrl(returnUrl);
                
                return redirectResult.Match(
                    onSuccess: url => Redirect(url.GetUrl()),
                    onFailure: _ => RedirectToAction("Index", "Home"));
            }
            
            return RedirectToAction("Index", "Home");
        }
        
        return View();
    }
    
    public IActionResult Logout(string? redirectTo)
    {
        SignOut();
        
        if (!string.IsNullOrEmpty(redirectTo))
        {
            var redirectResult = ValidateRedirectUrl(redirectTo);
            
            return redirectResult.Match(
                onSuccess: url => Redirect(url.GetUrl()),
                onFailure: _ => RedirectToAction("Index", "Home"));
        }
        
        return RedirectToAction("Index", "Home");
    }
    
    private Result<RedirectUrl, string> ValidateRedirectUrl(string url)
    {
        // Try as local path first
        var localResult = RedirectUrl.Local.Create(url);
        if (localResult.IsSuccess)
            return Result<RedirectUrl, string>.Success(localResult.Value!);
        
        // Try as external URL
        var externalResult = RedirectUrl.External.Create(url, _domainRegistry);
        if (externalResult.IsSuccess)
            return Result<RedirectUrl, string>.Success(externalResult.Value!);
        
        return Result<RedirectUrl, string>.Failure(
            "URL is neither a valid local path nor an allowed external domain");
    }
}
```

## Advanced Patterns

### Action Result Helper

```csharp
/// <summary>
/// Extension methods for safe redirects in controllers.
/// </summary>
public static class RedirectExtensions
{
    public static IActionResult SafeRedirect(
        this Controller controller,
        string? url,
        AllowedDomainRegistry registry)
    {
        if (string.IsNullOrEmpty(url))
            return controller.RedirectToAction("Index", "Home");
        
        var redirectResult = ValidateUrl(url, registry);
        
        return redirectResult.Match(
            onSuccess: redirect => controller.Redirect(redirect.GetUrl()),
            onFailure: _ => controller.RedirectToAction("Index", "Home"));
    }
    
    public static IActionResult SafeRedirectOrDefault(
        this Controller controller,
        string? url,
        AllowedDomainRegistry registry,
        string defaultAction,
        string defaultController)
    {
        if (string.IsNullOrEmpty(url))
            return controller.RedirectToAction(defaultAction, defaultController);
        
        var redirectResult = ValidateUrl(url, registry);
        
        return redirectResult.Match(
            onSuccess: redirect => controller.Redirect(redirect.GetUrl()),
            onFailure: _ => controller.RedirectToAction(defaultAction, defaultController));
    }
    
    private static Result<RedirectUrl, string> ValidateUrl(
        string url,
        AllowedDomainRegistry registry)
    {
        var localResult = RedirectUrl.Local.Create(url);
        if (localResult.IsSuccess)
            return Result<RedirectUrl, string>.Success(localResult.Value!);
        
        var externalResult = RedirectUrl.External.Create(url, registry);
        if (externalResult.IsSuccess)
            return Result<RedirectUrl, string>.Success(externalResult.Value!);
        
        return Result<RedirectUrl, string>.Failure("Invalid redirect URL");
    }
}

// Usage
public IActionResult Login(string? returnUrl)
{
    if (User.Identity?.IsAuthenticated == true)
    {
        return this.SafeRedirectOrDefault(returnUrl, _domainRegistry, "Index", "Home");
    }
    
    return View();
}
```

### Redirect with Confirmation

```csharp
/// <summary>
/// External redirect that requires user confirmation.
/// </summary>
public sealed record ConfirmedRedirect
{
    public RedirectUrl.External TargetUrl { get; }
    public bool Confirmed { get; }
    
    private ConfirmedRedirect(RedirectUrl.External targetUrl, bool confirmed)
    {
        TargetUrl = targetUrl;
        Confirmed = confirmed;
    }
    
    public static ConfirmedRedirect Create(RedirectUrl.External url)
    {
        return new ConfirmedRedirect(url, false);
    }
    
    public ConfirmedRedirect WithConfirmation()
    {
        return new ConfirmedRedirect(TargetUrl, true);
    }
}

public class LinkController : Controller
{
    public IActionResult ExternalLink(string url, bool confirmed = false)
    {
        var urlResult = RedirectUrl.External.Create(url, _domainRegistry);
        
        if (!urlResult.IsSuccess)
            return BadRequest(urlResult.Error);
        
        if (!confirmed)
        {
            // Show confirmation page
            return View("ConfirmRedirect", new ConfirmRedirectViewModel
            {
                TargetUrl = url,
                Domain = urlResult.Value!.Domain.Domain
            });
        }
        
        // User confirmed, perform redirect
        return Redirect(url);
    }
}
```

### URL Allowlist from Database

```csharp
/// <summary>
/// Dynamic allowlist loaded from database.
/// </summary>
public interface IAllowedDomainRepository
{
    Task<List<AllowedDomain>> GetAllowedDomainsAsync();
    Task<bool> IsAllowedAsync(string hostname);
}

public sealed class CachedDomainRegistry
{
    private readonly IAllowedDomainRepository _repository;
    private readonly IMemoryCache _cache;
    private readonly TimeSpan _cacheExpiration = TimeSpan.FromHours(1);
    
    public CachedDomainRegistry(
        IAllowedDomainRepository repository,
        IMemoryCache cache)
    {
        _repository = repository;
        _cache = cache;
    }
    
    public async Task<Result<AllowedDomain, string>> GetAllowedDomainAsync(
        string hostname)
    {
        var domains = await GetCachedDomainsAsync();
        
        var domain = domains.FirstOrDefault(d => d.Matches(hostname));
        
        if (domain == null)
            return Result<AllowedDomain, string>.Failure(
                $"Domain '{hostname}' is not in the allowlist");
        
        return Result<AllowedDomain, string>.Success(domain);
    }
    
    private async Task<List<AllowedDomain>> GetCachedDomainsAsync()
    {
        return await _cache.GetOrCreateAsync("AllowedDomains", async entry =>
        {
            entry.AbsoluteExpirationRelativeToNow = _cacheExpiration;
            return await _repository.GetAllowedDomainsAsync();
        }) ?? new List<AllowedDomain>();
    }
}
```

### Signed Redirect Tokens

```csharp
/// <summary>
/// Cryptographically signed redirect token.
/// Prevents tampering with return URLs.
/// </summary>
public sealed record SignedRedirectToken
{
    public string Token { get; }
    public DateTime ExpiresAt { get; }
    
    private SignedRedirectToken(string token, DateTime expiresAt)
    {
        Token = token;
        ExpiresAt = expiresAt;
    }
    
    public static SignedRedirectToken Create(
        RedirectUrl redirectUrl,
        IDataProtectionProvider dataProtection)
    {
        var protector = dataProtection.CreateProtector("RedirectTokens");
        
        var payload = new RedirectTokenPayload
        {
            Url = redirectUrl.GetUrl(),
            CreatedAt = DateTime.UtcNow,
            ExpiresAt = DateTime.UtcNow.AddMinutes(15)
        };
        
        var json = System.Text.Json.JsonSerializer.Serialize(payload);
        var token = protector.Protect(json);
        
        return new SignedRedirectToken(token, payload.ExpiresAt);
    }
    
    public static Result<RedirectUrl, string> Verify(
        string token,
        IDataProtectionProvider dataProtection,
        AllowedDomainRegistry registry)
    {
        try
        {
            var protector = dataProtection.CreateProtector("RedirectTokens");
            var json = protector.Unprotect(token);
            
            var payload = System.Text.Json.JsonSerializer.Deserialize<RedirectTokenPayload>(json);
            
            if (payload == null)
                return Result<RedirectUrl, string>.Failure("Invalid token");
            
            if (DateTime.UtcNow > payload.ExpiresAt)
                return Result<RedirectUrl, string>.Failure("Token expired");
            
            // Re-validate the URL
            var localResult = RedirectUrl.Local.Create(payload.Url);
            if (localResult.IsSuccess)
                return Result<RedirectUrl, string>.Success(localResult.Value!);
            
            var externalResult = RedirectUrl.External.Create(payload.Url, registry);
            if (externalResult.IsSuccess)
                return Result<RedirectUrl, string>.Success(externalResult.Value!);
            
            return Result<RedirectUrl, string>.Failure("URL validation failed");
        }
        catch (CryptographicException)
        {
            return Result<RedirectUrl, string>.Failure("Invalid or tampered token");
        }
    }
    
    private sealed record RedirectTokenPayload
    {
        public required string Url { get; init; }
        public required DateTime CreatedAt { get; init; }
        public required DateTime ExpiresAt { get; init; }
    }
}
```

## Testing

```csharp
public class OpenRedirectTests
{
    private readonly AllowedDomainRegistry _registry;
    
    public OpenRedirectTests()
    {
        _registry = new AllowedDomainRegistry();
        _registry.Register(AllowedDomain.Create("example.com", "Test domain").Value!);
        _registry.Register(AllowedDomain.Create("*.trusted.com", "Trusted subdomains").Value!);
    }
    
    [Fact]
    public void Local_WithValidPath_Succeeds()
    {
        var result = RedirectUrl.Local.Create("/dashboard");
        
        Assert.True(result.IsSuccess);
        Assert.Equal("/dashboard", result.Value!.Path);
    }
    
    [Fact]
    public void Local_WithProtocolRelativeUrl_Fails()
    {
        var result = RedirectUrl.Local.Create("//evil.com");
        
        Assert.False(result.IsSuccess);
        Assert.Contains("protocol-relative", result.Error!.ToLower());
    }
    
    [Fact]
    public void Local_WithJavaScriptUrl_Fails()
    {
        var result = RedirectUrl.Local.Create("javascript:alert('xss')");
        
        Assert.False(result.IsSuccess);
    }
    
    [Fact]
    public void External_WithAllowedDomain_Succeeds()
    {
        var result = RedirectUrl.External.Create(
            "https://example.com/page", 
            _registry);
        
        Assert.True(result.IsSuccess);
    }
    
    [Fact]
    public void External_WithDisallowedDomain_Fails()
    {
        var result = RedirectUrl.External.Create(
            "https://evil.com/phishing", 
            _registry);
        
        Assert.False(result.IsSuccess);
        Assert.Contains("not in the allowlist", result.Error!);
    }
    
    [Fact]
    public void External_WithWildcardSubdomain_Succeeds()
    {
        var result = RedirectUrl.External.Create(
            "https://api.trusted.com/callback", 
            _registry);
        
        Assert.True(result.IsSuccess);
    }
    
    [Theory]
    [InlineData("ftp://example.com")]
    [InlineData("file:///etc/passwd")]
    [InlineData("data:text/html,<script>alert()</script>")]
    public void External_WithNonHttpScheme_Fails(string url)
    {
        var result = RedirectUrl.External.Create(url, _registry);
        
        Assert.False(result.IsSuccess);
    }
}
```

## Why It's a Problem

1. **Phishing attacks**: Users redirected to fake login pages
2. **Trust exploitation**: Users trust the legitimate domain in the URL
3. **Session hijacking**: Attackers can steal authentication tokens
4. **Malware distribution**: Redirect to malware download sites
5. **No compile-time detection**: String-based redirects look innocent

## Symptoms

- Using `Redirect()` or `RedirectToAction()` with user input
- No URL validation before redirects
- Allowing arbitrary external URLs
- No domain allowlist for external redirects
- Login/logout endpoints with `returnUrl` parameters

## Benefits

- **Phishing prevention**: Only allowed domains can be redirect targets
- **Compile-time safety**: `RedirectUrl` type enforces validation
- **Self-documenting**: Makes redirect security explicit
- **Flexible allowlist**: Supports wildcards and dynamic domains
- **Signed tokens**: Prevents URL tampering

## Trade-offs

- **More verbose**: Redirect validation requires explicit checks
- **Domain management**: Allowlist must be maintained
- **User experience**: May need confirmation for external redirects
- **Configuration**: Requires domain allowlist setup

## See Also

- [Input Sanitization](./input-sanitization.md) — trusted types at boundaries
- [CSRF Protection](./csrf-protection.md) — preventing forged requests
- [Authentication Context](./authentication-context.md) — user identity validation
- [Type-Safe String Interpolation](./type-safe-string-interpolation.md) — preventing injection

# Security Headers (Type-Safe HTTP Security)

> Setting security headers with string literals—use typed security headers to enforce security policies at compile time.

## Problem

HTTP security headers (CSP, HSTS, X-Frame-Options, etc.) are set using string values, making them error-prone. Typos, invalid directives, and inconsistent policies across endpoints are common. There's no compile-time validation of header correctness.

## Example

### ❌ Before

```csharp
public class SecurityHeadersMiddleware
{
    public async Task InvokeAsync(HttpContext context)
    {
        // String-based headers—typos not caught until runtime
        context.Response.Headers.Add("Content-Security-Policy",
            "default-src 'self'; script-src 'self' 'unsafe-inline'");  // Oops! 'unsafe-inline' defeats CSP
        
        context.Response.Headers.Add("X-Frame-Options", "SAMEORIGIN");
        context.Response.Headers.Add("X-Content-Type-Options", "nosniff");
        context.Response.Headers.Add("Strict-Transport-Security",
            "max-age=31536000; includeSubDomains");
        
        // Easy to forget headers or set wrong values
        await _next(context);
    }
}

// Different endpoints might have different policies (inconsistent)
public class ApiController : ControllerBase
{
    public IActionResult GetData()
    {
        Response.Headers.Add("Content-Security-Policy", "default-src *");  // Too permissive!
        return Ok(data);
    }
}
```

**Problems:**
- String-based headers prone to typos
- No compile-time validation
- Inconsistent policies across endpoints
- Easy to set overly permissive or conflicting directives
- No type safety

### ✅ After

```csharp
/// <summary>
/// Content Security Policy configuration.
/// </summary>
public sealed record ContentSecurityPolicy
{
    public CspDirective DefaultSrc { get; init; } = CspDirective.Self();
    public CspDirective ScriptSrc { get; init; } = CspDirective.Self();
    public CspDirective StyleSrc { get; init; } = CspDirective.Self();
    public CspDirective ImgSrc { get; init; } = CspDirective.Self();
    public CspDirective ConnectSrc { get; init; } = CspDirective.Self();
    public CspDirective FontSrc { get; init; } = CspDirective.Self();
    public CspDirective ObjectSrc { get; init; } = CspDirective.None();
    public CspDirective FrameSrc { get; init; } = CspDirective.None();
    public CspDirective BaseUri { get; init; } = CspDirective.Self();
    public CspDirective FormAction { get; init; } = CspDirective.Self();
    public bool UpgradeInsecureRequests { get; init; } = true;
    
    public string ToHeaderValue()
    {
        var directives = new List<string>();
        
        AddDirective(directives, "default-src", DefaultSrc);
        AddDirective(directives, "script-src", ScriptSrc);
        AddDirective(directives, "style-src", StyleSrc);
        AddDirective(directives, "img-src", ImgSrc);
        AddDirective(directives, "connect-src", ConnectSrc);
        AddDirective(directives, "font-src", FontSrc);
        AddDirective(directives, "object-src", ObjectSrc);
        AddDirective(directives, "frame-src", FrameSrc);
        AddDirective(directives, "base-uri", BaseUri);
        AddDirective(directives, "form-action", FormAction);
        
        if (UpgradeInsecureRequests)
            directives.Add("upgrade-insecure-requests");
        
        return string.Join("; ", directives);
    }
    
    private void AddDirective(List<string> directives, string name, CspDirective directive)
    {
        if (directive.Sources.Count > 0)
            directives.Add($"{name} {string.Join(" ", directive.Sources)}");
    }
    
    public static ContentSecurityPolicy Strict => new()
    {
        DefaultSrc = CspDirective.Self(),
        ScriptSrc = CspDirective.Self(),
        StyleSrc = CspDirective.Self(),
        ImgSrc = CspDirective.Self(),
        ObjectSrc = CspDirective.None(),
        FrameSrc = CspDirective.None(),
        BaseUri = CspDirective.Self(),
        FormAction = CspDirective.Self(),
        UpgradeInsecureRequests = true
    };
    
    public static ContentSecurityPolicy AllowCdnAssets => new()
    {
        DefaultSrc = CspDirective.Self(),
        ScriptSrc = CspDirective.Self()
            .AllowSource("https://cdn.jsdelivr.net")
            .AllowSource("https://cdnjs.cloudflare.com"),
        StyleSrc = CspDirective.Self()
            .AllowSource("https://cdn.jsdelivr.net")
            .AllowUnsafeInline(),  // For inline styles (use sparingly)
        ImgSrc = CspDirective.Self()
            .AllowSource("data:")
            .AllowSource("https:"),
        ObjectSrc = CspDirective.None(),
        UpgradeInsecureRequests = true
    };
}

/// <summary>
/// CSP directive with type-safe source configuration.
/// </summary>
public sealed record CspDirective
{
    public IReadOnlySet<string> Sources { get; }
    
    private CspDirective(IReadOnlySet<string> sources)
    {
        Sources = sources;
    }
    
    public static CspDirective Self() => new(new HashSet<string> { "'self'" });
    
    public static CspDirective None() => new(new HashSet<string> { "'none'" });
    
    public static CspDirective All() => new(new HashSet<string> { "*" });
    
    public CspDirective AllowSource(string source)
    {
        var newSources = new HashSet<string>(Sources) { source };
        return new CspDirective(newSources);
    }
    
    public CspDirective AllowUnsafeInline()
    {
        var newSources = new HashSet<string>(Sources) { "'unsafe-inline'" };
        return new CspDirective(newSources);
    }
    
    public CspDirective AllowUnsafeEval()
    {
        var newSources = new HashSet<string>(Sources) { "'unsafe-eval'" };
        return new CspDirective(newSources);
    }
    
    public CspDirective AllowNonce(string nonce)
    {
        var newSources = new HashSet<string>(Sources) { $"'nonce-{nonce}'" };
        return new CspDirective(newSources);
    }
    
    public CspDirective AllowHash(string algorithm, string hash)
    {
        var newSources = new HashSet<string>(Sources) { $"'{algorithm}-{hash}'" };
        return new CspDirective(newSources);
    }
}

/// <summary>
/// Strict-Transport-Security configuration.
/// </summary>
public sealed record StrictTransportSecurity
{
    public required TimeSpan MaxAge { get; init; }
    public bool IncludeSubDomains { get; init; }
    public bool Preload { get; init; }
    
    public string ToHeaderValue()
    {
        var parts = new List<string>
        {
            $"max-age={MaxAge.TotalSeconds:F0}"
        };
        
        if (IncludeSubDomains)
            parts.Add("includeSubDomains");
        
        if (Preload)
            parts.Add("preload");
        
        return string.Join("; ", parts);
    }
    
    public static StrictTransportSecurity OneYear => new()
    {
        MaxAge = TimeSpan.FromDays(365),
        IncludeSubDomains = true,
        Preload = false
    };
    
    public static StrictTransportSecurity TwoYears => new()
    {
        MaxAge = TimeSpan.FromDays(730),
        IncludeSubDomains = true,
        Preload = true
    };
}

/// <summary>
/// X-Frame-Options configuration.
/// </summary>
public abstract record FrameOptions
{
    private FrameOptions() { }
    
    public sealed record Deny : FrameOptions
    {
        public override string ToHeaderValue() => "DENY";
    }
    
    public sealed record SameOrigin : FrameOptions
    {
        public override string ToHeaderValue() => "SAMEORIGIN";
    }
    
    public sealed record AllowFrom(Uri Uri) : FrameOptions
    {
        public override string ToHeaderValue() => $"ALLOW-FROM {Uri}";
    }
    
    public abstract string ToHeaderValue();
}

/// <summary>
/// Referrer-Policy configuration.
/// </summary>
public enum ReferrerPolicy
{
    NoReferrer,
    NoReferrerWhenDowngrade,
    Origin,
    OriginWhenCrossOrigin,
    SameOrigin,
    StrictOrigin,
    StrictOriginWhenCrossOrigin,
    UnsafeUrl
}

public static class ReferrerPolicyExtensions
{
    public static string ToHeaderValue(this ReferrerPolicy policy) => policy switch
    {
        ReferrerPolicy.NoReferrer => "no-referrer",
        ReferrerPolicy.NoReferrerWhenDowngrade => "no-referrer-when-downgrade",
        ReferrerPolicy.Origin => "origin",
        ReferrerPolicy.OriginWhenCrossOrigin => "origin-when-cross-origin",
        ReferrerPolicy.SameOrigin => "same-origin",
        ReferrerPolicy.StrictOrigin => "strict-origin",
        ReferrerPolicy.StrictOriginWhenCrossOrigin => "strict-origin-when-cross-origin",
        ReferrerPolicy.UnsafeUrl => "unsafe-url",
        _ => throw new ArgumentOutOfRangeException(nameof(policy))
    };
}

/// <summary>
/// Comprehensive security headers configuration.
/// </summary>
public sealed record SecurityHeadersConfiguration
{
    public ContentSecurityPolicy ContentSecurityPolicy { get; init; } = ContentSecurityPolicy.Strict;
    public StrictTransportSecurity StrictTransportSecurity { get; init; } = StrictTransportSecurity.OneYear;
    public FrameOptions FrameOptions { get; init; } = new FrameOptions.Deny();
    public ReferrerPolicy ReferrerPolicy { get; init; } = ReferrerPolicy.StrictOriginWhenCrossOrigin;
    public bool XContentTypeOptionsNoSniff { get; init; } = true;
    public bool XDownloadOptionsNoOpen { get; init; } = true;
    public bool XPermittedCrossDomainPoliciesNone { get; init; } = true;
    
    public void ApplyTo(HttpResponse response)
    {
        response.Headers["Content-Security-Policy"] = ContentSecurityPolicy.ToHeaderValue();
        response.Headers["Strict-Transport-Security"] = StrictTransportSecurity.ToHeaderValue();
        response.Headers["X-Frame-Options"] = FrameOptions.ToHeaderValue();
        response.Headers["Referrer-Policy"] = ReferrerPolicy.ToHeaderValue();
        
        if (XContentTypeOptionsNoSniff)
            response.Headers["X-Content-Type-Options"] = "nosniff";
        
        if (XDownloadOptionsNoOpen)
            response.Headers["X-Download-Options"] = "noopen";
        
        if (XPermittedCrossDomainPoliciesNone)
            response.Headers["X-Permitted-Cross-Domain-Policies"] = "none";
        
        // Remove headers that leak information
        response.Headers.Remove("Server");
        response.Headers.Remove("X-Powered-By");
        response.Headers.Remove("X-AspNet-Version");
    }
    
    public static SecurityHeadersConfiguration Default => new();
    
    public static SecurityHeadersConfiguration Api => new()
    {
        ContentSecurityPolicy = ContentSecurityPolicy.Strict,
        StrictTransportSecurity = StrictTransportSecurity.TwoYears,
        FrameOptions = new FrameOptions.Deny(),
        ReferrerPolicy = ReferrerPolicy.NoReferrer,
        XContentTypeOptionsNoSniff = true
    };
    
    public static SecurityHeadersConfiguration WebApp => new()
    {
        ContentSecurityPolicy = ContentSecurityPolicy.AllowCdnAssets,
        StrictTransportSecurity = StrictTransportSecurity.OneYear,
        FrameOptions = new FrameOptions.SameOrigin(),
        ReferrerPolicy = ReferrerPolicy.StrictOriginWhenCrossOrigin,
        XContentTypeOptionsNoSniff = true
    };
}

// Middleware applies security headers
public class SecurityHeadersMiddleware
{
    private readonly RequestDelegate _next;
    private readonly SecurityHeadersConfiguration _config;
    
    public SecurityHeadersMiddleware(
        RequestDelegate next,
        SecurityHeadersConfiguration config)
    {
        _next = next;
        _config = config;
    }
    
    public async Task InvokeAsync(HttpContext context)
    {
        _config.ApplyTo(context.Response);
        await _next(context);
    }
}

// Register in Startup
public void Configure(IApplicationBuilder app)
{
    app.UseMiddleware<SecurityHeadersMiddleware>(
        SecurityHeadersConfiguration.Default);
    
    // Rest of pipeline...
}
```

## CSP Nonce Generation

For inline scripts and styles with nonces:

```csharp
/// <summary>
/// CSP nonce provider for inline scripts/styles.
/// </summary>
public interface ICspNonceProvider
{
    string GenerateNonce();
}

public class CspNonceProvider : ICspNonceProvider
{
    public string GenerateNonce()
    {
        var bytes = RandomNumberGenerator.GetBytes(16);
        return Convert.ToBase64String(bytes);
    }
}

// Middleware generates nonce per request
public class CspNonceMiddleware
{
    private readonly RequestDelegate _next;
    
    public async Task InvokeAsync(
        HttpContext context,
        ICspNonceProvider nonceProvider)
    {
        var nonce = nonceProvider.GenerateNonce();
        context.Items["CspNonce"] = nonce;
        
        // Apply CSP with nonce
        var csp = new ContentSecurityPolicy
        {
            ScriptSrc = CspDirective.Self().AllowNonce(nonce),
            StyleSrc = CspDirective.Self().AllowNonce(nonce)
        };
        
        context.Response.Headers["Content-Security-Policy"] = csp.ToHeaderValue();
        
        await _next(context);
    }
}

// In Razor view:
// @inject IHttpContextAccessor HttpContextAccessor
// @{
//     var nonce = HttpContextAccessor.HttpContext?.Items["CspNonce"] as string;
// }
// <script nonce="@nonce">
//     console.log('Inline script with CSP nonce');
// </script>
```

## Permissions Policy (formerly Feature Policy)

```csharp
/// <summary>
/// Permissions Policy configuration.
/// </summary>
public sealed record PermissionsPolicy
{
    public PermissionDirective Camera { get; init; } = PermissionDirective.None();
    public PermissionDirective Microphone { get; init; } = PermissionDirective.None();
    public PermissionDirective Geolocation { get; init; } = PermissionDirective.None();
    public PermissionDirective Payment { get; init; } = PermissionDirective.None();
    public PermissionDirective Usb { get; init; } = PermissionDirective.None();
    
    public string ToHeaderValue()
    {
        var policies = new List<string>();
        
        AddPolicy(policies, "camera", Camera);
        AddPolicy(policies, "microphone", Microphone);
        AddPolicy(policies, "geolocation", Geolocation);
        AddPolicy(policies, "payment", Payment);
        AddPolicy(policies, "usb", Usb);
        
        return string.Join(", ", policies);
    }
    
    private void AddPolicy(List<string> policies, string name, PermissionDirective directive)
    {
        policies.Add($"{name}={directive.ToValue()}");
    }
    
    public static PermissionsPolicy DenyAll => new()
    {
        Camera = PermissionDirective.None(),
        Microphone = PermissionDirective.None(),
        Geolocation = PermissionDirective.None(),
        Payment = PermissionDirective.None(),
        Usb = PermissionDirective.None()
    };
}

public sealed record PermissionDirective
{
    private readonly string _value;
    
    private PermissionDirective(string value) => _value = value;
    
    public static PermissionDirective None() => new("()");
    public static PermissionDirective Self() => new("(self)");
    public static PermissionDirective All() => new("*");
    public static PermissionDirective Origins(params string[] origins) =>
        new($"({string.Join(" ", origins)})");
    
    public string ToValue() => _value;
}
```

## Endpoint-Specific Security Headers

```csharp
/// <summary>
/// Attribute to override security headers for specific endpoints.
/// </summary>
[AttributeUsage(AttributeTargets.Class | AttributeTargets.Method)]
public class SecurityHeadersAttribute : Attribute
{
    public SecurityHeadersConfiguration? Configuration { get; set; }
    
    public SecurityHeadersAttribute() { }
    
    public SecurityHeadersAttribute(Type configurationType)
    {
        if (!typeof(ISecurityHeadersProvider).IsAssignableFrom(configurationType))
            throw new ArgumentException(
                $"{configurationType} must implement ISecurityHeadersProvider");
        
        var provider = (ISecurityHeadersProvider)Activator.CreateInstance(configurationType)!;
        Configuration = provider.GetConfiguration();
    }
}

public interface ISecurityHeadersProvider
{
    SecurityHeadersConfiguration GetConfiguration();
}

// Example: Relaxed CSP for admin panel with rich text editor
public class AdminPanelHeaders : ISecurityHeadersProvider
{
    public SecurityHeadersConfiguration GetConfiguration() => new()
    {
        ContentSecurityPolicy = new ContentSecurityPolicy
        {
            DefaultSrc = CspDirective.Self(),
            ScriptSrc = CspDirective.Self().AllowUnsafeEval(),  // For WYSIWYG editor
            StyleSrc = CspDirective.Self().AllowUnsafeInline(),
            ImgSrc = CspDirective.Self().AllowSource("data:"),
            ObjectSrc = CspDirective.None()
        },
        FrameOptions = new FrameOptions.SameOrigin()
    };
}

[SecurityHeaders(typeof(AdminPanelHeaders))]
public class AdminController : ControllerBase
{
    // This controller uses relaxed CSP
}
```

## Testing

```csharp
public class SecurityHeadersTests
{
    [Fact]
    public void ContentSecurityPolicy_ToHeaderValue_ProducesCorrectString()
    {
        var csp = new ContentSecurityPolicy
        {
            DefaultSrc = CspDirective.Self(),
            ScriptSrc = CspDirective.Self().AllowSource("https://cdn.example.com"),
            ObjectSrc = CspDirective.None()
        };
        
        var header = csp.ToHeaderValue();
        
        Assert.Contains("default-src 'self'", header);
        Assert.Contains("script-src 'self' https://cdn.example.com", header);
        Assert.Contains("object-src 'none'", header);
    }
    
    [Fact]
    public void StrictTransportSecurity_OneYear_ProducesCorrectHeader()
    {
        var hsts = StrictTransportSecurity.OneYear;
        
        var header = hsts.ToHeaderValue();
        
        Assert.Equal("max-age=31536000; includeSubDomains", header);
    }
    
    [Fact]
    public void FrameOptions_Deny_ProducesCorrectHeader()
    {
        var frameOptions = new FrameOptions.Deny();
        
        var header = frameOptions.ToHeaderValue();
        
        Assert.Equal("DENY", header);
    }
    
    [Fact]
    public void SecurityHeadersConfiguration_ApplyTo_SetsAllHeaders()
    {
        var config = SecurityHeadersConfiguration.Default;
        var context = new DefaultHttpContext();
        
        config.ApplyTo(context.Response);
        
        Assert.True(context.Response.Headers.ContainsKey("Content-Security-Policy"));
        Assert.True(context.Response.Headers.ContainsKey("Strict-Transport-Security"));
        Assert.True(context.Response.Headers.ContainsKey("X-Frame-Options"));
        Assert.True(context.Response.Headers.ContainsKey("X-Content-Type-Options"));
    }
}
```

## Why It's a Problem

1. **String-based headers**: Typos and invalid values not caught until runtime
2. **No validation**: Incorrect CSP directives silently fail
3. **Inconsistent policies**: Different endpoints have different security levels
4. **Easy to misconfigure**: Complex CSP syntax error-prone
5. **No type safety**: Cannot leverage compiler for validation

## Symptoms

- Security headers set with string literals
- Typos in header names or values
- Inconsistent security policies across endpoints
- Overly permissive CSP directives ('unsafe-inline', '*')
- Missing security headers on some endpoints

## Benefits

- **Type safety**: Compile-time validation of header values
- **Self-documenting**: Types make security policies explicit
- **Consistent policies**: Centralized configuration prevents inconsistencies
- **Reusable presets**: Named configurations for common scenarios
- **Testable**: Easy to unit test header generation

## See Also

- [CORS Configuration](./cors-configuration.md) — type-safe origin validation
- [Input Sanitization](./input-sanitization.md) — trusted types
- [CSRF Protection](./csrf-protection.md) — anti-forgery tokens
- [Capability Security](./capability-security.md) — authorization tokens

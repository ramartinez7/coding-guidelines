# Common Security Attacks and Prevention Strategies

> Web applications face numerous security threats—understanding common attack vectors and implementing defense-in-depth strategies prevents vulnerabilities from becoming exploits.

## Overview

This document provides an overview of the most critical security attacks facing web applications and how to prevent them using type-safe patterns. Each attack pattern is covered in detail in dedicated pattern documents linked throughout.

## OWASP Top 10 Security Risks

The Open Web Application Security Project (OWASP) maintains a list of the most critical security risks to web applications. Here's how to address them in C#:

### 1. Injection Attacks

Injection flaws occur when untrusted data is sent to an interpreter as part of a command or query.

#### SQL Injection

**Attack**: Attacker manipulates SQL queries by inserting malicious SQL code through input fields.

```csharp
// ❌ VULNERABLE: SQL injection
var query = $"SELECT * FROM Users WHERE Username = '{username}'";
var user = dbContext.Database.SqlQuery<User>(query).FirstOrDefault();

// If username = "admin' OR '1'='1' --"
// Query becomes: SELECT * FROM Users WHERE Username = 'admin' OR '1'='1' --'
// This returns all users!
```

**Prevention**: Use parameterized queries and typed SQL builders.

```csharp
// ✅ SAFE: Parameterized query
var user = dbContext.Users
    .FromSqlInterpolated($"SELECT * FROM Users WHERE Username = {username}")
    .FirstOrDefault();

// Or better: use LINQ
var user = dbContext.Users
    .FirstOrDefault(u => u.Username == username);
```

**See**: [SQL Injection Prevention](./sql-injection-prevention.md)

#### Command Injection

**Attack**: Attacker executes arbitrary system commands by injecting shell commands through input.

```csharp
// ❌ VULNERABLE: Command injection
var fileName = userInput;
Process.Start("cmd.exe", $"/c type {fileName}");

// If fileName = "file.txt & del /f /q *.*"
// This deletes all files in the directory!
```

**Prevention**: Use validated command types and avoid shell execution.

```csharp
// ✅ SAFE: Validated file path
var validatedPath = SafeFilePath.Create(userInput);
if (validatedPath.IsSuccess)
{
    var content = File.ReadAllText(validatedPath.Value.FullPath);
}
```

**See**: [Command Injection Prevention](./command-injection-prevention.md)

#### LDAP Injection

**Attack**: Manipulating LDAP queries through user input.

```csharp
// ❌ VULNERABLE: LDAP injection
var filter = $"(uid={username})";
var search = new DirectorySearcher(filter);

// If username = "*)(uid=*"
// Filter becomes: (uid=*)(uid=*)
// This returns all users!
```

**Prevention**: Use parameterized LDAP filters with proper escaping.

```csharp
// ✅ SAFE: Escaped LDAP filter
// Use System.DirectoryServices.Protocols or Novell.Directory.Ldap
public static string EscapeLdapFilterValue(string value)
{
    // Escape special LDAP characters
    return value
        .Replace("\\", "\\5c")
        .Replace("*", "\\2a")
        .Replace("(", "\\28")
        .Replace(")", "\\29")
        .Replace("\0", "\\00");
}

var escapedUsername = EscapeLdapFilterValue(username);
var filter = $"(uid={escapedUsername})";
var search = new DirectorySearcher(filter);
```

**Pattern**: Create a `SafeLdapFilter` type similar to SQL injection prevention.

### 2. Cross-Site Scripting (XSS)

**Attack**: Injecting malicious JavaScript into web pages viewed by other users.

#### Stored XSS

```csharp
// ❌ VULNERABLE: Stored XSS
public class BlogPost
{
    public string Content { get; set; }  // Raw HTML from user
}

// In Razor view:
@Model.Content  // Renders: <script>alert('XSS')</script>
```

**Prevention**: Use trusted types and proper encoding.

```csharp
// ✅ SAFE: Sanitized HTML type
public sealed record SanitizedHtml
{
    private readonly string _value;
    
    private SanitizedHtml(string value)
    {
        _value = value;
    }
    
    public static SanitizedHtml Create(string rawHtml)
    {
        // Use AntiXSS library or similar
        var sanitizer = new HtmlSanitizer();
        var clean = sanitizer.Sanitize(rawHtml);
        return new SanitizedHtml(clean);
    }
    
    public string Value => _value;
}

public class BlogPost
{
    public SanitizedHtml Content { get; private set; }
}

// In Razor view:
@Html.Raw(Model.Content.Value)  // Safe because Content is sanitized
```

#### Reflected XSS

```csharp
// ❌ VULNERABLE: Reflected XSS
[HttpGet]
public IActionResult Search(string query)
{
    return Content($"<h1>Results for: {query}</h1>", "text/html");
}

// URL: /search?query=<script>steal(document.cookie)</script>
```

**Prevention**: Always encode output and use Content Security Policy.

```csharp
// ✅ SAFE: Encoded output
[HttpGet]
public IActionResult Search(string query)
{
    var encodedQuery = HtmlEncoder.Default.Encode(query);
    return Content($"<h1>Results for: {encodedQuery}</h1>", "text/html");
}

// Or better: Use Razor which encodes by default
public IActionResult Search(string query)
{
    return View(new SearchViewModel { Query = query });
}

// In Razor:
<h1>Results for: @Model.Query</h1>  // Automatically encoded
```

#### DOM-based XSS

**Attack**: Client-side JavaScript dynamically inserts untrusted data into the DOM.

**Prevention**: Use TypeScript and sanitize data before DOM insertion.

```typescript
// ❌ VULNERABLE: DOM XSS
const userInput = new URLSearchParams(location.search).get('name');
document.getElementById('greeting').innerHTML = `Hello, ${userInput}!`;

// ✅ SAFE: Text content only
document.getElementById('greeting').textContent = `Hello, ${userInput}!`;

// Or if HTML is needed, sanitize it
import DOMPurify from 'dompurify';
document.getElementById('greeting').innerHTML = DOMPurify.sanitize(`Hello, ${userInput}!`);
```

**See**: [Input Sanitization](./input-sanitization.md)

### 3. Cross-Site Request Forgery (CSRF)

**Attack**: Tricking a user's browser into making unwanted requests to a site where they're authenticated.

```html
<!-- ❌ Attacker's malicious page -->
<form action="https://bank.com/transfer" method="POST">
    <input type="hidden" name="to" value="attacker" />
    <input type="hidden" name="amount" value="10000" />
</form>
<script>document.forms[0].submit();</script>
```

**Prevention**: Use anti-forgery tokens.

```csharp
// ✅ SAFE: CSRF protection
[HttpPost]
[ValidateAntiForgeryToken]
public IActionResult Transfer(TransferCommand command)
{
    // Token is validated before this executes
    _transferService.Execute(command);
    return Ok();
}

// In Razor form:
<form method="post">
    @Html.AntiForgeryToken()
    <!-- Form fields -->
</form>
```

**See**: [CSRF Protection](./csrf-protection.md)

### 4. Insecure Deserialization

**Attack**: Exploiting deserialization of untrusted data to execute arbitrary code.

```csharp
// ❌ VULNERABLE: Unsafe deserialization
var formatter = new BinaryFormatter();
var obj = formatter.Deserialize(untrustedStream);  // Can execute arbitrary code!
```

**Prevention**: Never use `BinaryFormatter`, use safe serializers with type allowlists.

```csharp
// ✅ SAFE: JSON with known types only
var options = new JsonSerializerOptions
{
    TypeInfoResolver = new DefaultJsonTypeInfoResolver
    {
        Modifiers =
        {
            static typeInfo =>
            {
                // Only allow specific types
                if (typeInfo.Type != typeof(OrderDto) && 
                    typeInfo.Type != typeof(CustomerDto))
                {
                    throw new InvalidOperationException($"Type {typeInfo.Type} not allowed");
                }
            }
        }
    }
};

var obj = JsonSerializer.Deserialize<OrderDto>(json, options);

// Or use polymorphic serialization with explicit type discriminator
var options2 = new JsonSerializerOptions
{
    Converters = { new KnownTypeConverter() }  // Custom converter with allowlist
};
```

**See**: [Deserialization Prevention](./deserialization-prevention.md)

### 5. XML External Entity (XXE) Attacks

**Attack**: Exploiting XML parsers to access local files or perform SSRF attacks.

```csharp
// ❌ VULNERABLE: XXE attack
var doc = new XmlDocument();
doc.Load(untrustedXml);  // Default settings allow external entities!

// Attacker's XML:
// <?xml version="1.0"?>
// <!DOCTYPE foo [<!ENTITY xxe SYSTEM "file:///etc/passwd">]>
// <root>&xxe;</root>
```

**Prevention**: Disable DTD processing and external entities.

```csharp
// ✅ SAFE: Secure XML parsing
var settings = new XmlReaderSettings
{
    DtdProcessing = DtdProcessing.Prohibit,
    XmlResolver = null
};

using var reader = XmlReader.Create(untrustedXml, settings);
var doc = new XmlDocument();
doc.Load(reader);
```

**See**: [XXE Prevention](./xxe-prevention.md)

### 6. Security Misconfiguration

**Attack**: Exploiting default configurations, unnecessary features, or missing security headers.

#### Missing Security Headers

```csharp
// ❌ VULNERABLE: No security headers
public void Configure(IApplicationBuilder app)
{
    app.UseRouting();
    app.UseEndpoints(endpoints => endpoints.MapControllers());
}
```

**Prevention**: Configure security headers.

```csharp
// ✅ SAFE: Security headers configured
public void Configure(IApplicationBuilder app)
{
    app.Use(async (context, next) =>
    {
        // Prevent clickjacking
        context.Response.Headers.Add("X-Frame-Options", "DENY");
        
        // Prevent MIME sniffing
        context.Response.Headers.Add("X-Content-Type-Options", "nosniff");
        
        // Enable XSS protection
        context.Response.Headers.Add("X-XSS-Protection", "1; mode=block");
        
        // Content Security Policy
        context.Response.Headers.Add("Content-Security-Policy",
            "default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline'");
        
        // HSTS
        context.Response.Headers.Add("Strict-Transport-Security",
            "max-age=31536000; includeSubDomains");
        
        await next();
    });
    
    app.UseRouting();
    app.UseEndpoints(endpoints => endpoints.MapControllers());
}
```

**See**: [Security Headers](./security-headers.md)

#### Insecure CORS Configuration

```csharp
// ❌ VULNERABLE: Allow all origins
// Note: This combination throws InvalidOperationException at runtime
// (AllowAnyOrigin + AllowCredentials is invalid) but shows the security intent
services.AddCors(options =>
{
    options.AddPolicy("AllowAll",
        builder => builder
            .AllowAnyOrigin()      // Accepts requests from any domain
            .AllowAnyMethod()      // All HTTP methods allowed
            .AllowAnyHeader());    // All headers allowed - too permissive!
});

// Or this variation which works but is equally dangerous:
services.AddCors(options =>
{
    options.AddPolicy("AllowAll",
        builder => builder
            .SetIsOriginAllowed(_ => true)  // Allows any origin
            .AllowAnyMethod()
            .AllowAnyHeader()
            .AllowCredentials());  // Dangerous: any site can make credentialed requests
});
```

**Prevention**: Whitelist specific origins.

```csharp
// ✅ SAFE: Whitelist specific origins
services.AddCors(options =>
{
    options.AddPolicy("RestrictedPolicy",
        builder => builder
            .WithOrigins("https://trusted-site.com")
            .WithMethods("GET", "POST")
            .WithHeaders("Content-Type")
            .AllowCredentials());
});
```

**See**: [CORS Configuration](./cors-configuration.md)

### 7. Path Traversal

**Attack**: Accessing files outside the intended directory by manipulating file paths.

```csharp
// ❌ VULNERABLE: Path traversal
[HttpGet]
public IActionResult DownloadFile(string fileName)
{
    var filePath = Path.Combine(_uploadFolder, fileName);
    return PhysicalFile(filePath, "application/octet-stream");
}

// Request: /download?fileName=../../../etc/passwd
// Accesses: /var/www/uploads/../../../etc/passwd = /etc/passwd
```

**Prevention**: Validate and restrict file paths.

```csharp
// ✅ SAFE: Validated path
public sealed record SafeFilePath
{
    private readonly string _basePath;
    private readonly string _fullPath;
    
    private SafeFilePath(string basePath, string fullPath)
    {
        _basePath = basePath;
        _fullPath = fullPath;
    }
    
    public static Result<SafeFilePath, string> Create(string basePath, string userInput)
    {
        // Remove dangerous characters
        var sanitized = Path.GetFileName(userInput);
        
        // Combine and resolve
        var fullPath = Path.GetFullPath(Path.Combine(basePath, sanitized));
        
        // Verify still within base path
        if (!fullPath.StartsWith(Path.GetFullPath(basePath)))
            return Result<SafeFilePath, string>.Failure("Invalid path");
        
        return Result<SafeFilePath, string>.Success(
            new SafeFilePath(basePath, fullPath));
    }
    
    public string FullPath => _fullPath;
}

[HttpGet]
public IActionResult DownloadFile(string fileName)
{
    var safePathResult = SafeFilePath.Create(_uploadFolder, fileName);
    
    return safePathResult.Match(
        onSuccess: path => PhysicalFile(path.FullPath, "application/octet-stream"),
        onFailure: error => BadRequest(new { Error = error }));
}
```

**See**: [Path Traversal Prevention](./path-traversal-prevention.md)

### 8. Open Redirect

**Attack**: Redirecting users to malicious sites through unvalidated redirect URLs.

```csharp
// ❌ VULNERABLE: Open redirect
[HttpGet]
public IActionResult Redirect(string returnUrl)
{
    return Redirect(returnUrl);  // Can redirect anywhere!
}

// Request: /redirect?returnUrl=https://evil.com/phishing
```

**Prevention**: Validate redirect URLs against allowlist.

```csharp
// ✅ SAFE: Validated redirect
public sealed record SafeRedirectUrl
{
    private readonly string _url;
    
    private SafeRedirectUrl(string url)
    {
        _url = url;
    }
    
    public static Result<SafeRedirectUrl, string> Create(
        string url,
        HashSet<string> allowedDomains)
    {
        if (!Uri.TryCreate(url, UriKind.Absolute, out var uri))
        {
            // Allow relative URLs (same domain)
            if (url.StartsWith("/") && !url.StartsWith("//"))
                return Result<SafeRedirectUrl, string>.Success(new SafeRedirectUrl(url));
            
            return Result<SafeRedirectUrl, string>.Failure("Invalid URL");
        }
        
        // Check if absolute URL is in allowlist
        if (!allowedDomains.Contains(uri.Host))
            return Result<SafeRedirectUrl, string>.Failure("Domain not allowed");
        
        return Result<SafeRedirectUrl, string>.Success(new SafeRedirectUrl(url));
    }
    
    public string Value => _url;
}

[HttpGet]
public IActionResult Redirect(string returnUrl)
{
    var allowedDomains = new HashSet<string> { "mysite.com", "trusted.com" };
    var safeUrlResult = SafeRedirectUrl.Create(returnUrl, allowedDomains);
    
    return safeUrlResult.Match(
        onSuccess: url => Redirect(url.Value),
        onFailure: _ => RedirectToAction("Index", "Home"));
}
```

**See**: [Open Redirect Prevention](./open-redirect-prevention.md)

### 9. Cryptographic Failures

**Attack**: Exploiting weak cryptography, hardcoded keys, or improper key management.

```csharp
// ❌ VULNERABLE: Weak cryptography
public string EncryptData(string data)
{
    using var des = new DESCryptoServiceProvider();  // DES is broken!
    des.Key = Encoding.UTF8.GetBytes("12345678");    // Hardcoded key!
    // ...
}

// ❌ VULNERABLE: Weak hashing
var hash = MD5.Create().ComputeHash(Encoding.UTF8.GetBytes(password));
```

**Prevention**: Use modern algorithms and proper key management.

```csharp
// ✅ SAFE: Modern cryptography
public sealed class SecureEncryption
{
    private readonly byte[] _key;
    
    public SecureEncryption(IConfiguration config)
    {
        // Load key from secure storage (Key Vault, etc.)
        _key = Convert.FromBase64String(config["Encryption:Key"]);
    }
    
    public byte[] Encrypt(byte[] data)
    {
        using var aes = Aes.Create();
        aes.Key = _key;
        aes.GenerateIV();
        
        using var encryptor = aes.CreateEncryptor();
        var encrypted = encryptor.TransformFinalBlock(data, 0, data.Length);
        
        // Return IV + encrypted data
        return aes.IV.Concat(encrypted).ToArray();
    }
}

// ✅ SAFE: Strong password hashing
public sealed record HashedPassword
{
    private readonly string _hash;
    
    private HashedPassword(string hash)
    {
        _hash = hash;
    }
    
    public static HashedPassword Create(string password)
    {
        // Use Argon2, bcrypt, or PBKDF2
        var hash = BCrypt.Net.BCrypt.HashPassword(password, workFactor: 12);
        return new HashedPassword(hash);
    }
    
    public bool Verify(string password)
    {
        return BCrypt.Net.BCrypt.Verify(password, _hash);
    }
}
```

**See**: [Cryptographic Best Practices](./cryptographic-practices.md)

### 10. Server-Side Request Forgery (SSRF)

**Attack**: Tricking the server into making requests to internal systems or arbitrary URLs.

```csharp
// ❌ VULNERABLE: SSRF
[HttpPost]
public async Task<IActionResult> FetchImage(string imageUrl)
{
    using var client = new HttpClient();
    var response = await client.GetAsync(imageUrl);  // Can access internal services!
    return File(await response.Content.ReadAsByteArrayAsync(), "image/png");
}

// Request: /fetch?imageUrl=http://localhost:8080/admin/delete-all-users
```

**Prevention**: Validate and restrict URLs, use allowlists.

```csharp
// ✅ SAFE: URL validation
public sealed record SafeExternalUrl
{
    private readonly Uri _uri;
    
    private SafeExternalUrl(Uri uri)
    {
        _uri = uri;
    }
    
    public static Result<SafeExternalUrl, string> Create(
        string url,
        HashSet<string> allowedDomains)
    {
        if (!Uri.TryCreate(url, UriKind.Absolute, out var uri))
            return Result<SafeExternalUrl, string>.Failure("Invalid URL");
        
        // Block private IP ranges
        if (IsPrivateIpAddress(uri.Host))
            return Result<SafeExternalUrl, string>.Failure("Private IPs not allowed");
        
        // Block localhost
        if (uri.Host == "localhost" || uri.Host == "127.0.0.1")
            return Result<SafeExternalUrl, string>.Failure("Localhost not allowed");
        
        // Check allowlist
        if (!allowedDomains.Contains(uri.Host))
            return Result<SafeExternalUrl, string>.Failure("Domain not allowed");
        
        return Result<SafeExternalUrl, string>.Success(new SafeExternalUrl(uri));
    }
    
    private static bool IsPrivateIpAddress(string host)
    {
        if (!IPAddress.TryParse(host, out var ip))
            return false;
        
        var bytes = ip.GetAddressBytes();
        
        // Check private ranges: 10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16
        return bytes[0] == 10
            || (bytes[0] == 172 && bytes[1] >= 16 && bytes[1] <= 31)
            || (bytes[0] == 192 && bytes[1] == 168);
    }
    
    public Uri Value => _uri;
}

[HttpPost]
public async Task<IActionResult> FetchImage(string imageUrl)
{
    var allowedDomains = new HashSet<string> { "cdn.example.com", "images.example.com" };
    var safeUrlResult = SafeExternalUrl.Create(imageUrl, allowedDomains);
    
    return await safeUrlResult.Match(
        onSuccess: async url =>
        {
            using var client = new HttpClient();
            var response = await client.GetAsync(url.Value);
            return File(await response.Content.ReadAsByteArrayAsync(), "image/png");
        },
        onFailure: error => Task.FromResult<IActionResult>(BadRequest(new { Error = error })));
}
```

## Defense in Depth

Security is not a single pattern—it requires multiple layers of defense:

### 1. Input Validation Layer

```csharp
// Validate at the API boundary
[ApiController]
public class OrdersController
{
    [HttpPost]
    public async Task<IActionResult> PlaceOrder([FromBody] PlaceOrderRequest request)
    {
        // Validate and convert to domain command
        var emailResult = Email.Create(request.Email);
        if (emailResult.IsFailure)
            return BadRequest(new { Error = emailResult.Error });
        
        var command = new PlaceOrderCommand(emailResult.Value, ...);
        await _useCase.ExecuteAsync(command);
        return Ok();
    }
}
```

### 2. Domain Logic Layer

```csharp
// Enforce business rules in domain
public sealed class Order
{
    public Result<Unit, string> AddItem(Product product, int quantity)
    {
        if (quantity <= 0)
            return Result<Unit, string>.Failure("Quantity must be positive");
        
        if (quantity > 100)
            return Result<Unit, string>.Failure("Maximum 100 items per order");
        
        // Business logic here
    }
}
```

### 3. Authorization Layer

```csharp
// Check permissions before actions
public sealed class PlaceOrderUseCase
{
    public async Task<Result<OrderId, PlaceOrderError>> ExecuteAsync(
        PlaceOrderCommand command,
        Capability<PlaceOrder> capability)  // Proof of authorization
    {
        // Can only call if user has capability
    }
}
```

**See**: [Capability Security](./capability-security.md)

### 4. Data Access Layer

```csharp
// Use parameterized queries
public class OrderRepository : IOrderRepository
{
    public async Task<Order> GetByIdAsync(OrderId id)
    {
        return await _dbContext.Orders
            .Where(o => o.Id == id.Value)  // Parameterized by EF
            .FirstOrDefaultAsync();
    }
}
```

### 5. Audit Logging

```csharp
// Log security-relevant events
public sealed class AuditingUseCaseDecorator<TCommand, TResult>
{
    public async Task<Result<TResult, Error>> ExecuteAsync(TCommand command)
    {
        _logger.LogInformation("User {UserId} executing {Command}",
            _currentUser.Id,
            typeof(TCommand).Name);
        
        var result = await _inner.ExecuteAsync(command);
        
        if (result.IsFailure)
            _logger.LogWarning("Failed to execute {Command}: {Error}",
                typeof(TCommand).Name,
                result.Error);
        
        return result;
    }
}
```

## Security Testing

### Static Analysis

```csharp
// Use Roslyn analyzers to detect security issues at compile time
// Add to project file:
// <PackageReference Include="SecurityCodeScan.VS2019" Version="5.6.7" />
```

### Dynamic Analysis

```csharp
// Test for vulnerabilities with integration tests
[Fact]
public async Task LoginEndpoint_WithSqlInjection_ShouldNotAuthenticate()
{
    // Arrange
    var maliciousUsername = "admin' OR '1'='1' --";
    
    // Act
    var response = await _client.PostAsJsonAsync("/api/auth/login", new
    {
        Username = maliciousUsername,
        Password = "anything"
    });
    
    // Assert
    Assert.Equal(HttpStatusCode.Unauthorized, response.StatusCode);
}

[Fact]
public async Task SearchEndpoint_WithXssPayload_ShouldEncodeOutput()
{
    // Arrange
    var xssPayload = "<script>alert('XSS')</script>";
    
    // Act
    var response = await _client.GetAsync($"/api/search?q={Uri.EscapeDataString(xssPayload)}");
    var html = await response.Content.ReadAsStringAsync();
    
    // Assert
    Assert.DoesNotContain("<script>", html);
    Assert.Contains("&lt;script&gt;", html);  // Should be HTML encoded
}
```

## Security Checklist

Use this checklist when reviewing code for security issues:

- [ ] **Input Validation**: All user input validated and sanitized at boundaries
- [ ] **Output Encoding**: All dynamic content properly encoded for context (HTML, URL, JS, SQL)
- [ ] **Authentication**: Strong authentication mechanisms with MFA support
- [ ] **Authorization**: Principle of least privilege, capability-based security
- [ ] **Cryptography**: Modern algorithms (AES-256, SHA-256), proper key management
- [ ] **Error Handling**: No sensitive information in error messages
- [ ] **Logging**: Security events logged, PII protected
- [ ] **HTTPS**: All traffic encrypted, HSTS enabled
- [ ] **Dependencies**: All packages up-to-date, no known vulnerabilities
- [ ] **Configuration**: Secure defaults, secrets in vault (not code)
- [ ] **Headers**: Security headers configured (CSP, X-Frame-Options, etc.)
- [ ] **Rate Limiting**: Protections against brute force and DoS
- [ ] **Session Management**: Secure cookies, proper timeout, session fixation prevention

## See Also

### Injection Prevention
- [SQL Injection Prevention](./sql-injection-prevention.md)
- [Command Injection Prevention](./command-injection-prevention.md)
- [Path Traversal Prevention](./path-traversal-prevention.md)

### XSS and Content Security
- [Input Sanitization](./input-sanitization.md)
- [Security Headers](./security-headers.md)
- [Type-Safe String Interpolation](./type-safe-string-interpolation.md)

### Data Security
- [Cryptographic Best Practices](./cryptographic-practices.md)
- [Secret Types](./secret-types.md)
- [Deserialization Prevention](./deserialization-prevention.md)

### API Security
- [CSRF Protection](./csrf-protection.md)
- [CORS Configuration](./cors-configuration.md)
- [API Key Rotation](./api-key-rotation.md)
- [Request Signing](./request-signing.md)

### XML Security
- [XXE Prevention](./xxe-prevention.md)

### Redirect Security
- [Open Redirect Prevention](./open-redirect-prevention.md)

### Access Control
- [Capability Security](./capability-security.md)
- [Authentication Context](./authentication-context.md)

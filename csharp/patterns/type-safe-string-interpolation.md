# Type-Safe String Interpolation (Preventing Injection Attacks)

> Building SQL, HTML, or URLs from string concatenation risks injection attacks—use typed builders to make injection unrepresentable.

## Problem

String concatenation and interpolation for constructing SQL queries, HTML content, or URLs creates injection vulnerabilities. A single missed escaping allows attackers to inject malicious code. Traditional approaches rely on developers remembering to escape every value.

## Example

### ❌ Before

```csharp
public class UserRepository
{
    string GetUserById(string userId)
    {
        // SQL Injection vulnerability!
        string query = $"SELECT * FROM Users WHERE Id = '{userId}'";
        return database.ExecuteQuery(query);
    }
    
    string SearchUsers(string searchTerm)
    {
        // What if searchTerm contains "'; DROP TABLE Users; --"?
        return $"SELECT * FROM Users WHERE Name LIKE '%{searchTerm}%'";
    }
}

public class HtmlGenerator
{
    string RenderGreeting(string userName)
    {
        // XSS vulnerability!
        return $"<div>Hello, {userName}!</div>";
        // If userName = "<script>alert('XSS')</script>", we're compromised
    }
    
    string BuildLink(string url, string text)
    {
        // Multiple injection vectors
        return $"<a href=\"{url}\">{text}</a>";
    }
}

public class UrlBuilder
{
    string BuildSearchUrl(string query)
    {
        // URL injection—query needs encoding
        return $"https://api.example.com/search?q={query}";
    }
}
```

**Problems:**
- User input inserted directly into structured strings
- Easy to forget escaping/encoding
- No compile-time protection
- Each developer must remember security rules

### ✅ After

```csharp
/// <summary>
/// Marker interface for sanitized/escaped strings.
/// </summary>
public interface ITrustedString
{
    string Value { get; }
}

/// <summary>
/// SQL string that has been parameterized or properly escaped.
/// Cannot be constructed from raw strings without explicit sanitization.
/// </summary>
public readonly struct SqlString : ITrustedString
{
    public string Value { get; init; }
    
    // Private constructor prevents casual creation
    private SqlString(string value)
    {
        Value = value;
    }
    
    /// <summary>
    /// Build parameterized SQL safely.
    /// </summary>
    public static SqlString Raw(FormattableString sql)
    {
        // Extract parameters and create proper parameterized query
        var parameters = new List<object>();
        var formatArgs = new object[sql.ArgumentCount];
        
        for (int i = 0; i < sql.ArgumentCount; i++)
        {
            formatArgs[i] = $"@p{i}";
            parameters.Add(sql.GetArgument(i));
        }
        
        var query = string.Format(sql.Format, formatArgs);
        return new SqlString(query) { Parameters = parameters };
    }
    
    public IReadOnlyList<object> Parameters { get; init; } = Array.Empty<object>();
    
    /// <summary>
    /// Combine SQL fragments safely.
    /// </summary>
    public static SqlString operator +(SqlString left, SqlString right)
    {
        return new SqlString(left.Value + " " + right.Value)
        {
            Parameters = left.Parameters.Concat(right.Parameters).ToList()
        };
    }
    
    /// <summary>
    /// Only for SQL literals that are known-safe at compile time.
    /// </summary>
    public static SqlString Literal(string trustedSql) => new(trustedSql);
}

/// <summary>
/// HTML string that has been properly encoded to prevent XSS.
/// </summary>
public readonly struct HtmlString : ITrustedString
{
    public string Value { get; }
    
    private HtmlString(string value)
    {
        Value = value;
    }
    
    /// <summary>
    /// Encode user input for safe inclusion in HTML.
    /// </summary>
    public static HtmlString Encode(string unsafe)
    {
        // Use System.Net.WebUtility (available in all .NET versions)
        // or System.Text.Encodings.Web.HtmlEncoder for .NET Core/5+
        var encoded = System.Net.WebUtility.HtmlEncode(unsafe);
        return new HtmlString(encoded);
    }
    
    /// <summary>
    /// Mark string as trusted HTML (use only for HTML from trusted sources).
    /// </summary>
    public static HtmlString Trusted(string trustedHtml) => new(trustedHtml);
    
    /// <summary>
    /// Concatenate HTML strings safely.
    /// </summary>
    public static HtmlString operator +(HtmlString left, HtmlString right)
        => new(left.Value + right.Value);
}

/// <summary>
/// URL component that has been properly encoded.
/// </summary>
public readonly struct UrlEncodedString : ITrustedString
{
    public string Value { get; }
    
    private UrlEncodedString(string value)
    {
        Value = value;
    }
    
    public static UrlEncodedString Encode(string unsafe)
    {
        var encoded = Uri.EscapeDataString(unsafe);
        return new UrlEncodedString(encoded);
    }
    
    public static UrlEncodedString Trusted(string trustedUrl) => new(trustedUrl);
}

// Usage: SQL
public class UserRepository
{
    SqlQueryResult GetUserById(string userId)
    {
        // Type system forces parameterization
        SqlString query = SqlString.Raw($"SELECT * FROM Users WHERE Id = {userId}");
        // Becomes: "SELECT * FROM Users WHERE Id = @p0" with parameter userId
        
        return database.ExecuteParameterized(query);
    }
    
    SqlQueryResult SearchUsers(string searchTerm)
    {
        // Automatically parameterized—injection impossible
        SqlString query = SqlString.Raw(
            $"SELECT * FROM Users WHERE Name LIKE {'%' + searchTerm + '%'}"
        );
        
        return database.ExecuteParameterized(query);
    }
    
    SqlQueryResult ComplexQuery(int minAge, string city)
    {
        SqlString baseQuery = SqlString.Literal("SELECT * FROM Users WHERE");
        SqlString ageFilter = SqlString.Raw($"Age >= {minAge}");
        SqlString cityFilter = SqlString.Raw($"City = {city}");
        
        // Safe composition
        SqlString fullQuery = baseQuery + ageFilter + SqlString.Literal("AND") + cityFilter;
        
        return database.ExecuteParameterized(fullQuery);
    }
}

// Usage: HTML
public class HtmlGenerator
{
    HtmlString RenderGreeting(string userName)
    {
        // User input is explicitly encoded
        HtmlString safeUserName = HtmlString.Encode(userName);
        HtmlString greeting = HtmlString.Trusted("<div>Hello, ") + safeUserName + HtmlString.Trusted("!</div>");
        
        return greeting;
        // If userName = "<script>alert('XSS')</script>"
        // Result: "<div>Hello, &lt;script&gt;alert('XSS')&lt;/script&gt;!</div>"
    }
    
    HtmlString BuildLink(string url, string text)
    {
        // Both components must be explicitly encoded
        var safeUrl = HtmlString.Encode(url);
        var safeText = HtmlString.Encode(text);
        
        return HtmlString.Trusted("<a href=\"") + safeUrl + 
               HtmlString.Trusted("\">") + safeText + 
               HtmlString.Trusted("</a>");
    }
}

// Usage: URLs
public class UrlBuilder
{
    string BuildSearchUrl(string query)
    {
        var encodedQuery = UrlEncodedString.Encode(query);
        return $"https://api.example.com/search?q={encodedQuery.Value}";
    }
    
    string BuildComplexUrl(Dictionary<string, string> parameters)
    {
        var queryParams = string.Join("&", 
            parameters.Select(kvp => 
                $"{UrlEncodedString.Encode(kvp.Key).Value}={UrlEncodedString.Encode(kvp.Value).Value}"
            )
        );
        
        return $"https://api.example.com/search?{queryParams}";
    }
}
```

## Advanced: Compile-Time Template Validation

```csharp
/// <summary>
/// SQL query builder that validates syntax at compile time.
/// </summary>
public sealed class SqlQuery<TResult>
{
    string Template { get; }
    List<SqlParameter> Parameters { get; }
    
    private SqlQuery(string template, List<SqlParameter> parameters)
    {
        Template = template;
        Parameters = parameters;
    }
    
    public static SqlQuery<TResult> Create(FormattableString sql)
    {
        // Parse SQL at construction time to validate syntax
        var validator = new SqlSyntaxValidator();
        
        if (!validator.IsValid(sql.Format))
            throw new InvalidOperationException($"Invalid SQL syntax: {sql.Format}");
        
        var parameters = new List<SqlParameter>();
        
        for (int i = 0; i < sql.ArgumentCount; i++)
        {
            parameters.Add(new SqlParameter($"@p{i}", sql.GetArgument(i)));
        }
        
        return new SqlQuery<TResult>(sql.Format, parameters);
    }
}

// Usage
SqlQuery<User> query = SqlQuery<User>.Create(
    $"SELECT * FROM Users WHERE Age > {minAge}"
);
// Validates SQL syntax when query object is created
```

## HTML Template Engine

```csharp
/// <summary>
/// HTML template with typed holes for safe interpolation.
/// </summary>
public sealed class HtmlTemplate
{
    string Template { get; }
    
    HtmlTemplate(string template)
    {
        Template = template;
    }
    
    public static HtmlTemplate Create(string template) => new(template);
    
    public HtmlString Render(params object[] values)
    {
        // Encode all interpolated values
        var encodedValues = values.Select(v => 
            System.Net.WebUtility.HtmlEncode(v?.ToString() ?? "")
        ).ToArray();
        
        var result = string.Format(Template, encodedValues);
        return HtmlString.Trusted(result);
    }
}

// Usage
var template = HtmlTemplate.Create("<div>Hello, {0}! Your score is {1}.</div>");
var html = template.Render(userName, score);
// All interpolated values are auto-encoded
```

## Database Extension Methods

```csharp
public static class DatabaseExtensions
{
    public static async Task<List<T>> QueryAsync<T>(
        this IDbConnection connection,
        SqlString sql)
    {
        // Sql is already parameterized—safe to execute
        using var command = connection.CreateCommand();
        command.CommandText = sql.Value;
        
        foreach (var (index, value) in sql.Parameters.Select((v, i) => (i, v)))
        {
            var parameter = command.CreateParameter();
            parameter.ParameterName = $"@p{index}";
            parameter.Value = value ?? DBNull.Value;
            command.Parameters.Add(parameter);
        }
        
        // Execute safely
        using var reader = await command.ExecuteReaderAsync();
        return await MapResults<T>(reader);
    }
    
    // This method should NOT exist—prevents unsafe usage
    // public static Task<List<T>> QueryAsync<T>(this IDbConnection conn, string sql)
}
```

## Why It's a Problem

1. **Injection vulnerabilities**: Most common web security flaw
2. **Human error**: Easy to forget escaping in one place
3. **No compile-time protection**: Bugs found in production or security audits
4. **Scattered sanitization**: Validation logic duplicated across codebase

## Symptoms

- Direct string concatenation for SQL, HTML, URLs
- `string.Format` with user input
- String interpolation `$""` with untrusted data
- Comments like "// TODO: sanitize this" or "// Remember to escape"
- Security vulnerabilities in penetration tests

## Benefits

- **Injection impossible**: Type system prevents unsafe construction
- **Explicit encoding**: Must consciously encode user input
- **Single point of validation**: Encoding happens in trusted type constructors
- **Compile-time safety**: Won't compile without proper handling
- **Security by construction**: Architecture prevents vulnerabilities

## Trade-offs

- **Verbosity**: More types and explicit encoding calls
- **Learning curve**: Team must understand trusted types pattern
- **Migration effort**: Existing code must be converted
- **Performance**: Minor overhead from wrapper types (mitigated by inlining)

## See Also

- [Input Sanitization (Trusted Types)](./input-sanitization.md) — general trusted types pattern
- [Capability Security](./capability-security.md) — authorization via types
- [Secret Types](./secret-types.md) — preventing accidental secret exposure

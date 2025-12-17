# String Handling Best Practices

> Use efficient string operations, avoid string concatenation in loops, and leverage modern C# string features.

## Problem

Inefficient string handling causes performance issues, memory allocations, and security vulnerabilities.

## Example

### ❌ Before

```csharp
public class ReportGenerator
{
    // String concatenation in loop (very slow)
    public string GenerateReport(List<Order> orders)
    {
        string report = "";
        foreach (var order in orders)
        {
            report += $"Order {order.Id}: {order.Total}\n";  // Creates new string each iteration!
        }
        return report;
    }

    // Unnecessary string allocations
    public string FormatCustomerName(string firstName, string lastName)
    {
        return firstName + " " + lastName;  // Multiple allocations
    }

    // Unsafe string concatenation
    public string BuildSqlQuery(string table, string userId)
    {
        return $"SELECT * FROM {table} WHERE UserId = '{userId}'";  // SQL injection!
    }
}
```

### ✅ After

```csharp
public class ReportGenerator
{
    // ✅ Use StringBuilder for concatenation in loops
    public string GenerateReport(List<Order> orders)
    {
        var report = new StringBuilder(orders.Count * 50);  // Preallocate capacity

        foreach (var order in orders)
        {
            report.AppendLine($"Order {order.Id}: {order.Total}");
        }

        return report.ToString();
    }

    // ✅ Use string interpolation (compiled to String.Format)
    public string FormatCustomerName(string firstName, string lastName)
    {
        return $"{firstName} {lastName}";
    }

    // ✅ Use parameterized queries
    public string BuildSqlQuery(TableName table, UserId userId)
    {
        // Don't build SQL strings manually!
        // Use ORM or parameterized queries
        return _queryBuilder.Build(table, userId);
    }
}
```

## Best Practices

### 1. Use StringBuilder for Loops

```csharp
// ❌ O(n²) performance
public string ConcatenateItems(List<string> items)
{
    string result = "";
    foreach (var item in items)
    {
        result += item;  // Creates new string each time
    }
    return result;
}

// ✅ O(n) performance
public string ConcatenateItems(List<string> items)
{
    var sb = new StringBuilder(items.Count * 10);  // Estimate capacity
    foreach (var item in items)
    {
        sb.Append(item);
    }
    return sb.ToString();
}

// ✅ Or use string.Join
public string ConcatenateItems(List<string> items)
{
    return string.Join("", items);
}
```

### 2. Use String Interpolation

```csharp
// ❌ Harder to read
string message = "Hello, " + firstName + " " + lastName + "!";

// ❌ Old-style formatting
string message = string.Format("Hello, {0} {1}!", firstName, lastName);

// ✅ Modern string interpolation
string message = $"Hello, {firstName} {lastName}!";
```

### 3. Use Span<T> for Zero-Allocation Parsing

```csharp
// ❌ Creates substrings (allocations)
public (string, string) ParseName(string fullName)
{
    var parts = fullName.Split(' ');
    return (parts[0], parts[1]);
}

// ✅ Use Span<T> for zero allocations
public (ReadOnlySpan<char>, ReadOnlySpan<char>) ParseName(
    ReadOnlySpan<char> fullName)
{
    var spaceIndex = fullName.IndexOf(' ');
    if (spaceIndex == -1)
    {
        return (fullName, ReadOnlySpan<char>.Empty);
    }

    return (fullName[..spaceIndex], fullName[(spaceIndex + 1)..]);
}
```

### 4. Use String.Equals with Comparison

```csharp
// ❌ Case-sensitive comparison (potential bug)
if (status == "active")
{
    // Won't match "Active" or "ACTIVE"
}

// ❌ ToLower creates allocation
if (status.ToLower() == "active")
{
    // Works but allocates
}

// ✅ Use StringComparison
if (status.Equals("active", StringComparison.OrdinalIgnoreCase))
{
    // No allocation, correct comparison
}
```

### 5. Avoid String.Format in Hot Paths

```csharp
// ❌ String.Format has overhead
for (int i = 0; i < 1000000; i++)
{
    var message = string.Format("Value: {0}", i);
    Log(message);
}

// ✅ Use string interpolation or StringBuilder
for (int i = 0; i < 1000000; i++)
{
    var message = $"Value: {i}";  // Slightly more efficient
    Log(message);
}

// ✅ Best: avoid allocation if possible
for (int i = 0; i < 1000000; i++)
{
    LogValue(i);  // Pass value directly, format in logger
}
```

### 6. Use String.IsNullOrWhiteSpace

```csharp
// ❌ Multiple checks
if (input != null && input.Trim().Length > 0)
{
    // Process
}

// ✅ Single method
if (!string.IsNullOrWhiteSpace(input))
{
    // Process
}
```

### 7. Use Raw String Literals (C# 11+)

```csharp
// ❌ Escaping quotes is error-prone
string json = "{\"name\": \"John\", \"age\": 30}";

string sql = "SELECT * FROM Users WHERE Name = 'John' AND Age > 30";

// ✅ Raw string literals (no escaping needed)
string json = """
{
    "name": "John",
    "age": 30
}
""";

string sql = """
SELECT * FROM Users 
WHERE Name = 'John' 
AND Age > 30
""";
```

### 8. Use String.Create for Custom Formatting

```csharp
// ❌ Multiple allocations
public string FormatPhoneNumber(string digits)
{
    return $"({digits.Substring(0, 3)}) {digits.Substring(3, 3)}-{digits.Substring(6, 4)}";
}

// ✅ Single allocation with String.Create
public string FormatPhoneNumber(string digits)
{
    return string.Create(14, digits, (span, d) =>
    {
        span[0] = '(';
        d.AsSpan(0, 3).CopyTo(span[1..]);
        span[4] = ')';
        span[5] = ' ';
        d.AsSpan(3, 3).CopyTo(span[6..]);
        span[9] = '-';
        d.AsSpan(6, 4).CopyTo(span[10..]);
    });
}
```

### 9. Avoid Substring When Possible

```csharp
// ❌ Substring allocates
public bool StartsWithHttp(string url)
{
    return url.Substring(0, 4) == "http";
}

// ✅ Use StartsWith (no allocation)
public bool StartsWithHttp(string url)
{
    return url.StartsWith("http", StringComparison.Ordinal);
}

// ✅ Use Span for complex parsing
public bool StartsWithHttp(ReadOnlySpan<char> url)
{
    return url.StartsWith("http".AsSpan(), StringComparison.Ordinal);
}
```

### 10. Pool StringBuilder Instances

```csharp
// ❌ Creates new StringBuilder each time
public string BuildReport()
{
    var sb = new StringBuilder();
    // Build report
    return sb.ToString();
}

// ✅ Use object pooling for high-throughput scenarios
private static readonly ObjectPool<StringBuilder> StringBuilderPool =
    new DefaultObjectPoolProvider().CreateStringBuilderPool();

public string BuildReport()
{
    var sb = StringBuilderPool.Get();
    try
    {
        // Build report
        return sb.ToString();
    }
    finally
    {
        sb.Clear();
        StringBuilderPool.Return(sb);
    }
}
```

### 11. Use UTF-8 for Network/File Operations

```csharp
// ❌ Default encoding varies by system
var bytes = Encoding.Default.GetBytes(text);

// ✅ Explicit UTF-8 encoding
var bytes = Encoding.UTF8.GetBytes(text);

// ✅ Even better: use UTF-8 directly
var bytes = System.Text.Encoding.UTF8.GetBytes(text);
```

### 12. Validate and Sanitize User Input

```csharp
// ❌ Direct use of user input
public string BuildGreeting(string userName)
{
    return $"<h1>Hello, {userName}!</h1>";  // XSS vulnerability!
}

// ✅ Sanitize HTML
public string BuildGreeting(string userName)
{
    var sanitized = System.Web.HttpUtility.HtmlEncode(userName);
    return $"<h1>Hello, {sanitized}!</h1>";
}

// ✅ Better: use templating engine
public string BuildGreeting(string userName)
{
    return _templateEngine.Render("greeting", new { UserName = userName });
}
```

## Symptoms

- Poor performance with large strings
- High memory allocation rates
- SQL injection vulnerabilities
- XSS vulnerabilities in web applications
- Encoding issues with international characters

## Benefits

- **Better performance** with efficient string operations
- **Lower memory pressure** by avoiding unnecessary allocations
- **Security** by properly handling user input
- **Correctness** with proper string comparisons
- **Maintainability** with clear, readable string code

## See Also

- [Type-Safe String Interpolation](./type-safe-string-interpolation.md) — Preventing injection attacks
- [Input Sanitization](./input-sanitization.md) — Trusted types for validated input
- [Memory Safety](./memory-safety-span.md) — Span<T> for zero-allocation code
- [SQL Injection Prevention](./sql-injection-prevention.md) — Safe database queries

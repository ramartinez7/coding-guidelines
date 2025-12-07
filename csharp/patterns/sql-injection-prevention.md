# SQL Injection Prevention (Type-Safe Queries)

> Building SQL with string concatenation or interpolation—use parameterized queries and typed SQL builders to make injection attacks unrepresentable.

## Problem

SQL injection occurs when untrusted input is concatenated directly into SQL queries. Even experienced developers can accidentally create vulnerabilities through string interpolation or concatenation. There's no compile-time protection against building dangerous queries.

## Example

### ❌ Before

```csharp
public class UserRepository
{
    public User? GetUserByEmail(string email)
    {
        // Dangerous: SQL injection vulnerability!
        var sql = $"SELECT * FROM Users WHERE Email = '{email}'";
        // Attacker input: "' OR '1'='1" returns all users
        // Attacker input: "'; DROP TABLE Users; --" destroys data
        
        return _connection.Query<User>(sql).FirstOrDefault();
    }
    
    public List<Product> SearchProducts(string searchTerm, string category)
    {
        // Even with "safe" looking code, concatenation is dangerous
        var sql = "SELECT * FROM Products WHERE Name LIKE '%" + searchTerm + "%'";
        
        if (!string.IsNullOrEmpty(category))
        {
            sql += " AND Category = '" + category + "'";
        }
        
        return _connection.Query<Product>(sql).ToList();
    }
    
    public void UpdateUserStatus(int userId, string status)
    {
        // Vulnerable even with numeric IDs
        var sql = $"UPDATE Users SET Status = '{status}' WHERE Id = {userId}";
        _connection.Execute(sql);
    }
}
```

**Problems:**
- String concatenation/interpolation creates injection vulnerabilities
- No compile-time detection of unsafe queries
- Easy to make mistakes even with validation
- Dynamic query building is error-prone
- Cannot audit which queries use untrusted input

### ✅ After

```csharp
/// <summary>
/// SQL parameter that has been properly parameterized.
/// Cannot be constructed from raw strings—must go through parameter creation.
/// </summary>
public sealed record SqlParameter
{
    public string Name { get; }
    public object? Value { get; }
    
    private SqlParameter(string name, object? value)
    {
        Name = name;
        Value = value;
    }
    
    public static SqlParameter Create(string name, object? value)
    {
        if (string.IsNullOrWhiteSpace(name))
            throw new ArgumentException("Parameter name cannot be empty", nameof(name));
            
        if (!name.StartsWith("@"))
            name = "@" + name;
            
        return new SqlParameter(name, value);
    }
}

/// <summary>
/// Type-safe SQL query builder that prevents injection.
/// </summary>
public sealed class SafeSqlQuery
{
    public string QueryText { get; }
    public IReadOnlyDictionary<string, object?> Parameters { get; }
    
    private SafeSqlQuery(string queryText, Dictionary<string, object?> parameters)
    {
        QueryText = queryText;
        Parameters = parameters.AsReadOnly();
    }
    
    public static SafeSqlQuery Create(
        FormattableString query,
        Func<FormattableString, (string sql, Dictionary<string, object?> parameters)> builder)
    {
        var (sql, parameters) = builder(query);
        return new SafeSqlQuery(sql, parameters);
    }
}

/// <summary>
/// Builder for creating parameterized SQL queries.
/// </summary>
public static class SafeSql
{
    public static SafeSqlQuery Query(FormattableString query)
    {
        var parameters = new Dictionary<string, object?>();
        var args = query.GetArguments();
        var format = query.Format;
        
        // Replace each {index} with @param_index
        var sql = format;
        for (int i = 0; i < args.Length; i++)
        {
            var paramName = $"@param_{i}";
            parameters[paramName] = args[i];
            
            // Replace the placeholder
            sql = sql.Replace($"{{{i}}}", paramName);
        }
        
        return new SafeSqlQuery(sql, parameters);
    }
    
    public static SafeSqlQuery FromTemplate(
        string template,
        params SqlParameter[] parameters)
    {
        var paramDict = parameters.ToDictionary(p => p.Name, p => p.Value);
        return new SafeSqlQuery(template, paramDict);
    }
}

public class UserRepository
{
    public User? GetUserByEmail(Email email)
    {
        // Automatically parameterized—injection impossible
        var query = SafeSql.Query($"SELECT * FROM Users WHERE Email = {email.Value}");
        
        return _connection.Query<User>(
            query.QueryText,
            query.Parameters).FirstOrDefault();
    }
    
    public List<Product> SearchProducts(string searchTerm, string category)
    {
        // Multiple parameters, all safely parameterized
        var query = SafeSql.Query(
            $"SELECT * FROM Products WHERE Name LIKE {'%' + searchTerm + '%'} AND Category = {category}");
        
        return _connection.Query<Product>(
            query.QueryText,
            query.Parameters).ToList();
    }
    
    public void UpdateUserStatus(UserId userId, UserStatus status)
    {
        // Even with multiple parameters, safe
        var query = SafeSql.Query(
            $"UPDATE Users SET Status = {status.ToString()} WHERE Id = {userId.Value}");
        
        _connection.Execute(query.QueryText, query.Parameters);
    }
}
```

## Advanced Patterns

### Query Builder with Fluent API

```csharp
public sealed class SqlQueryBuilder
{
    private readonly List<string> _selectColumns = new();
    private readonly List<string> _fromTables = new();
    private readonly List<WhereClause> _whereClauses = new();
    private readonly Dictionary<string, object?> _parameters = new();
    private int _parameterCounter = 0;
    
    private sealed record WhereClause(string Condition);
    
    public SqlQueryBuilder Select(params string[] columns)
    {
        _selectColumns.AddRange(columns);
        return this;
    }
    
    public SqlQueryBuilder From(string table)
    {
        _fromTables.Add(table);
        return this;
    }
    
    public SqlQueryBuilder Where(string column, object? value)
    {
        var paramName = $"@p{_parameterCounter++}";
        _parameters[paramName] = value;
        _whereClauses.Add(new WhereClause($"{column} = {paramName}"));
        return this;
    }
    
    public SqlQueryBuilder WhereLike(string column, string value)
    {
        var paramName = $"@p{_parameterCounter++}";
        _parameters[paramName] = $"%{value}%";
        _whereClauses.Add(new WhereClause($"{column} LIKE {paramName}"));
        return this;
    }
    
    public SqlQueryBuilder WhereIn(string column, IEnumerable<object> values)
    {
        var paramNames = new List<string>();
        foreach (var value in values)
        {
            var paramName = $"@p{_parameterCounter++}";
            _parameters[paramName] = value;
            paramNames.Add(paramName);
        }
        
        _whereClauses.Add(new WhereClause($"{column} IN ({string.Join(", ", paramNames)})"));
        return this;
    }
    
    public SafeSqlQuery Build()
    {
        if (_selectColumns.Count == 0)
            throw new InvalidOperationException("SELECT columns required");
            
        if (_fromTables.Count == 0)
            throw new InvalidOperationException("FROM table required");
        
        var sql = $"SELECT {string.Join(", ", _selectColumns)} " +
                  $"FROM {string.Join(", ", _fromTables)}";
        
        if (_whereClauses.Count > 0)
        {
            var conditions = string.Join(" AND ", _whereClauses.Select(w => w.Condition));
            sql += $" WHERE {conditions}";
        }
        
        return new SafeSqlQuery(sql, _parameters);
    }
}

// Usage
public List<Order> GetCustomerOrders(CustomerId customerId, OrderStatus status)
{
    var query = new SqlQueryBuilder()
        .Select("OrderId", "OrderDate", "TotalAmount", "Status")
        .From("Orders")
        .Where("CustomerId", customerId.Value)
        .Where("Status", status.ToString())
        .Build();
    
    return _connection.Query<Order>(query.QueryText, query.Parameters).ToList();
}

public List<Product> SearchProducts(string searchTerm, List<string> categories)
{
    var query = new SqlQueryBuilder()
        .Select("*")
        .From("Products")
        .WhereLike("Name", searchTerm)
        .WhereIn("Category", categories.Cast<object>())
        .Build();
    
    return _connection.Query<Product>(query.QueryText, query.Parameters).ToList();
}
```

### Stored Procedure Wrapper

```csharp
/// <summary>
/// Type-safe stored procedure invocation.
/// </summary>
public sealed record StoredProcedure(string Name, IReadOnlyDictionary<string, object?> Parameters)
{
    public static StoredProcedure Create(string name, params SqlParameter[] parameters)
    {
        var paramDict = parameters.ToDictionary(p => p.Name, p => p.Value);
        return new StoredProcedure(name, paramDict);
    }
}

public class OrderRepository
{
    public List<Order> GetOrdersByDateRange(DateTime startDate, DateTime endDate)
    {
        var procedure = StoredProcedure.Create(
            "GetOrdersByDateRange",
            SqlParameter.Create("StartDate", startDate),
            SqlParameter.Create("EndDate", endDate));
        
        return _connection.Query<Order>(
            procedure.Name,
            procedure.Parameters,
            commandType: CommandType.StoredProcedure).ToList();
    }
}
```

### ORM Integration (Entity Framework Core)

```csharp
public class UserRepository
{
    private readonly DbContext _context;
    
    // EF Core automatically parameterizes LINQ queries
    public User? GetUserByEmail(Email email)
    {
        // Safe: EF Core parameterizes this
        return _context.Users
            .Where(u => u.Email == email.Value)
            .FirstOrDefault();
    }
    
    public List<Product> SearchProducts(string searchTerm)
    {
        // Safe: EF Core parameterizes all LINQ expressions
        return _context.Products
            .Where(p => EF.Functions.Like(p.Name, $"%{searchTerm}%"))
            .ToList();
    }
    
    // ⚠️ WARNING: FromSqlRaw requires manual parameterization
    public List<User> RawSqlQuery(string status)
    {
        // ❌ Vulnerable to injection
        // return _context.Users.FromSqlRaw($"SELECT * FROM Users WHERE Status = '{status}'").ToList();
        
        // ✅ Use FromSqlRaw with parameters
        return _context.Users
            .FromSqlRaw("SELECT * FROM Users WHERE Status = {0}", status)
            .ToList();
    }
    
    // ✅ Better: Use FromSqlInterpolated
    public List<User> SafeRawSqlQuery(string status)
    {
        // Automatically parameterized
        return _context.Users
            .FromSqlInterpolated($"SELECT * FROM Users WHERE Status = {status}")
            .ToList();
    }
}
```

### Dapper Integration

```csharp
public static class DapperExtensions
{
    public static IEnumerable<T> QuerySafe<T>(
        this IDbConnection connection,
        SafeSqlQuery query)
    {
        return connection.Query<T>(query.QueryText, query.Parameters);
    }
    
    public static int ExecuteSafe(
        this IDbConnection connection,
        SafeSqlQuery query)
    {
        return connection.Execute(query.QueryText, query.Parameters);
    }
}

public class ProductRepository
{
    public List<Product> SearchProducts(string term, decimal minPrice)
    {
        var query = SafeSql.Query(
            $"SELECT * FROM Products WHERE Name LIKE {'%' + term + '%'} AND Price >= {minPrice}");
        
        return _connection.QuerySafe<Product>(query).ToList();
    }
}
```

## Testing

```csharp
public class SqlInjectionTests
{
    [Fact]
    public void SafeSql_ParameterizesInput()
    {
        var maliciousInput = "'; DROP TABLE Users; --";
        var query = SafeSql.Query($"SELECT * FROM Users WHERE Email = {maliciousInput}");
        
        // The query text should not contain the malicious input literally
        Assert.DoesNotContain("DROP TABLE", query.QueryText);
        
        // The parameter should contain the full input
        Assert.Contains(maliciousInput, query.Parameters.Values.Cast<string>());
    }
    
    [Fact]
    public void QueryBuilder_ParameterizesWhereClause()
    {
        var maliciousCategory = "' OR '1'='1";
        
        var query = new SqlQueryBuilder()
            .Select("*")
            .From("Products")
            .Where("Category", maliciousCategory)
            .Build();
        
        // Should be parameterized, not concatenated
        Assert.DoesNotContain("OR '1'='1'", query.QueryText);
        Assert.Contains(maliciousCategory, query.Parameters.Values.Cast<string>());
    }
    
    [Fact]
    public void GetUserByEmail_WithMaliciousInput_DoesNotExecuteInjection()
    {
        var repository = new UserRepository(_mockConnection.Object);
        var maliciousEmail = Email.Create("test@example.com' OR '1'='1").Value;
        
        repository.GetUserByEmail(maliciousEmail);
        
        // Verify the executed query was parameterized
        _mockConnection.Verify(c => c.Query<User>(
            It.Is<string>(sql => !sql.Contains("OR '1'='1'")),
            It.IsAny<object>()));
    }
}
```

## Why It's a Problem

1. **SQL injection vulnerability**: Untrusted input can manipulate query logic
2. **Data exfiltration**: Attackers can access unauthorized data
3. **Data destruction**: `DROP TABLE` commands can destroy data
4. **Authentication bypass**: `OR '1'='1'` can bypass login checks
5. **No compile-time detection**: String concatenation looks innocent

## Symptoms

- String concatenation or interpolation in SQL queries
- Dynamic SQL built with `+` or `$"{}"` operators
- No parameterization in database queries
- Security vulnerabilities found in penetration testing
- SQL error messages revealing injection attempts

## Benefits

- **Injection-proof**: Parameterization prevents SQL injection at the protocol level
- **Compile-time safety**: Type system enforces safe query construction
- **Self-documenting**: `SafeSqlQuery` makes it clear the query is parameterized
- **Auditable**: Easy to identify which queries handle untrusted input
- **Performance**: Parameterized queries can be cached by the database

## Trade-offs

- **More verbose**: Requires wrapping queries in `SafeSql.Query()`
- **Learning curve**: Team must understand parameterization
- **ORM migration**: Existing string-based queries need refactoring
- **Dynamic queries**: Complex dynamic queries may need query builder

## See Also

- [Input Sanitization](./input-sanitization.md) — trusted types at boundaries
- [Type-Safe String Interpolation](./type-safe-string-interpolation.md) — preventing injection
- [Honest Functions](./honest-functions.md) — explicit error handling
- [Command Injection Prevention](./command-injection-prevention.md) — similar pattern for shell commands

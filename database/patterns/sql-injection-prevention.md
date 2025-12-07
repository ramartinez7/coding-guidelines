# SQL Injection Prevention

> Use parameterized queries and stored procedures—never concatenate user input into SQL strings.

## Problem

Building SQL queries by concatenating strings with user input creates SQL injection vulnerabilities. Attackers can inject malicious SQL code that executes with the database connection's privileges, potentially reading, modifying, or deleting data.

## Example

### ❌ Before (Vulnerable)

```sql
-- NEVER DO THIS: String concatenation creates injection vulnerability
CREATE PROCEDURE dbo.usp_GetCustomerByEmail_Unsafe
    @Email NVARCHAR(255)
AS
BEGIN
    DECLARE @SQL NVARCHAR(MAX);
    
    -- ❌ Dangerous: concatenating user input into SQL
    SET @SQL = 'SELECT CustomerId, FirstName, LastName, Email FROM dbo.Customer WHERE Email = ''' + @Email + '''';
    
    EXEC sp_executesql @SQL;
END;

-- Looks innocent:
EXEC dbo.usp_GetCustomerByEmail_Unsafe @Email = 'alice@example.com';

-- But attacker can inject SQL:
EXEC dbo.usp_GetCustomerByEmail_Unsafe @Email = 'alice@example.com'' OR ''1''=''1';
-- Executes: SELECT ... WHERE Email = 'alice@example.com' OR '1'='1'
-- Returns ALL customers!

-- Even worse:
EXEC dbo.usp_GetCustomerByEmail_Unsafe @Email = 'alice@example.com''; DROP TABLE dbo.Customer; --';
-- Executes: SELECT ... WHERE Email = 'alice@example.com'; DROP TABLE dbo.Customer; --'
-- DESTROYS THE TABLE!
```

### ✅ After (Safe)

```sql
-- ✅ Safe: Parameterized query
CREATE PROCEDURE dbo.usp_GetCustomerByEmail
    @Email NVARCHAR(255)
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Parameter is safely passed to the query
    SELECT 
        CustomerId,
        FirstName,
        LastName,
        Email
    FROM dbo.Customer
    WHERE Email = @Email;
END;

-- Injection attempts fail safely:
EXEC dbo.usp_GetCustomerByEmail @Email = 'alice@example.com'' OR ''1''=''1';
-- Looks for literal email address: "alice@example.com' OR '1'='1"
-- Returns no results (safe)

EXEC dbo.usp_GetCustomerByEmail @Email = 'alice@example.com''; DROP TABLE dbo.Customer; --';
-- Looks for literal email: "alice@example.com'; DROP TABLE dbo.Customer; --"
-- Returns no results (safe, table not dropped)
```

## Safe Patterns

### 1. Direct Parameterized Queries

The safest approach—use parameters directly in the query.

```sql
CREATE PROCEDURE dbo.usp_GetOrdersByCustomer
    @CustomerId INT,
    @StartDate DATETIME2,
    @EndDate DATETIME2
AS
BEGIN
    SET NOCOUNT ON;
    
    -- All parameters are safe
    SELECT 
        OrderId,
        OrderDate,
        Total,
        Status
    FROM dbo.Order
    WHERE CustomerId = @CustomerId
      AND OrderDate >= @StartDate
      AND OrderDate < @EndDate
    ORDER BY OrderDate DESC;
END;
```

### 2. sp_executesql with Parameters

When dynamic SQL is necessary, use `sp_executesql` with parameters.

```sql
CREATE PROCEDURE dbo.usp_GetCustomersByColumn
    @ColumnName NVARCHAR(50),  -- Column to filter on
    @Value NVARCHAR(255)       -- Value to search for
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Whitelist column names (don't trust user input for identifiers)
    IF @ColumnName NOT IN ('Email', 'Phone', 'LastName', 'FirstName')
    BEGIN
        THROW 50001, 'Invalid column name', 1;
        RETURN;
    END;
    
    DECLARE @SQL NVARCHAR(MAX);
    DECLARE @Params NVARCHAR(MAX);
    
    -- Use sp_executesql with parameters for values
    SET @SQL = N'SELECT CustomerId, FirstName, LastName, Email, Phone 
                 FROM dbo.Customer 
                 WHERE ' + QUOTENAME(@ColumnName) + ' = @SearchValue';
    
    SET @Params = N'@SearchValue NVARCHAR(255)';
    
    -- Parameter is safely passed
    EXEC sp_executesql @SQL, @Params, @SearchValue = @Value;
END;

-- Safe usage:
EXEC dbo.usp_GetCustomersByColumn @ColumnName = 'Email', @Value = 'alice@example.com';

-- Injection attempt fails:
EXEC dbo.usp_GetCustomersByColumn @ColumnName = 'Email'' OR ''1''=''1', @Value = 'test';
-- Throws: Invalid column name
```

### 3. QUOTENAME for Identifiers

When you must dynamically include table or column names, use `QUOTENAME()` to safely escape them.

```sql
CREATE PROCEDURE dbo.usp_GetTableData
    @SchemaName NVARCHAR(128),
    @TableName NVARCHAR(128)
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Validate schema and table exist (whitelist approach)
    IF NOT EXISTS (
        SELECT 1 
        FROM INFORMATION_SCHEMA.TABLES 
        WHERE TABLE_SCHEMA = @SchemaName 
          AND TABLE_NAME = @TableName
    )
    BEGIN
        THROW 50002, 'Table does not exist', 1;
        RETURN;
    END;
    
    DECLARE @SQL NVARCHAR(MAX);
    
    -- QUOTENAME safely escapes identifiers
    SET @SQL = N'SELECT * FROM ' + QUOTENAME(@SchemaName) + '.' + QUOTENAME(@TableName);
    
    EXEC sp_executesql @SQL;
END;
```

## Validation Strategies

### Input Validation

Validate and sanitize inputs before using them, especially for identifiers.

```sql
CREATE PROCEDURE dbo.usp_SearchOrders
    @SortColumn NVARCHAR(50),
    @SortDirection NVARCHAR(4)
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Whitelist allowed columns
    IF @SortColumn NOT IN ('OrderId', 'OrderDate', 'Total', 'Status')
    BEGIN
        THROW 50003, 'Invalid sort column', 1;
        RETURN;
    END;
    
    -- Whitelist allowed sort directions
    IF @SortDirection NOT IN ('ASC', 'DESC')
    BEGIN
        THROW 50004, 'Invalid sort direction', 1;
        RETURN;
    END;
    
    DECLARE @SQL NVARCHAR(MAX);
    
    -- Safe because inputs are validated
    SET @SQL = N'SELECT OrderId, CustomerId, OrderDate, Total, Status 
                 FROM dbo.Order 
                 ORDER BY ' + QUOTENAME(@SortColumn) + ' ' + @SortDirection;
    
    EXEC sp_executesql @SQL;
END;
```

### Use Stored Procedures

Stored procedures provide an additional layer of security by limiting direct table access.

```sql
-- Grant execute permission on procedure, not direct table access
GRANT EXECUTE ON dbo.usp_GetCustomerByEmail TO [ApplicationUser];

-- Do NOT grant direct SELECT access
-- DENY SELECT ON dbo.Customer TO [ApplicationUser];
```

## Application-Level Best Practices

### ADO.NET (C#)

```csharp
// ❌ NEVER concatenate user input
string email = userInput;
string sql = $"SELECT * FROM Customer WHERE Email = '{email}'";
command.CommandText = sql;

// ✅ Use parameters
string email = userInput;
command.CommandText = "SELECT CustomerId, FirstName, LastName, Email FROM Customer WHERE Email = @Email";
command.Parameters.AddWithValue("@Email", email);
```

### Entity Framework / Dapper

```csharp
// ✅ Entity Framework uses parameters automatically
var customers = dbContext.Customers
    .Where(c => c.Email == email)
    .ToList();

// ✅ Dapper with parameters
var customers = connection.Query<Customer>(
    "SELECT CustomerId, FirstName, LastName, Email FROM Customer WHERE Email = @Email",
    new { Email = email }
);

// ❌ NEVER use string interpolation in raw SQL
var sql = $"SELECT * FROM Customer WHERE Email = '{email}'";
var customers = connection.Query<Customer>(sql);  // VULNERABLE!
```

## Why It's a Problem

1. **Data breach**: Attackers can read sensitive data they shouldn't access
2. **Data corruption**: Malicious SQL can modify or delete data
3. **Privilege escalation**: Attackers can execute operations as the application's database user
4. **Compliance violations**: SQL injection vulnerabilities fail security audits (PCI-DSS, SOC 2, etc.)

## Symptoms

- String concatenation or interpolation used to build SQL queries
- `EXEC()` or `sp_executesql` called without parameters
- User input directly embedded in SQL strings
- Dynamic table or column names from untrusted sources

## Benefits

- **Security**: Prevents SQL injection attacks completely
- **Reliability**: Queries fail safely with invalid input instead of executing malicious code
- **Performance**: Parameterized queries enable query plan reuse
- **Maintainability**: Clear separation between SQL structure and data values

## Common Mistakes

### Mistake 1: Trusting "Sanitized" Input

```sql
-- ❌ Still vulnerable even with basic sanitization
DECLARE @Email NVARCHAR(255) = REPLACE(@UserInput, '''', '''''');  -- Escape single quotes
DECLARE @SQL NVARCHAR(MAX) = 'SELECT * FROM Customer WHERE Email = ''' + @Email + '''';
EXEC sp_executesql @SQL;

-- ✅ Use parameters instead
SELECT * FROM Customer WHERE Email = @UserInput;
```

### Mistake 2: Validating Then Concatenating

```sql
-- ❌ Validation doesn't make concatenation safe
IF @Email LIKE '%@%.%'  -- Basic email validation
BEGIN
    DECLARE @SQL NVARCHAR(MAX) = 'SELECT * FROM Customer WHERE Email = ''' + @Email + '''';
    EXEC sp_executesql @SQL;  -- Still vulnerable!
END

-- ✅ Validate AND use parameters
IF @Email LIKE '%@%.%'
BEGIN
    SELECT * FROM Customer WHERE Email = @Email;
END
```

### Mistake 3: Only Parameterizing Some Inputs

```sql
-- ❌ Column name is still vulnerable
DECLARE @SQL NVARCHAR(MAX);
SET @SQL = 'SELECT * FROM Customer WHERE ' + @ColumnName + ' = @Value';
EXEC sp_executesql @SQL, N'@Value NVARCHAR(255)', @Value = @SearchValue;

-- ✅ Whitelist column names
IF @ColumnName NOT IN ('Email', 'Phone', 'LastName')
    THROW 50001, 'Invalid column', 1;
    
DECLARE @SQL NVARCHAR(MAX);
SET @SQL = 'SELECT * FROM Customer WHERE ' + QUOTENAME(@ColumnName) + ' = @Value';
EXEC sp_executesql @SQL, N'@Value NVARCHAR(255)', @Value = @SearchValue;
```

## See Also

- [Parameterized Queries](./parameterized-queries.md)
- [Least Privilege](./least-privilege.md)
- [Row-Level Security](./row-level-security.md)

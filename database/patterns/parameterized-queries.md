# Parameterized Queries

> Always use parameters instead of string concatenation—prevent SQL injection and enable query plan reuse.

## Problem

Concatenating values directly into SQL strings creates SQL injection vulnerabilities and prevents query plan caching, hurting both security and performance.

## Example

### ❌ Before (String Concatenation)

```sql
-- DANGEROUS: SQL Injection vulnerability
DECLARE @Email NVARCHAR(255) = @UserInput;
DECLARE @SQL NVARCHAR(MAX);

SET @SQL = 'SELECT CustomerId, FirstName, LastName 
            FROM dbo.Customer 
            WHERE Email = ''' + @Email + '''';

EXEC(@SQL);

-- Problems:
-- 1. SQL Injection: malicious input can execute arbitrary SQL
-- 2. No plan reuse: each query text is unique
-- 3. Hard to read and maintain
```

### ✅ After (Parameterized)

```sql
-- Safe and efficient
DECLARE @Email NVARCHAR(255) = @UserInput;

SELECT 
    CustomerId,
    FirstName,
    LastName
FROM dbo.Customer
WHERE Email = @Email;

-- Benefits:
-- 1. SQL Injection proof: @Email treated as data, not code
-- 2. Plan reuse: same plan for all email values
-- 3. Clean and readable
```

## Dynamic SQL with Parameters

When dynamic SQL is necessary, use `sp_executesql` with parameters:

```sql
-- ✅ Safe dynamic SQL with parameters
CREATE PROCEDURE dbo.usp_SearchCustomers
    @SearchColumn NVARCHAR(50),
    @SearchValue NVARCHAR(255)
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Whitelist allowed columns
    IF @SearchColumn NOT IN ('Email', 'FirstName', 'LastName', 'Phone')
    BEGIN
        THROW 50001, 'Invalid search column', 1;
    END;
    
    DECLARE @SQL NVARCHAR(MAX);
    DECLARE @Params NVARCHAR(MAX);
    
    -- Build query with parameter placeholder
    SET @SQL = N'SELECT CustomerId, FirstName, LastName, Email 
                 FROM dbo.Customer 
                 WHERE ' + QUOTENAME(@SearchColumn) + ' = @SearchValue';
    
    -- Define parameter types
    SET @Params = N'@SearchValue NVARCHAR(255)';
    
    -- Execute with parameters
    EXEC sp_executesql @SQL, @Params, @SearchValue = @SearchValue;
END;

-- Safe usage
EXEC dbo.usp_SearchCustomers @SearchColumn = 'Email', @SearchValue = 'alice@example.com';

-- Injection attempt fails safely
EXEC dbo.usp_SearchCustomers @SearchColumn = 'Email', @SearchValue = '''; DROP TABLE Customer; --';
-- @SearchValue treated as literal string, not executed
```

## Multiple Parameters

```sql
-- Multiple parameters in sp_executesql
CREATE PROCEDURE dbo.usp_GetOrdersByDateRange
    @CustomerId INT,
    @StartDate DATE,
    @EndDate DATE,
    @MinTotal DECIMAL(18,2)
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @SQL NVARCHAR(MAX);
    DECLARE @Params NVARCHAR(MAX);
    
    SET @SQL = N'SELECT OrderId, OrderDate, Total, Status
                 FROM dbo.Order
                 WHERE CustomerId = @CustomerId
                   AND OrderDate >= @StartDate
                   AND OrderDate < @EndDate
                   AND Total >= @MinTotal
                 ORDER BY OrderDate DESC';
    
    SET @Params = N'@CustomerId INT,
                    @StartDate DATE,
                    @EndDate DATE,
                    @MinTotal DECIMAL(18,2)';
    
    EXEC sp_executesql 
        @SQL, 
        @Params,
        @CustomerId = @CustomerId,
        @StartDate = @StartDate,
        @EndDate = @EndDate,
        @MinTotal = @MinTotal;
END;
```

## Parameter Sniffing

Be aware of parameter sniffing with cached plans:

```sql
-- Parameter sniffing can cause performance issues
-- Query optimized for first parameter value used

CREATE PROCEDURE dbo.usp_GetOrdersByStatus
    @Status VARCHAR(20)
AS
BEGIN
    SET NOCOUNT ON;
    
    -- First call with 'Pending': creates plan for few rows
    -- Subsequent calls with 'Completed': plan may be inefficient
    SELECT 
        OrderId,
        CustomerId,
        OrderDate,
        Total
    FROM dbo.Order
    WHERE Status = @Status;
END;

-- Solutions:

-- 1. OPTION (RECOMPILE) - recompile each time
CREATE PROCEDURE dbo.usp_GetOrdersByStatus_Recompile
    @Status VARCHAR(20)
AS
BEGIN
    SET NOCOUNT ON;
    
    SELECT 
        OrderId,
        CustomerId,
        OrderDate,
        Total
    FROM dbo.Order
    WHERE Status = @Status
    OPTION (RECOMPILE);  -- New plan every time
END;

-- 2. OPTION (OPTIMIZE FOR) - optimize for specific value
CREATE PROCEDURE dbo.usp_GetOrdersByStatus_OptimizeFor
    @Status VARCHAR(20)
AS
BEGIN
    SET NOCOUNT ON;
    
    SELECT 
        OrderId,
        CustomerId,
        OrderDate,
        Total
    FROM dbo.Order
    WHERE Status = @Status
    OPTION (OPTIMIZE FOR (@Status = 'Completed'));  -- Optimize for most common case
END;

-- 3. Local variable (avoids sniffing)
CREATE PROCEDURE dbo.usp_GetOrdersByStatus_LocalVar
    @Status VARCHAR(20)
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @LocalStatus VARCHAR(20) = @Status;
    
    SELECT 
        OrderId,
        CustomerId,
        OrderDate,
        Total
    FROM dbo.Order
    WHERE Status = @LocalStatus;  -- Average plan for all values
END;
```

## IN Clause with Parameters

Table-valued parameters for dynamic IN clauses:

```sql
-- Create table type
CREATE TYPE dbo.IntListType AS TABLE (Value INT);
GO

-- Use in procedure
CREATE PROCEDURE dbo.usp_GetCustomersByIds
    @CustomerIds dbo.IntListType READONLY
AS
BEGIN
    SET NOCOUNT ON;
    
    SELECT 
        c.CustomerId,
        c.FirstName,
        c.LastName,
        c.Email
    FROM dbo.Customer c
    INNER JOIN @CustomerIds ids ON c.CustomerId = ids.Value;
END;
GO

-- Usage from application (C#)
/*
var customerIds = new DataTable();
customerIds.Columns.Add("Value", typeof(int));
customerIds.Rows.Add(1);
customerIds.Rows.Add(2);
customerIds.Rows.Add(3);

var param = new SqlParameter("@CustomerIds", SqlDbType.Structured)
{
    TypeName = "dbo.IntListType",
    Value = customerIds
};

command.Parameters.Add(param);
*/
```

## Application-Level Examples

### ADO.NET (C#)

```csharp
// ❌ NEVER concatenate
string email = userInput;
string sql = $"SELECT * FROM Customer WHERE Email = '{email}'";  // VULNERABLE!
command.CommandText = sql;

// ✅ Use parameters
string email = userInput;
command.CommandText = "SELECT CustomerId, FirstName, LastName FROM Customer WHERE Email = @Email";
command.Parameters.AddWithValue("@Email", email);  // Safe

// ✅ Better: specify type explicitly
command.Parameters.Add("@Email", SqlDbType.NVarChar, 255).Value = email;
```

### Dapper (C#)

```csharp
// ✅ Dapper uses parameters automatically
var customers = connection.Query<Customer>(
    "SELECT CustomerId, FirstName, LastName FROM Customer WHERE Email = @Email",
    new { Email = email }  // Anonymous object becomes parameters
);

// ✅ Multiple parameters
var orders = connection.Query<Order>(
    @"SELECT OrderId, OrderDate, Total 
      FROM [Order] 
      WHERE CustomerId = @CustomerId 
        AND OrderDate >= @StartDate",
    new { 
        CustomerId = customerId,
        StartDate = startDate
    }
);
```

### Entity Framework Core (C#)

```csharp
// ✅ LINQ uses parameters automatically
var customers = dbContext.Customers
    .Where(c => c.Email == email)  // Parameterized
    .ToList();

// ✅ Raw SQL with parameters
var customers = dbContext.Customers
    .FromSqlRaw("SELECT * FROM Customer WHERE Email = {0}", email)
    .ToList();

// ✅ Named parameters
var customers = dbContext.Customers
    .FromSqlRaw("SELECT * FROM Customer WHERE Email = @email", 
        new SqlParameter("@email", email))
    .ToList();

// ❌ NEVER use string interpolation in FromSqlRaw
var customers = dbContext.Customers
    .FromSqlRaw($"SELECT * FROM Customer WHERE Email = '{email}'")  // VULNERABLE!
    .ToList();

// ✅ Use FromSqlInterpolated for string interpolation
var customers = dbContext.Customers
    .FromSqlInterpolated($"SELECT * FROM Customer WHERE Email = {email}")  // Safe
    .ToList();
```

## Special Cases

### LIKE Patterns

```sql
-- ✅ Parameterize LIKE patterns
DECLARE @Pattern NVARCHAR(255) = @UserInput + '%';

SELECT CustomerId, FirstName, LastName
FROM dbo.Customer
WHERE LastName LIKE @Pattern;

-- Application builds pattern
-- C#: var pattern = userInput + "%";
```

### Date Ranges

```sql
-- ✅ Parameterize dates
DECLARE @StartDate DATE = @UserInputStart;
DECLARE @EndDate DATE = @UserInputEnd;

SELECT OrderId, OrderDate, Total
FROM dbo.Order
WHERE OrderDate >= @StartDate
  AND OrderDate < @EndDate;
```

## Performance Benefits

Query plan reuse demonstration:

```sql
-- Query 1: Parameterized
SELECT OrderId, Total FROM dbo.Order WHERE CustomerId = @CustomerId;
-- Query plan cached and reused

-- Query 2: Different parameter value, same plan
SELECT OrderId, Total FROM dbo.Order WHERE CustomerId = @CustomerId;  -- Reuses plan

-- Query 3: Concatenated (each is unique)
EXEC('SELECT OrderId, Total FROM dbo.Order WHERE CustomerId = 101');  -- New plan
EXEC('SELECT OrderId, Total FROM dbo.Order WHERE CustomerId = 102');  -- New plan
EXEC('SELECT OrderId, Total FROM dbo.Order WHERE CustomerId = 103');  -- New plan
-- Each creates separate plan, wastes plan cache
```

## Why String Concatenation Is a Problem

1. **SQL Injection**: Malicious input executes as SQL code
2. **Poor performance**: No plan reuse, plan cache pollution
3. **Maintenance**: String building is error-prone and hard to read
4. **Debugging**: Harder to log and debug dynamic SQL

## Symptoms

- String concatenation with `+` operator in SQL
- `EXEC()` or `EXEC sp_executesql` without parameters
- User input directly embedded in query strings
- Different query text for each value

## Benefits

- **Security**: Complete SQL injection protection
- **Performance**: Query plan reuse improves speed
- **Maintainability**: Clean, readable code
- **Type safety**: Database validates parameter types

## Trade-offs

- **Slight learning curve**: Must understand parameter syntax
- **Dynamic SQL complexity**: `sp_executesql` more verbose than simple EXEC
- **Parameter sniffing**: May need query hints in some cases

## See Also

- [SQL Injection Prevention](./sql-injection-prevention.md)
- [Explicit Transactions](./explicit-transactions.md)
- [Covering Indexes](./covering-indexes.md)

# Row-Level Security

> Enforce data access policies at the database level using SQL Server's RLS—ensure users only see authorized rows.

## Problem

When authorization is only enforced in application code, users can bypass restrictions by:
- Using admin tools to query directly
- Exploiting SQL injection vulnerabilities
- Using different applications that share the database
- Making mistakes in WHERE clauses that expose unauthorized data

## Example

### ❌ Before (Application-Level Filtering)

```sql
CREATE TABLE dbo.Order (
    OrderId INT NOT NULL,
    CustomerId INT NOT NULL,
    TenantId INT NOT NULL,  -- Multi-tenant system
    OrderDate DATETIME2(0) NOT NULL,
    Total DECIMAL(18,2) NOT NULL,
    
    CONSTRAINT PK_Order PRIMARY KEY (OrderId)
);

-- Application must remember to filter by TenantId
SELECT OrderId, OrderDate, Total
FROM dbo.Order
WHERE TenantId = @TenantId;  -- Easy to forget!

-- ❌ Oops, forgot to filter by TenantId
SELECT OrderId, OrderDate, Total
FROM dbo.Order
WHERE CustomerId = @CustomerId;  -- Leaks data from other tenants!

-- Application code is responsible for security
-- One mistake exposes data across tenants
```

### ✅ After (Row-Level Security)

```sql
-- 1. Create security predicate function
CREATE FUNCTION dbo.fn_SecurityPredicate_Order(@TenantId INT)
RETURNS TABLE
WITH SCHEMABINDING
AS
RETURN 
    SELECT 1 AS Result
    WHERE @TenantId = CAST(SESSION_CONTEXT(N'TenantId') AS INT);
GO

-- 2. Create security policy
CREATE SECURITY POLICY dbo.TenantFilter
ADD FILTER PREDICATE dbo.fn_SecurityPredicate_Order(TenantId) ON dbo.Order,
ADD BLOCK PREDICATE dbo.fn_SecurityPredicate_Order(TenantId) ON dbo.Order
    AFTER INSERT
WITH (STATE = ON);
GO

-- 3. Application sets context before querying
EXEC sp_set_session_context @key = N'TenantId', @value = 123;

-- Now ALL queries automatically filtered by TenantId
SELECT OrderId, OrderDate, Total
FROM dbo.Order;  -- Returns only orders for TenantId = 123

-- Even if you forget to filter, RLS protects you
SELECT OrderId, OrderDate, Total
FROM dbo.Order
WHERE CustomerId = @CustomerId;  -- Still filtered by TenantId automatically!

-- Trying to insert for wrong tenant fails
INSERT INTO dbo.Order (OrderId, CustomerId, TenantId, OrderDate, Total)
VALUES (1, 101, 999, GETDATE(), 99.99);  -- ❌ Blocked! TenantId 999 not allowed
```

## Security Policy Components

### Filter Predicate

Controls which rows are visible in SELECT, UPDATE, and DELETE:

```sql
-- Only see rows where TenantId matches session context
CREATE FUNCTION dbo.fn_TenantFilter(@TenantId INT)
RETURNS TABLE
WITH SCHEMABINDING
AS
RETURN 
    SELECT 1 AS Result
    WHERE @TenantId = CAST(SESSION_CONTEXT(N'TenantId') AS INT);

-- Apply to table
CREATE SECURITY POLICY dbo.TenantPolicy
ADD FILTER PREDICATE dbo.fn_TenantFilter(TenantId) ON dbo.Order;
```

### Block Predicate

Prevents INSERT, UPDATE, DELETE of unauthorized rows:

```sql
-- Prevent inserting/updating rows for other tenants
CREATE SECURITY POLICY dbo.TenantPolicy
ADD BLOCK PREDICATE dbo.fn_TenantFilter(TenantId) ON dbo.Order
    AFTER INSERT,
ADD BLOCK PREDICATE dbo.fn_TenantFilter(TenantId) ON dbo.Order
    AFTER UPDATE;

-- Trying to insert for wrong tenant fails
EXEC sp_set_session_context @key = N'TenantId', @value = 123;

INSERT INTO dbo.Order (OrderId, CustomerId, TenantId, OrderDate, Total)
VALUES (1, 101, 999, GETDATE(), 99.99);
-- Error: The attempted operation failed because the target object has a block predicate
```

### BEFORE vs AFTER Predicates

```sql
-- AFTER: Check value after operation (INSERT/UPDATE sets TenantId)
ADD BLOCK PREDICATE dbo.fn_TenantFilter(TenantId) ON dbo.Order
    AFTER INSERT;

-- BEFORE: Check value before operation (prevents seeing unauthorized rows during UPDATE)
ADD BLOCK PREDICATE dbo.fn_TenantFilter(TenantId) ON dbo.Order
    BEFORE UPDATE;
```

## Multi-Tenant Example

Complete multi-tenant setup with RLS:

```sql
-- Tenant isolation for all tables
CREATE FUNCTION dbo.fn_TenantAccess(@TenantId INT)
RETURNS TABLE
WITH SCHEMABINDING
AS
RETURN 
    SELECT 1 AS Result
    WHERE @TenantId = CAST(SESSION_CONTEXT(N'TenantId') AS INT)
       OR CAST(SESSION_CONTEXT(N'IsAdmin') AS BIT) = 1;  -- Admins see all
GO

-- Apply to multiple tables
CREATE SECURITY POLICY dbo.TenantSecurity
-- Customers
ADD FILTER PREDICATE dbo.fn_TenantAccess(TenantId) ON dbo.Customer,
ADD BLOCK PREDICATE dbo.fn_TenantAccess(TenantId) ON dbo.Customer AFTER INSERT,
-- Orders  
ADD FILTER PREDICATE dbo.fn_TenantAccess(TenantId) ON dbo.Order,
ADD BLOCK PREDICATE dbo.fn_TenantAccess(TenantId) ON dbo.Order AFTER INSERT,
-- Products
ADD FILTER PREDICATE dbo.fn_TenantAccess(TenantId) ON dbo.Product,
ADD BLOCK PREDICATE dbo.fn_TenantAccess(TenantId) ON dbo.Product AFTER INSERT
WITH (STATE = ON);
GO

-- Application sets tenant context from auth token
CREATE PROCEDURE dbo.usp_SetUserContext
    @UserId INT,
    @TenantId INT,
    @IsAdmin BIT
AS
BEGIN
    EXEC sp_set_session_context @key = N'UserId', @value = @UserId;
    EXEC sp_set_session_context @key = N'TenantId', @value = @TenantId;
    EXEC sp_set_session_context @key = N'IsAdmin', @value = @IsAdmin;
END;
GO

-- Regular user
EXEC dbo.usp_SetUserContext @UserId = 1, @TenantId = 123, @IsAdmin = 0;
SELECT * FROM dbo.Customer;  -- Only TenantId 123

-- Admin
EXEC dbo.usp_SetUserContext @UserId = 999, @TenantId = NULL, @IsAdmin = 1;
SELECT * FROM dbo.Customer;  -- All tenants
```

## User-Level Access Control

Restrict users to their own data:

```sql
CREATE TABLE dbo.Document (
    DocumentId INT NOT NULL,
    UserId INT NOT NULL,
    Title NVARCHAR(200) NOT NULL,
    Content NVARCHAR(MAX) NOT NULL,
    
    CONSTRAINT PK_Document PRIMARY KEY (DocumentId)
);

-- Users can only see their own documents
CREATE FUNCTION dbo.fn_DocumentAccess(@UserId INT)
RETURNS TABLE
WITH SCHEMABINDING
AS
RETURN 
    SELECT 1 AS Result
    WHERE @UserId = CAST(SESSION_CONTEXT(N'UserId') AS INT);
GO

CREATE SECURITY POLICY dbo.DocumentSecurity
ADD FILTER PREDICATE dbo.fn_DocumentAccess(UserId) ON dbo.Document,
ADD BLOCK PREDICATE dbo.fn_DocumentAccess(UserId) ON dbo.Document AFTER INSERT
WITH (STATE = ON);
GO

-- Set user context
EXEC sp_set_session_context @key = N'UserId', @value = 42;

-- User only sees their documents
SELECT * FROM dbo.Document;  -- Only UserId = 42

-- Cannot insert documents for other users
INSERT INTO dbo.Document (DocumentId, UserId, Title, Content)
VALUES (1, 999, 'Hack', 'Evil');  -- ❌ Blocked!
```

## Performance Considerations

### Indexed Columns

Ensure filtered columns are indexed:

```sql
-- Create index on TenantId for RLS performance
CREATE NONCLUSTERED INDEX IX_Order_TenantId
ON dbo.Order(TenantId)
INCLUDE (OrderDate, Total);

-- Execution plan uses index seek, not table scan
```

### Avoid Complex Predicates

Keep predicate functions simple for best performance:

```sql
-- ✅ Good: Simple equality check
CREATE FUNCTION dbo.fn_SimplePredicate(@TenantId INT)
RETURNS TABLE
WITH SCHEMABINDING
AS
RETURN 
    SELECT 1 AS Result
    WHERE @TenantId = CAST(SESSION_CONTEXT(N'TenantId') AS INT);

-- ❌ Avoid: Complex joins or subqueries
CREATE FUNCTION dbo.fn_ComplexPredicate(@TenantId INT)
RETURNS TABLE
WITH SCHEMABINDING
AS
RETURN 
    SELECT 1 AS Result
    WHERE @TenantId IN (
        SELECT TenantId FROM dbo.UserTenants  -- Slow!
        WHERE UserId = CAST(SESSION_CONTEXT(N'UserId') AS INT)
    );
```

## Bypassing RLS (Admin Queries)

Use `WITH (RLS_BYPASS)` for admin operations:

```sql
-- Admins can bypass RLS for maintenance
SELECT * FROM dbo.Order
WITH (RLS_BYPASS);

-- Requires ALTER ANY SECURITY POLICY permission
-- Don't grant to application users!
```

## Temporarily Disabling RLS

Disable for bulk operations or maintenance:

```sql
-- Disable policy
ALTER SECURITY POLICY dbo.TenantSecurity
WITH (STATE = OFF);

-- Perform maintenance
-- ...

-- Re-enable policy
ALTER SECURITY POLICY dbo.TenantSecurity
WITH (STATE = ON);
```

## Debugging RLS

Check which policies are active:

```sql
-- View security policies
SELECT 
    OBJECT_NAME(object_id) AS TableName,
    name AS PolicyName,
    is_enabled AS IsEnabled,
    is_schema_bound AS IsSchemabound
FROM sys.security_policies;

-- View predicates
SELECT 
    OBJECT_NAME(sp.object_id) AS PolicyName,
    OBJECT_NAME(spp.target_object_id) AS TableName,
    OBJECT_NAME(spp.predicate_id) AS PredicateFunction,
    spp.predicate_type_desc AS PredicateType,
    spp.operation_desc AS Operation
FROM sys.security_policies sp
INNER JOIN sys.security_predicates spp ON sp.object_id = spp.object_id;

-- Check session context
SELECT 
    SESSION_CONTEXT(N'TenantId') AS TenantId,
    SESSION_CONTEXT(N'UserId') AS UserId,
    SESSION_CONTEXT(N'IsAdmin') AS IsAdmin;
```

## Why Application-Level Filtering Is a Problem

1. **Easy to forget**: Every query must remember to filter
2. **SQL injection**: Bypasses application security
3. **Multiple applications**: Hard to ensure consistency
4. **Admin tools**: Direct database access bypasses application
5. **One mistake**: Single missing WHERE clause exposes data

## Symptoms

- Comments in code saying "don't forget to filter by TenantId"
- Security bugs from missing WHERE clauses
- Duplicate filtering logic across applications
- Inability to use admin tools safely

## Benefits

- **Security by default**: All queries automatically filtered
- **Defense in depth**: Works even if application makes mistakes
- **Consistent enforcement**: Same rules for all database access
- **Simpler code**: Application doesn't need authorization logic in every query

## Trade-offs

- **Performance overhead**: Predicate function evaluated on every query (minimal with proper indexing)
- **Complexity**: Requires understanding of session context and predicates
- **Debugging**: Harder to see what RLS is doing without proper monitoring

## See Also

- [SQL Injection Prevention](./sql-injection-prevention.md)
- [Least Privilege](./least-privilege.md)
- [Always Encrypted](./always-encrypted.md)

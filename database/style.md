# T-SQL Coding Style

Conventions for writing clear, maintainable T-SQL code in SQL Server.

---

## General Principles

- **Uppercase keywords**: Use uppercase for SQL keywords (`SELECT`, `FROM`, `WHERE`, `JOIN`, `CREATE`, `TABLE`, etc.)
- **Mixed case identifiers**: Use PascalCase for tables, columns, and database objects (`OrderId`, `CustomerId`, `CreatedAt`)
- **Schema-qualify all objects**: Always prefix table and view names with schema (`dbo.Orders`, not just `Orders`)
- **No `SELECT *`**: Explicitly list columns in production code
- **Explicit column lists in INSERT**: Always specify columns, even when inserting all of them
- **Named constraints**: Give constraints meaningful names, don't rely on auto-generated names
- **Indent for readability**: Align columns, indent nested queries
- **One statement per line**: Don't cram multiple statements on a single line

---

## Naming Conventions

### Tables

- **Singular nouns**: `Order`, not `Orders`
- **PascalCase**: `CustomerOrder`, `OrderLineItem`
- **No prefixes**: Avoid `tbl_`, `t_`, or Hungarian notation

```sql
-- ✅ Good
CREATE TABLE dbo.Order ( ... );
CREATE TABLE dbo.Customer ( ... );
CREATE TABLE dbo.OrderLineItem ( ... );

-- ❌ Bad
CREATE TABLE dbo.Orders ( ... );
CREATE TABLE dbo.tbl_customer ( ... );
CREATE TABLE dbo.tblOrderLineItems ( ... );
```

### Columns

- **PascalCase**: `OrderId`, `FirstName`, `CreatedAt`
- **Descriptive names**: Prefer `CreatedAt` over `Created` or `Date`
- **Suffix with type for clarity**: `DeletedAt` (datetime), `IsActive` (bit/boolean)
- **Avoid abbreviations**: Use `CustomerId` not `CustId`

```sql
-- ✅ Good
OrderId INT,
CustomerId INT,
FirstName NVARCHAR(100),
CreatedAt DATETIME2(0),
IsActive BIT

-- ❌ Bad
order_id INT,
cust_id INT,
fname NVARCHAR(100),
created DATETIME,
active BIT
```

### Constraints

- **Primary Keys**: `PK_TableName` (e.g., `PK_Order`)
- **Foreign Keys**: `FK_ChildTable_ParentTable` (e.g., `FK_Order_Customer`)
- **Unique Constraints**: `UQ_TableName_ColumnName` (e.g., `UQ_Customer_Email`)
- **Check Constraints**: `CK_TableName_ColumnName` (e.g., `CK_Order_Total`)
- **Default Constraints**: `DF_TableName_ColumnName` (e.g., `DF_Order_Status`)

```sql
CREATE TABLE dbo.Order (
    OrderId INT NOT NULL,
    CustomerId INT NOT NULL,
    Total DECIMAL(18,2) NOT NULL,
    Status VARCHAR(20) NOT NULL,
    
    CONSTRAINT PK_Order PRIMARY KEY (OrderId),
    CONSTRAINT FK_Order_Customer FOREIGN KEY (CustomerId) 
        REFERENCES dbo.Customer(CustomerId),
    CONSTRAINT CK_Order_Total CHECK (Total >= 0),
    CONSTRAINT DF_Order_Status DEFAULT 'Pending' FOR Status
);
```

### Indexes

- **Clustered**: `PK_TableName` (typically the primary key)
- **Non-clustered**: `IX_TableName_ColumnName` (e.g., `IX_Order_CustomerId`)
- **Covering**: `IX_TableName_KeyColumns_IncludeColumns` (e.g., `IX_Order_CustomerId_CreatedAt`)

```sql
CREATE NONCLUSTERED INDEX IX_Order_CustomerId
ON dbo.Order(CustomerId);

CREATE NONCLUSTERED INDEX IX_Order_CustomerId_CreatedAt
ON dbo.Order(CustomerId, CreatedAt)
INCLUDE (Total, Status);
```

### Stored Procedures

- **Prefix with `usp_`**: User stored procedure (e.g., `usp_GetOrdersByCustomer`)
- **Verb-noun pattern**: Action + object (e.g., `usp_CreateOrder`, `usp_UpdateCustomerEmail`)
- **No `sp_` prefix**: Reserved for system procedures

```sql
-- ✅ Good
CREATE PROCEDURE dbo.usp_GetOrdersByCustomer
CREATE PROCEDURE dbo.usp_CreateOrder
CREATE PROCEDURE dbo.usp_UpdateCustomerEmail

-- ❌ Bad
CREATE PROCEDURE GetOrders          -- Missing prefix
CREATE PROCEDURE sp_GetOrders       -- sp_ reserved for system
CREATE PROCEDURE dbo.proc_get_orders -- Inconsistent naming
```

### Functions

- **Scalar functions**: Prefix with `fn_` (e.g., `fn_CalculateOrderTotal`)
- **Table-valued functions**: Prefix with `tvf_` (e.g., `tvf_GetOrderDetails`)
- **Inline table-valued preferred**: More performant than multi-statement

```sql
-- Scalar function
CREATE FUNCTION dbo.fn_CalculateDiscount(@Total DECIMAL(18,2))
RETURNS DECIMAL(18,2)
AS
BEGIN
    RETURN @Total * 0.1;
END;

-- Inline table-valued function (preferred)
CREATE FUNCTION dbo.tvf_GetOrdersByCustomer(@CustomerId INT)
RETURNS TABLE
AS
RETURN
(
    SELECT OrderId, OrderDate, Total
    FROM dbo.Order
    WHERE CustomerId = @CustomerId
);
```

### Views

- **Prefix with `v_`**: (e.g., `v_CustomerOrders`)
- **Or no prefix**: Some teams prefer no prefix for views to make them look like tables

```sql
CREATE VIEW dbo.v_CustomerOrderSummary
AS
SELECT 
    c.CustomerId,
    c.FirstName,
    c.LastName,
    COUNT(o.OrderId) AS OrderCount,
    SUM(o.Total) AS TotalSpent
FROM dbo.Customer c
LEFT JOIN dbo.Order o ON c.CustomerId = o.CustomerId
GROUP BY c.CustomerId, c.FirstName, c.LastName;
```

---

## Formatting

### SELECT Statements

- Each column on its own line
- Align columns vertically
- Schema-qualify table names
- Use table aliases for clarity
- Indent JOINs and WHERE clauses

```sql
-- ✅ Good
SELECT 
    o.OrderId,
    o.OrderDate,
    o.Total,
    c.FirstName,
    c.LastName
FROM dbo.Order o
INNER JOIN dbo.Customer c ON o.CustomerId = c.CustomerId
WHERE o.OrderDate >= '2024-01-01'
  AND o.Status = 'Shipped'
ORDER BY o.OrderDate DESC;

-- ❌ Bad
select * from Order o join Customer c on o.CustomerId=c.CustomerId where o.OrderDate>='2024-01-01';
```

### INSERT Statements

- Always specify column list
- Values on separate lines for readability (if multiple rows)
- Align VALUES vertically

```sql
-- ✅ Good
INSERT INTO dbo.Order (OrderId, CustomerId, OrderDate, Total, Status)
VALUES (1, 101, '2024-01-15', 99.99, 'Pending');

-- Multiple rows
INSERT INTO dbo.Order (OrderId, CustomerId, OrderDate, Total, Status)
VALUES 
    (1, 101, '2024-01-15', 99.99, 'Pending'),
    (2, 102, '2024-01-16', 149.99, 'Shipped'),
    (3, 103, '2024-01-17', 199.99, 'Delivered');

-- ❌ Bad
INSERT INTO dbo.Order VALUES (1, 101, '2024-01-15', 99.99, 'Pending');
```

### UPDATE Statements

- WHERE clause on its own line
- Multiple SET assignments aligned vertically

```sql
-- ✅ Good
UPDATE dbo.Order
SET 
    Status = 'Shipped',
    ShippedAt = SYSDATETIME(),
    ShippedBy = @UserId
WHERE OrderId = @OrderId
  AND Status = 'Pending';

-- ❌ Bad
UPDATE dbo.Order SET Status='Shipped',ShippedAt=SYSDATETIME() WHERE OrderId=@OrderId;
```

### DELETE Statements

- Always include WHERE clause (or document why it's intentionally omitted)
- Consider soft deletes over hard deletes

```sql
-- ✅ Good
DELETE FROM dbo.OrderLineItem
WHERE OrderId = @OrderId;

-- Better: Soft delete
UPDATE dbo.Order
SET 
    DeletedAt = SYSDATETIME(),
    DeletedBy = @UserId
WHERE OrderId = @OrderId
  AND DeletedAt IS NULL;

-- ❌ Dangerous
DELETE FROM dbo.Order;  -- No WHERE clause!
```

---

## Best Practices

### Avoid SELECT *

Always specify columns explicitly. This improves performance, clarity, and prevents breaking changes.

```sql
-- ❌ Bad
SELECT * FROM dbo.Order;

-- ✅ Good
SELECT 
    OrderId,
    CustomerId,
    OrderDate,
    Total,
    Status
FROM dbo.Order;
```

### Use EXISTS Instead of COUNT

When checking for existence, use `EXISTS` instead of `COUNT(*)`.

```sql
-- ❌ Less efficient
IF (SELECT COUNT(*) FROM dbo.Order WHERE CustomerId = @CustomerId) > 0
BEGIN
    ...
END

-- ✅ More efficient
IF EXISTS (SELECT 1 FROM dbo.Order WHERE CustomerId = @CustomerId)
BEGIN
    ...
END
```

### Use SET NOCOUNT ON

In stored procedures and triggers, use `SET NOCOUNT ON` to prevent "rows affected" messages.

```sql
CREATE PROCEDURE dbo.usp_GetOrdersByCustomer
    @CustomerId INT
AS
BEGIN
    SET NOCOUNT ON;
    
    SELECT 
        OrderId,
        OrderDate,
        Total,
        Status
    FROM dbo.Order
    WHERE CustomerId = @CustomerId;
END;
```

### Use TRY...CATCH

Wrap transaction logic in error handling blocks.

```sql
BEGIN TRY
    BEGIN TRANSACTION;
    
    UPDATE dbo.Order
    SET Status = 'Shipped'
    WHERE OrderId = @OrderId;
    
    INSERT INTO dbo.OrderHistory (OrderId, Action, CreatedAt)
    VALUES (@OrderId, 'Shipped', SYSDATETIME());
    
    COMMIT TRANSACTION;
END TRY
BEGIN CATCH
    IF @@TRANCOUNT > 0
        ROLLBACK TRANSACTION;
    
    THROW;
END CATCH;
```

### Parameterize Queries

Always use parameters to prevent SQL injection.

```sql
-- ❌ SQL Injection vulnerability
DECLARE @SQL NVARCHAR(MAX) = 'SELECT * FROM dbo.Customer WHERE Email = ''' + @Email + '''';
EXEC sp_executesql @SQL;

-- ✅ Safe parameterized query
SELECT 
    CustomerId,
    FirstName,
    LastName,
    Email
FROM dbo.Customer
WHERE Email = @Email;
```

### Use Meaningful Aliases

Table aliases should be short but meaningful.

```sql
-- ✅ Good
SELECT 
    o.OrderId,
    c.FirstName,
    p.ProductName
FROM dbo.Order o
INNER JOIN dbo.Customer c ON o.CustomerId = c.CustomerId
INNER JOIN dbo.OrderLineItem oli ON o.OrderId = oli.OrderId
INNER JOIN dbo.Product p ON oli.ProductId = p.ProductId;

-- ❌ Bad
SELECT 
    t1.OrderId,
    t2.FirstName,
    t3.ProductName
FROM dbo.Order t1
INNER JOIN dbo.Customer t2 ON t1.CustomerId = t2.CustomerId
INNER JOIN dbo.OrderLineItem t4 ON t1.OrderId = t4.OrderId
INNER JOIN dbo.Product t3 ON t4.ProductId = t3.ProductId;
```

---

## Comments

- Use `--` for single-line comments
- Use `/* ... */` for multi-line comments
- Document complex business logic
- Don't comment obvious code

```sql
-- ✅ Good: Comments explain why, not what
-- Calculate discount based on customer loyalty tier
-- Gold customers get 15%, Silver 10%, Bronze 5%
SELECT 
    o.OrderId,
    o.Total,
    CASE c.LoyaltyTier
        WHEN 'Gold' THEN o.Total * 0.15
        WHEN 'Silver' THEN o.Total * 0.10
        WHEN 'Bronze' THEN o.Total * 0.05
        ELSE 0
    END AS Discount
FROM dbo.Order o
INNER JOIN dbo.Customer c ON o.CustomerId = c.CustomerId;

-- ❌ Bad: Comments state the obvious
-- Select OrderId from Order table
SELECT OrderId FROM dbo.Order;
```

---

## Spacing and Indentation

- Use 4 spaces for indentation (not tabs)
- Separate logical blocks with blank lines
- Align related elements vertically

```sql
CREATE PROCEDURE dbo.usp_ProcessOrder
    @OrderId INT,
    @UserId INT
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @Status VARCHAR(20);
    DECLARE @Total DECIMAL(18,2);
    
    -- Get current order status
    SELECT 
        @Status = Status,
        @Total = Total
    FROM dbo.Order
    WHERE OrderId = @OrderId;
    
    -- Validate order can be processed
    IF @Status <> 'Pending'
    BEGIN
        THROW 50001, 'Order is not in pending status', 1;
    END;
    
    -- Update order status
    BEGIN TRY
        BEGIN TRANSACTION;
        
        UPDATE dbo.Order
        SET 
            Status = 'Processing',
            ProcessedAt = SYSDATETIME(),
            ProcessedBy = @UserId
        WHERE OrderId = @OrderId;
        
        -- Log the action
        INSERT INTO dbo.OrderHistory (OrderId, Action, CreatedBy, CreatedAt)
        VALUES (@OrderId, 'Processing', @UserId, SYSDATETIME());
        
        COMMIT TRANSACTION;
    END TRY
    BEGIN CATCH
        IF @@TRANCOUNT > 0
            ROLLBACK TRANSACTION;
        
        THROW;
    END CATCH;
END;
```

---

## Summary

- Use **uppercase keywords**, **PascalCase identifiers**, **schema qualification**
- **Name constraints explicitly** with meaningful prefixes
- **Format for readability**: one column per line, align vertically, indent nested logic
- **Always use parameters** to prevent SQL injection
- **Specify column lists** in SELECT and INSERT statements
- **Include error handling** with TRY...CATCH blocks
- **Comment business logic**, not obvious code

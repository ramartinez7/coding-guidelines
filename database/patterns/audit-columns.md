# Audit Columns

> Include standard audit columns in every table—track who created/modified records and when.

## Problem

Without audit trails, it's impossible to answer questions like "Who changed this?", "When was this created?", and "What was the previous value?"—critical for debugging, compliance, and understanding system behavior.

## Example

### ❌ Before (No Audit Trail)

```sql
CREATE TABLE dbo.Customer (
    CustomerId INT NOT NULL,
    FirstName NVARCHAR(100) NOT NULL,
    LastName NVARCHAR(100) NOT NULL,
    Email NVARCHAR(255) NOT NULL,
    
    CONSTRAINT PK_Customer PRIMARY KEY (CustomerId)
);

-- Questions you cannot answer:
-- - When was this customer created?
-- - Who created this customer?
-- - When was the email last updated?
-- - Who made the last change?
```

### ✅ After (Standard Audit Columns)

```sql
CREATE TABLE dbo.Customer (
    CustomerId INT NOT NULL,
    FirstName NVARCHAR(100) NOT NULL,
    LastName NVARCHAR(100) NOT NULL,
    Email NVARCHAR(255) NOT NULL,
    
    -- Standard audit columns
    CreatedAt DATETIME2(0) NOT NULL DEFAULT SYSDATETIME(),
    CreatedBy INT NOT NULL,
    ModifiedAt DATETIME2(0) NOT NULL DEFAULT SYSDATETIME(),
    ModifiedBy INT NOT NULL,
    
    CONSTRAINT PK_Customer PRIMARY KEY (CustomerId),
    CONSTRAINT FK_Customer_CreatedBy FOREIGN KEY (CreatedBy)
        REFERENCES dbo.[User](UserId),
    CONSTRAINT FK_Customer_ModifiedBy FOREIGN KEY (ModifiedBy)
        REFERENCES dbo.[User](UserId)
);

-- Insert with audit info
INSERT INTO dbo.Customer (CustomerId, FirstName, LastName, Email, CreatedBy, ModifiedBy)
VALUES (1, 'Alice', 'Smith', 'alice@example.com', @CurrentUserId, @CurrentUserId);
-- CreatedAt and ModifiedAt automatically set to current time

-- Update with audit info
UPDATE dbo.Customer
SET 
    Email = 'newemail@example.com',
    ModifiedAt = SYSDATETIME(),
    ModifiedBy = @CurrentUserId
WHERE CustomerId = 1;

-- Query: Who created customers in the last 30 days?
SELECT 
    c.CustomerId,
    c.FirstName,
    c.LastName,
    c.CreatedAt,
    u.Username AS CreatedByUser
FROM dbo.Customer c
INNER JOIN dbo.[User] u ON c.CreatedBy = u.UserId
WHERE c.CreatedAt >= DATEADD(DAY, -30, GETDATE())
ORDER BY c.CreatedAt DESC;
```

## Standard Audit Columns

### Minimal Set

Every table should have at minimum:

```sql
CreatedAt DATETIME2(0) NOT NULL DEFAULT SYSDATETIME()
CreatedBy INT NOT NULL
ModifiedAt DATETIME2(0) NOT NULL DEFAULT SYSDATETIME()
ModifiedBy INT NOT NULL
```

### Extended Set

For more comprehensive auditing:

```sql
CreatedAt DATETIME2(0) NOT NULL DEFAULT SYSDATETIME()
CreatedBy INT NOT NULL
ModifiedAt DATETIME2(0) NOT NULL DEFAULT SYSDATETIME()
ModifiedBy INT NOT NULL
DeletedAt DATETIME2(0) NULL          -- For soft deletes
DeletedBy INT NULL                    -- For soft deletes
Version INT NOT NULL DEFAULT 1        -- For optimistic concurrency
```

## Automatic Update Triggers

Use triggers to automatically maintain ModifiedAt:

```sql
CREATE TRIGGER TR_Customer_UpdateModified
ON dbo.Customer
AFTER UPDATE
AS
BEGIN
    SET NOCOUNT ON;
    
    UPDATE c
    SET c.ModifiedAt = SYSDATETIME()
    FROM dbo.Customer c
    INNER JOIN inserted i ON c.CustomerId = i.CustomerId
    WHERE c.ModifiedAt = i.ModifiedAt;  -- Only update if not explicitly set
END;
```

## Stored Procedure Pattern

Encapsulate audit column handling in procedures:

```sql
CREATE PROCEDURE dbo.usp_CreateCustomer
    @FirstName NVARCHAR(100),
    @LastName NVARCHAR(100),
    @Email NVARCHAR(255),
    @UserId INT
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @Now DATETIME2(0) = SYSDATETIME();
    
    INSERT INTO dbo.Customer (
        CustomerId,
        FirstName,
        LastName,
        Email,
        CreatedAt,
        CreatedBy,
        ModifiedAt,
        ModifiedBy
    )
    VALUES (
        NEXT VALUE FOR dbo.CustomerIdSequence,
        @FirstName,
        @LastName,
        @Email,
        @Now,
        @UserId,
        @Now,
        @UserId
    );
    
    SELECT 
        CustomerId,
        FirstName,
        LastName,
        Email,
        CreatedAt,
        CreatedBy
    FROM dbo.Customer
    WHERE CustomerId = SCOPE_IDENTITY();
END;

CREATE PROCEDURE dbo.usp_UpdateCustomerEmail
    @CustomerId INT,
    @Email NVARCHAR(255),
    @UserId INT
AS
BEGIN
    SET NOCOUNT ON;
    
    UPDATE dbo.Customer
    SET 
        Email = @Email,
        ModifiedAt = SYSDATETIME(),
        ModifiedBy = @UserId
    WHERE CustomerId = @CustomerId;
    
    IF @@ROWCOUNT = 0
        THROW 50001, 'Customer not found', 1;
    
    SELECT 
        CustomerId,
        FirstName,
        LastName,
        Email,
        ModifiedAt,
        ModifiedBy
    FROM dbo.Customer
    WHERE CustomerId = @CustomerId;
END;
```

## Querying Audit Information

### Recent Changes

```sql
-- Find records modified in last 24 hours
SELECT 
    c.CustomerId,
    c.FirstName,
    c.LastName,
    c.ModifiedAt,
    u.Username AS ModifiedByUser
FROM dbo.Customer c
INNER JOIN dbo.[User] u ON c.ModifiedBy = u.UserId
WHERE c.ModifiedAt >= DATEADD(HOUR, -24, GETDATE())
ORDER BY c.ModifiedAt DESC;
```

### Activity by User

```sql
-- What did this user create/modify?
DECLARE @UserId INT = 123;

-- Created
SELECT 
    'Customer' AS EntityType,
    CustomerId AS EntityId,
    FirstName + ' ' + LastName AS Description,
    CreatedAt AS ActionDate
FROM dbo.Customer
WHERE CreatedBy = @UserId

UNION ALL

-- Modified
SELECT 
    'Customer' AS EntityType,
    CustomerId AS EntityId,
    FirstName + ' ' + LastName AS Description,
    ModifiedAt AS ActionDate
FROM dbo.Customer
WHERE ModifiedBy = @UserId
  AND ModifiedAt <> CreatedAt  -- Exclude initial creation

ORDER BY ActionDate DESC;
```

### Audit Report

```sql
-- Activity summary for date range
SELECT 
    u.Username,
    COUNT(DISTINCT c.CustomerId) AS CustomersCreated,
    COUNT(DISTINCT o.OrderId) AS OrdersCreated,
    MIN(c.CreatedAt) AS FirstAction,
    MAX(o.CreatedAt) AS LastAction
FROM dbo.[User] u
LEFT JOIN dbo.Customer c ON u.UserId = c.CreatedBy 
    AND c.CreatedAt >= @StartDate 
    AND c.CreatedAt < @EndDate
LEFT JOIN dbo.Order o ON u.UserId = o.CreatedBy 
    AND o.CreatedAt >= @StartDate 
    AND o.CreatedAt < @EndDate
GROUP BY u.Username
HAVING COUNT(DISTINCT c.CustomerId) > 0 OR COUNT(DISTINCT o.OrderId) > 0
ORDER BY CustomersCreated + OrdersCreated DESC;
```

## Time Zone Considerations

Always use UTC for audit timestamps:

```sql
-- ✅ Good: Use SYSDATETIME() or SYSUTCDATETIME() for UTC
CreatedAt DATETIME2(0) NOT NULL DEFAULT SYSUTCDATETIME()

-- ❌ Bad: GETDATE() uses server's local time
CreatedAt DATETIME NOT NULL DEFAULT GETDATE()

-- Query with time zone conversion
SELECT 
    CustomerId,
    FirstName,
    LastName,
    CreatedAt AT TIME ZONE 'UTC' AT TIME ZONE 'Pacific Standard Time' AS CreatedAtPST
FROM dbo.Customer;
```

## Precision Considerations

Use appropriate precision for timestamps:

```sql
-- ✅ Good: DATETIME2(0) for second precision (7 bytes)
CreatedAt DATETIME2(0) NOT NULL

-- ✅ Good: DATETIME2(3) for millisecond precision (7-8 bytes)
CreatedAt DATETIME2(3) NOT NULL

-- ❌ Overkill: DATETIME2(7) for nanosecond precision (8 bytes)
-- Most applications don't need nanosecond precision
CreatedAt DATETIME2(7) NOT NULL

-- ❌ Old: DATETIME for 3.33ms precision (8 bytes)
-- Use DATETIME2 instead in new designs
CreatedAt DATETIME NOT NULL
```

## Combining with Soft Deletes

```sql
CREATE TABLE dbo.Order (
    OrderId INT NOT NULL,
    CustomerId INT NOT NULL,
    Total DECIMAL(18,2) NOT NULL,
    Status VARCHAR(20) NOT NULL,
    
    -- Creation audit
    CreatedAt DATETIME2(0) NOT NULL DEFAULT SYSUTCDATETIME(),
    CreatedBy INT NOT NULL,
    
    -- Modification audit
    ModifiedAt DATETIME2(0) NOT NULL DEFAULT SYSUTCDATETIME(),
    ModifiedBy INT NOT NULL,
    
    -- Deletion audit (soft delete)
    DeletedAt DATETIME2(0) NULL,
    DeletedBy INT NULL,
    
    CONSTRAINT PK_Order PRIMARY KEY (OrderId)
);

-- Soft delete with audit
UPDATE dbo.Order
SET 
    DeletedAt = SYSDATETIME(),
    DeletedBy = @UserId,
    ModifiedAt = SYSDATETIME(),
    ModifiedBy = @UserId
WHERE OrderId = @OrderId
  AND DeletedAt IS NULL;

-- Query active orders
SELECT * FROM dbo.Order WHERE DeletedAt IS NULL;

-- Audit: Who deleted what?
SELECT 
    o.OrderId,
    o.Total,
    o.DeletedAt,
    u.Username AS DeletedByUser
FROM dbo.Order o
INNER JOIN dbo.[User] u ON o.DeletedBy = u.UserId
WHERE o.DeletedAt IS NOT NULL
ORDER BY o.DeletedAt DESC;
```

## Application-Level vs Database-Level

### Database Defaults

```sql
-- Database sets timestamps automatically
CreatedAt DATETIME2(0) NOT NULL DEFAULT SYSUTCDATETIME()
ModifiedAt DATETIME2(0) NOT NULL DEFAULT SYSUTCDATETIME()

-- Application must provide user ID
CreatedBy INT NOT NULL  -- No default
ModifiedBy INT NOT NULL  -- No default
```

### Application Control

```sql
-- Application sets everything explicitly
CreatedAt DATETIME2(0) NOT NULL  -- No default
CreatedBy INT NOT NULL
ModifiedAt DATETIME2(0) NOT NULL
ModifiedBy INT NOT NULL

-- Better control, but requires discipline
```

## Why Missing Audit Columns Is a Problem

1. **No accountability**: Cannot determine who made changes
2. **Debugging nightmare**: Cannot trace when issues were introduced
3. **Compliance failures**: Regulations often require audit trails
4. **Data forensics**: Cannot investigate incidents or data corruption
5. **Performance analysis**: Cannot identify patterns in data creation/modification

## Symptoms

- Tables without CreatedAt or ModifiedAt columns
- "When was this record created?" questions cannot be answered
- Cannot track user activity
- Compliance audit findings
- No way to debug data issues

## Benefits

- **Accountability**: Always know who did what
- **Debugging**: Trace changes to specific times and users
- **Compliance**: Meet audit requirements automatically
- **Analytics**: Understand usage patterns and user behavior
- **Data quality**: Identify and fix problematic patterns

## Trade-offs

- **Storage overhead**: 4 extra columns per table (minimal)
- **Insert overhead**: Must provide user ID (small)
- **Application changes**: Must pass user ID to all operations
- **Privacy concerns**: User tracking must comply with regulations

## See Also

- [Soft Deletes](./soft-deletes.md)
- [Temporal Tables](./temporal-tables.md)
- [Optimistic Concurrency](./optimistic-concurrency.md)
- [Event Sourcing](./event-sourcing.md)

# Soft Deletes

> Mark records as deleted instead of removing them—preserve audit trails and enable recovery.

## Problem

Hard deletes (`DELETE FROM ...`) permanently destroy data, making it impossible to:
- Audit who deleted what and when
- Recover accidentally deleted records
- Understand historical state
- Comply with data retention requirements

## Example

### ❌ Before (Hard Delete)

```sql
CREATE TABLE dbo.Customer (
    CustomerId INT NOT NULL,
    FirstName NVARCHAR(100) NOT NULL,
    LastName NVARCHAR(100) NOT NULL,
    Email NVARCHAR(255) NOT NULL,
    
    CONSTRAINT PK_Customer PRIMARY KEY (CustomerId)
);

-- Delete destroys the record permanently
DELETE FROM dbo.Customer WHERE CustomerId = 123;

-- No way to:
-- - See who deleted it
-- - Know when it was deleted  
-- - Recover the data if it was a mistake
-- - Query historical state
```

### ✅ After (Soft Delete)

```sql
CREATE TABLE dbo.Customer (
    CustomerId INT NOT NULL,
    FirstName NVARCHAR(100) NOT NULL,
    LastName NVARCHAR(100) NOT NULL,
    Email NVARCHAR(255) NOT NULL,
    CreatedAt DATETIME2(0) NOT NULL DEFAULT SYSDATETIME(),
    CreatedBy INT NOT NULL,
    DeletedAt DATETIME2(0) NULL,
    DeletedBy INT NULL,
    
    CONSTRAINT PK_Customer PRIMARY KEY (CustomerId)
);

-- "Delete" by marking the record
UPDATE dbo.Customer
SET 
    DeletedAt = SYSDATETIME(),
    DeletedBy = @UserId
WHERE CustomerId = @CustomerId
  AND DeletedAt IS NULL;  -- Prevent double-deletion

-- Query active customers (most common case)
SELECT 
    CustomerId,
    FirstName,
    LastName,
    Email
FROM dbo.Customer
WHERE DeletedAt IS NULL;

-- Query deleted customers (for audit)
SELECT 
    CustomerId,
    FirstName,
    LastName,
    Email,
    DeletedAt,
    DeletedBy
FROM dbo.Customer
WHERE DeletedAt IS NOT NULL;

-- Recover a deleted customer
UPDATE dbo.Customer
SET 
    DeletedAt = NULL,
    DeletedBy = NULL
WHERE CustomerId = @CustomerId
  AND DeletedAt IS NOT NULL;
```

## Standard Soft Delete Columns

Include these columns in tables that need soft delete:

```sql
CREATE TABLE dbo.Order (
    OrderId INT NOT NULL,
    CustomerId INT NOT NULL,
    OrderDate DATETIME2(0) NOT NULL,
    Total DECIMAL(18,2) NOT NULL,
    Status VARCHAR(20) NOT NULL,
    
    -- Standard audit columns
    CreatedAt DATETIME2(0) NOT NULL DEFAULT SYSDATETIME(),
    CreatedBy INT NOT NULL,
    ModifiedAt DATETIME2(0) NOT NULL DEFAULT SYSDATETIME(),
    ModifiedBy INT NOT NULL,
    
    -- Soft delete columns
    DeletedAt DATETIME2(0) NULL,
    DeletedBy INT NULL,
    
    CONSTRAINT PK_Order PRIMARY KEY (OrderId),
    CONSTRAINT FK_Order_Customer FOREIGN KEY (CustomerId) 
        REFERENCES dbo.Customer(CustomerId)
);
```

## Filtered Indexes for Performance

Create filtered indexes to optimize queries on active records:

```sql
-- Index for active orders only (most queries)
CREATE NONCLUSTERED INDEX IX_Order_CustomerId_Active
ON dbo.Order(CustomerId, OrderDate)
INCLUDE (Total, Status)
WHERE DeletedAt IS NULL;

-- Smaller index, faster queries on active data
```

## Views for Active Records

Create views to simplify queries and hide soft delete logic:

```sql
-- View shows only active customers
CREATE VIEW dbo.v_ActiveCustomer
AS
SELECT 
    CustomerId,
    FirstName,
    LastName,
    Email,
    CreatedAt,
    CreatedBy
FROM dbo.Customer
WHERE DeletedAt IS NULL;
GO

-- Application code uses the view
SELECT * FROM dbo.v_ActiveCustomer WHERE Email = @Email;

-- Simpler than repeating WHERE DeletedAt IS NULL everywhere
```

## Cascade Soft Deletes

When deleting a parent record, soft delete child records too:

```sql
CREATE PROCEDURE dbo.usp_DeleteOrder
    @OrderId INT,
    @UserId INT
AS
BEGIN
    SET NOCOUNT ON;
    
    BEGIN TRY
        BEGIN TRANSACTION;
        
        DECLARE @Now DATETIME2(0) = SYSDATETIME();
        
        -- Soft delete the order
        UPDATE dbo.Order
        SET 
            DeletedAt = @Now,
            DeletedBy = @UserId
        WHERE OrderId = @OrderId
          AND DeletedAt IS NULL;
        
        IF @@ROWCOUNT = 0
        BEGIN
            THROW 50001, 'Order not found or already deleted', 1;
        END;
        
        -- Cascade soft delete to line items
        UPDATE dbo.OrderLineItem
        SET 
            DeletedAt = @Now,
            DeletedBy = @UserId
        WHERE OrderId = @OrderId
          AND DeletedAt IS NULL;
        
        COMMIT TRANSACTION;
    END TRY
    BEGIN CATCH
        IF @@TRANCOUNT > 0
            ROLLBACK TRANSACTION;
        
        THROW;
    END CATCH;
END;
```

## Handling Unique Constraints

Soft deletes complicate unique constraints because deleted records still exist:

```sql
-- ❌ Problem: Can't reuse email after soft delete
CREATE TABLE dbo.Customer (
    CustomerId INT NOT NULL,
    Email NVARCHAR(255) NOT NULL,
    DeletedAt DATETIME2(0) NULL,
    
    CONSTRAINT PK_Customer PRIMARY KEY (CustomerId),
    CONSTRAINT UQ_Customer_Email UNIQUE (Email)  -- Includes deleted records!
);

-- Delete customer with email 'alice@example.com'
UPDATE dbo.Customer SET DeletedAt = SYSDATETIME() WHERE CustomerId = 1;

-- Try to create new customer with same email
INSERT INTO dbo.Customer (CustomerId, Email) VALUES (2, 'alice@example.com');
-- ❌ Error: Duplicate email!

-- ✅ Solution: Filtered unique index
ALTER TABLE dbo.Customer DROP CONSTRAINT UQ_Customer_Email;

CREATE UNIQUE NONCLUSTERED INDEX UQ_Customer_Email_Active
ON dbo.Customer(Email)
WHERE DeletedAt IS NULL;  -- Only enforce uniqueness for active records

-- Now it works:
INSERT INTO dbo.Customer (CustomerId, Email) VALUES (2, 'alice@example.com');
-- ✅ Success: email unique among active customers
```

## Automatic Soft Delete with Triggers

Use triggers to automatically soft delete child records:

```sql
CREATE TRIGGER TR_Customer_SoftDelete
ON dbo.Customer
AFTER UPDATE
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Check if this is a soft delete operation
    IF UPDATE(DeletedAt)
    BEGIN
        -- Cascade soft delete to orders
        UPDATE o
        SET 
            o.DeletedAt = i.DeletedAt,
            o.DeletedBy = i.DeletedBy
        FROM dbo.Order o
        INNER JOIN inserted i ON o.CustomerId = i.CustomerId
        WHERE o.DeletedAt IS NULL
          AND i.DeletedAt IS NOT NULL;
    END;
END;
```

## Why Hard Deletes Are a Problem

1. **Lost audit trail**: No record of who deleted data or when
2. **Accidental deletion**: No recovery mechanism for mistakes
3. **Compliance issues**: Many regulations require data retention
4. **Breaking relationships**: Foreign key constraints prevent deletion of referenced records
5. **Lost analytics**: Historical reports become impossible

## Symptoms

- `DELETE` statements in production code
- Customer complaints about lost data
- Inability to answer "what happened to this record?"
- Foreign key constraint violations preventing deletions

## Benefits

- **Audit trail**: Complete history of deletions
- **Recovery**: Undelete accidentally removed records
- **Historical queries**: Analyze data as it existed at any point
- **Compliance**: Meet data retention requirements
- **Referential integrity**: No foreign key constraint issues

## When to Use Hard Deletes

Soft deletes aren't always appropriate. Use hard deletes when:

- **PII regulations**: GDPR/CCPA require permanent deletion
- **Truly temporary data**: Shopping carts, sessions, cache tables
- **Extremely high volume**: Soft deletes increase table size indefinitely
- **No business value**: Some data has no audit or recovery value

## Archiving Old Soft Deletes

For high-volume tables, move old soft deletes to archive tables:

```sql
-- Archive table (same structure)
CREATE TABLE dbo.Customer_Archive (
    CustomerId INT NOT NULL,
    FirstName NVARCHAR(100) NOT NULL,
    LastName NVARCHAR(100) NOT NULL,
    Email NVARCHAR(255) NOT NULL,
    CreatedAt DATETIME2(0) NOT NULL,
    CreatedBy INT NOT NULL,
    DeletedAt DATETIME2(0) NOT NULL,  -- Always set in archive
    DeletedBy INT NOT NULL,
    ArchivedAt DATETIME2(0) NOT NULL DEFAULT SYSDATETIME(),
    
    CONSTRAINT PK_Customer_Archive PRIMARY KEY (CustomerId)
);

-- Move old soft deletes to archive (run monthly)
INSERT INTO dbo.Customer_Archive 
    (CustomerId, FirstName, LastName, Email, CreatedAt, CreatedBy, DeletedAt, DeletedBy)
SELECT 
    CustomerId, FirstName, LastName, Email, CreatedAt, CreatedBy, DeletedAt, DeletedBy
FROM dbo.Customer
WHERE DeletedAt < DATEADD(YEAR, -1, GETDATE());  -- Older than 1 year

-- Remove from main table
DELETE FROM dbo.Customer
WHERE DeletedAt < DATEADD(YEAR, -1, GETDATE());
```

## See Also

- [Audit Columns](./audit-columns.md)
- [Temporal Tables](./temporal-tables.md)
- [Event Sourcing](./event-sourcing.md)

# Temporal Tables

> Use SQL Server's system-versioned temporal tables to automatically track all changes—query data as it existed at any point in time.

## Problem

Manually tracking data history requires custom audit tables, triggers, and complex queries. This is error-prone, adds development overhead, and makes historical queries difficult.

## Example

### ❌ Before (Manual History Tracking)

```sql
-- Main table
CREATE TABLE dbo.Product (
    ProductId INT NOT NULL,
    Name NVARCHAR(200) NOT NULL,
    Price DECIMAL(18,2) NOT NULL,
    ModifiedAt DATETIME2(0) NOT NULL,
    ModifiedBy INT NOT NULL,
    
    CONSTRAINT PK_Product PRIMARY KEY (ProductId)
);

-- Separate history table (manually maintained)
CREATE TABLE dbo.ProductHistory (
    ProductHistoryId INT IDENTITY(1,1) NOT NULL,
    ProductId INT NOT NULL,
    Name NVARCHAR(200) NOT NULL,
    Price DECIMAL(18,2) NOT NULL,
    ModifiedAt DATETIME2(0) NOT NULL,
    ModifiedBy INT NOT NULL,
    OperationType VARCHAR(10) NOT NULL,  -- 'INSERT', 'UPDATE', 'DELETE'
    
    CONSTRAINT PK_ProductHistory PRIMARY KEY (ProductHistoryId)
);

-- Trigger to maintain history
CREATE TRIGGER TR_Product_History
ON dbo.Product
AFTER INSERT, UPDATE, DELETE
AS
BEGIN
    -- Complex logic to track changes
    -- Easy to forget to update
    -- Performance overhead on every write
END;

-- Querying history is complex
SELECT * FROM dbo.ProductHistory
WHERE ProductId = @ProductId
  AND ModifiedAt <= @HistoricalDate
ORDER BY ModifiedAt DESC;
```

### ✅ After (System-Versioned Temporal Table)

```sql
-- Create temporal table (SQL Server does the rest automatically)
CREATE TABLE dbo.Product (
    ProductId INT NOT NULL,
    Name NVARCHAR(200) NOT NULL,
    Price DECIMAL(18,2) NOT NULL,
    ModifiedBy INT NOT NULL,
    
    -- Required period columns
    ValidFrom DATETIME2(2) GENERATED ALWAYS AS ROW START HIDDEN NOT NULL,
    ValidTo DATETIME2(2) GENERATED ALWAYS AS ROW END HIDDEN NOT NULL,
    PERIOD FOR SYSTEM_TIME (ValidFrom, ValidTo),
    
    CONSTRAINT PK_Product PRIMARY KEY (ProductId)
)
WITH (SYSTEM_VERSIONING = ON (HISTORY_TABLE = dbo.ProductHistory));

-- SQL Server automatically:
-- - Creates history table
-- - Tracks all changes
-- - Maintains ValidFrom/ValidTo timestamps
-- - No triggers needed!

-- Query current data (normal query)
SELECT ProductId, Name, Price
FROM dbo.Product
WHERE ProductId = @ProductId;

-- Query as of specific date
SELECT ProductId, Name, Price
FROM dbo.Product
FOR SYSTEM_TIME AS OF '2024-01-01T00:00:00'
WHERE ProductId = @ProductId;

-- Query all changes between dates
SELECT ProductId, Name, Price, ValidFrom, ValidTo
FROM dbo.Product
FOR SYSTEM_TIME BETWEEN '2024-01-01' AND '2024-12-31'
WHERE ProductId = @ProductId;

-- Query all history
SELECT ProductId, Name, Price, ValidFrom, ValidTo
FROM dbo.Product
FOR SYSTEM_TIME ALL
WHERE ProductId = @ProductId
ORDER BY ValidFrom;
```

## Temporal Query Clauses

### AS OF

Get data as it existed at a specific point in time:

```sql
-- What was the price on January 1, 2024?
SELECT ProductId, Name, Price
FROM dbo.Product
FOR SYSTEM_TIME AS OF '2024-01-01T12:00:00'
WHERE ProductId = 123;
```

### FROM ... TO

Get all versions that were valid during a time range (exclusive end):

```sql
-- All versions valid between Jan 1 and Feb 1 (excluding Feb 1)
SELECT ProductId, Name, Price, ValidFrom, ValidTo
FROM dbo.Product
FOR SYSTEM_TIME FROM '2024-01-01' TO '2024-02-01'
WHERE ProductId = 123;
```

### BETWEEN ... AND

Get all versions that were valid during a time range (inclusive):

```sql
-- All versions valid from Jan 1 through Feb 1 (including Feb 1)
SELECT ProductId, Name, Price, ValidFrom, ValidTo
FROM dbo.Product
FOR SYSTEM_TIME BETWEEN '2024-01-01' AND '2024-02-01'
WHERE ProductId = 123;
```

### CONTAINED IN

Get only versions that started and ended within a time range:

```sql
-- Only changes that occurred entirely within this period
SELECT ProductId, Name, Price, ValidFrom, ValidTo
FROM dbo.Product
FOR SYSTEM_TIME CONTAINED IN ('2024-01-01', '2024-02-01')
WHERE ProductId = 123;
```

### ALL

Get current and all historical versions:

```sql
-- Everything, past and present
SELECT ProductId, Name, Price, ValidFrom, ValidTo
FROM dbo.Product
FOR SYSTEM_TIME ALL
WHERE ProductId = 123
ORDER BY ValidFrom;
```

## Adding Temporal to Existing Tables

Convert an existing table to temporal:

```sql
-- 1. Add period columns
ALTER TABLE dbo.Product
ADD 
    ValidFrom DATETIME2(2) GENERATED ALWAYS AS ROW START HIDDEN NOT NULL
        CONSTRAINT DF_Product_ValidFrom DEFAULT SYSUTCDATETIME(),
    ValidTo DATETIME2(2) GENERATED ALWAYS AS ROW END HIDDEN NOT NULL
        CONSTRAINT DF_Product_ValidTo DEFAULT CAST('9999-12-31 23:59:59.9999999' AS DATETIME2(2)),
    PERIOD FOR SYSTEM_TIME (ValidFrom, ValidTo);

-- 2. Enable system versioning
ALTER TABLE dbo.Product
SET (SYSTEM_VERSIONING = ON (HISTORY_TABLE = dbo.ProductHistory));

-- SQL Server creates history table automatically
```

## Removing System Versioning

Temporarily disable to perform schema changes:

```sql
-- Disable system versioning
ALTER TABLE dbo.Product
SET (SYSTEM_VERSIONING = OFF);

-- Make schema changes
ALTER TABLE dbo.Product
ADD NewColumn INT NULL;

-- Re-enable system versioning
ALTER TABLE dbo.Product
SET (SYSTEM_VERSIONING = ON (HISTORY_TABLE = dbo.ProductHistory));
```

## Querying Changes Over Time

Track when a product's price changed:

```sql
-- Price history for a product
SELECT 
    ProductId,
    Name,
    Price,
    ValidFrom AS EffectiveDate,
    ValidTo AS ExpirationDate,
    DATEDIFF(DAY, ValidFrom, ValidTo) AS DaysActive
FROM dbo.Product
FOR SYSTEM_TIME ALL
WHERE ProductId = 123
ORDER BY ValidFrom;

-- Who changed what and when (requires audit columns)
CREATE TABLE dbo.Product (
    ProductId INT NOT NULL,
    Name NVARCHAR(200) NOT NULL,
    Price DECIMAL(18,2) NOT NULL,
    ModifiedBy INT NOT NULL,  -- Track who made changes
    
    ValidFrom DATETIME2(2) GENERATED ALWAYS AS ROW START HIDDEN NOT NULL,
    ValidTo DATETIME2(2) GENERATED ALWAYS AS ROW END HIDDEN NOT NULL,
    PERIOD FOR SYSTEM_TIME (ValidFrom, ValidTo),
    
    CONSTRAINT PK_Product PRIMARY KEY (ProductId)
)
WITH (SYSTEM_VERSIONING = ON (HISTORY_TABLE = dbo.ProductHistory));

-- Query changes with user info
SELECT 
    p.ProductId,
    p.Name,
    p.Price,
    p.ValidFrom,
    p.ValidTo,
    p.ModifiedBy,
    u.Username
FROM dbo.Product FOR SYSTEM_TIME ALL p
INNER JOIN dbo.[User] u ON p.ModifiedBy = u.UserId
WHERE p.ProductId = 123
ORDER BY p.ValidFrom;
```

## Performance Considerations

### Index the History Table

```sql
-- Create indexes on history table for better query performance
CREATE NONCLUSTERED INDEX IX_ProductHistory_ProductId_ValidFrom
ON dbo.ProductHistory(ProductId, ValidFrom, ValidTo);

-- Covering index for common queries
CREATE NONCLUSTERED INDEX IX_ProductHistory_ProductId
ON dbo.ProductHistory(ProductId)
INCLUDE (Name, Price, ValidFrom, ValidTo);
```

### Data Retention Policy

Archive or delete old history to manage size:

```sql
-- Delete history older than 7 years (if allowed by policy)
ALTER TABLE dbo.Product
SET (SYSTEM_VERSIONING = OFF);

DELETE FROM dbo.ProductHistory
WHERE ValidTo < DATEADD(YEAR, -7, GETDATE());

ALTER TABLE dbo.Product
SET (SYSTEM_VERSIONING = ON (HISTORY_TABLE = dbo.ProductHistory));
```

### History Table Compression

Enable page compression on history table:

```sql
-- History is read-only, compress it
ALTER TABLE dbo.ProductHistory
REBUILD WITH (DATA_COMPRESSION = PAGE);
```

## HIDDEN Columns

Use HIDDEN to exclude period columns from `SELECT *`:

```sql
-- Period columns are HIDDEN
CREATE TABLE dbo.Product (
    ProductId INT NOT NULL,
    Name NVARCHAR(200) NOT NULL,
    
    ValidFrom DATETIME2(2) GENERATED ALWAYS AS ROW START HIDDEN NOT NULL,
    ValidTo DATETIME2(2) GENERATED ALWAYS AS ROW END HIDDEN NOT NULL,
    PERIOD FOR SYSTEM_TIME (ValidFrom, ValidTo),
    
    CONSTRAINT PK_Product PRIMARY KEY (ProductId)
)
WITH (SYSTEM_VERSIONING = ON);

-- SELECT * excludes ValidFrom and ValidTo
SELECT * FROM dbo.Product;
-- Returns: ProductId, Name (not ValidFrom, ValidTo)

-- Explicitly include them if needed
SELECT ProductId, Name, ValidFrom, ValidTo
FROM dbo.Product FOR SYSTEM_TIME ALL;
```

## Why Manual History Tracking Is a Problem

1. **Development overhead**: Custom triggers, procedures, and tables
2. **Error-prone**: Easy to forget to update history on changes
3. **Performance cost**: Triggers add overhead to every write
4. **Complex queries**: Reconstructing historical state is difficult
5. **Maintenance burden**: Schema changes require updating triggers

## Symptoms

- Custom history tables with `_History` suffix
- Triggers on tables to track changes
- Complex queries to reconstruct historical state
- Incomplete audit trails (forgot to update trigger)

## Benefits

- **Automatic tracking**: SQL Server handles everything
- **Zero code**: No triggers or custom logic needed
- **Simple queries**: Built-in temporal query syntax
- **Point-in-time recovery**: Query data as it existed at any moment
- **Compliance**: Meet audit requirements automatically

## Trade-offs

- **Storage**: History table grows over time
- **SQL Server version**: Requires SQL Server 2016+ or Azure SQL
- **Schema changes**: Must disable versioning to alter table structure
- **No DELETE tracking**: Temporal tables don't track who deleted a row (use soft deletes for that)

## Combining with Soft Deletes

Use both for complete audit trail:

```sql
CREATE TABLE dbo.Customer (
    CustomerId INT NOT NULL,
    FirstName NVARCHAR(100) NOT NULL,
    LastName NVARCHAR(100) NOT NULL,
    Email NVARCHAR(255) NOT NULL,
    
    -- Soft delete columns
    DeletedAt DATETIME2(0) NULL,
    DeletedBy INT NULL,
    
    -- Temporal columns
    ValidFrom DATETIME2(2) GENERATED ALWAYS AS ROW START HIDDEN NOT NULL,
    ValidTo DATETIME2(2) GENERATED ALWAYS AS ROW END HIDDEN NOT NULL,
    PERIOD FOR SYSTEM_TIME (ValidFrom, ValidTo),
    
    CONSTRAINT PK_Customer PRIMARY KEY (CustomerId)
)
WITH (SYSTEM_VERSIONING = ON (HISTORY_TABLE = dbo.CustomerHistory));

-- Track all changes (temporal) AND who deleted what (soft delete)
```

## See Also

- [Audit Columns](./audit-columns.md)
- [Soft Deletes](./soft-deletes.md)
- [Event Sourcing](./event-sourcing.md)

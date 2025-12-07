# Covering Indexes

> Create indexes that include all columns needed by a query—eliminate expensive key lookups.

## Problem

Non-covering indexes require SQL Server to look up additional columns from the base table (key lookups), significantly slowing queries. Each key lookup is a random I/O operation that can dominate query time.

## Example

### ❌ Before (Non-Covering Index)

```sql
CREATE TABLE dbo.Order (
    OrderId INT NOT NULL,
    CustomerId INT NOT NULL,
    OrderDate DATETIME2(0) NOT NULL,
    Total DECIMAL(18,2) NOT NULL,
    Status VARCHAR(20) NOT NULL,
    ShippingAddress NVARCHAR(500) NOT NULL,
    
    CONSTRAINT PK_Order PRIMARY KEY CLUSTERED (OrderId)
);

-- Index on CustomerId only
CREATE NONCLUSTERED INDEX IX_Order_CustomerId
ON dbo.Order(CustomerId);

-- Query needs CustomerId, OrderDate, Total, and Status
SELECT 
    OrderId,
    OrderDate,
    Total,
    Status
FROM dbo.Order
WHERE CustomerId = @CustomerId
  AND OrderDate >= @StartDate
ORDER BY OrderDate DESC;

-- Execution plan shows:
-- 1. Index Seek on IX_Order_CustomerId (finds OrderId for customer)
-- 2. Key Lookup on PK_Order for EACH row (gets OrderDate, Total, Status)
-- 3. Sort by OrderDate
-- Key lookups are EXPENSIVE: random I/O for each row
```

### ✅ After (Covering Index)

```sql
-- Covering index includes all columns needed by the query
CREATE NONCLUSTERED INDEX IX_Order_CustomerId_OrderDate
ON dbo.Order(CustomerId, OrderDate DESC)
INCLUDE (Total, Status);

-- Same query now fully covered
SELECT 
    OrderId,
    OrderDate,
    Total,
    Status
FROM dbo.Order
WHERE CustomerId = @CustomerId
  AND OrderDate >= @StartDate
ORDER BY OrderDate DESC;

-- Execution plan shows:
-- 1. Index Seek on IX_Order_CustomerId_OrderDate
-- NO KEY LOOKUPS! All data read from index.
-- Result is already sorted (OrderDate DESC in index)
```

## Index Structure

### Key Columns

Columns in the index key used for:
- Filtering (`WHERE CustomerId = @CustomerId`)
- Sorting (`ORDER BY OrderDate DESC`)
- Joining (`JOIN ... ON ...`)

```sql
-- Key columns: CustomerId (filter), OrderDate (sort)
CREATE NONCLUSTERED INDEX IX_Order_CustomerId_OrderDate
ON dbo.Order(CustomerId, OrderDate DESC)
```

### INCLUDE Columns

Columns not used for filtering/sorting but needed in the SELECT list:

```sql
-- INCLUDE: Total, Status (in SELECT but not in WHERE/ORDER BY)
CREATE NONCLUSTERED INDEX IX_Order_CustomerId_OrderDate
ON dbo.Order(CustomerId, OrderDate DESC)
INCLUDE (Total, Status);
```

### Benefits of INCLUDE

- **Smaller index keys**: Key columns are in B-tree, INCLUDE columns only in leaf pages
- **No sort order**: INCLUDE columns don't affect index ordering
- **More efficient**: Less index page splits during updates

## Column Order Matters

Put most selective columns first, then those used in equality before ranges:

```sql
-- ✅ Good: Selective column first, equality before range
CREATE NONCLUSTERED INDEX IX_Order_CustomerId_Status_OrderDate
ON dbo.Order(CustomerId, Status, OrderDate)
INCLUDE (Total);

-- Covers:
SELECT OrderId, OrderDate, Total
FROM dbo.Order
WHERE CustomerId = @CustomerId    -- Most selective
  AND Status = 'Pending'           -- Equality
  AND OrderDate >= @StartDate;     -- Range

-- ❌ Bad: Range column in the middle prevents using Status
CREATE NONCLUSTERED INDEX IX_Order_Bad
ON dbo.Order(CustomerId, OrderDate, Status)  -- OrderDate range stops index use
INCLUDE (Total);

-- Can only use CustomerId and OrderDate, not Status
```

## Sort Order in Indexes

Match the query's ORDER BY direction:

```sql
-- Query sorts by OrderDate DESC
SELECT OrderId, CustomerId, OrderDate, Total
FROM dbo.Order
WHERE CustomerId = @CustomerId
ORDER BY OrderDate DESC;

-- ✅ Index matches sort order (DESC)
CREATE NONCLUSTERED INDEX IX_Order_CustomerId_OrderDate
ON dbo.Order(CustomerId, OrderDate DESC)
INCLUDE (Total);
-- No sort needed in execution plan

-- ❌ Index has opposite sort order (ASC)
CREATE NONCLUSTERED INDEX IX_Order_CustomerId_OrderDate_Asc
ON dbo.Order(CustomerId, OrderDate ASC)
INCLUDE (Total);
-- Index can be scanned backward, but sort might still appear
```

## Multiple Queries, One Index

Design indexes to cover multiple related queries:

```sql
-- Common queries:
-- 1. Get customer orders by date range
SELECT OrderId, OrderDate, Total, Status
FROM dbo.Order
WHERE CustomerId = @CustomerId
  AND OrderDate >= @StartDate;

-- 2. Get customer orders by status
SELECT OrderId, OrderDate, Total
FROM dbo.Order
WHERE CustomerId = @CustomerId
  AND Status = @Status;

-- 3. Get customer order count
SELECT COUNT(*)
FROM dbo.Order
WHERE CustomerId = @CustomerId;

-- Single covering index for all three:
CREATE NONCLUSTERED INDEX IX_Order_CustomerId
ON dbo.Order(CustomerId)
INCLUDE (OrderDate, Total, Status);

-- Covers all three queries efficiently
```

## Filtered Indexes

Combine covering with filtering for smaller, faster indexes:

```sql
-- Most queries only look at active orders
SELECT OrderId, OrderDate, Total
FROM dbo.Order
WHERE CustomerId = @CustomerId
  AND DeletedAt IS NULL;

-- Filtered covering index (only active orders)
CREATE NONCLUSTERED INDEX IX_Order_CustomerId_Active
ON dbo.Order(CustomerId)
INCLUDE (OrderDate, Total)
WHERE DeletedAt IS NULL;

-- Smaller index (excludes deleted orders)
-- Faster queries (fewer rows to scan)
-- Lower maintenance cost (deleted orders don't update index)
```

## Avoiding Over-Indexing

Too many indexes slow down writes:

```sql
-- ❌ Too many similar indexes
CREATE INDEX IX_Order_CustomerId ON dbo.Order(CustomerId);
CREATE INDEX IX_Order_CustomerId_OrderDate ON dbo.Order(CustomerId, OrderDate);
CREATE INDEX IX_Order_CustomerId_Status ON dbo.Order(CustomerId, Status);
CREATE INDEX IX_Order_CustomerId_Total ON dbo.Order(CustomerId, Total);

-- ✅ One well-designed index
CREATE NONCLUSTERED INDEX IX_Order_CustomerId
ON dbo.Order(CustomerId)
INCLUDE (OrderDate, Status, Total);

-- Covers all the queries the separate indexes would have
```

## Analyzing Coverage with Execution Plans

Check if your query uses covering indexes:

```sql
-- Enable actual execution plan
SET STATISTICS IO ON;
SET STATISTICS TIME ON;

-- Run your query
SELECT OrderId, OrderDate, Total, Status
FROM dbo.Order
WHERE CustomerId = @CustomerId;

-- Look for in execution plan:
-- ✅ Good: "Index Seek" or "Index Scan", no "Key Lookup"
-- ❌ Bad: "Key Lookup (Clustered)" indicates non-covering

-- Check I/O statistics:
-- Logical reads should only be from your index
```

## When NOT to Use Covering Indexes

- **Too many INCLUDE columns**: Index becomes as large as the table
- **Large VARCHAR/NVARCHAR columns**: Keep these out of indexes
- **Rarely-used queries**: Don't optimize every query, focus on frequent ones
- **High write volume**: Each index adds write overhead

```sql
-- ❌ Don't include large columns
CREATE NONCLUSTERED INDEX IX_Order_Bad
ON dbo.Order(CustomerId)
INCLUDE (OrderDate, Total, Status, ShippingAddress, BillingAddress, Notes);
-- ShippingAddress, BillingAddress, Notes are too large

-- ✅ Exclude large columns
CREATE NONCLUSTERED INDEX IX_Order_Good
ON dbo.Order(CustomerId)
INCLUDE (OrderDate, Total, Status);
-- Query can still do key lookup for large columns when needed
```

## Maintenance

Monitor index usage to remove unused indexes:

```sql
-- Find unused indexes
SELECT 
    OBJECT_NAME(s.object_id) AS TableName,
    i.name AS IndexName,
    s.user_seeks,
    s.user_scans,
    s.user_lookups,
    s.user_updates
FROM sys.dm_db_index_usage_stats s
INNER JOIN sys.indexes i ON s.object_id = i.object_id AND s.index_id = i.index_id
WHERE s.database_id = DB_ID()
  AND OBJECTPROPERTY(s.object_id, 'IsUserTable') = 1
  AND s.user_seeks = 0
  AND s.user_scans = 0
  AND s.user_lookups = 0
  AND s.user_updates > 0  -- Index being maintained but never used
ORDER BY s.user_updates DESC;
```

## Why It's a Problem

1. **Slow queries**: Key lookups are random I/O, much slower than sequential index scans
2. **High CPU**: Sorting results when index doesn't match ORDER BY
3. **Wasted I/O**: Reading same data multiple times (index + table)
4. **Scalability**: Performance degrades as table grows

## Symptoms

- Queries with "Key Lookup" in execution plan
- High logical reads for simple queries
- Queries that select few columns but are still slow
- Sort operations in execution plan

## Benefits

- **Faster queries**: Eliminate key lookups, read only from index
- **Less I/O**: Read data once from index instead of index + table
- **Reduced CPU**: No sorting when index matches ORDER BY
- **Better scalability**: Query time grows slowly with table size

## Trade-offs

- **Storage overhead**: Covering indexes store duplicate data
- **Write overhead**: Every INSERT/UPDATE/DELETE maintains all indexes
- **Maintenance**: More indexes to monitor and update statistics

## See Also

- [Filtered Indexes](./filtered-indexes.md)
- [Indexed Views](./indexed-views.md)
- [Execution Plans](./execution-plans.md)

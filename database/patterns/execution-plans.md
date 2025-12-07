# Execution Plans

> Analyze query execution plans to understand performance—identify missing indexes, expensive operations, and optimization opportunities.

## Problem

Slow queries need diagnosis, but without examining execution plans, developers guess at solutions. Adding random indexes or rewriting queries without understanding the actual problem wastes time.

## Example

### ❌ Before (Guessing at Solutions)

```sql
-- Slow query
SELECT c.FirstName, c.LastName, o.OrderDate, o.Total
FROM dbo.Customer c
INNER JOIN dbo.Order o ON c.CustomerId = o.CustomerId
WHERE c.City = 'Seattle'
  AND o.OrderDate >= '2024-01-01';

-- Developer guesses:
-- "Maybe need index on City?"
CREATE INDEX IX_Customer_City ON dbo.Customer(City);
-- Still slow!

-- "Maybe need index on OrderDate?"
CREATE INDEX IX_Order_OrderDate ON dbo.Order(OrderDate);
-- Still slow!

-- ❌ Trial and error without understanding the problem
```

### ✅ After (Analyzing Execution Plan)

```sql
-- Get execution plan
SET STATISTICS IO ON;
SET STATISTICS TIME ON;

-- Run query and examine plan
SELECT c.FirstName, c.LastName, o.OrderDate, o.Total
FROM dbo.Customer c
INNER JOIN dbo.Order o ON c.CustomerId = o.CustomerId
WHERE c.City = 'Seattle'
  AND o.OrderDate >= '2024-01-01';

-- Plan shows:
-- 1. Table scan on Customer (missing index on City)
-- 2. Index seek on Order.CustomerId (good!)
-- 3. Key lookup for FirstName/LastName (missing covering index)

-- ✅ Create appropriate indexes based on plan
CREATE NONCLUSTERED INDEX IX_Customer_City_Covering
ON dbo.Customer(City)
INCLUDE (FirstName, LastName);

-- Now plan shows index seek + no key lookups (fast!)
```

## Getting Execution Plans

### Estimated Plan

```sql
-- Get estimated plan without executing query
SET SHOWPLAN_ALL ON;
GO

SELECT * FROM dbo.Customer WHERE City = 'Seattle';
GO

SET SHOWPLAN_ALL OFF;
GO

-- Or in SSMS: Ctrl+L (Display Estimated Execution Plan)
```

### Actual Plan

```sql
-- Get actual plan after execution
SET STATISTICS XML ON;
GO

SELECT * FROM dbo.Customer WHERE City = 'Seattle';
GO

SET STATISTICS XML OFF;
GO

-- Or in SSMS: Ctrl+M (Include Actual Execution Plan)
```

### Execution Statistics

```sql
-- Show I/O statistics
SET STATISTICS IO ON;

SELECT * FROM dbo.Order WHERE CustomerId = 123;
-- Logical reads: 4
-- Physical reads: 0
-- Read-ahead reads: 0

SET STATISTICS IO OFF;

-- Show time statistics
SET STATISTICS TIME ON;

SELECT * FROM dbo.Order WHERE CustomerId = 123;
-- CPU time = 0 ms, elapsed time = 1 ms

SET STATISTICS TIME OFF;
```

## Reading Execution Plans

### Operators to Watch For

```sql
-- ❌ Table Scan (reads entire table)
SELECT * FROM dbo.Customer WHERE City = 'Seattle';
-- Shows: Table Scan (dbo.Customer)
-- Cost: High
-- Problem: No index on City

-- ✅ Index Seek (uses index efficiently)
SELECT * FROM dbo.Customer WHERE CustomerId = 123;
-- Shows: Index Seek (PK_Customer)
-- Cost: Low
-- Good: Using clustered index

-- ⚠️ Index Scan (reads entire index)
SELECT * FROM dbo.Customer WHERE FirstName LIKE '%son%';
-- Shows: Index Scan (IX_Customer_FirstName)
-- Cost: Medium-High
-- Problem: Cannot use index for leading wildcard

-- ⚠️ Key Lookup (additional lookup for columns not in index)
SELECT CustomerId, FirstName, LastName, Email, Phone
FROM dbo.Customer
WHERE City = 'Seattle';
-- Shows: Index Seek + Key Lookup
-- Cost: Medium
-- Problem: Email and Phone not in index
```

### Join Operators

```sql
-- ✅ Nested Loop Join (good for small result sets)
SELECT c.FirstName, o.OrderId
FROM dbo.Customer c
INNER JOIN dbo.Order o ON c.CustomerId = o.CustomerId
WHERE c.CustomerId = 123;
-- Shows: Nested Loops
-- Good for: Small outer input

-- ✅ Hash Match (good for large result sets)
SELECT c.FirstName, o.OrderId
FROM dbo.Customer c
INNER JOIN dbo.Order o ON c.CustomerId = o.CustomerId;
-- Shows: Hash Match
-- Good for: Large inputs without indexes

-- ✅ Merge Join (good for sorted inputs)
SELECT c.FirstName, o.OrderId
FROM dbo.Customer c
INNER JOIN dbo.Order o ON c.CustomerId = o.CustomerId
ORDER BY c.CustomerId;
-- Shows: Merge Join
-- Good for: Both inputs sorted on join key
```

## Common Performance Issues

### Missing Index

```sql
-- Query without index
SELECT * FROM dbo.Order
WHERE OrderDate >= '2024-01-01'
  AND Status = 'Pending';

-- Plan shows: Table Scan
-- Missing Index suggestion in plan:
-- CREATE INDEX IX_Order_OrderDate_Status
-- ON dbo.Order(OrderDate, Status)
-- INCLUDE (CustomerId, Total);

-- Apply suggestion
CREATE NONCLUSTERED INDEX IX_Order_OrderDate_Status
ON dbo.Order(OrderDate, Status)
INCLUDE (CustomerId, Total);

-- Now plan shows: Index Seek
```

### Key Lookup (RID Lookup)

```sql
-- Non-covering index
CREATE INDEX IX_Customer_City 
ON dbo.Customer(City);

-- Query requires columns not in index
SELECT CustomerId, City, FirstName, LastName, Email
FROM dbo.Customer
WHERE City = 'Seattle';

-- Plan shows:
-- 1. Index Seek on IX_Customer_City
-- 2. Key Lookup for FirstName, LastName, Email
-- Cost: ~50% seek + ~50% lookup

-- Solution: Covering index
DROP INDEX IX_Customer_City ON dbo.Customer;

CREATE INDEX IX_Customer_City_Covering
ON dbo.Customer(City)
INCLUDE (FirstName, LastName, Email);

-- Now plan shows: Index Seek only (no key lookup)
```

### Parameter Sniffing

```sql
-- Procedure with parameter sniffing issue
CREATE PROCEDURE dbo.usp_GetOrders
    @Status VARCHAR(20)
AS
BEGIN
    SELECT OrderId, CustomerId, OrderDate, Total
    FROM dbo.Order
    WHERE Status = @Status;
END;
GO

-- First call: 'Delivered' (99% of orders)
EXEC dbo.usp_GetOrders @Status = 'Delivered';
-- Plan: Table Scan (correct for large result set)

-- Second call: 'Pending' (1% of orders)
EXEC dbo.usp_GetOrders @Status = 'Pending';
-- Plan: Table Scan (wrong! Should be Index Seek)
-- Reuses plan from 'Delivered' call

-- Solution 1: Recompile per execution
ALTER PROCEDURE dbo.usp_GetOrders
    @Status VARCHAR(20)
WITH RECOMPILE
AS
BEGIN
    SELECT OrderId, CustomerId, OrderDate, Total
    FROM dbo.Order
    WHERE Status = @Status;
END;

-- Solution 2: Optimize for unknown
ALTER PROCEDURE dbo.usp_GetOrders
    @Status VARCHAR(20)
AS
BEGIN
    SELECT OrderId, CustomerId, OrderDate, Total
    FROM dbo.Order
    WHERE Status = @Status
    OPTION (OPTIMIZE FOR UNKNOWN);
END;

-- Solution 3: Local variable
ALTER PROCEDURE dbo.usp_GetOrders
    @Status VARCHAR(20)
AS
BEGIN
    DECLARE @StatusLocal VARCHAR(20) = @Status;
    
    SELECT OrderId, CustomerId, OrderDate, Total
    FROM dbo.Order
    WHERE Status = @StatusLocal;
END;
```

### Implicit Conversion

```sql
-- Table with VARCHAR column
CREATE TABLE dbo.Customer (
    CustomerId INT NOT NULL,
    CustomerCode VARCHAR(20) NOT NULL,
    
    CONSTRAINT PK_Customer PRIMARY KEY (CustomerId)
);

CREATE INDEX IX_Customer_CustomerCode 
ON dbo.Customer(CustomerCode);

-- Query with NVARCHAR parameter (implicit conversion)
DECLARE @Code NVARCHAR(20) = N'CUST001';

SELECT * FROM dbo.Customer
WHERE CustomerCode = @Code;  -- ❌ VARCHAR = NVARCHAR

-- Plan shows:
-- CONVERT_IMPLICIT(nvarchar(20), CustomerCode, 0) = @Code
-- Table Scan! (Cannot use index due to function on column)

-- Solution: Match data types
DECLARE @Code VARCHAR(20) = 'CUST001';

SELECT * FROM dbo.Customer
WHERE CustomerCode = @Code;  -- ✅ VARCHAR = VARCHAR

-- Plan shows: Index Seek
```

### Missing Statistics

```sql
-- Check statistics age
SELECT 
    OBJECT_NAME(s.object_id) AS TableName,
    s.name AS StatsName,
    sp.last_updated,
    sp.rows,
    sp.rows_sampled,
    sp.modification_counter AS RowsModifiedSinceUpdate
FROM sys.stats s
CROSS APPLY sys.dm_db_stats_properties(s.object_id, s.stats_id) sp
WHERE OBJECT_NAME(s.object_id) = 'Order'
ORDER BY sp.modification_counter DESC;

-- Update statistics manually
UPDATE STATISTICS dbo.Order;

-- Update specific statistics
UPDATE STATISTICS dbo.Order IX_Order_Status;

-- Create statistics manually (if auto-create disabled)
CREATE STATISTICS ST_Order_CustomerId
ON dbo.Order(CustomerId)
WITH FULLSCAN;
```

### Cardinality Estimation Issues

```sql
-- Plan shows poor estimate
SELECT o.OrderId, o.Total
FROM dbo.Order o
WHERE o.CustomerId = 123
  AND o.OrderDate >= '2024-01-01';

-- Plan shows:
-- Estimated rows: 1000
-- Actual rows: 10
-- ❌ Huge difference causes suboptimal plan

-- Check statistics histogram
DBCC SHOW_STATISTICS('dbo.Order', 'IX_Order_CustomerId');

-- Solution 1: Update statistics with full scan
UPDATE STATISTICS dbo.Order IX_Order_CustomerId WITH FULLSCAN;

-- Solution 2: Create statistics on column combination
CREATE STATISTICS ST_Order_CustomerId_OrderDate
ON dbo.Order(CustomerId, OrderDate);

-- Solution 3: Use query hint
SELECT o.OrderId, o.Total
FROM dbo.Order o
WHERE o.CustomerId = 123
  AND o.OrderDate >= '2024-01-01'
OPTION (RECOMPILE);  -- Force recompile for accurate estimate
```

## Query Hints

### Index Hints

```sql
-- Force specific index
SELECT * FROM dbo.Order WITH (INDEX(IX_Order_CustomerId))
WHERE CustomerId = 123;

-- Force table scan
SELECT * FROM dbo.Order WITH (INDEX(0))
WHERE CustomerId = 123;

-- Force clustered index
SELECT * FROM dbo.Order WITH (INDEX(1))
WHERE CustomerId = 123;
```

### Join Hints

```sql
-- Force specific join type
SELECT c.FirstName, o.OrderId
FROM dbo.Customer c
INNER LOOP JOIN dbo.Order o ON c.CustomerId = o.CustomerId;  -- Nested loop

-- Or
SELECT c.FirstName, o.OrderId
FROM dbo.Customer c
INNER HASH JOIN dbo.Order o ON c.CustomerId = o.CustomerId;  -- Hash join

-- Or
SELECT c.FirstName, o.OrderId
FROM dbo.Customer c
INNER MERGE JOIN dbo.Order o ON c.CustomerId = o.CustomerId;  -- Merge join
```

### Query Options

```sql
-- Recompile (avoid parameter sniffing)
SELECT * FROM dbo.Order
WHERE Status = @Status
OPTION (RECOMPILE);

-- Use plan guide
SELECT * FROM dbo.Order
WHERE Status = @Status
OPTION (OPTIMIZE FOR (@Status = 'Pending'));

-- Parallelize
SELECT COUNT(*) FROM dbo.Order
OPTION (MAXDOP 4);

-- Force order
SELECT * FROM dbo.Order
WHERE Status = 'Pending'
  AND OrderDate >= '2024-01-01'
OPTION (FORCE ORDER);
```

## Plan Cache

### Viewing Cached Plans

```sql
-- Find plans using most CPU
SELECT TOP 10
    qs.execution_count,
    qs.total_worker_time / 1000000 AS total_cpu_sec,
    qs.total_elapsed_time / 1000000 AS total_elapsed_sec,
    qs.total_logical_reads,
    SUBSTRING(st.text, (qs.statement_start_offset/2)+1,
        ((CASE qs.statement_end_offset
            WHEN -1 THEN DATALENGTH(st.text)
            ELSE qs.statement_end_offset
        END - qs.statement_start_offset)/2) + 1) AS statement_text
FROM sys.dm_exec_query_stats qs
CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) st
ORDER BY qs.total_worker_time DESC;

-- View plan for specific query
SELECT 
    qs.execution_count,
    qs.total_elapsed_time / qs.execution_count AS avg_elapsed_time,
    qs.plan_handle,
    qp.query_plan
FROM sys.dm_exec_query_stats qs
CROSS APPLY sys.dm_exec_query_plan(qs.plan_handle) qp
CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) st
WHERE st.text LIKE '%YourQuery%'
ORDER BY qs.total_elapsed_time DESC;
```

### Clearing Plan Cache

```sql
-- Clear entire plan cache (use with caution!)
DBCC FREEPROCCACHE;

-- Clear cache for specific database
ALTER DATABASE SCOPED CONFIGURATION CLEAR PROCEDURE_CACHE;

-- Clear specific plan
DBCC FREEPROCCACHE (plan_handle);

-- Clear procedure cache for specific procedure
EXEC sp_recompile 'dbo.usp_MyProcedure';
```

## Query Store

```sql
-- Enable Query Store (SQL Server 2016+)
ALTER DATABASE MyDatabase
SET QUERY_STORE = ON (
    OPERATION_MODE = READ_WRITE,
    CLEANUP_POLICY = (STALE_QUERY_THRESHOLD_DAYS = 30),
    DATA_FLUSH_INTERVAL_SECONDS = 900,
    MAX_STORAGE_SIZE_MB = 1000,
    INTERVAL_LENGTH_MINUTES = 60
);

-- View top resource-consuming queries
SELECT TOP 10
    q.query_id,
    qt.query_sql_text,
    rs.count_executions,
    rs.avg_duration / 1000 AS avg_duration_ms,
    rs.avg_cpu_time / 1000 AS avg_cpu_ms,
    rs.avg_logical_io_reads
FROM sys.query_store_query q
INNER JOIN sys.query_store_query_text qt ON q.query_text_id = qt.query_text_id
INNER JOIN sys.query_store_plan p ON q.query_id = p.query_id
INNER JOIN sys.query_store_runtime_stats rs ON p.plan_id = rs.plan_id
WHERE rs.last_execution_time >= DATEADD(HOUR, -24, GETUTCDATE())
ORDER BY rs.avg_duration DESC;

-- Find queries with plan regressions
SELECT 
    q.query_id,
    qt.query_sql_text,
    p1.plan_id AS old_plan_id,
    p2.plan_id AS new_plan_id,
    rs1.avg_duration AS old_avg_duration,
    rs2.avg_duration AS new_avg_duration,
    (rs2.avg_duration - rs1.avg_duration) * 100.0 / rs1.avg_duration AS regression_percent
FROM sys.query_store_query q
INNER JOIN sys.query_store_query_text qt ON q.query_text_id = qt.query_text_id
INNER JOIN sys.query_store_plan p1 ON q.query_id = p1.query_id
INNER JOIN sys.query_store_plan p2 ON q.query_id = p2.query_id
INNER JOIN sys.query_store_runtime_stats rs1 ON p1.plan_id = rs1.plan_id
INNER JOIN sys.query_store_runtime_stats rs2 ON p2.plan_id = rs2.plan_id
WHERE p1.plan_id < p2.plan_id
  AND rs2.avg_duration > rs1.avg_duration * 1.5  -- 50% slower
ORDER BY regression_percent DESC;

-- Force specific plan
EXEC sp_query_store_force_plan @query_id = 123, @plan_id = 456;

-- Unforce plan
EXEC sp_query_store_unforce_plan @query_id = 123, @plan_id = 456;
```

## Best Practices

1. **Always check execution plans** before adding indexes
2. **Look for table scans** and index scans in WHERE clauses
3. **Identify key lookups** that could be covered
4. **Watch for implicit conversions** (function calls on columns)
5. **Check cardinality estimates** (estimated vs actual rows)
6. **Update statistics** regularly
7. **Use Query Store** to track performance over time
8. **Test with production-like data** volumes
9. **Avoid hints** unless absolutely necessary
10. **Profile top queries** and optimize them first

## Why Ignoring Execution Plans Is a Problem

1. **Wasted effort**: Random index changes without understanding
2. **Incorrect solutions**: Adding indexes that don't help
3. **Performance degradation**: New indexes slow writes without helping reads
4. **No visibility**: Cannot see what optimizer is doing
5. **Reactive tuning**: Wait for problems instead of proactive optimization

## Symptoms

- Slow queries with no clear cause
- Adding indexes doesn't help
- Query performance varies unpredictably
- "It was fast in dev, slow in production"
- Cannot explain why query is slow

## Benefits

- **Understand bottlenecks**: See exactly where time is spent
- **Targeted optimization**: Add indexes that actually help
- **Validate changes**: Prove optimization improved plan
- **Learn patterns**: Recognize common issues quickly
- **Communicate**: Show team why query is slow

## Trade-offs

- **Learning curve**: Takes time to read plans
- **Analysis time**: Must examine plans, not just run queries
- **Tool dependency**: Need SSMS or other plan viewer
- **Plan complexity**: Large queries have complex plans

## When to Use

✅ Always analyze plans when:
- Query is slower than expected
- Adding/changing indexes
- Optimizing stored procedures
- Troubleshooting performance
- Learning query optimization

## See Also

- [Covering Indexes](./covering-indexes.md)
- [Filtered Indexes](./filtered-indexes.md)
- [Indexed Views](./indexed-views.md)
- [Parameterized Queries](./parameterized-queries.md)

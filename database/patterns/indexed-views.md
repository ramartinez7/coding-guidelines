# Indexed Views

> Materialize complex query results with clustered indexes—precompute aggregations and joins for fast read performance.

## Problem

Complex queries with aggregations, joins, and calculations run slowly even with proper indexes. Repeatedly computing the same expensive results wastes CPU and I/O resources.

## Example

### ❌ Before (Expensive Query)

```sql
-- Tables
CREATE TABLE dbo.Order (
    OrderId INT NOT NULL,
    CustomerId INT NOT NULL,
    OrderDate DATETIME2(0) NOT NULL,
    Status VARCHAR(20) NOT NULL,
    CONSTRAINT PK_Order PRIMARY KEY (OrderId)
);

CREATE TABLE dbo.OrderLineItem (
    OrderLineItemId INT NOT NULL,
    OrderId INT NOT NULL,
    ProductId INT NOT NULL,
    Quantity INT NOT NULL,
    UnitPrice DECIMAL(18,2) NOT NULL,
    CONSTRAINT PK_OrderLineItem PRIMARY KEY (OrderLineItemId)
);

-- Expensive query run frequently
SELECT 
    o.CustomerId,
    COUNT(*) AS OrderCount,
    SUM(oli.Quantity * oli.UnitPrice) AS TotalRevenue,
    AVG(oli.Quantity * oli.UnitPrice) AS AvgOrderValue
FROM dbo.Order o
INNER JOIN dbo.OrderLineItem oli ON o.OrderId = oli.OrderId
WHERE o.Status = 'Delivered'
GROUP BY o.CustomerId;
-- ❌ Scans millions of rows, computes aggregates every time
```

### ✅ After (Indexed View)

```sql
-- Create view with SCHEMABINDING
CREATE VIEW dbo.vw_CustomerRevenue
WITH SCHEMABINDING
AS
SELECT 
    o.CustomerId,
    COUNT_BIG(*) AS OrderCount,  -- Must use COUNT_BIG for indexed views
    SUM(oli.Quantity * oli.UnitPrice) AS TotalRevenue,
    SUM(oli.Quantity * oli.UnitPrice) / COUNT_BIG(*) AS AvgOrderValue
FROM dbo.Order o
INNER JOIN dbo.OrderLineItem oli ON o.OrderId = oli.OrderId
WHERE o.Status = 'Delivered'
GROUP BY o.CustomerId;
GO

-- Create unique clustered index on view
CREATE UNIQUE CLUSTERED INDEX IX_vw_CustomerRevenue_CustomerId
ON dbo.vw_CustomerRevenue(CustomerId);
GO

-- Query now uses materialized view
SELECT CustomerId, OrderCount, TotalRevenue, AvgOrderValue
FROM dbo.vw_CustomerRevenue
WHERE CustomerId = 123;
-- ✅ Index seek on materialized data (instant results)
```

## Requirements for Indexed Views

### Must Use SCHEMABINDING

```sql
-- ✅ Correct: WITH SCHEMABINDING
CREATE VIEW dbo.vw_ProductSummary
WITH SCHEMABINDING
AS
SELECT 
    CategoryId,
    COUNT_BIG(*) AS ProductCount,
    AVG(Price) AS AvgPrice
FROM dbo.Product
GROUP BY CategoryId;
GO

-- ❌ Error without SCHEMABINDING
CREATE VIEW dbo.vw_ProductSummary
AS
SELECT 
    CategoryId,
    COUNT(*) AS ProductCount
FROM dbo.Product
GROUP BY CategoryId;
GO

CREATE CLUSTERED INDEX IX ON dbo.vw_ProductSummary(CategoryId);
-- Error: Cannot index views which are not schema bound.
```

### Must Use Two-Part Names

```sql
-- ❌ Error: single-part name
CREATE VIEW dbo.vw_OrderSummary
WITH SCHEMABINDING
AS
SELECT 
    CustomerId,
    COUNT_BIG(*) AS OrderCount
FROM Order  -- Missing schema prefix
GROUP BY CustomerId;

-- ✅ Correct: schema-qualified names
CREATE VIEW dbo.vw_OrderSummary
WITH SCHEMABINDING
AS
SELECT 
    CustomerId,
    COUNT_BIG(*) AS OrderCount
FROM dbo.Order  -- Schema-qualified
GROUP BY CustomerId;
```

### Must Use COUNT_BIG

```sql
-- ❌ Error: COUNT(*) not allowed
CREATE VIEW dbo.vw_Summary
WITH SCHEMABINDING
AS
SELECT 
    CategoryId,
    COUNT(*) AS Total  -- Error with indexed view
FROM dbo.Product
GROUP BY CategoryId;

-- ✅ Correct: COUNT_BIG(*)
CREATE VIEW dbo.vw_Summary
WITH SCHEMABINDING
AS
SELECT 
    CategoryId,
    COUNT_BIG(*) AS Total
FROM dbo.Product
GROUP BY CategoryId;
```

### Must Have Unique Clustered Index First

```sql
-- ✅ Correct: Unique clustered index first
CREATE VIEW dbo.vw_ProductSummary
WITH SCHEMABINDING
AS
SELECT 
    ProductId,
    CategoryId,
    Price
FROM dbo.Product;
GO

CREATE UNIQUE CLUSTERED INDEX IX_vw_ProductSummary_ProductId
ON dbo.vw_ProductSummary(ProductId);
GO

-- Now can add nonclustered indexes
CREATE NONCLUSTERED INDEX IX_vw_ProductSummary_CategoryId
ON dbo.vw_ProductSummary(CategoryId);
GO
```

## Common Patterns

### Aggregated Data

```sql
CREATE VIEW dbo.vw_DailySales
WITH SCHEMABINDING
AS
SELECT 
    CAST(OrderDate AS DATE) AS SaleDate,
    COUNT_BIG(*) AS OrderCount,
    SUM(Total) AS TotalRevenue,
    AVG(Total) AS AvgOrderValue,
    MIN(Total) AS MinOrderValue,
    MAX(Total) AS MaxOrderValue
FROM dbo.Order
WHERE Status = 'Delivered'
GROUP BY CAST(OrderDate AS DATE);
GO

CREATE UNIQUE CLUSTERED INDEX IX_vw_DailySales_Date
ON dbo.vw_DailySales(SaleDate);
GO

-- Query daily sales (instant)
SELECT SaleDate, TotalRevenue
FROM dbo.vw_DailySales
WHERE SaleDate >= '2024-01-01';
-- Index seek on precomputed data
```

### Denormalized Joins

```sql
CREATE VIEW dbo.vw_OrderDetails
WITH SCHEMABINDING
AS
SELECT 
    o.OrderId,
    o.OrderDate,
    o.Status,
    c.CustomerId,
    c.FirstName,
    c.LastName,
    c.Email
FROM dbo.Order o
INNER JOIN dbo.Customer c ON o.CustomerId = c.CustomerId;
GO

CREATE UNIQUE CLUSTERED INDEX IX_vw_OrderDetails_OrderId
ON dbo.vw_OrderDetails(OrderId);
GO

CREATE NONCLUSTERED INDEX IX_vw_OrderDetails_CustomerId
ON dbo.vw_OrderDetails(CustomerId);
GO

-- Query avoids join
SELECT OrderId, OrderDate, FirstName, LastName
FROM dbo.vw_OrderDetails
WHERE CustomerId = 123;
-- No join needed, instant results
```

### Complex Calculations

```sql
CREATE VIEW dbo.vw_ProductProfitMargin
WITH SCHEMABINDING
AS
SELECT 
    p.ProductId,
    p.Name,
    p.Price,
    p.Cost,
    p.Price - p.Cost AS Profit,
    CASE 
        WHEN p.Price > 0 THEN ((p.Price - p.Cost) / p.Price) * 100
        ELSE 0
    END AS ProfitMarginPercent
FROM dbo.Product p
WHERE p.IsActive = 1;
GO

CREATE UNIQUE CLUSTERED INDEX IX_vw_ProductProfitMargin_ProductId
ON dbo.vw_ProductProfitMargin(ProductId);
GO

-- Query precomputed calculations
SELECT ProductId, Name, ProfitMarginPercent
FROM dbo.vw_ProductProfitMargin
WHERE ProfitMarginPercent > 20;
```

### Multi-Table Aggregations

```sql
CREATE VIEW dbo.vw_CustomerOrderSummary
WITH SCHEMABINDING
AS
SELECT 
    c.CustomerId,
    c.FirstName,
    c.LastName,
    COUNT_BIG(*) AS OrderCount,
    SUM(o.Total) AS TotalSpent,
    MAX(o.OrderDate) AS LastOrderDate,
    MIN(o.OrderDate) AS FirstOrderDate
FROM dbo.Customer c
INNER JOIN dbo.Order o ON c.CustomerId = o.CustomerId
WHERE o.Status = 'Delivered'
GROUP BY c.CustomerId, c.FirstName, c.LastName;
GO

CREATE UNIQUE CLUSTERED INDEX IX_vw_CustomerOrderSummary_CustomerId
ON dbo.vw_CustomerOrderSummary(CustomerId);
GO

-- Instant customer metrics
SELECT CustomerId, FirstName, TotalSpent, OrderCount
FROM dbo.vw_CustomerOrderSummary
WHERE TotalSpent > 1000
ORDER BY TotalSpent DESC;
```

## Automatic View Matching

```sql
-- Create indexed view
CREATE VIEW dbo.vw_ProductByCategory
WITH SCHEMABINDING
AS
SELECT 
    CategoryId,
    COUNT_BIG(*) AS ProductCount,
    AVG(Price) AS AvgPrice
FROM dbo.Product
GROUP BY CategoryId;
GO

CREATE UNIQUE CLUSTERED INDEX IX
ON dbo.vw_ProductByCategory(CategoryId);
GO

-- Query can use view automatically (Enterprise Edition)
SELECT CategoryId, COUNT(*) AS Total, AVG(Price) AS Avg
FROM dbo.Product
GROUP BY CategoryId;
-- Optimizer automatically uses indexed view

-- Force view usage (Standard Edition)
SELECT CategoryId, ProductCount, AvgPrice
FROM dbo.vw_ProductByCategory WITH (NOEXPAND);
```

## Maintenance and Updates

```sql
-- Indexed views are automatically maintained
CREATE VIEW dbo.vw_DailySales
WITH SCHEMABINDING
AS
SELECT 
    CAST(OrderDate AS DATE) AS SaleDate,
    COUNT_BIG(*) AS OrderCount,
    SUM(Total) AS TotalRevenue
FROM dbo.Order
GROUP BY CAST(OrderDate AS DATE);
GO

CREATE UNIQUE CLUSTERED INDEX IX ON dbo.vw_DailySales(SaleDate);
GO

-- Insert order
INSERT INTO dbo.Order (OrderId, CustomerId, OrderDate, Total)
VALUES (1, 100, '2024-01-15', 99.99);
-- ✅ View automatically updated (SaleDate 2024-01-15 incremented)

-- Update order
UPDATE dbo.Order
SET Total = 149.99
WHERE OrderId = 1;
-- ✅ View automatically updated (revenue recalculated)

-- Delete order
DELETE FROM dbo.Order WHERE OrderId = 1;
-- ✅ View automatically updated (count/revenue decreased)
```

## Restrictions

### No Nullable Expressions in Index Key

```sql
-- ❌ Error: nullable columns in GROUP BY
CREATE VIEW dbo.vw_Summary
WITH SCHEMABINDING
AS
SELECT 
    CategoryId,  -- Nullable
    COUNT_BIG(*) AS Total
FROM dbo.Product
GROUP BY CategoryId;
GO

CREATE UNIQUE CLUSTERED INDEX IX ON dbo.vw_Summary(CategoryId);
-- Error: Cannot create index on view because key column 'CategoryId' is nullable.

-- ✅ Solution: Filter out NULLs
CREATE VIEW dbo.vw_Summary
WITH SCHEMABINDING
AS
SELECT 
    CategoryId,
    COUNT_BIG(*) AS Total
FROM dbo.Product
WHERE CategoryId IS NOT NULL
GROUP BY CategoryId;
GO

CREATE UNIQUE CLUSTERED INDEX IX ON dbo.vw_Summary(CategoryId);
```

### No Outer Joins in Indexed Views

```sql
-- ❌ Not allowed: LEFT/RIGHT/FULL OUTER JOIN
CREATE VIEW dbo.vw_OrderWithCustomer
WITH SCHEMABINDING
AS
SELECT 
    o.OrderId,
    o.Total,
    c.FirstName
FROM dbo.Order o
LEFT OUTER JOIN dbo.Customer c ON o.CustomerId = c.CustomerId;
-- Cannot create indexed view (OUTER JOIN not allowed)

-- ✅ Use INNER JOIN only
CREATE VIEW dbo.vw_OrderWithCustomer
WITH SCHEMABINDING
AS
SELECT 
    o.OrderId,
    o.Total,
    c.FirstName
FROM dbo.Order o
INNER JOIN dbo.Customer c ON o.CustomerId = c.CustomerId;
```

### No Subqueries or Derived Tables

```sql
-- ❌ Not allowed: subqueries
CREATE VIEW dbo.vw_TopCustomers
WITH SCHEMABINDING
AS
SELECT CustomerId, TotalRevenue
FROM (
    SELECT CustomerId, SUM(Total) AS TotalRevenue
    FROM dbo.Order
    GROUP BY CustomerId
) AS CustomerRevenue;
-- Not allowed in indexed views

-- ✅ Use flat query
CREATE VIEW dbo.vw_TopCustomers
WITH SCHEMABINDING
AS
SELECT 
    CustomerId,
    SUM(Total) AS TotalRevenue
FROM dbo.Order
GROUP BY CustomerId;
```

### No Non-Deterministic Functions

```sql
-- ❌ Not allowed: GETDATE(), NEWID(), RAND(), etc.
CREATE VIEW dbo.vw_RecentOrders
WITH SCHEMABINDING
AS
SELECT OrderId, CustomerId, OrderDate
FROM dbo.Order
WHERE OrderDate > GETDATE() - 30;
-- Cannot create index (non-deterministic function)

-- ✅ Use column comparison
CREATE VIEW dbo.vw_ActiveOrders
WITH SCHEMABINDING
AS
SELECT 
    OrderId,
    CustomerId,
    OrderDate,
    Status
FROM dbo.Order
WHERE Status IN ('Pending', 'Processing');
```

## Viewing Indexed Views

```sql
-- Find all indexed views
SELECT 
    OBJECT_SCHEMA_NAME(v.object_id) + '.' + OBJECT_NAME(v.object_id) AS ViewName,
    i.name AS IndexName,
    i.type_desc,
    ps.row_count,
    ps.used_page_count * 8 / 1024 AS SizeMB
FROM sys.views v
INNER JOIN sys.indexes i ON v.object_id = i.object_id
INNER JOIN sys.dm_db_partition_stats ps ON i.object_id = ps.object_id AND i.index_id = ps.index_id
WHERE i.index_id > 0  -- Has index
ORDER BY SizeMB DESC;

-- View index usage
SELECT 
    OBJECT_NAME(s.object_id) AS ViewName,
    i.name AS IndexName,
    s.user_seeks,
    s.user_scans,
    s.user_lookups,
    s.user_updates,
    s.last_user_seek,
    s.last_user_scan
FROM sys.dm_db_index_usage_stats s
INNER JOIN sys.indexes i ON s.object_id = i.object_id AND s.index_id = i.index_id
INNER JOIN sys.views v ON s.object_id = v.object_id
WHERE s.database_id = DB_ID()
ORDER BY s.user_seeks + s.user_scans DESC;
```

## Performance Considerations

```sql
-- Indexed views trade write performance for read performance

-- Writes are slower (must update base table + view)
INSERT INTO dbo.Order (OrderId, CustomerId, OrderDate, Total)
VALUES (1, 100, SYSDATETIME(), 99.99);
-- Updates Order table AND vw_CustomerOrderSummary

-- Reads are faster (precomputed results)
SELECT CustomerId, TotalSpent
FROM dbo.vw_CustomerOrderSummary
WHERE CustomerId = 100;
-- Index seek on materialized view (instant)

-- Best for:
-- - Read-heavy workloads
-- - Expensive queries run frequently
-- - Aggregations with many rows

-- Avoid for:
-- - Write-heavy workloads
-- - Rarely queried data
-- - Simple queries that already have good indexes
```

## Rebuilding Indexed Views

```sql
-- Rebuild to defragment or reclaim space
ALTER INDEX IX_vw_CustomerRevenue_CustomerId 
ON dbo.vw_CustomerRevenue
REBUILD;

-- Rebuild with options
ALTER INDEX IX_vw_CustomerRevenue_CustomerId
ON dbo.vw_CustomerRevenue
REBUILD WITH (
    ONLINE = ON,
    MAXDOP = 4,
    SORT_IN_TEMPDB = ON
);

-- Update statistics
UPDATE STATISTICS dbo.vw_CustomerRevenue;
```

## Why Repeated Expensive Queries Are a Problem

1. **CPU waste**: Recalculating same aggregations repeatedly
2. **I/O overhead**: Scanning large tables every query
3. **Slow response**: Users wait for complex calculations
4. **Blocking**: Long-running queries hold locks
5. **Resource contention**: Many concurrent expensive queries

## Symptoms

- Same complex query in profiler repeatedly
- High CPU from aggregations
- Queries scanning millions of rows
- Response time unacceptable for users
- Denormalization needed but expensive to maintain

## Benefits

- **Instant queries**: Precomputed results indexed
- **Reduced CPU**: Calculate once, query many times
- **Automatic maintenance**: View updated with base tables
- **Transparent**: Optimizer can use automatically (Enterprise Edition)
- **Space efficient**: More efficient than redundant denormalized columns

## Trade-offs

- **Slower writes**: Must update view indexes
- **Storage overhead**: Materialized data consumes space
- **Maintenance complexity**: Must rebuild/reorganize view indexes
- **Restrictions**: Many T-SQL features not allowed
- **Enterprise features**: Auto-matching requires Enterprise Edition

## When to Use

✅ Use indexed views when:
- Complex queries run frequently
- Read-heavy workload
- Aggregations involve many rows
- Joins between large tables
- Calculations expensive to compute repeatedly

❌ Don't use when:
- Write-heavy workload
- Simple queries already fast
- Data changes constantly
- Query patterns vary widely (view won't match)

## See Also

- [Covering Indexes](./covering-indexes.md)
- [Filtered Indexes](./filtered-indexes.md)
- [Execution Plans](./execution-plans.md)
- [Explicit Transactions](./explicit-transactions.md)

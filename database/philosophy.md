# Philosophy

> These patterns sit at the intersection of Database Design, Domain-Driven Design, and Type Safety at the data layer.

This document explains the *why* behind our database patterns. Each pattern draws from principles that ensure data integrity, performance, and security at the database level.

---

## 1. Schema as Contract

The database schema is a contract that enforces business rules at the lowest level. Just as types enforce correctness in application code, database constraints enforce correctness in persisted data.

### Core Concepts

**Constraints Over Code**  
Business rules enforced in the database cannot be bypassed by buggy application code or rogue queries.

```sql
-- Not this: hoping application code validates
CREATE TABLE Orders (
    OrderId INT PRIMARY KEY,
    CustomerId INT,
    Total DECIMAL(18,2)
);

-- This: database enforces the rules
CREATE TABLE Orders (
    OrderId INT PRIMARY KEY,
    CustomerId INT NOT NULL,
    Total DECIMAL(18,2) NOT NULL CHECK (Total >= 0),
    CreatedAt DATETIME2 NOT NULL DEFAULT SYSDATETIME(),
    CONSTRAINT FK_Orders_Customers FOREIGN KEY (CustomerId) 
        REFERENCES Customers(CustomerId)
);
```

**Explicit Nullability**  
Every column should explicitly declare whether `NULL` is allowed. `NOT NULL` is the default assumption unless optionality has business meaning.

**Domain-Specific Types**  
Use appropriate data types that match the domain. Don't use `NVARCHAR(MAX)` for everything or `INT` for monetary values.

### Related Patterns

- [Schema Constraints](./patterns/schema-constraints.md)
- [Explicit Nullability](./patterns/explicit-nullability.md)
- [Domain-Appropriate Types](./patterns/domain-appropriate-types.md)

---

## 2. Immutable Data Patterns

Treat certain tables as append-only ledgers rather than mutable state. This provides audit trails, temporal queries, and eliminates whole classes of concurrency bugs.

### The Golden Rule

**Don't Delete, Mark as Deleted.**

Instead of destroying data, preserve history and mark records as inactive.

```sql
-- ❌ Destructive operation loses history
DELETE FROM Orders WHERE OrderId = 123;

-- ✅ Preserves history
UPDATE Orders 
SET DeletedAt = SYSDATETIME(), DeletedBy = @UserId
WHERE OrderId = 123 AND DeletedAt IS NULL;
```

### Event Sourcing

Store domain events as an immutable log, derive current state through projection.

```sql
-- Events table: append-only
CREATE TABLE OrderEvents (
    EventId BIGINT IDENTITY(1,1) PRIMARY KEY,
    OrderId INT NOT NULL,
    EventType VARCHAR(50) NOT NULL,
    EventData NVARCHAR(MAX) NOT NULL,
    CreatedAt DATETIME2 NOT NULL DEFAULT SYSDATETIME(),
    CreatedBy INT NOT NULL
);

-- Current state: materialized view
CREATE VIEW CurrentOrders AS
SELECT 
    OrderId,
    JSON_VALUE(EventData, '$.Status') AS Status,
    MAX(CreatedAt) AS LastUpdated
FROM OrderEvents
WHERE EventType IN ('OrderCreated', 'OrderStatusChanged')
GROUP BY OrderId;
```

### Related Patterns

- [Soft Deletes](./patterns/soft-deletes.md)
- [Temporal Tables](./patterns/temporal-tables.md)
- [Event Sourcing](./patterns/event-sourcing.md)

---

## 3. Security by Design

Security should be impossible to bypass through database design, not just application-level checks.

### Core Principles

| Principle | Description |
|-----------|-------------|
| **Least Privilege** | Grant only the minimum permissions needed. Application accounts should not have `db_owner`. |
| **Row-Level Security** | Use SQL Server's RLS to enforce data access policies at the database level. |
| **Encrypted Columns** | Use Always Encrypted for sensitive data so even DBAs cannot read it. |
| **Parameterized Queries** | Never concatenate user input into SQL—use parameters to prevent injection. |

```sql
-- ❌ SQL Injection vulnerability
DECLARE @SQL NVARCHAR(MAX) = 'SELECT * FROM Users WHERE Username = ''' + @Username + '''';
EXEC sp_executesql @SQL;

-- ✅ Parameterized query
SELECT * FROM Users WHERE Username = @Username;
```

### Row-Level Security Example

```sql
-- Create security policy
CREATE FUNCTION dbo.fn_SecurityPredicate(@TenantId INT)
RETURNS TABLE
WITH SCHEMABINDING
AS
RETURN SELECT 1 AS Result
WHERE @TenantId = CAST(SESSION_CONTEXT(N'TenantId') AS INT);

CREATE SECURITY POLICY TenantFilter
ADD FILTER PREDICATE dbo.fn_SecurityPredicate(TenantId) ON dbo.Orders,
ADD BLOCK PREDICATE dbo.fn_SecurityPredicate(TenantId) ON dbo.Orders
WITH (STATE = ON);
```

### Related Patterns

- [Row-Level Security](./patterns/row-level-security.md)
- [Always Encrypted](./patterns/always-encrypted.md)
- [SQL Injection Prevention](./patterns/sql-injection-prevention.md)

---

## 4. Performance by Design

Design schemas and queries for performance from the start. Optimization should be built-in, not bolted on.

### Core Concepts

**Index Strategy**  
Every query should have a supporting index. Avoid table scans in OLTP workloads.

**Covering Indexes**  
Include all columns needed by a query in the index to avoid key lookups.

```sql
-- Query that needs optimization
SELECT OrderId, OrderDate, Total 
FROM Orders 
WHERE CustomerId = @CustomerId 
  AND OrderDate >= @StartDate;

-- Covering index for this query
CREATE NONCLUSTERED INDEX IX_Orders_CustomerId_OrderDate
ON Orders(CustomerId, OrderDate)
INCLUDE (Total);
```

**Denormalization for Reads**  
In read-heavy scenarios, duplicate data to avoid expensive joins.

**Partitioning**  
Split large tables by date ranges or other logical boundaries for better query performance and maintenance.

### Related Patterns

- [Covering Indexes](./patterns/covering-indexes.md)
- [Indexed Views](./patterns/indexed-views.md)
- [Table Partitioning](./patterns/table-partitioning.md)
- [Read Models](./patterns/read-models.md)

---

## 5. Explicit Over Implicit

Make intentions clear in your database schema and queries. Future developers (including yourself) should understand the purpose without archaeology.

### Core Principles

| Principle | Instead of... | Use... |
|-----------|---------------|--------|
| **Named Constraints** | System-generated names | Descriptive constraint names |
| **Explicit Transactions** | Auto-commit | `BEGIN TRAN`/`COMMIT`/`ROLLBACK` |
| **Schema-Qualified Names** | `SELECT * FROM Orders` | `SELECT * FROM dbo.Orders` |
| **Column Lists in INSERT** | `INSERT Orders VALUES (...)` | `INSERT Orders (OrderId, ...) VALUES (...)` |

```sql
-- ❌ Implicit, unclear
CREATE TABLE Orders (
    OrderId INT PRIMARY KEY,
    Status INT CHECK (Status IN (1,2,3))
);

-- ✅ Explicit, clear
CREATE TABLE Orders (
    OrderId INT NOT NULL,
    Status VARCHAR(20) NOT NULL,
    CONSTRAINT PK_Orders PRIMARY KEY (OrderId),
    CONSTRAINT CK_Orders_Status CHECK (Status IN ('Pending', 'Shipped', 'Delivered')),
    CONSTRAINT DF_Orders_Status DEFAULT 'Pending' FOR Status
);
```

### Related Patterns

- [Named Constraints](./patterns/named-constraints.md)
- [Explicit Transactions](./patterns/explicit-transactions.md)
- [Schema Qualification](./patterns/schema-qualification.md)

---

## 6. Idempotency and Consistency

Database operations should be idempotent where possible, and always maintain consistency.

### ACID Principles

Every transaction must be:
- **Atomic**: All or nothing
- **Consistent**: Valid state to valid state
- **Isolated**: Concurrent transactions don't interfere
- **Durable**: Committed data survives failures

```sql
-- ❌ Non-idempotent: running twice creates duplicates
INSERT INTO Orders (OrderId, CustomerId, Total)
VALUES (@OrderId, @CustomerId, @Total);

-- ✅ Idempotent: safe to retry
MERGE Orders AS target
USING (SELECT @OrderId AS OrderId, @CustomerId AS CustomerId, @Total AS Total) AS source
ON target.OrderId = source.OrderId
WHEN NOT MATCHED THEN
    INSERT (OrderId, CustomerId, Total)
    VALUES (source.OrderId, source.CustomerId, source.Total);
```

### Related Patterns

- [Idempotent Operations](./patterns/idempotent-operations.md)
- [Optimistic Concurrency](./patterns/optimistic-concurrency.md)
- [Pessimistic Locking](./patterns/pessimistic-locking.md)

---

## 7. Type Safety at the Data Layer

Just as application code benefits from strong typing, database schemas should enforce types that match domain concepts.

### Core Principles

**Use Appropriate Types**  
Don't use `VARCHAR` for dates or `FLOAT` for money.

```sql
-- ❌ Wrong types lose precision and semantics
CREATE TABLE Orders (
    OrderDate VARCHAR(50),      -- "2024-01-01" or "Jan 1" or "01/01/2024"?
    Total FLOAT                 -- Loses precision: 19.99 becomes 19.989999...
);

-- ✅ Correct types enforce meaning
CREATE TABLE Orders (
    OrderDate DATETIME2(0) NOT NULL,           -- Unambiguous, sortable
    Total DECIMAL(18,2) NOT NULL CHECK (Total >= 0)  -- Exact precision
);
```

**User-Defined Types**  
Create custom types for domain concepts used across multiple tables.

```sql
-- Define domain type once
CREATE TYPE dbo.EmailAddress FROM NVARCHAR(255) NOT NULL;
CREATE TYPE dbo.Money FROM DECIMAL(18,2) NOT NULL;

-- Use consistently
CREATE TABLE Customers (
    CustomerId INT PRIMARY KEY,
    Email dbo.EmailAddress,
    CreditLimit dbo.Money
);
```

### Related Patterns

- [Domain-Appropriate Types](./patterns/domain-appropriate-types.md)
- [User-Defined Types](./patterns/user-defined-types.md)
- [Check Constraints](./patterns/check-constraints.md)

---

## 8. Observability and Debugging

Production databases need built-in observability for debugging and performance analysis.

### Core Principles

**Audit Columns**  
Every table should track who created/modified records and when.

```sql
CREATE TABLE Orders (
    OrderId INT PRIMARY KEY,
    CustomerId INT NOT NULL,
    Total DECIMAL(18,2) NOT NULL,
    -- Audit columns
    CreatedAt DATETIME2(0) NOT NULL DEFAULT SYSDATETIME(),
    CreatedBy INT NOT NULL,
    ModifiedAt DATETIME2(0) NOT NULL DEFAULT SYSDATETIME(),
    ModifiedBy INT NOT NULL
);
```

**Query Statistics**  
Use Extended Events or Query Store to track slow queries.

**Correlation IDs**  
Log correlation IDs with database operations to trace requests across system boundaries.

### Related Patterns

- [Audit Columns](./patterns/audit-columns.md)
- [Query Store](./patterns/query-store.md)
- [Extended Events](./patterns/extended-events.md)

---

## Summary

These eight philosophies guide database design:

| Philosophy | Key Question |
|------------|--------------|
| **Schema as Contract** | Does the schema enforce business rules? |
| **Immutable Data** | Should this be append-only? |
| **Security by Design** | Can security be bypassed? |
| **Performance by Design** | Does every query have a supporting index? |
| **Explicit Over Implicit** | Will future developers understand this? |
| **Idempotency** | Is this operation safe to retry? |
| **Type Safety** | Do column types match domain concepts? |
| **Observability** | Can we debug this in production? |

When designing schemas or writing queries, ask these questions. The patterns in this catalog are tools to answer "yes" to each one.

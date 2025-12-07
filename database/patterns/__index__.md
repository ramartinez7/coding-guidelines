# Database Patterns

A catalog of database design patterns and best practices for SQL Server using T-SQL.

## Schema Design

### [Schema Constraints](./schema-constraints.md)

Use database constraints to enforce business rules at the data layer—prevent invalid data from ever being persisted.

### [Explicit Nullability](./explicit-nullability.md)

Every column should explicitly declare whether `NULL` is allowed—make optionality a conscious design decision.

### [Domain-Appropriate Types](./domain-appropriate-types.md)

Use data types that match domain concepts—`DECIMAL` for money, `DATETIME2` for timestamps, not `VARCHAR` for everything.

## Data Integrity Patterns

### [Soft Deletes](./soft-deletes.md)

Mark records as deleted instead of removing them—preserve audit trails and enable recovery.

### [Temporal Tables](./temporal-tables.md)

Use SQL Server's system-versioned temporal tables to automatically track all changes—query data as it existed at any point in time.

### [Audit Columns](./audit-columns.md)

Include standard audit columns in every table—track who created/modified records and when.

## Query Patterns

### [Parameterized Queries](./parameterized-queries.md)

Always use parameters instead of string concatenation—prevent SQL injection and enable query plan reuse.

### [Covering Indexes](./covering-indexes.md)

Create indexes that include all columns needed by a query—eliminate expensive key lookups.

## Security Patterns

### [SQL Injection Prevention](./sql-injection-prevention.md)

Use parameterized queries and stored procedures—never concatenate user input into SQL strings.

### [Row-Level Security](./row-level-security.md)

Enforce data access policies at the database level using SQL Server's RLS—ensure users only see authorized rows.



## Transaction Patterns

### [Explicit Transactions](./explicit-transactions.md)

Wrap related operations in explicit transactions—ensure atomicity of business operations.

### [Idempotent Operations](./idempotent-operations.md)

Design operations to be safely repeatable—use `MERGE` or conditional logic to prevent duplicate effects.

### [Optimistic Concurrency](./optimistic-concurrency.md)

Use row versions or timestamps to detect conflicts—allow concurrent reads while preventing lost updates.

## Event-Driven Patterns

### [Event Sourcing](./event-sourcing.md)

Store domain events as an immutable log—derive current state through projection and enable time-travel queries.

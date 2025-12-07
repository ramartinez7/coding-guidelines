# Database Patterns

A catalog of database design patterns and best practices for SQL Server using T-SQL.

## Schema Design

### [Schema Constraints](./schema-constraints.md)

Use database constraints to enforce business rules at the data layer—prevent invalid data from ever being persisted.

### [Check Constraints](./check-constraints.md)

Use CHECK constraints to enforce domain rules and value ranges—prevent invalid data at the database level.

### [Foreign Key Constraints](./foreign-key-constraints.md)

Enforce referential integrity between tables—prevent orphaned records and maintain data consistency.

### [Unique Constraints](./unique-constraints.md)

Enforce uniqueness of values beyond the primary key—prevent duplicate data and model natural keys.

### [Named Constraints](./named-constraints.md)

Use explicit, descriptive names for all constraints—make schema self-documenting and errors understandable.

### [Explicit Nullability](./explicit-nullability.md)

Every column should explicitly declare whether `NULL` is allowed—make optionality a conscious design decision.

### [Domain-Appropriate Types](./domain-appropriate-types.md)

Use data types that match domain concepts—`DECIMAL` for money, `DATETIME2` for timestamps, not `VARCHAR` for everything.

### [User-Defined Types](./user-defined-types.md)

Create custom types for domain concepts used across multiple tables—ensure consistency and reduce duplication.

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

### [Execution Plans](./execution-plans.md)

Analyze query execution plans to understand performance—identify missing indexes, expensive operations, and optimization opportunities.

## Indexing Patterns

### [Covering Indexes](./covering-indexes.md)

Create indexes that include all columns needed by a query—eliminate expensive key lookups.

### [Filtered Indexes](./filtered-indexes.md)

Create indexes on subsets of data—improve query performance while reducing index maintenance overhead and storage.

### [Indexed Views](./indexed-views.md)

Materialize complex query results with clustered indexes—precompute aggregations and joins for fast read performance.

## Security Patterns

### [SQL Injection Prevention](./sql-injection-prevention.md)

Use parameterized queries and stored procedures—never concatenate user input into SQL strings.

### [Row-Level Security](./row-level-security.md)

Enforce data access policies at the database level using SQL Server's RLS—ensure users only see authorized rows.

### [Always Encrypted](./always-encrypted.md)

Encrypt sensitive columns so data remains encrypted at rest and in transit—even DBAs cannot read plaintext values.

### [Least Privilege](./least-privilege.md)

Grant only the minimum permissions needed—reduce blast radius of compromised accounts and prevent unauthorized access.



## Transaction Patterns

### [Explicit Transactions](./explicit-transactions.md)

Wrap related operations in explicit transactions—ensure atomicity of business operations.

### [Idempotent Operations](./idempotent-operations.md)

Design operations to be safely repeatable—use `MERGE` or conditional logic to prevent duplicate effects.

### [Optimistic Concurrency](./optimistic-concurrency.md)

Use row versions or timestamps to detect conflicts—allow concurrent reads while preventing lost updates.

### [Pessimistic Locking](./pessimistic-locking.md)

Lock rows during read to prevent concurrent modifications—guarantee exclusive access during updates.

## Event-Driven Patterns

### [Event Sourcing](./event-sourcing.md)

Store domain events as an immutable log—derive current state through projection and enable time-travel queries.

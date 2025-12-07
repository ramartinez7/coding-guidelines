# Database Patterns

A catalog of database design patterns and best practices for SQL Server using T-SQL.

## Schema Design

### [Schema Constraints](./schema-constraints.md)

Use database constraints to enforce business rules at the data layer—prevent invalid data from ever being persisted.

### [Explicit Nullability](./explicit-nullability.md)

Every column should explicitly declare whether `NULL` is allowed—make optionality a conscious design decision.

### [Domain-Appropriate Types](./domain-appropriate-types.md)

Use data types that match domain concepts—`DECIMAL` for money, `DATETIME2` for timestamps, not `VARCHAR` for everything.

### [Named Constraints](./named-constraints.md)

Give constraints meaningful names instead of accepting system-generated names—improves maintainability and error messages.

### [User-Defined Types](./user-defined-types.md)

Create reusable domain types for concepts used across multiple tables—ensures consistency and expresses intent.

### [Check Constraints](./check-constraints.md)

Enforce value ranges and business rules at the column level—prevent invalid data at insertion time.

## Data Integrity Patterns

### [Foreign Key Constraints](./foreign-key-constraints.md)

Enforce referential integrity between tables—prevent orphaned records and maintain data consistency.

### [Unique Constraints](./unique-constraints.md)

Guarantee uniqueness of values beyond the primary key—enforce business rules like "one email per customer."

### [Soft Deletes](./soft-deletes.md)

Mark records as deleted instead of removing them—preserve audit trails and enable recovery.

### [Temporal Tables](./temporal-tables.md)

Use SQL Server's system-versioned temporal tables to automatically track all changes—query data as it existed at any point in time.

### [Audit Columns](./audit-columns.md)

Include standard audit columns in every table—track who created/modified records and when.

## Query Patterns

### [Schema Qualification](./schema-qualification.md)

Always prefix table names with schema (`dbo.Orders`)—improves performance and prevents ambiguity.

### [Explicit Column Lists](./explicit-column-lists.md)

Never use `SELECT *` in production code—list columns explicitly for clarity and stability.

### [Parameterized Queries](./parameterized-queries.md)

Always use parameters instead of string concatenation—prevent SQL injection attacks.

### [Covering Indexes](./covering-indexes.md)

Create indexes that include all columns needed by a query—eliminate expensive key lookups.

### [Indexed Views](./indexed-views.md)

Materialize frequently-queried aggregations as indexed views—pre-compute expensive joins and calculations.

### [Query Store](./query-store.md)

Enable Query Store to track query performance over time—identify regressions and optimize problem queries.

## Security Patterns

### [SQL Injection Prevention](./sql-injection-prevention.md)

Use parameterized queries and stored procedures—never concatenate user input into SQL strings.

### [Row-Level Security](./row-level-security.md)

Enforce data access policies at the database level using SQL Server's RLS—ensure users only see authorized rows.

### [Always Encrypted](./always-encrypted.md)

Encrypt sensitive columns at rest and in transit—protect data even from DBAs and infrastructure admins.

### [Least Privilege](./least-privilege.md)

Grant minimum necessary permissions to application accounts—avoid `db_owner`, use specific permissions.

### [Dynamic Data Masking](./dynamic-data-masking.md)

Mask sensitive data for non-privileged users—show `XXX-XX-1234` instead of full SSN without changing application code.

## Transaction Patterns

### [Explicit Transactions](./explicit-transactions.md)

Wrap related operations in explicit transactions—ensure atomicity of business operations.

### [Idempotent Operations](./idempotent-operations.md)

Design operations to be safely repeatable—use `MERGE` or conditional logic to prevent duplicate effects.

### [Optimistic Concurrency](./optimistic-concurrency.md)

Use row versions or timestamps to detect conflicts—allow concurrent reads while preventing lost updates.

### [Pessimistic Locking](./pessimistic-locking.md)

Use lock hints when you need exclusive access—prevent concurrent modifications during critical operations.

## Performance Patterns

### [Table Partitioning](./table-partitioning.md)

Split large tables into smaller physical partitions—improve query performance and maintenance operations.

### [Filtered Indexes](./filtered-indexes.md)

Create indexes on subsets of data—reduce index size and improve query performance for common filters.

### [Columnstore Indexes](./columnstore-indexes.md)

Use columnstore indexes for analytical queries on large datasets—achieve massive compression and query speedups.

### [Read Models](./read-models.md)

Denormalize data into read-optimized tables or views—separate read and write models for better performance.

### [Batch Operations](./batch-operations.md)

Process multiple rows in a single statement—reduce round trips and improve throughput.

## Event-Driven Patterns

### [Event Sourcing](./event-sourcing.md)

Store domain events as an immutable log—derive current state through projection and enable time-travel queries.

### [Change Data Capture](./change-data-capture.md)

Automatically track changes to tables using SQL Server's CDC—propagate changes to other systems.

### [Service Broker](./service-broker.md)

Use SQL Server's built-in message queuing for reliable async processing—decouple producers and consumers.

## Observability Patterns

### [Extended Events](./extended-events.md)

Use lightweight event tracing to monitor database activity—diagnose performance issues without significant overhead.

### [Execution Plans](./execution-plans.md)

Analyze query execution plans to understand performance—identify missing indexes and inefficient operations.

### [Statistics Maintenance](./statistics-maintenance.md)

Keep statistics up-to-date for optimal query plans—schedule regular updates or enable auto-update.

## Migration Patterns

### [Schema Versioning](./schema-versioning.md)

Track database schema versions—enable safe, repeatable migrations across environments.

### [Blue-Green Deployments](./blue-green-deployments.md)

Use views and synonyms to enable zero-downtime schema changes—redirect traffic without application changes.

### [Backward-Compatible Changes](./backward-compatible-changes.md)

Make schema changes that don't break existing code—add columns with defaults, deprecate before removing.

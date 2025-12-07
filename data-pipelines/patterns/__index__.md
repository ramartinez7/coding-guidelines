# Data Pipeline Patterns

A catalog of best practices for building reliable, scalable data pipelines. These patterns are technology-agnostic and apply to batch processing, streaming systems, and hybrid architectures.

## Reliability Patterns

### [Idempotent Processing](./idempotent-processing.md)

Design pipeline stages to be safely repeatable—same input produces same output, retries don't create duplicates.

### [Exactly-Once Semantics](./exactly-once-semantics.md)

Guarantee each record is processed exactly once—neither duplicated nor lost—using transactional commits and deduplication.

### [Checkpointing and Recovery](./checkpointing-recovery.md)

Save progress periodically to resume from last known state after failures—avoid reprocessing from the beginning.

### [Dead Letter Queues](./dead-letter-queues.md)

Move unparseable or invalid records to a separate queue for inspection—don't block valid data with bad data.

## Data Quality Patterns

### [Data Validation at Boundaries](./data-validation-boundaries.md)

Validate and parse data at ingestion points—transform raw input into trusted types before downstream processing.

### [Schema Evolution](./schema-evolution.md)

Handle schema changes gracefully with versioning and compatibility checks—add fields without breaking existing pipelines.

### [Data Quality Monitoring](./data-quality-monitoring.md)

Continuously monitor data quality metrics—completeness, accuracy, consistency, timeliness—and alert on anomalies.

### [Type-Safe Transformations](./type-safe-transformations.md)

Use strongly-typed data structures throughout the pipeline—catch errors at compile time, not runtime.

## Observability Patterns

### [Data Lineage Tracking](./data-lineage-tracking.md)

Track data provenance from source through all transformations to destination—enable debugging and compliance auditing.

### [Observability Patterns](./observability-patterns.md)

Instrument pipelines with metrics, logs, and traces—make every stage inspectable and debuggable.

### [Correlation IDs](./correlation-ids.md)

Propagate unique identifiers through all pipeline stages—trace individual records across distributed systems.

## Performance Patterns

### [Backpressure Handling](./backpressure-handling.md)

Handle mismatched throughput between producers and consumers—use bounded buffers and flow control to prevent overload.

### [Partitioning Strategies](./partitioning-strategies.md)

Split data by key or range to enable parallel processing—balance load and maintain ordering within partitions.

### [Parallel Processing](./parallel-processing.md)

Design stateless stages that can run concurrently—scale horizontally by adding more workers.

### [Incremental Processing](./incremental-processing.md)

Process only new or changed data—use watermarks, timestamps, or change data capture to avoid full scans.

## Architecture Patterns

### [Batch vs Streaming](./batch-vs-streaming.md)

Choose between batch (bounded, scheduled) and streaming (unbounded, continuous) based on latency and throughput requirements.

### [Lambda Architecture](./lambda-architecture.md)

Combine batch and streaming layers for both completeness and low latency—batch reprocesses history, streaming handles real-time.

### [Kappa Architecture](./kappa-architecture.md)

Unify batch and streaming with a single stream processing system—treat batch as replaying historical event streams.

### [State Management](./state-management.md)

Manage stateful computations in distributed pipelines—use partitioned state stores with checkpointing for fault tolerance.

## Integration Patterns

### [Change Data Capture (CDC)](./change-data-capture.md)

Capture database changes as events without custom triggers—use transaction logs to propagate updates downstream.

### [API Polling vs Webhooks](./api-polling-webhooks.md)

Choose between polling APIs on a schedule or receiving push notifications—trade-offs in latency, complexity, and reliability.

### [Fan-Out and Fan-In](./fan-out-fan-in.md)

Distribute data to multiple destinations (fan-out) or aggregate from multiple sources (fan-in)—manage dependencies and partial failures.

## Windowing Patterns

### [Watermarks and Windowing](./watermarks-windowing.md)

Process out-of-order event streams with time windows—use watermarks to track progress and trigger computations.

### [Session Windows](./session-windows.md)

Group events by activity sessions with gaps—handle variable-length windows based on user behavior.

### [Tumbling vs Sliding Windows](./tumbling-sliding-windows.md)

Choose between non-overlapping tumbling windows (hourly aggregates) or overlapping sliding windows (rolling averages).

## Security Patterns

### [Encryption Patterns](./encryption-patterns.md)

Encrypt sensitive data in transit and at rest—use TLS for network communication and encrypted storage for persistence.

### [Audit Logging](./audit-logging.md)

Log all data access and transformations for compliance and debugging—include who, what, when, where, and why.

### [PII Handling](./pii-handling.md)

Identify and protect personally identifiable information—apply tokenization, encryption, or anonymization based on requirements.

### [Secrets Management](./secrets-management.md)

Never hardcode credentials—use secret management systems to inject credentials at runtime with automatic rotation.

## Cost Optimization Patterns

### [Storage Optimization](./storage-optimization.md)

Use columnar formats (Parquet, ORC) for analytics—compress data and enable predicate pushdown for efficient queries.

### [Data Retention Policies](./data-retention.md)

Define and enforce retention periods—archive or delete old data to reduce storage costs and meet compliance requirements.

### [Resource Right-Sizing](./resource-right-sizing.md)

Match compute and storage resources to workload requirements—avoid over-provisioning waste and under-provisioning failures.

## Testing Patterns

### [Pipeline Testing Strategies](./pipeline-testing.md)

Test pipelines with unit tests (transformations), integration tests (stages), and end-to-end tests (full pipelines).

### [Data Quality Tests](./data-quality-tests.md)

Assert expectations on data—schema validation, null checks, range checks, referential integrity, and statistical distributions.

### [Chaos Testing](./chaos-testing.md)

Inject failures deliberately to validate resilience—test recovery from node crashes, network partitions, and data corruption.

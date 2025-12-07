# Distributed Systems Patterns

A catalog of patterns for building resilient, scalable distributed systems.

## Foundational Concepts

### [CAP Theorem](./cap-theorem.md)

Consistency, Availability, Partition Tolerance—understand the fundamental trade-offs in distributed data.

### [Eventual Consistency](./eventual-consistency.md)

Accept temporary inconsistency across nodes for better availability and performance—eventually all nodes converge.

<!-- ### [Time and Ordering](./time-and-ordering.md)

Wall-clock time is unreliable in distributed systems—use logical clocks and causal ordering for correctness. -->

## Data Consistency Patterns

### [Two-Phase Commit (2PC)](./two-phase-commit.md)

Distributed transactions with a coordinator that ensures all participants commit or abort atomically.

### [Saga Pattern](./saga-pattern.md)

Long-running transactions broken into local transactions with compensating actions for rollback.

### [Event Sourcing](./event-sourcing.md)

Store state as an append-only log of events rather than mutable records—enables time-travel debugging and audit trails.

### [CQRS (Command Query Responsibility Segregation)](./cqrs.md)

Separate read and write models to optimize each independently—often combined with event sourcing.

## Resilience Patterns

### [Circuit Breaker](./circuit-breaker.md)

Prevent cascading failures by failing fast when a downstream service is unhealthy.

<!-- ### [Bulkhead Pattern](./bulkhead.md)

Isolate resources to prevent failure in one component from exhausting shared resources.

### [Retry with Exponential Backoff](./retry-exponential-backoff.md)

Retry failed operations with increasing delays to avoid overwhelming recovering services.

### [Timeout Pattern](./timeout-pattern.md)

Set explicit time limits for operations to prevent indefinite waiting and resource exhaustion. -->

## Communication Patterns

### [Event-Driven Architecture](./event-driven-architecture.md)

Services communicate through events published to a message bus—loose coupling and asynchrony.

<!-- ### [Message Queues](./message-queues.md)

Asynchronous communication through durable queues that buffer messages and enable retry.

### [Request-Response vs. Fire-and-Forget](./request-response-vs-fire-forget.md)

Choose between synchronous request-response and asynchronous fire-and-forget based on requirements. -->

## Coordination Patterns

<!-- ### [Leader Election](./leader-election.md)

Select a single node to coordinate actions across a cluster—prevents split-brain scenarios.

### [Distributed Locks](./distributed-locks.md)

Coordinate access to shared resources across multiple processes using consensus-based locks.

### [Consensus Algorithms](./consensus-algorithms.md)

Achieve agreement among distributed nodes—Raft, Paxos, and their practical applications. -->

## Data Patterns

<!-- ### [Database per Service](./database-per-service.md)

Each microservice owns its database—eliminates shared database coupling but complicates queries.

### [Shared Database Anti-Pattern](./shared-database-anti-pattern.md)

Multiple services accessing the same database couples services and prevents independent evolution.

### [Materialized Views](./materialized-views.md)

Pre-compute and store query results to avoid expensive joins across service boundaries.

### [Change Data Capture (CDC)](./change-data-capture.md)

Capture database changes as events to propagate updates without polling or dual writes. -->

## Observability Patterns

<!-- Note: These patterns reference the C# patterns section where applicable -->

<!-- ### [Distributed Tracing](./distributed-tracing.md)

Trace request flow through multiple services to identify bottlenecks and failures—OpenTelemetry, Zipkin, Jaeger.

### [Health Checks](./health-checks.md)

Expose endpoints that report service health—enables load balancers and orchestrators to route traffic appropriately.

### [Centralized Logging](./centralized-logging.md)

Aggregate logs from all services into a single system for cross-service analysis and debugging. -->

## Scalability Patterns

<!-- ### [Sharding](./sharding.md)

Partition data across multiple databases to distribute load—horizontal scaling for writes.

### [Read Replicas](./read-replicas.md)

Separate read traffic from write traffic using database replicas—scales read-heavy workloads.

### [Load Balancing](./load-balancing.md)

Distribute requests across multiple instances—round-robin, least connections, consistent hashing. -->

## Deployment Patterns

<!-- ### [Blue-Green Deployment](./blue-green-deployment.md)

Run two identical production environments and switch traffic between them for zero-downtime deployments.

### [Canary Deployment](./canary-deployment.md)

Gradually roll out changes to a subset of users before full deployment—detect issues early.

### [Feature Flags](./feature-flags.md)

Toggle features on/off without deploying code—enables trunk-based development and gradual rollouts. -->

## Security Patterns

<!-- ### [API Gateway](./api-gateway.md)

Single entry point for all client requests—handles authentication, rate limiting, and routing.

### [Service Mesh](./service-mesh.md)

Infrastructure layer for service-to-service communication—observability, security, and traffic management.

### [Zero Trust Architecture](./zero-trust-architecture.md)

Never trust, always verify—authenticate and authorize every request regardless of network location. -->

## Testing Patterns

<!-- ### [Contract Testing](./contract-testing.md)

Verify service APIs match consumer expectations—prevents integration failures.

### [Chaos Engineering](./chaos-engineering.md)

Deliberately inject failures to test system resilience—Netflix's Chaos Monkey and similar tools.

### [Synthetic Monitoring](./synthetic-monitoring.md)

Continuously test production systems with automated requests—detect issues before users do. -->

## Legacy Integration Patterns

<!-- ### [Anti-Corruption Layer](./anti-corruption-layer.md)

Translate between legacy systems and modern services without polluting domain models.

### [Strangler Fig Pattern](./strangler-fig-pattern.md)

Gradually replace legacy systems by intercepting and migrating functionality piece by piece. -->

# Database Performance

## 1. Purpose

This document defines the performance standards for database access, query execution, indexing, data retrieval, connection management, transactions, and large-scale data processing across KAMPYN.

KAMPYN serves multiple universities, tenants, students, faculty members, administrators, vendors, and operational teams. As the platform grows, database performance directly affects API latency, user experience, resource consumption, operational costs, and system reliability.

Every database interaction must be designed to minimize unnecessary I/O, avoid inefficient query patterns, control resource consumption, and maintain predictable performance under concurrent workloads.

This policy applies to PostgreSQL, MongoDB, Redis, OpenSearch, and other data infrastructure integrated into KAMPYN, while recognizing that each technology has distinct performance characteristics and responsibilities.

## 2. Core Principles

All database-related implementations MUST follow these principles:

- **Query efficiency:** Retrieve only the data required by the operation.
- **Index-driven access:** Use appropriate indexes for frequently executed queries and validate their effectiveness.
- **Bounded retrieval:** Avoid unbounded queries and enforce pagination, limits, or streaming.
- **Tenant isolation:** Include validated tenant scope in every tenant-owned data access path.
- **Connection efficiency:** Reuse managed database connections and prevent connection exhaustion.
- **Transaction discipline:** Keep transactions short and avoid unnecessary locks.
- **Concurrency safety:** Preserve correctness under simultaneous reads and writes.
- **Measured optimization:** Use query plans, profiling, and benchmarks to identify actual bottlenecks.
- **Predictable resource usage:** Bound query complexity, memory consumption, connection usage, and execution time.
- **Data integrity:** Never sacrifice correctness, authorization, or transactional guarantees for performance.
- **Workload-aware design:** Optimize based on actual read/write patterns, data distribution, and expected scale.
- **Operational resilience:** Design for database latency, partial failures, overload, and recovery.

## 3. Database Responsibilities

Each datastore must serve a clearly defined purpose.

| Datastore | Primary responsibility | Performance focus |
|---|---|---|
| PostgreSQL | Relational and transactional source of truth | Query plans, indexes, joins, transactions, connection pooling |
| MongoDB | Document-oriented workloads | Document design, indexes, projections, aggregation pipelines |
| Redis | Caching and coordination | Low-latency access, bounded memory, efficient structures |
| OpenSearch | Search and derived indexing | Query efficiency, mappings, shards, indexing throughput |
| Object storage | Persistent binary data | Efficient upload/download, streaming, metadata retrieval |

Do not use multiple datastores for the same workload without a documented architectural reason.

### 3.1 Source of Truth

- PostgreSQL is the primary source of truth for relational and transactional data.
- MongoDB is authoritative for explicitly assigned document-oriented workloads.
- Redis must not be treated as the permanent source of truth for business records.
- OpenSearch is a derived index and must not be relied upon for transactional correctness.
- Object storage owns binary objects, while database records may store their metadata and references.

Queries that determine critical business outcomes must use the appropriate authoritative datastore.

## 4. Query Design Standards

### 4.1 Retrieve Only Required Fields

Avoid retrieving complete records when an operation needs only a subset of fields.

Inefficient:

```sql
SELECT *
FROM orders
WHERE tenant_id = $1
  AND user_id = $2;
```

Preferred:

```sql
SELECT id, status, total_amount, created_at
FROM orders
WHERE tenant_id = $1
  AND user_id = $2;
```

Requirements:

- Select only the fields required by the consuming operation.
- Avoid returning large text, JSON, or binary fields unnecessarily.
- Use projections in MongoDB.
- Avoid exposing internal database fields through API responses.
- Keep result payloads bounded.

### 4.2 Filter at the Database Layer

Apply selective filters in the database rather than retrieving large datasets and filtering them in application memory.

Inefficient:

```ts
const orders = await orderRepository.findAll({
  tenantId,
});

const userOrders = orders.filter(
  (order) => order.userId === userId
);
```

Preferred:

```ts
const userOrders = await orderRepository.findMany({
  tenantId,
  userId,
});
```

The repository query must enforce tenant scope and use suitable indexes for its filters.

### 4.3 Avoid Unnecessary Queries

- Combine compatible retrieval operations when it reduces round trips.
- Reuse results within a request when their validity is unchanged.
- Use batch retrieval for related records.
- Avoid querying the same record repeatedly in a single workflow.
- Avoid unnecessary count queries when the API contract does not require a total count.
- Avoid fetching data that is immediately discarded.

Do not combine unrelated operations into excessively complex queries merely to reduce the number of database calls.

### 4.4 Query Complexity

Analyze both application-level and database-level complexity.

Consider:

- Number of records scanned.
- Index selectivity.
- Join cardinality.
- Sort and aggregation cost.
- Data volume transferred.
- Memory used by intermediate results.
- Lock contention.
- Network round trips.
- Concurrent query volume.

Big-O analysis alone is insufficient to predict database performance. Validate important queries using actual execution plans and representative datasets.

## 5. Indexing Strategy

Indexes are essential for efficient retrieval but introduce write, storage, and maintenance overhead.

### 5.1 Index Design

Indexes should be based on real query patterns.

Before adding an index, identify:

- The query or workload it supports.
- Filter and join fields.
- Sorting requirements.
- Expected selectivity.
- Data volume and growth.
- Write frequency.
- Index storage cost.
- Potential overlap with existing indexes.

### 5.2 Indexing Requirements

- Index frequently used selective filters where justified.
- Index foreign-key and join fields when query patterns require them.
- Use composite indexes for common multi-field access patterns.
- Match composite index field order to the workload.
- Use partial or filtered indexes when supported and appropriate.
- Avoid redundant indexes.
- Avoid indexing every field by default.
- Review unused and low-value indexes periodically.
- Validate index usage with query plans.

### 5.3 Tenant-Aware Indexing

Tenant-scoped data access is fundamental to KAMPYN.

Where tenant filtering is part of the query, evaluate indexes that include `tenant_id` alongside the relevant filtering, sorting, or joining fields.

Example:

```sql
CREATE INDEX idx_orders_tenant_user_created
ON orders (
  tenant_id,
  user_id,
  created_at DESC
);
```

The exact index order must be determined by the actual query patterns, data distribution, and selectivity.

Tenant scope must be enforced by query logic and authorization, not by the existence of an index.

### 5.4 Composite Indexes

A composite index may support multiple query patterns when the leftmost prefix and ordering requirements align.

Avoid creating wide composite indexes without evidence. Larger indexes consume storage, increase write amplification, and can reduce cache efficiency.

### 5.5 Index Maintenance

- Monitor index size and usage.
- Identify redundant and unused indexes.
- Evaluate index bloat and maintenance requirements.
- Schedule expensive index operations appropriately.
- Use safe migration strategies for production indexes.
- Test index changes with representative query plans.

Refer to `.ai/data/indexing.md` for detailed indexing policies.

## 6. PostgreSQL Performance

PostgreSQL is KAMPYN's primary relational and transactional datastore.

### 6.1 Query Optimization

- Use selective `WHERE` clauses.
- Avoid unnecessary sequential scans on large tables.
- Use appropriate joins and indexes.
- Retrieve only required columns.
- Avoid unnecessary casts and transformations on indexed filter fields.
- Avoid repeated correlated subqueries when a join or aggregation is more efficient.
- Review sorting and grouping costs.
- Use database-side aggregation where suitable.
- Keep frequently executed queries predictable.

A sequential scan is not inherently inefficient. For small tables or queries returning a large proportion of rows, it may be the appropriate plan.

### 6.2 Execution Plans

Use `EXPLAIN` and, where safe, `EXPLAIN ANALYZE` to investigate expensive queries.

Review:

- Scan type.
- Estimated versus actual row counts.
- Join algorithms.
- Sort operations.
- Filter selectivity.
- Buffer reads and hits.
- Execution time.
- Temporary file usage.
- Rows removed by filters.

Do not execute potentially expensive analysis on production workloads without understanding its impact.

### 6.3 Joins

- Join on compatible, appropriately indexed fields.
- Avoid accidental Cartesian products.
- Limit rows before expensive downstream processing where semantics allow.
- Review join cardinality and actual query plans.
- Avoid fetching excessive related records.
- Use batch retrieval or carefully scoped eager loading when appropriate.

### 6.4 Aggregations

For expensive aggregation workloads:

- Filter early where semantically valid.
- Use indexes that support common filters.
- Avoid unnecessarily wide grouping keys.
- Consider materialized views for repeated expensive computations.
- Precompute stable aggregates when freshness requirements permit.
- Use asynchronous processing for large reports.
- Avoid transferring raw datasets to application memory solely for aggregation.

### 6.5 PostgreSQL Configuration

Database configuration must be based on available resources and measured workload.

Review:

- Connection limits.
- Shared buffers.
- Work memory.
- Effective cache size.
- WAL configuration.
- Checkpoint behavior.
- Autovacuum settings.
- Replication and recovery requirements.
- Statement timeouts.

Do not copy tuning values from unrelated environments without benchmarking and validating the effects.

### 6.6 Vacuum and Statistics

PostgreSQL performance depends on healthy statistics and table maintenance.

- Monitor autovacuum activity.
- Track dead tuples and table bloat.
- Ensure statistics remain representative.
- Investigate slow query plans caused by poor estimates.
- Review vacuum and analyze activity.
- Avoid disabling routine maintenance without a documented alternative.

### 6.7 Partitioning

Partitioning may be considered when large tables have clear partition keys and queries can benefit from partition pruning.

Potential use cases include:

- High-volume event records.
- Large order histories.
- Audit logs.
- Time-series operational data.

Partitioning must be justified by measured query or maintenance requirements. It introduces additional schema, indexing, migration, and operational complexity and is not a substitute for appropriate query design.

## 7. MongoDB Performance

MongoDB is used for justified document-oriented workloads.

### 7.1 Document Design

- Model documents around access patterns.
- Avoid unnecessarily large documents.
- Avoid deeply nested structures that complicate frequent updates.
- Embed data when it is read and updated together and its growth is bounded.
- Reference data when independent access, updates, or unbounded growth justify it.
- Avoid unbounded arrays.
- Avoid excessive duplication of mutable information.
- Establish document size and growth expectations.

Document modeling must balance retrieval efficiency, update costs, consistency, and storage.

### 7.2 Query Efficiency

- Use selective filters.
- Apply projections.
- Set limits for list queries.
- Use appropriate indexes.
- Avoid unbounded collection scans.
- Review aggregation pipeline stages.
- Avoid transferring large documents when only a few fields are needed.
- Use bulk operations for suitable high-volume writes.

### 7.3 Aggregation Pipelines

- Filter early when it preserves semantics.
- Use indexed match conditions where possible.
- Project only required fields.
- Avoid expensive `$lookup` operations on high-volume workloads without measurement.
- Avoid unbounded `$group` operations.
- Review sort stages and memory requirements.
- Use `allowDiskUse` only when appropriate and operationally understood.
- Validate pipelines using representative data and explain output.

### 7.4 Indexes

- Create indexes based on frequent filters, sorting, and lookup patterns.
- Evaluate compound index order.
- Use partial indexes where suitable.
- Avoid excessive multikey indexes.
- Monitor index size and write amplification.
- Review index use through query statistics and explain plans.

### 7.5 Connection Management

- Reuse managed MongoDB clients.
- Configure bounded connection pools.
- Set appropriate server-selection and operation timeouts.
- Avoid creating a new database client for each request.
- Monitor connection-pool saturation and wait times.
- Keep connection settings appropriate for each deployment.

### 7.6 Sharding and Multi-Cluster Workloads

When KAMPYN uses multiple MongoDB clusters or sharded deployments:

- Define data ownership and routing clearly.
- Select shard keys based on actual query patterns and distribution.
- Avoid hot partitions and uneven workloads.
- Minimize cross-shard operations.
- Avoid scatter-gather queries where targeted routing is possible.
- Monitor cluster-specific utilization and latency.
- Ensure tenant scope is enforced consistently across clusters.
- Define operational recovery and rebalancing procedures.

Multiple clusters do not automatically improve performance. Data partitioning, request routing, consistency, and operational overhead must be measured.

Refer to `.ai/data/mongodb.md` for detailed MongoDB standards.

## 8. Redis Performance

Redis should be used for workloads that benefit from fast in-memory access, caching, or coordination.

### 8.1 Efficient Access

- Select appropriate Redis data structures.
- Use atomic commands where possible.
- Avoid retrieving entire large collections when only a subset is needed.
- Use pipelining for independent operations where it reduces round trips.
- Avoid excessive small commands when batching is safe.
- Keep payloads appropriately sized.
- Set explicit TTLs for transient data.

### 8.2 Memory Efficiency

- Define memory limits and eviction policies.
- Monitor memory consumption and fragmentation.
- Avoid unbounded keys and collections.
- Avoid storing duplicate representations unnecessarily.
- Use bounded cache values and explicit retention.
- Separate workloads with conflicting durability or eviction needs where justified.

### 8.3 Redis Bottlenecks

Monitor:

- Command latency.
- CPU utilization.
- Memory pressure.
- Evictions.
- Key cardinality.
- Large values.
- Connection count.
- Network throughput.
- Slow commands.
- Cache hit and miss ratios.

Avoid expensive Redis commands on large collections in latency-sensitive request paths.

Refer to `.ai/data/redis.md` and `.ai/performance/cache-strategy.md` for the complete Redis and caching policies.

## 9. OpenSearch Performance

OpenSearch is KAMPYN's derived search and discovery system.

### 9.1 Search Query Design

- Use mappings that reflect actual search and filter requirements.
- Avoid retrieving unnecessary fields.
- Set explicit result limits.
- Use appropriate pagination strategies.
- Avoid unrestricted wildcard and expensive regex queries.
- Restrict complex aggregations.
- Filter by validated tenant scope.
- Monitor slow queries and resource-intensive search patterns.

### 9.2 Index Design

- Avoid indexing fields that are never searched, filtered, sorted, or aggregated.
- Use suitable field types and analyzers.
- Avoid excessive mapping complexity.
- Keep documents appropriately sized.
- Minimize unnecessary nested fields.
- Define lifecycle and retention policies for large derived indexes.

### 9.3 Shards and Replicas

Shard and replica counts must be determined based on:

- Index size.
- Query throughput.
- Indexing throughput.
- Cluster resources.
- Availability requirements.
- Tenant distribution.
- Recovery time objectives.

Excessive shard counts introduce coordination and memory overhead. Too few shards can constrain parallelism or complicate scaling.

### 9.4 Search Consistency

OpenSearch indexing may be asynchronous.

- Document indexing delay.
- Reconcile derived data with authoritative records.
- Reindex when mappings or transformation logic change.
- Avoid relying on search results for final business validation.
- Use authoritative database reads for critical transactional operations.

## 10. Connection Pooling

Connection management is critical for preventing resource exhaustion and reducing connection setup overhead.

### 10.1 Requirements

- Use managed connection pools.
- Reuse database clients across requests.
- Configure maximum and minimum pool sizes where supported.
- Set connection acquisition timeouts.
- Set idle connection policies.
- Monitor pool utilization and wait times.
- Avoid unbounded connection creation.
- Close connections and clients gracefully during shutdown.
- Account for all application instances when calculating total connections.

### 10.2 Pool Sizing

Pool sizes must be based on:

- Database connection capacity.
- Number of application instances.
- Expected concurrent database operations.
- Query execution time.
- Background worker concurrency.
- Administrative and monitoring connections.
- Deployment scaling behavior.

The sum of connections across all application instances must remain within the database's practical connection budget.

Increasing the pool size does not necessarily improve throughput. Excessive concurrency can increase lock contention, memory consumption, and database scheduling overhead.

### 10.3 Connection Saturation

When a pool is saturated:

- Identify long-running or blocked queries.
- Review transaction duration.
- Check for connection leaks.
- Examine concurrency levels.
- Review pool acquisition timeouts.
- Avoid unbounded retries.
- Scale or tune only after identifying the bottleneck.

## 11. Transaction Performance

Transactions preserve correctness but can reduce throughput when held open unnecessarily.

### 11.1 Transaction Boundaries

- Keep transactions as short as practical.
- Perform validation and independent computation before opening a transaction where safe.
- Avoid network calls inside database transactions.
- Avoid waiting for user interaction inside transactions.
- Minimize rows and tables locked.
- Use appropriate isolation levels.
- Keep transactional writes focused on one business operation.

### 11.2 Lock Contention

- Identify frequently contended records.
- Avoid unnecessary exclusive locks.
- Keep update operations targeted.
- Access resources in a consistent order when possible.
- Use optimistic concurrency where appropriate.
- Handle deadlocks and serialization failures with bounded, safe retries.

### 11.3 Bulk Writes

For bulk operations:

- Use appropriate batch sizes.
- Avoid enormous transactions.
- Use bulk insert or update operations when supported.
- Preserve atomicity requirements.
- Make operations idempotent when retries are possible.
- Monitor lock duration, WAL generation, and replication impact.

### 11.4 Critical Workflows

Order placement, payment processing, inventory reservation, and bookings must preserve domain invariants.

Performance improvements must not weaken:

- Transactional guarantees.
- Uniqueness constraints.
- Idempotency.
- Concurrency control.
- Authorization.
- Audit requirements.

Refer to `.ai/data/transactions.md` for transaction policies and recovery strategies.

## 12. Read and Write Optimization

### 12.1 Read Optimization

- Use selective queries and projections.
- Add appropriate indexes.
- Use pagination.
- Batch related reads.
- Reduce unnecessary round trips.
- Cache only suitable read workloads.
- Use precomputed aggregates where justified.
- Use read replicas only when the consistency model allows them.

### 12.2 Write Optimization

- Avoid unnecessary database updates.
- Use bulk writes for appropriate workloads.
- Update only changed fields where practical.
- Avoid excessive secondary indexes on write-heavy collections.
- Batch operations where transaction and consistency requirements permit.
- Use asynchronous processing for non-critical downstream work.
- Keep write transactions small.

### 12.3 Read Replicas

Read replicas may be used to distribute eligible read workloads.

Requirements:

- Identify workloads that can tolerate replication lag.
- Route authoritative writes to the primary.
- Define behavior for read-after-write requirements.
- Monitor replica lag.
- Prevent stale replica reads from affecting critical operations.
- Avoid assuming a replica will always reduce latency.
- Document failover and routing behavior.

Do not route critical booking, payment, or inventory confirmation reads to a lagging replica unless the design guarantees correctness.

## 13. Database Access Patterns

### 13.1 Repository Pattern

Database access should be centralized in appropriate repository or data-access modules.

Repositories must:

- Use parameterized queries.
- Enforce required tenant filters.
- Return only necessary fields.
- Apply bounded retrieval.
- Use appropriate indexes through query design.
- Avoid hidden N+1 behavior.
- Expose clear and predictable contracts.
- Avoid leaking persistence details into unrelated application layers.

Repositories must not become generic abstractions that conceal expensive query behavior.

### 13.2 Batching

Batching can reduce network round trips and repeated query overhead.

Use batching for:

- Loading related entities.
- Bulk status updates.
- Import and export operations.
- Notification recipient retrieval.
- Dashboard aggregation.
- Repeated resolver lookups.

Batch sizes must be bounded and validated under realistic workloads.

### 13.3 DataLoader-Style Batching

For workloads involving repeated entity resolution:

- Collect identifiers during a bounded execution context.
- Retrieve matching entities in a single or small number of queries.
- Restore the requested result order.
- Preserve tenant and authorization boundaries.
- Scope loader instances appropriately to avoid cross-request or cross-tenant leakage.

Do not use a global DataLoader cache for user-specific or tenant-specific results without strict isolation.

## 14. Data Volume and Retention

Growing datasets can degrade performance even when individual queries are well designed.

### 14.1 Data Lifecycle

Each high-volume domain should define:

- Expected record growth.
- Retention requirements.
- Archival strategy.
- Deletion requirements.
- Index maintenance.
- Backup and recovery implications.
- Reporting and analytics access patterns.

### 14.2 Archival

Archiving may be appropriate for:

- Old orders.
- Historical audit events.
- Expired operational records.
- Completed background jobs.
- Historical analytics datasets.

Archival must preserve compliance, auditability, and required retrieval behavior.

### 14.3 Partitioning and Retention

Partitioning or retention-based cleanup may be used where justified.

- Select partition keys based on access patterns.
- Avoid unnecessary partition proliferation.
- Ensure common queries can prune irrelevant partitions.
- Define retention and deletion workflows.
- Test migrations and operational maintenance.
- Preserve tenant isolation across partitions.

### 14.4 Large Exports

Large exports must not execute as unbounded synchronous requests.

Use background jobs, pagination, streaming, or chunked retrieval as appropriate.

Provide:

- Progress tracking.
- Cancellation where practical.
- Bounded concurrency.
- Failure handling.
- Access-controlled output storage.
- Retention and cleanup policies.

## 15. Database Performance and Multi-Tenancy

KAMPYN must maintain predictable database performance across tenants of different sizes and usage patterns.

### 15.1 Tenant-Aware Querying

- Include validated tenant scope in every tenant-owned query.
- Use tenant-aware indexes where appropriate.
- Avoid retrieving cross-tenant datasets for application-side filtering.
- Scope aggregates and reports to the requesting tenant and permitted roles.
- Test tenant isolation for both indexed and fallback query paths.

### 15.2 Noisy Neighbor Protection

A tenant generating unusually high query volume must not degrade service for all other tenants.

Use appropriate controls:

- Per-tenant rate limits.
- Query timeouts.
- Bounded background-job concurrency.
- Request quotas where required.
- Workload prioritization where justified.
- Resource monitoring by tenant.
- Isolation of particularly heavy workloads where needed.

Avoid high-cardinality monitoring labels that create excessive observability overhead.

### 15.3 Tenant Data Distribution

Evaluate whether tenant data distribution is skewed.

Large tenants may require:

- Dedicated indexes or partitions.
- Separate processing queues.
- Workload-specific database routing.
- Dedicated deployments where contractually or operationally required.

These measures must be justified through measured bottlenecks rather than assumed tenant size.

## 16. Caching and Database Performance

Caching may reduce repeated reads but must not hide inefficient database design.

Before adding caching, evaluate:

- Query execution time.
- Index effectiveness.
- Query frequency.
- Reuse rate.
- Cache memory requirements.
- Staleness tolerance.
- Invalidation complexity.
- Failure behavior.

Cache entries must never be treated as authoritative for final transactional decisions.

Database and cache invalidation must be coordinated when data changes. For reliable asynchronous invalidation, use durable event delivery such as the transactional outbox pattern.

Refer to `.ai/performance/cache-strategy.md` for cache key design, TTLs, invalidation, and cache failure behavior.

## 17. Timeouts, Retries, and Backpressure

Database operations must have suitable resource and time boundaries.

### 17.1 Timeouts

Configure appropriate timeouts for:

- Connection acquisition.
- Connection establishment.
- Query execution.
- Transactions.
- Database selection or server discovery.
- External database dependencies.

Timeouts should reflect endpoint requirements and expected query duration.

### 17.2 Retries

Retries must be:

- Bounded.
- Selective.
- Delayed using appropriate backoff and jitter.
- Safe for the operation's idempotency characteristics.
- Observable.
- Designed to avoid retry storms.

Do not retry deterministic query errors or validation failures as though they were transient failures.

### 17.3 Backpressure

When database capacity is constrained:

- Bound concurrent queries.
- Limit background workers.
- Prevent unbounded request queues.
- Reject or defer non-critical work where appropriate.
- Protect database connections for critical workflows.
- Monitor queue wait time and connection acquisition latency.

Backpressure must be applied consistently across application instances and worker processes where shared capacity is involved.

## 18. Database Migrations and Performance

Schema changes can introduce production performance risks.

### 18.1 Migration Requirements

- Use backward-compatible migration strategies where possible.
- Prefer additive changes before destructive changes.
- Assess table rewrites and index creation costs.
- Avoid long-running locks on critical tables.
- Backfill large datasets in bounded batches.
- Monitor replication lag and database load.
- Define rollback or forward-recovery procedures.
- Test migrations with representative data volumes.

### 18.2 Index Migrations

- Use online or concurrent index creation where supported and appropriate.
- Understand database-specific limitations.
- Avoid running multiple resource-intensive migrations concurrently.
- Verify index validity and usage after deployment.
- Remove redundant indexes through a reviewed migration.

### 18.3 Backfills

Large backfills must:

- Be restartable.
- Be idempotent where feasible.
- Use bounded batch sizes.
- Track progress.
- Support safe retries.
- Limit concurrency.
- Avoid starving production traffic.
- Record errors and skipped records.
- Define completion and reconciliation checks.

Refer to `.ai/data/migrations.md` for migration and backfill standards.

## 19. Observability

Database performance must be measurable across application and datastore layers.

### 19.1 Required Metrics

Monitor where supported:

- Query latency.
- Query throughput.
- Slow-query count.
- Connection pool utilization.
- Connection acquisition latency.
- Active and idle connections.
- Transaction duration.
- Lock waits.
- Deadlocks.
- Query timeouts.
- Error and retry rates.
- Database CPU and memory usage.
- Disk I/O and storage growth.
- Cache hit ratios for database pages or query caches where applicable.
- Replication lag.
- Index size and usage.
- Table and collection growth.
- Background job query volume.
- Database saturation and queueing.

### 19.2 Slow Query Analysis

Slow-query monitoring must:

- Use thresholds based on workload and service-level objectives.
- Capture normalized query patterns.
- Avoid exposing sensitive query parameters.
- Identify query frequency as well as individual latency.
- Correlate expensive queries with application endpoints and background jobs.
- Support investigation through execution plans.

A query with moderate latency but extremely high frequency may be a more significant bottleneck than a rarely executed slow query.

### 19.3 Application Tracing

Database calls should be visible in distributed traces.

Traces should help identify:

- The endpoint or job initiating the query.
- Query latency.
- Connection acquisition delays.
- Number of database calls per request.
- Retry attempts.
- Transaction boundaries.
- Downstream dependencies.
- Tenant scope where safe and appropriate.

Avoid including raw SQL parameters or sensitive record contents in traces.

### 19.4 Alerting

Create alerts based on operational thresholds for:

- Sustained query latency increases.
- Connection pool saturation.
- High lock contention.
- Excessive deadlocks.
- Repeated query timeouts.
- Abnormal error rates.
- Database resource exhaustion.
- Replication lag beyond the permitted threshold.
- Unusual data growth.
- Migration-related performance degradation.

Alert thresholds must be based on service objectives, production baselines, and the expected capacity of each deployment.

## 20. Performance Testing

Database performance must be tested under realistic data volumes and concurrency.

### 20.1 Query Benchmarks

For critical queries, benchmark:

- Small datasets.
- Typical production-sized datasets.
- Expected growth scenarios.
- Selective and non-selective filters.
- Cold and warm cache behavior where relevant.
- Concurrent access.
- Read-heavy and write-heavy workloads.
- Tenant data distribution.

### 20.2 Load Testing

Load tests should evaluate:

- Requests per second.
- Concurrent connections.
- Query latency percentiles.
- Database CPU and memory.
- Connection pool saturation.
- Lock contention.
- Error and timeout rates.
- Replication lag.
- Cache hit ratios.
- Recovery after temporary overload.

### 20.3 Stress Testing

Stress tests should identify the point at which:

- Query latency becomes unacceptable.
- Connection pools saturate.
- Database resource limits are reached.
- Background queues grow uncontrollably.
- Error rates increase.
- Recovery becomes unstable.

Tests must include recovery behavior after load is reduced.

### 20.4 Test Data

Use representative test data with realistic:

- Record counts.
- Field distributions.
- Tenant sizes.
- Relationship cardinalities.
- Null and optional values.
- Time distributions.
- Index selectivity.
- Concurrent access patterns.

Small synthetic datasets may not reveal production query-plan or memory problems.

## 21. Performance Optimization Workflow

Database optimization must follow a repeatable process.

1. **Identify:** Define the endpoint, query, or workflow experiencing poor performance.
2. **Measure:** Establish latency, throughput, resource use, and workload baselines.
3. **Inspect:** Review query logs, execution plans, indexes, and connection behavior.
4. **Diagnose:** Identify the actual bottleneck rather than assuming the database is responsible.
5. **Design:** Select the simplest change that addresses the bottleneck.
6. **Implement:** Make the change while preserving correctness and tenant isolation.
7. **Validate:** Run correctness tests, query-plan analysis, and representative benchmarks.
8. **Compare:** Evaluate performance, memory, write overhead, and operational impact.
9. **Deploy:** Release safely with monitoring and a recovery plan.
10. **Review:** Verify production results and document any remaining limitations.

Optimization work should include the impact on write throughput, index maintenance, memory, consistency, and operational complexity.

## 22. Anti-Patterns

The following practices are prohibited unless an explicit architectural exception is approved:

- Using `SELECT *` for large or sensitive queries without justification.
- Retrieving large datasets for application-side filtering.
- Running unbounded queries in user-facing request paths.
- Ignoring tenant scope in database queries.
- Creating indexes without identifying a supporting workload.
- Adding redundant or excessively wide indexes.
- Assuming an index guarantees an efficient query plan.
- Ignoring N+1 queries.
- Executing one query per item in a large collection.
- Creating database connections for individual requests.
- Increasing connection pools without assessing database capacity.
- Holding transactions open during network calls or user interaction.
- Running expensive aggregations synchronously without workload limits.
- Performing large exports entirely in application memory.
- Using read replicas for critical operations without accounting for replication lag.
- Applying unbounded retries to database failures.
- Performing large backfills without batching and progress tracking.
- Introducing partitioning or sharding without a measured requirement.
- Using caches to bypass authoritative validation.
- Optimizing based solely on average query latency.
- Ignoring query frequency, resource utilization, and tail latency.
- Sacrificing correctness, security, or data integrity for throughput.

## 23. Implementation Checklist

### Query Design
- [ ] Only required fields are retrieved.
- [ ] Filters are applied at the database layer.
- [ ] Query complexity is appropriate for expected data volume.
- [ ] Joins and aggregations are justified.
- [ ] N+1 query patterns are avoided.
- [ ] Query results are bounded.

### Indexing
- [ ] Indexes support actual query patterns.
- [ ] Tenant-aware indexing is considered.
- [ ] Composite index order is justified.
- [ ] Redundant indexes are avoided.
- [ ] Execution plans are reviewed for critical queries.
- [ ] Index storage and write overhead are evaluated.

### Connections and Transactions
- [ ] Database clients and connection pools are reused.
- [ ] Pool sizes reflect total deployment capacity.
- [ ] Connection acquisition timeouts are configured.
- [ ] Transactions are short and focused.
- [ ] Lock contention and deadlocks are handled.
- [ ] Retries are bounded and safe.

### Data Volume
- [ ] Pagination, streaming, or chunking is used where required.
- [ ] Large datasets are not unnecessarily loaded into memory.
- [ ] Retention and archival policies are defined.
- [ ] Backfills are restartable and bounded.
- [ ] Partitioning or sharding is justified where used.

### Multi-Tenancy and Security
- [ ] Tenant scope is validated and enforced.
- [ ] Authorization is applied before data is returned.
- [ ] Cross-tenant access is tested.
- [ ] Query resource consumption is bounded.
- [ ] Sensitive values are excluded from logs and traces.

### Performance and Operations
- [ ] Critical queries have performance baselines.
- [ ] Query plans are reviewed.
- [ ] Load and concurrency behavior are evaluated.
- [ ] Metrics and slow-query monitoring are available.
- [ ] Timeouts, backpressure, and failure handling are defined.
- [ ] Production performance is reviewed after deployment.

### Testing
- [ ] Functional tests validate query correctness.
- [ ] Tenant isolation tests cover relevant query paths.
- [ ] Integration tests cover transactions and concurrency.
- [ ] Performance tests use representative datasets.
- [ ] Migration and backfill performance is assessed.
- [ ] Recovery behavior is tested where applicable.

## 24. Definition of Done

A database-related implementation is complete when:

- Query patterns and expected data volumes are understood.
- Queries are designed for selective, bounded retrieval.
- Indexes are justified by actual workloads.
- Connection pooling and transaction boundaries are appropriate.
- Tenant isolation and authorization are enforced at every access path.
- Memory, concurrency, and database resource consumption are bounded.
- Critical queries have been validated using execution plans and representative data.
- Performance-sensitive changes have relevant benchmarks or profiling evidence.
- Failure, timeout, retry, and backpressure behavior are defined.
- Monitoring and alerting support production diagnosis.
- Tests validate correctness, concurrency, security, and relevant performance requirements.
- Documentation reflects the implementation and operational trade-offs.

**Final principle:** Database performance must be engineered through efficient query design, appropriate data modeling, bounded resource usage, and continuous measurement. KAMPYN must scale without compromising transactional correctness, tenant isolation, security, or maintainability.
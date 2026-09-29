# Data Indexing

## 1. Purpose

This document defines the indexing standards for KAMPYN's data layer. It establishes how indexes must be designed, implemented, reviewed, monitored, and maintained across PostgreSQL, MongoDB, Redis, and OpenSearch.

Indexing must improve query performance without compromising data integrity, write throughput, storage efficiency, or maintainability.

The objectives are to:
- Optimize frequent and performance-critical queries.
- Minimize unnecessary full-table scans and document scans.
- Support efficient filtering, sorting, pagination, joins, and lookups.
- Preserve tenant isolation in all indexed access patterns.
- Reduce unnecessary index storage and write amplification.
- Keep indexing strategies aligned with data ownership and query patterns.
- Make index changes measurable, reversible, and safe to deploy.

Indexing is a data-access design decision, not a mechanism to compensate for inefficient queries or poor data modeling.

---

## 2. Core Principles

### 2.1 Query-Driven Indexing

- Indexes MUST be created to support known query patterns, constraints, or operational requirements.
- Index design MUST be based on actual or explicitly anticipated query workloads.
- Every non-trivial index MUST have a documented purpose.
- Indexes MUST NOT be added speculatively without a reasonable access-pattern justification.
- Query design and data modeling MUST be reviewed before adding an index to resolve performance issues.

### 2.2 Correctness Before Performance

- Indexes MUST NOT replace database constraints or application-level validation.
- Unique indexes MUST be used where uniqueness is a data-integrity requirement and supported by the datastore.
- Indexes MUST preserve the intended semantics of filtering, sorting, and lookups.
- Tenant-scoped indexes MUST NOT weaken tenant isolation.
- Correctness MUST be maintained during index creation, rebuilding, and migration.

### 2.3 Minimal Effective Indexing

- Create the smallest set of indexes that adequately supports the workload.
- Avoid redundant, overlapping, unused, and excessively wide indexes.
- Prefer reusing an existing index when it adequately supports the query.
- Every additional index MUST justify its storage, write, and maintenance costs.
- Indexes MUST be reviewed as the workload and schema evolve.

### 2.4 Measurable Performance

- Performance-critical indexes SHOULD be validated using query plans and representative data.
- Query performance MUST be evaluated against realistic data volumes and distributions.
- Index changes MUST be measured against the original query behavior where practical.
- Index existence MUST NOT be treated as proof that a query uses it efficiently.

### 2.5 Tenant-Aware Design

- Tenant-scoped queries MUST include the tenant boundary in their query predicates.
- Indexes supporting tenant-scoped access MUST reflect the actual filtering and sorting patterns.
- Tenant identifiers SHOULD be leading columns in composite indexes when tenant filtering is the primary access pattern.
- Indexes MUST NOT be treated as an authorization or tenant-isolation control.
- Cross-tenant queries MUST be restricted to explicitly authorized platform-level workflows.

---

## 3. Indexing Responsibilities by Datastore

Each datastore has a distinct responsibility in KAMPYN. Indexes MUST support that responsibility rather than duplicate another datastore's role.

| Datastore | Indexing responsibility |
|---|---|
| PostgreSQL | Relational lookups, joins, constraints, transactional queries, filtering, and sorting |
| MongoDB | Document lookups, document-specific filters, nested-field access, and justified aggregation workloads |
| Redis | Efficient key-based access using appropriate key structures; avoid unnecessary secondary-index behavior |
| OpenSearch | Full-text search, relevance scoring, faceted filtering, and search-oriented retrieval |
| Object Storage | No conventional database indexes; use metadata records and object keys for discovery and access |

PostgreSQL remains the authoritative source of relational transactional data. MongoDB indexes MUST support only workloads assigned to MongoDB. Redis and OpenSearch MUST NOT become accidental sources of truth for persistent business records.

---

## 4. PostgreSQL Indexing

### 4.1 Default Index Strategy

PostgreSQL indexes SHOULD be selected based on the query operator, ordering, selectivity, data distribution, and expected write frequency.

Common index types include:

- **B-tree:** Default choice for equality, range comparisons, and ordering.
- **GIN:** Suitable for supported array, JSONB, and full-text search use cases.
- **GiST:** Suitable for supported geometric, range, and specialized operator workloads.
- **BRIN:** Suitable for very large tables where values correlate with physical row order, such as append-oriented time-series data.
- **Hash:** Only where a measured equality-only workload justifies its use over B-tree.

The index type MUST match the operators and query behavior. Specialized index types MUST NOT be selected without confirming operator support and workload suitability.

### 4.2 Primary Keys

- Every persistent entity MUST have a stable primary key.
- Primary keys MUST be indexed through the database's primary-key constraint.
- Primary keys SHOULD remain compact and stable where practical.
- Primary-key design MUST consider index size, insertion patterns, distributed operation, and external reference requirements.
- A primary key MUST NOT be used as a substitute for tenant-scoped access control.

### 4.3 Foreign Keys

- Foreign-key relationships MUST be explicitly defined where relational integrity is required.
- PostgreSQL does not automatically create indexes on referencing foreign-key columns.
- Referencing columns SHOULD be indexed when they are used frequently for joins, lookups, parent deletion checks, or update operations.
- Indexing MUST be based on actual access and mutation patterns rather than blindly indexing every foreign key.

### 4.4 Single-Column Indexes

Single-column indexes SHOULD be used when a column is frequently queried independently and the index is selective enough to be useful.

Suitable examples include:
- Unique email or external reference lookups.
- Frequent entity status filtering when sufficiently selective.
- Timestamp range queries.
- Foreign-key lookups.

Low-selectivity columns, such as boolean flags, MUST NOT be indexed automatically. They may be appropriate as part of a composite or partial index when supported by a demonstrated workload.

### 4.5 Composite Indexes

Composite indexes MUST be designed around the complete query pattern.

For a query such as:

```sql
SELECT id, status, created_at
FROM orders
WHERE tenant_id = $1
  AND status = $2
ORDER BY created_at DESC
LIMIT 50;
```

A potential index is:

```sql
CREATE INDEX idx_orders_tenant_status_created
ON orders (tenant_id, status, created_at DESC);
```

Guidelines:
- Column order MUST reflect the query's equality filters, range predicates, and ordering needs.
- Equality-filtered columns commonly precede range-filtered or ordering columns.
- The leftmost-prefix behavior of B-tree indexes MUST be considered.
- Avoid including columns that add index size without improving the target query.
- Validate the index against the actual query plan.
- Do not create several permutations of similar indexes without evidence that distinct workloads require them.

Composite indexes MUST NOT be designed solely by counting the number of columns in a query.

### 4.6 Covering Indexes

PostgreSQL `INCLUDE` columns MAY be used to support index-only scans when they provide measurable benefit.

Example:

```sql
CREATE INDEX idx_orders_tenant_created
ON orders (tenant_id, created_at DESC)
INCLUDE (status, total_amount);
```

- `INCLUDE` columns MUST be selected based on frequently retrieved fields.
- Wide or frequently updated columns SHOULD NOT be included without a strong justification.
- Index-only scans depend on visibility-map conditions and are not guaranteed.
- Covering indexes MUST be benchmarked against their additional storage and write costs.

### 4.7 Partial Indexes

Partial indexes SHOULD be used when queries consistently target a subset of rows.

Example:

```sql
CREATE INDEX idx_orders_pending_tenant
ON orders (tenant_id, created_at DESC)
WHERE status = 'pending';
```

Guidelines:
- The query predicate MUST be compatible with the partial-index predicate for the planner to use it.
- Partial indexes SHOULD target stable, meaningful subsets.
- Avoid excessive partial indexes for many individual status values unless measured workloads justify them.
- Changes to business-state semantics MUST include a review of affected partial indexes.

### 4.8 Unique Indexes

Unique indexes MUST be used when a uniqueness requirement must be enforced by PostgreSQL and is not already guaranteed by an appropriate primary-key or unique constraint.

Example:

```sql
CREATE UNIQUE INDEX uq_users_tenant_email
ON users (tenant_id, email);
```

- Tenant-scoped uniqueness MUST include the tenant identifier where the business rule is unique-per-tenant.
- Globally unique values MUST use a global uniqueness constraint.
- Case-insensitive uniqueness MUST use an explicitly defined normalization or case-insensitive strategy.
- NULL behavior MUST be understood before relying on a unique index for business rules.
- Application-level checks MUST NOT be the only protection against concurrent duplicate writes.

### 4.9 JSONB Indexes

JSONB indexes MAY be used where document-style fields are intentionally stored in PostgreSQL and frequently queried.

- Use GIN for supported containment and JSONB operators where appropriate.
- Use expression indexes for stable, frequently queried JSON paths when justified.
- Prefer ordinary typed columns for frequently queried, business-critical attributes.
- Do not use JSONB indexing to avoid defining a clear data model.
- Validate the chosen operator class against the actual query operators.

Example:

```sql
CREATE INDEX idx_items_metadata_gin
ON items USING GIN (metadata);
```

Expression index example:

```sql
CREATE INDEX idx_items_metadata_category
ON items ((metadata ->> 'category'));
```

### 4.10 Time-Based and Large Tables

- High-volume append-oriented tables SHOULD be evaluated for time-oriented partitioning and BRIN indexes where physical ordering supports them.
- BRIN MUST NOT be assumed to outperform B-tree for all time-based queries.
- Partitioning MUST be justified by retention, maintenance, query pruning, or scale requirements.
- Indexes on partitioned tables MUST be planned with partition creation, migration, and maintenance procedures.
- Time-based indexes SHOULD match the actual timestamp filtering and ordering patterns.

### 4.11 Full-Text Search

- PostgreSQL full-text search MAY be used for bounded search workloads where its capabilities meet product requirements.
- Search requirements involving advanced relevance, distributed search, faceting, or dedicated search infrastructure SHOULD use OpenSearch where appropriate.
- Full-text indexes MUST define the intended language configuration and text normalization behavior.
- PostgreSQL full-text search MUST NOT silently become a competing search source of truth.

---

## 5. MongoDB Indexing

### 5.1 General Requirements

- MongoDB indexes MUST align with document structure and actual query patterns.
- Indexes MUST be evaluated against collection size, selectivity, document shape, and write frequency.
- Frequently queried nested fields SHOULD be indexed where doing so provides measurable benefit.
- Compound indexes MUST be designed around the expected query filters and sorting.
- MongoDB schema design MUST be reviewed before creating large numbers of indexes.

### 5.2 Single-Field Indexes

Single-field indexes SHOULD be created for frequent, selective queries on individual document fields.

Example:

```javascript
db.orders.createIndex({ tenantId: 1 });
```

Avoid indexing fields that are rarely queried or that have poor selectivity without a demonstrated benefit.

### 5.3 Compound Indexes

Compound indexes SHOULD support recurring multi-field queries.

Example:

```javascript
db.orders.createIndex({
  tenantId: 1,
  status: 1,
  createdAt: -1
});
```

Guidelines:
- Consider equality, sort, and range predicates when ordering fields.
- Follow MongoDB's Equality-Sort-Range principles as a starting point, then validate with actual query plans.
- Consider index-prefix reuse before creating another compound index.
- Use ascending and descending order deliberately for sort support.
- Avoid excessive compound indexes that overlap or duplicate existing access paths.

### 5.4 Unique Indexes

- Unique indexes MUST enforce uniqueness rules that belong to MongoDB.
- Tenant-scoped unique constraints MUST include the tenant identifier when uniqueness is tenant-specific.
- Unique index creation MUST account for existing duplicate documents.
- Sparse and partial unique-index behavior MUST be explicitly understood before use.
- Sharded collection constraints MUST be reviewed against the MongoDB version and shard-key rules in use.

Example:

```javascript
db.users.createIndex(
  { tenantId: 1, email: 1 },
  { unique: true }
);
```

### 5.5 Multikey Indexes

- Multikey indexes MAY be used to query array fields.
- Index behavior for arrays MUST be understood before introducing compound multikey indexes.
- Index size and cardinality MUST be considered when arrays contain many values.
- Queries MUST be tested against representative documents with realistic array sizes.
- Avoid indexing unbounded arrays without a clear access-pattern justification.

### 5.6 Text and Search Indexes

- MongoDB text indexes MAY support basic text-search use cases when appropriate.
- Product-wide search, relevance ranking, advanced filtering, and faceted discovery SHOULD use OpenSearch when it is the designated search platform.
- MongoDB text indexes MUST NOT be added as a redundant search system without a clear requirement.
- Search-index synchronization MUST follow the authoritative-data and eventual-consistency policies.

### 5.7 TTL Indexes

TTL indexes MAY be used for MongoDB documents with datastore-managed expiration requirements.

- TTL MUST only be used when automatic deletion is consistent with the data-retention policy.
- TTL indexes MUST NOT be used as a substitute for audit retention, legally required preservation, or controlled archival workflows.
- Expiration MUST NOT be treated as an exact-time execution guarantee.
- TTL fields MUST be valid date values with clearly defined semantics.

Example:

```javascript
db.sessions.createIndex(
  { expiresAt: 1 },
  { expireAfterSeconds: 0 }
);
```

### 5.8 Partial and Sparse Indexes

- Partial indexes SHOULD be preferred when the indexed subset can be expressed precisely and consistently.
- Sparse indexes MUST only be used when excluding documents without the indexed field matches the intended query behavior.
- Unique, sparse, and partial-index interactions MUST be reviewed for correctness.
- Queries MUST contain compatible predicates to benefit from these indexes.

### 5.9 Sharding Considerations

- Index design MUST be reviewed alongside the shard key.
- Queries SHOULD include the shard key when targeted routing is expected.
- Indexes MUST NOT be assumed to eliminate scatter-gather operations.
- Shard-key cardinality, distribution, write patterns, and query locality MUST be considered.
- Unique-index constraints MUST comply with the applicable sharding rules.
- Indexes MUST be evaluated across the complete sharded workload, not only a single shard.

---

## 6. Redis Indexing and Data Structures

Redis is intended for transient state, caching, coordination, and other explicitly assigned workloads.

- Prefer native Redis key lookups and data structures over building a secondary indexing system.
- Key naming MUST be consistent, namespaced, and tenant-aware where required.
- Data structures MUST be selected based on access patterns.
- Use sets, sorted sets, hashes, streams, or other supported structures only when their semantics fit the workload.
- Avoid unbounded collections and uncontrolled key growth.
- TTL policies MUST be defined for expiring data.
- Redis keys MUST NOT be treated as durable business records unless an explicitly approved architecture establishes that responsibility.
- Any secondary lookup structure MUST have a defined consistency, update, expiration, and recovery strategy.

Redis indexing MUST NOT bypass tenant-scoped authorization or data-access boundaries.

---

## 7. OpenSearch Indexing

### 7.1 Search Index Responsibility

OpenSearch is a derived search and retrieval layer, not the authoritative source for transactional business data.

It MAY support:
- Full-text search.
- Relevance scoring.
- Faceted search and aggregations.
- Filtered discovery of items, vendors, services, and other approved entities.
- Search-oriented sorting and retrieval.
- Search suggestions and autocomplete where configured.

### 7.2 Mapping Design

- Every OpenSearch index MUST have an intentional mapping.
- Field types MUST be explicitly defined for important searchable, filterable, sortable, and aggregatable fields.
- Dynamic mapping SHOULD be restricted or controlled for production indexes.
- Text fields MUST be distinguished from keyword fields according to intended usage.
- Numeric, date, boolean, and identifier fields MUST use appropriate mappings.
- Nested objects MUST use nested mappings only when nested-query semantics are required.
- Mapping changes MUST follow a versioned index migration strategy where incompatible changes are involved.

### 7.3 Text and Keyword Fields

- `text` fields SHOULD be used for analyzed full-text search.
- `keyword` fields SHOULD be used for exact matching, sorting, filtering, and aggregation.
- Multi-fields MAY be used where the same logical field requires both analyzed and exact-match behavior.
- Analyzers MUST reflect product language, tokenization, normalization, and relevance requirements.
- Search analyzers MUST be tested with representative user queries and multilingual data where supported.

### 7.4 Tenant-Aware Search

- Every tenant-scoped search document MUST include the relevant tenant identifier.
- Tenant filters MUST be included in all tenant-scoped search requests.
- Tenant identifiers MUST be mapped to exact-match fields, typically `keyword`.
- Search API authorization MUST enforce tenant scope independently of the search index.
- Direct client access to OpenSearch MUST NOT permit users to remove or override mandatory tenant filters.
- Cross-tenant administrative searches MUST require explicit authorization and auditing.

### 7.5 Index Lifecycle and Versioning

- Index names MUST follow a consistent naming convention.
- Incompatible mapping changes SHOULD use a new index version and controlled reindexing.
- Aliases SHOULD be used to decouple application queries from physical index versions.
- Reindexing MUST define how writes occurring during the process are captured and reconciled.
- Alias cutover MUST be coordinated to prevent incomplete or inconsistent search results.
- Old indexes MUST only be removed after successful validation and an appropriate rollback window.

### 7.6 Synchronization and Consistency

- OpenSearch documents MUST be derived from authoritative data or approved event streams.
- Index updates SHOULD be driven through reliable event delivery or controlled synchronization jobs.
- Eventual consistency MUST be documented for search-visible changes.
- Indexing operations SHOULD be idempotent and support retries.
- Failed indexing events MUST be observable and recoverable.
- Reconciliation processes SHOULD detect missing, stale, or duplicate search documents.
- Search indexing failures MUST NOT silently invalidate successful authoritative database writes unless synchronous search consistency is an explicit product requirement.

### 7.7 Search Performance

- Avoid returning unnecessary fields in search responses.
- Use pagination strategies appropriate to the result size and consistency requirements.
- Deep pagination MUST be avoided where it creates excessive resource consumption.
- Use `search_after` or another suitable strategy for deep result traversal where appropriate.
- Expensive aggregations MUST be justified and bounded.
- Query clauses MUST be reviewed for unnecessary scoring, wildcard expansion, and high-cardinality aggregation costs.
- Index sharding and replica configuration MUST be based on workload and operational evidence.

---

## 8. Multi-Tenant Indexing Standards

Tenant isolation is a system-wide invariant and MUST be enforced independently of indexing.

### 8.1 Tenant-Scoped Queries

- Tenant-scoped data queries MUST include the tenant identifier in the access predicate.
- Repository methods MUST require or derive the authorized tenant context through trusted application boundaries.
- Indexes MUST support common tenant-scoped lookup and sorting patterns.
- Index selection MUST NOT allow an unscoped query to become an accepted access pattern.

### 8.2 Composite Index Patterns

For a common tenant-scoped access pattern:

```sql
SELECT id, status, created_at
FROM bookings
WHERE tenant_id = $1
  AND status = $2
ORDER BY created_at DESC
LIMIT 25;
```

A candidate index is:

```sql
CREATE INDEX idx_bookings_tenant_status_created
ON bookings (tenant_id, status, created_at DESC);
```

This is a starting point, not a universal rule. Indexes MUST be validated against the actual workload, selectivity, and query plan.

### 8.3 Shared and Dedicated Tenant Storage

- Shared-table architectures SHOULD use tenant-aware indexes for frequent tenant-scoped access patterns.
- Dedicated database or schema deployments MUST still follow appropriate indexing standards.
- Index strategies MAY differ between shared SaaS and self-hosted installations when their workload or deployment model differs.
- Tenant-specific indexes MUST NOT be introduced indiscriminately if they cause excessive catalog growth or operational complexity.

### 8.4 Isolation Verification

- Tests MUST verify that tenant-scoped queries return only authorized tenant records.
- Query construction MUST be reviewed for missing tenant predicates.
- Search and cache access MUST apply equivalent tenant boundaries.
- Indexes MUST NOT be treated as a substitute for authorization, row-level security, or repository-level tenant enforcement.

---

## 9. Pagination, Sorting, and Filtering

### 9.1 Pagination

- Pagination queries MUST use deterministic ordering.
- Stable tie-breaker columns SHOULD be included when ordering fields are not unique.
- Large datasets SHOULD use keyset or cursor-based pagination where appropriate.
- Offset pagination MAY be used for bounded datasets and shallow navigation.
- Deep offset pagination MUST be reviewed for performance impact.
- Pagination indexes MUST match the filter and ordering pattern where practical.

### 9.2 Sorting

- Frequently used sort orders SHOULD be supported by appropriate indexes when the workload justifies them.
- Composite indexes MUST reflect the ordering direction and preceding filter conditions.
- Sorting by unindexed fields MUST be evaluated against realistic result sizes.
- Dynamic arbitrary sorting MUST be bounded to an approved set of fields and safe query construction.

### 9.3 Filtering

- Frequently used, selective filters SHOULD be indexed when supported by query evidence.
- Optional filters MUST be evaluated as actual query combinations.
- Avoid creating every possible combination of optional filter indexes.
- For complex search and discovery filters, evaluate whether the workload belongs in OpenSearch rather than adding excessive relational indexes.

---

## 10. Index Naming Conventions

Index names MUST be descriptive, consistent, and easy to identify during operations and schema review.

Recommended PostgreSQL naming patterns:

| Index type | Naming pattern | Example |
|---|---|---|
| Primary key | Database-generated or `pk_<table>` | `pk_orders` |
| Unique | `uq_<table>_<columns>` | `uq_users_tenant_email` |
| Standard | `idx_<table>_<columns>` | `idx_orders_tenant_created` |
| Partial | `idx_<table>_<purpose>` | `idx_orders_pending` |
| Expression | `idx_<table>_<expression>` | `idx_items_metadata_category` |
| Foreign key | `idx_<table>_<foreign_key>` | `idx_orders_customer_id` |

Guidelines:
- Names MUST communicate the table or collection and indexed fields or purpose.
- Names MUST remain within datastore identifier limits.
- Names MUST NOT depend on temporary developer-specific naming.
- Naming conventions MUST be consistent across migrations and repositories.

MongoDB and OpenSearch indexes SHOULD use similarly descriptive names or documented equivalent conventions.

---

## 11. Index Migrations and Deployment

### 11.1 Migration Requirements

- Production index changes MUST be managed through version-controlled migrations or an approved deployment process.
- Manual production index changes MUST be recorded and reconciled with migration state.
- Migrations MUST define the intended index, its purpose, and rollback or recovery approach.
- Existing data MUST be checked for violations before creating unique indexes.
- Index changes MUST be coordinated with application compatibility and rollout strategy.

### 11.2 Safe Index Creation

- Large PostgreSQL indexes SHOULD be created concurrently where operationally appropriate to reduce write blocking.
- Concurrent index creation MUST account for PostgreSQL migration transaction restrictions.
- Failed or invalid indexes MUST be detected and cleaned up safely.
- MongoDB and OpenSearch index builds MUST consider workload impact, resource consumption, and deployment behavior.
- Index builds MUST be scheduled and monitored appropriately for high-volume production workloads.

PostgreSQL example:

```sql
CREATE INDEX CONCURRENTLY idx_orders_tenant_created
ON orders (tenant_id, created_at DESC);
```

### 11.3 Rollback and Removal

- Index removal MUST be justified by evidence that the index is redundant, unused, harmful, or no longer required.
- Critical query plans MUST be reviewed before removing an index.
- Rollback plans MUST consider application versions and workload dependencies.
- Index removal SHOULD be separated from unrelated schema changes when doing so reduces deployment risk.
- OpenSearch index replacement MUST preserve a rollback path until the new index is validated.

---

## 12. Query Plan Analysis

### 12.1 PostgreSQL

Use `EXPLAIN` and, where safe, `EXPLAIN ANALYZE` to inspect query plans.

Example:

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT id, status, created_at
FROM orders
WHERE tenant_id = 'tenant-id'
  AND status = 'pending'
ORDER BY created_at DESC
LIMIT 50;
```

Review:
- Sequential scans versus index scans.
- Estimated versus actual row counts.
- Rows filtered and rows returned.
- Join algorithms and join order.
- Sort operations and memory or disk usage.
- Buffer hits and reads.
- Execution time and planning time.
- Whether the chosen index matches the intended query pattern.

`EXPLAIN ANALYZE` executes the query. It MUST be used cautiously for expensive queries or operations with side effects.

### 12.2 MongoDB

Use `explain("executionStats")` for relevant query analysis.

Example:

```javascript
db.orders
  .find({ tenantId: "tenant-id", status: "pending" })
  .sort({ createdAt: -1 })
  .limit(50)
  .explain("executionStats");
```

Review:
- Winning plan and rejected plans.
- Index scans and collection scans.
- Documents examined versus documents returned.
- Keys examined versus documents returned.
- Sort stages and blocking operations.
- Execution time under representative data conditions.

### 12.3 OpenSearch

Use the Profile API, explain functionality, slow logs, and query metrics where appropriate.

Review:
- Query and fetch phase costs.
- Shard-level execution.
- Scoring and aggregation costs.
- Mapping suitability.
- Query expansion and expensive operators.
- Result size and pagination strategy.

Query-plan analysis MUST be interpreted in the context of realistic data distribution and concurrent workload.

---

## 13. Performance and Resource Management

### 13.1 Read Performance

- Indexes SHOULD reduce the amount of data examined for common queries.
- Indexes MUST be validated against expected selectivity and query shape.
- Queries MUST retrieve only required fields where practical.
- Avoid repeated lookups that can be served by a well-designed query or existing index.
- Indexing MUST be considered alongside join strategy, batching, pagination, and data layout.

### 13.2 Write Performance

- Every index adds write and maintenance overhead.
- High-write tables MUST avoid unnecessary indexes.
- Frequently updated columns SHOULD NOT be indexed without a clear read-performance justification.
- Indexes on mutable fields MUST be reviewed for update amplification.
- Bulk ingestion and batch-update workloads MUST account for index maintenance costs.

### 13.3 Storage

- Index storage MUST be monitored for large and high-growth tables or collections.
- Wide indexes MUST be justified.
- Index bloat and fragmentation MUST be monitored where applicable.
- Index maintenance MUST be included in operational planning.
- Large indexes MUST be evaluated for backup, restore, and replication impact.

### 13.4 Concurrency

- Index changes MUST consider lock behavior and concurrent writes.
- Unique-index creation MUST account for concurrent inserts and existing duplicates.
- Index maintenance MUST follow datastore-specific concurrency guarantees.
- Performance tests SHOULD include representative concurrent read and write workloads.

---

## 14. Monitoring and Maintenance

### 14.1 PostgreSQL

Monitor:
- Index scans and index usage statistics.
- Sequential scans on high-volume tables.
- Index size and table-to-index size ratios.
- Dead tuples, vacuum activity, and table bloat.
- Slow queries and query-plan regressions.
- Index creation failures and migration duration.
- Replication and backup impact.

Unused-index statistics MUST be interpreted carefully, considering database restarts, statistics resets, rare workloads, and constraint dependencies.

### 14.2 MongoDB

Monitor:
- Index usage statistics.
- Query execution and collection-scan frequency.
- Index size and memory pressure.
- Documents and keys examined.
- Slow queries and rejected query plans.
- Index build duration and resource consumption.
- Replication lag and shard-level query distribution where applicable.

### 14.3 OpenSearch

Monitor:
- Index size, shard count, and segment count.
- Query latency and rejection rates.
- Indexing throughput and indexing failures.
- Search and indexing thread-pool queues.
- Heap pressure and garbage collection.
- Shard allocation and cluster health.
- Refresh, merge, and reindexing activity.

### 14.4 Maintenance Rules

- Index statistics MUST be maintained using datastore-appropriate mechanisms.
- Maintenance operations MUST be planned for production impact.
- Index usage MUST be reviewed periodically for high-volume and high-cost datasets.
- Index removal MUST be preceded by dependency and query-pattern review.
- Index health MUST be part of database and search operational reviews.

---

## 15. Testing Requirements

Index changes MUST be tested according to their risk and workload impact.

### 15.1 Functional Tests

- Verify that indexed queries return the expected results.
- Verify that unique indexes reject duplicate values where required.
- Verify that partial and sparse indexes preserve intended query behavior.
- Verify tenant-scoped queries do not return cross-tenant records.
- Verify pagination and ordering remain deterministic.

### 15.2 Performance Tests

- Use representative data volumes and distributions.
- Measure query latency before and after index changes where possible.
- Compare rows or documents examined with rows or documents returned.
- Test common filters, sort combinations, and pagination patterns.
- Evaluate write overhead for high-ingestion and high-update workloads.
- Include concurrency for critical read/write paths.
- Test worst-case and skewed data distributions when relevant.

### 15.3 Migration Tests

- Validate migration execution on a representative database.
- Check unique-index creation against duplicate and NULL edge cases.
- Verify migration behavior during concurrent application activity where applicable.
- Validate rollback or recovery procedures.
- Confirm application compatibility during rolling deployments.

Performance tests MUST NOT rely exclusively on tiny local datasets or synthetic distributions that fail to represent production behavior.

---

## 16. Security and Privacy

- Indexes MUST NOT be treated as security boundaries.
- Tenant filters MUST be enforced by trusted application and database boundaries.
- Sensitive values MUST NOT be copied into search indexes or cache structures without a documented need and appropriate protection.
- Search documents MUST include only fields required for approved search behavior.
- Sensitive identifiers MUST NOT be exposed through index names, logs, query traces, or operational dashboards.
- Data deletion and retention requirements MUST include secondary indexes, caches, and search projections.
- Reindexing and index snapshots MUST follow applicable access-control and data-protection requirements.
- Administrative index operations MUST be restricted to authorized roles.

---

## 17. Anti-Patterns

The following practices are prohibited unless an explicit, documented exception is approved:

- Creating indexes without identifying a query or constraint they support.
- Adding an index before inspecting the query plan and query structure.
- Indexing every column or every foreign key by default.
- Creating many overlapping composite indexes without evidence.
- Using indexes to compensate for incorrect data modeling.
- Treating low-selectivity indexes as automatically beneficial.
- Building extremely wide covering indexes without measurement.
- Relying on application-level uniqueness checks without database enforcement where database enforcement is required.
- Omitting tenant predicates because a tenant-aware index exists.
- Using OpenSearch or Redis as an undocumented replacement for authoritative storage.
- Creating unbounded multikey indexes or Redis secondary structures without capacity planning.
- Using deep offset pagination on very large datasets without evaluating its cost.
- Removing indexes solely because current usage statistics show zero scans.
- Making undocumented manual production index changes.
- Deploying incompatible search mappings without a versioned migration strategy.
- Assuming an index guarantees a specific query plan or latency.

---

## 18. Index Review Checklist

Before adding or changing an index, verify:

- [ ] The query pattern, integrity constraint, or operational need is documented.
- [ ] The owning datastore is appropriate for the workload.
- [ ] Existing indexes have been reviewed for reuse.
- [ ] The query and data model have been inspected for structural inefficiencies.
- [ ] Index type and column or field order match the operators and access pattern.
- [ ] Tenant-scoped access patterns are supported without weakening isolation.
- [ ] Selectivity and data distribution have been considered.
- [ ] Write amplification and storage costs have been evaluated.
- [ ] Query plans have been inspected using representative data.
- [ ] Performance improvements have been measured where practical.
- [ ] Unique and partial-index semantics have been validated.
- [ ] Migration, deployment, rollback, and recovery plans are documented.
- [ ] Monitoring and maintenance implications are understood.
- [ ] Tests cover correctness, tenant isolation, and relevant performance risks.
- [ ] Documentation reflects the final index purpose and ownership.

---

## 19. Definition of Done

An indexing change is complete only when:

- Its purpose and supported workload are documented.
- The index is aligned with the authoritative data model and datastore responsibility.
- Tenant isolation and data-integrity requirements remain intact.
- The index is created through a version-controlled migration or approved process.
- Query behavior and relevant execution plans have been reviewed.
- Performance and resource impact have been evaluated proportionately to risk.
- Tests cover the relevant correctness and migration scenarios.
- Monitoring and operational considerations are addressed.
- Unnecessary or redundant indexes have not been introduced.
- The change has been reviewed for security, maintainability, and deployment safety.

Exceptions MUST be explicit, justified, documented, and approved by the appropriate engineering owner.

---

## 20. Final Principle

**An index is justified by a demonstrated access pattern, not by the mere possibility that a query might use it.**

KAMPYN MUST maintain a deliberate balance between read performance, write throughput, data integrity, tenant isolation, storage cost, and operational complexity. Indexes should make common workloads efficient while keeping the data layer understandable, measurable, and safe to evolve.
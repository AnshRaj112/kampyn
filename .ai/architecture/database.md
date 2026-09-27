# KAMPYN Database Architecture

## 1. Purpose

This document defines the database architecture for KAMPYN.

KAMPYN uses multiple data systems because different workloads have different data characteristics.

The database architecture must preserve:

- Data correctness.
- Clear ownership.
- Strong tenant isolation.
- Transactional integrity.
- Query efficiency.
- Scalability.
- Recoverability.
- Operational simplicity.
- Clear boundaries between authoritative and derived data.

The primary principle is:

> Use the simplest data store that correctly satisfies the workload.

A database must never be selected merely because it is available or familiar.

---

# 2. Database System Responsibilities

KAMPYN may use the following storage systems:

```text
                         KAMPYN
                            │
          ┌─────────────────┼─────────────────┐
          │                 │                 │
      PostgreSQL         MongoDB            Redis
          │                 │                 │
  Transactional data   Document-oriented    Cache / transient
  relational data      data                 state
          │                 │
          └────────────┬────┘
                       │
                  Application
                       │
              ┌────────┴────────┐
              │                 │
         OpenSearch        Object Storage
              │                 │
        Search/read       Files/images/
          projection      large objects
```

Each system has a defined responsibility.

---

# 3. PostgreSQL

PostgreSQL is the primary relational and transactional datastore.

Use PostgreSQL for data requiring:

- Strong consistency.
- Transactions.
- Referential integrity.
- Complex relational queries.
- Constraints.
- Structured business entities.
- Financial or transactional records.
- Booking and reservation state.
- Relationships requiring database-enforced integrity.

Typical examples:

```text
Users
Tenants
Memberships
Roles
Permissions
Orders
Payments
Bookings
Reservations
Invoices
Complaints
Inventory transactions
Audit records
```

The exact ownership of each entity must be defined by the domain architecture.

---

# 4. MongoDB

MongoDB may be used for workloads where document-oriented storage provides a meaningful architectural advantage.

Appropriate candidates may include:

- Flexible document structures.
- Large nested documents.
- Domain data with naturally document-oriented access patterns.
- Data where schema flexibility is intentional.
- High-volume document workloads where relational modeling provides limited benefit.

MongoDB must not become a generic secondary database for arbitrary application data.

Every MongoDB collection must have a defined domain owner.

---

# 5. Redis

Redis is primarily an infrastructure datastore for:

- Caching.
- Rate limiting.
- Short-lived coordination state.
- Distributed locks where explicitly justified.
- Session state where the authentication architecture requires it.
- Temporary processing state.

Redis should not become the authoritative store for durable business data unless explicitly defined by an architectural decision.

See:

```text
architecture/caching.md
```

for the caching architecture.

---

# 6. OpenSearch

OpenSearch is a derived search and discovery system.

It should be used for:

- Full-text search.
- Fuzzy search.
- Autocomplete.
- Ranking.
- Search filtering.
- Faceted discovery.
- Search-oriented aggregations.

OpenSearch is not the default source of truth for business entities.

The general architecture is:

```text
PostgreSQL / MongoDB
        │
        │ Change / Event
        ↓
   Projection Pipeline
        │
        ↓
    OpenSearch
```

If OpenSearch data is lost, it must be possible to rebuild the index from authoritative data.

---

# 7. Object Storage

Object storage should be used for large binary or file-based data.

Examples:

- Food images.
- User-uploaded images.
- Documents.
- Reports.
- Export files.
- Large generated artifacts.
- Application attachments.

The database should generally store metadata and references rather than large binary objects.

Example:

```text
Database
    ↓
object_key
content_type
size
checksum
owner_id
created_at

Object Storage
    ↓
actual binary content
```

---

# 8. Source of Truth

Every durable entity must have one clearly defined authoritative source.

Example:

```text
Order
  → PostgreSQL

Food Item
  → PostgreSQL or MongoDB
     depending on finalized domain ownership

Search Representation
  → OpenSearch

Cached Order
  → Redis
```

A derived representation must never silently become a second authoritative source.

Avoid:

```text
PostgreSQL says A
MongoDB says B
OpenSearch says C
```

without an explicit conflict-resolution model.

---

# 9. Data Ownership

Every entity must have:

- One owning domain.
- One authoritative datastore.
- One responsible module.
- Clearly defined read interfaces.
- Clearly defined write interfaces.

For example:

```text
Food Ordering Domain
        │
        ├── Orders
        ├── Order Items
        ├── Payments
        └── Order State
```

Another domain should not directly mutate these tables or collections.

Cross-domain access should occur through defined interfaces or application-level contracts.

---

# 10. Domain Boundaries

Database boundaries should follow domain boundaries.

Prefer:

```text
Ordering
 ├── order repository
 ├── order service
 └── order data

Inventory
 ├── inventory repository
 ├── inventory service
 └── inventory data

Booking
 ├── booking repository
 ├── booking service
 └── booking data
```

over a system where every module can directly access every table.

Database structure must reinforce application architecture rather than undermine it.

---

# 11. Relational Modeling

Relational data should be modeled around actual business relationships.

Use:

- Primary keys.
- Foreign keys where appropriate.
- Unique constraints.
- Check constraints.
- Not-null constraints.
- Appropriate indexes.
- Explicit relationship tables.

Do not rely exclusively on application code for invariants that the database can safely enforce.

---

# 12. Primary Keys

Primary keys must be:

- Stable.
- Unique.
- Non-semantic where practical.
- Safe for distributed generation.
- Appropriate for indexing.

Do not use mutable business values as primary keys unless there is a documented reason.

Examples of values that should generally not be primary identifiers:

```text
email
phone number
username
food item name
hostel room label
```

Business identifiers may change.

Internal identity should remain stable.

---

# 13. Foreign Keys

Use foreign keys when relational integrity should be enforced by PostgreSQL.

Foreign keys are especially important for:

- Ownership relationships.
- Required parent-child relationships.
- Transactional entities.
- Referentially important data.

Avoid unnecessary foreign keys only when the architectural tradeoff is intentional and documented.

---

# 14. Nullability

Nullability must represent a real domain state.

Do not use nullable columns simply because a value is inconvenient to populate.

Distinguish between:

```text
Unknown
Not applicable
Not provided
Empty
Zero
False
Deleted
```

These states are not automatically equivalent.

---

# 15. Timestamps

Persist timestamps consistently.

Prefer timezone-aware timestamps for events and business records.

Typical fields may include:

```text
created_at
updated_at
deleted_at
processed_at
completed_at
expires_at
```

Timestamp semantics must be documented where they are not obvious.

Do not mix local server time and UTC-based persistence without an explicit reason.

---

# 16. Soft Deletion

Soft deletion should only be used where the business domain requires retention or recoverability.

Typical representation:

```text
deleted_at
```

If soft deletion is used, every relevant query path must consider it.

Do not accidentally expose soft-deleted records through:

- APIs.
- Search.
- Reports.
- Caches.
- Background jobs.

---

# 17. Schema Constraints

Important business invariants should be enforced as close to the authoritative data as practical.

Examples:

```text
Unique email per tenant
Unique booking reference
Non-negative inventory quantity
Valid status transitions where enforceable
Unique membership relationship
```

Application validation remains necessary, but database constraints provide the final protection against concurrent or unexpected writes.

---

# 18. Transactions

Transactions must have clearly defined boundaries.

A transaction should contain the smallest set of operations that must succeed or fail together.

Example:

```text
Create Order
    ↓
Reserve Inventory
    ↓
Create Order Items
    ↓
Commit
```

If these operations must be atomic, they belong within the appropriate transactional boundary.

Do not keep transactions open while performing:

- External HTTP calls.
- Slow file uploads.
- Long computations.
- User interaction.
- Unbounded background processing.

---

# 19. Transaction Ownership

The application service responsible for a business operation should generally own the transaction boundary.

Prefer:

```text
Application Service
      ↓
Begin Transaction
      ↓
Repository A
Repository B
Repository C
      ↓
Commit
```

rather than repositories independently opening unrelated transactions for one logical operation.

This keeps atomicity explicit.

---

# 20. Cross-Database Transactions

Avoid requiring atomic transactions across PostgreSQL, MongoDB, Redis, OpenSearch, and external systems.

Instead:

```text
Authoritative Transaction
        ↓
Commit
        ↓
Event / Outbox
        ↓
Derived Systems
```

Use eventual consistency where appropriate.

Distributed transactions should only be introduced when the business requirement genuinely requires them and the operational consequences are understood.

---

# 21. Transactional Outbox

When a database mutation must reliably produce an event, consider the transactional outbox pattern.

Conceptually:

```text
┌──────────────────────────────┐
│ Database Transaction         │
│                              │
│ Update Business Data         │
│ Insert Outbox Event          │
│                              │
│ COMMIT                       │
└──────────────┬───────────────┘
               ↓
        Outbox Processor
               ↓
      Event / Message Bus
               ↓
 ┌─────────────┼─────────────┐
 ↓             ↓             ↓
Cache       OpenSearch     Other
Update       Update        Consumers
```

This prevents the classic failure where the database commits but event publication fails.

---

# 22. Eventual Consistency

Derived systems may be eventually consistent.

Examples:

```text
Database → OpenSearch
Database → Redis
Database → Analytics
Database → Notifications
```

The architecture must explicitly define:

- Expected propagation delay.
- Failure behavior.
- Retry behavior.
- Reconciliation.
- Rebuild process.

Do not present eventually consistent data as immediately authoritative.

---

# 23. MongoDB Modeling

MongoDB schemas should follow access patterns and domain boundaries.

Choose embedding versus referencing based on:

- Read patterns.
- Update patterns.
- Document size.
- Relationship cardinality.
- Consistency requirements.
- Query frequency.

Do not blindly normalize MongoDB like a relational database.

Do not blindly embed everything either.

---

# 24. MongoDB Document Size

MongoDB documents must remain within platform limits.

Large or unbounded arrays should be treated carefully.

Avoid documents where a frequently growing field can become effectively unbounded.

Examples requiring caution:

```text
user.messages[]
order.history[]
foodCourt.orders[]
```

Large collections of related entities should generally be modeled separately.

---

# 25. MongoDB Indexing

MongoDB indexes must follow actual query patterns.

Consider:

- Equality filters.
- Sort order.
- Compound indexes.
- Selectivity.
- Cardinality.
- Write overhead.

Do not create indexes for every field.

Each index adds:

- Storage.
- Write cost.
- Memory pressure.
- Operational complexity.

---

# 26. PostgreSQL Indexing

Indexes should support actual access patterns.

Common candidates include:

- Foreign keys used in queries.
- Frequently filtered columns.
- Sorting columns.
- Unique constraints.
- Composite query patterns.

Consider composite index order carefully.

An index should be justified by query behavior rather than by the existence of a column.

---

# 27. Query Design

Database queries must retrieve only the data required.

Avoid:

```text
SELECT *
```

when the application needs only a subset of fields.

Prefer:

```text
SELECT id, name, status
```

when those are the only required fields.

The same principle applies to MongoDB projections and OpenSearch queries.

---

# 28. N+1 Queries

N+1 query patterns must be avoided where they create meaningful overhead.

Problem:

```text
Load 100 orders
    ↓
100 additional customer queries
```

Prefer:

```text
Load orders
    ↓
Batch related customer data
```

or an appropriate join/projection.

Do not blindly replace every query with a huge join. Query shape must match the access pattern.

---

# 29. Pagination

Large datasets must never be returned without bounded pagination.

Prefer cursor/keyset pagination for large or frequently changing datasets where appropriate.

Example:

```text
GET /orders?cursor=...
```

rather than:

```text
GET /orders?page=90000
```

Offset pagination may remain appropriate for smaller datasets or interfaces where its semantics are acceptable.

---

# 30. Bulk Operations

Use bulk database operations where they reduce unnecessary round trips.

Examples:

```text
Bulk insert
Bulk update
Batch reads
Batch deletes
```

However, bulk operations must still respect:

- Transaction boundaries.
- Validation.
- Authorization.
- Resource limits.
- Error semantics.

Do not create enormous unbounded bulk requests.

---

# 31. Connection Pooling

Database connections are finite resources.

Configure connection pools based on:

- Application instance count.
- Database capacity.
- Query latency.
- Concurrent workload.
- Deployment topology.

Avoid allowing every application instance to independently create excessive connections.

Horizontal scaling must account for aggregate connection count.

---

# 32. Query Timeouts

Long-running queries must have appropriate timeout protection.

Timeouts prevent one pathological query from consuming resources indefinitely.

The system should distinguish between:

- Query timeout.
- Connection timeout.
- Transaction timeout.
- Application request timeout.

Timeout values should reflect actual workload requirements.

---

# 33. Database Concurrency

Concurrent writes must preserve business invariants.

Possible tools include:

- Atomic updates.
- Database constraints.
- Row locks.
- Optimistic concurrency.
- Version columns.
- Serializable transactions where justified.

Do not rely on:

```text
read → check → write
```

without considering concurrent requests.

---

# 34. Inventory and Resource Allocation

Inventory, seats, rooms, washing machines, shuttle capacity, and other finite resources require atomic allocation.

Avoid:

```text
Read available = 1
        ↓
Application checks available
        ↓
Two requests both reserve
```

Prefer database-enforced atomic operations or appropriate locking.

The authoritative datastore must enforce the final resource invariant.

---

# 35. Booking and Reservation Data

Booking systems require explicit state modeling.

Example:

```text
PENDING
   ↓
CONFIRMED
   ↓
COMPLETED
```

with possible failure states:

```text
PENDING
   ↓
CANCELLED
```

State transitions must be validated.

Do not allow arbitrary status updates from generic CRUD operations.

---

# 36. Payment Data

Payment state must be authoritative and auditable.

Do not treat:

- Redis.
- OpenSearch.
- Browser state.
- Cached API responses.

as authoritative payment state.

Payment updates should be:

- Idempotent.
- Transactionally consistent where applicable.
- Auditable.
- Resistant to duplicate callbacks.

External payment providers must be integrated through explicit state transitions.

---

# 37. Audit Data

Important security and business actions may require durable audit records.

Examples:

```text
Role changed
Permission granted
Order cancelled
Booking modified
Payment state changed
Inventory adjusted
Administrative action performed
```

Audit data should be durable and protected from ordinary application mutation.

---

# 38. Sensitive Data

Sensitive data must receive appropriate database protection.

Consider:

- Encryption at rest.
- Encryption in transit.
- Application-level encryption for especially sensitive fields.
- Access control.
- Least privilege.
- Retention.
- Deletion.
- Auditability.

Do not store secrets or credentials in ordinary database columns unless they are specifically designed and protected for that purpose.

---

# 39. Password Data

Passwords must never be stored in plaintext.

Store only strong password hashes using an approved password hashing algorithm.

Authentication architecture defines the complete credential lifecycle.

See:

```text
architecture/authentication.md
```

---

# 40. Multi-Tenant Data

Tenant-aware data must include an explicit tenant boundary where required.

Conceptually:

```text
tenant_id
```

should be part of the data model for tenant-owned resources where appropriate.

Tenant isolation must be enforced at:

- Application layer.
- Query layer.
- Authorization layer.
- Cache layer.
- Search layer.

Do not rely on developers remembering to add a tenant filter to every query.

---

# 41. Cross-Tenant Administrative Access

Cross-tenant administrative operations must be explicit.

Do not implement:

```text
tenant_id = NULL
```

as an implicit representation of unrestricted access unless the architecture explicitly defines that model.

Administrative scope should be represented through explicit authorization context.

---

# 42. Data Retention

Every important data category should eventually have a retention policy.

Consider:

- Legal requirements.
- Business requirements.
- Privacy.
- Storage cost.
- Audit requirements.
- User deletion.
- Institutional policies.

Retention and deletion should be deliberate rather than accidental.

---

# 43. Data Deletion

Deletion must account for derived systems.

Example:

```text
Delete Entity
    ↓
Authoritative Database
    ↓
Event
    ├── OpenSearch deletion
    ├── Cache invalidation
    ├── Object cleanup
    └── Derived-data cleanup
```

Do not delete an authoritative record while leaving indefinitely accessible derived copies without an intentional retention policy.

---

# 44. OpenSearch Synchronization

OpenSearch documents must be derived from authoritative data.

Preferred flow:

```text
Authoritative Database
        ↓
Change Event
        ↓
Projection Worker
        ↓
OpenSearch
```

The projection process must support:

- Retry.
- Idempotency.
- Reprocessing.
- Failure handling.
- Full rebuild.

---

# 45. Search Reindexing

Indexes must be rebuildable.

A typical rebuild process:

```text
Authoritative Data
       ↓
Read in batches
       ↓
Transform
       ↓
Bulk index
       ↓
Validate
       ↓
Promote index
```

Never make the only copy of critical business data an OpenSearch index.

---

# 46. Redis and Database Consistency

Redis should generally derive its values from authoritative state.

If Redis contains stale data:

```text
Invalidate
   ↓
Read authoritative source
   ↓
Repopulate
```

Do not repair authoritative database state based solely on a cache value.

---

# 47. Database Migrations

Schema changes must be version-controlled.

Every migration should be:

- Deterministic.
- Reviewable.
- Repeatable in deployment environments.
- Compatible with the application rollout strategy.

Avoid manual production schema changes.

---

# 48. Backward-Compatible Migrations

For zero- or low-downtime deployments, prefer:

```text
Expand
   ↓
Deploy compatible application
   ↓
Migrate data
   ↓
Switch reads/writes
   ↓
Contract
```

Avoid simultaneously deploying an application that requires a schema change which does not yet exist.

---

# 49. Data Backfills

Large backfills must be treated as production workloads.

Consider:

- Batching.
- Rate limiting.
- Lock duration.
- Database load.
- Retry.
- Resume capability.
- Progress tracking.
- Monitoring.

Never run an unbounded migration query against a large production table without understanding its impact.

---

# 50. Database Seeds

Seed data must be clearly distinguished from production data.

Seeds may provide:

- Development data.
- Test data.
- Local configuration.
- Initial system configuration.

Production initialization must be explicit and idempotent.

Never place real secrets into seed files.

---

# 51. Database Backups

Every authoritative datastore requires an appropriate backup strategy.

Define:

- Backup frequency.
- Retention.
- Storage location.
- Encryption.
- Access control.
- Recovery procedure.
- Recovery point objective.
- Recovery time objective.

A backup that has never been restored is not sufficient evidence of recoverability.

---

# 52. Disaster Recovery

Recovery procedures must account for all persistent systems.

Potential recovery dependencies include:

```text
PostgreSQL
MongoDB
Object Storage
OpenSearch
Redis
Message/Event Infrastructure
```

Not every system needs identical recovery treatment.

For example:

```text
PostgreSQL
→ Restore authoritative data

OpenSearch
→ Rebuild from authoritative data

Redis
→ Repopulate from authoritative data
```

---

# 53. Database Observability

Monitor:

- Query latency.
- Slow queries.
- Connection usage.
- Connection pool saturation.
- Lock contention.
- Deadlocks.
- Replication lag where applicable.
- Storage utilization.
- Index usage.
- Cache effectiveness.
- Error rates.
- Transaction duration.

Database metrics should be correlated with API and application metrics.

---

# 54. Slow Query Management

Slow queries must be investigated using actual evidence.

Consider:

- Query plan.
- Index usage.
- Cardinality estimates.
- Data volume.
- Join strategy.
- Sort operations.
- Network transfer.
- Lock contention.

Do not add indexes blindly to fix a slow query.

---

# 55. ORM and Query Builders

ORMs and query builders may be used to improve developer productivity, but they must not hide database behavior.

Developers must understand:

- Generated SQL.
- Query count.
- Transaction boundaries.
- Index usage.
- Joins.
- Pagination.
- Locking.

Raw queries may be appropriate when they provide a measurable architectural or performance benefit.

They must remain isolated and documented.

---

# 56. Repository Layer

Application and domain code should not depend directly on database drivers.

Prefer:

```text
Application
    ↓
Repository Interface
    ↓
Repository Implementation
    ↓
Database Driver
```

Repositories should expose domain-relevant operations rather than generic database primitives.

Avoid APIs such as:

```text
repository.executeAnything(...)
```

that bypass architectural boundaries.

---

# 57. Database Errors

Database errors must be translated into meaningful application-level behavior.

Examples:

```text
Unique constraint violation
        ↓
Conflict

Foreign key violation
        ↓
Invalid relationship

Serialization failure
        ↓
Retryable operation

Connection failure
        ↓
Infrastructure failure
```

Do not expose raw database errors to API consumers.

---

# 58. Deadlocks and Serialization Failures

Concurrent transactional systems may encounter:

- Deadlocks.
- Serialization failures.
- Lock timeouts.

Where retry is safe, use bounded retries with appropriate backoff.

Retries must only occur when the operation is known to be retry-safe.

Do not blindly retry every database error.

---

# 59. Database and Application Compatibility

During deployments, old and new application versions may coexist.

Schema changes must therefore consider:

```text
Old Application
        +
New Application
        +
Current Schema
```

All combinations that can occur during rollout must remain safe.

---

# 60. Large Data Processing

Large exports, reports, reconciliation jobs, and analytics workloads must not load entire datasets into application memory.

Prefer:

- Streaming.
- Pagination.
- Chunking.
- Cursors.
- Batch processing.
- Temporary files.
- Object storage.

The database should return bounded amounts of data to each processing step.

---

# 61. Analytics Data

Operational databases should not automatically become the analytics warehouse.

Heavy analytical workloads should be isolated when they begin affecting transactional performance.

Potential architecture:

```text
Operational Database
        ↓
CDC / Events / ETL
        ↓
Analytics Store
```

Introduce additional infrastructure only when workload requirements justify it.

---

# 62. Database Security

Database access must follow least privilege.

Applications should have only the permissions they require.

Separate where appropriate:

- Application credentials.
- Migration credentials.
- Administrative credentials.
- Read-only credentials.
- Reporting credentials.

Never use database administrator credentials from normal application workloads.

---

# 63. Database Configuration

Database configuration must be environment-specific.

Examples:

```text
Development
Staging
Production
Self-hosted
```

Do not hardcode:

- Connection strings.
- Passwords.
- API credentials.
- Production hostnames.

Use the centralized configuration system defined by the application architecture.

---

# 64. Self-Hosted Deployments

KAMPYN may be deployed by universities or institutions in their own infrastructure.

Database architecture must therefore support:

- Institution-controlled credentials.
- Institution-controlled backups.
- Configurable database endpoints.
- Database migrations.
- Version compatibility.
- Upgrade procedures.
- Health checks.
- Recovery procedures.

Self-hosted installations must not depend on EXSOLVIA's internal database infrastructure.

---

# 65. Database Version Compatibility

Supported database versions must be explicitly documented.

The application must not depend on undocumented vendor-specific behavior unless the supported deployment architecture guarantees that behavior.

Database upgrades should be tested before production rollout.

---

# 66. Testing Database Changes

Database-related changes should test:

- Migrations.
- Constraints.
- Queries.
- Transactions.
- Rollbacks.
- Concurrent writes where relevant.
- Authorization boundaries.
- Tenant isolation.
- Pagination.
- Repository behavior.
- Failure scenarios.

Integration tests should use realistic database behavior where mocking would hide important semantics.

---

# 67. Database Change Checklist

Before completing a database-related change:

- [ ] Data owner is identified.
- [ ] Authoritative datastore is identified.
- [ ] Schema is appropriate for access patterns.
- [ ] Tenant isolation is preserved.
- [ ] Authorization boundaries are preserved.
- [ ] Constraints are defined where appropriate.
- [ ] Indexes are justified.
- [ ] Query shape is understood.
- [ ] N+1 behavior is checked.
- [ ] Pagination is bounded.
- [ ] Transaction boundary is explicit.
- [ ] Concurrency behavior is understood.
- [ ] Idempotency is considered.
- [ ] External side effects are outside database transactions where appropriate.
- [ ] Outbox/event propagation is considered.
- [ ] Cache invalidation is defined.
- [ ] OpenSearch synchronization is defined.
- [ ] Migration is version-controlled.
- [ ] Deployment compatibility is considered.
- [ ] Backfill impact is understood.
- [ ] Backup/recovery implications are understood.
- [ ] Observability is sufficient.
- [ ] Tests cover important invariants.
- [ ] Documentation reflects the new data model.

---

# 68. Final Database Principle

KAMPYN's database architecture must always make three things obvious:

```text
Where does the data live?
Who owns it?
What guarantees protect it?
```

The architecture should follow:

```text
                ┌─────────────────┐
                │   Application   │
                └────────┬────────┘
                         │
              ┌──────────┼──────────┐
              ↓          ↓          ↓
         PostgreSQL   MongoDB     Other
              │          │
              └─────┬────┘
                    │
              Authoritative
                  Data
                    │
          ┌─────────┴─────────┐
          ↓                   ↓
      OpenSearch            Redis
      Projection             Cache
          │                   │
          └─────────┬─────────┘
                    ↓
               Consumers
```

The system of record must remain authoritative.

Derived systems must be rebuildable.

Caches must be disposable.

Search indexes must be reconstructable.

Transactions must protect business invariants.

And every data boundary must preserve:

```text
Correctness
Isolation
Consistency
Performance
Recoverability
Operational clarity
```

The database architecture exists to protect the integrity of KAMPYN's business state, not merely to store data.
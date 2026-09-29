# PostgreSQL Engineering Standards

## 1. Purpose

PostgreSQL is KAMPYN's primary relational and transactional database. It is the default source of truth for structured business data, relationships, financial records, bookings, inventory transactions, user accounts, tenant configuration, and other workloads that require relational integrity and reliable transactions.

PostgreSQL must provide strong data consistency, enforceable constraints, predictable query performance, and safe concurrent access across KAMPYN's multi-tenant architecture.

These standards define how PostgreSQL schemas, tables, relationships, indexes, transactions, queries, migrations, security, performance, and operational infrastructure must be designed and maintained.

### Core Principles

- PostgreSQL is the default relational and transactional source of truth.
- Data integrity must be enforced at the database level wherever practical.
- Transactions must be explicit, short-lived, and correctly scoped.
- Schema design must prioritize correctness, clarity, and maintainability.
- Queries and indexes must be designed around real access patterns.
- Tenant isolation must be enforced consistently.
- Database access must be encapsulated behind repositories and domain services.
- Migrations must be version-controlled, reversible where practical, and production-safe.
- Concurrency must be handled explicitly to prevent lost updates, double bookings, and overselling.
- Performance optimizations must be based on measurements, query plans, and realistic workloads.
- Sensitive data must be protected through least privilege, encryption, and appropriate retention.
- PostgreSQL must integrate cleanly with Redis, MongoDB, OpenSearch, and event infrastructure without duplicating ownership.

---

## 2. Architectural Position

PostgreSQL is the authoritative relational datastore in KAMPYN's polyglot persistence architecture.

### Data Ownership

| System | Responsibility |
|---|---|
| PostgreSQL | Relational data, transactional records, business invariants, and authoritative structured state |
| MongoDB | Justified document-oriented workloads |
| Redis | Cache, ephemeral state, rate limiting, coordination, and transient data |
| OpenSearch | Derived full-text search, filtering, facets, and discovery |
| Object Storage | Images, documents, exports, and other binary assets |
| Backend Services | Business rules, authorization, transaction orchestration, and data access |

### Appropriate PostgreSQL Workloads

PostgreSQL SHOULD be used for:

- User profiles and account relationships.
- University and tenant configuration.
- Roles, permissions, and access-control relationships.
- Food courts, vendors, menus, and structured catalogue data.
- Orders, order items, and payment references.
- Inventory records and stock movements.
- Hostel, guest house, and shuttle bookings.
- Washing machine schedules and resource reservations.
- Library availability and structured resource allocation.
- Complaints, support requests, and workflow states.
- HR records and organizational relationships, subject to privacy controls.
- Audit records and durable business events where required.
- Subscription, billing, and invoicing records.
- Relational analytics and reporting data where appropriate.

These are illustrative domain boundaries. Final ownership must align with KAMPYN's approved domain model.

### Inappropriate Use

PostgreSQL MUST NOT be used as an unbounded replacement for:

- Redis-style high-frequency ephemeral caching.
- OpenSearch full-text discovery workloads.
- Object storage for large binary files.
- A general-purpose message broker.
- Unstructured document storage when a document database is materially more appropriate.

PostgreSQL may support specialized workloads such as JSONB storage, full-text search, and job queues where justified, but these capabilities must not be adopted by default when a dedicated system better fits the requirement.

---

## 3. Database Design Principles

### Schema-First Design

Every production table must have a documented purpose, ownership, and relationship to the domain model.

Before creating a table, define:

- The business entity it represents.
- Its authoritative owner.
- Its primary key and identity strategy.
- Its relationships and cardinality.
- Its tenant ownership, if applicable.
- Its lifecycle and deletion behavior.
- Its expected data volume and growth.
- Its primary read and write patterns.
- Its integrity constraints.
- Its indexing requirements.
- Its migration and retention requirements.

### Normalization

- Prefer normalized relational models for transactional business data.
- Use foreign keys to enforce relationships.
- Avoid unnecessary duplication of authoritative values.
- Denormalize only when there is a measurable performance or product requirement.
- Document ownership and synchronization rules for duplicated data.
- Avoid designs that require application code to maintain fragile cross-table invariants without database enforcement.

Third normal form is a useful baseline, not an inflexible requirement. Modeling decisions must account for domain semantics, access patterns, and operational cost.

### Entity Boundaries

- Represent independent business entities in separate tables when they have their own identity and lifecycle.
- Use junction tables for many-to-many relationships.
- Use child tables for repeated relational data.
- Avoid storing multiple independent entities in serialized text or JSON when they require relational querying, constraints, or independent updates.
- Keep tables cohesive and aligned with domain ownership.

### Naming Conventions

Use consistent, descriptive, lowercase `snake_case` naming.

Examples:

```sql
CREATE TABLE food_items (
    id UUID PRIMARY KEY,
    tenant_id UUID NOT NULL,
    vendor_id UUID NOT NULL,
    name TEXT NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

Rules:

- Use plural table names consistently.
- Use descriptive column names.
- Use `id` as the primary key column name unless a documented exception exists.
- Use `<entity>_id` for foreign keys.
- Use `created_at`, `updated_at`, and `deleted_at` consistently where applicable.
- Avoid reserved SQL keywords as identifiers.
- Avoid ambiguous abbreviations.
- Use explicit constraint and index names.
- Do not mix naming conventions across schemas.

---

## 4. Schema and Table Organization

PostgreSQL schemas may be used to organize database objects by domain, ownership, or operational boundaries.

### Rules

- Keep schema organization consistent across environments.
- Use schemas to create meaningful boundaries, not to compensate for unclear service ownership.
- Avoid creating a schema for every table or small feature.
- Clearly document cross-schema relationships and permissions.
- Separate application-owned objects from operational or extension-owned objects where appropriate.
- Use explicit schema qualification in migration and administrative scripts where ambiguity is possible.

An illustrative schema layout:

```text
kampyn
  users
  tenants
  memberships
  vendors
  food_courts
  food_items
  orders
  order_items
  inventory
  bookings
  complaints
  audit_logs
  outbox_events
```

This is a conceptual layout. It may be implemented using a single application schema or multiple domain schemas according to the approved repository and database architecture.

### Table Responsibilities

Each table must have:

- A clear entity or relationship purpose.
- A primary key.
- Appropriate `NOT NULL` constraints.
- Appropriate uniqueness constraints.
- Foreign keys where relationships require enforcement.
- Check constraints for valid local invariants.
- Explicit timestamp and lifecycle conventions.
- Documented indexes for important access patterns.

Avoid excessively wide tables that combine unrelated domain concerns.

---

## 5. Data Types

Use PostgreSQL data types that accurately represent the domain.

### Common Types

| Use Case | Recommended Type |
|---|---|
| Internal identifiers | `UUID` |
| Short bounded text | `VARCHAR(n)` where a meaningful limit exists |
| General text | `TEXT` |
| Boolean state | `BOOLEAN` |
| Integer counts | `INTEGER` or `BIGINT` |
| Monetary minor units | `BIGINT` |
| Decimal measurements | `NUMERIC(p, s)` |
| Event or creation time | `TIMESTAMPTZ` |
| Calendar date | `DATE` |
| Local time-of-day | `TIME` |
| Structured flexible metadata | `JSONB` |
| Binary data | External object storage by default |
| Enumerated finite state | `TEXT` with constraints or a governed PostgreSQL enum |
| IP addresses | `INET` |
| Geospatial data | PostGIS types when the extension is approved and required |

### Identifiers

- Use stable identifiers for persistent business entities.
- UUIDs are the default where globally unique identifiers are beneficial.
- Identity or sequence-backed integers may be used where they better fit a bounded internal dataset and are approved by the architecture.
- Never expose sensitive internal sequential identifiers as the sole authorization mechanism.
- Avoid changing primary keys after entity creation.
- Use consistent identifier types across related tables.

### Text

- Prefer `TEXT` when no meaningful maximum length exists.
- Use `VARCHAR(n)` when the limit is a genuine business rule.
- Enforce non-empty or normalized content through appropriate validation and database constraints.
- Avoid arbitrary string limits that do not reflect product requirements.
- Consider database collation and normalization behavior for case-insensitive uniqueness and sorting.

### Monetary Values

- Do not use `REAL` or `DOUBLE PRECISION` for exact financial amounts.
- Prefer integer minor units, such as paise for INR, where the currency's minor-unit convention is supported.
- Alternatively, use `NUMERIC(p, s)` with an explicitly defined precision and scale.
- Store currency explicitly for monetary values where multiple currencies are possible.
- Define rounding rules at the business layer.
- Avoid implicit currency conversions within generic persistence code.

Example:

```sql
CREATE TABLE order_items (
    id UUID PRIMARY KEY,
    order_id UUID NOT NULL,
    unit_price_minor BIGINT NOT NULL CHECK (unit_price_minor >= 0),
    quantity INTEGER NOT NULL CHECK (quantity > 0),
    currency CHAR(3) NOT NULL
);
```

The example uses integer minor units; actual currency validation must follow the supported currency configuration.

### Timestamps

- Use `TIMESTAMPTZ` for instants.
- Store timestamps in UTC-compatible form and render them in the user's or institution's intended timezone.
- Use `DATE` for calendar dates without time-of-day semantics.
- Use `TIME` only for local time-of-day values.
- Do not store timestamps as arbitrary strings.
- Use consistent timestamp precision unless a domain requires otherwise.
- Avoid mixing server-local time and UTC assumptions.

### JSONB

`JSONB` is permitted for flexible, document-like attributes where relational modeling would add unnecessary complexity.

Rules:

- Do not use JSONB to avoid designing relationships.
- Keep frequently queried, constrained, or joined attributes as normal columns.
- Validate JSONB structure at the application boundary and, where practical, through database constraints.
- Define versioning and migration behavior for persisted JSON structures.
- Avoid unbounded nested documents and uncontrolled keys.
- Index JSONB paths only when supported by measured query requirements.
- Do not store secrets or unnecessary personal information in JSONB.

---

## 6. Primary Keys, Foreign Keys, and Constraints

Database constraints are a core part of data correctness.

### Primary Keys

- Every persistent entity table MUST have a primary key.
- Primary keys MUST be stable and immutable.
- Primary key type must be consistent across related tables.
- Avoid composite primary keys for entities unless they clearly express the natural identity of the relationship.
- Use surrogate keys for independently addressable entities when appropriate.

### Foreign Keys

- Use foreign keys for relationships that must remain valid.
- Choose `ON DELETE` behavior explicitly.
- Prefer `RESTRICT` or `NO ACTION` for important business entities where cascading deletion could destroy valuable records.
- Use `CASCADE` only when child records are truly dependent and their deletion is intended.
- Avoid application-only referential integrity for ordinary relational relationships.
- Index referencing columns when query patterns or deletion/update operations require it.

### Unique Constraints

Use unique constraints for business identifiers and invariants such as:

- Tenant-specific vendor codes.
- Tenant-scoped usernames where required.
- External integration identifiers.
- Idempotency keys within their defined scope.
- Unique resource slots where the domain requires exclusivity.

Example:

```sql
ALTER TABLE vendors
ADD CONSTRAINT uq_vendors_tenant_code
UNIQUE (tenant_id, vendor_code);
```

### Check Constraints

Use `CHECK` constraints for local invariants that can be expressed safely within a row.

Example:

```sql
ALTER TABLE bookings
ADD CONSTRAINT chk_booking_valid_range
CHECK (ends_at > starts_at);
```

Use database constraints for essential invariants that must hold regardless of which service or process writes the data.

### Constraint Naming

Use predictable names, such as:

```text
pk_<table>
fk_<table>_<referenced_table>
uq_<table>_<columns>
chk_<table>_<rule>
idx_<table>_<columns>
```

Constraint names must be unique within the applicable PostgreSQL namespace and remain readable in error messages and migration scripts.

---

## 7. Multi-Tenancy

KAMPYN must support multiple universities with secure and predictable tenant isolation.

### Tenant Ownership

Every tenant-scoped table MUST include a `tenant_id` column unless the entity is explicitly global and that exception is documented.

Example:

```sql
CREATE TABLE vendors (
    id UUID PRIMARY KEY,
    tenant_id UUID NOT NULL,
    name TEXT NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (tenant_id, id)
);
```

### Tenant Isolation Rules

- Tenant context must be resolved from trusted authentication and authorization logic.
- Do not rely exclusively on a client-supplied `tenant_id`.
- Tenant-scoped reads and writes must always be constrained to the authorized tenant.
- Foreign-key relationships between tenant-scoped entities must prevent cross-tenant references.
- Unique constraints must include `tenant_id` when uniqueness is tenant-local.
- Background jobs and event consumers must preserve tenant context.
- Administrative cross-tenant access must be explicit, permission-checked, and audited.
- Tenant deletion and data export workflows must be documented.
- Tenant boundaries must be tested in repositories, services, and database policies.

### Composite Tenant-Safe Foreign Keys

Where appropriate, use composite unique keys and foreign keys to ensure that related records belong to the same tenant.

Example:

```sql
CREATE TABLE vendors (
    id UUID NOT NULL,
    tenant_id UUID NOT NULL,
    name TEXT NOT NULL,
    PRIMARY KEY (id),
    UNIQUE (tenant_id, id)
);

CREATE TABLE food_items (
    id UUID PRIMARY KEY,
    tenant_id UUID NOT NULL,
    vendor_id UUID NOT NULL,
    name TEXT NOT NULL,
    CONSTRAINT fk_food_items_vendor
        FOREIGN KEY (tenant_id, vendor_id)
        REFERENCES vendors (tenant_id, id)
);
```

This prevents a food item from referencing a vendor belonging to another tenant.

### Row-Level Security

PostgreSQL Row-Level Security (RLS) MAY be used as an additional tenant-isolation layer.

When enabled:

- Define explicit policies for every applicable operation.
- Use transaction-scoped tenant context.
- Ensure connection pooling does not leak tenant context between requests.
- Set tenant context safely using transaction-local settings where applicable.
- Test missing, invalid, and cross-tenant contexts.
- Define how migrations, administrative tasks, and background jobs operate under RLS.
- Do not treat RLS as a replacement for application authorization.

Example policy pattern:

```sql
ALTER TABLE vendors ENABLE ROW LEVEL SECURITY;

CREATE POLICY vendors_tenant_policy
ON vendors
USING (
    tenant_id = current_setting('app.tenant_id', true)::UUID
)
WITH CHECK (
    tenant_id = current_setting('app.tenant_id', true)::UUID
);
```

The application must establish `app.tenant_id` securely within each transaction. Roles with `BYPASSRLS`, table ownership, and other privileged access paths require explicit review.

### Tenant Isolation Models

Possible deployment models include:

- Shared database and shared schema with tenant-scoped rows.
- Dedicated schemas for selected tenants.
- Dedicated databases for tenants requiring stronger isolation.
- Dedicated infrastructure for universities operating self-hosted installations.

The default model must be chosen according to operational scale, isolation requirements, backup and recovery needs, and self-hosting constraints.

Do not introduce one database per tenant by default without documenting provisioning, migration, monitoring, and recovery costs.

---

## 8. Relationships and Data Integrity

Relationships must accurately reflect business ownership and lifecycle.

### Relationship Rules

- Use one-to-one relationships only when the domain requires a separate lifecycle or security boundary.
- Use one-to-many relationships for dependent or repeated entities.
- Use junction tables for many-to-many relationships.
- Define ownership for child records.
- Avoid circular dependencies unless there is a clear domain justification.
- Use foreign keys and uniqueness constraints to enforce relationship integrity.
- Document deletion and archival behavior for related records.

### Referential Integrity

- A record must not reference a nonexistent parent where the relationship is mandatory.
- Tenant-scoped relationships must enforce tenant consistency.
- Soft deletion must not be mistaken for physical referential integrity.
- Historical records must retain required references or immutable snapshots where the domain demands it.
- Cross-domain relationships must have clear ownership and update semantics.

### Historical Snapshots

For orders, invoices, and other historical records, store the values required to preserve the original transaction context when those values may change later.

For example, an order item may retain the product name and price at purchase time rather than depending entirely on the current catalogue entry.

Snapshots must be intentional, limited to required fields, and clearly distinguished from the current authoritative entity.

---

## 9. Transactions and ACID Guarantees

PostgreSQL transactions must be used to protect related business operations that must succeed or fail together.

### Transaction Requirements

- Keep transactions as short as practical.
- Do not perform external network requests while holding database locks unless explicitly justified.
- Use appropriate isolation levels.
- Roll back the full transaction on failure.
- Avoid unnecessary nested transaction abstractions.
- Use parameterized queries.
- Ensure transaction-scoped tenant context is established before accessing tenant-protected data.
- Keep transaction boundaries within the appropriate application service or unit of work.

### Transaction Scope

A transaction should contain only the database operations needed to preserve a defined business invariant.

Example: creating an order may require atomically inserting the order, its line items, and an outbox event.

It should not keep the database transaction open while waiting for an external payment provider.

### Isolation Levels

| Isolation Level | Typical Use |
|---|---|
| `READ COMMITTED` | Default for most routine transactional operations |
| `REPEATABLE READ` | Operations requiring a consistent transaction snapshot |
| `SERIALIZABLE` | Critical operations where serializable behavior is needed and retries are handled |

Use the least restrictive isolation level that correctly protects the business invariant.

### Serializable Retries

When using `SERIALIZABLE`:

- Detect serialization failures.
- Retry the entire transaction where safe.
- Apply a bounded retry policy.
- Use idempotency protections for operations that may be retried.
- Avoid retrying operations that have already caused external side effects.

### Transactional Outbox

Use a transactional outbox where database changes must reliably produce events.

The business update and outbox event insertion must commit in the same transaction.

Example:

```sql
BEGIN;

UPDATE food_items
SET name = 'Paneer Curry',
    updated_at = NOW()
WHERE id = $1
  AND tenant_id = $2;

INSERT INTO outbox_events (
    id,
    tenant_id,
    event_type,
    aggregate_id,
    payload,
    created_at
)
VALUES (
    $3,
    $2,
    'food_item.updated',
    $1,
    $4::JSONB,
    NOW()
);

COMMIT;
```

Outbox delivery must be handled by a separate reliable publisher. Refer to `architecture/events.md` and `backend/background-jobs.md`.

---

## 10. Concurrency and Locking

Concurrent writes must preserve business invariants.

### General Rules

- Prefer atomic SQL updates over application-side read-modify-write patterns.
- Use optimistic concurrency where appropriate.
- Use row-level locks for critical workflows that require serialized access.
- Keep lock duration short.
- Acquire locks in a consistent order where multiple resources are involved.
- Define deadlock handling and transaction retry behavior.
- Avoid locking more rows than required.
- Never assume that a prior read guarantees the same data remains valid at write time.

### Atomic Updates

Prefer:

```sql
UPDATE inventory
SET available_quantity = available_quantity - $1,
    updated_at = NOW()
WHERE tenant_id = $2
  AND id = $3
  AND available_quantity >= $1
RETURNING available_quantity;
```

This pattern makes the stock deduction conditional and atomic.

The service must check whether a row was returned and handle insufficient stock explicitly.

### Optimistic Concurrency

Use a version column where concurrent edits must be detected.

Example:

```sql
UPDATE food_items
SET name = $1,
    version = version + 1,
    updated_at = NOW()
WHERE id = $2
  AND tenant_id = $3
  AND version = $4
RETURNING version;
```

If no row is returned, the service must handle a stale version or missing entity.

### Pessimistic Locking

Use `SELECT ... FOR UPDATE` or related locking clauses only where necessary.

- Lock the smallest relevant row set.
- Avoid long-running application work while locks are held.
- Apply deterministic lock ordering.
- Define timeout and retry behavior.
- Monitor lock waits and deadlocks.

### Booking and Resource Allocation

For exclusive time slots and resources:

- Enforce overlap prevention through appropriate database constraints or locking strategies.
- Use transactions to check and claim resources atomically.
- Do not rely on checking availability in a separate request and assuming it remains available.
- Consider PostgreSQL exclusion constraints for overlapping time ranges where appropriate and supported.
- Use authoritative reservation records for booking confirmation.

### Inventory

- Use atomic stock updates or explicit reservation records.
- Keep stock movements auditable.
- Define reservation expiry and release behavior.
- Prevent overselling through database-enforced concurrency controls.
- Avoid treating cached inventory as authoritative.

---

## 11. Indexing Standards

Indexes must be driven by actual query patterns, constraints, and measured performance.

### General Rules

- Index columns used frequently in selective filters, joins, ordering, or uniqueness checks.
- Avoid creating indexes without a known access pattern.
- Consider write amplification and storage costs.
- Review index effectiveness using query plans and production metrics.
- Remove redundant or unused indexes through controlled changes.
- Include tenant context in index design for tenant-scoped access patterns.
- Ensure foreign-key referencing columns have appropriate indexes where workload requires them.

### Index Types

| Index Type | Typical Use |
|---|---|
| B-tree | Equality, range, sorting, and common relational queries |
| GIN | JSONB containment, arrays, and supported full-text search |
| GiST | Range, geometric, and specialized search operators |
| BRIN | Large naturally ordered datasets where block summaries are effective |
| Hash | Specialized equality workloads where measured benefits justify use |

B-tree is the default index type unless the query operators or workload justify another type.

### Composite Indexes

Composite indexes must follow actual query predicates and ordering requirements.

Example:

```sql
CREATE INDEX idx_orders_tenant_status_created
ON orders (tenant_id, status, created_at DESC);
```

Column order matters. Consider equality filters, range filters, sort direction, and selectivity before selecting an index structure.

### Partial Indexes

Use partial indexes when queries repeatedly target a stable subset of records.

Example:

```sql
CREATE INDEX idx_active_bookings_resource_start
ON bookings (tenant_id, resource_id, starts_at)
WHERE status = 'confirmed';
```

The query predicate must be compatible with the partial index condition for the planner to use it.

### Covering Indexes

Use `INCLUDE` columns when they reduce heap access for important, measured queries.

Avoid overly wide covering indexes that increase write and storage costs.

### JSONB Indexes

- Use GIN or expression indexes only for JSONB paths that are queried frequently.
- Prefer relational columns for frequently filtered, joined, or constrained attributes.
- Choose the GIN operator class based on actual query operators.
- Avoid indiscriminate indexing of entire JSON documents.

### Index Lifecycle

- Use version-controlled migrations for index changes.
- Use `CREATE INDEX CONCURRENTLY` for suitable production index creation where supported by the migration process.
- Account for the restrictions of concurrent index operations within transactions.
- Monitor index build progress and failure states.
- Verify query-plan improvement after deployment.
- Remove old indexes only after confirming they are not needed by other queries.

Refer to `data/indexing.md` for detailed indexing standards.

---

## 12. Query Design and Optimization

Queries must be correct, bounded, and efficient for expected production workloads.

### Query Requirements

- Use parameterized queries.
- Select only the columns required by the operation.
- Filter as early as practical.
- Avoid retrieving large result sets when pagination or aggregation is sufficient.
- Use joins where relational semantics are appropriate.
- Avoid N+1 query patterns.
- Avoid unnecessary correlated subqueries where a clearer or more efficient alternative exists.
- Use database-side aggregation for suitable workloads.
- Avoid fetching entire tables for application-side filtering.
- Use explicit ordering when deterministic results are required.
- Bound long-running or user-controlled queries.

### Avoiding N+1 Queries

- Identify relationships needed by the operation.
- Use suitable joins or batched queries.
- Avoid blindly joining multiple one-to-many relationships when this causes row multiplication.
- Measure the trade-off between joins and separate batched retrieval.
- Keep response assembly predictable.

### Pagination

For shallow pagination, bounded `LIMIT` and `OFFSET` may be acceptable.

For large datasets, prefer keyset pagination.

Example:

```sql
SELECT id, created_at, status
FROM orders
WHERE tenant_id = $1
  AND (created_at, id) < ($2, $3)
ORDER BY created_at DESC, id DESC
LIMIT $4;
```

Rules:

- Use a stable, deterministic sort.
- Include a unique tie-breaker.
- Ensure pagination predicates align with supporting indexes.
- Validate page size.
- Avoid unbounded `OFFSET`.
- Define cursor format, validation, and expiration where applicable.

### Query Plans

Use `EXPLAIN` and, where safe, `EXPLAIN (ANALYZE, BUFFERS)` to investigate important queries.

- Compare estimated and actual row counts.
- Inspect sequential scans and index scans in context.
- Identify expensive sorts, joins, and aggregations.
- Check buffer reads and temporary-file usage.
- Test with representative data volumes and distributions.
- Do not force planner behavior without evidence and documentation.

### Query Timeouts

- Set appropriate statement and lock timeouts for application workloads.
- Use stricter bounds for interactive endpoints where appropriate.
- Treat timeouts as explicit errors, not successful empty results.
- Monitor recurring timeout patterns and investigate the underlying query or capacity issue.

---

## 13. Partitioning and Large Tables

Partitioning may be used when table size, retention, maintenance, or query patterns justify the additional complexity.

### Appropriate Use Cases

- High-volume append-oriented event records.
- Large audit or activity tables.
- Time-series business records with predictable retention.
- Large datasets where partition pruning benefits common queries.
- Tables requiring partition-level archival or deletion.

### Partitioning Rules

- Define the partition key based on real access and lifecycle patterns.
- Ensure primary-key and uniqueness constraints are compatible with the partitioning design.
- Include partition-key filters in important queries where practical.
- Automate partition creation and retention maintenance.
- Monitor partition count and query planning overhead.
- Test inserts that cross partition boundaries.
- Document handling for late-arriving and backfilled records.
- Avoid partitioning small tables without a measurable benefit.

Partitioning does not replace indexing, query optimization, or appropriate data modeling.

---

## 14. Database Access Layer

All PostgreSQL access must follow KAMPYN's approved backend architecture.

### Layer Responsibilities

| Layer | Responsibility |
|---|---|
| API Handler | Request parsing, authentication context, response mapping |
| Application Service | Use-case orchestration and transaction boundaries |
| Domain Logic | Business invariants and domain behavior |
| Repository | Encapsulated database reads and writes |
| Database Client | Connection pooling, driver configuration, and low-level access |
| Migration System | Schema evolution and data migrations |

### Rules

- Do not place complex SQL directly in HTTP handlers.
- Keep repositories focused on a specific domain or aggregate.
- Keep business rules out of generic database client wrappers.
- Avoid generic repositories that obscure important domain operations.
- Use explicit transaction boundaries for multi-step business operations.
- Avoid leaking database-specific details through unrelated domain layers.
- Reuse approved connection pools and transaction utilities.
- Do not create a new database connection for every request.

### ORM and Query Builder Usage

If an ORM or query builder is used:

- Treat migrations and schema definitions as version-controlled artifacts.
- Inspect generated SQL for critical queries.
- Use raw SQL when it is materially clearer or more efficient, with parameterization.
- Avoid hidden N+1 behavior and implicit unbounded relation loading.
- Ensure transaction and locking semantics are understood.
- Do not let ORM convenience override database integrity or performance requirements.

---

## 15. Connection Pooling

Database connections are a finite resource and must be managed centrally.

### Requirements

- Use a shared connection pool per service process or approved database access boundary.
- Configure pool size based on database capacity and service concurrency.
- Set connection acquisition and query timeouts.
- Monitor active, idle, waiting, and rejected connections.
- Use TLS for network connections in production.
- Handle connection interruptions and pool recovery.
- Avoid holding idle connections inside long-running transactions.
- Ensure pool sizing across all service replicas stays within the database connection budget.

### Pool Sizing

Do not independently maximize the pool size of every service.

Capacity planning must account for:

- Maximum service replicas.
- Worker concurrency.
- Administrative connections.
- Migration connections.
- Monitoring and maintenance sessions.
- PostgreSQL's configured connection limit.
- Any connection proxy or pooler.

A connection pooler such as PgBouncer MAY be used where the workload and deployment model justify it. Transaction pooling requires careful review of session state, temporary tables, prepared statements, advisory locks, and other session-dependent features.

---

## 16. Migrations and Schema Evolution

All production schema changes must be version-controlled and applied through a governed migration process.

### Migration Requirements

- Every schema change must have a migration.
- Migrations must be deterministic and safe to rerun only when explicitly designed to be idempotent.
- Migrations must be reviewed before production deployment.
- Destructive changes require explicit approval and a recovery plan.
- Migrations must be tested against realistic data volumes where operational risk warrants it.
- Migration ownership and execution must be clear for SaaS and self-hosted deployments.
- Avoid relying on manual production schema edits.

### Expand-and-Contract

For changes that may affect active application versions:

1. Add compatible schema elements.
2. Deploy code that supports both old and new representations where needed.
3. Backfill existing data in bounded batches.
4. Validate consistency.
5. Switch application reads and writes.
6. Remove obsolete schema elements in a later approved migration.

### Migration Safety

- Avoid long-running table locks where possible.
- Use concurrent index operations when suitable.
- Plan large backfills as resumable jobs rather than one enormous transaction.
- Monitor migration progress, locks, and replication lag where applicable.
- Define timeout and failure-handling behavior.
- Maintain rollback or forward-recovery instructions.
- Coordinate schema changes with dependent services and SDK/API contracts.

### Production Migration Process

- Validate the migration in development and staging.
- Review lock behavior and expected execution time.
- Confirm backup and recovery readiness for high-risk changes.
- Deploy compatible application code.
- Apply the migration using the approved release process.
- Verify schema state and application health.
- Record the applied version.
- Monitor for regressions.

Refer to `data/migrations.md` for comprehensive migration governance.

---

## 17. Transactions for KAMPYN Domain Workloads

Different KAMPYN workflows require explicit consistency guarantees.

### Orders

Order creation may involve:

- Creating the order.
- Creating order items.
- Capturing price snapshots.
- Recording an idempotency key.
- Reserving or adjusting inventory when the design requires it.
- Creating an outbox event.

Operations that must be atomic must occur in one transaction. External payment processing must be coordinated through a separate, idempotent workflow rather than an open database transaction.

### Payments

- Store payment references and authoritative payment state according to the payment architecture.
- Enforce uniqueness for provider transaction IDs and idempotency keys within their correct scope.
- Record state transitions explicitly.
- Handle provider callbacks idempotently.
- Do not assume a client-side payment result is authoritative.
- Keep sensitive payment credentials and card data out of ordinary application tables unless a compliant, explicitly approved design requires otherwise.

### Inventory

- Represent stock adjustments through explicit, auditable operations.
- Use atomic conditional updates or reservation models.
- Prevent negative inventory where the business rules prohibit it.
- Handle concurrent requests without relying on stale reads.
- Keep inventory movements traceable.

### Bookings

- Enforce resource and time-slot consistency.
- Prevent overlapping confirmed reservations where required.
- Use database constraints, locking, or another proven concurrency strategy.
- Model cancellation, expiry, and confirmation as explicit state transitions.
- Make retries idempotent.

### Complaints and Workflows

- Store state transitions in a controlled manner.
- Preserve the history required for audit and support.
- Enforce valid state transitions in the domain layer and use database constraints for applicable local invariants.
- Use outbox events for reliable downstream notifications.

---

## 18. Data Lifecycle and Deletion

Every entity must have a documented lifecycle.

### Deletion Models

Choose deliberately between:

- Hard deletion.
- Soft deletion.
- Archival.
- Retention-limited records.
- Anonymization or pseudonymization where appropriate.

### Rules

- Do not add `deleted_at` to every table by default.
- Use soft deletion when restoration, auditability, or business retention requires it.
- Ensure queries consistently exclude soft-deleted records where appropriate.
- Define unique-constraint behavior for soft-deleted entities.
- Document cascade behavior and dependent record retention.
- Ensure deletion propagates to derived systems such as OpenSearch and relevant caches.
- Support tenant data export and deletion workflows where required.
- Do not remove historical financial or audit records without an approved retention and compliance basis.

### Archival

- Archive only where lifecycle and query requirements justify it.
- Preserve integrity and traceability across active and archived records.
- Document how archived data can be queried or restored.
- Avoid archive mechanisms that silently break foreign-key or audit requirements.

---

## 19. Security and Access Control

PostgreSQL must be treated as a critical security boundary.

### Authentication

- Use strong database authentication.
- Use separate identities for applications, migrations, monitoring, and administration.
- Store credentials in an approved secret-management system.
- Rotate credentials according to security policy.
- Use TLS for production connections.
- Never commit credentials or expose them in logs.

### Authorization

- Follow least-privilege access.
- Grant only required schema, table, sequence, and function permissions.
- Avoid using superuser accounts for application traffic.
- Restrict schema modification privileges to migration identities.
- Separate read-only reporting access from write-capable application access.
- Review service permissions as domain ownership evolves.

### Sensitive Data

- Classify sensitive fields.
- Store only data required for an approved purpose.
- Encrypt sensitive data where required by the security architecture.
- Apply field-level access controls in application services.
- Avoid unnecessary replication of personal data to caches, logs, analytics, and search indices.
- Define retention, export, and deletion policies.
- Protect backups and replicas to the same data classification as the source.

### SQL Injection Prevention

- Use parameterized queries.
- Do not concatenate untrusted input into SQL statements.
- Use allowlists for dynamic identifiers such as table names, column names, and sort directions.
- Validate query filters and pagination parameters.
- Review raw SQL and dynamic query generation carefully.

### Auditing

Audit security-relevant and high-integrity operations according to the platform's audit policy.

Audit design must consider:

- Actor identity.
- Tenant context.
- Operation type.
- Target entity.
- Timestamp.
- Correlation or request identifier.
- Relevant state changes, subject to data minimization.
- Tamper-resistance and retention requirements.

Do not treat ordinary application logs as a substitute for durable business audit records.

---

## 20. Reliability and Resilience

PostgreSQL must remain reliable under expected operational failures.

### Requirements

- Use managed database services or a documented self-managed high-availability design.
- Configure health checks and connection recovery.
- Monitor database availability and resource utilization.
- Define backup and restore procedures.
- Configure replication and failover according to the required service objectives.
- Ensure applications handle transient connection failures safely.
- Avoid unbounded retry loops.
- Use idempotency protections for retryable writes.
- Define operational ownership and escalation procedures.

### Retry Rules

Retries may be appropriate for transient errors, but:

- Retry only known retryable failures.
- Use bounded exponential backoff with jitter.
- Respect transaction boundaries.
- Retry the entire transaction when its consistency guarantees require it.
- Avoid repeating external side effects.
- Use idempotency keys or equivalent safeguards for operations that may be repeated.
- Do not retry permanent schema, validation, authorization, or constraint errors blindly.

### Failover

- Define connection recovery behavior.
- Ensure clients can reconnect after failover.
- Account for in-flight transactions being aborted.
- Reconcile ambiguous write outcomes using idempotency or authoritative reads.
- Test failover procedures.
- Document read-replica lag and its effect on user-facing features.

---

## 21. Replication and Read Scaling

Read replicas MAY be introduced when read workloads justify them and consistency requirements permit.

### Rules

- The primary remains authoritative for writes.
- Replica reads must account for replication lag.
- Critical read-after-write flows must use the primary or an explicitly consistent strategy.
- Do not use replicas for decisions requiring the latest committed state without validating consistency.
- Define routing rules for read and write workloads.
- Monitor replication lag and replica health.
- Ensure failover and replica promotion procedures are documented.
- Avoid introducing replicas before measuring primary workload bottlenecks.

### Read Routing

Suitable replica workloads may include:

- Non-critical catalogue browsing.
- Historical reporting.
- Analytics queries.
- Other read-heavy use cases that tolerate documented staleness.

Transactions, inventory decisions, booking confirmation, and payment-state transitions must use an authoritative consistency strategy.

---

## 22. Performance and Capacity Planning

Performance engineering must be evidence-based.

### Database Metrics

Monitor:

- CPU and memory utilization.
- Active and idle connections.
- Connection wait time.
- Transaction throughput.
- Query latency percentiles.
- Slow queries.
- Lock waits and deadlocks.
- Cache hit ratios.
- Temporary file usage.
- Table and index growth.
- Vacuum and analyze activity.
- Replication lag.
- WAL generation and retention.
- Disk throughput and latency.
- Checkpoint behavior.
- Database availability.

### Query Performance

- Identify high-volume and high-latency queries.
- Inspect execution plans.
- Validate index effectiveness.
- Reduce unnecessary round trips.
- Bound result sets.
- Avoid unbounded aggregation.
- Reassess query patterns as data volume increases.
- Benchmark changes with realistic datasets.

### Write Performance

- Batch writes when doing so preserves correctness and improves throughput.
- Keep transactions short.
- Avoid excessive index count.
- Avoid updating rows unnecessarily.
- Consider contention around frequently updated records.
- Use appropriate bulk-loading approaches for controlled imports.
- Monitor WAL and replication impact.

### Growth Planning

Capacity planning must account for:

- Tenant and user growth.
- Transaction volume.
- Order and booking retention.
- Index and table growth.
- Peak concurrent activity.
- Background jobs and backfills.
- Analytics workload.
- Replication and backup overhead.
- Failover capacity.
- Self-hosted deployment constraints.

Do not use premature sharding as a default solution to database scaling.

---

## 23. Vacuum, Statistics, and Maintenance

PostgreSQL maintenance is required for stable performance and storage management.

### Requirements

- Ensure autovacuum is enabled and appropriately configured.
- Monitor dead tuples and vacuum activity.
- Monitor table and index bloat.
- Ensure statistics are sufficiently current for query planning.
- Use `ANALYZE` where appropriate after significant data changes.
- Avoid unnecessary manual vacuum operations during peak workloads.
- Investigate long-running transactions that prevent cleanup.
- Define maintenance windows for disruptive operations.
- Review maintenance behavior as high-write tables grow.

### Operational Considerations

- Monitor transaction ID age and prevent transaction ID wraparound.
- Monitor replication slots and retained WAL.
- Investigate unexpectedly large temporary files.
- Avoid leaving idle-in-transaction sessions open.
- Review checkpoint and WAL settings against actual workload.
- Document database-specific tuning and why it is required.

Configuration changes must be measured and tested rather than copied from generic tuning guides.

---

## 24. Partitioning, Sharding, and Scaling Strategy

Scaling must proceed in stages based on observed constraints.

### Preferred Scaling Sequence

1. Correct data modeling and integrity constraints.
2. Optimize expensive queries.
3. Add or adjust appropriate indexes.
4. Improve connection pooling and transaction behavior.
5. Scale compute, memory, and storage.
6. Introduce read replicas where consistency allows.
7. Partition large tables where access and lifecycle patterns justify it.
8. Evaluate workload separation or dedicated databases.
9. Consider sharding only when simpler approaches are insufficient and operational readiness is established.

### Sharding

Sharding introduces complexity in:

- Cross-shard relationships.
- Transaction boundaries.
- Unique constraints.
- Query routing.
- Tenant migration.
- Rebalancing.
- Backup and restore.
- Observability and operations.

Do not introduce sharding without a documented design, data ownership boundaries, shard-key strategy, migration plan, and operational capability.

### Tenant-Aware Scaling

For KAMPYN's SaaS model:

- Track data volume and workload per tenant.
- Identify unusually large or high-traffic tenants.
- Keep tenant context available for query routing and observability.
- Design tenant movement or isolation strategies before scale requires urgent intervention.
- Avoid assumptions that all universities have equivalent usage patterns.

---

## 25. Extensions and Specialized Features

PostgreSQL extensions may be used when they solve a documented requirement.

### Rules

- Approve extensions through architecture and security review.
- Pin and validate supported extension versions.
- Confirm availability in managed and self-hosted deployments.
- Document operational and backup implications.
- Avoid extensions that create unnecessary deployment coupling.

### Potential Extensions

| Extension | Potential Purpose |
|---|---|
| PostGIS | Geospatial queries and location-aware discovery |
| `pg_stat_statements` | Query performance monitoring |
| `pg_trgm` | Similarity search and trigram indexing |
| `pgcrypto` | Cryptographic functions where explicitly approved |
| `uuid-ossp` | UUID generation for compatible legacy or specialized needs |

These are optional examples, not mandatory dependencies. OpenSearch remains the preferred engine for KAMPYN's broader full-text search and discovery requirements.

---

## 26. PostgreSQL and Other Data Systems

PostgreSQL must integrate with other data systems through explicit ownership and synchronization contracts.

### PostgreSQL and MongoDB

- Do not duplicate authoritative ownership of the same entity without a documented synchronization design.
- Define which store owns each domain entity.
- Use events or controlled synchronization for cross-store projections.
- Avoid assuming a transaction can span both databases atomically.
- Use idempotency, reconciliation, and compensating workflows where necessary.

### PostgreSQL and Redis

- PostgreSQL remains authoritative for durable business state.
- Redis may cache frequently accessed values or maintain ephemeral coordination state.
- Define cache invalidation or expiration rules.
- Avoid treating cached values as authoritative for inventory, payment, or booking decisions.
- Use appropriate consistency and idempotency controls when Redis coordinates work.

### PostgreSQL and OpenSearch

- OpenSearch indices must be derived from authoritative data.
- Use a reliable event or outbox mechanism for synchronization.
- Track indexing lag and failures.
- Support full reindexing and reconciliation.
- Apply tenant and visibility controls consistently.
- Validate critical actions against PostgreSQL or the appropriate authoritative service.

### PostgreSQL and Object Storage

- Store object metadata and ownership references in PostgreSQL where appropriate.
- Store large binary content in object storage.
- Define upload completion, cleanup, and orphaned-object handling.
- Do not assume database and object-storage operations share a transaction.
- Use durable workflows to reconcile partial failures.

---

## 27. Background Jobs and Bulk Operations

Background processing must not compromise database stability.

### Rules

- Use bounded worker concurrency.
- Process large datasets in manageable batches.
- Keep transactions short.
- Avoid loading unbounded datasets into application memory.
- Use keyset pagination or appropriate cursors for large scans.
- Apply backpressure when the database is under load.
- Make jobs restartable and idempotent.
- Track job progress, failures, and retry counts.
- Avoid holding row locks while performing slow external work.
- Use `SKIP LOCKED` only where its queue-processing semantics are appropriate.

### Bulk Imports

- Validate data before writing.
- Use staging tables or controlled import workflows for large ingestion tasks.
- Apply constraints and reconciliation checks.
- Define duplicate-handling behavior.
- Use bounded batches.
- Track rejected rows and reasons.
- Avoid disabling integrity constraints without an approved recovery and validation process.

### Data Backfills

- Make backfills resumable.
- Store progress checkpoints.
- Limit concurrent database load.
- Monitor lock and replication impact.
- Support safe retries.
- Validate results against the source.
- Document how to pause, resume, and roll back or repair the backfill.

---

## 28. Backup and Disaster Recovery

PostgreSQL backup and recovery must be defined according to the criticality of each workload.

### Requirements

- Define recovery time objectives (RTO) and recovery point objectives (RPO).
- Use automated backups.
- Configure point-in-time recovery where required and supported.
- Protect backups with encryption and access controls.
- Store backups separately from the primary failure domain where appropriate.
- Define retention policies.
- Test restoration periodically.
- Document failover and disaster-recovery procedures.
- Monitor backup completion and recovery readiness.
- Validate data integrity after restoration.

### Recovery Testing

A recovery exercise should verify:

- Database restoration completes.
- Required schemas, extensions, and permissions are present.
- Application services can reconnect.
- Tenant isolation remains intact.
- Critical transactions and relationships remain valid.
- Event publication and downstream synchronization resume safely.
- Recovery objectives are met or gaps are documented.

A backup is not considered operationally sufficient until its restoration has been tested.

---

## 29. Observability, Logging, and Auditing

Database operations must be observable without exposing sensitive data.

### Metrics

Monitor:

- Query count and latency.
- Transactions per second.
- Connection pool utilization.
- Active and idle sessions.
- Lock waits and deadlocks.
- Cache and buffer behavior.
- WAL and checkpoint activity.
- Replication lag.
- Vacuum activity.
- Table and index growth.
- Disk utilization.
- Backup and restore status.
- Database error rates.

### Logging

- Use structured application logs.
- Include request or correlation identifiers where appropriate.
- Record query duration and operation context without logging secrets.
- Avoid logging complete sensitive SQL parameters.
- Apply sampling or thresholds for high-volume query logs.
- Restrict access to database logs.
- Define retention based on operational and compliance needs.

### Tracing

- Propagate trace context across service and database operations where supported.
- Attribute query latency to the originating request or job.
- Avoid exposing sensitive SQL values in traces.
- Use tracing to identify expensive database round trips and transaction bottlenecks.

### Alerting

Alerts SHOULD cover:

- Database unavailability.
- Connection exhaustion.
- Sustained high query latency.
- Excessive lock waits or deadlocks.
- Disk capacity thresholds.
- Replication lag beyond service objectives.
- Failed backups.
- Transaction ID age risks.
- Abnormal WAL retention.
- Persistent autovacuum issues.
- Unexpected database growth.

---

## 30. Testing Standards

Database-backed features must include tests for correctness, concurrency, security, and performance where relevant.

### Unit Tests

Test:

- Domain validation.
- Query parameter construction.
- Repository input handling.
- Transaction orchestration.
- Error mapping.
- Idempotency logic.
- State-transition rules.

### Integration Tests

Test against a real PostgreSQL-compatible environment:

- Migrations from an empty database.
- Upgrade migrations from supported prior versions.
- Foreign-key constraints.
- Unique and check constraints.
- Transaction commit and rollback.
- Repository behavior.
- Tenant isolation.
- RLS policies, if enabled.
- JSONB validation and queries where used.
- Index creation and important query behavior.
- Outbox persistence and event recovery.
- Soft deletion and archival semantics.

Mocks must not be the only test of SQL behavior, constraint enforcement, locking, or transaction semantics.

### Concurrency Tests

Test critical concurrent workflows, including:

- Simultaneous inventory deductions.
- Concurrent booking attempts for the same resource.
- Duplicate payment callbacks.
- Repeated order submissions.
- Concurrent entity updates.
- Transaction retry behavior.
- Deadlock and serialization failure handling.

Tests must verify the final persisted state, not just successful HTTP responses.

### Performance Tests

Benchmark critical queries and write paths using representative:

- Row counts.
- Tenant distributions.
- Data cardinality.
- Query concurrency.
- Index configurations.
- Transaction durations.
- Background workloads.

Record baseline performance and investigate material regressions.

### Migration Tests

- Apply all migrations from an empty database.
- Apply supported upgrade paths.
- Validate constraints and indexes.
- Test rollback or forward-recovery instructions.
- Test large data migrations using representative volumes where risk warrants it.
- Confirm application compatibility during staged deployments.

---

## 31. Development and Configuration Standards

### Configuration

- Keep database URLs and credentials outside source code.
- Use environment-specific configuration.
- Validate required settings at startup.
- Centralize pool, timeout, TLS, and retry configuration.
- Separate application credentials from migration credentials.
- Ensure configuration is documented for SaaS and self-hosted installations.

Example:

```env
DATABASE_URL=
DATABASE_MAX_CONNECTIONS=20
DATABASE_CONNECTION_TIMEOUT_MS=5000
DATABASE_QUERY_TIMEOUT_MS=10000
DATABASE_TLS_ENABLED=true
```

These names are illustrative. The actual configuration contract must be consistent across services and deployment environments.

### Local Development

- Provide a reproducible local PostgreSQL setup.
- Version-control schema migrations.
- Support deterministic test database initialization.
- Avoid relying on manually prepared local database state.
- Provide safe seed data that contains no production secrets or personal information.
- Ensure local setup matches production database behavior closely enough for meaningful testing.

### Version Management

- Pin supported PostgreSQL major versions for production deployments.
- Test application and migration compatibility against the supported version range.
- Document extension compatibility.
- Define database upgrade and rollback procedures.
- Avoid unreviewed major-version upgrades.

---

## 32. Self-Hosted University Deployments

KAMPYN must support universities that operate their own database infrastructure.

### Requirements

Document:

- Supported PostgreSQL versions.
- Minimum and recommended compute and storage.
- Network and firewall requirements.
- TLS and authentication configuration.
- Role and permission setup.
- Database and schema initialization.
- Migration execution process.
- Backup and recovery procedures.
- Monitoring and alerting.
- Replication and high-availability options.
- Upgrade and compatibility policies.
- Data export and tenant offboarding.
- Integration requirements for Redis, MongoDB, OpenSearch, and object storage.

### Deployment Flexibility

- Do not hardcode cloud-provider-specific PostgreSQL endpoints.
- Keep database connectivity configurable.
- Separate application behavior from managed-database-specific features where practical.
- Document optional managed-service integrations.
- Clearly identify features that require extensions or additional infrastructure.
- Provide secure defaults while allowing institution-approved infrastructure policies.

Self-hosted support must be validated through documented installation and upgrade testing, not just configuration flexibility.

---

## 33. Anti-Patterns

The following practices are prohibited unless a documented exception is approved:

- Using PostgreSQL as a replacement for every datastore.
- Using the database as the full-text search engine for all discovery workloads without justification.
- Storing large binary assets directly in relational tables by default.
- Omitting primary keys from persistent entity tables.
- Omitting foreign keys where relational integrity is required.
- Relying solely on application code for essential data invariants.
- Using floating-point types for exact monetary values.
- Storing timestamps as arbitrary strings.
- Using JSONB to avoid appropriate relational modeling.
- Creating arbitrary indexes without query evidence.
- Fetching unbounded rows into application memory.
- Performing unbounded `OFFSET` pagination.
- Running external network calls inside long-lived transactions.
- Using read-modify-write logic without concurrency protection.
- Relying on cached data for critical transaction decisions.
- Performing cross-tenant queries without explicit authorization.
- Trusting client-provided tenant identifiers as the sole isolation control.
- Creating one database per tenant without a documented operational strategy.
- Using superuser credentials for application services.
- Concatenating untrusted input into SQL.
- Applying manual production schema changes outside migration governance.
- Running destructive migrations without recovery planning.
- Disabling constraints without explicit approval and validation.
- Ignoring deadlocks, lock waits, or serialization failures.
- Retrying non-idempotent operations blindly.
- Creating a new database connection for each request.
- Increasing connection pool sizes without capacity planning.
- Ignoring replication lag for consistency-sensitive operations.
- Assuming successful backups guarantee recoverability.
- Skipping concurrency, tenant-isolation, and migration tests.

---

## 34. Required Documentation

Every production database or significant domain schema must document:

- Database and schema ownership.
- Domain entities and relationships.
- Primary-key and identifier conventions.
- Tenant-isolation strategy.
- Table definitions and constraints.
- Important query patterns.
- Index rationale.
- Transaction and concurrency requirements.
- Data lifecycle and retention.
- Migration ownership and execution.
- Connection and pool configuration.
- Security and access-control policies.
- Backup and recovery expectations.
- Replication and scaling strategy.
- Observability and alerting.
- Testing requirements.
- Cross-datastore synchronization.
- Self-hosted deployment requirements, where applicable.

Significant design decisions should be captured in an architecture decision record (ADR) when they affect multiple services, long-term compatibility, or operational responsibilities.

---

## 35. Review Checklist

Before approving a PostgreSQL-backed feature, verify:

- [ ] PostgreSQL is the appropriate source of truth for the workload.
- [ ] Table and schema ownership are explicit.
- [ ] Data modeling follows clear entity and relationship boundaries.
- [ ] Primary keys and foreign keys are correctly defined.
- [ ] Required unique and check constraints are enforced.
- [ ] Data types correctly represent the domain.
- [ ] Monetary and timestamp fields follow platform standards.
- [ ] Tenant ownership and isolation are enforced.
- [ ] Cross-tenant relationships are prevented.
- [ ] Transaction boundaries protect business invariants.
- [ ] Concurrent writes are safe.
- [ ] Critical queries are parameterized and bounded.
- [ ] Indexes support measured access patterns.
- [ ] Pagination is deterministic and scalable.
- [ ] Connection pooling and timeouts are configured.
- [ ] Migrations are version-controlled and production-safe.
- [ ] Deletes, retention, and archival are documented.
- [ ] Security and least-privilege access are implemented.
- [ ] Backup, recovery, and replication plans are defined.
- [ ] Observability and alerting are in place.
- [ ] Unit and integration tests cover database behavior.
- [ ] Critical concurrency scenarios are tested.
- [ ] Performance has been evaluated with representative workloads.
- [ ] Cross-datastore synchronization is reliable.
- [ ] Self-hosted deployment requirements are documented where applicable.

---

## 36. Definition of Done

A PostgreSQL-backed feature is complete only when:

1. Its domain ownership and source-of-truth responsibility are documented.
2. Its schema accurately represents the business domain.
3. Primary keys, foreign keys, and essential constraints enforce data integrity.
4. Tenant isolation and authorization requirements are implemented and tested.
5. Transaction boundaries and concurrency behavior are explicitly defined.
6. Queries are parameterized, bounded, and supported by appropriate indexes.
7. Connection pooling, timeouts, and failure handling are configured.
8. Schema changes are delivered through reviewed migrations.
9. Data lifecycle, retention, and deletion behavior are documented.
10. Security, privacy, and least-privilege controls are satisfied.
11. Backup, recovery, and operational ownership are established.
12. Metrics, logs, traces, and alerts are implemented.
13. Integration, concurrency, migration, and relevant performance tests pass.
14. Cross-datastore synchronization and recovery paths are defined.
15. Deployment documentation supports the intended SaaS and self-hosted environments.

**Final principle:** PostgreSQL is KAMPYN's foundation for reliable relational and transactional data. Business invariants must be enforced as close to the data as practical, tenant boundaries must remain intact, and every critical write must have well-defined consistency, concurrency, and recovery behavior.
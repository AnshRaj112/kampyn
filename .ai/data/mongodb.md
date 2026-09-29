# MongoDB

## 1. Purpose

This document defines the engineering standards for using MongoDB within KAMPYN, including data modeling, schema design, indexing, query optimization, transactions, multi-tenancy, security, migrations, and operational management.

MongoDB is a document-oriented datastore intended for workloads where flexible document structures, nested data, or document-centric access patterns provide a clear architectural benefit.

MongoDB MUST be used deliberately. It MUST NOT become a secondary storage location for relational data that is more appropriately owned by PostgreSQL.

The objectives are to:

- Define clear responsibilities for MongoDB within KAMPYN.
- Establish consistent document modeling and schema validation standards.
- Support efficient, predictable, and scalable data access.
- Preserve tenant isolation and data integrity.
- Provide safe concurrency, atomicity, and transaction management.
- Enable controlled schema evolution and migrations.
- Support observability, backup, recovery, and operational reliability.
- Prevent unnecessary duplication of data across datastores.

---

## 2. Core Principles

### 2.1 Purpose-Driven Usage

- MongoDB MUST only be introduced when its document-oriented capabilities provide a justified benefit.
- Every MongoDB collection MUST have a documented business or technical purpose.
- Data ownership MUST be explicitly assigned to a domain module.
- MongoDB MUST NOT be used merely because its schema is flexible.
- PostgreSQL MUST remain the default datastore for relational, transactional business entities unless an approved architectural decision justifies otherwise.

### 2.2 Document-Centric Design

- Documents SHOULD be modeled around the application's access patterns.
- Related data SHOULD be embedded when it is commonly retrieved and updated as a single logical unit.
- Data SHOULD be referenced when it has independent ownership, lifecycle, or high reuse across documents.
- Documents MUST have clearly defined boundaries and lifecycle expectations.
- Unbounded document growth MUST be avoided.

### 2.3 Data Integrity

- Critical document structures MUST be validated.
- Required fields, field types, and domain invariants MUST be enforced.
- Database-level validation SHOULD be used where supported and appropriate.
- Application validation MUST remain in place for business rules.
- MongoDB's flexible schema MUST NOT be interpreted as permission to store inconsistent data.

### 2.4 Explicit Consistency

- Every collection MUST have documented consistency expectations.
- Atomicity requirements MUST be understood before selecting a data model.
- Multi-document transactions SHOULD be used only when the operation requires them.
- Eventual consistency MUST be explicit when data is replicated or derived.
- Critical business invariants MUST NOT rely on uncontrolled asynchronous updates.

### 2.5 Operational Simplicity

- Prefer simple collections and predictable queries over unnecessary document complexity.
- Avoid premature sharding, complex aggregation pipelines, and excessive indexes.
- MongoDB-specific behavior MUST be isolated behind appropriate repositories or infrastructure adapters.
- Operational complexity MUST be justified by measurable requirements.

---

## 3. MongoDB's Role in KAMPYN

MongoDB is one component of KAMPYN's polyglot persistence architecture.

| Datastore | Primary responsibility |
|---|---|
| PostgreSQL | Relational data, transactional business entities, constraints, and authoritative records |
| MongoDB | Approved document-oriented workloads, nested records, and independently justified document-centric data |
| Redis | Caching, ephemeral state, rate limiting, and coordination |
| OpenSearch | Full-text search, relevance, and search-oriented retrieval |
| Object Storage | Images, documents, media, and other binary assets |

MongoDB MAY be suitable for:

- Flexible application configuration documents.
- Complex nested records with document-centric access patterns.
- Certain community or activity metadata where document ownership and lifecycle justify it.
- Integration payloads or normalized external data when a document datastore is appropriate.
- Explicitly approved workloads where document updates and retrieval are naturally atomic at the document level.

MongoDB SHOULD NOT be the default datastore for:

- Financial ledgers and payment records.
- Highly relational entities with complex joins and constraints.
- Core inventory and order relationships that require strong relational integrity.
- Cross-entity reporting workloads that belong in a relational or analytical system.
- Search workloads that require OpenSearch capabilities.
- Ephemeral state that is better suited to Redis.
- Binary storage that belongs in object storage.

The choice of datastore MUST be documented in the relevant architecture decision or domain data-modeling documentation when it materially affects system architecture.

---

## 4. Architecture and Ownership

### 4.1 Layered Architecture

MongoDB access MUST follow KAMPYN's established dependency direction:

```text
Presentation / Transport
          |
          v
     Application
          |
          v
        Domain
          |
          v
   Repository Contract
          |
          v
 MongoDB Infrastructure
          |
          v
       MongoDB
```

Responsibilities:

- **Presentation / Transport:** Handles external requests and response formatting.
- **Application:** Orchestrates use cases and transaction boundaries.
- **Domain:** Defines business rules, invariants, and domain behavior.
- **Repository Contract:** Defines persistence operations required by the application or domain.
- **MongoDB Infrastructure:** Implements repository contracts, database operations, mapping, and persistence-specific behavior.
- **MongoDB:** Stores documents and enforces supported database-level constraints.

Domain logic MUST NOT depend directly on MongoDB drivers, BSON types, aggregation pipeline definitions, or MongoDB-specific query syntax.

### 4.2 Collection Ownership

- Each collection MUST have one clearly identified owning module.
- Only the owning module SHOULD perform direct writes to its authoritative collections.
- Other modules MUST access owned data through approved APIs, application services, or events.
- Cross-module collection access MUST NOT become an undocumented integration mechanism.
- Shared collections MUST have explicit ownership and field-level change policies.

### 4.3 Repository Boundaries

Repositories MUST:

- Encapsulate MongoDB-specific queries.
- Enforce the expected tenant scope.
- Apply appropriate projections and limits.
- Handle database errors consistently.
- Expose domain-appropriate methods rather than generic database operations.
- Support cancellation and context propagation.
- Avoid leaking driver-specific types into higher layers.

Repositories MUST NOT become containers for business workflows, authorization policies, or unrelated module logic.

### 4.4 Dependency Management

- MongoDB client configuration MUST be centralized within infrastructure.
- Connection creation MUST be managed by the application runtime or dependency-injection system.
- Repositories MUST reuse managed clients rather than creating a new connection pool per request.
- Database access MUST be mockable or replaceable at appropriate application boundaries.
- Database-specific dependencies MUST remain within approved infrastructure packages.

---

## 5. Data Modeling

### 5.1 Document Design

Documents MUST represent a coherent entity or aggregate with a clearly defined ownership and lifecycle.

A document SHOULD contain fields that are commonly accessed together and belong to the same logical unit.

Example:

```json
{
  "_id": "cfg_123",
  "tenantId": "tenant_456",
  "name": "Food Court Configuration",
  "settings": {
    "orderingEnabled": true,
    "pickupEnabled": true,
    "maximumActiveOrders": 100
  },
  "schemaVersion": 1,
  "createdAt": "2026-09-28T10:00:00Z",
  "updatedAt": "2026-09-28T10:00:00Z"
}
```

The structure is illustrative. Actual identifiers, types, and fields MUST follow the owning module's data contract.

### 5.2 Embedding

Embedding SHOULD be used when:

- Related data is normally retrieved together.
- Related data has a shared lifecycle.
- Updates are typically performed as one atomic document operation.
- The embedded data has a bounded and predictable size.
- The relationship does not require extensive independent querying.

Examples may include:

- Bounded configuration settings.
- Small, stable metadata objects.
- Address components owned by a single document.
- Small, bounded sets of options associated with a configuration.

Embedding MUST NOT be used merely to avoid defining a relationship.

### 5.3 Referencing

References SHOULD be used when:

- Related data has an independent lifecycle.
- Multiple entities reuse the same record.
- Embedded data could grow without a reliable bound.
- Related data is frequently queried independently.
- Updates need to be isolated from the parent document.
- The relationship has many-to-many semantics.

Example:

```json
{
  "_id": "booking_123",
  "tenantId": "tenant_456",
  "guestId": "guest_789",
  "roomId": "room_101",
  "status": "confirmed",
  "createdAt": "2026-09-28T10:00:00Z"
}
```

References MUST have documented ownership and integrity expectations. MongoDB references do not automatically enforce relational foreign-key constraints.

### 5.4 Avoiding Unbounded Growth

- Unbounded arrays MUST NOT be embedded in a single document.
- High-volume histories SHOULD be stored in separately managed collections or a datastore designed for that workload.
- Activity feeds, messages, event histories, and audit records MUST have explicit growth and retention policies.
- Large embedded arrays MUST be evaluated for document-size, update-cost, and contention risks.
- Bucketing MAY be used for time-series or sequential records when it simplifies bounded access and retention.

MongoDB's document-size limits MUST be considered during schema design and migration planning.

### 5.5 Aggregate Boundaries

- Documents SHOULD align with aggregate boundaries where document-level atomicity is sufficient.
- Aggregate boundaries MUST be based on business consistency requirements rather than convenience alone.
- Cross-aggregate workflows MUST use explicit application-level coordination.
- Transactions MUST NOT be used to conceal poorly defined ownership or aggregate boundaries.

### 5.6 Normalization and Denormalization

Denormalization MAY be used to reduce read-time joins or simplify a common access pattern.

Denormalized fields MUST have:

- A clearly identified authoritative source.
- A documented update strategy.
- Defined consistency expectations.
- A reconciliation or repair procedure where required.
- Tests for synchronization behavior.

Denormalization MUST NOT create multiple competing sources of truth.

---

## 6. Schema and Validation

### 6.1 Schema Standards

Every collection MUST have a documented schema, including:

- Collection purpose and owner.
- Required and optional fields.
- Field types and formats.
- Identifier strategy.
- Tenant relationship, where applicable.
- Validation requirements.
- Indexes and supported access patterns.
- Lifecycle and retention requirements.
- Schema evolution strategy.

### 6.2 Application-Level Validation

- External input MUST be validated before persistence.
- Domain invariants MUST be enforced in the application or domain layer.
- Data transfer objects MUST NOT be persisted blindly.
- Type conversion MUST be explicit.
- Unsupported or malformed fields MUST be handled according to the module's validation policy.
- Validation errors MUST be actionable without disclosing sensitive data.

In Go, strongly typed structs SHOULD be used for persistence and domain boundaries rather than unrestricted `bson.M` maps throughout the application.

### 6.3 Database-Level Validation

MongoDB collection validators SHOULD enforce structural requirements that can be expressed through supported validation mechanisms.

Example:

```javascript
db.createCollection("configurations", {
  validator: {
    $jsonSchema: {
      bsonType: "object",
      required: ["tenantId", "name", "schemaVersion"],
      properties: {
        tenantId: {
          bsonType: "string"
        },
        name: {
          bsonType: "string"
        },
        schemaVersion: {
          bsonType: "int"
        }
      }
    }
  },
  validationLevel: "strict",
  validationAction: "error"
});
```

Guidelines:

- Validators MUST reflect the approved collection schema.
- Validator changes MUST be version-controlled.
- Validation strictness MUST be selected deliberately.
- Validation failures MUST be monitored.
- Database validation MUST complement, not replace, business-level validation.

### 6.4 Schema Versioning

- Documents SHOULD contain a schema version when multiple versions may coexist.
- Version numbers MUST have clearly defined semantics.
- Version changes MUST be included in migration documentation.
- Application readers MUST support transitional versions when required.
- Legacy document handling MUST have a defined retirement plan.
- Schema-version fields MUST NOT be incremented without a corresponding structural or semantic change.

---

## 7. Identifiers and Data Types

### 7.1 Document Identifiers

- Every document MUST have a stable `_id`.
- Identifier strategy MUST be consistent within a collection.
- ObjectId, UUID, or application-generated identifiers MAY be used based on system requirements.
- Identifier format MUST be documented.
- Identifiers MUST NOT be treated as authorization credentials.
- External-facing identifiers SHOULD avoid exposing sensitive internal information.

### 7.2 BSON Types

- BSON types MUST be selected deliberately.
- Numeric fields MUST use types appropriate to their valid range and precision.
- Timestamps MUST use a consistent representation.
- Dates MUST use UTC-based semantics unless the domain explicitly requires a different representation.
- Monetary values MUST use an exact representation, such as integer minor units or an approved decimal strategy.
- Arbitrary string representations of numeric or date values MUST NOT be used when native types are appropriate.
- BSON type changes MUST be treated as schema migrations.

### 7.3 Null and Missing Values

- Required fields MUST have explicit presence requirements.
- Missing and `null` MUST have distinct semantics where business behavior depends on the difference.
- Queries MUST account for MongoDB's missing-field and null-matching behavior.
- Optional fields SHOULD be omitted when omission has a clear and consistent meaning.
- Uncontrolled combinations of missing, null, empty string, and default values MUST be avoided.

### 7.4 Timestamps

- Persisted timestamps MUST use a consistent UTC convention.
- Timestamp semantics MUST be explicit, such as creation, modification, expiration, or business-effective time.
- Update operations MUST maintain modification timestamps where required.
- Application and database clock assumptions MUST be understood.
- Timestamp-based ordering MUST include a stable tie-breaker when deterministic ordering is required.

---

## 8. Multi-Tenancy

### 8.1 Tenant Identification

- Every tenant-owned document MUST include a trusted tenant identifier.
- Tenant identifiers MUST be derived from authenticated and authorized application context.
- Tenant identifiers from request payloads MUST NOT be trusted without validation against the authorized context.
- Tenant identity MUST remain consistent across related records and derived projections.
- Platform-owned documents MUST have an explicitly documented ownership model.

Example:

```json
{
  "_id": "item_123",
  "tenantId": "tenant_456",
  "name": "Vegetable Sandwich",
  "status": "active"
}
```

### 8.2 Tenant-Scoped Queries

- Tenant-scoped repository operations MUST require an authorized tenant context.
- Tenant filters MUST be applied to reads, updates, deletes, and aggregations involving tenant-owned data.
- Query construction MUST prevent tenant predicates from being accidentally omitted.
- Bulk operations MUST be explicitly scoped to authorized tenant records.
- Tenant-scoped query behavior MUST be tested.

Example:

```javascript
db.items.find({
  tenantId: "tenant_456",
  status: "active"
});
```

### 8.3 Tenant-Scoped Updates

Updates MUST include the tenant predicate when operating on tenant-owned documents.

Example:

```javascript
db.items.updateOne(
  {
    _id: "item_123",
    tenantId: "tenant_456"
  },
  {
    $set: {
      status: "inactive"
    }
  }
);
```

- Update filters MUST identify the intended document and authorized tenant.
- `updateMany` MUST have a reviewed scope and bounded impact.
- Tenant-scoped deletes MUST follow the same enforcement principles.
- Tenant predicates MUST NOT be removed simply because `_id` is unique.

### 8.4 Tenant-Scoped Uniqueness

Where uniqueness applies within a tenant, use a compound unique index.

Example:

```javascript
db.items.createIndex(
  {
    tenantId: 1,
    sku: 1
  },
  {
    unique: true
  }
);
```

The exact uniqueness policy MUST follow the domain requirements. Global uniqueness and tenant-specific uniqueness MUST be distinguished explicitly.

### 8.5 Shared and Dedicated Deployments

KAMPYN may support shared SaaS storage and dedicated university deployments.

- Shared collections MUST enforce tenant-aware access and indexing.
- Dedicated databases MUST still preserve authentication, authorization, and ownership rules.
- Deployment-specific database configurations MUST be explicit.
- Migration and backup strategies MUST account for each deployment model.
- Tenant isolation MUST NOT depend exclusively on database naming or physical separation.

---

## 9. Indexing

### 9.1 Index Requirements

- Every index MUST have a documented purpose.
- Indexes MUST support identified query patterns or data-integrity requirements.
- Existing indexes MUST be reviewed before adding new ones.
- Index storage and write overhead MUST be considered.
- Index changes MUST be version-controlled.

Detailed indexing standards are defined in `data/indexing.md`.

### 9.2 Common Index Patterns

Tenant-scoped lookup:

```javascript
db.orders.createIndex({
  tenantId: 1,
  orderId: 1
});
```

Tenant-scoped status and sorting:

```javascript
db.orders.createIndex({
  tenantId: 1,
  status: 1,
  createdAt: -1
});
```

Tenant-scoped unique identifier:

```javascript
db.users.createIndex(
  {
    tenantId: 1,
    email: 1
  },
  {
    unique: true
  }
);
```

These are examples only. Index definitions MUST be validated against actual query plans, data distributions, and write patterns.

### 9.3 Index Management

- Indexes MUST be managed through approved migrations or controlled deployment automation.
- Duplicate and redundant indexes SHOULD be removed after appropriate dependency analysis.
- Index builds MUST be monitored on large collections.
- Index usage statistics MUST be interpreted in the context of workload and monitoring history.
- Index creation and removal MUST be reviewed for production impact.

---

## 10. Query Design

### 10.1 Query Requirements

- Queries MUST be explicit and bounded.
- Repositories MUST return only the fields required by the caller where practical.
- Queries MUST use appropriate filters and indexes.
- Unbounded collection reads MUST NOT be used in production request paths.
- Query behavior MUST be predictable for empty, large, and malformed result sets.
- Query timeouts and cancellation MUST be supported where appropriate.

### 10.2 Projections

Projections SHOULD be used to avoid retrieving unnecessary fields.

Example:

```javascript
db.orders.find(
  {
    tenantId: "tenant_456",
    status: "pending"
  },
  {
    _id: 1,
    status: 1,
    createdAt: 1
  }
);
```

- Large embedded fields SHOULD be excluded when they are not needed.
- Sensitive fields MUST NOT be returned unnecessarily.
- Projections MUST be consistent with the application's data contract.

### 10.3 Pagination

- Queries returning potentially large result sets MUST use pagination or bounded streaming.
- Sorting MUST be deterministic.
- Cursor-based pagination SHOULD be used for large datasets where appropriate.
- Deep `skip` pagination MUST be avoided for high-volume collections when it creates excessive scanning.
- Pagination tokens MUST be validated and scoped to the relevant tenant and query context.

### 10.4 Sorting

- Frequently used sort operations SHOULD be supported by suitable indexes.
- Sort fields MUST be explicitly allowlisted when chosen dynamically.
- Sorting on unindexed fields MUST be assessed against expected result sizes.
- Stable tie-breakers SHOULD be used when the sort field is not unique.

### 10.5 Aggregation Pipelines

Aggregation pipelines MAY be used for suitable document-oriented transformations and retrieval.

- Pipelines MUST have a documented purpose.
- `$match` stages SHOULD be placed early where semantically valid.
- Projection SHOULD remove unnecessary fields before expensive stages.
- Pipeline cardinality and memory requirements MUST be considered.
- `$lookup` MUST be justified and assessed for performance.
- Large aggregations MUST use appropriate limits, batching, or supported disk-use behavior.
- Tenant scoping MUST be enforced throughout the pipeline.
- Aggregations MUST NOT silently bypass authorization or domain ownership.

### 10.6 Query Plan Analysis

Use `explain("executionStats")` for performance-critical queries.

Review:

- Winning plan.
- Index scans versus collection scans.
- Keys examined.
- Documents examined.
- Documents returned.
- Sort stages.
- Execution time.
- Shard targeting and scatter-gather behavior where applicable.

Queries MUST be evaluated against representative data volumes and distributions. Index existence alone is not sufficient evidence of efficient query execution.

---

## 11. Writes and Atomicity

### 11.1 Atomic Document Updates

MongoDB supports atomic operations at the single-document level.

- Operations that belong to a single logical document SHOULD be designed to use atomic document updates.
- Update filters MUST include the conditions required to prevent invalid state transitions.
- Read-modify-write sequences MUST account for concurrent modifications.
- Atomic update operators SHOULD be preferred over application-side replacement where appropriate.
- Whole-document replacements MUST be used carefully to avoid overwriting concurrent changes.

### 11.2 Update Operators

- Use targeted update operators such as `$set`, `$unset`, `$inc`, `$push`, and `$pull` where they express the intended operation.
- Updates MUST be scoped to the intended document and tenant.
- Numeric updates MUST account for overflow and valid domain ranges.
- Array updates MUST have defined growth limits.
- Update behavior MUST be tested for missing fields and unexpected document versions.

### 11.3 Optimistic Concurrency

Optimistic concurrency SHOULD be used when concurrent modifications could invalidate a business operation.

Example:

```javascript
db.configurations.updateOne(
  {
    _id: "cfg_123",
    tenantId: "tenant_456",
    version: 4
  },
  {
    $set: {
      "settings.orderingEnabled": false
    },
    $inc: {
      version: 1
    }
  }
);
```

- Version fields MUST have clear semantics.
- A failed conditional update MUST be handled as a concurrency conflict where appropriate.
- The application MUST NOT assume a write succeeded merely because no exception was raised.
- Conflict handling MUST avoid uncontrolled retry loops.

### 11.4 Bulk Writes

Bulk operations MAY be used for high-throughput workloads.

- Bulk operations MUST have bounded batch sizes.
- Ordered versus unordered behavior MUST be selected deliberately.
- Partial failures MUST be inspected and handled.
- Operations MUST be idempotent where retry is possible.
- Tenant scope MUST be explicit for every operation.
- Bulk write throughput MUST be balanced against replication and resource impact.

### 11.5 Upserts

- Upserts MUST have deterministic identity and matching semantics.
- Unique indexes SHOULD enforce relevant identity constraints.
- Upserts MUST NOT create unintended duplicates under concurrent execution.
- Retry behavior MUST be understood.
- Upsert filters MUST include the required tenant and ownership boundaries.

---

## 12. Transactions

### 12.1 Transaction Policy

MongoDB multi-document transactions MAY be used when a business operation requires atomicity across multiple documents and cannot be safely modeled as a single-document operation.

- Transactions MUST have an explicit business justification.
- Single-document atomicity SHOULD be preferred when sufficient.
- Transaction scope MUST remain small and bounded.
- Long-running transactions MUST be avoided.
- Transaction behavior MUST be tested against the configured MongoDB deployment topology.

### 12.2 Transaction Boundaries

- Transaction boundaries MUST be defined in the application layer.
- Repository operations participating in a transaction MUST share the same session context.
- Transaction ownership MUST be explicit.
- Nested or overlapping transaction semantics MUST NOT be assumed unless supported and deliberately implemented.
- External network calls MUST NOT be performed inside a database transaction.

### 12.3 Retry Handling

- Transient transaction errors MUST be handled according to driver guidance.
- Retry logic MUST be bounded.
- Operations MUST be designed to tolerate retries.
- Unknown commit outcomes MUST be handled carefully to avoid duplicate business effects.
- Idempotency keys SHOULD be used for externally retriable operations where appropriate.

### 12.4 Cross-Datastore Transactions

MongoDB transactions MUST NOT be assumed to provide atomicity across PostgreSQL, Redis, OpenSearch, or external services.

Cross-datastore consistency SHOULD use explicit patterns such as:

- Transactional outbox in the authoritative datastore.
- Reliable event delivery.
- Idempotent consumers.
- Sagas or compensating actions where appropriate.
- Reconciliation and repair jobs.

The consistency model MUST be documented for each cross-datastore workflow.

---

## 13. Connection Management

### 13.1 Client Lifecycle

- MongoDB clients MUST be created through centralized infrastructure configuration.
- Clients MUST be reused to benefit from connection pooling.
- New clients MUST NOT be created per HTTP request or repository operation.
- Application shutdown MUST close managed clients gracefully.
- Connection lifecycle MUST be integrated with application startup and shutdown.

### 13.2 Pool Configuration

Connection-pool settings MUST be configured according to:

- Expected concurrent workload.
- Number of application instances.
- Database connection limits.
- Deployment topology.
- Latency and timeout requirements.
- Background job concurrency.

Pool sizes MUST NOT be selected arbitrarily or multiplied without accounting for the number of running instances.

### 13.3 Timeouts

- Server selection, connection establishment, socket, and operation timeouts MUST be configured deliberately.
- Request cancellation MUST propagate to database operations where supported.
- Timeouts MUST be consistent with application-level latency budgets.
- Timeout failures MUST be observable.
- Automatic retries MUST NOT extend requests beyond their intended operational limits.

### 13.4 Health Checks

- Readiness checks SHOULD verify that required MongoDB dependencies are reachable.
- Liveness checks MUST NOT trigger expensive database operations.
- Health checks MUST use bounded timeouts.
- Temporary database unavailability MUST be distinguished from permanent application failure.
- Optional MongoDB dependencies MUST not incorrectly prevent unrelated services from operating unless the architecture requires them.

---

## 14. Multi-Cluster and Sharding Strategy

### 14.1 Cluster Responsibilities

Where KAMPYN uses multiple MongoDB clusters, each cluster MUST have an explicitly defined responsibility and data ownership boundary.

Potential logical partitions include:

- User-related documents.
- Order-related documents.
- Item-related documents.
- Inventory-related documents.
- Food-court-related documents.
- Cache or analytics-related documents, only where MongoDB is explicitly justified for that workload.

These are logical examples, not an instruction to create a separate cluster for every domain.

### 14.2 Cluster Selection

- Cluster selection MUST be centralized and deterministic.
- Each collection MUST have one declared owning cluster or deployment target.
- Repository code MUST NOT construct cluster connection strings independently.
- Cluster routing MUST respect tenant boundaries and data ownership.
- Cross-cluster operations MUST have an explicit consistency and failure strategy.

### 14.3 Sharding

Sharding SHOULD only be introduced when measured scale or workload characteristics justify it.

Before sharding, evaluate:

- Data volume and growth rate.
- Query patterns.
- Shard-key cardinality and distribution.
- Write concentration.
- Tenant size distribution.
- Query targeting.
- Operational complexity.
- Backup, restore, and migration requirements.

Shard keys MUST be selected using representative workloads. They MUST NOT be selected solely because a field is present in every document.

### 14.4 Shard-Key Strategy

- Shard-key selection MUST consider write distribution and read locality.
- Monotonically increasing shard keys MUST be evaluated for potential hotspots.
- Tenant-based shard keys MUST be assessed for large-tenant concentration and cross-tenant query requirements.
- Queries SHOULD include shard-key predicates when targeted routing is expected.
- Unique-index and shard-key compatibility MUST be verified against the deployed MongoDB version.

### 14.5 Cross-Cluster Workflows

- Cross-cluster transactions MUST NOT be assumed to be available.
- Distributed workflows MUST use explicit coordination.
- Event-driven synchronization SHOULD be preferred for suitable asynchronous interactions.
- Partial failures MUST be recoverable.
- Reconciliation MUST be available for critical cross-cluster data relationships.

---

## 15. Caching and Redis Integration

MongoDB and Redis MUST have clearly separated responsibilities.

- MongoDB MUST store approved persistent document data.
- Redis SHOULD store transient values, hot-cache entries, coordination state, and other approved ephemeral workloads.
- Cache entries MUST have defined expiration and invalidation policies.
- Redis cache loss MUST NOT cause authoritative MongoDB data loss.
- Cache invalidation MUST be coordinated with successful writes.
- Cache keys MUST include tenant scope where applicable.
- Cache serialization formats MUST be versioned when incompatible changes are expected.

Where stale cache data could cause incorrect business behavior, the application MUST define a suitable consistency strategy rather than relying on TTL alone.

---

## 16. Events and OpenSearch Integration

### 16.1 Event-Driven Updates

When MongoDB is the authoritative source for an approved domain:

- Changes requiring downstream synchronization MUST produce reliable events through an approved event-delivery design.
- Events MUST have stable identifiers and explicit schema versions.
- Consumers MUST be idempotent.
- Event ordering requirements MUST be documented.
- Delivery failures MUST be observable and recoverable.

Where PostgreSQL is authoritative, MongoDB MUST NOT independently emit competing authoritative business events for the same state change.

### 16.2 Change Streams

MongoDB change streams MAY be used for approved synchronization or operational use cases.

- Change-stream consumers MUST handle resumability and interruptions.
- Resume-token persistence MUST follow a documented recovery strategy.
- Duplicate delivery MUST be expected and handled.
- Change streams MUST NOT be assumed to replace domain-level event contracts in every workflow.
- Change-stream availability and behavior MUST be evaluated against the MongoDB deployment topology.

### 16.3 OpenSearch Projection

When MongoDB is the approved source for a search projection:

- Search documents MUST be derived from authoritative records.
- Projection updates MUST be idempotent.
- Stale updates MUST be detected or prevented where ordering matters.
- Reindexing MUST be possible from authoritative data or a reliable event history.
- Search consistency expectations MUST be documented.
- Tenant filters MUST be applied in all tenant-scoped search requests.

OpenSearch remains a derived read model and MUST NOT become an undocumented write authority.

---

## 17. Migrations and Schema Evolution

### 17.1 Migration Policy

All MongoDB schema and data changes MUST follow `data/migrations.md`.

- Collection definitions and validators MUST be version-controlled.
- Index definitions MUST be managed through approved migrations.
- Data backfills MUST be bounded and resumable where appropriate.
- Applied migration history MUST be preserved.
- Destructive changes MUST have explicit recovery planning.

### 17.2 Expand-and-Contract

Breaking document changes SHOULD follow a staged approach:

1. Introduce the new field or document structure.
2. Deploy readers that support both old and new structures.
3. Update writes to populate the new structure.
4. Backfill existing documents.
5. Validate the migration results.
6. Transition reads to the new structure.
7. Remove legacy fields after all compatibility requirements are met.

### 17.3 Idempotent Backfills

- Backfills SHOULD use conditional updates to avoid overwriting already-migrated or newer records.
- Migration checkpoints MUST support safe resumption where practical.
- Migration scripts MUST handle missing, null, and malformed fields explicitly.
- Large collections MUST be processed without unbounded memory use.
- Failed records MUST be measurable and recoverable.

### 17.4 Schema Version Compatibility

- Applications MUST support the schema versions expected during the deployment compatibility window.
- Legacy-version support MUST have a documented retirement condition.
- Version conversion MUST preserve business semantics.
- Schema-version changes MUST be covered by migration tests.

---

## 18. Performance and Scalability

### 18.1 Query Efficiency

- Critical queries MUST use suitable filters, projections, and indexes.
- Unbounded scans MUST NOT be used in latency-sensitive request paths without explicit justification.
- Aggregation pipelines MUST be measured against representative data.
- Query execution statistics SHOULD be reviewed for performance-critical operations.
- N+1 query patterns SHOULD be eliminated through batching, projections, or appropriate data modeling.

### 18.2 Memory Management

- Large result sets MUST be processed through bounded cursors or batch-based operations.
- Application memory usage MUST remain bounded for large collections.
- Documents SHOULD avoid unnecessary duplicated payloads.
- Aggregation stages MUST be assessed for memory and disk usage.
- Large arrays MUST have explicit growth limits or a separate storage model.

### 18.3 Throughput

- Read and write throughput MUST be measured under representative concurrency.
- Connection pools MUST be configured with awareness of total application instances.
- Bulk operations MUST use appropriate batch sizes.
- Hot documents and highly contended records MUST be identified.
- High write contention SHOULD trigger a review of aggregate boundaries and update patterns.

### 18.4 Resource Limits

- Query and operation timeouts MUST be configured.
- Background jobs MUST use bounded concurrency.
- Expensive aggregations MUST have resource controls.
- Database capacity planning MUST consider storage, memory, CPU, network, and replication overhead.
- Performance degradation MUST be observable before it becomes a widespread service failure.

---

## 19. Reliability and Failure Handling

### 19.1 Database Unavailability

- Database failures MUST be handled through explicit error paths.
- Applications MUST NOT silently treat database failures as empty results.
- Retries MUST be bounded and restricted to retryable operations.
- Circuit breakers MAY be used where they provide clear protection from cascading failure.
- User-facing errors MUST remain actionable without exposing internal database details.

### 19.2 Retry Safety

- Retried writes MUST be idempotent or protected against duplicate effects.
- Retry logic MUST distinguish transient from permanent failures.
- Retry delays SHOULD use bounded backoff and jitter where appropriate.
- Retried transactions MUST follow driver-supported semantics.
- Retry exhaustion MUST produce an observable failure.

### 19.3 Graceful Degradation

- Optional MongoDB-backed functionality MAY degrade gracefully if the core application can safely continue.
- Critical business workflows MUST fail safely when required persistence is unavailable.
- Stale data MUST be clearly distinguished from authoritative data when presented to users.
- Degradation behavior MUST be documented and tested.

### 19.4 Recovery

- Recovery procedures MUST define how to restore collections and indexes.
- Replica-set recovery and failover behavior MUST be understood for the selected deployment.
- Restore procedures MUST be tested at an appropriate frequency.
- Data integrity MUST be validated after recovery.
- Reconciliation MUST be available for data synchronized to other stores.

---

## 20. Security and Privacy

### 20.1 Authentication and Authorization

- MongoDB authentication MUST be enabled in production.
- Applications MUST use dedicated database identities.
- Database permissions MUST follow least-privilege principles.
- Administrative credentials MUST NOT be used by application services.
- Read and write permissions MUST be scoped to the required databases and collections where supported.

### 20.2 Network Security

- Production MongoDB access MUST use approved private networking or restricted network paths.
- Public access MUST be disabled unless an explicitly approved architecture requires it.
- Firewall and network rules MUST restrict access to authorized services and operators.
- TLS MUST be enabled for connections in production.
- Network restrictions MUST be reviewed as part of deployment and infrastructure changes.

### 20.3 Secrets

- Connection strings and credentials MUST be stored in approved secret-management systems.
- Secrets MUST NOT be committed to source control.
- Credentials MUST NOT be included in logs, error messages, or monitoring labels.
- Credential rotation MUST be supported operationally.
- Separate credentials SHOULD be used for different services and environments.

### 20.4 Data Protection

- Sensitive data MUST be classified before storage.
- Data MUST be encrypted in transit and appropriately protected at rest.
- Sensitive fields MUST be excluded from unnecessary projections and logs.
- Personal data MUST follow applicable retention, deletion, and access policies.
- Backups and snapshots MUST follow the same protection requirements as live data.
- Database-level access MUST be complemented by application authorization.

### 20.5 Query Safety

- Dynamic queries MUST be constructed through safe driver APIs.
- Untrusted input MUST NOT be interpreted as arbitrary MongoDB operators.
- Field names and sort expressions MUST be allowlisted where user-controlled.
- Query complexity MUST be bounded for externally accessible endpoints.
- Aggregation pipelines MUST not permit users to inject arbitrary pipeline stages.

---

## 21. Backup, Retention, and Disaster Recovery

### 21.1 Backup

- Production MongoDB data MUST be covered by an approved backup strategy.
- Backup frequency MUST reflect business recovery requirements.
- Backup encryption and access control MUST be enforced.
- Backup retention MUST follow the data-retention policy.
- Backup completion MUST be monitored.
- Restore capability MUST be tested rather than assumed.

### 21.2 Recovery Objectives

- Recovery Time Objective (RTO) and Recovery Point Objective (RPO) MUST be defined for critical MongoDB workloads.
- Replica sets MUST NOT be treated as a substitute for independent backups.
- Recovery procedures MUST include data validation and application compatibility checks.
- Disaster recovery MUST account for deployment-specific cluster configuration.

### 21.3 Retention and Deletion

- Every collection MUST have a documented lifecycle policy.
- Data deletion MUST comply with business, legal, and privacy requirements.
- TTL indexes MAY be used for suitable automatically expiring records.
- TTL behavior MUST NOT be treated as an exact-time deletion guarantee.
- Archived data MUST have documented retrieval and restoration procedures.
- Deletion and retention policies MUST consider dependent projections and caches.

---

## 22. Observability

### 22.1 Metrics

Monitor, where applicable:

- Operation latency.
- Query throughput.
- Read and write error rates.
- Connection-pool usage.
- Active connections.
- Query timeouts.
- Replication lag.
- Lock and contention indicators.
- Document and collection growth.
- Index size and usage.
- Memory, CPU, disk, and network utilization.
- Slow-query frequency.
- Shard distribution and query targeting.

### 22.2 Logging

- Database errors MUST be logged with relevant operational context.
- Logs MUST include correlation or trace identifiers where available.
- Sensitive document contents MUST NOT be logged unnecessarily.
- Query logging MUST avoid exposing secrets or personal data.
- Error classification MUST distinguish connectivity, timeout, validation, conflict, and server-side failures.

### 22.3 Tracing

- Database operations SHOULD be integrated with distributed tracing for critical application workflows.
- Spans SHOULD include collection or operation context without sensitive values.
- Trace instrumentation MUST avoid creating excessive telemetry volume.
- Database latency MUST be distinguishable from application processing time.

### 22.4 Alerts

Alerts SHOULD be defined for:

- Sustained elevated query latency.
- Increased database error rates.
- Connection-pool exhaustion.
- Replication lag beyond defined thresholds.
- Storage pressure.
- Unexpected collection scans on critical workloads.
- Failed or delayed migrations.
- Backup failures.
- Cluster health degradation.

Alert thresholds MUST be based on service objectives and operational evidence.

---

## 23. Testing Requirements

### 23.1 Unit Testing

- Domain rules MUST be tested independently of MongoDB.
- Repository behavior SHOULD be tested using controlled dependencies or suitable test doubles.
- Validation and mapping logic MUST have focused tests.
- Error translation MUST be tested for relevant database failure categories.

### 23.2 Integration Testing

Integration tests MUST validate:

- Document creation, retrieval, update, and deletion.
- BSON type correctness.
- Collection validation.
- Unique and compound indexes.
- Tenant-scoped access.
- Atomic update behavior.
- Optimistic concurrency.
- Pagination and deterministic sorting.
- Aggregation pipeline correctness.
- Transaction behavior where used.
- Migration and schema compatibility.

Integration tests SHOULD use the supported MongoDB server version or a compatible test environment rather than relying solely on mocks.

### 23.3 Performance Testing

Performance-critical workloads SHOULD be tested for:

- Query latency under realistic data volumes.
- Index utilization.
- Read and write throughput.
- Connection-pool behavior.
- Aggregation performance.
- Concurrent update contention.
- Large-document and large-array behavior.
- Pagination at realistic dataset sizes.
- Replication or sharding effects where applicable.

### 23.4 Tenant-Isolation Testing

- Tests MUST verify that tenant-scoped operations cannot access other tenants' documents.
- Bulk updates and deletes MUST be tested for correct tenant boundaries.
- Aggregation pipelines MUST be tested for tenant filtering.
- Cross-tenant administrative operations MUST verify explicit authorization.
- Missing or invalid tenant context MUST fail safely.

### 23.5 Failure Testing

Where appropriate, test:

- Database unavailability.
- Operation timeouts.
- Transient network failures.
- Duplicate-key conflicts.
- Transaction retry scenarios.
- Interrupted migrations.
- Partial bulk-write failures.
- Replication or failover behavior in relevant environments.

---

## 24. Configuration and Environment Management

### 24.1 Configuration

MongoDB configuration MUST be environment-specific and centrally managed.

Configuration MAY include:

- Connection URI.
- Database name.
- Authentication mechanism.
- TLS configuration.
- Connection-pool limits.
- Operation timeouts.
- Retry behavior.
- Read preference.
- Write concern.
- Read concern.
- Deployment topology.
- Monitoring settings.

### 24.2 Configuration Safety

- Production configuration MUST NOT be embedded in source code.
- Environment variables or approved configuration providers SHOULD be used for non-secret settings.
- Secrets MUST be sourced from approved secret management.
- Unsupported or invalid configuration MUST fail safely during startup.
- Environment-specific defaults MUST be documented.
- Local development settings MUST NOT be copied into production deployments.

### 24.3 Read and Write Concerns

- Read concern and write concern MUST reflect the consistency and durability requirements of each workload.
- Defaults MUST be reviewed rather than assumed to satisfy every business requirement.
- Stronger guarantees MUST be applied where critical workflows require them.
- Read preferences MUST not expose stale data where freshness is a correctness requirement.
- Configuration changes MUST be tested against the actual deployment topology.

---

## 25. Development Standards

### 25.1 Code Quality

- MongoDB operations MUST be implemented in cohesive infrastructure modules.
- Repository methods MUST have clear names and explicit parameters.
- Public interfaces MUST use strongly typed inputs and outputs.
- BSON mapping MUST be centralized where reuse or consistency requires it.
- Error handling MUST be explicit.
- Context cancellation MUST be propagated.
- Database logic MUST NOT be duplicated across unrelated modules.
- Source files SHOULD remain within the 200-line guideline unless architectural justification is documented.

### 25.2 Reusability

- Common connection management MUST be centralized.
- Shared pagination, filtering, and mapping helpers MAY be introduced when genuinely reusable.
- Abstractions MUST not hide MongoDB-specific performance behavior where that behavior matters.
- Generic repository frameworks MUST NOT be introduced merely to eliminate a small amount of straightforward code.
- Domain-specific repository methods SHOULD be preferred over unrestricted CRUD interfaces.

### 25.3 Error Handling

- Database errors MUST be translated into appropriate application-level errors.
- Duplicate-key errors MUST be distinguished from general persistence failures.
- Context cancellation and deadline errors MUST be handled appropriately.
- Internal driver details MUST NOT leak into public API responses.
- Failed operations MUST not be represented as successful empty results.

### 25.4 Dependency and Driver Management

- MongoDB drivers MUST be approved and maintained.
- Driver versions MUST be pinned through the project's dependency-management process.
- Driver upgrades MUST include compatibility and regression testing.
- Deprecated APIs MUST be avoided.
- Driver-specific behavior MUST be documented when it affects correctness or operations.

---

## 26. Anti-Patterns

The following practices are prohibited unless an explicit, documented exception is approved:

- Using MongoDB as the default datastore for every KAMPYN domain.
- Storing relational business entities in MongoDB without an architectural justification.
- Creating collections without clear ownership.
- Allowing arbitrary modules to write to another module's collections.
- Treating schema flexibility as a replacement for validation.
- Embedding unbounded arrays or unlimited histories.
- Storing large binary files directly in ordinary documents without an approved design.
- Using application-side read-modify-write sequences without concurrency protection.
- Performing unbounded collection reads in production request paths.
- Creating indexes without a documented query or constraint requirement.
- Omitting tenant predicates from tenant-owned data operations.
- Treating `_id` uniqueness as tenant authorization.
- Using multi-document transactions for workflows that can be safely modeled with single-document atomicity.
- Performing external network calls inside database transactions.
- Assuming atomicity across MongoDB and other datastores.
- Creating MongoDB clients per request.
- Using unbounded retries or unbounded background database operations.
- Logging complete documents or sensitive query values.
- Running unversioned production schema changes.
- Using MongoDB as an undocumented cache or search engine.
- Introducing sharding without evidence and capacity planning.
- Relying on replicas without independent backup and recovery planning.
- Using generic repositories that conceal important query and consistency semantics.

---

## 27. Review Checklist

Before introducing or materially changing MongoDB usage, verify:

- [ ] The workload has a documented reason to use MongoDB.
- [ ] The owning domain and collection boundaries are explicit.
- [ ] PostgreSQL, Redis, OpenSearch, and object storage alternatives have been considered.
- [ ] Document structures align with access patterns and lifecycle requirements.
- [ ] Embedding and referencing choices are justified.
- [ ] Unbounded document growth is prevented.
- [ ] Schema and validation rules are documented.
- [ ] Identifier and BSON types are appropriate.
- [ ] Tenant identifiers and access boundaries are enforced.
- [ ] Indexes support actual query patterns.
- [ ] Queries are bounded and use suitable projections.
- [ ] Pagination and sorting are deterministic.
- [ ] Atomicity and concurrency requirements are addressed.
- [ ] Transactions are used only where justified.
- [ ] Connection pooling and timeouts are configured.
- [ ] Migration and schema-evolution strategies are documented.
- [ ] Cross-datastore consistency requirements are explicit.
- [ ] Security, privacy, and retention controls are implemented.
- [ ] Backup and disaster-recovery requirements are addressed.
- [ ] Observability and alerting are configured.
- [ ] Integration, tenant-isolation, and performance tests are present.
- [ ] Self-hosted deployment requirements are considered where applicable.
- [ ] Operational complexity is proportionate to the workload.

---

## 28. Definition of Done

MongoDB implementation or modification is complete only when:

- Its responsibility and ownership are documented.
- Data modeling aligns with the approved domain architecture.
- Schema validation and BSON typing are explicit.
- Tenant isolation and authorization boundaries are enforced.
- Indexes and queries are designed for the expected workload.
- Atomicity, concurrency, and consistency requirements are addressed.
- Connection lifecycle and resource limits are configured.
- Migrations are version-controlled and tested.
- Cross-datastore synchronization is reliable where required.
- Security, privacy, retention, and deletion requirements are met.
- Backup, recovery, and operational monitoring are addressed.
- Relevant integration, performance, and failure tests pass.
- Documentation and deployment configuration are updated.
- The implementation has been reviewed for maintainability, correctness, and operational risk.

Exceptions MUST be documented with their rationale, impact, mitigations, and approval.

---

## 29. Final Principle

**MongoDB should be used when document-oriented storage simplifies a clearly owned workload without compromising consistency, security, or operational clarity.**

KAMPYN MUST treat MongoDB as a deliberate part of its polyglot persistence architecture, not as a universal replacement for relational storage. Every collection, document structure, index, and transaction MUST serve a defined purpose and remain secure, measurable, maintainable, and recoverable.
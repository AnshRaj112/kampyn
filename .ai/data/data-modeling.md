# KAMPYN Data Modeling Standards

## 1. Purpose

This document defines the mandatory standards for designing, structuring, validating, storing, relating, and evolving data across KAMPYN.

Data modeling is a foundational architectural responsibility. Poorly designed data structures can lead to inconsistent business logic, duplicated records, inefficient queries, data corruption, security vulnerabilities, and expensive migrations.

Every data model MUST be designed with consideration for:

- Domain ownership and boundaries.
- Business invariants and data integrity.
- Multi-tenancy and isolation.
- Query patterns and access frequency.
- Relationships and lifecycle.
- Transactional consistency.
- Concurrency and idempotency.
- Security and privacy.
- Scalability and performance.
- Data retention and deletion.
- Migration and backward compatibility.
- SaaS and self-hosted deployment requirements.

Data models MUST represent the business domain clearly and MUST NOT be designed solely around the immediate needs of a single API endpoint or UI component.

This document applies to PostgreSQL, MongoDB, Redis, OpenSearch, object storage, event payloads, API contracts, SDKs, and other data representations used within KAMPYN.

## 2. Core Principles

All data modeling decisions MUST follow these principles:

1. **Domain ownership** — Every entity and data structure MUST have a clearly defined owning domain.
2. **Single source of truth** — Every authoritative data element MUST have a clearly identified source of truth.
3. **Explicit relationships** — Relationships between entities MUST be deliberate and understandable.
4. **Integrity by design** — Invalid states MUST be prevented wherever practical.
5. **Tenant isolation** — Tenant-scoped data MUST carry and enforce appropriate tenant context.
6. **Minimal duplication** — Data MUST NOT be duplicated without a justified consistency strategy.
7. **Query-aware design** — Models MUST support known access patterns without sacrificing domain correctness.
8. **Appropriate normalization** — Data MUST be normalized or denormalized according to ownership, consistency, and workload requirements.
9. **Explicit lifecycle** — Creation, modification, archival, deletion, and retention MUST be considered.
10. **Safe evolution** — Models MUST support controlled changes and migrations.
11. **Strong typing** — Data structures MUST use precise types and explicit constraints.
12. **Security and privacy** — Sensitive information MUST be classified and protected.
13. **Concurrency safety** — Models MUST support safe updates under concurrent access.
14. **Technology independence** — Domain concepts MUST NOT be unnecessarily coupled to a specific persistence technology.
15. **Operational clarity** — Data MUST be understandable, maintainable, observable, and recoverable.

## 3. Data Ownership

Every entity, aggregate, and authoritative data structure MUST belong to a clearly identified domain.

Examples of potential KAMPYN domains include:

- Identity and authentication.
- User profiles.
- Institutions and tenants.
- Memberships and roles.
- Food courts and vendors.
- Menus and catalogues.
- Orders and payments.
- Inventory.
- Hostel and guest house bookings.
- Library occupancy.
- Washing machine scheduling.
- Shuttle services.
- Community and messaging.
- Complaints and support.
- Notifications.
- Human resources.
- Analytics and reporting.

Each domain MUST define:

- The entities it owns.
- The attributes it is authoritative for.
- The operations allowed to mutate its data.
- The relationships it maintains.
- The public contracts through which other domains interact with it.
- The events it publishes, where applicable.
- The data it consumes from other domains.
- The lifecycle and retention requirements of its data.

A domain MUST NOT directly mutate another domain's authoritative records by bypassing the owning domain's application or domain logic.

Cross-domain requirements SHOULD be fulfilled through explicit application contracts, domain services, integration interfaces, or events, depending on the consistency requirements.

## 4. Source of Truth

Every authoritative data element MUST have one identifiable source of truth.

KAMPYN's default storage responsibilities are:

| Technology | Primary responsibility |
|---|---|
| PostgreSQL | Relational, transactional, and strongly consistent business data |
| MongoDB | Document-oriented data where flexible document structures are justified |
| Redis | Caching, transient state, coordination, and other explicitly temporary workloads |
| OpenSearch | Derived search and discovery read models |
| Object storage | Files, images, attachments, exports, and large binary objects |

These are default architectural roles, not a requirement to use every technology for every domain.

The chosen storage technology MUST be justified by the domain's access patterns, consistency requirements, operational constraints, and data lifecycle.

### 4.1 Authoritative data

Authoritative data MUST be mutated through its owning domain.

Examples:

- An order's authoritative status belongs to the order domain.
- A payment's authoritative state belongs to the payment domain or its explicitly designated owner.
- Inventory availability belongs to the inventory domain.
- User identity belongs to the identity domain.
- Search documents are derived from their authoritative records.

### 4.2 Derived data

Derived data includes:

- Search indexes.
- Cached responses.
- Materialized views.
- Analytics aggregates.
- Reporting snapshots.
- Denormalized read models.

Derived data MUST have a documented source, synchronization strategy, and recovery procedure where necessary.

Derived data MUST NOT be treated as authoritative merely because it is faster to access.

## 5. Entity Design

An entity represents a distinct domain concept with identity and a lifecycle.

Every entity MUST have:

- A clear business meaning.
- A stable identity.
- An owning domain.
- Defined attributes.
- Explicit relationships.
- Validation rules.
- Lifecycle rules.
- Appropriate authorization and tenant scope.
- Defined mutation responsibilities.

Entities SHOULD represent meaningful domain concepts rather than generic collections of unrelated fields.

### 5.1 Entity identity

Every persistent entity MUST have a stable identifier.

Identifiers SHOULD:

- Be unique within their defined scope.
- Remain stable throughout the entity's lifecycle.
- Be independent of mutable business attributes.
- Avoid exposing sensitive or sequential internal information unnecessarily.
- Be consistently represented across APIs, events, and persistence boundaries.

UUIDs or another established identifier strategy MAY be used according to repository standards.

Business identifiers, such as order numbers, student numbers, or booking references, MUST be distinguished from internal primary identifiers.

Business identifiers MUST have explicit uniqueness and scope requirements.

### 5.2 Identity immutability

Entity identifiers MUST NOT change after creation unless a documented migration or exceptional domain requirement explicitly permits it.

Mutable properties MUST NOT be used as primary identity.

Changing an entity's display name, email address, status, or other business attribute MUST NOT change its internal identity.

### 5.3 Entity boundaries

Entities SHOULD be designed around business invariants and lifecycle responsibilities.

A single entity MUST NOT become an unrestricted container for unrelated domain concepts.

If an entity accumulates unrelated responsibilities, its boundaries MUST be reviewed.

## 6. Attribute Design

Every attribute MUST have a clear semantic meaning and an appropriate type.

Attributes SHOULD:

- Use precise names.
- Represent one concept.
- Have explicit nullability.
- Use suitable data types.
- Define valid ranges or allowed values.
- Avoid ambiguous representations.
- Have documented units where applicable.
- Be validated at the appropriate boundaries.

Avoid generic names such as `data`, `value`, `info`, or `type` when a domain-specific name is available.

### 6.1 Nullability

Nullable fields MUST have a defined meaning.

`NULL` MUST NOT be used interchangeably to represent:

- Unknown values.
- Missing values.
- Not applicable.
- Not yet calculated.
- Explicitly cleared values.
- Unavailable external data.

Where these states have distinct business meanings, the model MUST represent them explicitly.

Required fields SHOULD be non-nullable in persistence unless a valid lifecycle or migration requirement justifies otherwise.

### 6.2 Defaults

Defaults MUST reflect valid domain behavior.

Database defaults MUST NOT silently hide missing application logic or required client input.

Defaults MUST be consistent across:

- Application creation logic.
- Database constraints.
- Event serialization.
- API contracts.
- Migration behavior.

### 6.3 Enumerations

Finite, well-defined states SHOULD use explicit enumerations or equivalent constrained types.

Enums MUST:

- Have stable, meaningful values.
- Represent only valid states.
- Avoid ambiguous abbreviations.
- Define compatibility behavior for consumers.
- Be safely evolved when new values are introduced.

Externally supplied enum values MUST be validated.

Unknown enum values MUST be handled deliberately, especially in events, SDKs, and mixed-version deployments.

## 7. Relationships and References

Relationships MUST accurately represent domain ownership and lifecycle.

Common relationship types include:

- One-to-one.
- One-to-many.
- Many-to-many.
- Parent-child.
- Aggregate membership.
- External references.
- Derived relationships.

Every relationship MUST define:

- Its cardinality.
- Its ownership.
- Its lifecycle.
- Whether it is optional or required.
- Whether it can change.
- Its deletion behavior.
- Its tenant scope.
- Its integrity requirements.

### 7.1 Foreign keys

Where relational persistence is used, foreign keys SHOULD enforce relationships when appropriate.

Foreign-key constraints MUST be used where they provide meaningful integrity guarantees and are compatible with the domain's storage and lifecycle design.

Application-level checks MUST NOT be treated as a complete substitute for database constraints when concurrent operations could violate the relationship.

### 7.2 Cross-domain references

Cross-domain relationships SHOULD use stable identifiers and explicit contracts.

A domain SHOULD NOT duplicate another domain's complete authoritative entity merely to establish a relationship.

If another domain's attributes are required for read performance, a deliberately maintained read model MAY be used.

Cross-domain references MUST define behavior for:

- Missing referenced records.
- Deleted or archived records.
- Tenant mismatches.
- Stale derived attributes.
- Provider or service unavailability.

### 7.3 Many-to-many relationships

Many-to-many relationships SHOULD use explicit join entities or collections where the relationship has its own attributes, lifecycle, permissions, or audit requirements.

Examples include:

- User membership in an institution.
- User assignment to a role.
- Vendor participation in a food court.
- Users joining a community.
- Staff assigned to operational units.

A join entity MUST be treated as a first-class domain concept when it carries meaningful business behavior.

## 8. Aggregate and Transaction Boundaries

Aggregates MUST protect a coherent set of business invariants.

An aggregate SHOULD have:

- A clearly defined root.
- A controlled mutation boundary.
- Explicit consistency requirements.
- Valid state transitions.
- A defined transaction scope.
- Clear ownership of contained entities.

Aggregate boundaries MUST NOT be chosen solely for convenience or based on the number of tables involved.

### 8.1 Aggregate roots

The aggregate root MUST control operations that could violate its invariants.

External modules SHOULD interact with aggregates through their public application or domain interfaces rather than mutating internal state directly.

### 8.2 Transaction boundaries

Transactions MUST be aligned with consistency requirements.

A transaction SHOULD cover the smallest coherent unit of work that must succeed or fail atomically.

Transactions MUST NOT be held open unnecessarily during:

- External network calls.
- Long-running file processing.
- User interaction.
- Unbounded computation.
- Uncontrolled background execution.

Workflows that span independent domains SHOULD use explicit orchestration, idempotency, events, or compensating actions rather than assuming a single distributed transaction.

### 8.3 Aggregate size

Aggregates SHOULD remain small enough to support understandable invariants, manageable transactions, and efficient persistence.

Large aggregates MUST be reviewed for unnecessary coupling, excessive locking, and contention.

## 9. Normalization and Denormalization

Data normalization and denormalization MUST be chosen deliberately.

### 9.1 Normalization

Normalization SHOULD be used when it:

- Reduces unintended duplication.
- Preserves authoritative ownership.
- Improves consistency.
- Makes updates safer.
- Clarifies relationships.
- Prevents conflicting representations.

Transactional business data SHOULD generally favor normalized relational modeling where it fits the domain.

### 9.2 Denormalization

Denormalization MAY be used when it provides a justified benefit for:

- High-volume reads.
- Search and discovery.
- Analytics and reporting.
- Read-heavy application views.
- Bounded access patterns.
- Reduced expensive joins or repeated computation.

Every denormalized structure MUST define:

- Its authoritative source.
- Its update mechanism.
- Its consistency expectations.
- Its staleness tolerance.
- Its deletion behavior.
- Its rebuild or reconciliation strategy.
- Its monitoring requirements.

Denormalization MUST NOT create multiple uncontrolled mutation paths for authoritative data.

### 9.3 Duplication control

Duplicated attributes MUST have an explicit ownership and synchronization rule.

The system MUST NOT rely on engineers manually keeping duplicated authoritative values synchronized across unrelated records.

## 10. PostgreSQL Modeling Standards

PostgreSQL SHOULD be used for relational business data requiring constraints, joins, and transactional consistency.

### 10.1 Schema design

PostgreSQL schemas MUST:

- Use clear table and column names.
- Use appropriate native data types.
- Define primary keys.
- Define required foreign keys.
- Define uniqueness constraints.
- Define nullability deliberately.
- Define check constraints for appropriate invariants.
- Avoid unnecessary generic or polymorphic structures.
- Keep ownership and relationship semantics explicit.

### 10.2 Primary and foreign keys

Every persistent relational entity MUST have a primary key.

Foreign keys SHOULD be used to enforce meaningful referential integrity.

Composite keys MAY be used where they accurately represent identity or uniqueness, but MUST NOT make public contracts unnecessarily difficult to evolve.

### 10.3 Constraints

Critical data invariants SHOULD be enforced at the database layer whenever feasible.

Examples include:

- Unique business references.
- Non-negative quantities.
- Valid status values.
- Required relationships.
- Tenant-scoped uniqueness.
- Valid time or range relationships.

Application validation and database constraints SHOULD complement each other.

### 10.4 Indexes

Indexes MUST be designed around actual query and mutation patterns.

Consider:

- Primary and unique indexes.
- Foreign-key access patterns.
- Tenant-scoped queries.
- Frequently used filters.
- Sorting and pagination.
- Join conditions.
- Partial indexes where justified.
- Composite indexes where appropriate.

Indexes MUST NOT be added indiscriminately. Unnecessary indexes increase storage usage and write costs.

Index effectiveness SHOULD be validated through query plans and workload measurements for performance-sensitive operations.

### 10.5 Query modeling

Queries SHOULD:

- Retrieve only necessary columns.
- Use appropriate filtering.
- Avoid N+1 access patterns.
- Use bounded pagination.
- Avoid uncontrolled full-table scans.
- Use batch operations where suitable.
- Apply tenant and authorization scope.
- Use deterministic ordering where pagination requires it.

### 10.6 JSON columns

JSON or JSONB columns MAY be used for genuinely flexible or semi-structured attributes.

They MUST NOT be used to avoid defining stable, frequently queried, or integrity-critical relational attributes.

Fields stored in JSON MUST have clear ownership, validation, and evolution rules.

## 11. MongoDB Modeling Standards

MongoDB MAY be used for document-oriented workloads where flexible document structures provide a concrete benefit.

### 11.1 Document design

Documents MUST:

- Represent a coherent domain concept.
- Have clear ownership.
- Use stable identifiers.
- Define required and optional fields.
- Respect document size limits.
- Avoid unbounded embedded collections.
- Define update and concurrency behavior.
- Preserve tenant scope where applicable.

### 11.2 Embedding and referencing

Embed related data when:

- It belongs to the same lifecycle.
- It is usually accessed together.
- Its size remains bounded.
- Atomic document updates are useful.
- Duplication is controlled.

Use references when:

- Data has an independent lifecycle.
- Relationships are shared across many documents.
- Embedded collections could grow without bound.
- Independent updates are frequent.
- Cross-document consistency requires explicit coordination.

Embedding MUST NOT be used indiscriminately to avoid modeling relationships.

### 11.3 Schema evolution

Flexible schemas MUST NOT become uncontrolled schemas.

MongoDB collections SHOULD use application-level schema validation and, where appropriate, database validation rules.

Schema evolution MUST account for existing documents, mixed versions, backfills, and consumers.

### 11.4 Indexing

Indexes MUST reflect real query patterns.

Review:

- Unique constraints.
- Compound indexes.
- Tenant-scoped access.
- Sort and filter patterns.
- TTL indexes where appropriate.
- Index size and write overhead.

Queries MUST be checked for unbounded scans and unexpected collection-wide operations.

### 11.5 Atomicity and concurrency

MongoDB single-document atomicity MUST be used deliberately.

Operations spanning multiple documents MUST define appropriate transactional, optimistic concurrency, or compensating behavior.

The application MUST NOT assume that independently updated documents remain consistent without an explicit strategy.

## 12. Redis Data Modeling

Redis MUST be used for clearly defined transient, caching, or coordination responsibilities.

Every Redis data structure MUST define:

- Its purpose.
- Its owner.
- Its key format.
- Its value structure.
- Its expiration policy.
- Its consistency expectations.
- Its invalidation behavior.
- Its memory limits.
- Its failure behavior.

### 12.1 Key design

Redis keys MUST be predictable, unambiguous, and appropriately scoped.

Tenant-scoped keys MUST include tenant context where applicable.

Key formats SHOULD include domain and purpose information to reduce collisions and support operational diagnosis.

### 12.2 Expiration

Transient and cached values SHOULD have explicit expiration behavior.

TTL values MUST reflect acceptable staleness and operational needs.

Redis MUST NOT be treated as durable storage unless a specific architecture explicitly establishes and documents that responsibility.

### 12.3 Coordination

Distributed locks, counters, rate limits, and coordination state MUST have explicit correctness requirements.

A Redis lock MUST NOT be treated as a replacement for database constraints or transactional correctness.

Coordination mechanisms MUST consider expiration, process failure, retries, and ownership.

## 13. OpenSearch Data Modeling

OpenSearch MUST be treated as a derived read model for search and discovery.

### 13.1 Search document design

Search documents SHOULD contain only the fields required for supported search, filtering, ranking, and display workflows.

Each document MUST have:

- A stable document identity.
- A source domain.
- A source record identifier.
- Tenant context where applicable.
- A defined schema version or compatible evolution strategy.
- Appropriate searchable and filterable fields.
- A synchronization and deletion strategy.

### 13.2 Mapping design

OpenSearch mappings MUST be deliberate.

Fields SHOULD have explicit types appropriate to their intended operations.

Text, keyword, numeric, date, geographic, and nested fields MUST be selected according to real query requirements.

Dynamic mapping MUST be controlled where uncontrolled field growth or type conflicts could cause operational problems.

### 13.3 Security and authorization

Search documents MUST preserve tenant and resource access boundaries.

Every search query MUST enforce appropriate authorization and tenant filtering.

Search results MUST NOT expose restricted records through:

- Direct matches.
- Suggestions.
- Facets.
- Aggregations.
- Counts.
- Sorting behavior.
- Highlighted content.

### 13.4 Synchronization

Search indexes MUST have a defined synchronization mechanism, such as transactional outbox-driven updates or another approved process.

Synchronization MUST handle:

- Duplicate events.
- Delayed updates.
- Out-of-order delivery.
- Deletions.
- Reindexing.
- Failed indexing.
- Reconciliation.
- Schema changes.

Search documents MUST be rebuildable from authoritative sources where the domain requires that capability.

## 14. Object Storage and File Metadata

Binary content SHOULD be stored in object storage rather than large database fields, unless a documented use case justifies another design.

Database records MAY store file metadata, including:

- Stable file identifier.
- Tenant ownership.
- Resource association.
- Storage reference.
- Content type.
- File size.
- Checksum where appropriate.
- Upload status.
- Access classification.
- Creation and retention metadata.
- Processing status.

Storage references MUST NOT be treated as authorization tokens.

Access MUST be authorized independently of knowledge of an object key or URL.

File metadata MUST have a defined lifecycle consistent with the associated resource and retention requirements.

## 15. Multi-Tenant Data Modeling

Tenant isolation MUST be incorporated into data models from the beginning.

### 15.1 Tenant identifiers

Tenant-scoped entities MUST carry a stable tenant identifier or have an explicit, verifiable tenant relationship.

Tenant identifiers MUST:

- Have a consistent representation.
- Be immutable under normal operations.
- Be validated against trusted context.
- Be used consistently across relevant data stores.
- Be available to required audit and operational workflows.

### 15.2 Tenant-scoped uniqueness

Uniqueness requirements MUST define whether uniqueness is:

- Global.
- Tenant-scoped.
- Scoped to another business entity.
- Scoped to a time period or lifecycle.

For tenant-scoped business identifiers, constraints SHOULD include tenant context.

### 15.3 Tenant relationships

Relationships between tenant-scoped entities MUST ensure tenant consistency.

The system MUST prevent associations that connect resources from different tenants unless an explicitly authorized cross-tenant relationship is part of the domain model.

### 15.4 Tenant-aware queries

Tenant scope MUST be applied consistently to reads, writes, updates, deletes, and aggregate operations.

Repositories and application services MUST NOT rely on optional caller-provided tenant filters for security.

### 15.5 Shared and dedicated storage

Shared database, schema-based, and dedicated database models MAY be supported according to deployment needs.

Domain models SHOULD remain independent of a particular tenant deployment strategy wherever practical.

Tenant routing and physical storage selection MUST remain explicit infrastructure responsibilities.

## 16. Time and Date Modeling

Time-related attributes MUST use precise and consistent semantics.

Every time field MUST distinguish its meaning, such as:

- An absolute instant.
- A local date.
- A local time.
- A timezone-aware scheduled time.
- A duration.
- A time interval.
- A business date.

### 16.1 Timezones

Absolute timestamps SHOULD be stored using a consistent timezone-aware representation, typically UTC.

Local scheduling requirements MUST retain the relevant timezone identifier where future local-time interpretation matters.

The system MUST account for daylight-saving transitions where relevant to the institution or deployment.

### 16.2 Intervals

Time intervals MUST define whether their endpoints are inclusive or exclusive.

Booking, scheduling, and occupancy models MUST use consistent overlap and availability rules.

### 16.3 Clock handling

Business logic MUST avoid relying on inconsistent local machine clocks.

Time-sensitive operations SHOULD use an explicit clock abstraction where testability and determinism require it.

## 17. Money and Quantities

Financial values and measurable quantities MUST use precise representations.

### 17.1 Money

Money MUST NOT use binary floating-point types for authoritative financial calculations.

Money models SHOULD define:

- Amount.
- Currency.
- Precision and rounding rules.
- Tax or fee treatment where applicable.
- Authoritative calculation rules.
- Display formatting separately from stored values.

Currency MUST be explicit where values can differ by currency.

### 17.2 Quantities

Quantities MUST have clear units and precision.

Inventory, measurements, durations, and rates MUST NOT rely on ambiguous numeric fields.

Conversions MUST be explicit and consistent.

### 17.3 Calculated values

Calculated financial or operational values MUST define whether they are:

- Authoritative stored values.
- Derived values.
- Snapshots taken at a specific point in time.
- Recomputed from current source data.

Historical transactions SHOULD preserve the values required to explain their original calculation even when referenced catalogue or pricing data later changes.

## 18. State and Lifecycle Modeling

Every persistent entity MUST have a defined lifecycle appropriate to its domain.

Lifecycle design SHOULD consider:

- Creation.
- Activation.
- Modification.
- Suspension.
- Completion.
- Cancellation.
- Expiration.
- Archival.
- Deletion.

### 18.1 Status fields

Status fields MUST represent meaningful domain states.

Avoid ambiguous statuses that combine unrelated concerns, such as lifecycle, payment, fulfillment, and moderation, into one field.

Independent state dimensions SHOULD be modeled separately when they evolve independently.

### 18.2 State transitions

Valid transitions MUST be explicit.

State changes MUST be performed through the owning domain's authorized operations.

Critical transitions SHOULD record appropriate metadata, such as actor, timestamp, reason, or source.

### 18.3 Soft deletion

Soft deletion MAY be used when recovery, auditability, legal retention, or business history requires it.

Soft-deleted records MUST have clear access and visibility rules.

Queries MUST consistently account for deleted records according to the operation's purpose.

Soft deletion MUST NOT be treated as a complete substitute for retention and permanent deletion policies.

### 18.4 Archival

Archival MAY be used to manage historical or infrequently accessed data.

Archived data MUST have explicit retrieval, access, retention, and restoration behavior.

Archival MUST NOT silently break historical relationships or audit requirements.

## 19. Audit and History Modeling

Auditability MUST be considered for security-sensitive, financial, administrative, and operational workflows.

Audit records SHOULD capture the minimum information needed to understand:

- Who performed the action.
- What action occurred.
- Which resource was affected.
- When the action occurred.
- Which tenant context applied.
- Whether the action succeeded or failed.
- Why the action occurred, where required.

Audit records MUST be protected against unauthorized modification and access.

Audit history MUST NOT unnecessarily copy passwords, tokens, private messages, or other sensitive payloads.

Where historical state is required, the model MUST distinguish between current authoritative state and immutable or append-only history.

## 20. API and Event Data Models

Persistence models, API DTOs, event payloads, and SDK types MUST remain appropriately separated.

### 20.1 API models

API contracts MUST represent externally supported behavior rather than exposing persistence details.

API models MUST define:

- Field names and types.
- Required and optional fields.
- Nullability.
- Validation rules.
- Authorization-sensitive attributes.
- Versioning and compatibility expectations.

Sensitive fields MUST be explicitly excluded from public responses.

### 20.2 Event models

Event payloads MUST represent meaningful facts or state changes.

Events SHOULD contain stable identifiers, event type, schema version, tenant context where applicable, occurrence time, and correlation metadata as appropriate.

Event schemas MUST evolve compatibly or use a documented versioning strategy.

Event payloads MUST avoid unnecessary personal or sensitive information.

### 20.3 Model transformation

Transformations between persistence models, domain objects, API DTOs, and event payloads MUST be explicit.

Automatic mapping MAY be used where it remains clear, safe, and reviewable.

Mappings MUST NOT silently bypass validation, authorization, or domain invariants.

Mass assignment of externally supplied fields into persistence models MUST be avoided.

## 21. Validation and Data Integrity

Data validation MUST be applied at appropriate trust boundaries and reinforced by domain and persistence constraints.

Validation MUST consider:

- Types and structure.
- Required fields.
- Length and size.
- Numeric ranges.
- Allowed values.
- Cross-field conditions.
- Relationships.
- Tenant consistency.
- State transitions.
- Business invariants.
- External data validity.

Validation logic SHOULD be reused where semantics are identical, but transport validation MUST remain distinct from domain invariants when their responsibilities differ.

Database constraints SHOULD protect critical invariants against concurrent requests and alternate application paths.

Invalid data MUST be rejected explicitly rather than silently coerced into an unintended state.

## 22. Concurrency and Data Consistency

Data models MUST support safe behavior under concurrent access.

Every concurrently mutable entity MUST have a defined concurrency strategy appropriate to its invariants.

Strategies MAY include:

- Database transactions.
- Unique and check constraints.
- Row-level locking.
- Optimistic concurrency.
- Version fields.
- Atomic update operations.
- Idempotency keys.
- Distributed coordination where justified.

### 22.1 Optimistic concurrency

Version fields or equivalent mechanisms SHOULD be considered when conflicting updates are possible and conflicts can be detected safely.

Conflicting updates MUST NOT silently overwrite important changes.

### 22.2 Pessimistic concurrency

Locks MAY be used where required to preserve critical invariants.

Lock scope and duration MUST be minimized.

Deadlock risks, lock contention, timeout behavior, and retry safety MUST be considered.

### 22.3 Eventual consistency

Eventual consistency MAY be used for derived data and independent asynchronous workflows.

Every eventually consistent model MUST define:

- Acceptable staleness.
- Expected propagation behavior.
- Failure and retry handling.
- Reconciliation or rebuild behavior.
- User-visible consistency expectations.

Critical business decisions MUST NOT rely on stale derived data when authoritative transactional validation is required.

## 23. Idempotency and Deduplication

Operations that may be retried, replayed, or delivered more than once MUST define idempotency behavior.

Idempotency models SHOULD specify:

- The operation scope.
- The idempotency key.
- The uniqueness boundary.
- The stored outcome or deduplication record.
- Expiration or retention.
- Conflict behavior.
- Retry behavior.
- Tenant scope.

Idempotency keys MUST NOT be treated as authentication or authorization credentials.

Deduplication MUST be enforced through appropriate atomic or transactional mechanisms rather than unsafe check-then-insert logic.

## 24. Data Migration and Schema Evolution

Data models MUST evolve through controlled, reviewable, and recoverable changes.

Every significant schema change MUST consider:

- Existing stored data.
- Active application versions.
- API and event consumers.
- Migration duration.
- Locking and operational impact.
- Backfill requirements.
- Index creation.
- Compatibility.
- Rollback or forward recovery.
- Data validation.
- Deployment order.

### 24.1 Expand-and-contract

For high-impact changes, prefer a staged migration approach:

1. Introduce backward-compatible schema additions.
2. Deploy code that supports both old and new representations where necessary.
3. Backfill or transform existing data safely.
4. Verify migration completeness and correctness.
5. Move reads and writes to the new representation.
6. Remove obsolete structures only after compatibility is confirmed.

### 24.2 Backfills

Backfills MUST:

- Be bounded and resumable where practical.
- Avoid excessive database load.
- Track progress.
- Be safe to retry.
- Validate transformed results.
- Provide a recovery strategy for partial completion.

### 24.3 Destructive changes

Destructive schema changes MUST NOT occur before dependencies and compatibility requirements have been reviewed.

Data loss MUST NOT be treated as an acceptable default rollback strategy.

## 25. Data Retention, Privacy, and Deletion

Every domain MUST define appropriate data lifecycle requirements.

Data models SHOULD identify:

- Retention period or policy.
- Legal or institutional retention constraints.
- Deletion eligibility.
- Archival requirements.
- Data export requirements.
- Personal data classification.
- Downstream copies and derived representations.

Deletion workflows MUST consider databases, caches, search indexes, object storage, analytics, exports, and asynchronous jobs.

Where data is retained for legal, operational, or recovery reasons, access MUST remain appropriately restricted and the retention behavior MUST be documented.

Data models MUST support privacy requirements without compromising necessary transactional integrity or auditability.

## 26. Search, Analytics, and Reporting Models

Search, analytics, and reporting structures MUST be designed for their specific workloads without undermining authoritative domain ownership.

### 26.1 Search models

Search documents MUST be derived from authoritative data and MUST preserve relevant access controls.

Search model changes MUST consider mapping compatibility, reindexing, deletion propagation, and recovery.

### 26.2 Analytics models

Analytics models SHOULD be separated from transactional workloads when scale or workload characteristics justify it.

Aggregations MUST define their source data, time semantics, refresh frequency, and expected accuracy.

Analytics MUST respect tenant isolation, access permissions, privacy requirements, and data retention.

### 26.3 Reporting snapshots

Snapshots MAY be used when historical reporting must preserve the values that existed at a specific time.

Snapshot models MUST define their generation point, source, retention, and correction behavior.

Reports MUST NOT silently present stale or incomplete data as authoritative current state.

## 27. Large-Scale Data Modeling

High-volume datasets MUST be modeled with attention to data access, processing, and storage costs.

Consider:

- Bounded record sizes.
- Appropriate partitioning.
- Efficient indexes.
- Streaming and chunked processing.
- Batch reads and writes.
- Retention and archival.
- Data locality.
- Query selectivity.
- Backpressure.
- Resource limits.
- Recovery from interrupted processing.

Partitioning or sharding MUST be introduced only when supported by actual workload and operational requirements.

Large collections and relationships MUST NOT rely on unbounded in-memory materialization.

Data models SHOULD support incremental processing where complete recomputation would be unnecessarily expensive.

## 28. Naming Conventions

Data structures MUST use consistent, descriptive naming conventions aligned with repository standards.

Names SHOULD:

- Reflect domain terminology.
- Distinguish identifiers from display values.
- Identify units where ambiguity is possible.
- Distinguish timestamps by semantic meaning.
- Avoid cryptic abbreviations.
- Avoid overloaded generic terms.
- Remain consistent across related contracts where appropriate.

Database naming conventions MUST be consistent within the selected persistence technology.

Public API and event names MUST remain stable unless intentionally versioned.

## 29. Data Model Documentation

Significant data models MUST be documented sufficiently for engineers to understand their meaning and constraints.

Documentation SHOULD include:

- Domain ownership.
- Entity purpose.
- Attributes and types.
- Nullability and defaults.
- Relationships.
- Constraints.
- Tenant scope.
- Source of truth.
- Lifecycle and state transitions.
- Authorization requirements.
- Query patterns.
- Indexing considerations.
- Concurrency requirements.
- Retention and deletion behavior.
- Migration and compatibility notes.

Complex models SHOULD use entity-relationship diagrams, state diagrams, or other appropriate representations.

Documentation MUST remain consistent with the implemented schema and application behavior.

## 30. Data Model Review Checklist

Before approving a significant data model or schema change, reviewers MUST consider:

- [ ] Is the domain ownership explicit?
- [ ] Is the source of truth identified?
- [ ] Does the model represent a clear business concept?
- [ ] Are entity identities stable?
- [ ] Are attributes precisely typed and named?
- [ ] Is nullability intentional?
- [ ] Are relationships and lifecycle rules explicit?
- [ ] Are required constraints enforced?
- [ ] Is normalization or denormalization justified?
- [ ] Are tenant boundaries enforced?
- [ ] Are authorization requirements understood?
- [ ] Are query patterns supported efficiently?
- [ ] Are indexes appropriate?
- [ ] Are concurrency and idempotency addressed?
- [ ] Are transaction boundaries correct?
- [ ] Are derived data and synchronization handled?
- [ ] Are sensitive fields classified and protected?
- [ ] Are retention and deletion requirements considered?
- [ ] Are migration and compatibility risks understood?
- [ ] Is the model testable and observable?
- [ ] Is the design maintainable and operationally practical?
- [ ] Is the documentation sufficient?

## 31. Data Modeling Anti-Patterns

The following practices MUST be avoided unless a specific, documented requirement justifies an exception:

- One generic table or collection for unrelated domain entities.
- Multiple competing sources of truth.
- Uncontrolled duplication of authoritative data.
- Unbounded embedded arrays or document growth.
- Storing stable, frequently queried attributes only in opaque JSON.
- Using mutable business attributes as entity identifiers.
- Missing tenant scope in tenant-owned records.
- Client-controlled tenant filters as the sole isolation mechanism.
- Unconstrained polymorphic references without integrity checks.
- Ambiguous nullable fields.
- Storing money as binary floating-point values.
- Mixing unrelated lifecycle states into one status field.
- Relying only on application checks for critical concurrent invariants.
- Treating caches or search indexes as authoritative.
- Unbounded queries or full collection scans for routine access.
- Direct cross-domain mutation of authoritative records.
- Destructive migrations without a recovery plan.
- Storing large files directly in transactional records without justification.
- Exposing persistence models as public API contracts.
- Adding fields without a defined owner, purpose, or lifecycle.
- Designing schemas solely around one screen or endpoint.

## 32. Definition of Done for Data Modeling

A data modeling task is complete only when all applicable conditions are satisfied:

- [ ] Domain ownership and data responsibility are clear.
- [ ] Source of truth is identified.
- [ ] Entity and attribute semantics are explicit.
- [ ] Identifiers and relationships are stable and valid.
- [ ] Nullability, defaults, and constraints are intentional.
- [ ] Tenant isolation is preserved.
- [ ] Data integrity and concurrency requirements are addressed.
- [ ] Storage technology is appropriate for the workload.
- [ ] Query patterns and indexes are reviewed.
- [ ] Security and privacy requirements are satisfied.
- [ ] Lifecycle, retention, and deletion are considered.
- [ ] Derived data has a synchronization and recovery strategy.
- [ ] Migrations and compatibility are addressed.
- [ ] Relevant tests and verification are performed.
- [ ] Documentation is updated.
- [ ] The model has received the required review.
- [ ] Remaining risks and limitations are disclosed.

## 33. Final Data Modeling Principle

KAMPYN data models MUST express business reality clearly while preserving integrity, security, ownership, and the ability to evolve.

A data model is not merely a schema that stores information. It defines how the system understands identities, relationships, permissions, state, and business invariants.

**KAMPYN MUST model data for correctness first, explicit ownership second, and efficient access without compromising either.**
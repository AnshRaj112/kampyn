# Data Migrations

## 1. Purpose

This document defines the standards for designing, implementing, reviewing, deploying, and maintaining database and data migrations across KAMPYN.

Migrations are controlled changes to persistent data structures, schemas, constraints, indexes, and stored data. They must preserve data integrity, support application compatibility, minimize downtime, and provide predictable recovery paths.

The objectives are to:
- Ensure safe and repeatable database schema evolution.
- Preserve data integrity during structural and semantic changes.
- Support zero-downtime and rolling deployments where required.
- Maintain compatibility between application versions and database schemas.
- Enable controlled migrations across PostgreSQL, MongoDB, Redis, and OpenSearch.
- Support SaaS multi-tenancy and university self-hosted deployments.
- Make migration execution observable, auditable, and recoverable.
- Prevent accidental data loss, corruption, and service disruption.

Every migration MUST be treated as a production change, regardless of whether it is initially developed or tested in a local environment.

---

## 2. Core Principles

### 2.1 Data Integrity First

- Migrations MUST preserve existing valid data unless data removal is explicitly required and approved.
- Data transformations MUST preserve documented business invariants.
- Referential integrity MUST be maintained throughout migration execution.
- Destructive operations MUST have an explicit impact assessment and recovery plan.
- Data correctness MUST take precedence over migration speed.

### 2.2 Version-Controlled Changes

- All production schema changes MUST be version-controlled.
- Migration files MUST be committed alongside the application changes that depend on them, where appropriate.
- Production database changes MUST NOT be performed through undocumented manual SQL or console operations.
- Migration history MUST be preserved and traceable.
- Each migration MUST have a stable, unique identifier.

### 2.3 Backward Compatibility

- Migrations SHOULD maintain compatibility with the currently deployed application during rolling deployments.
- Breaking schema changes MUST use a controlled transition strategy.
- Application code MUST NOT assume a new schema is available until the migration has successfully completed.
- Old application versions MUST remain compatible with transitional schemas for the required deployment window.

### 2.4 Small, Reversible Steps

- Large schema changes SHOULD be split into small, independently verifiable migrations.
- Migrations SHOULD be designed to support safe retries.
- Reversal strategies MUST be documented, even when an automated down migration is unsafe or impossible.
- Destructive changes SHOULD be separated from additive changes.
- Migration scope MUST be limited to the intended schema or data transformation.

### 2.5 Explicit Ownership

- Every migration MUST have a clearly identifiable owning module or service.
- Only the owning module or service SHOULD modify its authoritative data structures.
- Cross-module schema changes MUST respect domain ownership and dependency boundaries.
- Shared infrastructure schemas MUST have explicit ownership and change procedures.

### 2.6 Evidence-Based Execution

- Migration plans MUST be based on the current schema, data shape, data volume, and application dependencies.
- Migration performance MUST be evaluated against realistic data volumes for high-impact changes.
- Completion MUST be verified through schema inspection, data validation, and migration status.
- A migration MUST NOT be reported as successful solely because the migration command returned without an error.

---

## 3. Migration Scope

Migrations may include:

- Creating, modifying, or removing tables and collections.
- Adding, changing, or removing columns and document fields.
- Creating or modifying primary keys, foreign keys, and unique constraints.
- Creating, modifying, or removing indexes.
- Updating enum values, data types, and validation rules.
- Backfilling or transforming existing records.
- Splitting or merging entities.
- Changing relationships or ownership boundaries.
- Introducing partitioning or other storage-layout changes.
- Evolving search index mappings and document structures.
- Changing retention, archival, or data-lifecycle structures.
- Migrating data between datastores or storage systems.
- Updating tenant-specific schemas or deployment metadata.

Migrations MUST clearly distinguish between schema changes, data transformations, infrastructure changes, and application-level compatibility changes.

---

## 4. Migration Ownership and Source of Truth

### 4.1 Authoritative Data

KAMPYN MUST maintain a clear source of truth for every persistent business entity.

- PostgreSQL is the primary relational and transactional datastore.
- MongoDB is used only for justified document-oriented workloads.
- Redis is used for transient state, caching, and coordination.
- OpenSearch stores derived search-oriented projections.
- Object storage holds binary files and associated objects.

Migrations MUST respect these responsibilities. Derived data MUST be rebuildable from its authoritative source where the architecture requires it.

### 4.2 Module Ownership

- Each domain module MUST own its authoritative schema and migration lifecycle.
- Other modules MUST NOT modify another module's tables or collections directly without an approved contract.
- Shared tables and schemas MUST have explicit ownership and migration coordination.
- Cross-module changes SHOULD use versioned contracts, events, or approved migration plans rather than hidden dependencies.

### 4.3 Multiple Services

Where multiple services share a database:

- Each service MUST have a defined schema or ownership boundary.
- Migration execution MUST be coordinated to prevent conflicting schema changes.
- Services MUST NOT independently apply competing migrations to the same objects.
- Shared database changes MUST have an explicit compatibility and rollout plan.

---

## 5. Migration Versioning and Structure

### 5.1 Versioning

- Each migration MUST have a unique, immutable identifier.
- Migration identifiers MUST be sortable and consistently formatted.
- Applied migrations MUST NOT be edited or silently replaced.
- Corrections to an applied migration MUST be delivered as a new migration.
- Migration history MUST record which changes were applied and when.

A timestamp-based naming convention is recommended.

Example:

```text
202609280001_create_users_table.sql
202609280002_add_tenant_id_to_users.sql
202609280003_create_user_tenant_email_index.sql
```

Alternative migration frameworks MAY use sequential version numbers or generated identifiers, provided ordering and uniqueness are guaranteed.

### 5.2 File Structure

A recommended structure is:

```text
migrations/
├── postgres/
│   ├── 202609280001_create_users_table.up.sql
│   ├── 202609280001_create_users_table.down.sql
│   ├── 202609280002_add_tenant_id_to_users.up.sql
│   └── 202609280002_add_tenant_id_to_users.down.sql
├── mongodb/
│   ├── 202609280001_create_user_indexes.js
│   └── 202609280002_backfill_user_fields.js
├── opensearch/
│   ├── v1/
│   │   └── users-mapping.json
│   └── v2/
│       └── users-mapping.json
└── README.md
```

The actual layout MAY follow the chosen migration tooling, but MUST preserve clear datastore boundaries and migration ownership.

### 5.3 Migration Metadata

A migration tracking system SHOULD record:

- Migration identifier.
- Migration name or description.
- Target datastore and schema.
- Execution status.
- Start and completion timestamps.
- Execution duration.
- Migration tool version where relevant.
- Checksum or equivalent integrity marker.
- Failure details, when applicable.
- Deployment or tenant scope, where applicable.

Migration metadata MUST NOT expose sensitive data or credentials.

---

## 6. Migration Design

### 6.1 Additive-First Strategy

Breaking changes SHOULD follow an additive-first migration strategy:

1. Introduce the new schema in a backward-compatible manner.
2. Deploy application code that can operate with both old and new representations where required.
3. Backfill or transform existing data.
4. Validate the new representation.
5. Transition reads and writes to the new representation.
6. Remove obsolete schema elements only after compatibility is no longer required.

This approach minimizes downtime and supports rolling application deployments.

### 6.2 Schema Changes

- Schema changes MUST be explicit and reviewable.
- New required fields MUST have a safe population strategy for existing records.
- Type changes MUST account for existing values and conversion failures.
- Column and field renames SHOULD use staged transitions rather than direct destructive renames when deployed application versions may still depend on the old names.
- Constraint changes MUST account for existing records and concurrent writes.
- Changes to defaults MUST distinguish between future inserts and existing data.

### 6.3 Data Transformations

- Data transformations MUST define their input, output, and invariants.
- Transformations MUST be deterministic where practical.
- Transformations SHOULD be idempotent so they can be safely retried.
- Invalid or unconvertible records MUST be handled explicitly.
- Data transformations MUST be observable and report progress, failures, and completion.
- Large transformations SHOULD be performed in bounded batches.
- Business logic MUST NOT be duplicated inconsistently between migration code and application services.

### 6.4 Destructive Changes

Destructive migrations include dropping columns, deleting records, removing constraints, or overwriting existing values.

They MUST:
- Have a documented reason and impact assessment.
- Identify affected records and dependent services.
- Include a verified backup or recovery strategy where applicable.
- Be reviewed by the owning engineering team.
- Be separated from additive schema changes where doing so reduces deployment risk.
- Include a verification plan.
- Be scheduled to minimize operational impact.

Data MUST NOT be deleted merely because a field is no longer referenced by the current application version.

---

## 7. PostgreSQL Migrations

### 7.1 General Requirements

- PostgreSQL schema changes MUST be delivered through version-controlled migrations.
- Migration scripts MUST be deterministic and safe to execute in the intended environment.
- Transactional behavior MUST be understood for each operation.
- Long-running operations MUST be evaluated for locking, replication, and resource impact.
- Migration scripts MUST use explicit schema and object names where ambiguity is possible.
- Schema changes MUST respect the ownership of relational entities and their constraints.

### 7.2 Table Creation

- Tables MUST have clearly defined ownership and naming conventions.
- Primary keys MUST be explicitly defined.
- Data types MUST reflect domain semantics.
- Nullability MUST be intentional.
- Foreign keys and constraints MUST be included where required for data integrity.
- Tenant-owned entities MUST include the required tenant reference.
- Audit and lifecycle fields MUST follow the data-modeling standards.

Example:

```sql
CREATE TABLE users (
    id UUID PRIMARY KEY,
    tenant_id UUID NOT NULL,
    email TEXT NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

### 7.3 Adding Columns

Adding a nullable column is generally less disruptive than adding a required column with an immediate constraint.

For a required column on an existing table:

1. Add the column in a compatible state.
2. Deploy code that populates the new column.
3. Backfill existing rows in bounded batches if required.
4. Validate that no required values are missing.
5. Apply the required constraint using an appropriate low-impact strategy.
6. Remove transitional application logic after compatibility is confirmed.

Default values and database-version-specific behavior MUST be evaluated for large tables.

### 7.4 Constraints

- Constraints MUST reflect business and relational invariants.
- Existing data MUST be validated before constraints are enforced.
- Large-table constraint validation SHOULD use low-impact mechanisms where supported.
- Foreign-key additions MUST account for existing invalid references.
- Unique constraints MUST account for duplicates and concurrent inserts.
- Constraint enforcement MUST NOT be deferred indefinitely without a documented reason.

### 7.5 Index Migrations

- Indexes MUST follow `data/indexing.md`.
- Large production indexes SHOULD use `CREATE INDEX CONCURRENTLY` when appropriate.
- Concurrent index creation MUST account for migration-tool transaction restrictions.
- Failed or invalid index builds MUST be detected and handled.
- Index removal MUST consider query dependencies and actual usage.
- Index changes MUST be included in the migration plan and operational review.

### 7.6 Type Changes

- Type conversions MUST be validated against existing values.
- Potentially lossy conversions MUST be explicitly documented.
- Large-table type changes MUST be assessed for table rewrites and locks.
- Complex transformations SHOULD use a new column, backfill, validation, and staged cutover.
- Application compatibility MUST be maintained during the transition.

### 7.7 Partitioning

- Partitioning changes MUST be justified by measured workload, retention, maintenance, or scale requirements.
- Existing data MUST be migrated or redistributed safely.
- Partition keys MUST align with query patterns and uniqueness requirements.
- Partition creation and maintenance MUST be automated where appropriate.
- Rollback and recovery procedures MUST account for partition-specific behavior.

### 7.8 Transaction Boundaries

- Migrations SHOULD use transactions where supported and appropriate.
- Operations that cannot run inside a transaction MUST be isolated and documented.
- Transaction scope MUST avoid unnecessarily long locks and resource retention.
- Migration execution MUST account for partial completion where atomicity is unavailable.
- Recovery procedures MUST be defined for operations that may leave intermediate state.

---

## 8. MongoDB Migrations

### 8.1 Schema Evolution

MongoDB's flexible schema does not remove the need for controlled schema evolution.

- Document structure changes MUST be versioned and documented.
- Application code MUST handle transitional document versions when necessary.
- Required fields and field types MUST be validated according to the module's schema policy.
- Changes to embedded documents and arrays MUST account for existing document shapes.
- Migrations MUST not assume every document conforms to the latest application schema.

### 8.2 Backfills

- Large collection backfills SHOULD process documents in bounded batches.
- Use stable pagination or range-based iteration rather than unbounded in-memory collection.
- Migration queries MUST be selective and use suitable indexes where practical.
- Updates SHOULD be idempotent and guarded by explicit conditions.
- Progress MUST be tracked so interrupted migrations can resume safely.
- Write contention and replication impact MUST be evaluated.

Example:

```javascript
db.users.updateMany(
  { schemaVersion: { $exists: false } },
  {
    $set: {
      schemaVersion: 2
    }
  }
);
```

This example is appropriate only where the operation is safe for the collection's existing document shapes and update semantics.

### 8.3 Index Changes

- MongoDB indexes MUST follow `data/indexing.md`.
- Index creation MUST consider collection size, write activity, and replication.
- Unique-index creation MUST check for duplicate data first.
- Index definitions MUST be version-controlled.
- Existing indexes MUST be reviewed before creating replacements.
- TTL index changes MUST be reviewed against data-retention requirements.

### 8.4 Document Restructuring

When moving or renaming fields:

1. Introduce compatibility for the new field or document shape.
2. Deploy code capable of reading the transitional representation.
3. Update writes to populate the new representation.
4. Backfill existing documents.
5. Validate completeness and correctness.
6. Transition reads to the new representation.
7. Remove the old field only after compatibility requirements are satisfied.

### 8.5 Collection Changes

- Collection renames, splits, merges, or replacements MUST have a documented migration and cutover strategy.
- Cross-collection data movement MUST define consistency and failure handling.
- Collection deletion MUST be explicitly approved and verified against ownership and retention policies.
- Sharded collection changes MUST account for shard-key behavior and operational constraints.

---

## 9. Redis Migrations

Redis changes MUST preserve its designated role as transient storage, cache, or coordination infrastructure.

- Key schema changes MUST be versioned where compatibility is needed.
- Key namespaces MUST be tenant-aware where applicable.
- Changes to serialization formats MUST support a controlled transition.
- TTL behavior MUST remain consistent with data-retention expectations.
- Bulk key migrations MUST be bounded and avoid blocking the server.
- Large keyspace scans MUST use suitable incremental iteration rather than blocking key enumeration.
- Cache migrations MUST account for cold-cache behavior and regeneration load.
- Distributed locks and coordination keys MUST retain correct ownership, expiry, and release semantics.
- Redis migrations MUST NOT introduce undocumented dependencies on transient data for durable business correctness.

For incompatible key-format changes, use versioned namespaces or a dual-read/dual-write transition where appropriate, followed by cleanup of obsolete keys.

---

## 10. OpenSearch Migrations

### 10.1 Index Versioning

- OpenSearch mappings MUST be versioned.
- Incompatible mapping changes SHOULD create a new physical index.
- Index aliases SHOULD provide a stable application-facing reference.
- Mapping evolution MUST account for indexed documents and query compatibility.
- Existing search documents MUST be migrated or rebuilt from authoritative data as required.

### 10.2 Reindexing

A reindex operation MUST define:

- Source index and destination index.
- Mapping and analyzer changes.
- Data source and authoritative ownership.
- Strategy for changes occurring during reindexing.
- Validation criteria.
- Alias cutover procedure.
- Rollback window and cleanup policy.

### 10.3 Safe Cutover

A typical index replacement process is:

1. Create the new versioned index.
2. Start reindexing or rebuilding from the authoritative source.
3. Capture and apply concurrent updates using the approved synchronization mechanism.
4. Validate document counts, mappings, tenant coverage, and representative search results.
5. Switch the read alias to the new index.
6. Monitor query behavior, errors, and indexing lag.
7. Retain the previous index for the approved rollback period.
8. Remove the old index after validation and retention requirements are satisfied.

### 10.4 Consistency

- Reindexing MUST account for events or writes occurring during the operation.
- Duplicate and stale documents MUST be detected and reconciled.
- Indexing operations SHOULD be idempotent.
- Failed reindex operations MUST have a defined retry or rebuild procedure.
- Search-index migration MUST NOT silently overwrite authoritative business data.

---

## 11. Data Backfills and Large-Scale Transformations

### 11.1 General Requirements

Large backfills MUST be treated as operational workloads, not ordinary one-time scripts.

- Backfills MUST have a defined scope and selection predicate.
- Data MUST be processed in bounded batches.
- Memory consumption MUST remain bounded.
- Progress and completion MUST be measurable.
- Failed records MUST be tracked and handled explicitly.
- Retry behavior MUST be deterministic and safe.
- Backfills MUST be designed to coexist with active application reads and writes where required.

### 11.2 Batch Processing

- Batch size MUST be selected based on observed resource consumption, lock duration, and throughput.
- Batch processing SHOULD use stable ordering or key ranges to avoid repeatedly scanning processed data.
- Transactions SHOULD remain bounded to prevent long-lived locks.
- Backfills MUST avoid loading the entire target dataset into memory.
- Batch size MAY be adjusted based on monitoring and operational limits.

### 11.3 Concurrent Writes

Where application writes continue during a backfill:

- The migration MUST define how new and updated records are handled.
- Dual-write or change-capture mechanisms MUST be designed to prevent lost updates.
- Backfill updates MUST not overwrite newer application changes.
- Version fields, timestamps, conditional updates, or other concurrency controls SHOULD be used where appropriate.
- The final cutover MUST include a reconciliation step.

### 11.4 Validation

Backfills MUST define verification criteria, which may include:

- Expected record counts.
- Required-field completeness.
- Referential integrity.
- Aggregate checks or checksums where appropriate.
- Distribution and range checks.
- Sampled record-level comparisons.
- Tenant-level coverage.
- Reconciliation of failed or skipped records.

Validation MUST be proportionate to the risk and impact of the transformation.

---

## 12. Multi-Tenant Migration Strategy

### 12.1 Tenant Isolation

- Tenant-owned records MUST remain associated with the correct tenant throughout migration.
- Tenant identifiers MUST NOT be inferred from untrusted input.
- Backfills MUST use verified tenant relationships.
- Cross-tenant updates MUST NOT occur unintentionally.
- Migration scripts MUST explicitly distinguish tenant-scoped changes from platform-level changes.

### 12.2 Shared Database SaaS

For shared-schema deployments:

- Shared migrations MUST be compatible with all active tenants.
- Tenant-specific data transformations MUST be scoped and validated.
- Large tenant backfills SHOULD support resumable progress.
- Migration execution MUST account for uneven tenant sizes and workload distributions.
- Tenant-specific failures MUST be visible without silently corrupting other tenants' data.

### 12.3 Dedicated Tenant Databases

For dedicated database deployments:

- Schema versions MUST be tracked per deployment or database.
- Migration execution MUST support controlled rollout across tenant instances.
- Partial rollout MUST be observable.
- Failed instances MUST be safely retried or recovered.
- Version skew MUST remain within explicitly supported compatibility bounds.

### 12.4 Self-Hosted Universities

- Migrations MUST support documented self-hosted deployment procedures.
- Each installation MUST be able to identify its current schema version.
- Required migration prerequisites MUST be documented.
- Migration execution MUST be compatible with supported installation and upgrade paths.
- Self-hosted operators MUST receive actionable failure information without exposure of secrets.
- Installation-specific configuration MUST NOT be overwritten by generic migration logic.
- Unsupported schema versions MUST fail safely with a clear diagnostic.

---

## 13. Zero-Downtime and Rolling Deployments

### 13.1 Expand-and-Contract

Where uninterrupted availability is required, breaking changes SHOULD use the expand-and-contract pattern.

**Expand**
- Add new columns, tables, fields, or compatible structures.
- Preserve the existing schema and application behavior.
- Deploy code that supports the transitional state.

**Migrate**
- Backfill existing records.
- Dual-write or synchronize old and new representations where required.
- Validate that the new representation is complete and correct.

**Cut Over**
- Transition reads and writes to the new representation.
- Monitor correctness, latency, and errors.
- Confirm compatibility with all active application instances.

**Contract**
- Remove old application behavior.
- Remove obsolete fields, tables, or indexes in a separate migration.
- Perform final validation and cleanup.

### 13.2 Application Compatibility

- Old and new application versions MUST be considered during rolling deployment.
- Database changes MUST NOT require every application instance to update simultaneously unless the deployment explicitly supports coordinated downtime.
- Feature flags MAY be used to control migration-dependent application behavior.
- Rollback MUST account for whether new application code has written data that the old schema cannot represent.
- Compatibility windows MUST be defined for critical schema transitions.

### 13.3 Downtime Migrations

When a migration requires downtime:

- The reason MUST be documented.
- Affected services and users MUST be identified.
- Expected duration and operational risks MUST be assessed.
- Backups and recovery procedures MUST be verified.
- Maintenance windows and communication MUST be planned where applicable.
- Post-migration validation MUST be completed before restoring normal traffic.

---

## 14. Migration Execution and Orchestration

### 14.1 Controlled Execution

- Production migrations MUST be executed through an approved deployment or operational workflow.
- Migration execution MUST be restricted to authorized identities.
- Migration jobs MUST have explicit timeouts or operational limits where supported.
- Concurrent execution of conflicting migrations MUST be prevented.
- Migration status MUST be observable.
- Application startup MUST NOT trigger uncontrolled production migrations unless explicitly approved by the deployment architecture.

### 14.2 Ordering

- Migrations MUST execute in a deterministic order.
- Dependencies between migrations MUST be explicit.
- A migration MUST NOT run before its prerequisites are satisfied.
- Cross-datastore migrations MUST define their execution order and consistency expectations.
- Application releases MUST specify required schema versions or compatible version ranges where necessary.

### 14.3 Migration Locks

- Migration systems SHOULD use an appropriate locking mechanism to prevent concurrent application of the same migration set.
- Locking MUST be scoped to the migration domain or database.
- Lock acquisition and release MUST be safe under process crashes and timeouts.
- Stale locks MUST be recoverable through an explicit operational procedure.
- Distributed migration locks MUST not be implemented using unsafe assumptions about lease validity or clock synchronization.

### 14.4 Failure Handling

On migration failure:

1. Stop dependent migration steps where continuing would risk data integrity.
2. Capture the failure details and current migration state.
3. Determine whether the migration was atomic or partially applied.
4. Identify affected schema objects and records.
5. Follow the documented retry, repair, rollback, or restore strategy.
6. Validate the resulting state before resuming deployment.
7. Record the resolution and any required follow-up migration.

Failed migrations MUST NOT be marked as successful merely to bypass the migration system.

---

## 15. Backup and Recovery

### 15.1 Before High-Risk Migrations

Before high-risk or destructive migrations:

- Confirm that an appropriate backup or recovery mechanism exists.
- Verify backup freshness and coverage.
- Confirm that restore procedures are documented and tested at an appropriate frequency.
- Assess recovery time and data-loss objectives.
- Identify whether point-in-time recovery is available and appropriate.
- Ensure the backup itself is protected by access controls and encryption requirements.

A backup MUST NOT be assumed recoverable merely because a backup job reported success.

### 15.2 Recovery Strategy

Recovery plans SHOULD distinguish between:

- Application rollback.
- Schema rollback.
- Forward-fix migration.
- Data restoration.
- Point-in-time recovery.
- Rebuilding derived data.
- Reconciliation from authoritative sources.

For irreversible or lossy transformations, forward recovery or restoration may be safer than an automated down migration.

### 15.3 Derived Data

- Redis caches SHOULD be recoverable through regeneration or repopulation where applicable.
- OpenSearch indexes SHOULD be rebuildable from authoritative data or reliable change streams where designed.
- Derived-data rebuilds MUST include capacity and rate controls.
- Data loss in a derived store MUST NOT be treated as proof of authoritative-data loss.

---

## 16. Security and Privacy

- Migration credentials MUST follow least-privilege principles.
- Production migration credentials MUST NOT be embedded in source code, scripts, logs, or documentation.
- Secrets MUST be provided through approved secret-management mechanisms.
- Migration scripts MUST NOT log personal or sensitive record contents unnecessarily.
- Access to migration execution and migration history MUST be restricted appropriately.
- Sensitive data transformations MUST comply with applicable retention and privacy requirements.
- Temporary migration files MUST be protected and securely removed when no longer required.
- Backfills and reindexing MUST preserve tenant boundaries and data-access controls.
- Migration tooling MUST be reviewed for unsafe dynamic query construction and injection risks.

---

## 17. Observability and Auditability

Every production migration MUST produce sufficient operational evidence to understand its execution.

### 17.1 Required Signals

Where applicable, capture:
- Migration identifier and target datastore.
- Execution start and end times.
- Duration and status.
- Batch progress and records processed.
- Failure counts and error categories.
- Retry counts.
- Lock waits and timeout information.
- Resource consumption and throughput.
- Validation outcomes.
- Rollback, repair, or recovery actions.

### 17.2 Logging

- Logs MUST be structured where supported.
- Logs MUST identify the migration and deployment context.
- Logs MUST NOT expose secrets or unnecessary personal data.
- Error messages MUST be actionable.
- Migration output MUST distinguish warnings, recoverable errors, and terminal failures.

### 17.3 Audit

- Production migration execution SHOULD be auditable.
- High-risk changes MUST record authorization and approval where required.
- Manual intervention MUST be documented.
- Migration completion MUST be associated with the resulting schema or data version.
- Audit records MUST be protected against unauthorized modification.

---

## 18. Testing Requirements

### 18.1 Local and Development Testing

- Migrations MUST be tested against a clean database.
- Migrations MUST be tested against a representative existing schema where applicable.
- Existing-data transformations MUST be tested with representative data.
- Migration ordering and dependencies MUST be verified.
- Migration scripts MUST be checked for repeatability and failure behavior.

### 18.2 Integration Testing

- Validate migrations using the supported database engine and relevant version.
- Verify schema state after execution.
- Verify application behavior against the migrated schema.
- Validate constraints, indexes, and relationships.
- Test transitional application compatibility when required.
- Validate multi-tenant scoping for tenant-sensitive transformations.

### 18.3 Performance Testing

For large or high-risk migrations:

- Use representative data volumes.
- Measure execution time and resource utilization.
- Evaluate locking and write contention.
- Assess replication lag and service impact.
- Test batch sizing and resumability.
- Verify that operational limits and timeouts are appropriate.

### 18.4 Failure and Recovery Testing

- Test expected failure conditions where practical.
- Verify behavior after interruption.
- Verify safe retry behavior.
- Validate recovery from partial completion.
- Exercise rollback or forward-fix procedures for high-risk migrations.
- Confirm that deployment orchestration does not incorrectly proceed after failure.

### 18.5 CI Requirements

CI SHOULD:
- Validate migration syntax.
- Detect duplicate or invalid migration identifiers.
- Verify migration ordering.
- Apply migrations to a clean test database.
- Run relevant migration and integration tests.
- Detect unexpected modifications to applied migrations where tooling supports it.
- Validate that application code works against the resulting schema.

---

## 19. Migration Review Checklist

Before approving a migration, reviewers MUST verify:

- [ ] The migration has a clear purpose and owner.
- [ ] The target datastore and authoritative data source are identified.
- [ ] The current schema and data shape have been inspected.
- [ ] The migration is version-controlled and uniquely identified.
- [ ] The migration is appropriately scoped.
- [ ] Data integrity and business invariants are preserved.
- [ ] Tenant isolation is maintained.
- [ ] Application compatibility has been considered.
- [ ] Data backfills are bounded, observable, and resumable where necessary.
- [ ] Locking, transaction boundaries, and write contention have been evaluated.
- [ ] Index and constraint changes have been reviewed.
- [ ] Destructive operations have an impact assessment.
- [ ] Failure handling and recovery strategies are documented.
- [ ] Backup and restore requirements have been assessed.
- [ ] Migration execution and concurrency controls are defined.
- [ ] Tests cover correctness, compatibility, and relevant failure modes.
- [ ] Observability and audit requirements are addressed.
- [ ] Deployment order and application release dependencies are explicit.
- [ ] Self-hosted upgrade requirements are considered where applicable.
- [ ] Documentation reflects the new schema and operational procedure.

---

## 20. Anti-Patterns

The following practices are prohibited unless an explicit, documented exception is approved:

- Editing an already-applied migration to change its behavior.
- Making undocumented production schema changes.
- Dropping or overwriting data without an impact assessment and recovery strategy.
- Deploying application code that requires a schema change before that change is available.
- Performing unbounded full-dataset transformations in application memory.
- Running large backfills without progress tracking or retry support.
- Applying migrations concurrently without appropriate coordination.
- Assuming a failed migration left the database unchanged.
- Marking failed migrations as successful to bypass deployment checks.
- Running destructive migrations during a rolling deployment without compatibility analysis.
- Performing cross-tenant data transformations without explicit tenant scoping.
- Treating Redis cache contents or OpenSearch projections as authoritative data without an approved architecture.
- Creating incompatible OpenSearch mappings in place without a safe migration plan.
- Using application startup as an uncontrolled production migration mechanism.
- Removing old schema elements before all dependent application versions have been retired.
- Relying on untested backups or undocumented manual recovery procedures.
- Combining unrelated high-risk schema changes into one large migration.
- Using migrations to conceal unresolved data-model or domain-ownership problems.

---

## 21. Definition of Done

A migration is complete only when:

- Its purpose, scope, and ownership are documented.
- The migration is version-controlled and has a unique identifier.
- The change respects datastore and domain ownership.
- Existing data is preserved or transformed according to explicit requirements.
- Tenant isolation, authorization boundaries, and data integrity remain intact.
- Application compatibility and deployment ordering have been validated.
- Migration execution is deterministic and appropriately controlled.
- Backfills and other large transformations are bounded and recoverable.
- Required tests have passed in the supported environment.
- Relevant performance and operational impacts have been assessed.
- Failure, retry, rollback, or forward-recovery behavior is documented.
- Observability and auditability requirements are met.
- Schema and operational documentation have been updated.
- The migration has been reviewed and approved according to its risk.

Exceptions MUST be explicitly documented with their rationale, risks, mitigations, and approval.

---

## 22. Final Principle

**A migration is not complete when the schema changes; it is complete when the data is correct, the application is compatible, and the system can operate safely and recoverably.**

KAMPYN MUST evolve its data structures through controlled, observable, and verifiable changes. Every migration must protect data integrity, tenant isolation, operational reliability, and the ability to upgrade or recover both SaaS and self-hosted deployments.
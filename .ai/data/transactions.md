# Database Transactions & Consistency

## 1. Purpose

This document defines how KAMPYN handles database transactions, data consistency, concurrency, atomicity, isolation, and failure recovery across its data infrastructure.

The objective is to ensure that every operation preserves data integrity, even when requests fail, services restart, concurrent updates occur, or multiple services participate in a workflow.

KAMPYN operates across multiple data systems, including PostgreSQL, MongoDB, Redis, OpenSearch, and object storage. Each system has different transactional guarantees and must be used according to its intended role.

All transaction-related implementations MUST follow this document alongside:

- `.ai/data/data-modeling.md`
- `.ai/data/indexing.md`
- `.ai/data/migrations.md`
- `.ai/data/mongodb.md`
- `.ai/data/redis.md`
- `.ai/architecture/api.md`
- `.ai/architecture/events.md`
- `.ai/architecture/multi-tenancy.md`
- `.ai/architecture/backend.md`
- `.ai/constitution/05-security-bar.md`

---

## 2. Core Principles

All transaction implementations MUST follow these principles.

### 2.1 Atomicity

- A transaction MUST either complete all of its intended database changes or leave the authoritative data unchanged.
- Partial writes MUST NOT be treated as successful operations.
- Related state changes MUST be committed atomically whenever they reside within the same transactional boundary.
- Operations that cannot be performed atomically across systems MUST use an explicitly designed distributed workflow.

### 2.2 Consistency

- Every committed transaction MUST preserve defined business invariants.
- Database constraints MUST enforce critical integrity requirements wherever possible.
- Application-level validation MUST complement, not replace, database-level constraints.
- A transaction MUST NOT commit a state that violates the domain's documented rules.

### 2.3 Isolation

- Concurrent transactions MUST NOT cause lost updates, invalid state transitions, duplicate business operations, or inconsistent balances.
- Isolation levels MUST be selected according to the correctness requirements of the operation.
- Higher isolation MUST NOT be applied indiscriminately without understanding its performance and contention implications.

### 2.4 Durability

- A successful transaction MUST be committed to the authoritative datastore with the durability guarantees configured for that datastore.
- An API MUST NOT report success before the authoritative transaction commits.
- Asynchronous downstream processing MUST NOT be confused with the durability of the original transaction.

### 2.5 Explicit Boundaries

- Every transaction MUST have a clearly defined start, commit, and rollback boundary.
- Transactions MUST remain as short as reasonably possible.
- Network calls, file transfers, user interaction, and long-running computation MUST NOT occur inside database transactions unless there is a documented architectural justification.
- Transaction boundaries MUST align with business operations rather than arbitrary code structure.

---

## 3. Source-of-Truth and Transaction Ownership

PostgreSQL is KAMPYN's default authoritative datastore for relational and transactional business data.

| System | Transactional responsibility |
|---|---|
| PostgreSQL | Primary transactional state, relational integrity, orders, payments ledger, bookings, inventory records, and other authoritative business entities |
| MongoDB | Atomic document operations and justified document-oriented workflows |
| Redis | Atomic transient operations, caching, rate limiting, locks, and coordination |
| OpenSearch | Derived search and discovery data; not authoritative transaction state |
| Object Storage | Durable file and media storage, coordinated through application workflows |

### 3.1 Authoritative Writes

- Every business entity MUST have a clearly designated authoritative datastore.
- A single entity MUST NOT have multiple independent authoritative copies.
- All correctness-critical writes MUST target the authoritative datastore.
- Derived systems MUST be updated through reliable synchronization mechanisms.
- A successful write to Redis or OpenSearch MUST NOT be treated as proof that the authoritative business transaction succeeded.

### 3.2 Ownership

- A service MUST own the business transactions for the domain it controls.
- Other services MUST NOT directly modify another service's owned tables or collections.
- Cross-service workflows MUST use explicit service APIs, events, or workflow orchestration.
- Database credentials and access permissions SHOULD enforce ownership boundaries where practical.

---

## 4. Transaction Design

### 4.1 Business-Level Transactions

Transactions MUST represent a complete business operation or a well-defined atomic part of one.

Examples include:

- Creating an order and reserving its inventory.
- Confirming a booking and allocating a resource.
- Recording a payment result and updating the corresponding order state.
- Applying an inventory adjustment with an associated audit record.
- Creating a complaint and its initial status history.

Each operation MUST define:

1. The business invariant being protected.
2. The authoritative datastore.
3. The records that must change atomically.
4. The expected isolation and concurrency behavior.
5. The rollback and failure behavior.
6. The idempotency strategy, where applicable.
7. Any downstream events or side effects.

### 4.2 Keep Transactions Short

Transactions SHOULD contain only the database reads and writes required to preserve atomicity.

The following MUST NOT be performed inside an open transaction unless explicitly justified:

- External HTTP requests.
- Payment gateway calls.
- Sending emails, SMS, or push notifications.
- Uploading or downloading files.
- Long-running data transformations.
- Waiting for user input.
- Unbounded loops or bulk operations with uncontrolled duration.
- Calls to unrelated services.

Prepare required data before opening the transaction where possible. Perform external side effects after commit using an appropriate reliable delivery pattern.

### 4.3 Nested Transactions

- Nested transaction behavior MUST be explicit.
- Services MUST NOT assume that nested transaction calls create independent commits.
- Where supported, savepoints MAY be used for recoverable sub-operations.
- Transaction ownership MUST remain clear when repositories and services share a transaction context.
- A child operation MUST NOT independently commit a transaction owned by its caller.

---

## 5. PostgreSQL Transactions

PostgreSQL SHOULD be used for operations requiring relational integrity, multi-row atomicity, constraints, and transactional business workflows.

### 5.1 Transaction Usage

- Use explicit transactions when multiple writes or reads and writes must succeed together.
- Use single-statement atomic operations when they fully preserve the required invariant.
- Prefer database constraints for enforcing unique, foreign-key, check, and exclusion requirements.
- Use `RETURNING` where appropriate to avoid unnecessary follow-up queries.
- Avoid transactions for independent writes that do not require atomicity.

Example:

```sql
BEGIN;

INSERT INTO orders (
    id,
    tenant_id,
    user_id,
    status,
    total_amount
)
VALUES (
    $1,
    $2,
    $3,
    'PENDING',
    $4
);

INSERT INTO order_items (
    order_id,
    item_id,
    quantity,
    unit_price
)
VALUES
    ($1, $5, $6, $7);

COMMIT;
```

If any statement fails, the transaction MUST be rolled back. Production code MUST use parameterized queries or a safe query builder/ORM rather than interpolating untrusted input.

### 5.2 Isolation Levels

PostgreSQL isolation levels MUST be selected based on the business invariant.

| Isolation level | Appropriate use |
|---|---|
| Read Committed | Default for many standard CRUD and transactional operations |
| Repeatable Read | Workflows requiring a consistent transaction snapshot |
| Serializable | Operations where serialization anomalies could violate critical business invariants |

- `Read Committed` SHOULD be the default unless stronger isolation is required.
- `Repeatable Read` and `Serializable` MUST be used with an understanding of their retry and contention implications.
- Serializable transactions MUST handle serialization failures through bounded retries.
- Isolation level selection MUST be documented for high-risk workflows.

### 5.3 Row-Level Locking

Use row-level locking when concurrent modifications to the same records could violate business rules.

Supported locking strategies include:

- `SELECT ... FOR UPDATE`
- `SELECT ... FOR NO KEY UPDATE`
- `SELECT ... FOR SHARE`
- `SELECT ... FOR KEY SHARE`

Rules:

- Lock only the rows required by the transaction.
- Keep lock duration short.
- Acquire multiple locks in a consistent order to reduce deadlocks.
- Avoid locking large result sets when a conditional update can enforce the same invariant.
- Do not rely on a prior unlocked read to guarantee that a subsequent write remains valid.

### 5.4 Conditional Updates

Prefer atomic conditional updates when they can enforce a state transition or prevent an invalid update.

Example:

```sql
UPDATE inventory
SET available_quantity = available_quantity - $1
WHERE tenant_id = $2
  AND item_id = $3
  AND available_quantity >= $1
RETURNING available_quantity;
```

The application MUST verify whether the expected row was updated. A zero-row result MUST be interpreted according to the business rule, such as insufficient inventory or a stale request.

### 5.5 Deadlocks

- Transactions MUST acquire locks in a consistent order wherever possible.
- Transactions MUST avoid unnecessary lock escalation or long lock retention.
- Deadlocks MUST be handled as recoverable transaction failures when the operation is safe to retry.
- Retries MUST restart the complete transaction, not only the failed statement.
- Retry attempts MUST be bounded and use backoff with jitter where appropriate.
- Repeated deadlocks MUST generate actionable logs and metrics.

### 5.6 Transaction Timeouts

Production transactions MUST have suitable timeouts.

Configure and review:

- Transaction execution time.
- Lock acquisition time.
- Idle-in-transaction time.
- Query execution time.

Timeouts SHOULD reflect expected workload duration and must not be set so high that blocked transactions exhaust connection pools or degrade the service.

---

## 6. MongoDB Transactions

MongoDB MAY be used for justified document-oriented workflows where its document model is appropriate.

### 6.1 Single-Document Atomicity

- Prefer a single-document atomic update when the entire business invariant fits within one document.
- Use conditional filters to prevent invalid state transitions.
- Avoid multi-document transactions when a well-designed single-document operation can satisfy the requirement.
- Do not split a business entity across multiple documents without evaluating consistency and transaction requirements.

### 6.2 Multi-Document Transactions

Multi-document transactions MAY be used when multiple MongoDB documents must be updated atomically.

Rules:

- Use session-based transactions with explicit commit and abort behavior.
- Use supported transaction read and write concerns appropriate to the durability requirement.
- Keep transaction duration and document count controlled.
- Avoid external network calls inside a transaction.
- Handle transient transaction errors and uncertain commit outcomes according to the MongoDB driver’s transaction retry guidance.
- Ensure all participating operations use the same session.
- Do not assume MongoDB transactions provide atomicity across separate MongoDB deployments or independent clusters.

### 6.3 Transaction Suitability

Before introducing a MongoDB transaction, document:

- Why the data belongs in MongoDB.
- Why single-document atomicity is insufficient.
- The required consistency guarantees.
- Expected transaction size and frequency.
- Retry and failure-handling behavior.
- Operational impact on the deployment.

If the workflow is relational and requires frequent multi-entity transactions, evaluate whether PostgreSQL is a more suitable authoritative datastore.

---

## 7. Cross-Datastore Consistency

KAMPYN MUST NOT assume that a single ACID transaction can span PostgreSQL, MongoDB, Redis, OpenSearch, object storage, and external services.

### 7.1 No Implicit Distributed Transactions

- Do not simulate a distributed transaction by sequentially writing to multiple datastores and assuming all writes will succeed.
- Do not treat compensating operations as equivalent to a true atomic rollback.
- Every multi-system workflow MUST define its consistency model and recovery strategy.
- Distributed two-phase commit MUST NOT be introduced without explicit architectural approval and a documented operational justification.

### 7.2 Transactional Outbox

Use the transactional outbox pattern when a committed database change must reliably produce an event.

The business state and outbox record MUST be committed in the same transaction.

Example:

```sql
BEGIN;

UPDATE orders
SET status = 'CONFIRMED'
WHERE id = $1
  AND tenant_id = $2;

INSERT INTO outbox_events (
    id,
    tenant_id,
    aggregate_type,
    aggregate_id,
    event_type,
    payload,
    created_at
)
VALUES (
    $3,
    $2,
    'ORDER',
    $1,
    'ORDER_CONFIRMED',
    $4,
    NOW()
);

COMMIT;
```

Rules:

- The outbox record MUST be written in the same transaction as the business state change.
- A separate publisher MUST deliver committed outbox events.
- Delivery MUST be treated as at-least-once unless stronger guarantees are explicitly implemented.
- Consumers MUST be idempotent.
- Outbox records MUST have stable event IDs.
- Failed events MUST be retried with bounded backoff and surfaced through monitoring.
- Retention and cleanup MUST NOT delete events that have not been safely delivered or archived.

### 7.3 Idempotent Consumers

- Every consumer handling retryable events MUST define an idempotency strategy.
- Duplicate event delivery MUST NOT result in duplicate business effects.
- Consumers SHOULD record processed event IDs in a durable store when required.
- Deduplication records and their business effects SHOULD be committed atomically in the consumer’s authoritative datastore.
- Event ordering assumptions MUST be explicit and enforced where the domain requires ordered processing.

### 7.4 Sagas

Use a saga for long-running workflows spanning multiple independently committed services or datastores.

A saga consists of a sequence of local transactions and, where needed, compensating actions.

Example: an order workflow may involve:

1. Create a pending order.
2. Reserve inventory.
3. Initiate payment.
4. Confirm the order.
5. Publish order-confirmed events.

If a later step fails, the workflow MUST transition to a documented failure or compensation path, such as releasing inventory or marking the order for payment reconciliation.

Rules:

- Each step MUST have a durable state transition.
- Each step MUST be idempotent or protected against duplicate execution.
- Compensation MUST be defined for operations that can be meaningfully reversed.
- Compensation MUST NOT be assumed to erase irreversible external effects.
- Saga progress MUST be observable and recoverable after process restarts.
- Timeouts, retries, terminal failures, and manual intervention paths MUST be defined.
- The workflow MUST distinguish between failed, cancelled, compensated, and pending-reconciliation states.

### 7.5 Eventual Consistency

Eventual consistency MAY be used for derived systems such as search indexes, analytics, caches, and notifications.

- The authoritative state MUST be committed before downstream projections are treated as updated.
- APIs MUST NOT imply that asynchronous projections are immediately consistent unless they actually are.
- Acceptable propagation delays SHOULD be defined for critical projections.
- Reconciliation mechanisms MUST exist for important derived data.
- Read paths MUST account for projection lag where users could otherwise observe confusing or invalid states.

---

## 8. Transactions and KAMPYN Business Workflows

Every business workflow MUST explicitly identify its transactional boundary and consistency strategy.

### 8.1 Food Ordering

An order may involve order records, order items, inventory reservations, payment state, and notifications.

Rules:

- Order creation and its required order-item records MUST be atomic within the authoritative datastore.
- Inventory reservation MUST use an atomic conditional update or another concurrency-safe mechanism.
- Inventory MUST NOT be decremented based solely on an earlier, unlocked availability check.
- Payment gateway operations MUST NOT run inside the database transaction.
- Payment state changes MUST be validated against the current order state.
- Order confirmation and its outbox event MUST be committed atomically.
- Duplicate order submissions MUST be handled through idempotency keys or an equivalent business-safe mechanism.
- Failed or abandoned payment workflows MUST have a defined inventory release or reconciliation path.

### 8.2 Payments

Payment workflows MUST distinguish between the payment provider's external state and KAMPYN's internal transaction state.

Rules:

- Payment initiation MUST be idempotent wherever the provider supports idempotency.
- External payment requests MUST occur outside database transactions.
- Payment callbacks and webhooks MUST be authenticated and validated.
- Duplicate callbacks MUST NOT create duplicate payment records or duplicate fulfillment.
- Payment state transitions MUST be validated and persisted atomically.
- Amount, currency, order reference, tenant, and payment identity MUST be verified before applying a payment result.
- Refunds, reversals, chargebacks, and uncertain provider outcomes MUST have explicit states.
- Payment and order reconciliation MUST be supported for uncertain or delayed outcomes.
- A client request MUST NOT be treated as proof of successful payment.

### 8.3 Inventory

Inventory operations MUST preserve quantity and reservation invariants.

Rules:

- Inventory adjustments MUST be atomic.
- Reservation, release, consumption, and correction MUST be separate, explicit domain operations.
- Available quantity MUST NOT become negative unless the domain explicitly supports backorders.
- Concurrent reservations MUST be protected using conditional updates, locking, or another justified concurrency-control strategy.
- Every material inventory adjustment SHOULD have an auditable record.
- Duplicate inventory events MUST NOT apply the same adjustment twice.
- Reconciliation MUST identify inconsistencies between inventory balances and reservation records.

### 8.4 Hostel and Guest House Bookings

Booking workflows MUST prevent conflicting confirmed allocations for the same constrained resource and time period.

Rules:

- Availability checks MUST NOT be treated as reservations.
- Conflicting booking requests MUST be protected by database constraints, locking, serializable transactions, or another documented concurrency-safe mechanism.
- Booking confirmation and resource allocation MUST be atomic within the authoritative transactional boundary.
- Temporary holds MUST have explicit expiry and release behavior.
- Payment confirmation MUST be coordinated through a durable workflow.
- Cancellation, expiry, and refund operations MUST use validated state transitions.
- Concurrent booking attempts MUST result in a clearly defined winner or a retryable conflict, never silent double allocation.

### 8.5 Washing Machine Scheduling

- Scheduling MUST prevent overlapping confirmed allocations for a constrained machine.
- Availability checks MUST be followed by a concurrency-safe reservation operation.
- Slot reservation and booking-state updates MUST be atomic where they share the same datastore.
- Expired holds MUST be released through a reliable process.
- Duplicate booking submissions MUST NOT create multiple active reservations for the same request.

### 8.6 Shuttle Booking

- Seat allocation MUST be protected against concurrent overbooking.
- Seat reservation and booking state MUST be updated atomically within the authoritative datastore.
- Cancellation and seat release MUST be idempotent.
- If the booking workflow spans multiple services, use a saga or another explicitly documented recovery strategy.

### 8.7 Complaints and Community

- Complaint creation and its initial status history SHOULD be atomic.
- Moderation state transitions MUST be validated and auditable.
- Message persistence and any durable delivery event SHOULD be coordinated through an outbox when reliable downstream processing is required.
- Real-time notifications MUST NOT be treated as the source of truth for message persistence.
- Duplicate message submissions MUST be handled where retries could cause repeated content.

### 8.8 HR and Administrative Workflows

- Changes to sensitive administrative records MUST use explicit authorization and validated state transitions.
- Related records that form one business operation SHOULD be committed atomically.
- High-impact actions SHOULD create an audit record within the same transaction when supported by the datastore.
- Cross-service administrative workflows MUST define retry, compensation, and reconciliation behavior.
- Administrative actions MUST NOT bypass tenant isolation or domain ownership boundaries.

---

## 9. Idempotency

Idempotency MUST be designed for operations that may be retried due to client timeouts, network failures, duplicate submissions, queue redelivery, or uncertain outcomes.

### 9.1 Idempotency Keys

- Use idempotency keys for critical externally initiated operations, including order creation, payment initiation, bookings, and other operations where duplicate execution would cause harm.
- Keys MUST be scoped to the tenant and the appropriate user, operation, or business context.
- Keys MUST be associated with a request fingerprint or equivalent validation to prevent reuse with different payloads.
- The idempotency record and the resulting business operation SHOULD be committed atomically.
- Key retention MUST cover the documented retry and replay window.
- Repeated requests with the same key and same effective payload MUST return a consistent result.
- Reuse of a key with a different effective payload MUST be rejected.

### 9.2 Idempotent State Transitions

- State transitions MUST validate the current state before applying changes.
- Repeated transitions MUST either return the already-established outcome or fail with a documented domain error.
- Consumers MUST NOT rely solely on in-memory deduplication.
- Idempotency behavior MUST be tested under duplicate and concurrent requests.

---

## 10. Concurrency Control

Concurrency control MUST be selected according to the invariant being protected.

### 10.1 Optimistic Concurrency

Use optimistic concurrency when conflicts are expected to be infrequent and can be safely detected.

A version column or equivalent revision field MAY be used.

Example:

```sql
UPDATE bookings
SET
    status = $1,
    version = version + 1,
    updated_at = NOW()
WHERE id = $2
  AND tenant_id = $3
  AND version = $4
RETURNING version;
```

Rules:

- A zero-row result MUST be handled as a conflict, stale update, or missing record according to the operation.
- The application MUST NOT silently overwrite a newer state.
- Retries MUST re-read and revalidate the current business state.
- Versioning MUST be used consistently within the affected aggregate or record boundary.

### 10.2 Pessimistic Concurrency

Use pessimistic locking when concurrent modification is likely or conflicts would be expensive or unsafe.

- Lock only required records.
- Use consistent lock ordering.
- Define lock acquisition timeouts.
- Avoid long-lived locks.
- Provide conflict and timeout handling at the application boundary.

### 10.3 Unique Constraints

Database-level uniqueness MUST enforce invariants that require uniqueness.

Examples include:

- Unique tenant-scoped booking references.
- Unique idempotency keys within their intended scope.
- Unique payment provider references where applicable.
- Unique active resource-slot allocations when representable by an appropriate constraint.

Application-level pre-checks MUST NOT be the sole protection against duplicate creation under concurrency.

### 10.4 Lost Updates

- Read-modify-write sequences MUST be protected against concurrent overwrites.
- Use conditional updates, row locks, version checks, or another documented strategy.
- Critical numeric updates SHOULD use atomic database expressions rather than application-side arithmetic based on stale reads.
- Concurrent state transitions MUST be tested.

---

## 11. Retry and Failure Handling

Retries MUST be deliberate, bounded, observable, and safe.

### 11.1 Retryable Failures

Potentially retryable failures include:

- Serialization failures.
- Deadlocks.
- Temporary connection failures.
- Transient service unavailability.
- Explicitly retryable concurrency conflicts.
- Temporary queue or event delivery failures.

Retryability MUST be determined by error type and operation semantics, not by a blanket catch-all.

### 11.2 Retry Rules

- Retry only operations that are safe to repeat or protected by idempotency.
- Retry the complete transaction after a transaction-level failure.
- Use bounded attempts and exponential backoff with jitter where appropriate.
- Enforce an overall deadline.
- Do not retry permanent validation, authorization, or business-rule errors.
- Avoid retry storms by applying concurrency limits and circuit-breaking strategies where justified.
- Record retry counts and terminal failure reasons.

### 11.3 Unknown Commit Outcomes

A connection failure during commit can leave the application uncertain whether the transaction committed.

In such cases:

- Do not assume rollback.
- Do not blindly execute the operation again.
- Resolve the outcome using a stable operation identifier, idempotency key, or authoritative state query.
- Use reconciliation for workflows whose outcomes cannot be determined synchronously.
- Return a response that accurately reflects the uncertainty, where the API contract allows it.

### 11.4 Rollback

- Failed transactions MUST be rolled back or safely abandoned through the database driver's transaction mechanism.
- Connections with failed or incomplete transaction state MUST NOT be returned to the pool in an unusable state.
- Rollback failures MUST be logged and surfaced through appropriate operational monitoring.
- Cleanup logic MUST preserve the original failure while recording any rollback or cleanup failure.

---

## 12. Transactional Boundaries in Application Code

### 12.1 Service Layer

The service or use-case layer SHOULD define the business transaction boundary.

- Controllers MUST NOT own complex transaction orchestration.
- Repositories MUST support transaction-aware operations where required.
- Domain logic MUST remain independent of low-level connection management.
- Transaction context MUST be passed explicitly or through a well-defined transaction manager.
- Transaction lifecycle MUST NOT be hidden in unrelated helper methods.

### 12.2 Repository Layer

- Repositories MUST support participation in an existing transaction when the use case requires atomic operations across repositories.
- Repositories MUST NOT independently commit a transaction owned by the service layer.
- Repository methods MUST clearly document whether they require a transaction, participate in one, or operate independently.
- Database-specific transaction details SHOULD remain within infrastructure and persistence layers.

### 12.3 Transaction Manager

Where a transaction manager is used, it MUST:

- Start transactions using the appropriate datastore driver.
- Provide a clear commit and rollback lifecycle.
- Prevent accidental use after completion.
- Propagate transaction failures.
- Support cancellation and deadlines.
- Ensure all participating operations use the same transaction context.
- Release resources safely.
- Expose observability without leaking sensitive data.

### 12.4 Example Service Pattern

```typescript
async function confirmOrder(input: ConfirmOrderInput) {
  return database.transaction(async (tx) => {
    const order = await orderRepository.findForUpdate(
      tx,
      input.tenantId,
      input.orderId
    );

    if (!order || order.status !== "PENDING") {
      throw new InvalidOrderStateError();
    }

    await inventoryRepository.reserve(tx, {
      tenantId: input.tenantId,
      items: order.items
    });

    const confirmed = await orderRepository.confirm(
      tx,
      input.tenantId,
      input.orderId
    );

    await outboxRepository.create(tx, {
      tenantId: input.tenantId,
      eventType: "ORDER_CONFIRMED",
      aggregateId: input.orderId
    });

    return confirmed;
  });
}
```

This example illustrates the transaction boundary. Production implementations MUST validate all relevant business invariants, use typed errors, and ensure repository operations use the same transaction.

---

## 13. Transactions and Multi-Tenancy

Tenant isolation MUST be enforced throughout transaction execution.

- Every tenant-owned query and mutation MUST be scoped to the authenticated tenant.
- Tenant identity MUST be derived from trusted authentication and authorization context, not blindly accepted from request payloads.
- Tenant context MUST be applied consistently across all repository calls participating in a transaction.
- Cross-tenant operations MUST require explicit privileged authorization and an audited workflow.
- Transaction retries MUST preserve the original tenant context.
- Idempotency keys, lock keys, and event identifiers MUST use appropriate tenant scoping.
- Database constraints SHOULD include tenant identity where uniqueness is tenant-scoped.
- Background jobs and event consumers MUST establish tenant context before accessing tenant-owned data.
- A transaction MUST NOT combine data from different tenants unless a documented and authorized cross-tenant operation requires it.

Where PostgreSQL row-level security is used, transaction-scoped tenant context MUST be established and cleared safely to prevent connection-pool context leakage.

---

## 14. Transactions with Redis

Redis MAY provide atomic operations and transient coordination but MUST NOT replace authoritative transactions for durable business state.

### 14.1 Redis Atomicity

- Use atomic Redis commands for single-key state changes where possible.
- Use Lua scripts or Redis transactions when multiple Redis operations require atomic execution.
- Understand Redis Cluster key-slot restrictions before implementing multi-key atomic operations.
- Use tenant-scoped key design and appropriate key expiration.
- Treat Redis atomicity as limited to Redis-managed state, not as a cross-datastore transaction.

### 14.2 Distributed Locks

- Distributed locks MUST NOT be the only protection for critical business invariants when database constraints or transactional conditional updates can enforce them.
- Lock acquisition MUST use a unique token and an explicit expiry.
- Lock release MUST verify ownership.
- Lock renewal and expiry behavior MUST be documented.
- Long-running operations MUST account for lock expiry and possible concurrent ownership.
- Critical workflows SHOULD use fencing tokens or authoritative conditional writes when stale lock holders could corrupt state.

### 14.3 Cache Consistency

- Cache invalidation MUST occur only after the authoritative transaction commits.
- Cache failures MUST NOT cause an already-committed business transaction to be incorrectly reported as rolled back.
- Cache refreshes SHOULD be asynchronous or retried where appropriate.
- Cache entries MUST be treated as potentially stale.
- Critical reads MUST use authoritative data or a consistency strategy suitable for the business requirement.

---

## 15. Transactions with OpenSearch and Object Storage

### 15.1 OpenSearch

- OpenSearch MUST be treated as a derived projection.
- Search-index updates MUST NOT be considered part of the authoritative business transaction.
- Use transactional outbox events or another reliable synchronization mechanism to update indexes.
- Indexing consumers MUST be idempotent and resilient to duplicate or out-of-order events.
- Failed indexing MUST be retryable and observable.
- Reindexing and reconciliation procedures MUST be documented.
- Business correctness MUST NOT depend solely on search-index state.

### 15.2 Object Storage

Object storage operations generally do not participate in database transactions.

For workflows involving uploaded files:

1. Validate the request and authorize the tenant.
2. Upload the file to an appropriate temporary or staged location.
3. Validate upload completion and required metadata.
4. Persist the file reference and business state in the authoritative datastore.
5. Finalize or promote the object according to the storage workflow.
6. Clean up abandoned temporary objects through a reliable process.

Rules:

- Database records MUST NOT point to incomplete or unverified uploads.
- Failed database commits MUST have a cleanup or reconciliation path for uploaded objects.
- Object deletion MUST be coordinated with database state transitions.
- Destructive storage operations MUST be idempotent where possible.
- File metadata, ownership, and tenant scope MUST be validated.
- The workflow MUST account for partial failures between object storage and the database.

---

## 16. Transactional Events and Messaging

- Business events MUST represent committed state transitions.
- Events MUST NOT be published before the corresponding authoritative transaction commits.
- Use an outbox when event loss would cause business inconsistency.
- Event schemas MUST be versioned and documented.
- Events MUST include stable identifiers and sufficient metadata for safe processing.
- Tenant context MUST be propagated securely.
- Consumers MUST handle duplicates, retries, and poison messages.
- Ordering guarantees MUST be explicit.
- Event payloads MUST contain only the information required by consumers.
- Sensitive data MUST NOT be included unless necessary and authorized.
- Dead-letter and replay procedures MUST be defined for important event pipelines.

Events are notifications of committed facts, not a replacement for authoritative state validation.

---

## 17. Transaction Observability

Transaction behavior MUST be observable in production without exposing sensitive information.

### 17.1 Metrics

Track relevant metrics, including:

- Transaction count and throughput.
- Commit and rollback rates.
- Transaction duration and latency percentiles.
- Lock wait duration.
- Deadlock count.
- Serialization failure count.
- Retry count and retry exhaustion.
- Connection pool saturation.
- Long-running and idle transactions.
- Outbox backlog and event delivery latency.
- Saga age, failed steps, and compensation outcomes.
- Reconciliation backlog and unresolved discrepancies.

### 17.2 Structured Logs

Transaction-related logs SHOULD include:

- Correlation or trace ID.
- Tenant identifier in an appropriately protected form.
- Service and operation name.
- Transaction outcome.
- Retry attempt.
- Failure category.
- Relevant business entity reference.

Logs MUST NOT expose credentials, payment secrets, authentication tokens, sensitive personal data, or complete confidential payloads.

### 17.3 Tracing

- Distributed traces SHOULD connect the stages of cross-service workflows.
- Transaction spans SHOULD record relevant timing and outcome metadata.
- External service calls SHOULD be distinguishable from local database work.
- Trace propagation MUST NOT be used as a substitute for durable workflow state.

---

## 18. Security Requirements

- Transactions MUST execute under least-privilege database identities.
- All queries MUST be parameterized or safely constructed.
- Tenant scoping MUST be enforced for all tenant-owned operations.
- Authorization MUST be checked before executing privileged business mutations.
- Sensitive changes SHOULD create auditable records.
- Payment-related transaction metadata MUST be handled according to the payment security policy.
- Transaction errors MUST NOT reveal internal database details to clients.
- Retries and recovery processes MUST preserve authorization and tenant boundaries.
- Background transaction workers MUST use controlled service identities.
- Database credentials and encryption keys MUST NOT be hardcoded or logged.

---

## 19. Performance and Scalability

Transactions MUST preserve correctness without introducing avoidable contention or resource exhaustion.

- Keep transactions short and bounded.
- Avoid unnecessary reads and writes inside transactions.
- Use suitable indexes for transactional predicates and locking queries.
- Avoid broad locks and unbounded scans.
- Use bulk operations only when they preserve the required atomicity and remain operationally safe.
- Apply pagination or chunking to large data-processing workflows that do not require one atomic transaction.
- Use connection pools sized for expected concurrent transactional demand.
- Avoid holding database connections while waiting on external services.
- Monitor contention and tune isolation levels based on observed workload.
- Separate long-running workflows from latency-sensitive request transactions.
- Do not trade away critical business invariants solely to improve throughput.

Large workflows SHOULD be split into durable stages when a single transaction would create excessive lock duration, log volume, or resource pressure.

---

## 20. Testing Requirements

Transaction behavior MUST be tested at the database integration level. Mock-only tests are insufficient for validating database guarantees.

### 20.1 Atomicity Tests

Verify that:

- All required records are committed on success.
- No partial business state remains after a transaction failure.
- Outbox records are committed or rolled back with the business state.
- Failed operations leave the authoritative data in a valid state.

### 20.2 Concurrency Tests

Verify:

- Concurrent inventory reservations cannot violate quantity constraints.
- Concurrent bookings cannot double-allocate a constrained resource.
- Concurrent state transitions cannot silently overwrite newer states.
- Duplicate idempotency keys cannot create duplicate business effects.
- Unique constraints protect against concurrent duplicate inserts.
- Deadlocks and serialization failures are handled safely.

### 20.3 Failure-Injection Tests

Test failures occurring:

- Before transaction start.
- During a query.
- During a multi-step write.
- At commit.
- During event publication.
- During a downstream service call.
- During compensation.
- During worker restart or message redelivery.
- During an uncertain external payment outcome.

### 20.4 Distributed Workflow Tests

For outbox, saga, and event-driven workflows, verify:

- Duplicate event delivery.
- Delayed event delivery.
- Out-of-order events where applicable.
- Consumer restarts.
- Retry exhaustion.
- Poison messages.
- Compensation and reconciliation.
- Recovery from partially completed workflows.

### 20.5 Isolation Tests

- Test the selected isolation level against the business invariants.
- Validate behavior under simultaneous transactions.
- Test stale-read and lost-update scenarios.
- Confirm that lock timeouts and transaction deadlines behave as expected.

### 20.6 Tenant Isolation Tests

- Verify transactions cannot read or mutate another tenant's data.
- Verify retries preserve tenant context.
- Verify tenant-scoped idempotency and uniqueness.
- Verify connection-pool reuse does not leak transaction-scoped tenant settings.

---

## 21. Anti-Patterns

The following are prohibited unless an explicit architectural exception is approved and documented.

- Performing external API calls inside database transactions.
- Treating sequential writes to multiple datastores as an atomic transaction.
- Assuming a successful Redis write means a business transaction committed.
- Assuming OpenSearch updates are immediately consistent with authoritative state.
- Publishing critical events before the database commit.
- Retrying non-idempotent operations blindly.
- Retrying only the failed statement after a transaction-level failure.
- Using application-side availability checks as the only protection against double booking.
- Using read-modify-write operations without concurrency control.
- Holding transactions open while waiting for user input or long-running work.
- Using distributed locks as a replacement for database integrity constraints.
- Ignoring uncertain commit outcomes.
- Silently swallowing rollback failures.
- Using broad isolation levels without evaluating contention.
- Creating unbounded transactions for large imports or backfills.
- Allowing repositories to commit transactions owned by a higher application layer.
- Assuming compensation is equivalent to rollback.
- Omitting reconciliation for important cross-system workflows.
- Mixing tenant data in a transaction without explicit authorization.
- Treating cache or projection failures as proof that an already-committed authoritative transaction failed.

---

## 22. Required Documentation

Every critical transactional workflow MUST document:

- Business purpose and invariants.
- Authoritative datastore and ownership.
- Transaction boundary.
- Isolation and concurrency-control strategy.
- Locking and uniqueness requirements.
- Idempotency behavior.
- Retryable and non-retryable failures.
- Unknown commit outcome handling.
- Cross-datastore consistency strategy.
- Event publication and consumer behavior.
- Compensation and reconciliation paths, where applicable.
- Tenant isolation requirements.
- Observability and alerting.
- Integration and failure-injection tests.

Complex workflows SHOULD include a sequence diagram or state-transition diagram in the relevant architecture documentation.

---

## 23. Transaction Review Checklist

Before approving a transaction-related implementation, verify:

- [ ] The business invariant is clearly defined.
- [ ] The authoritative datastore is identified.
- [ ] The transaction boundary is explicit.
- [ ] All required atomic writes share the same transaction.
- [ ] Constraints enforce critical integrity rules.
- [ ] Isolation and concurrency controls are appropriate.
- [ ] Tenant isolation is enforced throughout the workflow.
- [ ] External calls are outside the transaction.
- [ ] Idempotency is implemented where duplicate execution is possible.
- [ ] Retry behavior is bounded and safe.
- [ ] Unknown commit outcomes are handled.
- [ ] Cross-datastore workflows use an explicit consistency pattern.
- [ ] Events are emitted only for committed state.
- [ ] Compensation and reconciliation are defined where required.
- [ ] Transaction duration and resource usage are bounded.
- [ ] Logs, metrics, and traces are sufficient for diagnosis.
- [ ] Atomicity, concurrency, failure, and tenant-isolation tests exist.
- [ ] No sensitive information is exposed in logs or errors.
- [ ] Documentation is updated.

---

## 24. Definition of Done

A transactional implementation is complete only when:

- Its business invariants are explicitly defined and enforced.
- Its authoritative datastore and transaction ownership are clear.
- Atomicity and isolation requirements are implemented correctly.
- Concurrent execution cannot silently violate business rules.
- Idempotency and retry behavior are safe.
- Cross-system consistency and recovery are documented.
- Tenant boundaries are enforced.
- Failure scenarios have been tested.
- Observability supports production diagnosis.
- Performance and connection usage are bounded.
- Security and audit requirements are met.
- Relevant architecture and data documentation is updated.

**Final rule:** Every KAMPYN transaction MUST preserve the integrity of authoritative business state. When atomicity cannot span the full workflow, the implementation MUST provide an explicit, durable, observable, and recoverable consistency strategy rather than relying on assumptions about successful execution.
# KAMPYN Event Architecture

## 1. Purpose

This document defines the event-driven architecture for KAMPYN.

Events allow KAMPYN domains and infrastructure components to communicate without creating unnecessary direct dependencies.

Events may be used for:

- Cross-domain communication.
- Asynchronous processing.
- Search projection.
- Cache invalidation.
- Notifications.
- Analytics.
- Audit processing.
- Background workflows.
- External integrations.
- Long-running operations.

Events must not be introduced merely because asynchronous architecture appears more scalable.

The primary goals are:

- Clear ownership.
- Reliable delivery.
- Idempotent processing.
- Controlled coupling.
- Observable behavior.
- Recoverability.
- Explicit consistency boundaries.

---

# 2. Event Architecture Principles

KAMPYN events follow these principles:

1. The producer owns the event meaning.
2. Events represent facts that have occurred.
3. Consumers must not modify the producer's authoritative state directly.
4. Event delivery is generally asynchronous.
5. Consumers must be idempotent.
6. Eventual consistency must be explicit.
7. Event schemas are contracts.
8. Events must be observable.
9. Failed events must be recoverable.
10. Events must not silently replace transactions.

---

# 3. Event Flow

The general architecture is:

```text id="q8m3v7"
┌────────────────────┐
│   Domain Operation │
└─────────┬──────────┘
          ↓
┌────────────────────┐
│ Authoritative DB   │
└─────────┬──────────┘
          ↓
┌────────────────────┐
│    Outbox Event    │
└─────────┬──────────┘
          ↓
┌────────────────────┐
│ Event Publisher    │
└─────────┬──────────┘
          ↓
┌────────────────────┐
│ Event Infrastructure│
└─────────┬──────────┘
          │
     ┌────┼─────┬──────────┐
     ↓    ↓     ↓          ↓
 Search Cache Notification Analytics
```

The authoritative database mutation and event creation must be coordinated when reliable event publication is required.

---

# 4. Events Represent Facts

An event should describe something that happened.

Prefer:

```text id="n5r2k8"
OrderPlaced
BookingConfirmed
PaymentCaptured
InventoryAdjusted
UserRegistered
FoodItemUpdated
```

over commands disguised as events:

```text id="x7m4p2"
CreateOrder
ReserveInventory
SendEmail
UpdateSearch
```

Commands request an action.

Events communicate that an action has occurred.

---

# 5. Commands vs Events

Commands:

```text id="j3q8v1"
"Please perform this operation."
```

Events:

```text id="c6m2n9"
"This operation has happened."
```

Example:

```text id="k8p4r2"
Command
    ↓
PlaceOrder
    ↓
Order Domain
    ↓
OrderPlaced
```

Consumers should react to `OrderPlaced` rather than receiving a command to modify the order database.

---

# 6. Domain Events

Domain events represent meaningful business facts inside a domain.

Examples:

```text id="r4n7m2"
OrderPlaced
OrderCancelled
BookingConfirmed
BookingCancelled
InventoryReserved
InventoryReleased
PaymentCaptured
PaymentFailed
```

Domain events should originate from meaningful state transitions rather than low-level database operations.

Avoid events such as:

```text id="z2q8m5"
RowUpdated
ColumnChanged
DocumentSaved
```

unless a specific infrastructure mechanism requires them.

---

# 7. Integration Events

Integration events are events intended for consumers outside the originating domain or service boundary.

They should be designed as stable contracts.

Example:

```text id="v6p3k9"
Ordering Domain
      ↓
OrderPlaced Integration Event
      ↓
 ┌────┼──────────────┐
 ↓    ↓              ↓
Search Analytics Notifications
```

Integration events should expose only the information consumers legitimately need.

---

# 8. Event Ownership

Every event must have an owner.

The owner is responsible for:

- Event meaning.
- Schema.
- Versioning.
- Publication behavior.
- Documentation.
- Compatibility.
- Deprecation.

For example:

```text id="a9m4q7"
Ordering Domain
    ↓
OrderPlaced
```

The analytics system does not own `OrderPlaced`.

It consumes it.

---

# 9. Event Naming

Event names should describe completed facts.

Prefer past-tense names:

```text id="w3k8p1"
UserRegistered
OrderPlaced
OrderCancelled
PaymentCaptured
BookingConfirmed
```

Avoid ambiguous names:

```text id="d6q2n8"
OrderEvent
UserEvent
DataChanged
SomethingUpdated
```

The name should communicate what happened without requiring the consumer to inspect implementation details.

---

# 10. Event Granularity

Events should represent meaningful business transitions.

Too coarse:

```text id="e4m7q2"
OrderChanged
```

Too granular:

```text id="n8p3c6"
OrderFieldUpdated
OrderTimestampChanged
OrderStatusFieldUpdated
```

Prefer:

```text id="v2r9m5"
OrderPlaced
OrderConfirmed
OrderCancelled
OrderCompleted
```

The appropriate granularity depends on the domain contract.

---

# 11. Event Payloads

Event payloads should contain the information required by legitimate consumers.

Typical metadata:

```text id="c7n4x2"
event_id
event_type
event_version
occurred_at
producer
tenant_id
aggregate_type
aggregate_id
correlation_id
causation_id
```

Business payload:

```text id="m8q2v5"
{
  order_id,
  customer_id,
  ...
}
```

Do not include entire database records by default.

---

# 12. Event Metadata

Events should contain enough metadata to support tracing and processing.

Recommended fields include:

```text id="f5r8k3"
event_id
event_type
event_version
occurred_at
producer
tenant_id
aggregate_id
correlation_id
causation_id
```

Additional metadata may be introduced where operationally useful.

Metadata must not become a dumping ground for arbitrary application state.

---

# 13. Event IDs

Every published event should have a unique event identifier.

Example:

```text id="p3m7v8"
event_id = UUID
```

This supports:

- Deduplication.
- Idempotency.
- Debugging.
- Tracing.
- Replay tracking.

Event IDs must remain stable across retries of the same event.

A retry must not create a new logical event identity.

---

# 14. Aggregate Identity

Where appropriate, events should identify the aggregate or primary resource that changed.

Example:

```text id="j7n2q4"
aggregate_type = "order"
aggregate_id   = "..."
```

This allows consumers to:

- Partition events.
- Preserve ordering where required.
- Correlate related events.
- Rebuild projections.

---

# 15. Tenant Context

Tenant-aware events must carry tenant context.

Example:

```text id="c8m4r1"
tenant_id
```

Consumers must validate tenant scope before processing tenant-owned data.

Never assume that event routing alone provides tenant isolation.

---

# 16. Sensitive Data

Events should contain the minimum information required.

Do not publish:

- Passwords.
- Authentication tokens.
- Secrets.
- Unnecessary personal information.
- Internal credentials.
- Sensitive database fields.

If consumers need sensitive information, explicitly evaluate whether the event is the correct transport mechanism.

---

# 17. Event Schema

Events are contracts.

A schema should define:

- Event name.
- Version.
- Metadata.
- Required fields.
- Optional fields.
- Data types.
- Semantics.
- Compatibility rules.

Example:

```text id="s4q8n2"
OrderPlaced.v1

Metadata
  event_id
  occurred_at
  tenant_id
  aggregate_id

Payload
  order_id
  customer_id
  total_amount
  currency
```

The event schema must be documented and testable.

---

# 18. Schema Evolution

Event schemas must evolve without unexpectedly breaking consumers.

Prefer additive changes:

```text id="r7m2x5"
v1
  ↓
Add optional field
  ↓
Consumers adopt field
```

Avoid removing or changing the meaning of an existing field without a compatibility strategy.

---

# 19. Event Versioning

Version events when compatibility cannot be maintained through additive changes.

Possible representations:

```text id="b3q8n6"
OrderPlaced.v1
OrderPlaced.v2
```

or an explicit version field:

```text id="p9m4k2"
event_type: OrderPlaced
event_version: 2
```

The chosen strategy must be consistent across the platform.

---

# 20. Backward Compatibility

During deployment:

```text id="x5n7c3"
Old Producer
New Consumers
```

and:

```text id="m8q2r6"
New Producer
Old Consumers
```

may temporarily coexist.

Event changes must account for this.

Do not deploy a breaking event schema assuming every consumer updates simultaneously.

---

# 21. Transactional Outbox

When an event must reliably correspond to a database mutation, use a transactional outbox.

Example:

```text id="k2v8m4"
┌─────────────────────────────┐
│ Database Transaction        │
│                             │
│ Update Order                │
│ Insert OrderPlaced Event    │
│                             │
│ COMMIT                      │
└──────────────┬──────────────┘
               ↓
         Outbox Processor
               ↓
        Event Infrastructure
```

This prevents:

```text id="q6n3p8"
Database Commit
      ↓
Application crashes
      ↓
Event never published
```

The outbox entry becomes the durable bridge between transactional state and asynchronous publication.

---

# 22. Outbox Ownership

The outbox belongs to the authoritative datastore/domain responsible for the event.

Application code should not:

```text id="v4r8m2"
Commit DB
   ↓
Publish event separately
```

when reliable atomic coordination is required.

Instead:

```text id="f7n2c5"
Commit DB + Outbox
        ↓
Publish asynchronously
```

---

# 23. Outbox Processing

An outbox processor should:

1. Find unpublished events.
2. Claim or safely select work.
3. Publish the event.
4. Record successful publication.
5. Retry failures.
6. Avoid indefinite processing loops.

Processing must be safe under multiple workers.

---

# 24. Outbox Idempotency

Publishing may be retried.

Therefore:

```text id="m2q8v4"
Same event_id
     ↓
Multiple delivery attempts
```

must remain safe.

Consumers must assume duplicate delivery is possible.

---

# 25. Delivery Guarantees

KAMPYN should generally design around **at-least-once delivery**.

Conceptually:

```text id="p7n3k8"
Event
 ↓
Delivered
 ↓
Consumer fails
 ↓
Retry
 ↓
Event delivered again
```

Consumers must therefore be idempotent.

Exactly-once processing should not be assumed merely because a messaging system provides an exactly-once feature.

---

# 26. Idempotent Consumers

Consumers must safely process the same logical event multiple times.

Possible mechanisms include:

- Event ID deduplication.
- Unique constraints.
- Idempotency records.
- Upserts.
- Version checks.
- State-transition guards.

Example:

```text id="c4m8q2"
OrderPlaced
     ↓
Create Projection
     ↓
Duplicate OrderPlaced
     ↓
No duplicate projection
```

Idempotency must be designed at the consumer boundary.

---

# 27. Event Processing Records

For important consumers, maintain durable processing state where necessary.

Example:

```text id="r8v2m5"
event_id
consumer_name
processed_at
status
```

This can support:

- Deduplication.
- Investigation.
- Replay.
- Operational recovery.

Do not store processing records indefinitely without a retention policy.

---

# 28. Event Ordering

Events are not automatically globally ordered.

Ordering requirements must be defined per aggregate or partition.

Example:

```text id="n6q3p8"
OrderPlaced
OrderConfirmed
OrderCompleted
```

must not be processed as:

```text id="k2m7r4"
OrderCompleted
OrderPlaced
OrderConfirmed
```

when order matters.

Use partitioning, sequence numbers, or another explicit ordering mechanism where required.

---

# 29. Ordering Scope

Prefer the smallest ordering scope necessary.

For example:

```text id="x8p4m1"
Order A
  event 1
  event 2
  event 3

Order B
  event 1
  event 2
```

does not necessarily require a globally ordered stream.

Global ordering can unnecessarily reduce throughput.

Prefer per-aggregate or per-partition ordering where business semantics allow it.

---

# 30. Event Sequence Numbers

For domains requiring strict ordering, events may contain an aggregate sequence.

Example:

```text id="q4n7c2"
OrderPlaced       sequence=1
OrderConfirmed    sequence=2
OrderCompleted    sequence=3
```

Consumers can detect:

- Missing events.
- Out-of-order events.
- Duplicate events.

---

# 31. Event Replay

Events may be replayed to:

- Rebuild projections.
- Recover failed consumers.
- Create new read models.
- Repair derived systems.

Replay must be safe.

Consumers must not accidentally trigger irreversible external side effects during historical replay.

---

# 32. Replay-Aware Consumers

Consumers should distinguish between:

```text id="v3m8q1"
Live Processing
```

and:

```text id="n5r2k7"
Historical Replay
```

when the behavior differs.

For example, replaying historical `OrderPlaced` events should not necessarily send thousands of real customer notifications.

---

# 33. Dead-Letter Handling

Events that repeatedly fail should eventually move to a dead-letter mechanism.

Conceptually:

```text id="b8q4m2"
Event
 ↓
Consumer
 ↓
Failure
 ↓
Retry
 ↓
Retry
 ↓
Retry exhausted
 ↓
Dead Letter
```

Dead-letter events must remain inspectable and recoverable.

---

# 34. Retry Policy

Retries should use:

- Bounded attempts.
- Exponential backoff.
- Jitter where appropriate.
- Clear retryable/non-retryable classification.

Retry:

```text id="m6q2n8"
Temporary network failure
Service unavailable
Transient database failure
```

Do not endlessly retry:

```text id="r4p8v3"
Invalid schema
Invalid authorization
Malformed event
Permanent business rule violation
```

---

# 35. Poison Events

A poison event repeatedly fails because its contents or processing assumptions are invalid.

These events must not block unrelated events indefinitely.

Use:

- Dead-letter handling.
- Failure classification.
- Alerting.
- Manual remediation.
- Replay after correction.

---

# 36. Event Consumer Isolation

One failing consumer should not prevent unrelated consumers from processing an event.

Conceptually:

```text id="q7m3n8"
OrderPlaced
    │
    ├── Search Consumer
    ├── Analytics Consumer
    ├── Notification Consumer
    └── Audit Consumer
```

A failure in the notification consumer should not prevent the search projection from updating.

---

# 37. Event Fan-Out

Events may have many consumers.

Each consumer must own its processing behavior.

Avoid creating a central consumer that performs unrelated responsibilities:

```text id="x2v8m4"
OrderPlaced
    ↓
God Consumer
    ├── Search
    ├── Email
    ├── Analytics
    ├── Cache
    └── Billing
```

Prefer independent consumers with clear ownership.

---

# 38. Event Dependencies

Consumers should depend on event contracts rather than producer implementation details.

A consumer should not require access to:

- Producer database tables.
- Producer internal packages.
- Producer private domain objects.
- Producer repository implementations.

The event contract is the integration boundary.

---

# 39. Cross-Domain Events

Events are useful for reducing direct dependencies between KAMPYN domains.

Example:

```text id="g4m8q2"
Ordering
   ↓
OrderPlaced
   ↓
Inventory
```

The inventory domain consumes the event rather than directly accessing ordering internals.

However, if the operation requires immediate atomic consistency, a synchronous application-level interaction may be more appropriate.

Do not force event-driven architecture onto inherently transactional operations.

---

# 40. Synchronous vs Asynchronous Communication

Use synchronous communication when:

- The caller requires an immediate result.
- The operation must be part of one transaction.
- Failure must immediately reach the caller.
- The operation is simple and bounded.

Use asynchronous events when:

- Processing can happen later.
- Consumers can operate independently.
- Eventual consistency is acceptable.
- Work is expensive.
- Multiple consumers need the same fact.
- Failure should not block the original operation.

---

# 41. Notifications

Notifications should generally be event-driven.

Example:

```text id="n7m4q2"
OrderConfirmed
      ↓
Notification Consumer
      ↓
Email / Push / SMS / In-App
```

Notification delivery must be idempotent where duplicate messages would be harmful.

The notification system should not become part of the order transaction.

---

# 42. Cache Invalidation

Events may be used for cache invalidation.

Example:

```text id="p5v8m2"
FoodItemUpdated
      ↓
Cache Consumer
      ↓
Invalidate food-item cache
```

Cache invalidation must remain safe if:

- The event is delayed.
- The event is duplicated.
- The event is processed out of order.

See:

```text id="y0m8q4"
architecture/caching.md
```

for cache architecture.

---

# 43. OpenSearch Projection

Events are a primary mechanism for updating OpenSearch projections.

Example:

```text id="r3q7n2"
FoodItemUpdated
      ↓
Search Projection Consumer
      ↓
Transform
      ↓
OpenSearch
```

The consumer must be idempotent.

A failed OpenSearch update must not corrupt the authoritative database.

---

# 44. Analytics

Events can provide a clean input stream for analytics.

Examples:

```text id="m8c4q2"
OrderPlaced
OrderCompleted
BookingConfirmed
PaymentCaptured
SearchPerformed
```

Analytics consumers should not affect transactional workflows.

Analytical processing should be isolated from latency-sensitive business operations.

---

# 45. Audit Events

Important domain actions may generate audit events.

Example:

```text id="v7n2k5"
RoleAssigned
      ↓
Audit Consumer
      ↓
Durable Audit Record
```

Audit processing must be designed so that important audit requirements are not accidentally lost during transient event failures.

---

# 46. Background Jobs

Not every background task needs a domain event.

Use a background job when the primary requirement is:

```text id="c8m3q7"
"Perform this task asynchronously."
```

Use an event when the primary requirement is:

```text id="n4p7v2"
"This business fact occurred."
```

For example:

```text id="x6m2q8"
GenerateMonthlyReport
```

may be a job.

```text id="q3r8n5"
OrderCompleted
```

is an event.

---

# 47. Scheduled Events

Scheduled processing should not pretend to be domain events if no business fact has occurred.

A scheduler may produce a command/job:

```text id="m5v2q8"
RunDailyInventoryReconciliation
```

which performs work and may subsequently emit:

```text id="p8n4c2"
InventoryReconciliationCompleted
```

Keep these concepts separate.

---

# 48. Event-Driven Workflows

Long-running workflows may combine events and state.

Example:

```text id="w7m3q2"
OrderPlaced
     ↓
PaymentRequested
     ↓
PaymentCaptured
     ↓
OrderConfirmed
     ↓
InventoryReserved
     ↓
OrderReady
```

Complex workflows should use explicit state rather than relying only on event history.

The current workflow state must remain queryable.

---

# 49. Saga-Style Workflows

When a business workflow crosses independent transactional boundaries, a saga-style architecture may be appropriate.

Example:

```text id="q8n4m2"
Order
  ↓
Payment
  ↓
Inventory
  ↓
Fulfillment
```

Each step has its own transaction.

If a later step fails, a compensating action may be required.

Example:

```text id="m2r7v5"
Inventory Reserved
      ↓
Payment Failed
      ↓
Release Inventory
```

Compensation is not the same as database rollback.

---

# 50. Eventual Consistency in Workflows

Event-driven workflows introduce delays.

User-facing APIs must therefore distinguish:

```text id="v4q8n2"
Request accepted
```

from:

```text id="n7m3c5"
Operation completed
```

For long-running operations, expose an explicit status model.

---

# 51. Correlation IDs

Every event-driven workflow should support correlation.

Example:

```text id="c2m8q4"
HTTP Request
   ↓
correlation_id = X
   ↓
OrderPlaced
   ↓
PaymentCaptured
   ↓
OrderConfirmed
```

This allows operators to trace one business workflow across services and consumers.

---

# 52. Causation IDs

Where useful, events should carry a causation identifier.

Example:

```text id="r5n7k2"
Event A
  event_id = A

Event B
  causation_id = A
```

This allows event chains to be reconstructed.

---

# 53. Observability

Event systems must expose operational visibility.

Useful metrics include:

- Events published.
- Events consumed.
- Processing latency.
- Consumer lag.
- Retry count.
- Dead-letter count.
- Processing failures.
- Event age.
- Outbox backlog.
- Queue depth.

Logs should include:

- Event ID.
- Event type.
- Tenant ID where appropriate.
- Aggregate ID.
- Correlation ID.
- Consumer name.
- Processing attempt.

---

# 54. Event Lag

Event lag measures the delay between:

```text id="m3q8v2"
Event Created
      ↓
Event Consumed
```

High lag may indicate:

- Consumer overload.
- Broker congestion.
- Database bottlenecks.
- Failed consumers.
- Insufficient worker capacity.

Event lag must be monitored for workflows where freshness matters.

---

# 55. Backpressure

Consumers must not process unlimited events concurrently.

Use:

- Bounded worker pools.
- Queue limits.
- Rate limiting.
- Concurrency controls.
- Consumer-specific capacity.

Do not allow a sudden event spike to exhaust:

- Database connections.
- Memory.
- CPU.
- External API quotas.

---

# 56. Consumer Concurrency

Consumer concurrency should be based on the downstream resource.

For example:

```text id="x7m4q2"
1000 events
      ↓
Consumer worker pool
      ↓
bounded concurrency
      ↓
Database / API
```

More workers do not necessarily mean more throughput.

Measure downstream saturation.

---

# 57. External Event Consumers

When consuming events from external systems, validate:

- Schema.
- Authentication.
- Signature where applicable.
- Event origin.
- Timestamp.
- Replay behavior.
- Idempotency.

Never blindly trust an external event payload.

---

# 58. Webhooks vs Events

Webhooks are external HTTP delivery mechanisms.

Events are internal or platform-level asynchronous facts.

A typical integration may be:

```text id="q4n8m2"
External Provider
      ↓
Webhook
      ↓
KAMPYN API
      ↓
Validate
      ↓
Persist State
      ↓
Internal Event
      ↓
Consumers
```

Do not treat an unverified webhook as an internal trusted event.

---

# 59. Event Security

Event infrastructure must enforce:

- Authentication.
- Authorization.
- Tenant isolation.
- Transport security.
- Access control.
- Secret management.

Event payloads should be treated as trusted only after passing the appropriate validation boundary.

---

# 60. Event Retention

Event retention should be based on purpose.

Different streams may require different retention periods:

```text id="n5q2v8"
Operational events
→ short/medium retention

Audit events
→ longer retention

Analytics events
→ analytics-specific retention
```

Retention must consider:

- Storage cost.
- Compliance.
- Privacy.
- Replay requirements.

---

# 61. Event Deletion and Privacy

Events may contain personal information.

If an event stream is retained for a long period, determine how privacy and deletion requirements apply.

Prefer publishing stable identifiers rather than unnecessary personal information.

Do not duplicate sensitive user data into every downstream system.

---

# 62. Event Replay and External Side Effects

Replay must not accidentally repeat irreversible actions.

Dangerous examples:

```text id="x2q8m5"
Send payment
Send SMS
Charge card
Create external booking
```

Consumers that perform external side effects should use:

- Idempotency keys.
- Durable state.
- Replay guards.
- Explicit replay modes.

---

# 63. Event Infrastructure Abstraction

Application domains should not depend directly on broker-specific APIs.

Prefer:

```text id="p7m3n8"
Domain
   ↓
Event Publisher Interface
   ↓
Infrastructure Adapter
   ↓
Message Broker
```

This keeps messaging infrastructure replaceable.

The abstraction must not hide important delivery semantics.

---

# 64. Broker Selection

The event infrastructure should be selected based on actual requirements.

Relevant considerations include:

- Throughput.
- Ordering.
- Durability.
- Replay.
- Consumer groups.
- Delivery semantics.
- Operational complexity.
- Self-hosting requirements.
- Cloud compatibility.

Do not introduce Kafka, NATS, RabbitMQ, or another broker simply because it is commonly used.

The selected infrastructure must match KAMPYN's actual event workload.

---

# 65. Self-Hosted Event Infrastructure

Universities may self-host KAMPYN.

Therefore event infrastructure should support:

- Institution-controlled deployment.
- Configurable broker endpoints.
- Authentication.
- Persistent storage.
- Monitoring.
- Backup/recovery where applicable.
- Version compatibility.
- Migration procedures.

The core KAMPYN application must not depend on EXSOLVIA-hosted event infrastructure for correctness.

---

# 66. Event Testing

Event-driven components must be tested for:

- Event schema.
- Publication.
- Serialization.
- Deserialization.
- Duplicate delivery.
- Retry.
- Dead-letter behavior.
- Ordering.
- Out-of-order events.
- Consumer failure.
- Event replay.
- Tenant isolation.
- Authorization-sensitive consumers.
- External side effects.
- Outbox processing.

Testing only the happy path is insufficient.

---

# 67. Contract Testing

Producer and consumer contracts should be tested independently.

Producer tests should verify:

```text id="k3q8m5"
Published event
    ↓
Matches contract
```

Consumer tests should verify:

```text id="n7v2r4"
Contract-compliant event
    ↓
Correct consumer behavior
```

This reduces the risk of independent deployments breaking event consumers.

---

# 68. Event Schema Registry

If KAMPYN reaches a scale where many producers and consumers share event contracts, a schema registry may be introduced.

A registry can provide:

- Schema storage.
- Version management.
- Compatibility checks.
- Discoverability.

Do not introduce a registry before the number of event contracts justifies its operational cost.

---

# 69. Event Documentation

Every externally consumed event should document:

- Event name.
- Purpose.
- Producer.
- Consumers.
- Version.
- Schema.
- Required fields.
- Optional fields.
- Delivery semantics.
- Ordering semantics.
- Retry behavior.
- Retention.
- Deprecation policy.

Events are APIs and must be documented accordingly.

---

# 70. Event Change Checklist

Before completing an event-related change:

- [ ] Event represents a meaningful fact.
- [ ] Producer ownership is clear.
- [ ] Consumer ownership is clear.
- [ ] Event schema is defined.
- [ ] Event version is defined.
- [ ] Tenant context is preserved.
- [ ] Sensitive data is minimized.
- [ ] Event ID is unique and stable.
- [ ] Correlation is supported.
- [ ] Causation is considered.
- [ ] Delivery semantics are explicit.
- [ ] Consumers are idempotent.
- [ ] Ordering requirements are explicit.
- [ ] Retry behavior is defined.
- [ ] Dead-letter handling exists where necessary.
- [ ] Outbox is used where transactional reliability requires it.
- [ ] Replay behavior is understood.
- [ ] External side effects are protected against duplicate execution.
- [ ] Event lag is observable.
- [ ] Consumer concurrency is bounded.
- [ ] Schema compatibility is tested.
- [ ] Documentation is updated.
- [ ] Self-hosted deployment implications are considered.

---

# 71. Final Event Principle

Events should communicate **what happened**, not hide how the system works.

The preferred architecture is:

```text id="z6m2q8"
             Domain Operation
                    ↓
          ┌──────────────────┐
          │ Authoritative DB │
          └────────┬─────────┘
                   │
              Transaction
                   │
             ┌─────┴─────┐
             │   Outbox  │
             └─────┬─────┘
                   ↓
             Event Broker
                   │
        ┌──────────┼──────────┐
        ↓          ↓          ↓
    Search       Cache    Notifications
    Consumer     Consumer    Consumer
        │          │          │
        ↓          ↓          ↓
 OpenSearch      Redis      Providers
```

The core guarantees are:

```text id="p4n8v2"
Authoritative state
        ↓
Reliable publication
        ↓
At-least-once delivery
        ↓
Idempotent consumers
        ↓
Observable processing
        ↓
Recoverable failures
```

Events should reduce unnecessary coupling while keeping correctness explicit.

They must never become an excuse to hide transactions, ignore failures, or distribute business logic without clear ownership.

The event architecture should optimize for:

```text id="r7m3q5"
Correctness
    ↓
Reliability
    ↓
Clear ownership
    ↓
Recoverability
    ↓
Scalability
    ↓
Operational simplicity
```

A good event system makes asynchronous behavior understandable, observable, replayable, and safe.
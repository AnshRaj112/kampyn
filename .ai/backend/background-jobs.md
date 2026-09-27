# KAMPYN Backend Background Jobs

## Purpose

Background jobs handle work that should not block synchronous API requests.

They are responsible for asynchronous, delayed, scheduled, retryable, or resource-intensive work such as:

- Notifications
- Email and SMS delivery
- Search indexing
- Cache invalidation
- Report generation
- Data exports
- File processing
- Analytics aggregation
- Inventory reconciliation
- Booking expiration
- Payment reconciliation
- Webhook processing
- Cleanup and retention
- Scheduled university operations
- Integration synchronization

Background jobs are part of the backend application architecture and must preserve the same guarantees as synchronous requests:

- Correctness
- Authorization
- Tenant isolation
- Idempotency
- Observability
- Security
- Failure isolation
- Bounded resource usage
- Explicit ownership

Jobs must never become an uncontrolled second application layer containing duplicated business logic.

---

# 1. Core Principles

## 1.1 Jobs are asynchronous execution mechanisms

A job describes **when or how work executes**, not where business rules live.

```text
Job
 ↓
Application Use Case
 ↓
Domain Logic
 ↓
Infrastructure
```

Do not place core business rules directly inside workers.

Bad:

```text
Worker
 ├── calculate order total
 ├── modify inventory
 ├── create payment
 └── send notification
```

Preferred:

```text
Worker
 ↓
ProcessOrderPaymentUseCase
 ↓
PaymentService
 ↓
Repositories / Integrations
```

The same application use case should be reusable from:

- HTTP handlers
- Background jobs
- Event consumers
- CLI/admin operations
- Scheduled tasks

---

# 2. When to Use a Background Job

Use a background job when work:

- Does not need to complete before the HTTP response
- Is expensive
- Involves external services
- Can tolerate eventual completion
- Requires retries
- May take significant time
- Processes large datasets
- Needs scheduled execution
- Should be isolated from request latency
- Needs controlled concurrency
- Can be resumed after failure

Examples:

```text
POST /orders
    ↓
Create Order
    ↓
Return response
    ↓
Publish event
    ↓
Background job
    ├── Send notification
    ├── Update search
    └── Update analytics
```

Do not use a job merely to hide slow application code.

If a request is inherently synchronous, keep it synchronous.

---

# 3. When NOT to Use a Background Job

Do not move work to a background job when the caller requires the result to continue.

Examples:

```text
Authenticate user
Authorize request
Create order
Validate payment request
Check booking availability
Validate inventory before order creation
```

These operations generally belong inside the synchronous request flow.

Do not create jobs for trivial in-process operations where asynchronous execution provides no meaningful benefit.

---

# 4. Job Architecture

KAMPYN background processing follows:

```text
                    ┌────────────────────┐
                    │   API / Scheduler  │
                    └─────────┬──────────┘
                              │
                              ▼
                    ┌────────────────────┐
                    │   Job Dispatcher   │
                    └─────────┬──────────┘
                              │
                              ▼
                    ┌────────────────────┐
                    │      Queue         │
                    │      Redis         │
                    └─────────┬──────────┘
                              │
             ┌────────────────┼────────────────┐
             │                │                │
             ▼                ▼                ▼
       Worker Pool       Worker Pool       Worker Pool
             │                │                │
             ▼                ▼                ▼
        Application      Application      Application
          Use Case          Use Case          Use Case
             │                │                │
             └────────────────┼────────────────┘
                              │
                              ▼
                    Infrastructure / DB
```

The queue is responsible for delivery and scheduling.

Workers are responsible for execution.

Application services remain responsible for business behavior.

---

# 5. Job Lifecycle

Every job should have an explicit lifecycle.

```text
Created
   ↓
Queued
   ↓
Claimed
   ↓
Running
   ↓
 ┌───────────────┐
 │               │
 ▼               ▼
Succeeded      Failed
                 │
                 ▼
              Retry?
              /    \
            Yes     No
             │       │
             ▼       ▼
          Queued   Dead Letter
```

A job must not remain indefinitely in an ambiguous state.

Workers must distinguish:

- Pending
- Running
- Succeeded
- Retryable failure
- Permanent failure
- Cancelled
- Dead-lettered

---

# 6. Job Definition

Every job should have a stable job type.

Example:

```text
notification.send
search.index
report.generate
payment.reconcile
booking.expire
inventory.reconcile
file.process
analytics.aggregate
integration.sync
```

Job types should be:

- Explicit
- Stable
- Versionable
- Domain-oriented
- Independently observable

Avoid generic names such as:

```text
processData
backgroundTask
runTask
doSomething
```

---

# 7. Job Payload

Job payloads should contain only the information required to execute the job.

Example:

```json
{
  "jobId": "job_123",
  "type": "notification.send",
  "version": 1,
  "tenantId": "tenant_123",
  "entityId": "order_123",
  "attempt": 1
}
```

Avoid putting large domain objects into queues.

Prefer:

```text
tenantId
entityId
operation parameters
```

over:

```text
entire Order object
entire User object
entire Inventory object
```

Workers should retrieve authoritative state from the source of truth when necessary.

---

# 8. Job Metadata

Jobs should carry sufficient metadata for execution and observability.

Recommended metadata:

```text
jobId
jobType
jobVersion
tenantId
entityId
attempt
createdAt
scheduledAt
startedAt
deadline
correlationId
causationId
priority
producer
```

Do not place secrets, passwords, access tokens, payment credentials, or unnecessary personal information inside job payloads.

---

# 9. Tenant Context

Every tenant-scoped job must carry tenant context.

Example:

```text
tenantId
```

Workers must establish tenant context before executing tenant-scoped application logic.

```text
Job
 ↓
Tenant Context
 ↓
Authorization / Scope
 ↓
Application Use Case
 ↓
Repository
```

A worker must never accidentally execute a tenant-scoped operation without tenant context.

Never derive tenant identity from mutable or ambiguous data when the tenant ID can be explicitly included in the job.

---

# 10. Tenant Isolation

Background jobs must obey the same tenant isolation rules as API requests.

The following must be tenant-aware where applicable:

- Database queries
- Cache keys
- Search documents
- Object storage paths
- Notifications
- Events
- Analytics
- Exports
- Reports
- Integrations
- Audit records

Example:

```text
tenant:{tenantId}:job:{jobId}
```

Never allow a worker to use global queries where tenant-scoped queries are required.

---

# 11. Idempotency

Background jobs must be safe against duplicate execution.

At-least-once delivery means a job may execute more than once.

Therefore:

```text
Same job
 +
Same operation
 =
Same final result
```

where practical.

Examples:

```text
notification.send
payment.reconcile
search.index
inventory.reconcile
booking.expire
```

must handle duplicate execution safely.

---

# 12. Job Idempotency Keys

Use deterministic idempotency keys when appropriate.

Example:

```text
notification:{notificationId}
```

or:

```text
payment-reconciliation:{paymentId}:{version}
```

or:

```text
search-index:{entityType}:{entityId}:{version}
```

The idempotency mechanism must be atomic.

Do not rely on:

```text
if not exists:
    create
```

without appropriate concurrency protection.

---

# 13. Database Transactions

A worker must not assume that queue delivery and database state are automatically atomic.

For operations requiring both database mutation and job publication, use the transactional outbox pattern where appropriate.

```text
Transaction
 ├── Domain mutation
 └── Outbox event
        ↓
Commit
        ↓
Outbox publisher
        ↓
Queue
        ↓
Worker
```

Do not:

```text
Update DB
Commit
Publish job
```

without considering the failure window between the two operations.

---

# 14. Retries

Retries are required for transient failures.

Examples:

- Temporary network failure
- External API timeout
- Database connection failure
- Temporary provider outage
- Rate limiting
- Temporary infrastructure failure

Retries must not be used for permanent failures.

Examples of generally non-retryable failures:

- Invalid payload
- Unauthorized operation
- Missing required entity
- Unsupported job version
- Invalid configuration
- Permanently rejected external request

---

# 15. Exponential Backoff

Retry delays should increase between attempts.

Example:

```text
Attempt 1 → immediate
Attempt 2 → short delay
Attempt 3 → longer delay
Attempt 4 → longer delay
Attempt 5 → maximum delay
```

Use bounded exponential backoff with jitter.

Avoid synchronized retries from many workers.

---

# 16. Retry Limits

Every retryable job must have a maximum retry policy.

Example:

```text
maxAttempts = 5
```

After retry exhaustion:

```text
Worker
 ↓
Dead Letter Queue
```

Never retry indefinitely.

Infinite retries can create:

- Queue congestion
- Cost amplification
- Duplicate side effects
- External provider overload
- Resource exhaustion

---

# 17. Dead-Letter Jobs

Jobs that cannot successfully complete after their retry policy should be moved to a dead-letter mechanism.

Dead-letter records should preserve enough information to investigate the failure.

Recommended metadata:

```text
jobId
jobType
tenantId
attempts
lastError
firstFailedAt
lastFailedAt
originalPayloadReference
correlationId
```

Sensitive payload data should not be copied unnecessarily.

Dead-letter jobs should support:

- Inspection
- Alerting
- Manual retry
- Cancellation
- Root-cause analysis

---

# 18. Poison Jobs

A poison job is a job that repeatedly fails because the payload or execution path is permanently invalid.

Examples:

```text
Invalid schema
Unsupported version
Missing required entity
Corrupted file
Invalid integration configuration
```

Poison jobs must not continuously circulate through the queue.

They should be:

```text
Detected
 ↓
Stopped
 ↓
Dead-lettered
 ↓
Alerted
```

---

# 19. Job Timeouts

Every job should have a bounded execution time where practical.

Example:

```text
job timeout = 30 seconds
```

Long-running jobs require explicit design.

Never allow a worker to remain blocked indefinitely.

Timeouts must propagate through:

- Context
- Database operations
- HTTP clients
- External integrations
- Storage operations

---

# 20. Cancellation

Workers must respect cancellation signals.

Go workers should propagate `context.Context`.

Example:

```text
Worker
 ↓
context.Context
 ├── DB operation
 ├── HTTP request
 ├── storage operation
 └── application service
```

When Kubernetes terminates a worker:

```text
SIGTERM
 ↓
Stop accepting new work
 ↓
Cancel active work where safe
 ↓
Finish safe operations
 ↓
Release resources
 ↓
Exit
```

Never abruptly terminate workers while they are performing non-idempotent critical operations without a recovery strategy.

---

# 21. Concurrency

Workers must use bounded concurrency.

Do not create unbounded goroutines for queue processing.

Bad:

```text
for job := range jobs {
    go process(job)
}
```

Preferred:

```text
Queue
 ↓
Bounded Worker Pool
 ↓
N workers
```

Concurrency should be controlled based on:

- CPU
- Memory
- Database capacity
- External API rate limits
- Queue throughput
- Job type
- Tenant limits

---

# 22. Per-Job Concurrency

Different job types may require different concurrency limits.

Example:

```text
notification.send       → high concurrency
search.index             → moderate concurrency
report.generate          → low concurrency
payment.reconcile        → tightly controlled
file.process             → bounded by memory
```

Resource-heavy jobs must not starve latency-sensitive jobs.

Prefer separate queues or worker pools when workload characteristics differ significantly.

---

# 23. Queue Priorities

Where supported, jobs may have priorities.

Example:

```text
Critical
High
Normal
Low
```

Priority must not allow low-priority work to become permanently starved.

Use priority queues carefully and document their behavior.

---

# 24. Scheduled Jobs

Scheduled jobs handle recurring or delayed work.

Examples:

```text
Expire unpaid orders
Expire bookings
Generate daily reports
Aggregate analytics
Clean temporary files
Process retention policies
Synchronize university systems
Reconcile payments
```

Scheduling should be explicit.

```text
Scheduler
 ↓
Create Job
 ↓
Queue
 ↓
Worker
```

Do not put the actual business logic inside the scheduler.

---

# 25. Cron Jobs

Kubernetes CronJobs may be used for scheduled execution where appropriate.

Preferred structure:

```text
Kubernetes CronJob
        ↓
Job Dispatcher
        ↓
Queue
        ↓
Worker
```

The CronJob should trigger work rather than contain complex domain logic.

Scheduled execution must also be idempotent because schedulers may retry or overlap executions.

---

# 26. Preventing Overlapping Scheduled Jobs

A scheduled job must define whether concurrent executions are allowed.

Examples:

```text
daily.analytics.aggregate
```

may require:

```text
one execution per tenant per period
```

while:

```text
notification.send
```

may safely run concurrently.

Use appropriate locking or deterministic execution keys.

Example:

```text
analytics:{tenantId}:{date}
```

---

# 27. Distributed Scheduling

In a Kubernetes deployment with multiple backend replicas:

```text
API-1
API-2
Worker-1
Worker-2
Worker-3
```

do not assume in-memory scheduling is globally unique.

Bad:

```text
Every application replica
    ↓
runs the same cron
```

This can execute the same scheduled operation multiple times.

Scheduling ownership must be explicit through:

- Kubernetes CronJobs
- Distributed locks
- Queue-based scheduling
- Database coordination
- Another centralized scheduling mechanism

---

# 28. Job Deduplication

Duplicate jobs should be prevented where duplicate work provides no value.

Example:

```text
search.index(product_123)
search.index(product_123)
search.index(product_123)
```

can potentially be coalesced into one operation.

Deduplication must not compromise correctness.

Do not deduplicate operations where every event represents a meaningful state transition.

---

# 29. Event-Driven Jobs

Domain and integration events may produce background jobs.

Example:

```text
OrderCreated
    ↓
Outbox
    ↓
Event Consumer
    ↓
notification.send
```

Another example:

```text
ProductUpdated
    ↓
ProductUpdated Event
    ↓
search.index
```

The event identifies **what happened**.

The job represents **work that must be performed**.

Keep these concepts separate.

---

# 30. Job Consumers

Consumers must:

1. Receive the message
2. Validate the job envelope
3. Establish context
4. Verify tenant scope
5. Validate payload
6. Execute the application use case
7. Record outcome
8. Acknowledge only after appropriate processing
9. Retry transient failures
10. Dead-letter permanent failures

Never acknowledge a job before the required work is safely completed.

---

# 31. Acknowledgement

Queue acknowledgement semantics must be explicit.

Preferred:

```text
Receive
 ↓
Process
 ↓
Persist required state
 ↓
Acknowledge
```

Avoid:

```text
Receive
 ↓
Acknowledge
 ↓
Process
```

because a worker crash after acknowledgement can lose the job.

---

# 32. Exactly-Once Assumptions

Do not design the application assuming exactly-once execution.

The system should generally assume:

```text
At-least-once delivery
+
Possible duplicate execution
```

Correctness should therefore come from:

- Idempotency
- Unique constraints
- Transactions
- Version checks
- State transitions
- Deduplication
- Outbox/inbox patterns

---

# 33. State Transitions

Jobs that modify state must respect domain state machines.

Example:

```text
PENDING
  ↓
CONFIRMED
  ↓
COMPLETED
```

A worker must not blindly overwrite state.

Before changing state:

```text
Expected state
+
Allowed transition
=
Valid mutation
```

This prevents stale or duplicate jobs from corrupting domain state.

---

# 34. Versioning

Job payloads may outlive the code version that created them.

Therefore job schemas should be versioned.

Example:

```text
notification.send.v1
notification.send.v2
```

or:

```json
{
  "type": "notification.send",
  "version": 2
}
```

Workers should support currently valid versions during rolling deployments.

Do not deploy a worker that immediately invalidates queued jobs created by the previous release.

---

# 35. Deployment Compatibility

During deployment:

```text
Old Worker
+
New Worker
+
Existing Queue
```

may temporarily coexist.

Therefore:

- Payloads must remain compatible
- Job schemas must be versioned
- Database migrations must support rolling deployment
- Consumers must tolerate previous versions
- Producers must not immediately require unsupported consumer behavior

Use expand-and-contract migration strategies where required.

---

# 36. Large File Jobs

Large files must not be loaded entirely into memory merely because they are being processed asynchronously.

Use:

```text
Object Storage
 ↓
Streaming
 ↓
Bounded Processing
 ↓
Output
```

rather than:

```text
Download entire file
 ↓
Load entire file into RAM
 ↓
Process
```

This is especially important for:

- Imports
- Exports
- Reports
- Validation
- CSV/TSV processing
- JSON/XML processing
- Data reconciliation

---

# 37. Job Resource Limits

Every worker deployment should have explicit resource constraints.

At minimum consider:

```text
CPU requests
CPU limits
Memory requests
Memory limits
Concurrency
Timeout
Queue depth
```

Memory-heavy jobs must have stricter concurrency limits.

Example:

```text
1 worker
2 concurrent large-file jobs
```

may be safer than:

```text
1 worker
50 concurrent large-file jobs
```

---

# 38. Database Interaction

Workers must use repositories/application services rather than bypassing the backend architecture.

Preferred:

```text
Worker
 ↓
Application Service
 ↓
Repository
 ↓
Database
```

Avoid:

```text
Worker
 ↓
Raw SQL everywhere
```

unless the operation is explicitly an infrastructure concern and follows the repository/data-access architecture.

Workers must also avoid N+1 queries and unbounded database operations.

---

# 39. Batch Processing

Jobs processing many entities should use batching.

Bad:

```text
for each user:
    query database
    update database
```

Preferred:

```text
Read batch
 ↓
Process batch
 ↓
Bulk operation
 ↓
Next batch
```

Batch size should be bounded and configurable.

Do not choose extremely large batches that create memory or transaction pressure.

---

# 40. Transactions

Keep transaction boundaries explicit.

Do not hold long database transactions while waiting for:

- HTTP requests
- Email providers
- Payment providers
- Object storage
- External university systems

Bad:

```text
BEGIN
 ↓
DB mutation
 ↓
HTTP request
 ↓
WAIT
 ↓
COMMIT
```

Preferred:

```text
Short DB transaction
 ↓
Commit
 ↓
External operation
 ↓
Persist result
```

Use events/outbox/workflows when multiple systems must coordinate.

---

# 41. External Services

Background jobs are often used for external integrations.

Every external call must define:

- Timeout
- Retry policy
- Rate-limit behavior
- Authentication
- Error classification
- Idempotency
- Circuit-breaking behavior where appropriate
- Observability

Never retry non-idempotent external operations blindly.

---

# 42. Notifications

Notification jobs may handle:

```text
Email
SMS
Push
In-app notification
```

The worker should invoke a notification application service.

Example:

```text
notification.send
 ↓
NotificationService
 ↓
Provider Adapter
 ↓
External Provider
```

Provider-specific implementation must remain behind the integration boundary.

---

# 43. Search Indexing Jobs

Search indexing should be asynchronous.

Example:

```text
ProductUpdated
 ↓
Outbox
 ↓
Search Index Job
 ↓
OpenSearch
```

The source database remains authoritative.

If the search index becomes inconsistent:

```text
Rebuild
 ↓
Source Database
 ↓
Index
```

Do not treat OpenSearch as the primary transactional source.

---

# 44. Cache Jobs

Background jobs may perform:

- Cache warming
- Cache invalidation
- Expiration
- Precomputation

Cache failures must not corrupt authoritative data.

If Redis is unavailable:

```text
Job
 ↓
Redis failure
 ↓
Retry / fallback
```

Do not modify the source of truth merely to compensate for cache failure.

---

# 45. Analytics Jobs

Analytics aggregation should generally be asynchronous.

Example:

```text
OrderCreated
 ↓
Event
 ↓
Analytics Job
 ↓
Aggregation
 ↓
Analytics Read Model
```

Analytics workloads must not degrade transactional request performance.

Large aggregations should be:

- Bounded
- Incremental where possible
- Partitioned
- Observable
- Retryable

---

# 46. Reports and Exports

Report generation should generally run asynchronously.

Example:

```text
POST /reports
 ↓
Create report request
 ↓
Queue job
 ↓
Worker
 ↓
Generate report
 ↓
Object Storage
 ↓
Mark report ready
 ↓
Notify user
```

Do not keep HTTP requests open while generating large reports.

Generated files must have:

- Authorization checks
- Tenant isolation
- Expiration
- Secure storage
- Access logging where appropriate

---

# 47. Cleanup Jobs

Cleanup jobs may handle:

- Expired temporary files
- Expired sessions
- Old job metadata
- Retention policies
- Dead-letter cleanup
- Temporary exports
- Old audit data where policy permits

Cleanup must be incremental.

Avoid:

```text
DELETE millions of records
```

in one transaction.

Prefer bounded batches.

---

# 48. Inventory Jobs

Inventory-related jobs require special care.

Examples:

```text
Inventory reconciliation
Low-stock notifications
Reservation expiry
Inventory projection
```

Inventory correctness must come from transactional domain operations.

A reconciliation job must detect discrepancies rather than blindly overwrite authoritative inventory without an explicit business rule.

---

# 49. Booking Jobs

Booking-related scheduled jobs may include:

```text
Expire unpaid booking
Release expired reservation
Send reminder
Mark completed booking
```

These operations must verify current state before mutation.

Example:

```text
Job says:
Booking should expire

Worker checks:
Booking still PENDING?
        │
       Yes → expire
        │
       No  → no-op
```

This protects against stale jobs.

---

# 50. Payment Jobs

Payment jobs require strict idempotency.

Examples:

```text
payment.reconcile
payment.verify
payment.webhook.process
payment.refund
```

Never assume that a job executing once means a payment operation happened once.

Use:

- Provider transaction IDs
- Internal payment IDs
- Idempotency keys
- Unique constraints
- Explicit payment states
- Reconciliation

Payment state must remain auditable.

---

# 51. Webhook Jobs

External webhooks may be converted into background jobs.

```text
Webhook
 ↓
Verify signature
 ↓
Persist event
 ↓
Acknowledge provider
 ↓
Queue processing job
 ↓
Application logic
```

Do not perform long-running processing before acknowledging a provider when the provider expects a fast response.

Webhook processing must be idempotent.

---

# 52. Job Security

Treat job payloads as untrusted input.

Validate:

- Job type
- Version
- Tenant ID
- Entity ID
- Parameters
- Authorization context
- State assumptions

Never execute arbitrary commands based on job payloads.

Never deserialize untrusted data into unsafe executable structures.

---

# 53. Secrets

Never store secrets directly inside job payloads.

Bad:

```json
{
  "apiKey": "...",
  "password": "..."
}
```

Preferred:

```text
tenantId
integrationId
```

The worker retrieves credentials through the secure configuration/integration system.

---

# 54. Logging

Every job should produce structured logs containing safe identifiers.

Recommended:

```text
job_id
job_type
tenant_id
attempt
correlation_id
entity_id
duration
outcome
```

Never log:

- Passwords
- Tokens
- API keys
- Payment credentials
- Sensitive personal information
- Entire untrusted payloads

---

# 55. Metrics

Track at minimum:

```text
jobs_total
jobs_succeeded_total
jobs_failed_total
jobs_retried_total
jobs_dead_lettered_total
job_duration_seconds
queue_depth
queue_wait_seconds
active_workers
```

Useful dimensions:

```text
job_type
tenant
outcome
```

Avoid unbounded metric labels.

Do not use arbitrary IDs such as:

```text
user_id
order_id
job_id
```

as metric labels.

---

# 56. Tracing

Jobs should preserve distributed tracing context where available.

Example:

```text
HTTP Request
 ↓
OrderCreated
 ↓
Job
 ↓
Worker
 ↓
External Provider
```

Use:

```text
trace_id
correlation_id
causation_id
```

to connect asynchronous operations.

A job should begin a new execution span while retaining linkage to the originating trace where supported.

---

# 57. Queue Monitoring

Monitor:

- Queue depth
- Queue age
- Processing latency
- Retry rate
- Failure rate
- Dead-letter count
- Worker utilization
- Job duration
- Scheduling delay

Queue depth alone is insufficient.

A queue with 10,000 jobs may be healthy if workers process them quickly.

A queue with 100 jobs may be unhealthy if they are blocked for hours.

---

# 58. Backpressure

Workers must support backpressure.

If downstream systems become overloaded:

```text
Database overloaded
        ↓
Reduce worker concurrency
        ↓
Queue grows temporarily
        ↓
Recover
        ↓
Drain safely
```

Do not increase concurrency indefinitely to reduce queue depth.

Backpressure protects:

- PostgreSQL
- MongoDB
- Redis
- OpenSearch
- External APIs
- Worker memory
- CPU

---

# 59. Fairness Between Tenants

A large tenant must not monopolize worker capacity.

Where workloads justify it, use:

```text
Global concurrency limit
+
Per-tenant concurrency limit
```

Example:

```text
Global: 100 jobs
Tenant A: max 20
Tenant B: max 20
Tenant C: max 20
```

This protects multi-tenant reliability.

---

# 60. Failure Isolation

Different workload classes should be isolated when necessary.

For example:

```text
Critical Jobs
 ├── Payment
 └── Booking

Standard Jobs
 ├── Notifications
 └── Search

Heavy Jobs
 ├── Reports
 └── File Processing
```

Heavy jobs should not consume all worker capacity required by critical operations.

Separate queues or worker deployments may be appropriate.

---

# 61. Worker Deployment

Workers should be independently scalable from API servers.

Example:

```text
                KAMPYN
                   │
       ┌───────────┴───────────┐
       │                       │
    API Pods              Worker Pods
       │                       │
       │                       ▼
       │                    Queues
       │                       │
       └────── PostgreSQL ─────┤
               MongoDB         │
               Redis           │
               OpenSearch      │
```

Scale workers based on queue workload rather than HTTP traffic.

---

# 62. Autoscaling

Worker autoscaling may consider:

```text
Queue depth
Queue age
Processing latency
CPU
Memory
```

CPU alone is often insufficient for queue workloads.

A worker can be CPU-idle while thousands of jobs are waiting.

---

# 63. Graceful Shutdown

On shutdown:

```text
Stop accepting new jobs
        ↓
Finish or safely cancel active jobs
        ↓
Release queue claims
        ↓
Close DB connections
        ↓
Close external clients
        ↓
Exit
```

Jobs must be recoverable if a worker terminates unexpectedly.

This is another reason idempotency is mandatory.

---

# 64. Job Ownership

Every job should have a clear owning domain or subsystem.

Examples:

```text
orders/
    jobs/

payments/
    jobs/

notifications/
    jobs/

search/
    jobs/

reports/
    jobs/
```

Avoid a single global package containing unrelated business logic.

Shared infrastructure may live under infrastructure-level packages.

---

# 65. Suggested Backend Organization

A conceptual structure:

```text
backend/
├── cmd/
│   ├── api/
│   └── worker/
│
├── internal/
│   ├── application/
│   │   ├── orders/
│   │   ├── payments/
│   │   ├── bookings/
│   │   ├── inventory/
│   │   ├── notifications/
│   │   └── reports/
│   │
│   ├── domain/
│   │
│   ├── jobs/
│   │   ├── dispatcher/
│   │   ├── queue/
│   │   ├── worker/
│   │   └── scheduler/
│   │
│   └── infrastructure/
│       ├── database/
│       ├── redis/
│       ├── search/
│       ├── storage/
│       └── integrations/
```

Domain-specific job handlers should remain close to their application use cases.

---

# 66. Job Handler Structure

A job handler should remain thin.

Preferred:

```text
Handle(ctx, job)
    ↓
Validate envelope
    ↓
Build execution context
    ↓
Call application use case
    ↓
Classify result
```

Avoid:

```text
Handle()
    ├── query DB
    ├── calculate business rules
    ├── call provider
    ├── mutate multiple tables
    ├── update cache
    ├── update search
    └── send email
```

The latter becomes an unmaintainable second application layer.

---

# 67. Error Classification

Every job failure should be classified.

```text
Success
Retryable
Permanent
Cancelled
Unknown
```

Example:

```text
HTTP 429 → Retryable
HTTP 503 → Retryable

Invalid request → Permanent
Invalid credentials → Permanent or configuration failure

Context cancellation → Cancelled
```

Classification must be based on the actual operation and provider semantics.

Do not retry blindly based only on HTTP status.

---

# 68. Error Handling

Errors must preserve context.

Bad:

```text
job failed
```

Preferred:

```text
process notification job: send email: provider timeout
```

Errors should include enough context for debugging without exposing sensitive data.

Do not swallow errors merely to prevent retries.

---

# 69. Partial Failure

Some jobs process multiple independent records.

Example:

```text
Generate notifications for 10,000 users
```

One failed record should not necessarily fail the entire job.

Use bounded batch processing:

```text
Batch
 ├── success
 ├── success
 ├── failure → retry/dead-letter
 └── success
```

The failure model must be explicit.

Do not silently discard failed records.

---

# 70. Job Result Persistence

Long-running jobs may require persistent status.

Example:

```text
Report
 ├── id
 ├── tenantId
 ├── status
 ├── requestedBy
 ├── createdAt
 ├── completedAt
 ├── failureReason
 └── outputReference
```

The API can expose:

```text
GET /reports/{id}
```

without requiring the API request to wait for processing.

---

# 71. Job Cancellation

Long-running user-requested jobs should support cancellation when practical.

Example:

```text
POST /reports/{id}/cancel
```

Cancellation should update durable state.

Workers should periodically check cancellation for long-running work.

Cancellation must be safe and idempotent.

---

# 72. Retention

Job records, execution metadata, and dead-letter data require retention policies.

Define:

```text
How long execution history is stored
How long dead-letter records are retained
When payload references expire
When temporary outputs are deleted
```

Retention must respect audit and compliance requirements.

---

# 73. Testing

Background jobs require multiple levels of testing.

## Unit tests

Test:

- Job validation
- Retry classification
- Idempotency logic
- State transitions
- Payload parsing
- Error classification

## Integration tests

Test:

- Queue integration
- Database interaction
- Redis
- OpenSearch
- Object storage
- External provider adapters

## Contract tests

Test:

- Job schemas
- Event-to-job contracts
- Provider contracts

## Failure tests

Test:

- Worker crash
- Duplicate delivery
- Timeout
- Retry exhaustion
- Dead-lettering
- Dependency outage
- Cancellation
- Shutdown

---

# 74. Idempotency Tests

Every job with side effects should have a duplicate execution test.

Example:

```text
Execute job
Execute same job again
```

Expected:

```text
No duplicate side effect
```

or, where duplication is intentionally allowed:

```text
Explicitly documented duplicate behavior
```

---

# 75. Concurrency Tests

Test concurrent execution of the same job.

Example:

```text
Worker A ──┐
           ├── same job
Worker B ──┘
```

Verify that:

- State remains valid
- Unique constraints hold
- No duplicate payment occurs
- No inventory corruption occurs
- No duplicate booking occurs

---

# 76. Performance Testing

Measure:

```text
Throughput
Latency
Queue wait
Execution duration
Memory
CPU
Database load
External API utilization
```

For large jobs test realistic workloads.

Examples:

```text
10,000 records
100,000 records
1,000,000 records
Large file
Multiple tenants
Concurrent workers
```

Do not benchmark only the happy path with tiny datasets.

---

# 77. Operational Alerts

Alert on meaningful conditions such as:

```text
High queue age
Repeated job failures
Dead-letter growth
Retry storms
Worker crash loops
External provider failure
Unusual job duration
Persistent queue growth
Tenant-specific overload
```

Alerts should represent actionable conditions.

Avoid alerting on every individual transient failure.

---

# 78. Manual Operations

Operators may need to:

```text
Inspect job
Retry job
Cancel job
Replay event
Drain queue
Pause worker
Resume worker
Inspect dead-letter jobs
```

Administrative operations must be:

- Authenticated
- Authorized
- Audited
- Tenant-scoped where appropriate

Never expose arbitrary job execution to ordinary users.

---

# 79. Reprocessing

Jobs should be reprocessable when business requirements justify it.

Reprocessing must not bypass:

- Authorization
- Tenant isolation
- Idempotency
- State validation
- Audit logging

A manual retry should use the same application path as normal execution.

---

# 80. Disaster Recovery

Queues are part of the operational architecture.

Define what happens when:

```text
Redis unavailable
Worker deployment lost
Queue data lost
Database unavailable
External provider unavailable
```

Critical jobs should have a recovery mechanism.

For durable business events, the authoritative record should not exist only inside an ephemeral queue.

Use durable database state and outbox/event patterns where required.

---

# 81. Self-Hosted Deployments

KAMPYN self-hosted deployments must be able to run background workers independently.

A deployment may contain:

```text
kampyn-api
kampyn-worker
kampyn-scheduler
kampyn-postgres
kampyn-mongodb
kampyn-redis
kampyn-opensearch
```

or an equivalent deployment model.

Job configuration must be environment-driven.

Do not hard-code:

- Queue endpoints
- Concurrency
- Retry counts
- Provider credentials
- Tenant configuration

---

# 82. Configuration

Job configuration should be explicit.

Examples:

```text
WORKER_CONCURRENCY
JOB_MAX_ATTEMPTS
JOB_DEFAULT_TIMEOUT
JOB_RETRY_BASE_DELAY
JOB_RETRY_MAX_DELAY
QUEUE_NAME
QUEUE_PREFIX
SCHEDULER_ENABLED
```

Configuration must be validated at startup.

Invalid production configuration should fail fast.

---

# 83. Environment Separation

Development, staging, and production workers must use isolated infrastructure.

Never allow:

```text
development worker
        ↓
production queue
```

or:

```text
staging worker
        ↓
production database
```

unless explicitly designed for a controlled operational procedure.

---

# 84. Definition of Done

A background job is complete only when:

- Job ownership is clear
- Job type is stable
- Payload is validated
- Tenant context is explicit
- Authorization requirements are defined
- Application logic is reusable
- Idempotency is addressed
- Retry policy is defined
- Permanent failures are classified
- Dead-letter behavior exists where needed
- Timeout is defined
- Cancellation behavior is defined where relevant
- Concurrency is bounded
- Resource consumption is understood
- Transactions are explicit
- External calls have timeouts
- Observability exists
- Metrics exist where operationally relevant
- Structured logging exists
- Tests cover success and failure
- Duplicate execution is tested
- Deployment compatibility is considered
- Documentation exists for operational behavior
- Security and tenant isolation are verified

---

# 85. Final Invariants

The following rules are mandatory:

```text
1. Jobs are execution mechanisms, not a second business-logic layer.

2. Every tenant-scoped job carries explicit tenant context.

3. Background work assumes at-least-once execution.

4. Side-effecting jobs must be idempotent or explicitly document
   why duplicate execution is safe.

5. Retries are bounded.

6. Retryable and permanent failures are distinguished.

7. Poison jobs must not circulate indefinitely.

8. Every long-running operation has bounded resource usage.

9. Worker concurrency is explicitly controlled.

10. Scheduled jobs must tolerate duplicate execution or use
    explicit coordination.

11. Queue acknowledgement happens only after appropriate processing.

12. Business-critical state must not exist only inside an ephemeral queue.

13. Workers use application services rather than duplicating business logic.

14. Database transactions remain short and explicit.

15. External calls always have appropriate timeouts.

16. Jobs respect authentication, authorization, and tenant isolation.

17. Secrets never belong in job payloads.

18. Large files are processed through bounded streaming/chunking.

19. Heavy workloads must not starve critical workloads.

20. Worker shutdown must be graceful and recoverable.

21. Job schemas must remain compatible across rolling deployments.

22. Dead-lettered work must be observable and operationally recoverable.

23. Queue health is measured by both depth and age/latency.

24. Background processing must remain observable, testable,
    secure, and independently scalable.

25. If a job can corrupt business state when executed twice,
    its idempotency and concurrency strategy must be explicitly
    designed before implementation.
```

## Summary

KAMPYN background processing follows:

```text
Request / Event / Scheduler
            │
            ▼
      Job Dispatcher
            │
            ▼
          Queue
            │
            ▼
      Bounded Worker
            │
            ▼
    Tenant + Execution Context
            │
            ▼
    Application Use Case
            │
            ▼
       Domain Logic
            │
            ▼
 Infrastructure / Database / APIs
            │
            ▼
     Observable Result
```

The queue provides asynchronous execution.

The worker provides controlled execution.

The application layer provides business behavior.

The domain provides correctness.

Infrastructure provides persistence and external communication.

Idempotency, tenant isolation, bounded concurrency, retries, observability, and failure recovery make the complete system reliable.
# KAMPYN Backend Concurrency

## Purpose

Concurrency defines how KAMPYN safely executes multiple operations at the same time without producing race conditions, inconsistent state, duplicate side effects, deadlocks, resource exhaustion, or tenant interference.

KAMPYN is inherently concurrent.

Multiple users may simultaneously:

- Place orders
- Reserve inventory
- Book hostels
- Reserve guest rooms
- Schedule washing machines
- Book shuttle seats
- Make payments
- Update profiles
- Submit complaints
- Send community messages
- Trigger searches
- Upload files
- Generate reports
- Execute background jobs

The backend must therefore treat concurrency as a correctness concern, not merely a performance concern.

---

# 1. Core Principle

Concurrent execution must never violate domain invariants.

```text
Concurrent Requests
        │
        ▼
   Application
        │
        ▼
Concurrency Control
        │
        ▼
Consistent Domain State
```

The primary objective is:

```text
Correctness
    >
Performance
```

Concurrency optimization must never compromise correctness.

---

# 2. Concurrency Model

KAMPYN uses concurrency at multiple levels:

```text
                    KAMPYN
                       │
       ┌───────────────┼────────────────┐
       │               │                │
       ▼               ▼                ▼
   HTTP Requests   Background Jobs   Event Consumers
       │               │                │
       └───────────────┼────────────────┘
                       ▼
                  Application
                       │
          ┌────────────┼────────────┐
          ▼            ▼            ▼
       Database      Redis      External APIs
```

Concurrency must therefore be considered across:

- Goroutines
- HTTP requests
- Worker pools
- Database transactions
- Distributed workers
- Scheduled jobs
- Event consumers
- Cache operations
- External integrations

---

# 3. Concurrency Categories

KAMPYN must distinguish between:

### Local concurrency

Multiple goroutines inside one process.

### Database concurrency

Multiple transactions operating simultaneously.

### Distributed concurrency

Multiple backend instances operating simultaneously.

### External concurrency

Multiple operations against external services.

### User-level concurrency

Multiple actions from the same user.

### Tenant-level concurrency

Multiple operations belonging to the same university.

These require different synchronization mechanisms.

---

# 4. Never Assume Single-Instance Execution

Production KAMPYN may run:

```text
API-1
API-2
API-3
Worker-1
Worker-2
Worker-3
```

Therefore:

```text
In-memory synchronization
```

cannot guarantee global correctness.

This is unsafe for distributed state:

```go
var mu sync.Mutex
```

if another application instance can modify the same resource.

A local mutex protects only the current process.

---

# 5. Concurrency Control Hierarchy

Use the smallest appropriate synchronization boundary.

Preferred order:

```text
1. Immutable data
2. Atomic operation
3. Database constraint
4. Database transaction
5. Row/document locking
6. Optimistic concurrency
7. Distributed coordination
8. Application-level mutex
```

Do not introduce distributed locks when a database constraint or atomic operation can guarantee correctness.

---

# 6. Immutability

Prefer immutable data when practical.

Immutable values eliminate entire classes of race conditions.

Example:

```text
Request
  ↓
Validated immutable input
  ↓
Application operation
```

Avoid sharing mutable state across unrelated goroutines.

---

# 7. Goroutines

Goroutines are cheap but not free.

Every goroutine must have:

- Clear ownership
- Bounded lifetime
- Cancellation behavior
- Resource expectations
- Error handling

Bad:

```go
go processSomething()
```

when the caller has no way to:

- Cancel it
- Observe errors
- Know when it completes
- Prevent leaks

Prefer structured concurrency.

---

# 8. Context Propagation

Every request-scoped goroutine must receive the appropriate `context.Context`.

```go
go func(ctx context.Context) {
    // work
}(ctx)
```

Context should propagate through:

```text
HTTP
 ↓
Application
 ↓
Repository
 ↓
Database
```

and:

```text
Worker
 ↓
Application
 ↓
External API
```

Never create unrelated background contexts merely to avoid cancellation.

---

# 9. Context Cancellation

When the parent request is cancelled:

```text
Client disconnects
        ↓
Request context cancelled
        ↓
Application operation cancelled
        ↓
Database/API operations stop where possible
```

Long-running operations must respect cancellation.

This prevents:

- Goroutine leaks
- Unnecessary database work
- Unnecessary API calls
- Resource exhaustion

---

# 10. Goroutine Leak Prevention

A goroutine must always have a termination path.

Common leak causes:

```text
Blocked channel receive
Blocked channel send
Missing context cancellation
Unclosed worker loop
Unbounded goroutine creation
Network operation without timeout
```

Every long-lived goroutine must have an explicit lifecycle.

---

# 11. Worker Pools

Use bounded worker pools for repeated concurrent work.

Preferred:

```text
              Queue
                │
                ▼
        ┌───────────────┐
        │ Worker Pool   │
        ├───────────────┤
        │ Worker 1      │
        │ Worker 2      │
        │ Worker 3      │
        │ Worker N      │
        └───────────────┘
```

Avoid:

```go
for _, item := range items {
    go process(item)
}
```

for large or unbounded collections.

This can exhaust:

- Memory
- CPU
- Database connections
- External API quotas

---

# 12. Bounded Concurrency

Concurrency should have an explicit upper bound.

Example:

```text
100,000 records
       ↓
Batch size: 500
       ↓
Workers: 8
```

Not:

```text
100,000 records
       ↓
100,000 goroutines
```

The appropriate concurrency limit depends on the bottleneck.

---

# 13. Concurrency Is Not Always Faster

Increasing concurrency can reduce performance.

Example:

```text
Workers
  1 → 100 req/s
  4 → 350 req/s
  8 → 500 req/s
 16 → 480 req/s
 32 → 300 req/s
```

Beyond the system's capacity, additional concurrency creates contention.

Measure before increasing concurrency.

---

# 14. Database Connection Pools

Worker concurrency must respect database pool capacity.

Example:

```text
100 workers
      ↓
20 DB connections
      ↓
80 workers waiting
```

Increasing workers does not increase database capacity.

Worker concurrency should therefore be coordinated with:

- PostgreSQL pool size
- MongoDB pool size
- Redis connections
- External provider limits

---

# 15. Race Conditions

A race condition occurs when correctness depends on timing between concurrent operations.

Example:

```text
Inventory = 1

Request A reads 1
Request B reads 1

A buys item
B buys item

Final inventory = -1 or invalid state
```

The fix is not:

```text
sleep()
```

The fix is proper synchronization.

---

# 16. Check-Then-Act Race

Avoid:

```text
if inventory >= quantity:
    inventory -= quantity
```

when the check and update are separate operations.

Another request may modify the value between them.

Prefer an atomic database operation or transaction.

Example concept:

```sql
UPDATE inventory
SET quantity = quantity - $quantity
WHERE product_id = $id
  AND quantity >= $quantity;
```

Then verify affected rows.

---

# 17. Database as Synchronization Authority

For persistent business state, the database should usually be the synchronization authority.

Use:

- Transactions
- Unique constraints
- Foreign keys
- Conditional updates
- Row locks
- Optimistic version checks

Do not attempt to reproduce database consistency with application memory.

---

# 18. Transactions

Transactions must protect logically related state changes.

Example:

```text
Create Order
+
Reserve Inventory
+
Create Order Items
```

may require a transaction depending on the domain design.

The transaction should be:

```text
Short
Explicit
Bounded
Failure-safe
```

---

# 19. Transaction Duration

Do not hold transactions while performing slow external operations.

Bad:

```text
BEGIN
 ↓
Update DB
 ↓
Call payment provider
 ↓
Wait 5 seconds
 ↓
Call email provider
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

Use events/outbox/workflows when coordination is required.

---

# 20. Row-Level Locking

Use row locks when concurrent modifications to the same resource must be serialized.

Conceptually:

```sql
SELECT *
FROM inventory
WHERE id = $1
FOR UPDATE;
```

The lock should be held only for the required transaction.

Avoid locking unnecessarily broad sets of rows.

---

# 21. Lock Scope

Prefer:

```text
One resource
```

over:

```text
Entire table
```

Prefer:

```text
Specific inventory record
```

over:

```text
All inventory records
```

Broad locks increase contention and reduce throughput.

---

# 22. Optimistic Concurrency

Optimistic concurrency is useful when conflicts are relatively uncommon.

Example:

```text
version = 7
```

Update:

```sql
UPDATE resource
SET value = $new_value,
    version = version + 1
WHERE id = $id
  AND version = 7;
```

If zero rows are affected:

```text
Concurrent modification detected
```

The application can then:

- Retry
- Reload
- Return conflict
- Re-evaluate the operation

---

# 23. When to Use Optimistic Concurrency

Use optimistic concurrency for:

- User profile updates
- Administrative configuration
- Editable resources
- Long-running workflows
- Resources where conflicts are uncommon

Avoid blindly retrying operations where repeating the business action creates side effects.

---

# 24. Pessimistic Concurrency

Pessimistic locking is appropriate when concurrent modification must be serialized.

Examples may include:

- Inventory reservation
- Limited booking capacity
- Seat allocation
- Resource reservation

The exact mechanism must be based on the domain invariant.

---

# 25. Inventory Concurrency

Inventory is a high-contention domain.

Example:

```text
Available = 1

User A ─┐
        ├── Reserve
User B ─┘
```

Exactly one operation should succeed.

Use:

```text
Atomic conditional update
```

or:

```text
Transaction + appropriate lock
```

Never rely on:

```text
Read inventory
 ↓
Check in Go
 ↓
Write inventory
```

without concurrency protection.

---

# 26. Booking Concurrency

Bookings have similar requirements.

Example:

```text
1 washing machine slot
+
2 concurrent users
```

The system must guarantee:

```text
Successful bookings <= available capacity
```

Use:

- Database constraints
- Transactions
- Locking
- Unique constraints
- Explicit state transitions

depending on the resource.

---

# 27. Unique Constraints

Use database uniqueness to enforce uniqueness.

Example:

```text
tenant_id
resource_id
booking_slot
```

may form a unique constraint when the business rule requires one booking per slot.

This is stronger than:

```text
if no booking exists:
    create booking
```

because the database handles concurrent requests atomically.

---

# 28. Payments

Payment operations require strict concurrency control.

Possible concurrent operations:

```text
Payment webhook
Payment polling
User retry
Reconciliation job
Admin operation
```

All may attempt to modify the same payment.

Use:

- Stable payment IDs
- Provider transaction IDs
- Unique constraints
- Idempotency keys
- State-machine validation
- Transactions

Never blindly overwrite payment state.

---

# 29. State-Machine Concurrency

State transitions must be validated.

Example:

```text
PENDING
  ├── CONFIRMED
  └── CANCELLED
```

Two concurrent operations:

```text
Request A → CONFIRMED
Request B → CANCELLED
```

must be resolved according to explicit domain rules.

Never allow arbitrary:

```text
UPDATE status = ...
```

operations to bypass the state machine.

---

# 30. Compare-and-Set

For state changes, use compare-and-set semantics where appropriate.

Conceptually:

```text
UPDATE booking
SET status = CONFIRMED
WHERE id = ?
AND status = PENDING
```

Then:

```text
affected rows == 1
    → transition succeeded

affected rows == 0
    → stale/conflicting operation
```

This avoids separate read-then-write races.

---

# 31. Redis Concurrency

Redis can provide atomic operations and coordination mechanisms.

Use Redis for:

- Atomic counters
- Rate limits
- Temporary coordination
- Distributed locks where justified
- Queue infrastructure
- Short-lived state

Do not automatically move persistent business invariants into Redis.

---

# 32. Distributed Locks

Distributed locks may be necessary when coordination cannot be safely handled through database constraints or transactions.

Examples:

```text
One global scheduled operation
One expensive rebuild
One tenant provisioning operation
```

A distributed lock must have:

- Ownership
- Expiration
- Failure handling
- Safe release
- Clear scope
- Observability

Never create a lock without defining what happens if the lock holder crashes.

---

# 33. Lock Expiration

Distributed locks must not live forever.

Conceptually:

```text
Acquire
 ↓
Lease expires
 ↓
Another worker can proceed
```

However, expiration introduces correctness risks.

A lock implementation must prevent an expired owner from continuing to perform protected operations as if it still owned the lock.

---

# 34. Do Not Overuse Distributed Locks

Bad:

```text
Every database update
    ↓
Redis distributed lock
```

This creates unnecessary:

- Latency
- Complexity
- Failure modes
- Operational dependency

Prefer database-native concurrency controls when the authoritative state lives in the database.

---

# 35. Deadlocks

Deadlocks occur when concurrent transactions wait on each other.

Example:

```text
Transaction A:
Lock A
 ↓
Wait for B

Transaction B:
Lock B
 ↓
Wait for A
```

Avoid by:

- Consistent lock ordering
- Short transactions
- Narrow lock scope
- Avoiding unnecessary locks
- Appropriate indexes
- Monitoring database deadlocks

---

# 36. Lock Ordering

When multiple resources must be locked, acquire them in deterministic order.

Example:

```text
Always lock:
resource A
then resource B
```

Never:

```text
Request 1:
A → B

Request 2:
B → A
```

This creates deadlock risk.

---

# 37. Retry After Deadlock

Some database deadlocks can be safely retried.

The retry must:

- Roll back the failed transaction
- Start a fresh transaction
- Have a bounded retry count
- Include jitter where appropriate

Never continue using a failed transaction as though it were valid.

---

# 38. Atomicity Across Services

Do not attempt to create distributed transactions across all KAMPYN services.

Avoid:

```text
Service A
 ↓
Service B
 ↓
Service C
 ↓
Service D
```

with one giant transaction.

Prefer:

```text
Local transaction
 ↓
Event
 ↓
Next operation
 ↓
Local transaction
```

Use sagas/workflows when multiple durable state transitions must coordinate.

---

# 39. Events and Concurrency

Events may arrive:

- Out of order
- More than once
- Concurrently

Consumers must therefore be resilient.

Use:

- Idempotency
- Event versions
- Aggregate versions
- Sequence numbers where necessary
- State validation

Never assume event order unless the architecture explicitly guarantees it.

---

# 40. Out-of-Order Events

Example:

```text
ProductUpdated v3
ProductUpdated v2
```

If v2 arrives after v3, the consumer must not blindly overwrite the newer state.

Possible mechanisms:

```text
version check
sequence number
event timestamp where appropriate
rebuild from source of truth
```

Do not rely solely on timestamps for strict ordering.

---

# 41. Background Job Concurrency

Multiple workers may receive the same logical operation.

Example:

```text
Worker A ─┐
          ├── booking.expire
Worker B ─┘
```

The operation must remain safe.

Use:

```text
State check
+
Atomic transition
+
Idempotency
```

rather than assuming one worker receives each job exactly once.

---

# 42. Queue Concurrency

Queue processing must have explicit limits.

Consider:

```text
Global workers
Per-queue workers
Per-job-type concurrency
Per-tenant concurrency
External-provider limits
```

A single queue should not automatically receive unlimited workers.

---

# 43. External API Concurrency

External providers may impose rate limits.

Example:

```text
Provider limit = 100 requests/sec
```

KAMPYN must not run:

```text
500 concurrent requests/sec
```

because internal worker capacity is higher.

Use:

- Rate limiting
- Bounded concurrency
- Backoff
- Provider-specific worker pools

---

# 44. File Processing Concurrency

Large file processing requires memory-aware concurrency.

Example:

```text
10 GB worker memory
+
5 GB processing workload
```

should not automatically run ten concurrent jobs.

Use:

```text
Memory budget
÷
Estimated job memory
=
Maximum safe concurrency
```

The estimate should include:

- Buffers
- Parsers
- Temporary structures
- Runtime overhead
- Database/network buffers

---

# 45. Streaming

Concurrency should not be used as a substitute for efficient processing.

Bad:

```text
Read entire file
 ↓
Create goroutine per row
```

Preferred:

```text
Stream
 ↓
Bounded batches
 ↓
Worker pool
 ↓
Aggregate/write results
```

This keeps memory bounded.

---

# 46. Channel Usage

Go channels should represent clear ownership and communication.

Use channels when:

- Passing work
- Coordinating goroutines
- Streaming data
- Signaling completion
- Propagating bounded pipelines

Do not use channels merely because concurrent code exists.

A mutex or atomic operation may be simpler for shared state.

---

# 47. Channel Ownership

The goroutine responsible for producing values should generally own closing the channel.

Avoid multiple goroutines independently closing the same channel.

Closing a channel more than once causes a panic.

---

# 48. Buffered Channels

Buffered channels can absorb bounded bursts.

Example:

```text
Producer
   ↓
[ bounded buffer ]
   ↓
Consumer
```

Buffer sizes must have a reason.

Do not use enormous buffers to hide slow consumers.

An ever-growing buffer is not backpressure.

---

# 49. Select and Cancellation

Concurrent Go loops should generally support cancellation.

Conceptually:

```go
select {
case job := <-jobs:
    process(job)
case <-ctx.Done():
    return
}
```

This ensures workers can shut down cleanly.

---

# 50. Mutex Usage

Use `sync.Mutex` for protecting local mutable state.

Example:

```text
shared in-memory state
        │
        ▼
     Mutex
        │
        ▼
serialized access
```

Keep critical sections small.

Do not hold a mutex while performing:

- Database calls
- Network calls
- File I/O
- Slow computation

unless there is an explicit reason.

---

# 51. RWMutex

`sync.RWMutex` may be appropriate for read-heavy local state.

However, do not assume it is automatically faster.

Measure contention before choosing:

```text
sync.Mutex
```

versus:

```text
sync.RWMutex
```

The simplest correct synchronization primitive is preferred.

---

# 52. Atomics

Use atomic operations for simple shared state such as:

- Counters
- Flags
- Sequence values

Do not use atomics to implement complicated business invariants.

Complex state usually belongs behind:

- A mutex
- A transaction
- A database constraint

---

# 53. Maps and Concurrency

Go maps are not safe for concurrent writes.

Unsafe:

```go
map[string]Value
```

shared across concurrent writers without synchronization.

Use:

- Mutex
- `sync.Map` where appropriate
- Immutable snapshots
- Channel ownership

Do not default to `sync.Map`.

---

# 54. Shared Global State

Avoid mutable global state.

Bad:

```text
global mutable cache
global request state
global tenant state
global current user
```

This creates:

- Race conditions
- Test contamination
- Tenant isolation risks
- Difficult lifecycle management

Dependencies should be explicit.

---

# 55. Request Isolation

Never store request-specific information in process-global variables.

Examples:

```text
Current user
Current tenant
Current request ID
Current authorization
Current transaction
```

These belong in request-scoped context or explicit parameters.

---

# 56. Cache Concurrency

Cache reads and writes can race.

Example:

```text
Request A → cache miss
Request B → cache miss
Request C → cache miss

All three query DB
```

This can create a cache stampede.

Use:

- Request coalescing
- Short locks
- Jittered TTLs
- Controlled refresh
- Appropriate cache-aside patterns

where justified.

---

# 57. Cache Invalidation

Concurrent updates can produce stale cache entries.

Example:

```text
DB update
Cache write
Cache invalidation
```

Ordering must be deliberate.

Prefer event-driven invalidation where appropriate and define the expected consistency model.

---

# 58. Search Index Concurrency

OpenSearch indexing may receive concurrent updates for the same entity.

Example:

```text
Product v4
Product v5
Product v6
```

The index should not end up with v4 after v6.

Use:

- Versioning
- Ordering guarantees where available
- Reconciliation
- Reindexing
- Source-of-truth recovery

Search remains a derived read model.

---

# 59. Idempotency

Concurrency and idempotency are closely related.

A safe operation should tolerate:

```text
Retry
Duplicate request
Duplicate event
Duplicate job
Concurrent execution
```

when the domain permits it.

Idempotency is not a substitute for synchronization.

Both may be required.

---

# 60. Request Idempotency

For operations such as:

```text
Create payment
Create order
Create booking
```

the API may require an idempotency key.

Example:

```text
Idempotency-Key: abc123
```

The server must define:

- Key scope
- Expiration
- Request matching
- Stored result
- Concurrent duplicate behavior

Two concurrent requests with the same idempotency key must not create two independent side effects.

---

# 61. Duplicate Request Handling

Example:

```text
Request A ─┐
           ├── Idempotency-Key = X
Request B ─┘
```

Expected behavior:

```text
One operation
+
Consistent result
```

Do not implement this using an unsafe:

```text
check key
then create
```

sequence without atomic coordination.

---

# 62. Tenant-Level Concurrency

Multi-tenancy adds another concurrency dimension.

A single university may generate:

```text
Thousands of simultaneous requests
```

while another tenant generates only a few.

The system should prevent one tenant from exhausting shared resources.

Possible controls:

```text
Rate limits
Concurrency limits
Queue quotas
Connection budgets
Worker quotas
```

These are capacity controls, not authorization mechanisms.

---

# 63. Noisy Neighbor Protection

Protect shared resources from a tenant consuming disproportionate capacity.

Potential resources:

- API concurrency
- Database connections
- Background workers
- Search requests
- File processing
- Notifications
- External integrations

Limits should be configurable by deployment and tenant plan where applicable.

---

# 64. Authentication and Authorization

Concurrency controls must never bypass authorization.

Example:

```text
Request A
Tenant A
        │
        ▼
Resource

Request B
Tenant B
        │
        ▼
Same Resource
```

If the resource is tenant-scoped, authorization must determine whether each request is permitted before synchronization occurs.

Do not use a global lock as a replacement for authorization.

---

# 65. Concurrency and Authorization

Locks should generally operate on the same resource scope as the authorization decision.

Example:

```text
Tenant
 ↓
Resource
 ↓
Authorization
 ↓
Concurrency control
 ↓
Mutation
```

This avoids protecting unauthorized operations unnecessarily and reduces leakage risk.

---

# 66. Avoid Holding Locks During Authorization Calls

Do not hold a database or distributed lock while making unrelated remote authorization calls.

Prefer:

```text
Validate identity
 ↓
Authorize
 ↓
Acquire required synchronization
 ↓
Validate critical state
 ↓
Mutate
```

The final state validation should still happen inside the protected operation when necessary.

---

# 67. Deadlock Prevention Rules

KAMPYN must:

1. Keep transactions short.
2. Acquire locks in deterministic order.
3. Avoid unnecessary nested locks.
4. Avoid holding locks during external calls.
5. Keep critical sections small.
6. Avoid global locks for local resources.
7. Avoid distributed locks where database primitives are sufficient.
8. Retry recoverable deadlocks with bounded attempts.
9. Monitor database deadlocks.
10. Document unusual locking strategies.

---

# 68. Concurrency and API Timeouts

Concurrency can amplify slow requests.

Example:

```text
100 concurrent requests
 ×
10-second dependency latency
 =
large resource occupation
```

Every external operation should therefore have a bounded timeout.

Timeouts protect:

- Goroutines
- Connections
- Worker slots
- Database pools
- Memory

---

# 69. Circuit Breaking

If an external dependency repeatedly fails:

```text
KAMPYN
   ↓
External Provider
   X
Repeated failure
```

continued concurrent requests may amplify the outage.

Use circuit-breaking or equivalent failure isolation where appropriate.

The system should fail fast rather than consume all available worker capacity.

---

# 70. Backpressure

When a downstream system cannot keep up:

```text
Producer
   ↓
Queue
   ↓
Consumer
   ↓
Slow dependency
```

do not increase concurrency without limit.

Instead:

```text
Bound concurrency
+
Queue
+
Retry/backoff
+
Rate limit
```

This protects the system from cascading failure.

---

# 71. Concurrency During Deployment

During rolling deployment:

```text
Old API
New API
Old Worker
New Worker
```

may execute concurrently.

Therefore:

- Database migrations must be compatible
- Job payloads must remain compatible
- Events must remain compatible
- API contracts must remain compatible
- State transitions must remain valid

Never assume all instances switch versions simultaneously.

---

# 72. Graceful Shutdown

Workers and servers must handle termination signals.

Shutdown should:

```text
1. Stop accepting new work
2. Signal cancellation
3. Finish safe in-flight operations
4. Release resources
5. Close connections
6. Exit
```

Do not terminate immediately while holding critical resources.

---

# 73. Testing Concurrency

Concurrency must be tested explicitly.

Tests should include:

- Race detection
- Concurrent requests
- Concurrent workers
- Duplicate execution
- Deadlocks
- Timeouts
- Cancellation
- Database conflicts
- Lock contention
- Retry behavior
- Queue concurrency
- External API limits

---

# 74. Go Race Detector

Go code involving shared memory should be tested with the race detector where appropriate.

```bash
go test -race ./...
```

A race detector failure is a correctness problem.

Do not ignore race detector warnings merely because the race is difficult to reproduce.

---

# 75. Stress Testing

Concurrency bugs may appear only under load.

Test scenarios such as:

```text
100 users
1,000 users
10,000 concurrent requests
```

where appropriate.

Focus especially on high-contention operations:

```text
Inventory
Bookings
Payments
Seat allocation
Limited resources
Rate limits
```

---

# 76. Deterministic Testing

Avoid tests that rely on:

```text
time.Sleep(...)
```

to "wait" for concurrent work.

Prefer:

- Channels
- Wait groups
- Explicit synchronization
- Test hooks
- Context cancellation
- Deterministic coordination

Tests should wait for a known condition rather than guessing timing.

---

# 77. Load Testing

Load tests should identify:

```text
Maximum safe concurrency
Database saturation point
Queue throughput
External API limits
Memory growth
Goroutine growth
Lock contention
```

Do not optimize solely for requests per second.

Measure correctness under load.

---

# 78. Observability

Concurrency-related metrics should include:

```text
Active goroutines
Worker utilization
Queue depth
Queue age
Request concurrency
Database pool utilization
Lock contention
Transaction duration
Retry count
Timeout count
External API concurrency
```

Do not create high-cardinality metrics from arbitrary IDs.

---

# 79. Profiling

When concurrency performance is poor, investigate before changing synchronization.

Useful areas include:

```text
CPU
Memory
Goroutines
Mutex contention
Blocking
Database waits
Network latency
```

Do not replace correct synchronization with unsafe lock removal simply for performance.

---

# 80. Common Anti-Patterns

Avoid:

```text
1. Unbounded goroutines
2. Global mutable state
3. In-memory locks for distributed correctness
4. Sleep-based synchronization
5. Read-then-write races
6. Long database transactions
7. Locks around network calls
8. Infinite retries
9. Unlimited worker concurrency
10. Ignoring context cancellation
11. Assuming exactly-once execution
12. Assuming event ordering
13. Using Redis for every concurrency problem
14. Using distributed locks unnecessarily
15. Using database locks without deterministic ordering
16. Storing request state globally
17. Increasing concurrency without measuring
18. Treating race detector failures as harmless
```

---

# 81. Recommended Decision Process

When a concurrency problem appears, ask:

```text
1. What state is being concurrently modified?

2. Is the state authoritative or derived?

3. Can the operation be immutable?

4. Can a database constraint enforce the invariant?

5. Can an atomic update solve it?

6. Does the operation require a transaction?

7. Is optimistic concurrency sufficient?

8. Is pessimistic locking required?

9. Is coordination distributed?

10. If distributed, is a distributed lock actually necessary?

11. What happens if the operation executes twice?

12. What happens if the worker crashes?

13. What happens if the dependency is slow?

14. What happens if two tenants compete for the same resource?

15. What is the maximum safe concurrency?

16. How will the behavior be observed and tested?
```

---

# 82. Example: Inventory Reservation

Unsafe:

```text
Request A
 ↓
Read inventory = 1

Request B
 ↓
Read inventory = 1

A → inventory = 0
B → inventory = 0
```

Correct conceptual flow:

```text
Request
 ↓
Transaction
 ↓
Atomic conditional inventory update
 ↓
Affected rows?
 ├── 1 → reservation succeeds
 └── 0 → insufficient inventory
 ↓
Commit
```

This avoids the check-then-act race.

---

# 83. Example: Booking Slot

```text
Two users
    │
    ▼
Same slot
    │
    ▼
Database transaction
    │
    ├── Validate capacity
    ├── Create reservation
    └── Enforce uniqueness
            │
            ▼
      One succeeds
      Other conflicts
```

The database remains authoritative.

---

# 84. Example: Concurrent Payment Webhooks

```text
Webhook A ─┐
           │
Webhook B ─┼── same payment
           │
Webhook C ─┘
             ↓
        Idempotency
             ↓
      State validation
             ↓
        Transaction
             ↓
       One valid state
```

Repeated delivery must not create duplicate financial effects.

---

# 85. Example: Background Processing

```text
100,000 jobs
      │
      ▼
Queue
      │
      ▼
Bounded worker pool
      │
      ├── Worker 1
      ├── Worker 2
      ├── Worker 3
      └── Worker N
```

Worker count must be chosen according to:

```text
CPU
+
Memory
+
Database capacity
+
External API limits
+
Job characteristics
```

---

# 86. Example: Large File Processing

```text
Large File
    │
    ▼
Streaming Reader
    │
    ▼
Bounded Chunks
    │
    ▼
Worker Pool
    │
    ▼
Aggregated Result
```

Never create one goroutine per row.

Never load the entire file into memory merely to parallelize processing.

---

# 87. Definition of Done

Concurrency-sensitive functionality is complete only when:

- Concurrent access patterns are identified.
- The authoritative state is identified.
- Domain invariants are documented.
- The synchronization mechanism is explicit.
- Local versus distributed concurrency is distinguished.
- Database constraints are used where appropriate.
- Transactions are correctly bounded.
- Lock scope is minimal.
- Lock ordering is deterministic where multiple locks exist.
- Goroutines have bounded lifetimes.
- Context cancellation is supported.
- Worker concurrency is bounded.
- External calls have timeouts.
- Retry behavior is bounded.
- Idempotency is addressed.
- Duplicate execution is safe.
- Tenant isolation is preserved.
- Deployment-version concurrency is considered.
- Race detection is performed where relevant.
- Concurrent behavior is tested.
- Observability exists for meaningful contention/failure modes.
- Performance is measured rather than assumed.

---

# 88. Final Invariants

The following rules are mandatory:

```text
1. Concurrency is a correctness concern before it is a performance concern.

2. Never assume KAMPYN has only one backend instance.

3. In-memory synchronization cannot provide distributed correctness.

4. Persistent business invariants should normally be enforced by
   the authoritative data store.

5. Prefer database constraints and atomic operations over
   application-level synchronization when appropriate.

6. Every goroutine must have a bounded and observable lifetime.

7. Every request-scoped goroutine must respect context cancellation.

8. Worker concurrency must always be bounded.

9. Database concurrency must respect connection-pool capacity.

10. Transactions must be short and must not wait on slow external
    operations.

11. Locks must be narrow and acquired in deterministic order.

12. Distributed locks must be used only when simpler mechanisms
    cannot guarantee correctness.

13. Background jobs and events must assume duplicate execution
    unless exactly-once behavior is explicitly guaranteed.

14. High-contention domains such as inventory, bookings, and
    payments require explicit concurrency strategies.

15. State transitions must be validated atomically where necessary.

16. External API concurrency must respect provider limits.

17. Backpressure is preferable to unlimited concurrency.

18. A performance optimization must never weaken a domain invariant.

19. Race detector failures must be treated as correctness failures.

20. Concurrency behavior must be tested under both normal and
    high-contention conditions.

21. Tenant isolation must remain correct under concurrent execution.

22. Deployment, retries, worker crashes, and duplicate delivery
    must all be considered part of the concurrency model.

23. If the correctness of an operation depends on "this will probably
    execute only once", the concurrency design is incomplete.
```

## Summary

KAMPYN concurrency should follow:

```text
                    Concurrent Work
                          │
          ┌───────────────┼────────────────┐
          │               │                │
       Requests         Jobs             Events
          │               │                │
          └───────────────┼────────────────┘
                          ▼
                  Application Layer
                          │
                          ▼
              Explicit Concurrency Model
                          │
          ┌───────────────┼────────────────┐
          │               │                │
       Atomic Ops      Transactions    Coordination
          │               │                │
          └───────────────┼────────────────┘
                          ▼
                 Authoritative State
                          │
                          ▼
                    Consistent Data
```

The goal is not to eliminate concurrency.

The goal is to make concurrent execution **bounded, deterministic, observable, recoverable, and incapable of violating KAMPYN's domain invariants**.
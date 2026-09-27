# KAMPYN Backend Repository Pattern

## Purpose

Repositories provide the persistence boundary between KAMPYN's application/domain logic and infrastructure-specific data stores.

The repository layer exists to prevent business logic from becoming coupled to:

- PostgreSQL drivers
- MongoDB drivers
- Redis clients
- OpenSearch clients
- ORM implementations
- Query builders
- Database schemas
- Provider-specific persistence details

The architectural boundary is:

```text
Transport
    ↓
Application
    ↓
Domain
    ↓
Repository Contract
    ↓
Repository Implementation
    ↓
Database / Storage
```

The repository pattern must be used deliberately.

A repository is not a generic wrapper around a database client and must not become a dumping ground for business logic.

---

# 1. Core Principles

Repositories must:

- Encapsulate persistence concerns.
- Expose domain/application-relevant operations.
- Keep database implementation details outside business logic.
- Preserve transaction boundaries.
- Respect tenant isolation.
- Handle persistence-specific errors.
- Support efficient queries.
- Avoid unnecessary abstractions.
- Remain cohesive around a meaningful aggregate or persistence boundary.

Repositories must not:

- Contain HTTP logic.
- Perform authorization decisions.
- Implement unrelated business workflows.
- Call controllers.
- Own application orchestration.
- Hide expensive queries behind innocent method names.
- Become generic `CRUD` utilities.

---

# 2. Repository Position

The preferred dependency structure is:

```text
┌─────────────────────────────┐
│        Transport            │
│ HTTP / WebSocket / Events   │
└──────────────┬──────────────┘
               ↓
┌─────────────────────────────┐
│       Application           │
│      Use Cases              │
└──────────────┬──────────────┘
               ↓
┌─────────────────────────────┐
│          Domain             │
│ Entities / Rules / Values   │
└──────────────┬──────────────┘
               ↓
┌─────────────────────────────┐
│    Repository Contract      │
└──────────────┬──────────────┘
               ↓
┌─────────────────────────────┐
│ Repository Implementation   │
│ PostgreSQL / Mongo / etc.   │
└──────────────┬──────────────┘
               ↓
┌─────────────────────────────┐
│       Data Store            │
└─────────────────────────────┘
```

The direction of dependency must not be reversed.

---

# 3. Repository Responsibility

A repository answers questions such as:

```text
Can this entity be loaded?
Can this entity be persisted?
Which records match this persistence query?
Can this aggregate be updated?
Does this record exist?
```

It should not answer:

```text
Should the user be allowed to perform this action?
Should an order be cancelled?
Should a booking be approved?
Should a refund be issued?
```

Those belong to application/domain logic.

---

# 4. Repository vs Service

Repositories persist and retrieve.

Application services coordinate.

Example:

```text
CreateOrderUseCase
    ↓
Load Menu
    ↓
Load Inventory
    ↓
Domain Validation
    ↓
Reserve Inventory
    ↓
Persist Order
    ↓
Publish Event
```

Repositories may perform:

```text
menuRepository.getById()
inventoryRepository.reserve()
orderRepository.create()
```

The use case owns the workflow.

---

# 5. Repository vs Domain

The domain owns business rules.

For example:

```text
Order
 ├── addItem()
 ├── calculateTotal()
 ├── transitionState()
 └── canCancel()
```

The repository owns persistence:

```text
OrderRepository
 ├── findById()
 ├── save()
 └── delete()
```

Do not place persistence logic inside domain entities.

---

# 6. Repository Boundaries

Repositories should generally be organized around meaningful domain ownership.

Examples:

```text
OrderRepository
InventoryRepository
MenuRepository
FoodCourtRepository
BookingRepository
HostelRepository
UserRepository
TenantRepository
NotificationRepository
```

Avoid:

```text
DatabaseRepository
GenericRepository
UniversalRepository
CommonRepository
```

unless there is a specific architectural reason.

---

# 7. Aggregate-Oriented Repositories

When domain aggregates are explicitly defined, repositories should generally operate around aggregate boundaries.

Example:

```text
Order
 ├── OrderItem
 ├── Pricing
 └── OrderState
```

An `OrderRepository` may load and persist the complete order aggregate when the use case requires it.

Do not split an aggregate across unrelated repositories merely because its tables are separate.

---

# 8. Persistence Model vs Domain Model

Database models do not necessarily have to equal domain models.

Example:

```text
PostgreSQL Row
      ↓
Persistence Model
      ↓
Mapper
      ↓
Domain Entity
```

This prevents database-specific concerns from leaking into domain logic.

---

# 9. When Models May Be Shared

For simple CRUD-like bounded contexts, a separate persistence/domain model may introduce unnecessary duplication.

Sharing may be acceptable when:

- The model has minimal business behavior.
- Persistence structure closely matches domain structure.
- No infrastructure concerns leak into the model.
- The coupling is intentional.

Do not create mappings solely for architectural appearance.

---

# 10. Repository Contracts

Repository contracts should expose operations meaningful to their consumers.

Example:

```go
type OrderRepository interface {
    GetByID(ctx context.Context, id OrderID) (*Order, error)
    Save(ctx context.Context, order *Order) error
}
```

Avoid interfaces such as:

```go
type Repository interface {
    Create(...)
    Read(...)
    Update(...)
    Delete(...)
    Query(...)
}
```

A generic CRUD interface usually hides domain intent and encourages weak boundaries.

---

# 11. Go Interface Ownership

In Go, repository interfaces should generally be defined by the consumer that needs them.

For example:

```text
Application
    ↓
OrderRepository interface
    ↓
PostgresOrderRepository
```

The concrete repository should satisfy the interface.

Do not create interfaces merely because a concrete type exists.

---

# 12. TypeScript Repository Contracts

For Node.js/TypeScript services, repository contracts may use interfaces or equivalent abstractions.

Example:

```ts
interface UserRepository {
  findById(id: UserId): Promise<User | null>;
  save(user: User): Promise<void>;
}
```

The same principles apply:

- Intent-driven methods.
- Explicit types.
- No infrastructure leakage.
- No generic database wrapper.

---

# 13. Repository Naming

Repository methods should describe intent.

Preferred:

```text
FindByID
FindActiveByTenant
ListAvailableRooms
FindPendingOrders
Save
Delete
```

Avoid ambiguous methods:

```text
Get
Process
Handle
Execute
DoSomething
Query
```

Method names should communicate the persistence operation.

---

# 14. `Get` vs `Find`

Use consistent repository semantics.

A useful convention is:

```text
GetByID
```

when absence is exceptional or represented as an explicit not-found error.

```text
FindByID
```

when absence is an expected result.

The repository contract must document its chosen semantics.

---

# 15. Not Found

Not-found behavior must be explicit.

Possible contract:

```text
record exists
    ↓
entity returned

record does not exist
    ↓
nil / null
```

or:

```text
record does not exist
    ↓
ErrNotFound
```

Do not allow every repository implementation to invent different semantics.

---

# 16. Repository Errors

Persistence-specific errors should be translated into stable application-level errors where appropriate.

Example:

```text
PostgreSQL unique violation
        ↓
Repository
        ↓
ErrConflict
        ↓
Application
        ↓
HTTP 409
```

Do not expose:

```text
pq error
Mongo driver error
Redis error
```

directly to API clients.

---

# 17. Error Identity

Errors should remain inspectable.

In Go:

```go
errors.Is(err, ErrNotFound)
```

or:

```go
errors.Is(err, ErrConflict)
```

is preferable to string comparison.

Avoid:

```go
if err.Error() == "record not found" {
    ...
}
```

---

# 18. Database-Specific Errors

Database-specific errors belong inside infrastructure.

Example:

```text
PostgreSQL duplicate key
Mongo duplicate key
Redis timeout
```

should be translated at the repository/infrastructure boundary.

Application logic should not need to understand driver-specific error codes.

---

# 19. PostgreSQL Repositories

PostgreSQL repositories must use:

- Parameterized queries.
- Appropriate indexes.
- Explicit transactions.
- Connection pooling.
- Bounded queries.
- Stable pagination.
- Appropriate locking.
- Constraint enforcement.

Never construct SQL using untrusted string concatenation.

---

# 20. PostgreSQL Query Ownership

Queries should live close to the repository responsible for the data.

Avoid scattering SQL across:

```text
handlers
controllers
services
domain objects
utility packages
```

A repository should make the query strategy explicit.

---

# 21. MongoDB Repositories

MongoDB repositories must:

- Define explicit collection ownership.
- Use appropriate indexes.
- Bound queries.
- Avoid unbounded document growth.
- Handle consistency requirements explicitly.
- Use projections when useful.
- Handle duplicate and version conflicts deliberately.

MongoDB does not eliminate the need for domain boundaries.

---

# 22. Redis Repositories

Redis requires special treatment.

Redis is primarily infrastructure for:

```text
Caching
Rate limiting
Transient state
Coordination
Queues
Short-lived data
```

A Redis repository must not pretend cached data is authoritative domain state.

For cache-backed reads:

```text
Application
    ↓
Cache
    ↓ miss
Repository / Source of Truth
```

---

# 23. OpenSearch Repositories

OpenSearch repositories represent search/read-model access.

They must not become the source of truth for transactional domain state.

Preferred:

```text
Domain Database
      ↓
Outbox / Event
      ↓
Search Index
      ↓
Search Repository
```

Search repositories may expose:

```text
SearchProducts
SearchFoodCourts
SearchVendors
SearchCommunityContent
```

but must remain separate from transactional repositories where their responsibilities differ.

---

# 24. Repository and Object Storage

Object storage should also have an explicit boundary.

Example:

```text
FileRepository
ObjectStorage
```

The repository should not expose provider-specific types throughout the application.

Instead expose application-relevant metadata:

```text
ObjectKey
ContentType
Size
Checksum
URL / signed URL
```

where appropriate.

---

# 25. Repository Transactions

Repositories must respect transaction boundaries established by the application.

A repository should not unexpectedly start and commit a transaction around every method if a larger application operation requires atomicity.

Bad:

```text
CreateOrder()
 └── transaction
ReserveInventory()
 └── transaction
CreatePayment()
 └── transaction
```

when these operations must participate in one defined transactional boundary.

---

# 26. Transaction-Aware Repositories

When necessary, repositories should support an explicit transaction context.

Conceptually:

```text
Application Use Case
        ↓
Transaction
   ┌────┴────┐
   ↓         ↓
Order Repo  Inventory Repo
   └────┬────┘
        ↓
      Commit
```

The exact transaction mechanism is infrastructure-specific.

---

# 27. Transactions Must Stay Short

Do not keep database transactions open while performing:

```text
HTTP calls
Payment provider calls
Email delivery
File uploads
Long computations
User interaction
```

Prefer durable coordination through events/jobs.

---

# 28. Repository Concurrency

Repositories must account for concurrent access.

Examples:

```text
Two users booking the same room
Two users buying the last inventory item
Two workers processing the same order
Two administrators editing the same resource
```

Use appropriate:

- Database constraints
- Transactions
- Row/document locks
- Optimistic concurrency
- Atomic updates
- Version fields

---

# 29. Check-Then-Act

Avoid:

```text
SELECT available
        ↓
application checks
        ↓
UPDATE
```

when concurrent requests can change the state between operations.

Prefer atomic database operations or appropriate locking.

---

# 30. Inventory

Inventory repositories are particularly sensitive to concurrency.

A safe operation may be:

```text
UPDATE inventory
SET quantity = quantity - requested
WHERE item_id = ?
  AND quantity >= requested
```

The application must inspect affected rows and treat zero affected rows as a failed reservation.

The exact implementation depends on the data model.

---

# 31. Booking

Booking repositories must enforce uniqueness and conflict prevention at the persistence boundary where possible.

Do not rely only on:

```text
application-level availability checks
```

when concurrent bookings are possible.

Database constraints or transactional locking should provide the final protection.

---

# 32. Idempotency

Repositories may participate in idempotent operations, but application-level idempotency ownership must remain explicit.

For example:

```text
Idempotency Key
      ↓
Application
      ↓
Persistence
```

A repository may persist an idempotency record but should not silently invent business semantics.

---

# 33. Pagination

Repositories must support bounded pagination for large datasets.

Avoid:

```text
SELECT * FROM orders;
```

for potentially large collections.

Preferred:

```text
Cursor
  ↓
Bounded query
  ↓
Page
```

Cursor pagination should use stable ordering.

---

# 34. Offset Pagination

Offset pagination may be acceptable for small or stable datasets.

It becomes increasingly expensive for large offsets.

For large datasets, prefer cursor/keyset pagination.

---

# 35. Sorting

Repository methods must define supported sort fields explicitly.

Never allow arbitrary user input to become raw SQL or database expressions.

Use allowlists:

```text
createdAt
name
price
```

rather than arbitrary strings.

---

# 36. Filtering

Filters should be represented using typed structures.

Example:

```go
type OrderFilter struct {
    TenantID  TenantID
    Status    *OrderStatus
    CreatedAt *TimeRange
}
```

Avoid passing arbitrary SQL fragments or Mongo expressions from application code.

---

# 37. Field Selection

If field selection is supported, it must be explicit.

Avoid allowing callers to request arbitrary database fields that may expose:

- Internal metadata
- Secrets
- Sensitive data
- Security fields

---

# 38. N+1 Queries

Repositories must avoid accidental N+1 queries.

Bad:

```text
Get 100 orders
   ↓
100 customer queries
```

Prefer:

```text
Get 100 orders
   ↓
Batch related records
```

or use an appropriate query strategy.

---

# 39. Batch Operations

Repositories should provide batch operations when they materially improve performance.

Examples:

```text
GetByIDs
BulkCreate
BulkUpdate
BatchDelete
```

But batch methods must still respect:

- Transaction semantics
- Validation
- Limits
- Error behavior
- Memory constraints

---

# 40. Batch Size

Do not create unlimited database batches.

Use bounded batch sizes appropriate to:

- Database limits
- Query size
- Memory
- Lock duration
- Network limits

---

# 41. Query Performance

Every repository query should have a known performance profile.

Review:

```text
Indexes
Query plan
Rows scanned
Rows returned
Join behavior
Sort behavior
Pagination
Network transfer
```

Do not optimize based solely on intuition.

---

# 42. Indexes

Indexes should exist for actual query patterns.

An index should have:

- Clear query justification.
- Appropriate column order.
- Known write overhead.
- Migration ownership.

Do not create indexes for every field.

---

# 43. Repository and Domain Events

Repositories should not secretly publish events as a side effect unless the architecture explicitly defines that behavior.

Prefer:

```text
Application
 ↓
Transaction
 ├── Repository mutation
 └── Outbox event
```

This keeps event semantics visible.

---

# 44. Transactional Outbox

When persistence and event publication must be reliable:

```text
BEGIN
 ↓
Repository mutation
 ↓
Outbox insert
 ↓
COMMIT
 ↓
Publisher
 ↓
Consumer
```

The repository may participate in writing the outbox record through the transaction boundary, but event ownership remains explicit.

---

# 45. Repository and Caching

Repositories should not hide arbitrary caching behavior.

Bad:

```text
repository.GetUser()
    ↓
sometimes DB
sometimes Redis
sometimes stale data
```

without clear semantics.

Prefer explicit cache architecture:

```text
Application
 ↓
Cache Boundary
 ↓
Repository
```

or an explicitly documented repository/cache composition.

---

# 46. Cache Invalidation

If repository mutations require cache invalidation, the consistency model must be explicit.

Potential flow:

```text
Transaction
 ↓
Database
 ↓
Event
 ↓
Cache Invalidation
```

Do not assume cache invalidation is automatically atomic with database mutation.

---

# 47. Repository and Search

Search indexes should generally be updated asynchronously.

Preferred:

```text
Repository mutation
      ↓
Outbox
      ↓
Event
      ↓
Search Consumer
      ↓
OpenSearch
```

Do not make every transactional repository write synchronously to OpenSearch unless the use case genuinely requires that consistency.

---

# 48. Repository Read Models

A read model may use a specialized repository.

Example:

```text
OrderRepository
    → transactional order state

OrderSearchRepository
    → OpenSearch read model
```

Do not force a single repository to hide fundamentally different storage semantics.

---

# 49. Repository Interfaces Should Be Small

Avoid giant interfaces.

Bad:

```text
UserRepository
 ├── create
 ├── update
 ├── delete
 ├── search
 ├── export
 ├── statistics
 ├── notifications
 ├── permissions
 └── audit
```

These are different responsibilities.

Split by meaningful ownership.

---

# 50. Repository Cohesion

A repository should answer:

> What persistence boundary does this repository own?

If that answer requires multiple unrelated domains, the repository is probably too broad.

---

# 51. Generic Repository Pattern

Generic repositories should generally be avoided for domain persistence.

For example:

```go
type Repository[T any] interface {
    Create(...)
    Get(...)
    Update(...)
    Delete(...)
}
```

may appear reusable but often loses domain semantics.

Prefer explicit repositories.

---

# 52. Utility Database Functions

Small shared database utilities are acceptable for genuinely cross-cutting infrastructure:

```text
Transaction helpers
Pagination utilities
Query instrumentation
Connection configuration
Retry helpers
```

They must not contain domain behavior.

---

# 53. Repository Dependency Injection

Repositories should be injected into application services.

Example:

```go
type OrderService struct {
    orders OrderRepository
}
```

Construction:

```text
Application
   ↓
Dependencies
   ├── OrderRepository
   ├── InventoryRepository
   └── PaymentProvider
```

Avoid hidden global repositories.

---

# 54. Repository Lifecycle

Infrastructure clients should normally be created during application startup.

Example:

```text
Startup
 ↓
Database Pool
 ↓
Repository
 ↓
Application Service
 ↓
HTTP Server
```

Do not repeatedly initialize clients per request.

---

# 55. Connection Pools

Repository implementations must respect connection pool limits.

Do not create:

```text
one DB connection per request
```

or:

```text
one DB pool per repository instance
```

unless explicitly required.

Prefer shared, centrally managed pools.

---

# 56. Context and Cancellation

Go repository methods must accept `context.Context`.

Example:

```go
func (r *OrderRepository) GetByID(
    ctx context.Context,
    id OrderID,
) (*Order, error)
```

Database operations must respect request cancellation and deadlines.

---

# 57. Node.js Repository Cancellation

Node.js repository operations should support appropriate cancellation mechanisms such as `AbortSignal` when supported by the underlying client.

Long-running operations must not continue unnecessarily after request cancellation.

---

# 58. Timeouts

Repository operations must not wait indefinitely.

Timeouts may exist at:

```text
HTTP
Application
Database client
External provider
```

They should complement one another rather than conflict.

---

# 59. Repository Logging

Repositories should not log every successful database operation by default.

Prefer metrics and tracing for normal operations.

Log meaningful failures with:

```text
operation
requestId
traceId
tenantId
duration
error classification
```

without sensitive values.

---

# 60. Repository Metrics

Useful metrics include:

```text
repository_operation_duration
repository_operation_errors
database_query_duration
database_connection_pool_usage
database_query_timeout
```

Metrics must avoid high-cardinality labels.

---

# 61. Tracing

Database calls should participate in distributed tracing.

A trace should allow:

```text
HTTP request
 ↓
Application use case
 ↓
Repository
 ↓
Database
```

to be understood as one execution path.

---

# 62. Testing Repository Implementations

Repository tests should verify:

- Correct persistence.
- Mapping.
- Constraints.
- Transactions.
- Error translation.
- Pagination.
- Filtering.
- Concurrency.
- Tenant isolation.
- Soft deletion where applicable.
- Index-dependent behavior where important.

---

# 63. Repository Unit Tests

Pure repository logic such as:

```text
mapping
query construction
error classification
pagination transformation
```

may be unit tested.

Do not mock the database so heavily that repository tests never verify actual persistence behavior.

---

# 64. Repository Integration Tests

Real database integration tests should verify important repository behavior.

Examples:

```text
PostgreSQL container
MongoDB container
Redis test instance
OpenSearch test instance
```

Use the repository's actual driver/ORM behavior.

---

# 65. Transaction Tests

Test:

```text
successful transaction
rollback
constraint violation
concurrent modification
deadlock handling
```

where applicable.

---

# 66. Tenant Isolation Tests

Repositories must have tests ensuring that tenant-scoped queries cannot return another tenant's records.

Example:

```text
Tenant A
 ├── Order A

Tenant B
 ├── Order B
```

A query executed with Tenant A context must never return Order B.

---

# 67. Authorization Boundary

Repositories generally should not perform authorization decisions.

For example, this is usually wrong:

```text
repository.GetOrderForUser()
```

if it mixes:

```text
persistence
+
business authorization
```

Prefer:

```text
Application
 ↓
Authorization
 ↓
Repository
```

However, tenant/resource scoping required to safely execute the persistence query must still be enforced.

---

# 68. Defense in Depth

Although authorization belongs above repositories, repositories must not provide APIs that accidentally enable unsafe cross-tenant access.

For example:

```text
FindOrderByID(id)
```

may be dangerous in a tenant-scoped application if tenant scope is not part of the trusted context.

Prefer:

```text
FindOrderByID(ctx, tenantID, orderID)
```

when tenant ownership is mandatory.

---

# 69. Soft Deletes

If KAMPYN uses soft deletion:

```text
deleted_at
```

must be handled consistently.

Repositories should normally exclude deleted records unless explicitly requested by an administrative/recovery operation.

Do not let individual callers forget deletion filters.

---

# 70. Audit Data

Audit records should have their own persistence boundary when appropriate.

Avoid mixing:

```text
OrderRepository
```

with:

```text
AuditRepository
```

unless the audit record is explicitly part of the transaction boundary.

---

# 71. Sensitive Data

Repositories handling sensitive data must enforce:

- Minimal selection.
- Access boundaries.
- Encryption where required.
- Safe logging.
- Retention rules.
- Deletion requirements.

Do not load sensitive fields when they are not required.

---

# 72. Data Ownership

Every repository must have a clear data owner.

Example:

```text
OrderRepository
    → Orders

InventoryRepository
    → Inventory

BookingRepository
    → Bookings

UserRepository
    → Users
```

A repository should not mutate another domain's authoritative data directly.

---

# 73. Cross-Domain Queries

Cross-domain reads may be necessary.

Preferred approaches include:

```text
Application orchestration
Read model
Dedicated query service
Search index
Database view
Explicit reporting repository
```

Do not silently turn every repository into a cross-domain query engine.

---

# 74. Reporting Queries

Heavy reporting queries should not unnecessarily execute against transactional request paths.

Consider:

```text
Read model
Analytics store
Materialized view
Background report generation
```

when workload requires it.

---

# 75. Large Exports

Repositories supporting exports must use bounded processing.

Preferred:

```text
Database Cursor
 ↓
Chunk
 ↓
Transform
 ↓
Object Storage
```

Avoid loading millions of rows into memory.

---

# 76. Streaming

Repository implementations should support streaming/cursor-based access where large datasets require it.

Streaming must define:

- Resource cleanup.
- Cancellation.
- Error handling.
- Transaction lifetime.
- Connection lifetime.

---

# 77. Deletion

Deletion methods must define semantics clearly:

```text
Hard delete
Soft delete
Archive
Deactivate
```

Do not call all of these `Delete` if their business meaning differs.

---

# 78. Bulk Deletion

Bulk deletion must be:

- Explicit
- Authorized at the application boundary
- Bounded
- Observable
- Transactionally understood
- Safe against accidental broad filters

Never allow an empty or malformed filter to accidentally delete the entire dataset.

---

# 79. Migrations

Repository changes that require schema changes must include the corresponding migration.

The implementation is incomplete if:

```text
Repository code
```

changes but:

```text
Database schema
```

does not.

---

# 80. Backfills

Large backfills should not run inside application startup or request paths.

Use:

```text
Background job
Migration job
Controlled script
```

with:

- Bounded batches
- Progress tracking
- Retry behavior
- Observability
- Recovery strategy

---

# 81. Repository Compatibility

During schema migrations, repository code may temporarily need to support:

```text
old schema
+
new schema
```

when zero-downtime deployment requires it.

The compatibility period must be explicit and removed after migration completion.

---

# 82. Repository Versioning

Repositories themselves normally do not require API-style versioning.

However, externally visible behavior must remain compatible when consumers depend on it.

If a repository contract changes:

```text
Search consumers
 ↓
Update contract
 ↓
Update implementation
 ↓
Update tests
```

must remain synchronized.

---

# 83. Self-Hosted Deployments

Repositories must work with the supported KAMPYN deployment model.

Do not hardcode:

```text
localhost
cloud provider endpoints
specific tenant
specific database name
```

into repository implementations.

Configuration must come from the service configuration layer.

---

# 84. Repository Failures

Repositories must fail predictably.

Potential failures include:

```text
Database unavailable
Connection timeout
Deadlock
Constraint violation
Serialization error
Network failure
Authentication failure
```

The application must distinguish:

```text
retryable
non-retryable
client-caused
server-caused
```

where meaningful.

---

# 85. Retry Policy

Do not blindly retry every repository operation.

Safe retrying depends on:

```text
operation idempotency
transaction state
database semantics
error type
timeout behavior
```

A timed-out write may have succeeded.

Never assume:

```text
timeout = operation did not happen
```

---

# 86. Deadlocks

Database deadlocks may be retryable.

If retrying:

```text
bounded attempts
exponential backoff
jitter
```

must be used.

The retry policy should exist at an appropriate infrastructure/application boundary.

---

# 87. Repository and Idempotent Writes

For operations requiring idempotency, prefer database constraints where possible.

Example:

```text
unique(tenant_id, idempotency_key)
```

This is stronger than relying only on an in-memory check.

---

# 88. Repository Security

Repository code must defend against:

- SQL injection
- NoSQL injection
- Unauthorized tenant access
- Excessive data exposure
- Unsafe dynamic queries
- Path traversal for storage
- Credential leakage

Parameterized queries and typed query builders should be preferred.

---

# 89. Dynamic Queries

Dynamic queries must use allowlists.

Do not directly interpolate:

```text
sort
filter
column
collection
field
```

from user input.

Translate external values into controlled internal query representations.

---

# 90. Repository API Stability

Repository methods are internal contracts, but they still require discipline.

Avoid frequent breaking changes caused by:

```text
minor implementation detail
ORM migration
database driver replacement
```

The application layer should remain insulated from these changes.

---

# 91. ORM Usage

An ORM may be used where approved.

However:

```text
ORM ≠ architecture
```

The repository remains responsible for:

- Query intent
- Transaction behavior
- Performance
- Mapping
- Error semantics

Do not expose ORM models throughout the application merely because the ORM makes it convenient.

---

# 92. Raw SQL

Raw SQL is acceptable when it provides:

- Better performance
- Required database functionality
- Complex queries
- Explicit control

It must still follow:

- Parameterization
- Review
- Testing
- Index considerations
- Migration ownership

Do not avoid raw SQL purely for stylistic reasons.

---

# 93. Query Builders

Query builders are acceptable when they improve:

- Safety
- Maintainability
- Dynamic query composition
- Portability where required

They must not produce opaque queries that developers cannot reason about.

---

# 94. Repository Mapping

Mappings should be explicit where domain and persistence structures differ.

Example:

```text
DB Row
 ↓
toDomainOrder()
 ↓
Order
```

and:

```text
Order
 ↓
toPersistenceOrder()
 ↓
DB Row
```

Mappings should remain deterministic and testable.

---

# 95. IDs

Repositories must preserve KAMPYN's ID strategy.

Do not silently convert identifiers between:

```text
UUID
integer
string
ObjectID
```

without an explicit boundary.

IDs should remain type-safe where practical.

---

# 96. Time

Persist timestamps consistently.

Repositories should not silently reinterpret time zones.

Prefer:

```text
UTC storage
explicit timezone conversion at presentation boundaries
```

unless the data model explicitly requires another representation.

---

# 97. Money

Do not represent monetary values using floating-point arithmetic.

Repositories should preserve the exact representation used by the domain.

For example:

```text
minor currency units
```

may be preferable.

Database column types must match the domain's precision requirements.

---

# 98. Quantities

Inventory quantities and similar values must use appropriate numeric representations.

Do not use floating-point quantities where exact arithmetic is required.

---

# 99. Repository and Search Consistency

Transactional repositories own authoritative state.

Search repositories own derived search state.

Therefore:

```text
PostgreSQL / MongoDB
        ↓
Authoritative state
        ↓
Event / Outbox
        ↓
OpenSearch
        ↓
Derived state
```

Search inconsistency should be treated as an eventual-consistency problem, not solved by making search the transactional source of truth.

---

# 100. Repository and Cache Consistency

Similarly:

```text
Database
   ↓
Source of Truth

Redis
   ↓
Optimization
```

A repository must not make the system dependent on Redis availability unless Redis is explicitly part of the authoritative workflow.

---

# 101. Repository Documentation

Every non-trivial repository should document:

- Owned data.
- Contract.
- Transaction expectations.
- Consistency model.
- Query characteristics.
- Tenant scope.
- Error semantics.
- Pagination behavior.
- Caching/search interaction where relevant.
- Performance considerations.

---

# 102. Repository Review Checklist

Before merging repository changes, verify:

### Architecture

- Is the repository boundary correct?
- Does it have a clear owner?
- Is the dependency direction correct?
- Is business logic outside the repository?

### Data

- Is the query correct?
- Are indexes appropriate?
- Are constraints respected?
- Is tenant isolation enforced?
- Are transactions correct?

### Performance

- Is there an N+1 query?
- Is pagination bounded?
- Are batch operations appropriate?
- Is memory bounded?
- Is connection pool usage safe?

### Concurrency

- Can concurrent requests corrupt state?
- Is check-then-act avoided?
- Are locks/constraints appropriate?
- Is the operation idempotent?

### Security

- Are queries parameterized?
- Can another tenant's data be accessed?
- Is sensitive data minimized?
- Are dynamic fields allowlisted?

### Reliability

- Are timeouts handled?
- Are retry semantics safe?
- Are errors translated?
- Are partial failures understood?

### Testing

- Are repository integration tests present?
- Are failure cases tested?
- Are concurrency cases tested?
- Is tenant isolation tested?

---

# 103. Definition of Done

A repository implementation is complete only when:

- Its persistence responsibility is clearly defined.
- Its contract is explicit.
- Its dependency direction is correct.
- It does not contain business workflow logic.
- It does not contain HTTP concerns.
- It uses the correct authoritative datastore.
- Queries are parameterized and bounded.
- Indexes support important query patterns.
- Pagination is safe for large datasets.
- N+1 behavior has been considered.
- Transactions are explicit.
- Concurrency behavior is understood.
- Tenant isolation is enforced.
- Error semantics are stable.
- Retry behavior is safe.
- Large datasets use bounded memory.
- Cache/search behavior is explicitly defined.
- Tests verify real persistence behavior.
- Required migrations exist.
- Observability is sufficient.
- Documentation is updated.

---

# 104. Final Invariants

```text
1. Repositories are persistence boundaries, not business-service boundaries.

2. Application services own workflows.

3. Domain code owns business rules.

4. Repository implementations own database-specific behavior.

5. Repository contracts must express domain/application intent.

6. Generic CRUD repositories should not be the default architecture.

7. A repository must have a clear data ownership boundary.

8. A repository must preserve tenant isolation.

9. Authorization does not belong inside repositories, but persistence
   operations must still enforce required tenant/resource scoping.

10. Repository methods must have explicit not-found semantics.

11. Database-specific errors must not leak through application APIs.

12. Database transactions must be explicit and appropriately scoped.

13. External network calls must not be hidden inside database transactions.

14. Check-then-act operations must not be used where concurrency can
    invalidate the assumption.

15. Database constraints should provide final protection for critical
    uniqueness and integrity requirements.

16. Repository queries must be bounded and performance-conscious.

17. N+1 database access is prohibited unless explicitly justified.

18. Large datasets must use pagination, batching, or streaming.

19. Unbounded database reads are prohibited on request paths.

20. Repository implementations must reuse database connection pools.

21. Repository methods must respect cancellation and timeouts.

22. Retries must account for operation idempotency and ambiguous outcomes.

23. Redis is not the authoritative source of transactional domain state
    unless explicitly defined otherwise.

24. OpenSearch is a derived search/read model unless explicitly defined
    otherwise.

25. Cache behavior must not silently change repository consistency semantics.

26. Events required for reliable state propagation should use the
    transactional outbox pattern.

27. Repository implementations must not expose database-driver details
    to the domain layer.

28. ORM models must not automatically become domain models.

29. Raw SQL is acceptable when it is justified, parameterized, tested,
    and maintainable.

30. Repository interfaces should be small and cohesive.

31. Interfaces must not exist solely for mocking.

32. Repository code must remain observable without logging sensitive data.

33. Schema changes and repository changes must remain synchronized.

34. Repository changes must include appropriate integration tests.

35. A repository must make persistence behavior easier to reason about,
    not hide it behind generic abstractions.

36. If a repository starts implementing workflows, authorization,
    external integrations, or unrelated domains, its boundary must be
    reviewed.
```

## KAMPYN Repository Model

The intended model is:

```text
                    APPLICATION
                         │
          ┌──────────────┼──────────────┐
          ↓              ↓              ↓
    OrderRepository  UserRepository  BookingRepository
          │              │              │
          ↓              ↓              ↓
    PostgreSQL       PostgreSQL       PostgreSQL
          │
          │
          ├──────────────→ Outbox
          │                    │
          │                    ↓
          │                 Events
          │                    │
          │              ┌─────┴─────┐
          │              ↓           ↓
          │          OpenSearch    Cache
          │
          ├─────────────────────────────┐
          ↓                             ↓
      MongoDB                         Redis
  (document workloads)           (optimization/
                                  transient state)
```

The core principle is:

```text
Repositories isolate persistence.

They do not hide architecture.
They do not own business decisions.
They do not become generic CRUD wrappers.

They provide a small, explicit, performant boundary
between KAMPYN's application/domain logic and its
persistence infrastructure.
```
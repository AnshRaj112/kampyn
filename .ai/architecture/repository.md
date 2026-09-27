# KAMPYN Repository Architecture

## 1. Purpose

The Repository layer provides a controlled abstraction between **application/domain logic and persistence infrastructure**.

Repositories are responsible for retrieving and persisting domain data without allowing business logic to become coupled to:

- PostgreSQL
- MongoDB
- Redis
- OpenSearch
- ORM/ODM implementations
- database drivers
- query builders
- persistence-specific data structures

The repository architecture must preserve:

```text
Business Logic
      │
      ▼
Repository Contract
      │
      ▼
Repository Implementation
      │
      ▼
Persistence Infrastructure
```

The core principle is:

> **Business logic depends on what it needs from persistence, not on how persistence is implemented.**

---

# 2. Repository Position in Architecture

KAMPYN follows:

```text
Presentation
      │
      ▼
Application
      │
      ▼
Domain
      │
      ▼
Infrastructure
```

Repositories sit at the boundary between domain/application requirements and infrastructure.

```text
┌─────────────────────────────────────┐
│           Presentation              │
│ HTTP / API / Webhooks / Jobs        │
└──────────────────┬──────────────────┘
                   │
                   ▼
┌─────────────────────────────────────┐
│           Application               │
│ Use Cases / Commands / Queries      │
└──────────────────┬──────────────────┘
                   │
                   ▼
┌─────────────────────────────────────┐
│             Domain                  │
│ Entities / Rules / Contracts        │
│                                     │
│ Repository Interfaces               │
└──────────────────┬──────────────────┘
                   │
                   ▼
┌─────────────────────────────────────┐
│          Infrastructure             │
│                                     │
│ PostgreSQL Repository               │
│ MongoDB Repository                  │
│ Redis Adapter                       │
│ Search Adapter                      │
└─────────────────────────────────────┘
```

The domain/application layer must not import concrete database implementations.

---

# 3. Repository Responsibilities

A repository may be responsible for:

- retrieving entities
- retrieving projections/read models
- persisting entities
- updating entities
- deleting entities
- checking existence
- executing persistence-specific queries
- handling persistence mapping
- supporting pagination
- supporting transactional operations where required
- translating persistence errors

A repository must **not** become responsible for:

- HTTP handling
- authorization decisions
- business workflows
- request validation
- UI state
- notification delivery
- arbitrary orchestration
- unrelated domain logic

---

# 4. Repository Contract

A repository contract defines what the application needs.

Example:

```go
type OrderRepository interface {
    Create(ctx context.Context, order Order) error
    FindByID(ctx context.Context, id OrderID) (Order, error)
    FindByUser(ctx context.Context, userID UserID, page Page) ([]Order, error)
    Update(ctx context.Context, order Order) error
}
```

The interface should represent meaningful domain operations rather than exposing generic database operations.

Prefer:

```text
FindPendingOrders(...)
FindByUser(...)
FindByID(...)
Save(...)
```

over:

```text
Query(...)
Execute(...)
RawSQL(...)
Find(...)
```

when the latter leaks infrastructure concepts.

---

# 5. Interface Ownership

Repository interfaces should generally be owned by the layer that consumes them.

For example:

```text
Application
    │
    ├── OrderService
    │
    └── OrderRepository interface
             ▲
             │
             │ implements
             │
Infrastructure
    │
    └── PostgresOrderRepository
```

This follows dependency inversion.

The application defines the required capability.

Infrastructure provides the implementation.

---

# 6. Dependency Direction

The dependency direction must remain:

```text
Application
     │
     ▼
Repository Contract
     ▲
     │
Infrastructure Implementation
     │
     ▼
Database
```

Never:

```text
Application
     │
     ▼
PostgreSQL Repository
     │
     ▼
PostgreSQL-specific API
```

The application should not need to know whether the data is stored in PostgreSQL, MongoDB, or another implementation.

---

# 7. Repository vs Service

Repositories and services have different responsibilities.

### Repository

Answers:

> How do I retrieve or persist this data?

### Application Service

Answers:

> What workflow should happen?

Example:

```text
CreateOrder
    │
    ├── validate command
    ├── authorize user
    ├── load menu/item
    ├── apply business rules
    ├── create order
    ├── persist order
    └── publish event
```

The repository should only handle persistence operations.

It should not orchestrate the entire workflow.

---

# 8. Repository vs Domain Logic

Repositories should not contain business rules.

Avoid:

```text
OrderRepository
    ├── calculate discount
    ├── check payment eligibility
    ├── determine delivery fee
    └── save order
```

Prefer:

```text
OrderApplicationService
    │
    ├── business rules
    │
    ▼
OrderRepository
    │
    └── persistence
```

Domain invariants belong in the domain model or appropriate domain services.

---

# 9. Repository vs Authorization

Repositories should not generally decide whether the caller is authorized.

Authorization belongs to:

```text
Authentication
       ↓
Authorization
       ↓
Application
       ↓
Repository
```

However, repository queries must still enforce required data boundaries when the architecture requires it.

For multi-tenant systems:

```text
FindOrder(
    ctx,
    tenantID,
    orderID,
)
```

is preferable to:

```text
FindOrder(orderID)
```

when tenant scope is part of the data access invariant.

---

# 10. Tenant Isolation

KAMPYN repositories must respect tenant boundaries.

A repository must never accidentally execute:

```sql
SELECT * FROM orders WHERE id = $1;
```

when tenant isolation requires:

```sql
SELECT *
FROM orders
WHERE tenant_id = $1
  AND id = $2;
```

Tenant context must be explicit and consistently enforced.

Repository methods should not silently infer tenant identity from arbitrary global state.

---

# 11. Tenant Context

A repository operation may require:

```go
type TenantContext struct {
    TenantID TenantID
}
```

or an equivalent established application context.

The important invariant is:

```text
Every tenant-scoped repository operation
        ↓
must have an unambiguous tenant scope
```

Cross-tenant operations must be explicit and restricted to authorized platform-level workflows.

---

# 12. Repository Granularity

Repositories should generally represent meaningful aggregates or persistence boundaries.

Examples:

```text
UserRepository
OrderRepository
MenuRepository
FoodCourtRepository
HostelRepository
BookingRepository
ComplaintRepository
PaymentRepository
InventoryRepository
```

Avoid creating repositories for every database table when those tables form one cohesive aggregate.

Do not create:

```text
OrderItemRepository
OrderStatusRepository
OrderMetadataRepository
```

simply because separate tables exist.

Persistence structure and domain boundaries are not automatically identical.

---

# 13. Aggregate Boundaries

Repositories should generally load and persist aggregate boundaries.

For example:

```text
Order
 ├── OrderItems
 ├── Pricing
 └── Status
```

If these form one consistency boundary, the repository should treat them accordingly.

Example:

```go
OrderRepository.Save(ctx, order)
```

rather than forcing application code to coordinate:

```text
OrderRepository
OrderItemRepository
OrderPricingRepository
OrderStatusRepository
```

for every order mutation.

---

# 14. Repository Methods

Repository methods should be explicit and intention-revealing.

Prefer:

```text
FindByID
FindByUserID
FindActive
FindPending
ListForTenant
ExistsByEmail
CountActive
Save
Delete
```

Avoid overly generic methods such as:

```text
Get
Fetch
Process
Handle
Execute
Run
```

unless their meaning is genuinely clear from the abstraction.

---

# 15. Queries vs Commands

Repositories may support both:

```text
Queries
    → retrieve information

Commands
    → modify persistence state
```

For example:

```text
Queries
 ├── FindByID
 ├── FindByUser
 └── ListPending

Commands
 ├── Create
 ├── Update
 └── Delete
```

This distinction improves clarity and observability.

---

# 16. Read Models

Not every read operation needs to return a full domain entity.

For high-volume or read-heavy workflows, repositories may return dedicated projections.

Example:

```go
type OrderSummary struct {
    ID        OrderID
    Status    OrderStatus
    Total     Money
    CreatedAt time.Time
}
```

This avoids loading unnecessary data.

The repository should not force the application to retrieve an entire aggregate when only a projection is required.

---

# 17. Repository DTOs

Persistence models should not automatically become domain models.

Example:

```text
Database Model
      │
      ▼
Repository Mapping
      │
      ▼
Domain Entity
```

Likewise:

```text
Database Model
      │
      ▼
Repository Mapping
      │
      ▼
Read Projection
```

This prevents database schema decisions from leaking into business logic.

---

# 18. Persistence Models

Database-specific structures belong in infrastructure.

Example:

```go
type orderRow struct {
    ID        string
    TenantID  string
    Status    string
    Total     int64
    CreatedAt time.Time
}
```

The domain should not need to know that the persistence layer stores:

```text
status VARCHAR
total BIGINT
tenant_id UUID
```

in a particular database representation.

---

# 19. Mapping

Repositories should own persistence-to-domain mapping.

```text
Database Row
     │
     ▼
Repository
     │
     ├── validate representation
     ├── map fields
     └── construct domain object
          │
          ▼
      Domain Entity
```

Mapping logic should be deterministic and testable.

---

# 20. PostgreSQL Repositories

PostgreSQL should be used for relational and transactional workloads.

Example:

```text
OrderRepository
      │
      ▼
PostgreSQL
```

The repository should manage:

- SQL execution
- parameter binding
- transaction participation
- row mapping
- pagination
- database errors
- query-specific optimization

The repository should not expose raw SQL to application callers.

---

# 21. MongoDB Repositories

MongoDB repositories should be used where document-oriented persistence is appropriate.

Example:

```text
DocumentRepository
      │
      ▼
MongoDB
```

The repository owns:

- MongoDB filters
- projections
- indexes
- document mapping
- update operators
- Mongo-specific errors

Mongo-specific types should not leak unnecessarily into domain/application code.

---

# 22. Redis

Redis should normally be treated as infrastructure rather than the primary repository for business entities.

Redis may provide:

```text
cache
distributed locks
rate limiting
short-lived state
sessions
coordination
```

If Redis becomes authoritative for a specific domain state, that decision must be explicit and documented.

Do not silently turn Redis cache data into a source of truth.

---

# 23. OpenSearch

OpenSearch is generally a derived search/read system.

It should not replace the authoritative repository for transactional entities.

Example:

```text
PostgreSQL
    │
    ▼
Domain Event
    │
    ▼
Search Indexer
    │
    ▼
OpenSearch
```

Search repositories may provide:

```text
SearchItems
SearchVendors
SearchFoodCourts
SearchHostels
```

but the underlying source of truth remains the appropriate primary datastore.

---

# 24. Search Repository Separation

Search operations should not be forced through transactional repositories.

Prefer:

```text
ItemRepository
    → PostgreSQL

ItemSearchRepository
    → OpenSearch
```

This keeps transactional persistence and derived search infrastructure separate.

---

# 25. Cache and Repository Interaction

Caching should not be hidden inside every repository automatically.

A repository may use caching where it is part of the established architecture, but the ownership must be clear.

Possible architecture:

```text
Application
    │
    ▼
Cached Service
    │
    ├── Cache
    │
    └── Repository
```

or:

```text
Application
    │
    ▼
Repository
    │
    ├── Cache
    └── Database
```

The chosen approach must be consistent.

Cache behavior must not change the correctness semantics of the repository.

---

# 26. Transactions

Repositories must participate correctly in transaction boundaries.

The application layer should define the business transaction.

Example:

```text
CreateOrder
    │
    ▼
Transaction
    │
    ├── Create Order
    ├── Create Order Items
    ├── Update Inventory
    └── Write Outbox Event
```

All operations that require atomicity must share the same transaction boundary.

---

# 27. Transaction Context

Repository methods should support transaction participation without exposing database implementation details unnecessarily.

Conceptually:

```text
Application Transaction
        │
        ▼
Repository A
        │
        ├── DB operation
        │
Repository B
        │
        └── DB operation
```

The implementation may use:

```text
transaction context
unit of work
transaction executor
```

depending on the selected infrastructure approach.

---

# 28. Transaction Ownership

Avoid repositories independently creating transactions for every operation when the workflow requires multiple operations to be atomic.

Bad:

```text
Repository A
    → transaction 1

Repository B
    → transaction 2
```

when both operations must succeed or fail together.

Prefer:

```text
Application Workflow
        │
        ▼
Transaction
   ├── Repository A
   └── Repository B
```

---

# 29. Outbox Pattern

When a database mutation must produce an event, the repository transaction should support the outbox architecture.

```text
Transaction
 ├── Business Data
 └── Outbox Event
```

Then:

```text
Outbox
   │
   ▼
Publisher
   │
   ▼
Event Bus
```

This prevents:

```text
Database committed
       +
Event publish failed
```

from silently producing inconsistent system state.

---

# 30. Concurrency

Repositories must account for concurrent operations.

Examples include:

```text
inventory updates
seat allocation
hostel booking
washing machine booking
payment state transitions
order state transitions
```

Appropriate mechanisms may include:

```text
optimistic locking
pessimistic locking
unique constraints
atomic updates
transactions
compare-and-swap
```

The repository should expose persistence capabilities needed to enforce the domain invariant.

---

# 31. Idempotency

Repositories must support idempotent workflows where required.

For example:

```text
payment webhook
      │
      ▼
idempotency key
      │
      ▼
repository
      │
      ├── already processed → return existing result
      │
      └── new → persist result
```

Database uniqueness constraints should be used where appropriate instead of relying only on application-level checks.

---

# 32. Uniqueness

Uniqueness requirements should be enforced at the persistence layer when they represent data integrity constraints.

Examples:

```text
unique tenant domain
unique university identifier
unique payment provider transaction
unique booking allocation
unique external provider ID
```

Do not rely solely on:

```text
SELECT first
IF not exists
INSERT
```

for concurrent uniqueness-sensitive operations.

---

# 33. Pagination

Repositories handling large datasets must support efficient pagination.

Prefer cursor-based pagination where appropriate:

```text
FindOrders(
    tenantID,
    cursor,
    limit,
)
```

Avoid loading an entire dataset into memory.

Offset pagination may still be appropriate for certain administrative or small datasets.

The repository should choose pagination based on workload characteristics.

---

# 34. Sorting and Filtering

Repositories should support only meaningful and controlled filters.

Example:

```text
FindOrders(
    tenantID,
    status,
    createdAfter,
    createdBefore,
    cursor,
    limit,
)
```

Do not expose unrestricted SQL or arbitrary query expressions through repository interfaces.

---

# 35. Dynamic Queries

Dynamic filtering must remain controlled.

Avoid APIs such as:

```go
Query(ctx, map[string]any)
```

when they allow callers to construct arbitrary persistence queries.

Prefer typed query specifications:

```go
type OrderFilter struct {
    Status      *OrderStatus
    CreatedFrom *time.Time
    CreatedTo   *time.Time
}
```

This preserves type safety and keeps query semantics explicit.

---

# 36. N+1 Prevention

Repositories must avoid accidental N+1 database access.

Bad:

```text
Find 100 orders
   │
   ├── query customer
   ├── query customer
   ├── query customer
   └── ...
```

Prefer:

```text
Find orders
     │
     ▼
Batch related data
     │
     ▼
Map results
```

or an appropriate joined/preloaded query.

Repository performance must be measured for realistic dataset sizes.

---

# 37. Batch Operations

Repositories should provide batch operations where they materially improve performance.

Examples:

```text
CreateMany
UpdateMany
FindManyByIDs
DeleteMany
```

Batch APIs must preserve:

- transaction semantics
- error handling
- tenant isolation
- idempotency
- data integrity

Do not create batch methods merely for theoretical optimization.

---

# 38. Large Data

Repositories must not blindly load large datasets into memory.

For large workloads use:

```text
streaming
cursor-based iteration
chunked queries
batch processing
bounded concurrency
```

For example:

```text
Database
   │
   ▼
Cursor
   │
   ▼
Chunk
   │
   ▼
Process
   │
   ▼
Next Chunk
```

This is especially important for:

- analytics
- exports
- inventory reconciliation
- large file processing
- administrative bulk operations

---

# 39. Repository Error Mapping

Infrastructure-specific errors should be translated into meaningful application/domain errors.

Example:

```text
PostgreSQL unique violation
        │
        ▼
Repository
        │
        ▼
ErrAlreadyExists
```

Another example:

```text
Database timeout
        │
        ▼
ErrDependencyUnavailable
```

The application should not need to inspect PostgreSQL-specific error codes unless explicitly required by the architecture.

---

# 40. Error Categories

Repositories may expose categories such as:

```text
NotFound
AlreadyExists
Conflict
InvalidData
Unavailable
Timeout
TransactionFailure
Unknown
```

Errors must preserve enough context for logging and tracing without exposing sensitive persistence details.

---

# 41. Not Found Semantics

Repositories should distinguish:

```text
resource does not exist
```

from:

```text
database failed
```

For example:

```text
FindByID
    │
    ├── found → entity
    ├── not found → ErrNotFound
    └── database failure → infrastructure error
```

The application layer can then map these into appropriate API responses.

---

# 42. Repository Timeouts

Repository operations must respect context deadlines.

Example:

```go
func (r *OrderRepository) FindByID(
    ctx context.Context,
    id OrderID,
) (Order, error)
```

The repository should not create arbitrary long-lived contexts that ignore caller cancellation.

This is especially important for:

- HTTP requests
- background jobs
- worker shutdown
- database operations
- external integrations

---

# 43. Connection Management

Repositories must use managed connection pools.

Do not create a new database connection for every repository operation.

The infrastructure layer should own:

```text
connection pool
connection limits
timeouts
health checks
reconnection behavior
```

Repositories consume the managed dependency.

---

# 44. Repository Initialization

Repositories should receive their dependencies through dependency injection.

Example:

```text
Database
   │
   ▼
Repository Constructor
   │
   ▼
Repository
   │
   ▼
Application Service
```

Avoid package-level database globals.

Prefer explicit dependencies.

---

# 45. Repository Lifecycle

Repositories should not own global application lifecycle.

The application/infrastructure composition root should manage:

```text
database
connection pool
repository
service
server
```

Shutdown should occur in dependency order.

---

# 46. Repository Interfaces and Go

Go interfaces should remain small.

Prefer:

```go
type UserReader interface {
    FindByID(ctx context.Context, id UserID) (User, error)
}
```

when only reading is required.

A larger interface can be used when the consuming component genuinely needs all operations.

Avoid massive interfaces such as:

```go
type UserRepository interface {
    Create(...)
    Update(...)
    Delete(...)
    Search(...)
    Export(...)
    Import(...)
    Authenticate(...)
    SendNotification(...)
    ...
}
```

Large interfaces indicate multiple responsibilities.

---

# 47. Interface Segregation

Separate capabilities when consumers need different subsets.

Example:

```text
UserReader
UserWriter
UserSearcher
```

Only introduce these interfaces when there is a real architectural benefit.

Do not split every interface mechanically.

---

# 48. Repository Package Structure

A possible Go structure:

```text
internal/
├── domain/
│   ├── order/
│   │   ├── entity.go
│   │   ├── repository.go
│   │   └── service.go
│   │
│   └── user/
│       ├── entity.go
│       └── repository.go
│
├── application/
│   └── order/
│       ├── create.go
│       └── get.go
│
└── infrastructure/
    └── persistence/
        ├── postgres/
        │   ├── order_repository.go
        │   └── user_repository.go
        │
        ├── mongo/
        │   └── ...
        │
        └── redis/
            └── ...
```

Exact organization may vary according to bounded contexts and repository size.

---

# 49. Feature-Oriented Organization

For larger domains, repositories may be colocated with the bounded context.

Example:

```text
internal/
├── modules/
│   ├── orders/
│   │   ├── domain/
│   │   ├── application/
│   │   ├── infrastructure/
│   │   │   └── repository/
│   │   └── transport/
│   │
│   ├── inventory/
│   └── bookings/
```

The important rule is clear ownership and dependency direction rather than one mandatory directory layout.

---

# 50. Repository Testing

Repositories require integration tests against the actual persistence technology.

Do not rely exclusively on mocks.

Test:

```text
create
read
update
delete
not found
constraints
transactions
rollback
concurrency
pagination
filtering
tenant isolation
mapping
error translation
```

---

# 51. Repository Unit Tests

Pure mapping and transformation logic can be unit tested.

Examples:

```text
database row → domain entity
database error → application error
projection mapping
query specification conversion
```

These tests should be fast and deterministic.

---

# 52. Repository Integration Tests

Integration tests should exercise:

```text
Repository
    ↓
Real Database
```

where practical.

Use isolated test databases or containers.

Verify actual:

- SQL
- indexes
- constraints
- transactions
- locking
- serialization
- database behavior

---

# 53. Concurrency Tests

Concurrency-sensitive repositories should have dedicated tests.

Example:

```text
100 concurrent booking attempts
          │
          ▼
Repository
          │
          ▼
Database
```

The test should verify that the invariant holds.

For example:

```text
Only one booking succeeds
```

rather than merely checking that no request returned an error.

---

# 54. Tenant Isolation Tests

Every tenant-scoped repository should have tests proving:

```text
Tenant A cannot retrieve Tenant B data.
```

Test both:

```text
direct lookup
list/query operations
```

because leaks often occur through list queries rather than direct lookups.

---

# 55. Repository Contract Tests

When multiple implementations exist, contract tests can verify that they satisfy the same behavioral expectations.

Example:

```text
Repository Contract
       │
       ├── PostgreSQL implementation
       │
       └── Test implementation
```

If MongoDB or another persistence implementation provides the same abstraction, shared contract tests can verify equivalent semantics where equivalence is actually intended.

---

# 56. Mock Repositories

Mocks or fakes may be appropriate for application-service unit tests.

Example:

```text
CreateOrderService
       │
       ▼
Mock OrderRepository
```

However, mocks must not replace integration testing of the real repository.

Avoid mocking every dependency simply to make tests pass.

---

# 57. Repository Performance

Repository performance should be evaluated using realistic data volumes.

Measure:

```text
query latency
rows scanned
rows returned
index usage
connection utilization
memory usage
network transfer
batch size
transaction duration
```

Do not optimize based solely on theoretical complexity.

Use actual query plans and measurements.

---

# 58. Index Awareness

Repository design and database indexing must remain aligned.

If a repository frequently performs:

```text
tenant_id + status + created_at
```

the database architecture should consider an appropriate index.

Indexes must be driven by real query patterns.

Unused indexes should not accumulate indefinitely.

---

# 59. Soft Deletes

If KAMPYN uses soft deletion, repositories must apply the policy consistently.

For example:

```text
FindByID
```

should not accidentally return deleted records when normal application behavior excludes them.

Administrative or recovery operations may explicitly request deleted records.

The distinction must be explicit.

---

# 60. Temporal Data

Repositories must handle timestamps consistently.

Use a consistent time representation.

Avoid mixing:

```text
local time
UTC
database-local time
application-local time
```

without an explicit reason.

Persist timestamps in the established canonical representation and convert only at presentation boundaries.

---

# 61. IDs

Repositories must treat IDs as stable identifiers.

They should not:

- regenerate IDs unexpectedly
- expose database-specific internal identifiers unnecessarily
- convert IDs inconsistently between layers

Domain/application identifiers should be mapped deliberately to persistence identifiers.

---

# 62. Bulk Administrative Operations

Administrative bulk operations should not bypass repository rules casually.

Example:

```text
Admin
  ↓
Bulk Disable Users
  ↓
Application Workflow
  ↓
Repository Batch Operation
```

The workflow must still enforce:

- authorization
- tenant boundaries
- validation
- audit requirements
- transaction semantics
- rate/resource limits

---

# 63. Imports and Exports

Large imports and exports should generally use specialized application workflows rather than making repositories responsible for entire file-processing pipelines.

Example:

```text
Import Workflow
    │
    ├── parse
    ├── validate
    ├── transform
    ├── batch
    └── repository
```

Repositories should provide efficient persistence primitives.

---

# 64. Repository and Events

Repositories should not silently publish integration events.

Prefer:

```text
Application Workflow
    │
    ├── Repository
    │
    └── Event / Outbox
```

unless event persistence is explicitly part of the repository's transaction boundary.

This keeps side effects visible.

---

# 65. Repository and External APIs

Repositories are generally for persistence, not arbitrary external API calls.

Avoid:

```text
PaymentRepository
    → database
    → Razorpay
    → email
    → notification
```

Instead:

```text
Application Service
    │
    ├── Repository
    ├── Payment Gateway
    └── Notification Service
```

External providers should use integration abstractions.

---

# 66. Repository and Domain Events

Repositories may persist domain events through an outbox when transactional consistency requires it.

However, domain event creation and business decisions should remain outside persistence implementation.

The repository should persist what the application/domain has decided.

---

# 67. Repository and Authorization Filtering

Authorization filtering must not be implemented as an arbitrary hidden repository behavior.

For example, silently changing:

```text
FindOrders()
```

based on global current-user state can make behavior difficult to reason about.

Prefer explicit scope:

```text
FindOrders(
    tenantID,
    userScope,
    filter,
)
```

when access scope materially affects the query.

---

# 68. Repository Security

Repositories must protect against:

- SQL injection
- NoSQL injection
- unrestricted queries
- tenant data leakage
- unsafe dynamic sorting
- unsafe dynamic filtering
- excessive result sizes
- unauthorized bulk access
- sensitive data exposure

Use parameterized queries and controlled query construction.

---

# 69. Repository Resource Limits

Repository methods should prevent accidental resource exhaustion.

Use:

```text
maximum page size
bounded batch size
query timeout
context cancellation
controlled concurrency
```

Avoid:

```text
ListAll()
```

for potentially unbounded datasets.

If an export truly requires all records, use streaming or controlled iteration.

---

# 70. Repository Observability

Repositories should integrate with the observability architecture.

Useful telemetry:

```text
repository
operation
database
duration
success/failure
error category
rows affected
tenant context
trace ID
```

Do not log:

```text
password
token
full SQL with sensitive parameters
full document payload
```

Repository telemetry should support debugging without leaking data.

---

# 71. Repository Naming

Names should communicate intent.

Prefer:

```text
OrderRepository
UserRepository
BookingRepository
InventoryRepository
OrderSearchRepository
```

Implementations may be:

```text
PostgresOrderRepository
MongoUserRepository
OpenSearchOrderSearchRepository
```

Avoid meaningless names such as:

```text
DBService
DataManager
DatabaseHelper
RepositoryUtil
```

---

# 72. Repository Reuse

Repositories should be reused when they represent the same persistence responsibility.

Do not create:

```text
OrderRepository
OrderDataRepository
OrderDBService
OrderPersistenceService
```

to access the same data.

One clear abstraction should normally own the responsibility.

---

# 73. Avoid Generic Repository Abstractions

Avoid generic abstractions such as:

```text
BaseRepository<T>
CRUDRepository<T>
GenericMongoRepository<T>
GenericSQLRepository<T>
```

unless they solve a proven repeated problem without hiding important semantics.

Generic CRUD abstractions often become too weak for real domain operations.

Prefer domain-specific repositories.

---

# 74. Repository Abstraction Rule

Use abstraction when it protects a meaningful boundary.

Do not abstract merely because:

```text
two functions look similar
```

The repository boundary should exist because:

```text
business logic
        ≠
persistence implementation
```

not because every database call needs an interface.

---

# 75. Migration Safety

Database migrations must be coordinated with repository changes.

For schema changes:

```text
Migration
   ↓
Backward-compatible repository change
   ↓
Deployment
   ↓
Data migration
   ↓
Cleanup
```

Avoid deploying application code that immediately assumes a schema change that has not been safely rolled out.

---

# 76. Backward Compatibility

During rolling deployments, repository implementations may temporarily interact with old and new schema versions.

Use compatible migration strategies such as:

```text
expand
migrate
contract
```

where required.

This is particularly important in Kubernetes environments with multiple application versions running simultaneously.

---

# 77. Repository and Self-Hosting

Universities may self-host KAMPYN.

Repository implementations should therefore avoid assumptions about a single cloud provider.

The repository boundary should allow:

```text
Managed PostgreSQL
       OR
Self-hosted PostgreSQL
```

without changing business logic.

Likewise for other supported persistence infrastructure.

---

# 78. Repository Configuration

Repository configuration should come from infrastructure configuration.

Examples:

```text
database URL
connection limits
query timeout
pool settings
MongoDB URI
Redis endpoint
OpenSearch endpoint
```

Repositories should not read arbitrary environment variables directly throughout business logic.

Configuration should be constructed centrally and injected.

---

# 79. Repository Documentation

Public repository contracts should document:

- purpose
- ownership
- consistency guarantees
- transaction expectations
- tenant scope
- error semantics
- pagination behavior
- concurrency guarantees
- idempotency expectations

Complex repository operations should include examples where useful.

---

# 80. Definition of Done

A repository implementation is complete when:

- [ ] Its responsibility is clearly defined.
- [ ] Its interface is consumer-oriented.
- [ ] Dependency direction is correct.
- [ ] Domain logic is not hidden inside persistence.
- [ ] Tenant isolation is enforced.
- [ ] Transactions are correctly handled.
- [ ] Concurrency requirements are addressed.
- [ ] Pagination is bounded.
- [ ] Large datasets are handled safely.
- [ ] Errors are translated appropriately.
- [ ] Sensitive data is protected.
- [ ] Observability is present.
- [ ] Integration tests cover persistence behavior.
- [ ] Concurrency tests exist where required.
- [ ] Performance has been evaluated.
- [ ] Documentation is updated.

---

# 81. Final Principle

KAMPYN repositories exist to protect the application and domain layers from persistence implementation details.

The intended architecture is:

```text
                 Presentation
                      │
                      ▼
                 Application
                      │
                      ▼
               Domain Contract
                      │
              ┌───────┴────────┐
              │                │
              ▼                ▼
       Repository          Search Contract
              │                │
              ▼                ▼
       Infrastructure      OpenSearch
              │
      ┌───────┼────────┐
      │       │        │
      ▼       ▼        ▼
 PostgreSQL MongoDB   Redis
```

The core rules are:

> **Repositories persist data; application services orchestrate workflows; domain logic enforces business rules.**

> **Repository interfaces express application needs, not database capabilities.**

> **Persistence implementation must remain replaceable without rewriting business logic.**

> **Tenant isolation, transactions, concurrency, performance, and data integrity are repository responsibilities where persistence is the enforcement boundary.**

> **Do not build generic repositories unless a real, repeated abstraction justifies them.**

> **A repository should be simple to use, difficult to misuse, and explicit about its consistency and data-access guarantees.**
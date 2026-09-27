# KAMPYN Backend Go Engineering

## Purpose

KAMPYN uses Go as the primary backend implementation language.

This document defines Go-specific engineering rules that sit below the broader backend architecture.

It covers:

- Go project structure
- Package boundaries
- Naming
- Types
- Interfaces
- Dependency injection
- Context propagation
- Error handling
- Concurrency
- HTTP services
- Database access
- Serialization
- Configuration
- Logging
- Observability
- Testing
- Performance
- Resource management
- Security
- Graceful shutdown
- Production deployment

These rules complement:

- `AGENTS.md`
- `.ai/`
- `architecture/backend.md`
- `architecture/api.md`
- `architecture/database.md`
- `backend/concurrency.md`
- `backend/background-jobs.md`

When rules overlap, the more specific architectural constraint should be followed.

---

# 1. Core Go Principles

KAMPYN Go code must prioritize:

```text
Correctness
    ↓
Clarity
    ↓
Maintainability
    ↓
Performance
    ↓
Optimization
```

Go code should be:

- Simple
- Explicit
- Strongly typed
- Easy to test
- Easy to reason about
- Efficient
- Observable
- Safe under concurrency

Do not write complicated Go merely to demonstrate language features.

---

# 2. Idiomatic Go

Prefer idiomatic Go over patterns copied from other languages.

Prefer:

```go
if err != nil {
    return err
}
```

over unnecessarily abstract error pipelines.

Prefer:

```go
for _, item := range items {
    process(item)
}
```

when index manipulation is unnecessary.

Prefer small functions and explicit control flow.

Avoid:

- Excessive abstraction
- Deep inheritance-like patterns
- Reflection-heavy designs
- Global state
- Giant utility packages
- Clever one-liners that reduce readability

---

# 3. Go Version

The repository must define one supported Go version.

Use the version declared by:

```text
go.mod
```

All development and CI environments should use a compatible version.

Do not introduce language or standard-library features unavailable in the project's supported version.

Go version upgrades must be deliberate and tested.

---

# 4. Module Definition

The backend must use Go modules.

The root module should contain:

```text
go.mod
go.sum
```

Dependencies must be explicitly declared.

Do not manually copy third-party source code into the repository unless there is an explicit architectural reason.

---

# 5. Dependency Management

Every dependency must have a clear purpose.

Before adding a dependency:

1. Check whether the standard library already solves the problem.
2. Search the repository for an existing solution.
3. Check whether an existing dependency already provides the capability.
4. Evaluate maintenance and security.
5. Evaluate transitive dependencies.
6. Consider operational impact.
7. Add only if justified.

Do not introduce libraries for trivial functionality.

---

# 6. Package Architecture

KAMPYN follows:

```text
Transport
   ↓
Application
   ↓
Domain
   ↓
Infrastructure
```

Go packages must respect this dependency direction.

Example:

```text
internal/
├── transport/
├── application/
├── domain/
└── infrastructure/
```

The exact directory structure may vary by domain, but dependency direction must remain explicit.

---

# 7. Domain-Oriented Organization

Prefer organizing significant backend functionality around domain boundaries.

Example:

```text
internal/
├── orders/
├── payments/
├── bookings/
├── inventory/
├── users/
├── notifications/
├── search/
└── reports/
```

Each domain may contain its own:

```text
application
domain
repository
transport
jobs
events
```

where justified.

Avoid one giant:

```text
internal/services/
```

package containing unrelated business logic.

---

# 8. Package Cohesion

A package should have one clear responsibility.

Good:

```text
inventory
```

containing inventory-specific functionality.

Bad:

```text
utils
```

containing:

```text
JWT
database helpers
string formatting
payment calculations
email helpers
HTTP utilities
```

Avoid generic dumping-ground packages.

---

# 9. Package Naming

Package names should be:

- Short
- Lowercase
- Meaningful
- Stable
- Noun-oriented

Prefer:

```go
package inventory
```

Avoid:

```go
package inventoryservice
package inventory_utils
package inventoryHelpers
```

Do not repeat the package name in exported identifiers unnecessarily.

Prefer:

```go
inventory.Repository
```

over:

```go
inventory.InventoryRepository
```

where the context already makes the meaning clear.

---

# 10. Internal Packages

Backend implementation details should generally live under:

```text
internal/
```

This prevents accidental external imports.

Use `internal` to protect:

- Domain implementation
- Database adapters
- Infrastructure clients
- Authentication internals
- Job handlers
- Internal application services

Only code intentionally designed for external consumption should be outside `internal`.

---

# 11. Public Packages

Do not expose packages publicly merely because they are technically reusable.

A public package should have:

- Stable API
- Clear ownership
- Documentation
- Compatibility expectations
- Tests

SDK-specific or public Go packages should be intentionally separated from internal backend code.

---

# 12. File Size

Production Go source files should normally remain below the KAMPYN guideline of:

```text
200 lines
```

A file exceeding this limit should trigger architectural review.

Do not artificially split files merely to satisfy the line count.

Instead ask:

```text
Does this file contain multiple responsibilities?
```

If yes, separate the responsibilities.

---

# 13. Function Size

Functions should remain small enough to understand locally.

Prefer:

```text
Validate
 ↓
Load
 ↓
Apply domain operation
 ↓
Persist
 ↓
Publish event
```

over one function containing hundreds of lines.

A function should have one primary responsibility.

---

# 14. Cyclomatic Complexity

Avoid unnecessarily complicated control flow.

Prefer early returns:

```go
if err != nil {
    return err
}

if !authorized {
    return ErrForbidden
}
```

over deeply nested conditionals.

Complex business rules should be extracted into cohesive domain functions.

---

# 15. Naming

Names should communicate intent.

Prefer:

```go
CreateOrder
ReserveInventory
FindBooking
CancelBooking
```

over:

```go
DoOrder
ProcessData
HandleThing
Execute
Run
```

Variables should be concise where their scope is small.

Longer descriptive names are appropriate for broader scopes.

---

# 16. Exported Identifiers

Exported identifiers should be intentionally public.

Do not export something simply because another package currently needs it.

Exported APIs create coupling.

Before exporting:

```text
Can the dependency direction be improved?
Can the operation remain internal?
Is this actually part of the package contract?
```

---

# 17. Comments

Comments should explain **why**, not restate **what**.

Bad:

```go
// Increment count by one.
count++
```

Useful:

```go
// We increment the version after a successful state transition
// so concurrent consumers can detect stale events.
version++
```

Comments must remain synchronized with implementation.

---

# 18. Documentation for Exported APIs

Exported types, functions, variables, and constants should have meaningful documentation when they form part of a public package contract.

Documentation should explain:

- Purpose
- Important invariants
- Error behavior
- Concurrency behavior
- Ownership
- Usage constraints

---

# 19. Types First

Use Go's type system to represent domain concepts.

Prefer:

```go
type OrderID string
type TenantID string
type UserID string
```

when stronger type separation materially reduces mistakes.

Do not introduce custom types for every primitive without a meaningful benefit.

---

# 20. Avoid Stringly-Typed Domains

Avoid representing important domain concepts entirely as arbitrary strings.

Bad:

```go
func Process(id string, status string, tenant string)
```

Prefer explicit domain types where appropriate:

```go
func Process(
    ctx context.Context,
    orderID OrderID,
    status OrderStatus,
    tenantID TenantID,
)
```

This improves compile-time safety.

---

# 21. Zero Values

Go's zero values should be considered deliberately.

Ask:

```text
What does zero mean?
```

For every important type.

Do not allow:

```text
empty ID
zero monetary amount
zero timestamp
empty tenant
```

to accidentally represent valid business state.

Validate values at appropriate boundaries.

---

# 22. Pointer Usage

Use pointers when:

- Mutation is required
- Nil has semantic meaning
- Avoiding large value copies is useful
- Identity/reference semantics are required

Do not use pointers everywhere automatically.

Avoid pointer-heavy APIs that make ownership and nil behavior unclear.

---

# 23. Nil Safety

Nil behavior must be explicit.

Do not rely on callers to magically know whether:

```go
*User
```

can be nil.

Document or structure APIs so invalid states are difficult to create.

---

# 24. Interfaces

Interfaces should be small and consumer-oriented.

Prefer:

```go
type OrderRepository interface {
    Create(ctx context.Context, order *Order) error
    FindByID(ctx context.Context, id OrderID) (*Order, error)
}
```

over large interfaces containing dozens of unrelated methods.

---

# 25. Define Interfaces Near Consumers

When practical, define interfaces where they are consumed rather than where implementations live.

Example:

```text
application/orders/
    repository.go

infrastructure/postgres/
    order_repository.go
```

The application defines what it needs.

Infrastructure implements it.

This keeps dependency direction clean.

---

# 26. Interface Pollution

Do not create interfaces merely because Go supports them.

Bad:

```go
type UserService interface {
    Create(...)
}
```

when only one concrete implementation exists and no abstraction is required.

Interfaces should solve a real problem such as:

- Dependency inversion
- Testing
- Provider switching
- Architectural boundary

---

# 27. Dependency Injection

Dependencies should be explicit.

Prefer constructor injection:

```go
func NewOrderService(
    repo OrderRepository,
    publisher EventPublisher,
) *OrderService
```

Avoid global service locators.

Avoid hidden dependencies inside functions.

---

# 28. Constructors

Constructors should validate required dependencies.

Example:

```go
func NewOrderService(repo OrderRepository) (*OrderService, error) {
    if repo == nil {
        return nil, ErrMissingDependency
    }

    return &OrderService{
        repo: repo,
    }, nil
}
```

If invalid construction should be impossible and dependencies are compile-time guaranteed, simpler constructors may be appropriate.

---

# 29. Dependency Ownership

The composition root should construct dependencies.

Conceptually:

```text
cmd/
  ↓
configuration
  ↓
infrastructure
  ↓
repositories
  ↓
application services
  ↓
transport / workers
```

Business packages should not construct infrastructure clients themselves.

---

# 30. Global Variables

Avoid mutable global variables.

Bad:

```go
var db *sql.DB
var redisClient *redis.Client
var currentTenant TenantID
```

Prefer explicit dependency ownership.

Global immutable constants are fine when appropriate.

---

# 31. Context

`context.Context` should be the first parameter of request-scoped operations.

Prefer:

```go
func (r *Repository) FindByID(
    ctx context.Context,
    id OrderID,
) (*Order, error)
```

Avoid storing context inside long-lived structs.

Do not use context as a general-purpose parameter bag.

---

# 32. Context Values

Context values should be reserved for request-scoped metadata that crosses architectural boundaries.

Examples:

```text
request ID
trace ID
authenticated identity
tenant context
```

Do not use context values for ordinary business parameters.

Prefer explicit function parameters for business data.

---

# 33. Error Handling

Errors are values and must be handled explicitly.

Bad:

```go
result, _ := repository.Find(ctx, id)
```

unless ignoring the error is intentional and documented.

Every ignored error must have a clear reason.

---

# 34. Error Wrapping

Wrap errors with meaningful context.

Prefer:

```go
return fmt.Errorf("find order %s: %w", id, err)
```

The `%w` wrapper preserves the underlying error.

Do not destroy the original error unnecessarily.

---

# 35. Sentinel Errors

Use sentinel errors for stable categories when callers need to inspect them.

Example:

```go
var (
    ErrNotFound  = errors.New("not found")
    ErrConflict  = errors.New("conflict")
    ErrForbidden = errors.New("forbidden")
)
```

Use:

```go
errors.Is(err, ErrNotFound)
```

rather than comparing error strings.

---

# 36. Typed Errors

Use typed errors when structured information is necessary.

Example:

```go
type ValidationError struct {
    Field string
    Code  string
}
```

Do not create custom error types for every possible failure.

---

# 37. Error Translation

Infrastructure errors should not leak directly through public API boundaries.

Example:

```text
PostgreSQL unique violation
        ↓
Repository
        ↓
Domain/Application conflict
        ↓
API
        ↓
HTTP 409
```

The transport layer should expose stable API error semantics.

---

# 38. Error Classification

Errors should be distinguishable as:

```text
Validation
Authentication
Authorization
Not Found
Conflict
Rate Limited
Dependency Failure
Internal Failure
Cancellation
Timeout
```

Classification must drive:

- HTTP status
- Retry behavior
- Logging
- Metrics
- Client behavior

---

# 39. Panic

Do not use panic for normal application errors.

Bad:

```go
if err != nil {
    panic(err)
}
```

Panic may be appropriate for:

- Impossible programmer errors
- Invalid startup configuration
- Truly unrecoverable initialization conditions

Production request handlers should generally return errors rather than panic.

---

# 40. Panic Recovery

HTTP servers and worker systems should have controlled panic recovery at appropriate process boundaries.

Recovery should:

- Prevent process-wide crashes where appropriate
- Record the failure
- Preserve observability
- Avoid leaking sensitive information
- Fail the current operation safely

Do not use recovery to hide programming errors.

---

# 41. HTTP Servers

HTTP transport code should remain thin.

Preferred:

```text
HTTP Handler
 ↓
Decode
 ↓
Validate
 ↓
Authenticate
 ↓
Authorize
 ↓
Application Use Case
 ↓
Encode Response
```

Do not put business logic into handlers.

---

# 42. Request Validation

Validate request data at the transport boundary.

Validation should include:

- Required fields
- Formats
- Lengths
- Ranges
- Enumerations
- Pagination limits
- File metadata
- Content types

Domain invariants must still be enforced deeper in the system.

Transport validation is not a replacement for domain validation.

---

# 43. Response Serialization

Response structures should be explicit DTOs.

Avoid returning internal domain objects directly.

Example:

```text
Domain Entity
      ↓
Application Result
      ↓
Response DTO
      ↓
JSON
```

This prevents accidental exposure of:

- Internal fields
- Sensitive data
- Infrastructure details
- Database-specific representations

---

# 44. JSON

Use explicit JSON tags.

Example:

```go
type OrderResponse struct {
    ID        string `json:"id"`
    CreatedAt string `json:"createdAt"`
}
```

JSON contracts are public contracts.

Changes must be deliberate.

---

# 45. Time

Use `time.Time` internally for timestamps.

Persist and serialize timestamps consistently.

Prefer UTC for backend storage and transport unless the contract explicitly requires another representation.

Never use local server time as an implicit source of truth.

---

# 46. Timeouts

Every external network operation must have a bounded timeout.

Do not rely indefinitely on:

```go
http.DefaultClient
```

for critical production integrations.

Use appropriately configured HTTP clients.

---

# 47. HTTP Clients

HTTP clients should be reused.

Do not construct a new client for every request.

A client should be configured with:

- Timeout
- Transport
- Connection pooling
- TLS settings
- Proxy configuration where required

---

# 48. HTTP Retries

Retries must be explicit.

Only retry when:

- The operation is safe to retry
- The error is transient
- The retry budget allows it

Use backoff and jitter.

Never blindly retry all HTTP failures.

---

# 49. Database Access

Database access belongs in infrastructure/repository packages.

Application code should depend on repository contracts rather than database-specific implementations.

Example:

```text
Application
   ↓
OrderRepository
   ↓
PostgreSQL Repository
   ↓
PostgreSQL
```

---

# 50. SQL

SQL should be:

- Parameterized
- Explicit
- Reviewable
- Efficient

Never concatenate untrusted input into SQL.

Bad:

```go
query := "SELECT * FROM users WHERE id = '" + id + "'"
```

Use parameterized queries.

---

# 51. Query Performance

Every significant database query should be evaluated for:

- Index usage
- N+1 behavior
- Row count
- Pagination
- Join cost
- Lock behavior
- Connection usage

Do not solve slow queries by blindly adding concurrency.

---

# 52. Transactions

Transaction boundaries should be controlled by the application/use-case layer where possible.

Repositories should provide the necessary primitives without hiding important transaction semantics.

Avoid:

```text
Repository A starts transaction
Repository B independently starts another transaction
```

when both operations must be atomic.

---

# 53. Connection Management

Database connections must be reused through properly configured pools.

Configure:

- Maximum open connections
- Maximum idle connections
- Connection lifetime
- Connection idle lifetime

Values must reflect actual deployment capacity.

---

# 54. MongoDB

MongoDB repositories must follow the same architectural principles.

Use:

- Explicit filters
- Appropriate indexes
- Bounded queries
- Projection where useful
- Context-aware operations
- Proper transaction semantics where required

Do not treat MongoDB as an unstructured dumping ground.

---

# 55. Redis

Redis clients should be shared and managed by infrastructure.

Use Redis for:

- Cache
- Queues
- Rate limiting
- Temporary coordination
- Short-lived state

Do not store authoritative transactional state in Redis unless explicitly designed for it.

---

# 56. OpenSearch

OpenSearch access belongs behind the search/infrastructure boundary.

Application code should not contain raw OpenSearch queries throughout business logic.

Prefer:

```text
Search Application
 ↓
Search Repository
 ↓
OpenSearch Adapter
```

Search remains a derived read model.

---

# 57. Goroutines

Use goroutines when concurrency provides measurable value.

Good candidates:

- Independent external calls
- Worker pools
- Background processing
- Event consumers
- Parallel CPU work with bounded workload

Do not use goroutines merely because an operation is slow.

---

# 58. WaitGroups and ErrGroups

Use structured coordination for related goroutines.

When multiple operations must complete together, use an appropriate synchronization mechanism such as `errgroup` where justified.

Requirements:

- Cancellation
- Error propagation
- Bounded concurrency
- Explicit ownership

Avoid manually managing complex goroutine lifecycles when a structured abstraction provides the same behavior more safely.

---

# 59. Channels

Use channels for:

- Work queues
- Pipelines
- Signaling
- Streaming

Do not use channels as a replacement for every mutex.

Choose the simplest concurrency primitive that expresses the problem correctly.

---

# 60. Worker Pools

Worker pools must define:

```text
Worker count
Queue capacity
Shutdown behavior
Error handling
Retry behavior
Context cancellation
Backpressure
```

See:

```text
backend/background-jobs.md
backend/concurrency.md
```

for detailed job and concurrency rules.

---

# 61. Goroutine Lifecycle

Every long-lived goroutine must have:

```text
Start
 ↓
Run
 ↓
Cancellation signal
 ↓
Cleanup
 ↓
Exit
```

The owner of the goroutine must know how it terminates.

---

# 62. Resource Management

Use `defer` for cleanup when appropriate.

Examples:

```go
defer rows.Close()
defer resp.Body.Close()
```

Do not defer cleanup inside huge loops when doing so would retain many resources simultaneously.

---

# 63. File Handling

File operations must:

- Validate paths
- Avoid path traversal
- Use bounded memory
- Close handles
- Enforce size limits
- Respect cancellation where possible
- Avoid unnecessary copies

Large files should be streamed.

---

# 64. Memory Allocation

Do not optimize allocations prematurely.

First identify actual allocation hotspots through profiling.

When optimization is justified:

- Preallocate known-capacity slices
- Reuse buffers carefully
- Avoid unnecessary conversions
- Stream large data
- Avoid copying large structures unnecessarily

Correctness comes first.

---

# 65. Slice Capacity

Preallocate when the size is known or reasonably estimated.

Example:

```go
items := make([]Item, 0, expectedCount)
```

Do not preallocate enormous capacities based on unreliable estimates.

---

# 66. Maps

Preallocate maps when expected size is known.

Example:

```go
items := make(map[string]Item, expectedCount)
```

Again, avoid huge speculative allocations.

---

# 67. String Building

For repeated string construction, use appropriate tools such as `strings.Builder`.

Do not repeatedly concatenate strings inside large loops when profiling shows it is expensive.

Do not optimize trivial strings without evidence.

---

# 68. Reflection

Avoid reflection in core business logic unless there is a strong reason.

Reflection can reduce:

- Type safety
- Readability
- Performance
- Tooling quality

Prefer generics or explicit types where appropriate.

---

# 69. Generics

Use generics when they remove meaningful duplication while preserving clarity.

Good use cases:

- Reusable data structures
- Generic infrastructure helpers
- Strongly typed utilities

Do not introduce generic abstractions simply because generic programming is possible.

---

# 70. Serialization

Serialization should be explicit and version-aware.

For external contracts:

```text
DTO
 ↓
Serializer
 ↓
Wire format
```

Do not expose internal structs directly when their representation is not a stable contract.

---

# 71. Configuration

Configuration should be loaded at startup.

Conceptually:

```text
Environment
 ↓
Configuration Loader
 ↓
Validation
 ↓
Application Dependencies
```

Do not repeatedly read environment variables throughout business logic.

---

# 72. Configuration Validation

Fail fast when required configuration is invalid.

Validate:

- URLs
- Ports
- Credentials
- Timeouts
- Concurrency limits
- Database configuration
- Queue configuration
- Feature configuration

Do not allow invalid configuration to fail later during a user request.

---

# 73. Secrets

Secrets must come from secure configuration mechanisms.

Never hard-code:

```text
Database passwords
API keys
JWT secrets
Encryption keys
Provider credentials
```

Do not log secrets during startup.

---

# 74. Environment Variables

Environment variables may provide deployment-specific configuration.

Prefer typed configuration structures:

```go
type Config struct {
    DatabaseURL string
    Port        int
    Environment string
}
```

Parse and validate once.

---

# 75. Logging

Use structured logging.

Prefer:

```text
level
message
request_id
trace_id
tenant_id
user_id where safe
operation
duration
```

Avoid:

```go
fmt.Println("something happened")
```

for production application logging.

---

# 76. Log Levels

Use levels intentionally:

```text
DEBUG
INFO
WARN
ERROR
```

Do not log everything at `ERROR`.

Do not log sensitive information merely for debugging convenience.

---

# 77. Tenant-Aware Logging

Tenant context should be available in structured logs for tenant-scoped operations.

Example:

```text
tenant_id=tenant_123
operation=CreateOrder
```

This greatly improves operational debugging.

Never allow tenant IDs or user identifiers to become uncontrolled metric cardinality.

---

# 78. Observability

Go services should expose sufficient information for:

- Logs
- Metrics
- Traces
- Health
- Dependency failures
- Database latency
- Queue processing
- Request latency

Observability should be implemented through shared infrastructure rather than duplicated manually in every handler.

---

# 79. Health Checks

Expose appropriate health checks.

Distinguish:

```text
Liveness
Readiness
Dependency health
```

A process being alive does not necessarily mean it is ready to serve traffic.

---

# 80. Graceful Shutdown

The Go application must handle termination signals.

Conceptually:

```text
SIGTERM
  ↓
Stop accepting new requests/jobs
  ↓
Cancel background work
  ↓
Drain active work
  ↓
Close dependencies
  ↓
Exit
```

Shutdown must have a bounded deadline.

---

# 81. Startup Order

Startup should roughly follow:

```text
Load configuration
        ↓
Validate configuration
        ↓
Initialize logging
        ↓
Initialize infrastructure clients
        ↓
Initialize repositories
        ↓
Initialize application services
        ↓
Initialize transport/workers
        ↓
Start server
```

Failures during required initialization should prevent startup.

---

# 82. Dependency Health

Do not blindly declare the service ready before critical dependencies are usable.

For example:

```text
API ready
```

should not necessarily be true if:

```text
Required database connection unavailable
```

The exact readiness policy must reflect whether the service can meaningfully serve requests.

---

# 83. Security

Go backend code must enforce:

- Input validation
- Authentication
- Authorization
- Tenant isolation
- Secure serialization
- Safe file handling
- SQL/NoSQL injection prevention
- Secret protection
- Rate limiting
- Request size limits
- Secure TLS configuration

Never rely on frontend validation for security.

---

# 84. Authentication Context

Authenticated identity should be represented using explicit server-side context.

Example conceptual flow:

```text
HTTP
 ↓
Authentication Middleware
 ↓
Identity Context
 ↓
Authorization
 ↓
Application
```

Do not allow application services to trust arbitrary user IDs supplied by clients.

---

# 85. Authorization

Authorization must happen server-side.

Application operations should receive enough trusted context to enforce:

```text
Identity
Tenant
Role
Permission
Resource ownership
Resource state
```

Do not rely solely on middleware for resource-level authorization.

---

# 86. Multi-Tenancy

Every tenant-scoped operation must carry explicit tenant context.

Prefer:

```go
type TenantContext struct {
    ID TenantID
}
```

over passing arbitrary tenant strings throughout the codebase.

Repositories must apply tenant scoping consistently.

---

# 87. Data Leakage Prevention

Go APIs must not accidentally return fields belonging to another tenant.

Particular care is required for:

- Repository queries
- Batch operations
- Cache reads
- Search queries
- Background jobs
- Admin endpoints
- Exports

Concurrency must never bypass tenant boundaries.

---

# 88. API Error Responses

Internal errors should not be returned directly to clients.

Bad:

```json
{
  "error": "pq: duplicate key value violates unique constraint ..."
}
```

Preferred:

```json
{
  "error": {
    "code": "RESOURCE_CONFLICT",
    "message": "The requested resource already exists.",
    "requestId": "req_123"
  }
}
```

Internal details belong in logs.

---

# 89. Testing Structure

Go tests should exist close to the code they validate.

Typical:

```text
order.go
order_test.go
```

Tests should cover behavior rather than implementation details.

---

# 90. Unit Tests

Unit tests should cover:

- Domain rules
- Validation
- State transitions
- Error classification
- Application logic
- Pure functions

Keep them deterministic.

---

# 91. Integration Tests

Integration tests should cover:

- PostgreSQL
- MongoDB
- Redis
- OpenSearch
- Object storage
- External provider adapters

Use realistic infrastructure where the behavior cannot be meaningfully tested with mocks.

---

# 92. HTTP Tests

Test:

- Request validation
- Authentication
- Authorization
- Tenant isolation
- Status codes
- Response schema
- Error contract
- Pagination
- Idempotency

Do not only test successful requests.

---

# 93. Concurrency Tests

Use explicit concurrent tests for:

- Inventory
- Booking
- Payments
- Idempotency
- Worker execution
- Cache stampede prevention
- State transitions

Run race detection where appropriate.

```bash
go test -race ./...
```

---

# 94. Benchmarks

Use Go benchmarks when performance matters.

Example:

```go
func BenchmarkOperation(b *testing.B) {
    for i := 0; i < b.N; i++ {
        operation()
    }
}
```

Benchmarks should answer a real performance question.

Do not create benchmarks merely to increase test count.

---

# 95. Fuzz Testing

Fuzz testing can be useful for:

- Parsers
- Validators
- Serialization
- File processing
- Query parsing
- Input normalization

Fuzzing should focus on boundaries where malformed input can expose correctness or security problems.

---

# 96. Test Isolation

Tests must not depend on execution order.

Avoid:

```text
Test A creates global state
Test B assumes it exists
```

Each test should establish its required state explicitly.

---

# 97. Mocks

Mock interfaces at architectural boundaries.

Good candidates:

- External providers
- Repositories
- Event publishers
- Notification providers

Do not mock every function.

Over-mocking can make tests validate implementation rather than behavior.

---

# 98. Test Data

Test fixtures should be:

- Minimal
- Explicit
- Reusable
- Deterministic

Avoid enormous fixtures for simple unit tests.

Use builders/factories where they genuinely improve readability.

---

# 99. Static Analysis

CI should run appropriate Go tooling.

At minimum:

```bash
gofmt
go vet ./...
go test ./...
```

Additional linters may be configured by the repository.

Formatting should be automated.

---

# 100. Formatting

All Go code must be formatted with:

```bash
gofmt
```

Do not manually enforce formatting conventions that `gofmt` already controls.

Formatting changes should remain separate from unrelated behavioral changes when practical.

---

# 101. Import Organization

Keep imports clean and let standard tooling manage formatting.

Avoid unnecessary dependencies and unused imports.

Go compilation should remain the final authority on import correctness.

---

# 102. Dead Code

Remove:

- Unused functions
- Unused packages
- Dead configuration
- Obsolete interfaces
- Deprecated implementations after migration

Do not preserve dead code "just in case."

Git preserves history.

---

# 103. Generated Code

Generated code must be clearly identified.

Do not manually modify generated files unless the generator workflow explicitly requires it.

Generated output should be reproducible.

---

# 104. Code Generation

Code generation may be used for:

- API contracts
- Database models
- Serialization
- Mocks
- SDKs

Generated artifacts must have:

- A documented generator
- Reproducible commands
- Clear ownership
- CI validation where appropriate

---

# 105. Dependency Direction

Go packages must not create circular architectural dependencies.

Preferred:

```text
transport
   ↓
application
   ↓
domain
   ↑
infrastructure implements interfaces
```

Infrastructure must not force domain code to depend on infrastructure-specific details.

---

# 106. Domain Purity

Domain logic should avoid direct dependency on:

- HTTP
- Redis
- PostgreSQL
- MongoDB
- OpenSearch
- Kubernetes
- External SDKs

Infrastructure concerns belong outside the domain layer.

---

# 107. Application Layer

Application services coordinate use cases.

They may handle:

- Transaction boundaries
- Authorization context
- Repository calls
- Domain operations
- Event publication
- Integration coordination

They should not become giant orchestration functions containing every business rule.

---

# 108. Repository Layer

Repositories abstract persistence operations.

Repositories should:

- Respect tenant scope
- Accept context
- Handle persistence-specific errors
- Use efficient queries
- Avoid business decisions
- Expose domain/application-relevant operations

Avoid generic repositories such as:

```text
GenericRepository[T]
```

unless they genuinely improve the architecture.

---

# 109. Service Layer

Do not create services solely because every package needs a `service.go`.

A service should represent meaningful application behavior.

Prefer:

```text
OrderService
PaymentService
BookingService
```

when they correspond to actual use cases.

---

# 110. Utility Packages

Utility code should remain narrowly scoped.

Prefer:

```text
/internal/clock
/internal/encoding
/internal/pagination
```

over:

```text
/internal/utils
```

Every utility must have a clear reason to exist.

---

# 111. Clock Abstraction

Time-dependent business logic may require an injectable clock.

Example:

```text
Clock.Now()
```

This improves deterministic testing.

Do not abstract time everywhere unnecessarily.

---

# 112. Randomness

Random values used for:

- Tokens
- Security identifiers
- Password reset codes
- Session identifiers

must use cryptographically secure randomness.

Do not use pseudo-random generators for security-sensitive values.

---

# 113. IDs

ID generation must be consistent across the system.

IDs should:

- Be unique
- Be safely serializable
- Have clear ownership
- Avoid accidental tenant ambiguity

Do not generate IDs in multiple unrelated formats without a reason.

---

# 114. Money

Money must not be represented using floating-point arithmetic for financial calculations.

Use:

- Integer minor units
- Decimal representation
- A dedicated money type

depending on the financial domain.

Example:

```text
₹100.50
```

should not be represented as an imprecise binary floating-point value.

---

# 115. Quantities

Inventory and item quantities must use types appropriate to their domain.

Avoid silently converting quantities between incompatible units.

Where units matter, model them explicitly.

---

# 116. Pagination

Go API implementations must enforce pagination limits.

Never allow an arbitrary client request such as:

```text
limit=100000000
```

to cause massive database reads.

Use:

```text
Default limit
Maximum limit
Cursor-based pagination
```

where appropriate.

---

# 117. Query Cancellation

Database and external calls must receive the request context.

Example:

```go
rows, err := db.QueryContext(ctx, query, args...)
```

This allows abandoned requests to release resources.

---

# 118. Resource Ownership

Every resource opened by Go code must have a clear owner.

Examples:

```text
DB client → application lifecycle
HTTP client → application lifecycle
File → function/operation
Rows → query scope
Goroutine → owning subsystem
Worker → worker lifecycle
```

Ownership must determine cleanup.

---

# 119. Graceful Resource Cleanup

Shutdown must close:

- Database pools
- Redis clients
- HTTP transports where required
- Search clients
- Message consumers
- File resources
- Worker pools

Cleanup should happen in reverse dependency order where appropriate.

---

# 120. Production Readiness Checklist

Before shipping Go backend code:

```text
[ ] gofmt passes
[ ] go vet passes
[ ] Tests pass
[ ] Relevant race tests pass
[ ] No unnecessary dependencies
[ ] Package boundaries are correct
[ ] Context is propagated
[ ] Errors are handled
[ ] Errors are wrapped where useful
[ ] No accidental panic paths
[ ] No global mutable state
[ ] Concurrency is bounded
[ ] External calls have timeouts
[ ] Database queries are reviewed
[ ] Transactions are explicit
[ ] Tenant isolation is verified
[ ] Authorization is enforced
[ ] Secrets are protected
[ ] Logs contain useful context
[ ] Sensitive data is not logged
[ ] Metrics/traces are appropriate
[ ] Graceful shutdown works
[ ] Configuration is validated
[ ] API contracts are compatible
[ ] Background jobs are idempotent where required
[ ] Documentation is updated
```

---

# 121. Final Invariants

The following rules are mandatory:

```text
1. Go code must remain idiomatic, explicit, and strongly typed.

2. Package boundaries must follow KAMPYN's architectural dependency direction.

3. Business logic must not depend directly on infrastructure.

4. Dependencies must be explicit and preferably injected through constructors.

5. Mutable global state is prohibited unless explicitly justified.

6. Every request-scoped operation must propagate context.

7. Every goroutine must have an explicit lifecycle and cancellation strategy.

8. Concurrency must be bounded.

9. Database constraints and transactions must protect persistent invariants.

10. External calls must have explicit timeouts.

11. Errors must be handled explicitly and retain useful context.

12. API errors must not expose internal infrastructure details.

13. Domain types should be used when they materially improve correctness.

14. Interfaces should be small and created for real abstraction boundaries.

15. Reflection and abstraction must not be introduced without a concrete benefit.

16. Large files and functions require architectural justification.

17. Database access must remain behind the appropriate repository/infrastructure boundary.

18. Redis and OpenSearch must not silently become sources of transactional truth.

19. Tenant context must remain explicit across concurrent and asynchronous operations.

20. Security-sensitive randomness must use cryptographically secure mechanisms.

21. Money must not use floating-point arithmetic for financial calculations.

22. Large datasets must be processed with bounded memory and concurrency.

23. Tests must cover failure, concurrency, and boundary behavior where relevant.

24. Race detector failures are correctness failures.

25. Performance optimizations must be evidence-driven.

26. Generated code must be reproducible.

27. Configuration must be validated at startup.

28. Graceful shutdown must be implemented for long-lived processes.

29. Dead code and obsolete abstractions must be removed rather than preserved.

30. Simpler correct Go code is preferred over clever Go code.
```

## Go Backend Flow

```text
                    KAMPYN Go Backend
                           │
                           ▼
                    ┌─────────────┐
                    │  Transport  │
                    │ HTTP / Jobs │
                    └──────┬──────┘
                           │
                           ▼
                    ┌─────────────┐
                    │ Application │
                    │  Use Cases  │
                    └──────┬──────┘
                           │
                           ▼
                    ┌─────────────┐
                    │   Domain    │
                    │ Rules/State │
                    └──────┬──────┘
                           │
                           ▼
                    ┌─────────────┐
                    │Infrastructure│
                    ├─────────────┤
                    │ PostgreSQL  │
                    │ MongoDB     │
                    │ Redis       │
                    │ OpenSearch  │
                    │ Storage     │
                    │ Integrations│
                    └─────────────┘
```

The Go implementation should remain **small, explicit, strongly typed, dependency-aware, concurrency-safe, observable, and boring where boring is the correct engineering choice**.
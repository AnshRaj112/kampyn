# KAMPYN Backend Architecture

## 1. Purpose

This document defines the architectural structure and responsibilities of the KAMPYN backend.

The backend is responsible for:

- Executing business logic.
- Enforcing authorization.
- Maintaining data integrity.
- Coordinating persistence.
- Managing external integrations.
- Processing asynchronous workloads.
- Publishing and consuming events.
- Providing stable APIs.
- Maintaining observability.
- Protecting system boundaries.

The backend must remain modular, testable, predictable, and independently deployable where the architecture requires it.

---

# 2. Backend Technology

The primary KAMPYN backend is built using:

- Go
- HTTP-based APIs
- PostgreSQL where relational transactional storage is appropriate
- MongoDB where document-oriented storage is appropriate
- Redis for caching and infrastructure state where appropriate
- OpenSearch for derived search workloads
- Object storage for large files and assets
- Background workers for asynchronous processing

Do not introduce additional infrastructure without architectural justification.

---

# 3. Backend Architecture

The backend follows:

```text id="z0d2a8"
Transport
    ↓
Application
    ↓
Domain
    ↓
Infrastructure
```

With supporting infrastructure:

```text id="i6kq9n"
                    ┌───────────────┐
                    │   Transport   │
                    └───────┬───────┘
                            ↓
                    ┌───────────────┐
                    │  Application  │
                    └───────┬───────┘
                            ↓
                    ┌───────────────┐
                    │    Domain     │
                    └───────┬───────┘
                            ↓
                    ┌───────────────┐
                    │ Infrastructure│
                    └───────┬───────┘
                            ↓
             ┌──────────────┼──────────────┐
             ↓              ↓              ↓
         PostgreSQL      MongoDB         Redis
                                          
                         OpenSearch
                         Object Storage
                         External APIs
```

The exact dependency direction must follow the repository's architecture rules.

---

# 4. Transport Layer

The transport layer handles external communication.

Responsibilities include:

- HTTP routing.
- Request parsing.
- Request validation.
- Authentication context extraction.
- Calling application services.
- Response serialization.
- HTTP error mapping.

Transport code must not contain substantial business logic.

Avoid:

```text id="6c0bte"
HTTP Handler
    ├── Database query
    ├── Business calculation
    ├── Payment API
    ├── Inventory update
    └── Response
```

Prefer:

```text id="8v7z5a"
HTTP Handler
    ↓
Application Service
    ↓
Domain
    ↓
Infrastructure
```

---

# 5. Application Layer

The application layer coordinates use cases.

Responsibilities include:

- Executing business workflows.
- Coordinating domain operations.
- Managing transaction boundaries.
- Calling repositories.
- Calling domain services.
- Publishing application events.
- Coordinating external integrations through interfaces.
- Enforcing workflow-level authorization requirements.

Application services should represent meaningful use cases.

Examples:

```text id="d2jz0u"
CreateOrder
CancelOrder
BookHostelRoom
ScheduleWashingMachine
SubmitComplaint
ApproveBooking
ProcessPayment
```

Avoid application services that merely rename a single database call without adding meaningful behavior.

---

# 6. Domain Layer

The domain layer contains business concepts and invariants.

It may contain:

- Entities.
- Value objects.
- Domain services.
- Domain errors.
- Domain events.
- Business rules.
- State transitions.

The domain should not depend directly on:

- HTTP frameworks.
- Database drivers.
- Redis clients.
- OpenSearch clients.
- Cloud provider SDKs.
- Specific external API providers.

Domain logic should remain independently testable.

---

# 7. Infrastructure Layer

Infrastructure implements technical capabilities required by the application.

Examples:

- PostgreSQL repositories.
- MongoDB repositories.
- Redis clients.
- OpenSearch clients.
- Object storage adapters.
- Email providers.
- Payment providers.
- External university integrations.
- Message brokers.

Infrastructure code may depend on external libraries and providers.

Business logic should not be embedded unnecessarily inside infrastructure adapters.

---

# 8. Dependency Direction

Dependencies should generally flow inward:

```text id="xg0xga"
Transport
   ↓
Application
   ↓
Domain
```

Infrastructure implements interfaces required by inner layers.

Conceptually:

```text id="xjpjp0"
Domain/Application
       ↓
    Interface
       ↑
Infrastructure Implementation
```

The domain must not depend on infrastructure merely because the infrastructure is convenient.

---

# 9. Package Boundaries

Packages should represent meaningful responsibilities.

Prefer:

```text id="g5ip4c"
orders/
bookings/
inventory/
users/
auth/
complaints/
foodcourts/
search/
notifications/
```

with appropriate internal boundaries.

Avoid packages such as:

```text id="q5v5x1"
misc/
helpers/
common/
stuff/
everything/
```

that become dumping grounds.

A package should have a clear reason to exist.

---

# 10. Domain Ownership

Each domain must have a clear owner.

For example:

```text id="0q9m1m"
Orders
 ├── Order domain rules
 ├── Order application services
 ├── Order persistence
 └── Order API

Inventory
 ├── Inventory rules
 ├── Inventory application services
 ├── Inventory persistence
 └── Inventory API
```

Another domain should not directly modify another domain's internal persistence structures.

Cross-domain interaction should use:

- Application contracts.
- Domain interfaces.
- Events.
- Explicit APIs.

---

# 11. Cross-Domain Communication

Cross-domain dependencies must be intentional.

For synchronous workflows:

```text id="q8cx3b"
Domain A
    ↓
Application Contract
    ↓
Domain B
```

For asynchronous workflows:

```text id="3kgf25"
Domain A
    ↓
Domain Event
    ↓
Event Infrastructure
    ↓
Domain B
```

Do not create hidden cross-domain dependencies through shared database tables.

---

# 12. Business Logic Placement

Business rules belong in the domain/application layers.

Avoid placing business logic in:

- HTTP handlers.
- Database repositories.
- ORM hooks.
- Serialization functions.
- React clients.
- Infrastructure adapters.

Example:

Bad:

```text id="s2xj5w"
OrderHandler
    ↓
if inventory > quantity
    ↓
update inventory
    ↓
create order
```

Prefer:

```text id="p6p6yz"
OrderApplicationService
    ↓
Order Domain
    ↓
Inventory Contract
    ↓
Transaction / Workflow
```

---

# 13. Domain Entities

Entities should encapsulate meaningful business behavior where appropriate.

An entity should not become a giant struct containing unrelated operations.

Keep:

- State.
- Invariants.
- Valid transitions.

close to the concept they represent.

Do not create domain entities merely to mirror database tables.

---

# 14. Value Objects

Use value objects for concepts with meaningful validation or semantics.

Examples:

```text id="e6h6q0"
Email
Money
TenantID
OrderID
BookingID
PhoneNumber
DateRange
Quantity
```

A value object should make invalid states harder to represent.

Do not create a new type for every primitive without a meaningful domain reason.

---

# 15. Domain Services

Use domain services when behavior:

- Belongs to the domain.
- Does not naturally belong to a single entity/value object.
- Represents a meaningful business operation.

Avoid creating:

```text id="r8p5wu"
OrderService
UserService
BookingService
```

that contain every operation associated with a noun.

Prefer cohesive services around actual business capabilities.

---

# 16. Application Services

Application services coordinate workflows.

They may:

- Validate application-level preconditions.
- Resolve authenticated identity.
- Start transactions.
- Load entities.
- Call domain behavior.
- Persist changes.
- Publish events.

They should not become giant procedural workflows containing every domain rule.

If an application service becomes difficult to understand, identify domain logic that belongs elsewhere.

---

# 17. Repository Pattern

Repositories abstract persistence where that abstraction provides value.

Repositories should expose domain-oriented operations.

Prefer:

```text id="8m4w6h"
FindActiveBooking(...)
ReserveSlot(...)
SaveOrder(...)
```

over exposing generic persistence mechanics everywhere.

Repositories should not contain unrelated business workflows.

---

# 18. Repository Interfaces

Repository interfaces should be defined close to the layer that consumes them when practical.

Example:

```text id="8t3b2q"
Application / Domain
    ↓
OrderRepository interface
    ↑
PostgresOrderRepository
```

This allows infrastructure implementations to change without rewriting business logic.

---

# 19. Database Access

Database access must remain within appropriate infrastructure boundaries.

The backend must:

- Use connection pooling.
- Use parameterized queries.
- Bound queries.
- Handle transactions explicitly.
- Handle errors correctly.
- Avoid N+1 queries.
- Use appropriate indexes.
- Avoid unnecessary data retrieval.

Do not allow handlers to directly access database clients.

---

# 20. Transaction Ownership

Transactions should be owned by the application workflow that requires atomicity.

Typical structure:

```text id="j4k9fi"
Application Service
    ↓
Begin Transaction
    ↓
Repository Operations
    ↓
Domain Operations
    ↓
Commit
```

Do not allow unrelated repository methods to silently create independent transactions when the caller expects atomicity.

Do not keep transactions open across external network calls unless explicitly required.

---

# 21. Context Propagation

Go `context.Context` must be propagated through request-scoped operations.

Context should carry:

- Cancellation.
- Deadlines.
- Request-scoped metadata where appropriate.

Do not use context as a general-purpose dependency container.

Avoid storing arbitrary business state in context.

---

# 22. Goroutines

Every goroutine must have a clear ownership and termination strategy.

Before starting a goroutine, answer:

```text id="y6o0js"
Who starts it?
Who stops it?
What happens if it fails?
What resources does it hold?
Can it leak?
Can it run forever?
```

Do not create unmanaged background goroutines inside request handlers.

---

# 23. Concurrency

Concurrent operations must be designed intentionally.

Consider:

- Shared memory.
- Mutexes.
- Channels.
- Atomic operations.
- Database locking.
- Distributed coordination.
- Race conditions.

Prefer the simplest concurrency model that correctly solves the problem.

Do not introduce concurrency merely because it appears faster.

---

# 24. Worker Pools

Use bounded worker pools for workloads where concurrency must be controlled.

Examples:

- File processing.
- Batch operations.
- External API processing.
- Event consumers.
- Background jobs.

Avoid launching one goroutine per unbounded input item.

Concurrency must be bounded by system resources and dependency capacity.

---

# 25. Background Jobs

Long-running work should be moved out of synchronous request handling.

Examples:

- Email sending.
- Large exports.
- Search indexing.
- Report generation.
- Notifications.
- Data imports.
- File processing.

Typical flow:

```text id="px6qnd"
API Request
    ↓
Create Job
    ↓
Return
    ↓
Worker
    ↓
Process Job
    ↓
Persist Result
```

Jobs must define:

- State.
- Retry behavior.
- Idempotency.
- Failure handling.
- Timeout.
- Observability.

---

# 26. Retries

Retries must be bounded.

A retry policy should define:

- Maximum attempts.
- Backoff.
- Jitter where appropriate.
- Retryable errors.
- Non-retryable errors.
- Timeout.
- Final failure behavior.

Never retry every error indiscriminately.

Do not retry non-idempotent operations without protection.

---

# 27. Events

Events should represent meaningful state changes or domain facts.

Prefer:

```text id="o1zqjm"
OrderCreated
BookingConfirmed
PaymentCompleted
ComplaintResolved
```

over generic:

```text id="2u7tqf"
SomethingChanged
DataUpdated
ProcessFinished
```

Events should contain enough information for consumers to act without exposing unnecessary internal data.

---

# 28. Event Delivery

Consumers must assume events may be delivered more than once unless the infrastructure explicitly guarantees otherwise.

Consumers should be idempotent.

Handle:

- Duplicate events.
- Delayed events.
- Out-of-order events where applicable.
- Failed processing.
- Retries.
- Dead-letter handling.

---

# 29. Outbox Pattern

When a database change and event publication must remain consistent, use an outbox pattern where appropriate.

Example:

```text id="4dqf0y"
Transaction
 ├── Update domain state
 └── Insert outbox event
        ↓
Commit
        ↓
Outbox Worker
        ↓
Publish Event
```

This prevents the application from successfully committing state while silently losing the corresponding event.

---

# 30. Eventual Consistency

Asynchronous workflows may create temporary inconsistency.

The system must explicitly define:

- Source of truth.
- Expected propagation delay.
- Consumer behavior.
- Retry behavior.
- Reconciliation mechanism.

Do not treat eventual consistency as an excuse for undefined behavior.

---

# 31. Caching

Redis may be used for caching where appropriate.

Cache design must define:

- Key.
- Owner.
- TTL.
- Invalidation.
- Staleness tolerance.
- Failure behavior.
- Tenant isolation.

Never allow cache availability to determine business correctness unless explicitly designed for it.

---

# 32. OpenSearch

OpenSearch is a derived search system.

The typical architecture is:

```text id="a7y7n5"
Primary Database
      ↓
Event / Projection Pipeline
      ↓
OpenSearch
      ↓
Search API
      ↓
Client
```

Do not treat OpenSearch as the authoritative transactional datastore.

Search indexes must be rebuildable where practical.

---

# 33. External Integrations

External services must be isolated behind adapters.

Examples:

- Payment providers.
- Email providers.
- SMS providers.
- University systems.
- Cloud storage.
- Identity providers.

Prefer:

```text id="4j9r2c"
Application
    ↓
Interface
    ↓
Provider Adapter
    ↓
External Service
```

Do not spread provider-specific SDK calls throughout the application.

---

# 34. External Service Failures

External integrations must handle:

- Timeout.
- Connection failure.
- Rate limits.
- Invalid responses.
- Authentication failure.
- Partial failure.
- Provider outage.

The backend must define whether the operation:

- Retries.
- Fails immediately.
- Enters a pending state.
- Moves to a background job.
- Requires reconciliation.

---

# 35. HTTP Clients

HTTP clients should be centralized or consistently configured.

Configure:

- Timeouts.
- Connection pooling.
- Retry behavior.
- TLS.
- Headers.
- Authentication.
- Observability.

Do not create arbitrary HTTP clients throughout the codebase.

---

# 36. Configuration

Configuration must be:

- Centralized.
- Typed.
- Validated.
- Environment-specific.
- Secure.

Examples:

- Database URLs.
- Redis configuration.
- OpenSearch configuration.
- Authentication settings.
- External service credentials.
- Feature flags.
- Runtime limits.

Invalid required configuration should fail clearly during startup.

---

# 37. Secrets

Secrets must never be:

- Hardcoded.
- Committed.
- Logged.
- Returned through APIs.
- Embedded in frontend bundles.

Use appropriate secret-management mechanisms for the deployment environment.

---

# 38. Application Startup

Startup should follow a predictable sequence.

Conceptually:

```text id="v5l5is"
Load Configuration
      ↓
Validate Configuration
      ↓
Initialize Logger
      ↓
Initialize Infrastructure
      ↓
Initialize Repositories
      ↓
Initialize Services
      ↓
Initialize Transport
      ↓
Start Server
```

Failures in required initialization should fail clearly.

Do not silently continue with partially initialized critical infrastructure.

---

# 39. Graceful Shutdown

The backend must shut down gracefully.

Shutdown should:

1. Stop accepting new work.
2. Allow in-flight requests to finish where appropriate.
3. Stop background workers.
4. Close message consumers.
5. Close database connections.
6. Close external clients.
7. Flush important telemetry/logs.
8. Exit within a bounded timeout.

Do not terminate active resources abruptly when graceful shutdown is possible.

---

# 40. Health and Readiness

Expose appropriate health checks.

### Liveness

Determines whether the process is alive.

### Readiness

Determines whether the process is ready to receive traffic.

Do not make liveness depend on every external service.

Readiness may fail when critical dependencies are unavailable.

---

# 41. Logging

Backend logs should be structured and useful.

Include where appropriate:

- Timestamp.
- Level.
- Service.
- Environment.
- Request ID.
- Correlation ID.
- Tenant context where safe.
- Operation.
- Error category.
- Duration.

Never log:

- Passwords.
- Access tokens.
- Refresh tokens.
- Secrets.
- Sensitive payloads unnecessarily.

---

# 42. Metrics

Important backend operations should expose metrics where appropriate.

Monitor:

- Request rate.
- Request latency.
- Error rate.
- Database latency.
- Queue depth.
- Job failures.
- External API latency.
- Cache hit rate.
- Worker utilization.
- Resource usage.

Metrics should help identify both immediate failures and gradual degradation.

---

# 43. Distributed Tracing

Use tracing for workflows that cross service or infrastructure boundaries.

Useful trace boundaries include:

```text id="9cgf5j"
HTTP Request
    ↓
Application Service
    ↓
Database
    ↓
External API
    ↓
Background Event
```

Trace context should propagate across supported boundaries.

Do not include sensitive payloads in traces.

---

# 44. Error Handling

Backend errors should be classified.

Examples:

```text id="zkl8pu"
Validation Error
Authorization Error
Not Found
Conflict
Dependency Failure
Timeout
Internal Error
```

Internal errors must not expose implementation details through APIs.

Expected domain errors should be represented explicitly.

Unexpected errors should be logged with sufficient context and mapped to safe external responses.

---

# 45. Panic Handling

Panics should not be used for ordinary application errors.

Use errors for expected failure.

Panics may be appropriate for unrecoverable programmer/configuration failures during startup, but runtime request handling should protect the server process from unintended panics where the framework supports recovery middleware.

Do not use panic as normal control flow.

---

# 46. Serialization

Serialization must be explicit and stable.

Consider:

- JSON field naming.
- Null behavior.
- Optional fields.
- Time representation.
- Numeric precision.
- Backward compatibility.

Do not expose internal Go structs directly when their fields are not intended as API contracts.

Use response DTOs where appropriate.

---

# 47. IDs

Backend identifiers should be:

- Unique.
- Stable.
- Non-semantic where possible.
- Safe to expose when required.
- Consistent across APIs and persistence.

Do not encode mutable business information into IDs.

---

# 48. Time Handling

Use UTC internally unless a domain requirement requires otherwise.

Business-local time must be explicit.

Do not rely on:

```text id="0f5h6h"
time.Local
```

for business-critical behavior.

Handle:

- Timezones.
- Day boundaries.
- Scheduling.
- Expiration.
- Clock skew.

---

# 49. File and Object Storage

Large files should generally use object storage rather than database blobs unless there is a clear reason otherwise.

The backend should control:

- Authorization.
- Upload policy.
- File metadata.
- Object ownership.
- Lifecycle.
- Expiration.
- Deletion.

Do not trust client-supplied storage paths.

---

# 50. Large Data Processing

Backend workloads involving large datasets must be bounded.

Prefer:

- Streaming.
- Chunking.
- Pagination.
- Batch processing.
- Worker pools.
- Incremental aggregation.

Avoid loading entire large datasets into memory.

For file-processing workloads, design around actual memory constraints rather than assuming unlimited RAM.

---

# 51. Batch Processing

Batch operations should define:

- Maximum batch size.
- Concurrency.
- Transaction boundaries.
- Failure behavior.
- Retry behavior.
- Idempotency.

Avoid unbounded batch sizes.

Do not process thousands of independent records sequentially when safe bounded concurrency provides a meaningful improvement.

---

# 52. API Handler Rules

Handlers should generally follow:

```text id="3om6tq"
Parse
  ↓
Validate
  ↓
Authenticate
  ↓
Call Application Service
  ↓
Map Result
  ↓
Respond
```

Handlers should not:

- Contain complex business logic.
- Query databases directly.
- Manage infrastructure clients.
- Perform large computations.
- Contain long-running workflows.

---

# 53. Dependency Injection

Dependencies should be explicit.

Prefer constructor injection:

```text id="m6zq0m"
NewOrderService(
    orderRepository,
    inventoryRepository,
    paymentService,
)
```

over hidden global dependencies.

Dependency injection should remain simple.

Do not introduce a dependency injection framework merely because constructor injection becomes repetitive.

---

# 54. Global State

Avoid mutable global state.

Global mutable state creates:

- Hidden dependencies.
- Race conditions.
- Difficult tests.
- Lifecycle problems.

Use explicit dependency ownership.

Read-only immutable configuration may be shared safely after initialization.

---

# 55. Package Initialization

Avoid meaningful side effects inside package initialization.

Do not perform:

- Network calls.
- Database connections.
- Background worker startup.
- Environment-dependent initialization.

inside `init()` unless explicitly justified.

Prefer explicit startup orchestration.

---

# 56. Resource Ownership

Every resource must have a clear owner.

Resources include:

- Database pools.
- Redis clients.
- HTTP clients.
- Workers.
- Timers.
- Consumers.
- File handles.

The owner is responsible for initialization and cleanup.

Avoid creating resources repeatedly inside request paths.

---

# 57. Backend Performance

Follow the system-wide performance rules.

Review:

- Algorithmic complexity.
- Database query count.
- Network calls.
- Serialization.
- Memory allocations.
- Goroutine count.
- Lock contention.
- Cache behavior.
- Connection pools.

Prefer performance improvements that reduce expensive work rather than micro-optimizing already cheap operations.

---

# 58. Backend Security

Backend code must enforce:

- Authentication.
- Authorization.
- Tenant isolation.
- Input validation.
- Secure secrets handling.
- Rate limiting where required.
- Safe error handling.
- Secure file handling.
- Dependency security.

Never rely on frontend enforcement.

---

# 59. Testing

Backend changes should have appropriate tests.

Test:

- Domain behavior.
- Application workflows.
- Repository behavior.
- API contracts.
- Authorization.
- Tenant isolation.
- Transactions.
- Concurrency.
- Idempotency.
- External integration boundaries.
- Failure behavior.

Tests should verify behavior rather than implementation details.

---

# 60. Backend Documentation

Document non-obvious backend architecture.

Document:

- Domain boundaries.
- Important workflows.
- Transaction decisions.
- Event flows.
- External integrations.
- Retry policies.
- Background jobs.
- Operational requirements.
- Deployment dependencies.

Do not document obvious implementation details that can be understood directly from clear code.

---

# 61. Backend Change Checklist

Before completing a backend change:

- [ ] Correct architectural layer selected.
- [ ] Domain ownership is clear.
- [ ] Business logic is in the appropriate layer.
- [ ] Dependencies flow correctly.
- [ ] Handler contains no unnecessary business logic.
- [ ] Application workflow is explicit.
- [ ] Domain invariants are protected.
- [ ] Repository boundaries are appropriate.
- [ ] Transactions are correctly scoped.
- [ ] Concurrency is safe.
- [ ] Goroutines have clear ownership.
- [ ] Background jobs are bounded and retry-safe.
- [ ] External calls have timeouts.
- [ ] Retry behavior is intentional.
- [ ] Idempotency is handled where required.
- [ ] Database access is efficient.
- [ ] Cache behavior is safe.
- [ ] OpenSearch remains a derived system.
- [ ] Authentication and authorization are enforced.
- [ ] Tenant isolation is preserved.
- [ ] Secrets are protected.
- [ ] Logs do not leak sensitive data.
- [ ] Metrics/tracing are sufficient.
- [ ] Tests cover important behavior.
- [ ] Documentation is updated where necessary.
- [ ] Final diff is focused.

---

# 62. Final Backend Principle

The backend should make business behavior explicit and infrastructure replaceable.

Prefer:

```text id="2cg4sm"
Transport
    ↓
Application
    ↓
Domain
    ↓
Interfaces
    ↑
Infrastructure
```

with:

```text id="n4m3qf"
Explicit dependencies
Explicit transactions
Explicit ownership
Explicit errors
Explicit concurrency
Explicit contracts
```

Avoid hidden behavior.

Avoid global state.

Avoid infrastructure leaking into the domain.

Avoid business logic leaking into handlers.

Build the backend so that another engineer can understand where a behavior belongs before changing it.
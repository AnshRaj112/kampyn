# KAMPYN Backend Services

## Purpose

Services coordinate application behavior between transport, domain logic, repositories, integrations, events, and infrastructure.

A service represents **application behavior**, not merely a collection of helper functions.

The preferred architecture is:

```text
Transport
    ↓
Application Service
    ↓
Domain
    ↓
Repositories / Infrastructure
```

A service may coordinate multiple dependencies:

```text
Application Service
 ├── Repository
 ├── Domain Service
 ├── External Integration
 ├── Event Publisher
 └── Cache
```

Services must remain focused on use cases and orchestration.

They must not become:

- Generic utility containers
- Database wrappers
- HTTP controllers
- Giant business-logic classes
- Global state containers
- Cross-domain dumping grounds

---

# 1. Core Principles

Services must:

- Represent meaningful application operations.
- Coordinate domain behavior.
- Enforce application-level workflow rules.
- Call repositories through defined contracts.
- Coordinate transactions.
- Coordinate external integrations.
- Publish events where required.
- Preserve tenant context.
- Respect authorization boundaries.
- Handle idempotency where required.
- Remain observable and testable.
- Keep dependencies explicit.

Services must not:

- Directly parse HTTP requests.
- Construct HTTP responses.
- Execute arbitrary SQL throughout the codebase.
- Contain framework-specific transport logic.
- Hide unrelated workflows.
- Become generic `Manager`, `Helper`, or `Util` classes.

---

# 2. Service Position in KAMPYN

The backend architecture is:

```text
┌──────────────────────────────┐
│          Transport           │
│ HTTP / WebSocket / Events    │
└──────────────┬───────────────┘
               ↓
┌──────────────────────────────┐
│       Application Service    │
│         Use Cases            │
└──────────────┬───────────────┘
               ↓
┌──────────────────────────────┐
│            Domain            │
│ Entities / Rules / Policies  │
└──────────────┬───────────────┘
               ↓
┌──────────────────────────────┐
│ Repositories / Integrations  │
└──────────────┬───────────────┘
               ↓
┌──────────────────────────────┐
│       Infrastructure         │
└──────────────────────────────┘
```

The service layer is primarily the **application orchestration boundary**.

---

# 3. What Is a Service?

A service performs a meaningful application operation.

Examples:

```text
CreateOrder
CancelOrder
ReserveInventory
CreateBooking
CancelBooking
ProcessPayment
ApproveComplaint
PublishAnnouncement
SearchFoodItems
CreateCommunityPost
ModerateCommunityContent
GenerateReport
```

These represent behavior that users or system processes care about.

---

# 4. Service vs Repository

Repositories answer:

```text
"How do I persist or retrieve this data?"
```

Services answer:

```text
"What application operation needs to happen?"
```

Example:

```text
CreateOrderService
    ↓
InventoryRepository
MenuRepository
OrderRepository
PaymentProvider
EventPublisher
```

The service coordinates the workflow.

The repositories only perform persistence operations.

---

# 5. Service vs Domain

The domain owns business rules.

The service coordinates those rules.

Example:

```text
OrderService
    ↓
Order.canCancel()
```

The service should not replace domain behavior with:

```text
if order.status == "PAID" {
    ...
}
```

when the state transition is a domain rule.

Prefer:

```text
Service
    ↓
Domain Entity
    ↓
Business Rule
```

---

# 6. Service vs Controller

Controllers handle transport.

Example:

```text
HTTP Request
    ↓
Handler
    ↓
Service
    ↓
Domain
```

The handler should not contain the complete workflow.

Bad:

```text
Handler
 ├── Validate inventory
 ├── Calculate price
 ├── Create order
 ├── Charge payment
 ├── Update inventory
 └── Send notification
```

Preferred:

```text
Handler
    ↓
CreateOrderService
    ↓
Complete Workflow
```

---

# 7. Service vs Middleware

Middleware handles cross-cutting request concerns.

Examples:

```text
Authentication
Tenant Resolution
Request ID
Tracing
Rate Limiting
Request Size
```

Services handle application behavior.

Do not move business workflows into middleware.

---

# 8. Service vs Background Job

A service represents the actual application operation.

A background job represents how that operation is executed asynchronously.

Example:

```text
Queue
  ↓
GenerateReportJob
  ↓
ReportService
  ↓
Repositories
```

The job should not duplicate the report business logic.

---

# 9. Service Naming

Names should represent meaningful actions.

Preferred:

```text
CreateOrderService
CancelOrderService
ReserveInventoryService
CreateBookingService
ProcessPaymentService
```

or, when the package already establishes context:

```text
OrderService.Create()
OrderService.Cancel()
```

Avoid:

```text
OrderManager
OrderHelper
OrderProcessor
CommonService
BusinessService
MainService
UtilityService
```

unless the name has a precise architectural meaning.

---

# 10. Use-Case-Oriented Services

For complex domains, prefer explicit use cases.

Example:

```text
orders/
├── create_order.go
├── cancel_order.go
├── confirm_order.go
├── refund_order.go
└── get_order.go
```

Each operation should have a clear responsibility.

This is preferable to:

```text
orders/
└── order_service.go
```

containing thousands of lines.

---

# 11. Service Cohesion

A service should have a focused responsibility.

Good:

```text
CreateOrderService
```

Bad:

```text
CampusService
```

containing:

```text
Orders
Bookings
Hostels
Complaints
Food Courts
Users
Notifications
```

The latter becomes a dependency and maintenance bottleneck.

---

# 12. Service Boundaries

A service boundary should generally align with:

- A business capability
- A use case
- An aggregate operation
- A meaningful workflow

It should not exist simply because a folder needs more files.

---

# 13. Application Service Responsibilities

Application services may:

- Load required data.
- Validate application-level conditions.
- Invoke domain behavior.
- Start/coordinate transactions.
- Persist changes.
- Call integrations.
- Create outbox records.
- Schedule jobs.
- Publish application events.
- Return application results.

---

# 14. What Services Should Not Own

Services should not directly own:

```text
HTTP status codes
HTTP headers
SQL syntax
Database driver objects
React state
Browser APIs
Framework-specific request objects
```

Those belong to their respective boundaries.

---

# 15. Service Dependencies

Dependencies must be explicit.

Example:

```go
type CreateOrderService struct {
    orders     OrderRepository
    inventory  InventoryRepository
    menu       MenuRepository
    payments   PaymentProvider
    events     EventPublisher
}
```

This makes dependencies visible and testable.

---

# 16. Dependency Direction

Preferred:

```text
Service
 ├── Repository Contract
 ├── Domain
 ├── Integration Contract
 └── Event Contract
```

Not:

```text
Service
 └── PostgreSQL Driver
```

or:

```text
Service
 └── Razorpay SDK
```

when a provider abstraction exists.

---

# 17. Dependency Injection

Dependencies should be supplied during construction.

Example:

```go
func NewCreateOrderService(
    orders OrderRepository,
    inventory InventoryRepository,
    payments PaymentProvider,
) *CreateOrderService {
    return &CreateOrderService{
        orders: orders,
        inventory: inventory,
        payments: payments,
    }
}
```

Avoid hidden dependency lookup.

---

# 18. Global Services

Avoid mutable global service instances containing:

```text
Current User
Current Tenant
Current Request
Current Transaction
```

Request-specific state must never be global.

Long-lived stateless service objects may be safely shared when their dependencies are safe for concurrent use.

---

# 19. Service Input

Service methods should receive explicit application inputs.

Example:

```go
type CreateOrderInput struct {
    TenantID   TenantID
    CustomerID UserID
    Items      []OrderItemInput
}
```

Avoid passing raw HTTP request objects into application services.

---

# 20. Service Output

Services should return application/domain results rather than transport-specific responses.

Example:

```go
type CreateOrderResult struct {
    OrderID OrderID
    Status  OrderStatus
    Total   Money
}
```

The transport layer converts this into an HTTP response.

---

# 21. DTOs

DTOs may be used at boundaries.

Example:

```text
HTTP Request DTO
        ↓
Application Input
        ↓
Domain
```

Do not automatically reuse HTTP DTOs as domain entities.

---

# 22. Validation

Services may perform application-level validation.

Examples:

```text
Required workflow inputs
Allowed operation combinations
Tenant context
Resource existence
Cross-resource conditions
```

Domain validation belongs to the domain.

Transport syntax validation belongs at the transport boundary.

---

# 23. Validation Layers

The preferred hierarchy is:

```text
Transport
 ↓
Input shape / syntax validation
 ↓
Application Service
 ↓
Application-level rules
 ↓
Domain
 ↓
Business invariants
 ↓
Database
 ↓
Persistence constraints
```

Each layer protects its own boundary.

---

# 24. Authorization

Services participate in authorization but should not invent authorization rules.

Preferred:

```text
Request
 ↓
Identity
 ↓
Tenant
 ↓
Authorization
 ↓
Service
 ↓
Domain
```

Resource-specific authorization may be evaluated by the application layer using the loaded resource.

---

# 25. Tenant Context

Tenant context must be explicit.

A service handling tenant-owned data should receive trusted tenant identity.

Example:

```go
type CreateOrderInput struct {
    TenantID   TenantID
    CustomerID UserID
}
```

Do not infer tenant identity from arbitrary user-provided fields.

---

# 26. Tenant Isolation

Every service must preserve tenant isolation across:

```text
Repositories
Cache
Search
Events
Jobs
Files
Notifications
Integrations
```

A service must never accidentally operate against another tenant's data.

---

# 27. Service Transactions

Services own application-level transaction boundaries.

Example:

```text
CreateOrder
    ↓
BEGIN
    ↓
Reserve Inventory
    ↓
Create Order
    ↓
Create Outbox Event
    ↓
COMMIT
```

The repository implements persistence behavior.

The service determines which operations must be atomic.

---

# 28. Transaction Scope

Transactions should contain only operations that must be atomic.

Do not include:

```text
HTTP requests
Payment provider calls
Email sending
Push notifications
Long computations
Large file processing
```

unless the architecture explicitly requires it.

---

# 29. External Calls

Services may coordinate external providers.

Example:

```text
PaymentService
    ↓
PaymentProvider
    ↓
External Payment API
```

External calls must have:

- Timeouts
- Retry policy
- Idempotency
- Error translation
- Observability

---

# 30. Payment Workflows

Payment services must explicitly model payment states.

Example:

```text
Created
 ↓
Pending
 ↓
Authorized
 ↓
Captured
 ↓
Failed
 ↓
Refunded
```

Do not infer payment state solely from whether an HTTP request succeeded.

---

# 31. Ambiguous External Results

A timeout does not necessarily mean an external operation failed.

For example:

```text
Service
 ↓
Payment Provider
 ↓
Request succeeds
 ↓
Response lost
```

The service must reconcile the state rather than blindly retrying an unsafe operation.

---

# 32. Idempotency

Services handling retriable operations must define idempotency.

Examples:

```text
CreateOrder
CreatePayment
ReserveInventory
ProcessWebhook
SendNotification
GenerateExport
```

Repeated execution should not accidentally create duplicate effects.

---

# 33. Idempotency Keys

Where appropriate:

```text
Client
 ↓
Idempotency Key
 ↓
Service
 ↓
Persistent Idempotency Record
 ↓
Operation
```

Idempotency must be durable when correctness depends on it.

---

# 34. State Transitions

Services must not arbitrarily mutate state.

Example:

```text
Pending
  ↓
Confirmed
  ↓
Completed
```

The domain should define valid transitions.

The service coordinates the transition.

---

# 35. Service Orchestration

A service may orchestrate multiple domain operations.

Example:

```text
CreateOrder
    │
    ├── Load Menu
    ├── Validate Items
    ├── Reserve Inventory
    ├── Create Order
    ├── Create Outbox Event
    └── Return Order
```

The orchestration should remain understandable.

---

# 36. Long Workflows

Long-running workflows should not remain inside a single synchronous service call.

Prefer:

```text
Service
 ↓
Persist State
 ↓
Event / Job
 ↓
Next Step
```

For example:

```text
BookingCreated
    ↓
PaymentJob
    ↓
PaymentCompleted
    ↓
BookingConfirmationJob
    ↓
Notification
```

---

# 37. Sagas / Workflow Coordination

When multiple systems must participate in a distributed workflow, use explicit workflow coordination.

Example:

```text
Booking
 ↓
Payment
 ↓
Room Allocation
 ↓
Notification
```

Failure handling must define compensation or recovery.

Do not pretend distributed operations are one database transaction.

---

# 38. Domain Services

Domain services are different from application services.

A domain service contains business logic that:

- Does not naturally belong to one entity.
- Requires domain concepts.
- Must remain independent of infrastructure.

Example:

```text
PricingPolicy
AvailabilityPolicy
FareCalculation
```

Domain services must not directly call:

```text
PostgreSQL
Redis
HTTP APIs
Email
```

---

# 39. Application vs Domain Service

Use an application service when the problem is:

```text
"How do I execute this use case?"
```

Use a domain service when the problem is:

```text
"What business rule determines this result?"
```

Example:

```text
Application Service
    ↓
Pricing Domain Service
    ↓
Price
```

---

# 40. Service Placement Decision

When adding logic, ask:

```text
Is this transport behavior?
    → Handler / Middleware

Is this workflow coordination?
    → Application Service

Is this business rule?
    → Domain / Domain Service

Is this persistence?
    → Repository

Is this external provider interaction?
    → Integration

Is this asynchronous execution?
    → Job / Worker

Is this cross-cutting?
    → Middleware / Infrastructure
```

Do not place code in a service merely because it does not obviously fit elsewhere.

---

# 41. Service Methods

Service methods should be small enough to understand.

A method should generally describe one use case.

Avoid:

```text
ProcessEverything()
```

or:

```text
HandleRequest()
```

containing unrelated behavior.

---

# 42. Service File Size

KAMPYN's production source-file guideline applies.

A service file should normally remain below:

```text
200 LOC
```

If a service exceeds this:

1. Determine whether multiple use cases exist.
2. Extract cohesive operations.
3. Extract domain behavior.
4. Extract infrastructure adapters.
5. Preserve meaningful boundaries.

Do not split files arbitrarily.

---

# 43. Service Complexity

Avoid unnecessary nested branching.

Prefer:

```text
Guard Conditions
 ↓
Clear Workflow
 ↓
Domain Operation
```

rather than deeply nested:

```text
if
 └── if
      └── if
           └── if
```

Complexity should reflect actual business complexity, not poor structure.

---

# 44. O(n²) Service Logic

Avoid unnecessary O(n²) operations.

Example:

```go
for _, order := range orders {
    for _, user := range users {
        ...
    }
}
```

Prefer indexed lookup:

```text
users
 ↓
Map<UserID, User>
 ↓
O(1) lookup
```

unless the dataset is intentionally tiny and the simpler implementation is justified.

---

# 45. Batch Operations

Services should use batch operations when processing large datasets.

Avoid:

```text
for each item:
    database query
```

when the operation can safely use:

```text
batch load
 ↓
in-memory index
 ↓
batch write
```

while maintaining bounded memory.

---

# 46. Service Concurrency

Services may perform independent operations concurrently when safe.

Example:

```text
Load User
Load Menu
Load Configuration
```

may execute concurrently.

But concurrency must remain bounded.

Never introduce concurrency merely because the language supports it.

---

# 47. Go Concurrency

Go services must:

- Respect `context.Context`.
- Avoid goroutine leaks.
- Bound worker counts.
- Handle cancellation.
- Propagate errors.
- Avoid shared mutable state.
- Use synchronization deliberately.

See:

```text
backend/concurrency.md
```

for detailed concurrency rules.

---

# 48. Node.js Concurrency

Node.js services must:

- Avoid unbounded `Promise.all`.
- Avoid event-loop blocking.
- Use streams for large data.
- Use bounded concurrency.
- Cancel abandoned work where supported.

---

# 49. Service Timeouts

Service operations should have explicit execution boundaries.

A service should not allow a request to remain active indefinitely because:

```text
database
external provider
queue
file storage
```

failed to respond.

---

# 50. Retry Behavior

Retries should happen only where appropriate.

A service should consider:

```text
Is the operation idempotent?
Is the failure transient?
Could the operation have succeeded?
Will retry duplicate side effects?
```

Do not blindly retry all errors.

---

# 51. Error Handling

Services should translate lower-level failures into application-level semantics.

Example:

```text
Repository
 ↓
Database Timeout
 ↓
Service
 ↓
DependencyUnavailable
 ↓
Transport
 ↓
HTTP 503
```

Do not expose database driver errors.

---

# 52. Business Errors

Business failures should have stable meanings.

Examples:

```text
OrderNotFound
InsufficientInventory
BookingUnavailable
InvalidOrderState
PaymentRequired
PermissionDenied
TenantSuspended
```

Avoid relying on human-readable error strings for application control flow.

---

# 53. Error Wrapping

Preserve underlying causes where useful.

In Go:

```go
fmt.Errorf("reserve inventory: %w", err)
```

allows higher layers to inspect the original cause.

Do not expose the internal error directly to users.

---

# 54. Service Return Semantics

Services should make result semantics explicit.

For example:

```text
Success
NotFound
Conflict
ValidationError
Forbidden
DependencyFailure
```

The transport layer converts these into protocol-specific responses.

---

# 55. Service Observability

Services are important tracing boundaries.

A trace should make the workflow visible:

```text
HTTP Request
    ↓
CreateOrderService
    ↓
InventoryRepository
    ↓
OrderRepository
    ↓
Outbox
```

Use meaningful operation names.

---

# 56. Service Metrics

Useful service metrics include:

```text
service_operation_total
service_operation_duration
service_operation_errors
service_operation_conflicts
service_operation_retries
```

Metrics must use bounded labels.

Do not label metrics by arbitrary:

```text
userId
orderId
email
requestId
```

---

# 57. Logging

Service logs should provide useful context:

```text
operation
tenantId
requestId
traceId
result
duration
error classification
```

Never log:

```text
passwords
tokens
payment credentials
private messages
sensitive personal data
```

unless explicitly required and protected by policy.

---

# 58. Audit Events

Security-sensitive service operations may produce audit records.

Examples:

```text
RoleChanged
UserSuspended
PaymentRefunded
TenantConfigurationChanged
AdminAccessGranted
```

Audit recording should be explicit and durable.

---

# 59. Events

Services may publish domain/application events.

Example:

```text
OrderService
    ↓
OrderCreated
```

Events should describe facts that have occurred.

Avoid events that are merely hidden commands.

---

# 60. Event Consistency

When state mutation and event publication must be atomic:

```text
Service
 ↓
Transaction
 ├── Repository mutation
 └── Outbox event
 ↓
Commit
```

Do not publish the event before the transaction succeeds.

---

# 61. Service and Cache

Services may coordinate cache behavior.

However:

```text
Database
```

remains authoritative for transactional state unless explicitly documented otherwise.

Cache invalidation should follow the established caching architecture.

---

# 62. Service and Search

Services should not synchronously update OpenSearch after every database mutation unless required.

Preferred:

```text
Service
 ↓
Database + Outbox
 ↓
Event
 ↓
Search Consumer
 ↓
OpenSearch
```

This keeps transactional operations isolated from search availability.

---

# 63. Notifications

Services may request notifications:

```text
OrderService
    ↓
NotificationService
    ↓
Email/SMS/Push Adapter
```

Do not directly embed provider-specific notification code in unrelated business services.

---

# 64. Notification Service

A notification service may coordinate:

```text
Template
Recipient
Channel
Provider
Delivery
Retry
```

The actual provider implementation belongs to integrations.

---

# 65. Service and Background Jobs

Services should be reusable from:

```text
HTTP requests
Background jobs
Event consumers
CLI operations
Scheduled tasks
```

This prevents business logic from being duplicated across execution mechanisms.

---

# 66. Jobs Must Not Duplicate Services

Bad:

```text
HTTP Order Logic
        +
Background Order Logic
```

with two implementations.

Preferred:

```text
HTTP Handler
    ↓
Order Service
```

and:

```text
Background Job
    ↓
Order Service
```

---

# 67. Service and WebSockets

WebSocket handlers should call application services.

Example:

```text
WebSocket Message
    ↓
Validate
    ↓
Authorize
    ↓
Chat Service
    ↓
Repository / Event
```

Do not implement chat business rules directly inside the socket connection handler.

---

# 68. Community Services

KAMPYN community functionality may include:

```text
CreatePost
EditPost
DeletePost
ReportPost
ModeratePost
Comment
React
JoinCommunity
LeaveCommunity
```

These should remain separate application operations rather than one giant `CommunityService`.

---

# 69. Chat Services

Private/community chat services must coordinate:

```text
Identity
Tenant
Conversation
Membership
Message
Moderation
Persistence
Realtime Event
```

Authorization must be evaluated before message access or mutation.

---

# 70. File Services

File-related services should coordinate:

```text
Authorization
Metadata
Object Storage
Upload Session
Validation
Scanning
Download Access
Deletion
```

Do not pass raw file uploads through unnecessary service layers.

Large files should use streaming or direct object-storage upload mechanisms where appropriate.

---

# 71. Search Services

A search service may coordinate:

```text
Query validation
Tenant filtering
Authorization
Search repository
Ranking parameters
Pagination
```

The service should not expose arbitrary OpenSearch DSL to clients.

---

# 72. Report Services

Report services should determine:

```text
What report is requested
Who can access it
What data is required
Whether generation is synchronous
Whether a background job is required
Where the result is stored
```

Large reports should generally be generated asynchronously.

---

# 73. Import Services

Import services should coordinate:

```text
File validation
Parsing
Schema validation
Normalization
Deduplication
Persistence
Error reporting
```

Large imports should use background jobs and bounded processing.

---

# 74. Export Services

Export services should:

```text
Validate request
Authorize
Create export job
Process bounded batches
Write output
Store artifact
Notify user
```

Do not generate huge exports inside normal HTTP request lifetimes.

---

# 75. Service Dependencies and Cycles

Services must not create circular dependencies.

Bad:

```text
OrderService
   ↓
PaymentService
   ↓
OrderService
```

Possible alternatives:

```text
OrderService
    ↓
PaymentProvider
```

or:

```text
OrderCreated
    ↓
PaymentWorkflow
```

Use events/workflows when direct coupling creates cycles.

---

# 76. Cross-Domain Service Calls

Cross-domain calls must be intentional.

For example:

```text
OrderService
    ↓
InventoryService
```

may be appropriate when the inventory operation is an application capability.

But avoid creating:

```text
OrderService
 ↔ InventoryService
 ↔ BookingService
 ↔ UserService
 ↔ PaymentService
```

with unrestricted synchronous calls.

This creates a distributed monolith.

---

# 77. Domain Ownership

Each service must respect domain ownership.

Example:

```text
InventoryService
    → owns inventory behavior

OrderService
    → owns order workflow

PaymentService
    → owns payment workflow
```

A service should request another domain's capability instead of directly modifying its data.

---

# 78. Service Composition

Composition should be explicit.

Example:

```text
CreateOrderService
    ↓
InventoryService.Reserve()
    ↓
Order aggregate
    ↓
OrderRepository.Save()
```

If the workflow becomes distributed, move toward events/workflows rather than deep synchronous chains.

---

# 79. Service Discovery

Services should not dynamically discover arbitrary internal services from user-controlled input.

Internal service endpoints should come from:

- Configuration
- Service discovery
- Cluster DNS
- Explicit infrastructure configuration

---

# 80. Configuration

Service configuration should be loaded centrally and validated at startup.

Avoid scattered:

```go
os.Getenv(...)
```

throughout application code.

Preferred:

```text
Environment
 ↓
Config
 ↓
Dependencies
 ↓
Services
```

---

# 81. Service Lifecycle

Long-lived services should be initialized during application startup.

Example:

```text
Configuration
 ↓
Infrastructure Clients
 ↓
Repositories
 ↓
Services
 ↓
Handlers
 ↓
Server
```

Shutdown should happen in reverse dependency order.

---

# 82. Service Testing

Application services should have strong unit tests.

Test:

- Successful workflows.
- Validation failures.
- Domain rule failures.
- Repository failures.
- Integration failures.
- Authorization failures.
- Tenant isolation.
- Idempotency.
- Concurrency-sensitive operations.
- Event behavior.

---

# 83. Mocking

Mock dependencies at architectural boundaries.

Good candidates:

```text
Repository
PaymentProvider
NotificationProvider
EventPublisher
Clock
ID Generator
```

Do not mock every internal function.

Over-mocking makes tests validate implementation rather than behavior.

---

# 84. Service Integration Tests

Important workflows should also be tested against real infrastructure.

Examples:

```text
Service
 ↓
PostgreSQL
 ↓
Transaction
 ↓
Outbox
```

or:

```text
Service
 ↓
Redis
 ↓
Cache behavior
```

---

# 85. Service Contract Tests

Services consuming external providers should have contract tests where appropriate.

Verify:

```text
Request shape
Response mapping
Error mapping
Timeout behavior
Webhook behavior
```

---

# 86. Service Performance

Service performance must be measured at the workflow level.

Look for:

```text
Repeated queries
N+1 access
Unnecessary serialization
Sequential independent calls
Unbounded concurrency
Large memory allocations
Slow external dependencies
```

---

# 87. Service Caching

Do not cache entire service methods blindly.

Cache based on:

```text
Data volatility
Read frequency
Consistency requirements
Tenant scope
Invalidation strategy
```

The cache architecture must be explicit.

---

# 88. Service Security

Services must enforce:

```text
Authentication context
Tenant isolation
Authorization
Input validation
Rate limits where appropriate
Idempotency
Audit requirements
Sensitive-data handling
```

Never assume that because a request reached the service it is authorized.

---

# 89. Service-to-Service Authentication

Internal service calls must use authenticated service identity where required.

Do not trust:

```text
internal network
private IP
Kubernetes namespace
```

as the sole authorization mechanism.

---

# 90. Service-to-Service Authorization

A service should only have the permissions required for its role.

Prefer:

```text
Order Service
    → order permissions
    → required inventory capability
```

rather than granting unrestricted database access.

---

# 91. Database Access from Services

Services should normally access databases through repositories.

Avoid:

```text
Service
 ↓
SQL driver
```

when the repository boundary already exists.

Exceptions require architectural justification.

---

# 92. Service and Transactions

Services should make transaction boundaries visible in code.

A reviewer should be able to answer:

```text
Which operations are atomic?
What happens if step 3 fails?
What happens if the request times out?
Can the operation be retried?
```

without reverse-engineering multiple layers.

---

# 93. Service and Partial Failure

Distributed workflows must define partial failure behavior.

Example:

```text
Order Created
 ↓
Payment Pending
 ↓
Notification Failed
```

The order should not necessarily be rolled back simply because notification failed.

Each dependency's criticality must be defined.

---

# 94. Critical vs Non-Critical Dependencies

Services should distinguish:

### Critical

Failure prevents the operation.

Examples:

```text
Inventory reservation
Payment authorization
Required database write
```

### Non-Critical

Failure can be retried asynchronously.

Examples:

```text
Email
Push notification
Analytics
Search indexing
```

Do not make non-critical systems synchronous blockers unnecessarily.

---

# 95. Service Resilience

Services must handle:

```text
timeouts
retries
backpressure
circuit breaking where appropriate
dependency failure
partial failure
duplicate execution
```

Do not add retries and circuit breakers blindly.

They must correspond to actual failure modes.

---

# 96. Backpressure

Services processing large volumes must avoid overwhelming dependencies.

Use:

```text
bounded workers
queues
batching
rate limits
concurrency limits
```

Backpressure must be observable.

---

# 97. Rate Limits

Rate limiting may exist at:

```text
Gateway
Middleware
Service
Provider adapter
```

The service should still protect expensive operations when necessary.

Examples:

```text
OTP generation
Report generation
Search
File processing
Bulk imports
```

---

# 98. Service Documentation

Every significant service should document:

- Responsibility.
- Input.
- Output.
- Dependencies.
- Transaction boundary.
- Authorization assumptions.
- Tenant behavior.
- Events emitted.
- Jobs created.
- External calls.
- Failure behavior.
- Idempotency requirements.

---

# 99. Service Change Process

Before changing a service:

1. Read relevant `.ai/` architecture documents.
2. Inspect the service implementation.
3. Search for all callers.
4. Search for related repositories.
5. Search for emitted/consumed events.
6. Identify transaction boundaries.
7. Identify tenant/auth requirements.
8. Identify external dependencies.
9. Identify tests.
10. Make the smallest coherent change.

---

# 100. Avoid Premature Service Extraction

Do not turn every function into a service.

A service should exist when there is a meaningful application boundary.

Avoid:

```text
EmailFormatterService
StringService
DateService
ArrayService
```

unless there is an actual architectural reason.

Simple pure functions should remain simple functions.

---

# 101. Avoid Service Locator Patterns

Do not create:

```go
services.Get("OrderService")
```

and retrieve arbitrary services dynamically.

This hides dependencies and makes reasoning difficult.

Prefer explicit dependency injection.

---

# 102. Avoid God Services

A service containing:

```text
Orders
Users
Payments
Bookings
Food
Hostels
Notifications
Community
Analytics
```

is a strong architectural smell.

Split by capability and ownership.

---

# 103. Avoid Pass-Through Services

This adds little value:

```text
Service
 ↓
Repository
```

when the service does nothing except expose the repository.

If no application behavior exists, consider whether the service layer is actually needed.

---

# 104. Avoid Business Logic in Repositories

This is equally problematic:

```text
Repository
 ↓
Calculate price
 ↓
Check business rules
 ↓
Send notification
```

Repositories must remain persistence-focused.

---

# 105. Avoid Infrastructure Leakage

Application services should not depend directly on:

```text
*sql.DB
Mongo Client
Redis Client
OpenSearch Client
HTTP framework request
Payment SDK
```

unless that dependency is explicitly an infrastructure boundary.

---

# 106. Service Result Ownership

Services should return results appropriate for application consumers.

They should not return:

```text
*sql.Row
mongo.Document
HTTP Response
Framework Context
```

Instead return domain/application types.

---

# 107. Service and API Versioning

API versioning belongs to the API boundary.

A service should generally remain version-neutral.

For example:

```text
API v1
   ↓
CreateOrderService
```

and:

```text
API v2
   ↓
CreateOrderService
```

when both versions can safely share the same application behavior.

---

# 108. Service and Compatibility

If API versions require materially different behavior:

```text
v1 adapter
   ↓
Application Service
```

or:

```text
v2 adapter
   ↓
Application Service
```

should be preferred over duplicating the complete business workflow.

---

# 109. Service and SDKs

SDKs must call APIs.

They should not contain copies of backend business rules.

Example:

```text
SDK
 ↓
KAMPYN API
 ↓
Application Service
 ↓
Domain
```

Client-side convenience validation is acceptable, but the backend remains authoritative.

---

# 110. Service and Self-Hosting

Services must not depend on SaaS-only infrastructure unless the self-hosted architecture explicitly provides it.

External dependencies should have:

```text
configuration
provider abstraction
self-hosting strategy
failure behavior
```

---

# 111. Service and Multi-Tenant SaaS

For SaaS deployments, services must handle:

```text
tenant context
tenant configuration
tenant feature flags
tenant limits
tenant integrations
tenant branding
```

without mixing tenants.

---

# 112. Tenant Feature Flags

Services may check tenant feature configuration when a feature is tenant-dependent.

Prefer:

```text
Tenant Configuration
 ↓
Application Policy
 ↓
Service Behavior
```

Do not scatter raw feature-flag checks throughout domain code.

---

# 113. Service Quotas

Services responsible for resource-heavy operations should enforce appropriate quotas.

Examples:

```text
Report generation
File uploads
Community posting
Search
Bulk imports
Notifications
```

Quota decisions should be explicit and tenant-aware.

---

# 114. Service State

Prefer stateless services.

Persistent workflow state should live in:

```text
PostgreSQL
MongoDB
Redis where explicitly appropriate
```

Do not depend on process-local state for correctness.

---

# 115. Restart Safety

A service must remain correct after:

```text
process restart
pod restart
deployment
machine failure
```

Any state required to continue a workflow must be durable.

---

# 116. Service Shutdown

Services must stop accepting new work and allow in-flight operations to complete within a bounded shutdown window.

For background processing:

```text
Stop intake
 ↓
Finish safe work
 ↓
Acknowledge completed work
 ↓
Release resources
```

---

# 117. Service Recovery

A service must be able to recover from:

```text
database restart
Redis restart
pod restart
network interruption
external provider timeout
worker failure
```

without manual data repair for normal transient failures.

---

# 118. Service State Machines

When a workflow has multiple states, represent them explicitly.

Example:

```text
Order:
Pending
Confirmed
Preparing
Ready
Completed
Cancelled
```

Transitions should be controlled by domain rules.

Services invoke transitions rather than directly mutating arbitrary status values.

---

# 119. Service Ownership Matrix

KAMPYN should maintain clear ownership:

```text
Capability              Owner

Orders                  Order Service
Inventory               Inventory Service
Bookings                Booking Service
Payments                Payment Service
Users                   Identity/User Service
Notifications           Notification Service
Community               Community Service
Chat                    Chat Service
Search                  Search Service
Reports                 Report Service
Files                   File Service
Tenant Management       Tenant Service
```

This is a conceptual ownership model; the actual service boundaries should follow the deployed architecture.

---

# 120. Definition of Done

A service implementation is complete only when:

- It represents a meaningful application operation.
- Its responsibility is clearly documented.
- Its dependencies are explicit.
- It uses repository contracts rather than database drivers.
- Domain rules remain in the domain.
- Transport logic remains outside the service.
- Authorization assumptions are explicit.
- Tenant isolation is preserved.
- Transaction boundaries are clear.
- External calls have timeout and failure semantics.
- Idempotency is defined where required.
- Concurrency is bounded.
- Large operations use jobs/streaming where necessary.
- Events are published through the established event architecture.
- Cache/search behavior is explicit.
- Errors are stable and meaningful.
- Observability exists.
- Security requirements are satisfied.
- Unit and integration tests exist where appropriate.
- Documentation is updated.

---

# 121. Final Invariants

```text
1. Services represent application behavior and workflows.

2. Repositories represent persistence.

3. Domain objects and domain services represent business rules.

4. Controllers and handlers represent transport concerns.

5. Middleware represents cross-cutting request concerns.

6. Background jobs represent asynchronous execution.

7. External providers are accessed through integration boundaries.

8. Services must have explicit dependencies.

9. Services must not depend on HTTP framework objects.

10. Services must not directly expose database-driver types.

11. Services must not become generic utility containers.

12. Services must not become god objects.

13. Services must not duplicate domain business rules.

14. Services coordinate domain behavior rather than replacing it.

15. Transaction boundaries must be explicit.

16. External network calls must not unnecessarily occur inside
    database transactions.

17. Retried operations must have deliberate idempotency semantics.

18. Services must preserve tenant isolation.

19. Services must respect authorization boundaries.

20. Services must not trust user-provided tenant identifiers without
    trusted tenant resolution.

21. Services must not perform unbounded concurrent work.

22. Services must respect request and operation cancellation.

23. Long-running workflows should move to jobs/events/workflows.

24. Non-critical side effects should not unnecessarily block critical
    transactional operations.

25. Services must remain restart-safe.

26. Services must not rely on process-local state for correctness.

27. Service dependencies must not form uncontrolled circular chains.

28. Cross-domain operations must respect domain ownership.

29. Services should be reusable from HTTP, jobs, events, and other
    approved execution mechanisms.

30. Services should be observable at meaningful workflow boundaries.

31. Service methods should have explicit input and output contracts.

32. Application errors must remain independent of transport protocols.

33. Service files should normally remain below KAMPYN's 200 LOC guideline.

34. New services require a meaningful architectural responsibility.

35. A service that merely forwards every call to a repository should
    be reconsidered.

36. A service that begins accumulating unrelated domains must be split
    or its boundary reviewed.

37. The service layer exists to make application workflows explicit,
    testable, secure, and maintainable.
```

## KAMPYN Service Model

```text
                         TRANSPORT
                    HTTP / WS / Events
                           │
                           ▼
                  ┌─────────────────┐
                  │ APPLICATION     │
                  │    SERVICES     │
                  └────────┬────────┘
                           │
            ┌──────────────┼──────────────┐
            ▼              ▼              ▼
       Order Service  Booking Service  Community
            │              │              │
            └──────────────┼──────────────┘
                           ▼
                     DOMAIN RULES
                           │
            ┌──────────────┼──────────────┐
            ▼              ▼              ▼
       Repositories    Integrations     Events
            │              │              │
            ▼              ▼              ▼
       PostgreSQL /     Providers      Outbox /
       MongoDB /        Payments /     Event Bus
       Redis            Email / etc.
            │
            ▼
       Infrastructure
```

The governing principle is:

```text
Services coordinate.

Domains decide.

Repositories persist.

Integrations communicate.

Jobs execute asynchronously.

Transport exposes the system.

No layer should silently take ownership of another layer's responsibility.
```
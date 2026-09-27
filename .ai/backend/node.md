# KAMPYN Backend Node.js Engineering

## Purpose

KAMPYN's primary backend runtime is Go.

Node.js must therefore be used selectively and must not become an accidental second backend architecture.

Approved Node.js use cases may include:

- Next.js application server functionality
- Frontend build tooling
- SDK tooling
- Code generation
- Development scripts
- Documentation tooling
- Build-time services
- Small supporting services where Node.js provides a concrete advantage
- Explicitly approved Node.js microservices

The default rule is:

```text
Core Backend Business Logic
        ↓
       Go
```

Node.js must not duplicate the Go backend merely because the JavaScript/TypeScript ecosystem is convenient.

---

# 1. Core Principles

Node.js code must be:

- Type-safe
- Explicit
- Modular
- Secure
- Observable
- Testable
- Resource-conscious
- Asynchronous where appropriate
- Consistent with KAMPYN architecture

The same architectural principles apply regardless of runtime:

```text
Transport
   ↓
Application
   ↓
Domain
   ↓
Infrastructure
```

---

# 2. When Node.js Is Appropriate

Node.js is appropriate when it provides a meaningful advantage.

Examples:

```text
Next.js server functionality
Frontend tooling
TypeScript SDK tooling
Code generation
Build pipelines
Developer tooling
JavaScript-specific integrations
```

Node.js may also be used for a standalone service when:

1. The service has a clearly defined responsibility.
2. The service benefits materially from Node.js.
3. Its API contract is explicit.
4. Its ownership is clear.
5. Its deployment is independent.
6. Its data ownership is explicit.
7. It does not duplicate Go business logic.

---

# 3. When Node.js Is Not Appropriate

Do not create a Node.js service simply because:

```text
"Node is easier."
"Everyone knows JavaScript."
"The frontend already uses TypeScript."
```

Do not duplicate:

```text
Order logic
Payment logic
Inventory logic
Booking logic
Authorization logic
Tenant logic
```

between Go and Node.js.

If the same business rule exists in multiple runtimes, the architecture is likely becoming inconsistent.

---

# 4. Primary Backend Boundary

KAMPYN's primary backend remains:

```text
Client
   ↓
API
   ↓
Go Backend
   ↓
Application
   ↓
Domain
   ↓
Infrastructure
```

A Node.js service should normally appear as an explicit supporting component:

```text
Go Backend
     │
     ▼
Node.js Service
     │
     ▼
Specific Capability
```

It must not silently become another general-purpose API layer.

---

# 5. Node.js and Next.js

Next.js may execute server-side JavaScript/TypeScript.

This does not automatically make Next.js the KAMPYN business backend.

The preferred separation is:

```text
Next.js
 ├── UI rendering
 ├── Server Components
 ├── Frontend data fetching
 ├── Metadata
 └── Frontend-specific server functionality
          │
          ▼
      Go API
          │
          ▼
      KAMPYN Backend
```

Next.js should not independently recreate Go application services.

---

# 6. Next.js API Routes

Next.js API routes or route handlers may be used for frontend-specific concerns where appropriate.

Examples:

```text
Frontend-specific proxy
Browser-facing callback
Next.js-specific server behavior
Frontend framework integration
```

They should not become a replacement for the primary Go API.

Avoid:

```text
Next.js API
     +
Go API
     +
duplicated business logic
```

---

# 7. Business Logic Ownership

Business logic must have one authoritative owner.

Example:

```text
Order Creation
     ↓
Go Application
     ↓
Order Domain
```

A Next.js route should not independently implement:

```text
Validate inventory
Calculate order total
Create payment
Update order
```

Instead:

```text
Next.js
   ↓
Go API
   ↓
Order Application
```

---

# 8. TypeScript

All Node.js application code should use TypeScript unless there is a documented reason not to.

Avoid new JavaScript-only backend code.

Preferred:

```text id="4x4zqv"
.ts
.tsx
```

rather than:

```text id="r5s7e1"
.js
```

TypeScript must use strict type checking.

---

# 9. TypeScript Configuration

Node.js TypeScript projects must use a strict configuration.

Recommended baseline:

```json id="h7d9r2"
{
  "compilerOptions": {
    "strict": true
  }
}
```

The actual repository configuration is authoritative.

Do not weaken strictness merely to make existing code compile.

---

# 10. Package Management

Use the package manager defined by the repository.

Examples may include:

```text id="f5d2e4"
pnpm
npm
yarn
```

Do not introduce another package manager into the same project without explicit justification.

The lockfile must be committed where repository policy requires it.

---

# 11. Dependencies

Before adding an npm dependency:

1. Search existing code.
2. Check whether the standard platform API solves the problem.
3. Check existing dependencies.
4. Evaluate maintenance.
5. Evaluate security.
6. Evaluate bundle/runtime impact.
7. Verify license compatibility where relevant.
8. Confirm the dependency has a real architectural purpose.

Avoid dependency accumulation.

---

# 12. Node.js Package Structure

A standalone Node.js service should follow clear boundaries.

Example:

```text id="q1k2t3"
service/
├── src/
│   ├── transport/
│   ├── application/
│   ├── domain/
│   ├── infrastructure/
│   ├── config/
│   └── main.ts
│
├── tests/
├── package.json
├── tsconfig.json
└── package-lock.json / pnpm-lock.yaml
```

The exact structure may vary, but dependency direction must remain clear.

---

# 13. Domain-Oriented Structure

For larger Node.js services:

```text id="j6w9v2"
src/
├── orders/
├── notifications/
├── search/
├── reports/
└── integrations/
```

should be preferred over a collection of unrelated global folders.

Do not create:

```text id="n8s3c4"
utils/
helpers/
misc/
common/
```

as dumping grounds.

---

# 14. File Size

Production Node.js/TypeScript source files should normally follow KAMPYN's:

```text id="m2k8v7"
200 LOC guideline
```

A file exceeding this should trigger review.

Do not split files artificially.

Instead determine whether multiple responsibilities have been combined.

---

# 15. Function Size

Functions should remain focused.

Avoid:

```text id="q5h3k9"
500-line controller
```

or:

```text id="a6j2p8"
1000-line service
```

Prefer:

```text id="v3c7n1"
Validate
 ↓
Load
 ↓
Execute
 ↓
Persist
 ↓
Return
```

---

# 16. Controllers / Route Handlers

Route handlers should remain thin.

Preferred:

```text id="g5x8c2"
Request
 ↓
Parse
 ↓
Validate
 ↓
Authentication Context
 ↓
Application Use Case
 ↓
Response
```

Do not place business logic directly inside route handlers.

---

# 17. Application Layer

Application services coordinate use cases.

Example:

```text id="d8m1s6"
CreateOrder
CancelOrder
ReserveInventory
GenerateReport
```

They may coordinate:

- Repositories
- Domain operations
- Transactions
- Events
- Integrations

They must not become giant procedural scripts.

---

# 18. Domain Layer

Domain code should remain independent of:

- Express
- Next.js
- Fastify
- PostgreSQL clients
- MongoDB drivers
- Redis clients
- Provider SDKs

The domain should represent KAMPYN business concepts.

---

# 19. Infrastructure

Infrastructure contains:

```text id="w4q6s9"
Database adapters
Redis
OpenSearch
Object storage
External integrations
Message queues
Provider SDKs
```

Infrastructure should implement application-defined contracts where appropriate.

---

# 20. Dependency Direction

Preferred:

```text id="z6y8p3"
Transport
   ↓
Application
   ↓
Domain
   ↑
Infrastructure
```

Infrastructure must not force domain code to depend on provider-specific implementations.

---

# 21. Interfaces

TypeScript interfaces may define application contracts.

Example:

```ts id="r7x2m5"
interface NotificationProvider {
  send(
    request: NotificationRequest,
  ): Promise<NotificationResult>;
}
```

Do not create interfaces merely because TypeScript supports them.

Use abstractions at meaningful architectural boundaries.

---

# 22. Dependency Injection

Dependencies should be explicit.

Preferred:

```ts id="b3n7k2"
const service = new OrderService(
  orderRepository,
  paymentProvider,
);
```

Avoid hidden global service locators.

Avoid modules that initialize unrelated infrastructure merely because they are imported.

---

# 23. Global State

Avoid mutable global state.

Bad:

```ts id="y4c8m1"
let currentTenant: string;
let currentUser: User;
```

This is especially dangerous in server environments where many requests execute concurrently.

Request state must remain request-scoped.

---

# 24. Request Context

Request-scoped data should be explicitly passed or stored in a carefully controlled request context mechanism.

Possible context:

```text id="p2k6r8"
requestId
traceId
identity
tenantId
correlationId
```

Do not use global variables as request context.

---

# 25. Async/Await

Use `async`/`await` for asynchronous operations.

Preferred:

```ts id="t5v8q1"
const user = await repository.findById(id);
```

Avoid deeply nested promise chains when `async`/`await` improves readability.

---

# 26. Promise Handling

Every promise must have deliberate error handling.

Avoid:

```ts id="x8c2v6"
someOperation();
```

when the promise can reject and the rejection is not intentionally handled.

Unhandled promise rejections must not be ignored.

---

# 27. Sequential vs Parallel Operations

Do not accidentally serialize independent operations.

Bad:

```ts id="k7m2p9"
const user = await getUser();
const settings = await getSettings();
const menu = await getMenu();
```

when all three are independent.

Potentially:

```ts id="n4r6x8"
const [user, settings, menu] = await Promise.all([
  getUser(),
  getSettings(),
  getMenu(),
]);
```

However, only parallelize when:

- Operations are independent
- Downstream capacity allows it
- Failure semantics are understood

---

# 28. Promise.all Failure Behavior

`Promise.all()` fails when one operation rejects.

If partial success is meaningful, use an appropriate mechanism such as:

```text id="h2v5q9"
Promise.allSettled()
```

Do not accidentally discard successful results when partial failure is expected.

---

# 29. Bounded Concurrency

Never launch unlimited promises.

Bad:

```ts id="r9k3v6"
await Promise.all(
  items.map(item => process(item))
);
```

for an unbounded dataset.

Prefer bounded concurrency:

```text id="w2n7c4"
Items
 ↓
Batch / Concurrency Limit
 ↓
Workers
```

This protects:

- Memory
- Database
- APIs
- CPU
- External providers

---

# 30. Node.js Event Loop

Node.js uses an event-driven execution model.

Do not block the event loop with expensive synchronous operations.

Avoid synchronous operations such as:

```text id="c6m4p1"
Large synchronous file processing
Huge JSON parsing
CPU-heavy loops
Synchronous cryptographic operations
Large compression operations
```

on request paths unless explicitly justified.

---

# 31. CPU-Heavy Work

CPU-heavy workloads should not block request processing.

Possible approaches:

```text id="y8k2q6"
Worker Threads
Separate Service
Background Job
Go Backend
```

For KAMPYN's primary backend, Go is generally preferred for CPU-heavy backend workloads.

---

# 32. Large JSON

Avoid parsing enormous JSON payloads into memory when streaming or bounded processing is possible.

Request limits must exist.

For large datasets:

```text id="p7m3x9"
Stream
 ↓
Parse incrementally
 ↓
Process bounded chunks
```

rather than:

```text id="s5n8k2"
Entire file
 ↓
JSON.parse()
 ↓
Huge object tree
```

---

# 33. File Processing

Node.js file processing must use streams for large files.

Preferred:

```text id="f4q8m1"
Object Storage
 ↓
Readable Stream
 ↓
Transform
 ↓
Writable Stream
```

Avoid loading large files entirely into RAM.

---

# 34. Backpressure

Node.js streams must respect backpressure.

Do not continuously produce data faster than the consumer can process it.

Streaming pipelines should allow downstream consumers to regulate upstream production.

---

# 35. Database Access

Database clients should be initialized once and reused.

Do not create a new database client for every request.

Preferred:

```text id="c5n7r2"
Application Startup
 ↓
DB Client / Pool
 ↓
Requests
```

---

# 36. PostgreSQL

PostgreSQL access must follow the same KAMPYN database rules as Go.

Use:

- Parameterized queries
- Transactions
- Constraints
- Indexes
- Pagination
- Connection pooling
- Context/deadline equivalents
- Explicit error handling

Node.js must not bypass the established database architecture.

---

# 37. MongoDB

MongoDB access should use:

- Explicit filters
- Appropriate indexes
- Bounded queries
- Projection where useful
- Connection reuse
- Transaction semantics where required

Do not treat MongoDB as an unstructured cache.

---

# 38. Redis

Redis clients should be shared.

Use Redis for:

```text id="z3y7m1"
Cache
Rate limiting
Queue infrastructure
Short-lived coordination
Temporary state
```

Do not make Redis the source of truth for transactional domain state without explicit architecture.

---

# 39. OpenSearch

OpenSearch should remain behind the search boundary.

Node.js code should not scatter raw OpenSearch queries throughout unrelated application services.

Prefer:

```text id="x5m8q3"
Search Application
 ↓
Search Repository
 ↓
OpenSearch
```

---

# 40. Transactions

Transactions must remain explicit.

Do not keep transactions open while waiting for external providers.

Bad:

```text id="m4r8c1"
BEGIN
 ↓
Database mutation
 ↓
External API
 ↓
WAIT
 ↓
COMMIT
```

Prefer short local transactions and durable asynchronous coordination.

---

# 41. Error Handling

Use explicit error types or stable error codes.

Avoid throwing arbitrary strings:

```ts id="n7c2v4"
throw "failed";
```

Prefer:

```ts id="j5x8m3"
throw new ApplicationError(
  "ORDER_NOT_FOUND",
  "Order was not found.",
);
```

---

# 42. Error Translation

Infrastructure errors must be translated before reaching API clients.

Example:

```text id="p3m7x9"
Database unique violation
       ↓
Application Conflict
       ↓
HTTP 409
```

Do not expose raw database or provider errors.

---

# 43. Error Boundaries

Errors should be handled at meaningful boundaries.

```text id="a6k2r8"
Infrastructure
   ↓
Application
   ↓
Transport
   ↓
Stable API Error
```

Do not wrap every function with identical try/catch blocks.

---

# 44. Exception Handling

Use `try/catch` when you can:

- Recover
- Translate
- Add meaningful context
- Perform required cleanup

Do not catch errors merely to rethrow the same error.

---

# 45. Unhandled Errors

The process must have a defined policy for:

- Unhandled promise rejections
- Uncaught exceptions

Unexpected process-level failures should be observable and should not leave the process in an unknown state.

For critical failures, controlled process termination and orchestration-level restart may be safer than continuing in a potentially corrupted state.

---

# 46. HTTP Clients

Use a shared HTTP client abstraction.

Every external call should have:

- Timeout
- Authentication
- Error handling
- Retry policy where appropriate
- Connection reuse
- Observability

Never allow an external request to wait indefinitely.

---

# 47. External Integrations

Node.js integrations must follow:

```text id="e9r3x6"
Application Contract
 ↓
Provider Adapter
 ↓
External API
```

Provider SDKs must not leak into the domain.

See:

```text id="k2m6q8"
backend/integrations.md
```

for integration rules.

---

# 48. Authentication

Authentication logic must remain centralized.

Do not implement separate authentication rules in:

```text id="c4r7n2"
Next.js
Node service
Go service
```

unless they are explicitly different authentication boundaries.

There should be one authoritative identity model.

---

# 49. Authorization

Node.js code must not invent independent authorization rules.

If a Node service requires authorization:

```text id="p7x3m9"
Identity
 ↓
Tenant
 ↓
Permission
 ↓
Application Resource
```

must follow the same KAMPYN authorization model.

---

# 50. Multi-Tenancy

Node.js must preserve tenant isolation.

Every tenant-scoped operation must carry trusted tenant context.

Never trust:

```text id="r6c8v2"
tenantId
```

solely because it came from:

```text id="y7m3n5"
query parameter
request body
custom header
```

Tenant identity must be established through the trusted authentication/routing model.

---

# 51. Next.js Tenant Handling

If Next.js uses tenant-aware rendering:

```text id="v8q2k5"
Host / Domain
 ↓
Tenant Resolution
 ↓
Trusted KAMPYN API
```

Do not let the browser arbitrarily choose a tenant and treat it as authorized.

---

# 52. Validation

Use runtime validation for external input.

KAMPYN already uses Zod on the TypeScript side.

Zod may be used for:

- HTTP requests
- Configuration
- External API responses
- Webhook payloads
- SDK boundaries

Example:

```ts id="w5m8q2"
const result = Schema.safeParse(input);
```

Do not trust TypeScript types alone at runtime.

---

# 53. TypeScript vs Runtime Validation

This is unsafe:

```ts id="q4m7x2"
type User = {
  id: string;
};

const user = input as User;
```

The type assertion does not validate runtime data.

Use runtime schemas at trust boundaries.

---

# 54. API Contracts

Node.js services must consume the same API contracts as other KAMPYN clients.

Avoid manually duplicating:

```text id="h3r8c1"
DTO definitions
Enums
Error codes
Pagination rules
```

where generated or shared contracts are available.

---

# 55. Generated Types

Where OpenAPI or another contract-generation system is established, prefer generated client/types over manually maintained duplicates.

Generated code must remain reproducible.

---

# 56. API Client

If Node.js calls the Go API, use a centralized API client.

```text id="b7x4m9"
Node.js
   ↓
KAMPYN API Client
   ↓
Go API
```

The client should centralize:

- Base URL
- Authentication
- Headers
- Serialization
- Error mapping
- Timeouts
- Retries where safe
- Request IDs
- Tracing

---

# 57. Node.js as API Consumer

When Node.js consumes Go APIs:

```text id="j5r9c2"
Node.js
   ↓
HTTP Contract
   ↓
Go API
```

Node.js must treat the Go API as a remote boundary.

Do not access Go service databases directly merely because both applications run in the same deployment.

---

# 58. Shared Database Access

Avoid allowing a Node.js service and Go service to independently mutate the same domain tables.

Bad:

```text id="q8x3m7"
Go ─────┐
        ├── PostgreSQL
Node ───┘
```

with both owning the same business state.

Preferred:

```text id="z6r2k9"
Node
 ↓
Go API
 ↓
Domain Owner
 ↓
Database
```

unless the database ownership model explicitly defines separate bounded contexts.

---

# 59. Event Consumption

Node.js services may consume events when appropriate.

Consumers must support:

- Duplicate delivery
- Retry
- Idempotency
- Versioning
- Dead-letter handling
- Tenant context
- Observability

Do not assume exactly-once execution.

---

# 60. Event Publishing

Node.js must publish events only through the established event architecture.

When a database mutation and event publication must be atomic:

```text id="f3k7m1"
Database Transaction
 ↓
Outbox
 ↓
Event Publisher
```

Do not publish an event first and assume the database mutation will succeed.

---

# 61. Background Jobs

Node.js jobs must follow the same principles as Go jobs:

```text id="k5r9x2"
Queue
 ↓
Bounded Worker
 ↓
Application Use Case
 ↓
Infrastructure
```

Every job must define:

- Timeout
- Retry policy
- Idempotency
- Concurrency
- Tenant context
- Failure behavior

---

# 62. Node.js Worker Processes

If Node.js workers are required:

```text id="w4q7m2"
node worker
```

must be independently deployable and scalable.

Do not run unbounded background work inside a Next.js request process.

---

# 63. Scheduled Tasks

Scheduled Node.js tasks must not rely on every application replica executing the schedule.

Bad:

```text id="x8m3k6"
API Pod 1 → cron
API Pod 2 → cron
API Pod 3 → cron
```

This may create duplicate execution.

Use a centralized scheduler or Kubernetes CronJob.

---

# 64. Node.js Concurrency

Node.js concurrency is primarily event-loop based.

However, asynchronous operations can still overwhelm dependencies.

This is dangerous:

```ts id="y5k8r2"
await Promise.all(
  hugeArray.map(processItem),
);
```

Bound concurrency explicitly.

---

# 65. Worker Threads

Worker Threads may be used for CPU-heavy operations when appropriate.

However, they are not a substitute for architectural separation.

For substantial backend processing, consider:

```text id="j3m7q8"
Go worker
Background job
Dedicated service
```

before introducing complex Node.js thread pools.

---

# 66. Event Loop Monitoring

Production Node.js services should monitor event-loop health where relevant.

Useful signals include:

```text id="m6r2x9"
Event loop delay
CPU utilization
Memory usage
Heap usage
Active handles
Request latency
```

High event-loop delay can indicate CPU-bound work or blocking operations.

---

# 67. Memory Management

Node.js services must have explicit memory expectations.

Avoid:

```text id="p4x8m2"
Global unbounded arrays
Unbounded caches
Entire large files in memory
Huge JSON objects
Unbounded Promise collections
```

Memory limits should be compatible with deployment configuration.

---

# 68. Streams

Use streams for:

- Large file downloads
- Large file uploads
- Data transformation
- Export generation
- Large responses

Always handle:

- Backpressure
- Errors
- Cleanup
- Cancellation

---

# 69. Cache Usage

Node.js may use Redis through the common KAMPYN cache architecture.

Cache keys must include appropriate scope.

Example:

```text id="x7m3q9"
tenant:{tenantId}:product:{productId}
```

Never allow cross-tenant cache collisions.

---

# 70. Logging

Use structured logging.

Include appropriate fields:

```text id="c4r8m1"
requestId
traceId
tenantId
operation
duration
status
```

Never log:

```text id="y5k2p7"
password
access token
refresh token
API key
payment credential
sensitive payload
```

---

# 71. Metrics

Metrics should use bounded labels.

Good:

```text id="m8r3q2"
route
method
status
service
operation
```

Avoid:

```text id="z7k4p1"
userId
orderId
requestId
email
```

Use logs/traces for those identifiers.

---

# 72. Health Checks

Node.js services should expose appropriate health endpoints.

Example:

```text id="f6x2m8"
GET /health/live
GET /health/ready
```

Liveness should be lightweight.

Readiness may verify required dependencies according to service requirements.

---

# 73. Graceful Shutdown

Node.js processes must handle:

```text id="q9m3x5"
SIGTERM
SIGINT
```

Shutdown should:

```text id="v4r8k2"
Stop accepting work
 ↓
Stop schedulers
 ↓
Drain requests/jobs
 ↓
Close DB connections
 ↓
Close Redis
 ↓
Close HTTP clients/consumers
 ↓
Exit
```

Shutdown must have a bounded deadline.

---

# 74. Configuration

Configuration must be loaded and validated at startup.

Example:

```text id="k7x3m9"
Environment
 ↓
Config Parser
 ↓
Runtime Validation
 ↓
Application
```

Zod is appropriate for validating TypeScript configuration.

---

# 75. Environment Variables

Do not read environment variables throughout business logic.

Avoid:

```ts id="w3r7m2"
process.env.DATABASE_URL
```

in dozens of files.

Prefer:

```text id="q8m4x1"
Environment
 ↓
Config
 ↓
Dependencies
```

---

# 76. Secrets

Never commit secrets.

Never expose secrets through:

- Browser bundles
- Public Next.js environment variables
- Logs
- Error responses
- Job payloads
- Git history

For Next.js, only explicitly public variables should be exposed to the browser.

---

# 77. Node.js Security

Protect against:

- Prototype pollution
- Injection
- SSRF
- Path traversal
- XSS
- CSRF where applicable
- Unsafe deserialization
- Dependency vulnerabilities
- Arbitrary code execution
- Request smuggling
- Oversized payloads

Validate external input at every trust boundary.

---

# 78. SSRF

Server-side HTTP requests must not blindly fetch arbitrary user-provided URLs.

If URL fetching is a required feature:

```text id="p6x8m2"
Validate URL
 ↓
Validate protocol
 ↓
Apply allowlist / policy
 ↓
Resolve safely
 ↓
Block internal/private destinations where required
 ↓
Bound request
```

---

# 79. Prototype Pollution

Avoid unsafe merging of untrusted objects.

Do not blindly merge:

```text id="c7m4x2"
user input
+
configuration
```

without validation and safe object handling.

Runtime schemas should validate external objects.

---

# 80. Dependency Security

Node.js dependencies must be monitored.

Use repository-approved tooling for:

```text id="h8x3m5"
Dependency audit
Lockfile integrity
Vulnerability scanning
Automated updates
```

Do not automatically upgrade major dependencies without testing compatibility.

---

# 81. Package Scripts

Package scripts should be explicit.

Typical scripts may include:

```text id="r4m7x2"
dev
build
start
test
lint
typecheck
format
```

Scripts should not hide dangerous destructive operations.

---

# 82. Build Process

Production builds must be reproducible.

The build should:

```text id="q6x3m9"
Install locked dependencies
 ↓
Typecheck
 ↓
Lint
 ↓
Test
 ↓
Build
 ↓
Package
```

Do not rely on undeclared globally installed tools.

---

# 83. Production Dependencies

Production containers should install only required dependencies where practical.

Separate:

```text id="w8m2k4"
development dependencies
production dependencies
```

This reduces:

- Image size
- Attack surface
- Startup overhead

---

# 84. Node.js Containers

Node.js containers should:

- Use a supported Node.js version
- Run as a non-root user where possible
- Include only required files
- Use deterministic dependency installation
- Define resource limits
- Handle SIGTERM correctly
- Avoid development servers in production

---

# 85. Container Health

Container health should reflect the actual service lifecycle.

Do not mark a service healthy merely because the Node.js process started.

Readiness should represent whether it can serve its intended workload.

---

# 86. API Compatibility

Node.js services consuming Go APIs must tolerate compatible server changes.

Avoid assuming:

```text id="f7m3x8"
field will always exist
enum will never gain a value
error body never changes
```

Clients should handle unknown fields and compatible additions safely.

---

# 87. Enum Handling

When consuming external APIs, avoid assuming every enum value is known forever.

Unknown values should fail safely or map to an explicit fallback where appropriate.

Do not silently treat an unknown security-sensitive state as safe.

---

# 88. Serialization Compatibility

API serialization is a contract.

Changes to:

```text id="m5x8q2"
field names
field types
nullability
enum values
pagination
error structure
```

must be deliberate and version-aware.

---

# 89. Testing

Node.js code must have appropriate:

- Unit tests
- Integration tests
- API tests
- Contract tests
- Security tests
- Concurrency tests

The exact test framework is repository-defined.

---

# 90. Unit Tests

Test:

- Application logic
- Validation
- Error mapping
- State transitions
- Pure functions
- Provider adapters

Tests should focus on behavior.

---

# 91. Integration Tests

Integration tests should verify:

```text id="x4m7p2"
Database
Redis
OpenSearch
Go API
External provider adapters
Queues
Object storage
```

where those dependencies are part of the service contract.

---

# 92. Contract Tests

When Node.js consumes the Go API, contract tests should verify:

```text id="q7x3m5"
Request schema
Response schema
Error schema
Authentication behavior
Pagination
Compatibility
```

This reduces accidental cross-runtime drift.

---

# 93. Concurrency Tests

Test:

```text id="w8m2q4"
Duplicate requests
Concurrent operations
Job duplication
Rate limiting
Shared cache access
Graceful shutdown
```

Use realistic concurrency rather than only sequential unit tests.

---

# 94. Type Checking

CI must run TypeScript type checking.

Example:

```bash id="f3m7x2"
tsc --noEmit
```

or the repository's equivalent.

Do not rely only on runtime tests.

---

# 95. Linting

Linting should run consistently in local development and CI.

Lint rules should enforce:

- Unused variables
- Unsafe patterns
- Promise handling
- Import consistency
- Complexity where configured
- Security-sensitive patterns where supported

---

# 96. Formatting

Use the repository's formatter consistently.

Do not mix multiple formatting systems without explicit configuration.

Formatting should be automatic.

---

# 97. Observability

Node.js services must expose enough telemetry to diagnose:

```text id="z4m8x2"
Request failures
Dependency failures
Event-loop delays
Memory pressure
Queue delays
External API latency
Database latency
```

---

# 98. Development vs Production

Development may use:

```text id="r7x3m5"
Hot reload
Verbose logs
Local services
Debugging tools
```

Production must use:

```text id="q4m8x2"
Compiled/built output
Structured logs
Secure configuration
Bounded resources
Health checks
Graceful shutdown
```

Never accidentally deploy development configuration.

---

# 99. Node.js and Self-Hosting

If a KAMPYN self-hosted deployment requires a Node.js service, it must be packaged independently.

Example:

```text id="m8x3q7"
kampyn-api
kampyn-web
kampyn-worker
kampyn-node-service
```

Only required services should be deployed.

Do not force self-hosted installations to run Node.js services they do not need.

---

# 100. Documentation

Any Node.js service must document:

- Purpose
- Why Node.js is used
- API contract
- Dependencies
- Environment variables
- Deployment
- Health checks
- Resource requirements
- Failure behavior
- Database ownership
- Queue ownership
- Integration ownership

The reason for introducing Node.js should be especially clear.

---

# 101. Definition of Done

Node.js backend code is complete only when:

- Its responsibility is explicitly defined.
- Go remains the owner of core backend behavior unless explicitly approved otherwise.
- Architectural boundaries are clear.
- TypeScript strictness is enabled.
- Dependencies are justified.
- Request handlers remain thin.
- Business logic is in the application/domain layer.
- Infrastructure dependencies are isolated.
- Runtime validation exists at trust boundaries.
- Authentication follows KAMPYN's identity model.
- Authorization follows KAMPYN's authorization model.
- Tenant isolation is enforced.
- External calls have timeouts.
- Concurrency is bounded.
- Large data is streamed where necessary.
- Errors are translated appropriately.
- Secrets are protected.
- Logs are structured and safe.
- Metrics have bounded cardinality.
- Health checks exist.
- Graceful shutdown works.
- Tests cover success and failure.
- API contracts are verified.
- Dependency/security checks pass.
- Documentation is updated.

---

# 102. Final Invariants

The following rules are mandatory:

```text id="k7m2x8"
1. Go is KAMPYN's primary backend runtime.

2. Node.js must have a clearly justified responsibility.

3. Node.js must not become a duplicate implementation of the Go backend.

4. Core business rules must have one authoritative owner.

5. Next.js server functionality must not silently become a second
   business backend.

6. Node.js backend code must use TypeScript unless explicitly justified.

7. TypeScript strictness must not be weakened merely to bypass errors.

8. Runtime validation is required at external trust boundaries.

9. TypeScript type assertions are not runtime validation.

10. Route handlers must remain thin.

11. Domain logic must remain independent of Node.js frameworks.

12. Provider SDKs must remain behind integration boundaries.

13. Database clients must be reused rather than created per request.

14. Node.js must not independently mutate Go-owned domain data
    without an explicit ownership model.

15. Shared database mutation by multiple runtimes requires explicit
    bounded-context ownership.

16. Request-scoped state must never be stored in mutable globals.

17. Unbounded Promise concurrency is prohibited.

18. CPU-heavy work must not block the event loop.

19. Large files and datasets must use bounded-memory processing.

20. External calls must have explicit timeouts.

21. Errors must be translated into stable application/API semantics.

22. Secrets must never enter browser bundles, logs, source code,
    or untrusted job payloads.

23. Tenant context must be trusted, explicit, and preserved across
    asynchronous operations.

24. Node.js services must support graceful shutdown.

25. Scheduled work must not accidentally execute once per application replica.

26. Events and jobs must assume duplicate delivery unless
    exactly-once execution is explicitly guaranteed.

27. Node.js services must remain independently observable and deployable.

28. A Node.js service should not exist merely because implementing
    the same functionality in Go would be inconvenient.

29. If a Node.js component starts accumulating substantial domain logic,
    its architectural ownership must be reviewed.

30. The existence of Node.js must simplify a clearly defined boundary,
    not create a second backend architecture.
```

## Node.js's Position in KAMPYN

```text id="v8m3q6"
                         KAMPYN
                            │
              ┌─────────────┴─────────────┐
              │                           │
          Next.js                      Go Backend
              │                           │
       Frontend Server             Core Backend
       Functionality                    │
              │                         │
              │                  ┌──────┴──────┐
              │                  │             │
              │               Domain      Infrastructure
              │
              │
              └──────────────┐
                             │
                     Explicit Node.js
                       Service Only
                     When Justified
```

The governing principle is simple:

```text id="m4x7q2"
Use Node.js where Node.js provides a clear architectural advantage.

Use Go for KAMPYN's core backend domain.

Never maintain two implementations of the same business rule.
```
# KAMPYN Backend Middleware

## Purpose

Middleware provides cross-cutting behavior around HTTP requests without placing unrelated concerns inside individual handlers.

KAMPYN middleware is responsible for concerns such as:

- Request IDs
- Distributed tracing
- Structured logging
- Panic recovery
- Request size limits
- CORS
- Security headers
- Authentication
- Tenant resolution
- Rate limiting
- Request timeouts
- Metrics
- Access logging
- API versioning where applicable

Middleware must remain thin.

Business logic, domain rules, resource authorization, database mutations, and integration behavior belong elsewhere.

---

# 1. Core Principles

Middleware must be:

- Explicit
- Ordered
- Small
- Composable
- Testable
- Observable
- Security-conscious
- Tenant-aware where required
- Failure-safe

Middleware should solve cross-cutting concerns.

It must not become a hidden application layer.

---

# 2. Middleware Pipeline

KAMPYN requests conceptually follow:

```text
Client
  │
  ▼
Network / Load Balancer
  │
  ▼
Security / Edge Controls
  │
  ▼
Request ID
  │
  ▼
Tracing
  │
  ▼
Recovery
  │
  ▼
Request Limits
  │
  ▼
CORS / Security Headers
  │
  ▼
Logging / Metrics
  │
  ▼
Authentication
  │
  ▼
Tenant Resolution
  │
  ▼
Rate Limiting
  │
  ▼
Authorization Boundary
  │
  ▼
Router
  │
  ▼
Handler
  │
  ▼
Application
```

The exact ordering may vary where architecture requires it, but dependencies between middleware must be explicit.

---

# 3. Middleware Is Not Business Logic

Middleware should not contain logic such as:

```text
Calculate order total
Reserve inventory
Create booking
Process payment
Approve complaint
Assign hostel room
```

Instead:

```text
Middleware
    ↓
Request Context
    ↓
Handler
    ↓
Application Use Case
    ↓
Domain
```

Middleware prepares the request for application execution.

---

# 4. Thin Middleware Rule

A middleware should generally:

```text
Receive request
 ↓
Perform cross-cutting operation
 ↓
Update context / response
 ↓
Call next handler
```

Avoid middleware that:

- Queries many database tables
- Calls external APIs unnecessarily
- Implements domain state transitions
- Performs large computations
- Mutates business state
- Contains complex branching

Complex middleware usually indicates misplaced responsibilities.

---

# 5. Middleware Categories

KAMPYN middleware can be divided into:

### Infrastructure middleware

```text
Request ID
Tracing
Recovery
Logging
Metrics
Timeout
```

### Security middleware

```text
CORS
Security headers
Authentication
Rate limiting
Request limits
```

### Context middleware

```text
Tenant resolution
Identity context
Localization where required
```

### Routing middleware

```text
API versioning
Route-level policies
```

Authorization may be partially implemented through middleware, but resource-level authorization belongs to the application/domain boundary.

---

# 6. Global vs Route Middleware

Not every middleware should run for every route.

Global middleware may include:

```text
Request ID
Recovery
Tracing
Logging
Metrics
Request limits
Security headers
```

Protected API routes may additionally use:

```text
Authentication
Tenant resolution
Rate limiting
```

Specific routes may require additional policies.

---

# 7. Middleware Ordering

Ordering is a correctness and security concern.

For example:

```text
Authentication
     ↓
Tenant Resolution
     ↓
Authorization
```

should not be replaced with:

```text
Authorization
     ↓
Authentication
```

when authorization depends on authenticated identity.

Document ordering dependencies explicitly.

---

# 8. Request ID

Every request should receive a request ID.

Preferred behavior:

```text
Client-provided valid request ID
        ↓
Validate / normalize
        ↓
Use
```

or:

```text
No request ID
        ↓
Generate one
```

The request ID should be:

- Unique enough for operational correlation
- Safe to expose
- Included in structured logs
- Included in appropriate error responses

Never use a user-controlled request ID as a security credential.

---

# 9. Request ID Propagation

The request ID should be available through request context.

Conceptually:

```text
HTTP Request
     │
     ▼
Request ID Middleware
     │
     ▼
context.Context
     │
     ├── Handler
     ├── Application
     ├── Repository
     └── Integration
```

Do not store the request ID in global state.

---

# 10. Distributed Tracing

Tracing middleware should establish or propagate trace context.

Flow:

```text
Client
  ↓
API
  ↓
Application
  ↓
Database / Redis / Integration
```

Trace information should connect related operations.

Do not expose internal tracing data unnecessarily to clients.

---

# 11. Logging Middleware

Logging middleware should capture request-level information such as:

```text
method
path
status
duration
request_id
trace_id
tenant_id where available
authenticated principal where appropriate
```

Do not log:

- Passwords
- Access tokens
- Refresh tokens
- API keys
- Payment credentials
- Sensitive request bodies
- Full authorization headers

---

# 12. Access Logging

Access logs should be structured.

Example conceptual record:

```text
method=POST
path=/api/v1/orders
status=201
duration_ms=84
request_id=req_123
tenant_id=tenant_456
```

Avoid logging full URLs when query parameters may contain sensitive information.

---

# 13. Logging Request Bodies

Do not log request bodies by default.

Request bodies may contain:

- Passwords
- Personal information
- Payment data
- Uploaded content
- Authentication tokens

If debugging requires body inspection, use tightly controlled redacted logging in non-production environments.

---

# 14. Response Logging

Do not log complete response bodies by default.

Responses may contain:

- Personal information
- Internal identifiers
- Sensitive data
- Large payloads

Log metadata rather than content unless explicitly required.

---

# 15. Panic Recovery

HTTP middleware must provide a controlled panic recovery boundary.

Conceptually:

```text
Request
  ↓
Handler
  ↓
Unexpected panic
  ↓
Recovery Middleware
  ↓
Log / Trace
  ↓
Safe 5xx response
```

Recovery should:

- Prevent uncontrolled process termination where appropriate
- Record the panic
- Preserve trace/request context
- Return a safe response
- Avoid exposing internal details

---

# 16. Panic Recovery Does Not Fix Bugs

Recovery must not be used to make broken application logic appear healthy.

A recovered panic should be:

- Logged
- Observable
- Investigated
- Tested when reproducible

Do not silently convert every panic into success.

---

# 17. Error Responses

Middleware should return stable error responses.

Example:

```json
{
  "error": {
    "code": "INTERNAL_ERROR",
    "message": "An unexpected error occurred.",
    "requestId": "req_123"
  }
}
```

Do not return:

```text
database stack trace
Go panic message
provider credentials
internal filesystem path
```

to clients.

---

# 18. Authentication Middleware

Authentication middleware establishes who the caller is.

Conceptually:

```text
Request
  ↓
Credential extraction
  ↓
Credential verification
  ↓
Identity
  ↓
Request Context
```

Authentication answers:

> Who is this caller?

It does not answer:

> Is this caller allowed to modify this particular resource?

---

# 19. Authentication Sources

Depending on KAMPYN's authentication architecture, middleware may process:

- Secure session cookies
- Access tokens
- Service credentials
- API keys
- OAuth/OIDC identity
- Internal service authentication

Each authentication mechanism must have explicit validation rules.

---

# 20. Credential Extraction

Credentials must be extracted from defined locations.

Examples:

```text
Secure cookie
Authorization header
Explicit service credential header
```

Do not accept credentials from arbitrary query parameters.

Avoid allowing multiple ambiguous authentication sources unless the precedence rules are explicit.

---

# 21. Token Validation

Token-based authentication must validate appropriate claims.

Depending on the token system:

```text
signature
issuer
audience
expiration
not-before
subject
token type
```

Do not merely decode a token and trust its contents.

---

# 22. Session Authentication

For session-based authentication:

```text
Cookie
  ↓
Session lookup / verification
  ↓
Identity
  ↓
Context
```

Sessions must support:

- Expiration
- Revocation
- Secure cookies
- Appropriate SameSite policy
- HTTPS-only transport in production
- Session rotation where required

---

# 23. Authentication Failure

Authentication failures should not reveal unnecessary information.

Prefer a generic response such as:

```text
401 Unauthorized
```

with a stable error code.

Do not reveal:

```text
Token expired because user exists
User does not exist
Session belongs to another tenant
```

when doing so creates account or resource enumeration risk.

---

# 24. Optional Authentication

Some routes may allow anonymous access.

For example:

```text
Marketing website
Public catalog
Public university information
```

Optional authentication must be explicit.

Do not silently treat failed authentication as anonymous access on routes intended to be protected.

---

# 25. Tenant Resolution

KAMPYN is multi-tenant.

Tenant resolution establishes the tenant associated with the request.

Possible sources include:

```text
Authenticated membership
Subdomain
Custom domain
Explicit API context
Deployment configuration
```

The chosen source must be trusted and validated.

---

# 26. Tenant Context

After tenant resolution:

```text
Request
  ↓
Tenant Resolver
  ↓
TenantContext
  ↓
Handler / Application
```

The tenant context should include the stable internal tenant identifier.

Do not pass raw hostnames throughout business logic.

---

# 27. Tenant Resolution vs Authorization

Tenant resolution does not automatically authorize the caller.

Example:

```text
User belongs to Tenant A
Request targets Tenant B
```

The system must verify whether the user has legitimate membership or administrative access to Tenant B.

Never assume:

```text
tenant resolved
=
tenant authorized
```

---

# 28. Tenant Isolation

Middleware should establish tenant context, but application and repository layers must still enforce tenant isolation.

Middleware is not the final defense.

The complete flow is:

```text
Resolve Tenant
      ↓
Authenticate Identity
      ↓
Authorize Membership
      ↓
Application
      ↓
Tenant-Scoped Repository
```

---

# 29. Authorization Middleware

Middleware may perform coarse authorization checks such as:

```text
Authenticated
Admin-only route
Service-only route
Required permission
```

However, resource-level authorization belongs closer to the application/domain operation.

Example:

```text
Middleware:
User has "orders:read"

Application:
User can read THIS order?
```

Both may be necessary.

---

# 30. Permission Middleware

If route-level permissions are used:

```text
RequirePermission("orders:read")
```

the permission requirement should be explicit.

Do not hide authorization requirements in arbitrary handler code.

---

# 31. Resource Authorization

Middleware must not attempt to fully authorize every resource.

Example:

```text
GET /orders/{orderId}
```

Middleware can verify:

```text
Authenticated
orders:read permission
```

The application must verify:

```text
Does this user have access to orderId?
Does this order belong to the active tenant?
Is this order visible in the current state?
```

---

# 32. Rate Limiting

Rate limiting protects the API and its dependencies.

Potential dimensions include:

```text
IP
User
Tenant
API key
Route
Provider
```

The chosen scope must match the threat model.

Do not use IP-only rate limiting for authenticated operations when users may share IP addresses.

---

# 33. Rate Limit Response

When a request is rejected because of rate limiting:

```text
HTTP 429 Too Many Requests
```

The response should provide appropriate retry information when supported.

Do not expose internal rate-limit implementation details unnecessarily.

---

# 34. Distributed Rate Limiting

If KAMPYN runs multiple API replicas:

```text
API-1
API-2
API-3
```

an in-memory rate limiter does not provide a global limit.

Use an appropriate distributed mechanism such as Redis where global coordination is required.

---

# 35. Tenant Rate Limits

Multi-tenant deployments may require tenant-level limits.

Example:

```text
Tenant A → 1,000 requests/minute
Tenant B → 1,000 requests/minute
```

Limits should be configurable.

Rate limiting must not become a substitute for authorization.

---

# 36. Request Size Limits

Middleware should enforce maximum request sizes.

Protect against:

- Memory exhaustion
- Oversized JSON payloads
- Multipart abuse
- Malicious uploads

Different routes may require different limits.

For example:

```text
JSON API
Small payload

File upload
Larger bounded payload
```

Do not apply one enormous global limit simply to avoid route configuration.

---

# 37. File Upload Middleware

File uploads require more than middleware.

Middleware may enforce:

```text
Content-Length
Maximum body size
Content-Type
Authentication
Rate limits
```

The application must additionally validate:

- File type
- File content
- Filename
- Storage destination
- Tenant ownership
- Malware/security requirements where applicable

---

# 38. Content-Type Validation

Endpoints expecting JSON should validate the content type where appropriate.

Do not assume:

```text
Content-Type: application/json
```

means the payload is safe or valid.

The body must still be parsed and validated.

---

# 39. CORS

CORS must be configured explicitly.

Do not use:

```text
Access-Control-Allow-Origin: *
```

for authenticated browser APIs unless the architecture explicitly permits it.

Allowed origins should be derived from trusted configuration.

---

# 40. Credentialed CORS

When cookies or credentialed requests are used:

- Allowed origins must be explicit
- Credential handling must be deliberate
- Wildcard origins must not be combined incorrectly with credentials
- CSRF protections must be considered

CORS is not authentication.

---

# 41. CSRF

If browser authentication uses cookies, CSRF protection must be considered for state-changing requests.

Possible mechanisms include:

```text
SameSite cookies
CSRF tokens
Origin validation
Referer validation where appropriate
```

The exact mechanism must match the authentication architecture.

Bearer-token APIs have different CSRF characteristics from cookie-authenticated browser APIs.

---

# 42. Security Headers

Middleware may apply appropriate security headers.

Examples include:

```text
Content-Security-Policy
X-Content-Type-Options
Referrer-Policy
Strict-Transport-Security
Permissions-Policy
```

Headers must be configured according to the frontend/API architecture.

Do not blindly copy security headers without understanding their effects.

---

# 43. HSTS

HSTS should only be enabled when the deployment is fully prepared for HTTPS.

Production configuration should enforce secure transport consistently.

Do not enable aggressive HSTS settings in environments where HTTPS readiness is uncertain.

---

# 44. Request Timeout

Middleware may apply an upper bound to request execution.

Conceptually:

```text
Request
  ↓
Timeout Context
  ↓
Handler
  ↓
Application
```

Timeout values should reflect the endpoint.

Do not apply a short global timeout to long-running operations that should instead be asynchronous.

---

# 45. Timeout Ownership

Middleware timeouts should complement, not replace, lower-level timeouts.

For example:

```text
HTTP request timeout
       ↓
Application deadline
       ↓
Database timeout
       ↓
External API timeout
```

Each layer should have appropriate bounds.

---

# 46. Request Cancellation

When the client disconnects:

```text
Client
  X
Request Context Cancelled
        ↓
Application
        ↓
Database / External Calls
```

Handlers and downstream operations must respect cancellation.

Do not continue expensive work after the request is no longer relevant unless the work was intentionally detached into a durable background job.

---

# 47. Metrics Middleware

Metrics middleware should capture request-level information such as:

```text
HTTP requests
Request duration
Response status
Route
Method
```

Avoid using raw URLs containing arbitrary IDs as metric labels.

Prefer normalized route patterns:

```text
/orders/{id}
```

rather than:

```text
/orders/order_123
/orders/order_456
```

---

# 48. High-Cardinality Metrics

Never use unbounded values as metric labels.

Avoid:

```text
user_id
order_id
request_id
trace_id
email
```

Use logs and traces for high-cardinality identifiers.

---

# 49. Request Metrics

Useful metrics include:

```text
http_requests_total
http_request_duration_seconds
http_requests_in_flight
http_response_size_bytes
http_request_size_bytes
```

Useful dimensions:

```text
method
route
status
```

Keep the label set bounded.

---

# 50. Middleware and Tracing

Middleware should establish tracing before application execution.

Example:

```text
Request
 ↓
Extract trace context
 ↓
Create server span
 ↓
Application
 ↓
Child spans
 ↓
Response
 ↓
Close span
```

The trace should include status and error information.

---

# 51. Middleware and Audit Logs

Not every request needs a business audit event.

Access logs and audit logs are different.

```text
Access log:
GET /orders/123 → 200

Audit event:
Admin changed order status from X to Y
```

Business-sensitive mutations should create explicit audit records/events in the application layer.

Do not attempt to derive complete audit history from HTTP middleware.

---

# 52. Middleware and Idempotency

Middleware may extract idempotency keys for relevant endpoints.

However, the application/domain layer must own the actual idempotency semantics.

Flow:

```text
Middleware
 ↓
Extract Idempotency-Key
 ↓
Application
 ↓
Idempotency Store
 ↓
Business Operation
```

Do not implement generic idempotency by caching arbitrary HTTP responses without understanding the operation.

---

# 53. Request Context Structure

Request context may carry:

```text
Request ID
Trace ID
Authenticated Identity
Tenant Context
Authorization Context
Correlation ID
```

It should not carry arbitrary application state.

Avoid turning context into a hidden dependency container.

---

# 54. Context Security

Context values must be treated as trusted only after the middleware that establishes them has validated them.

Do not allow clients to directly inject:

```text
tenant_id
role
permission
user_id
```

into context through arbitrary headers.

Trusted identity information must originate from validated authentication and authorization mechanisms.

---

# 55. Service-to-Service Authentication

Internal KAMPYN services/workers may require service authentication.

Middleware should validate:

```text
service identity
credential
audience
expiration
scope
```

Do not assume that internal network access means trusted access.

---

# 56. Internal Routes

Internal endpoints must remain separate from public APIs where possible.

Examples:

```text
/health
/internal/*
/metrics
```

must have explicit exposure and security policies.

Do not accidentally expose administrative or operational endpoints publicly.

---

# 57. Health Endpoints

Health endpoints should be intentionally designed.

Typical:

```text
GET /health/live
GET /health/ready
```

Liveness should answer:

> Is the process alive?

Readiness should answer:

> Should this instance receive traffic?

Do not make liveness depend on every external dependency.

---

# 58. Metrics Endpoints

Metrics endpoints may contain operational information.

Restrict access appropriately.

Do not expose metrics publicly unless the deployment explicitly requires it.

---

# 59. Middleware Failure Behavior

Every middleware must define what happens if it fails.

Examples:

```text
Authentication failure → 401
Rate limit exceeded → 429
Invalid request size → 413
Invalid CORS origin → browser-enforced rejection / appropriate response
Timeout → 408/504 depending on boundary
Unexpected middleware failure → 5xx
```

Do not silently continue after security middleware fails.

---

# 60. Fail Closed

Security-related middleware should fail closed.

For example:

```text
Authorization service unavailable
        ↓
Do not grant access
```

Do not convert:

```text
cannot verify authorization
```

into:

```text
authorized
```

---

# 61. Fail Open

Fail-open behavior should be limited to non-security-critical functionality.

Example:

```text
Optional analytics unavailable
        ↓
Request may continue
```

Never use fail-open behavior for:

- Authentication
- Authorization
- Tenant isolation
- Signature verification
- Security controls

unless explicitly designed and reviewed.

---

# 62. Middleware Ordering Example

A reasonable API ordering is:

```text
1. Request ID
2. Trace Context
3. Recovery
4. Request Size / Transport Limits
5. Security Headers
6. CORS
7. Request Metrics
8. Access Logging
9. Authentication
10. Tenant Resolution
11. Rate Limiting
12. Route-Level Authorization
13. Handler
```

The exact order should be documented when dependencies differ.

---

# 63. Why Recovery Is Early

Recovery should surround as much application execution as possible.

Conceptually:

```text
Recovery
 ├── Authentication
 ├── Tenant Resolution
 ├── Rate Limiting
 ├── Handler
 └── Application
```

This ensures unexpected failures do not escape the HTTP boundary.

---

# 64. Why Authentication Precedes Tenant Authorization

Authentication establishes identity.

Then tenant context and membership can be validated.

```text
Credential
 ↓
Identity
 ↓
Tenant
 ↓
Membership
 ↓
Authorization
```

This prevents anonymous or forged identity data from establishing tenant access.

---

# 65. Route-Specific Middleware

Some routes may require additional middleware.

Examples:

```text
POST /payments
    ↓
Authentication
    ↓
Tenant
    ↓
Rate Limit
    ↓
Idempotency
```

File upload:

```text
POST /files
    ↓
Authentication
    ↓
Tenant
    ↓
Request Size
    ↓
Upload Policy
```

Admin route:

```text
/admin/*
    ↓
Authentication
    ↓
Tenant
    ↓
Admin Authorization
```

---

# 66. Middleware Composition

Middleware should be composable.

Conceptually:

```go
handler := Chain(
    RequestID(),
    Tracing(),
    Recovery(),
    Logging(),
    Authentication(),
    Tenant(),
)(router)
```

The implementation may use another composition style, but the resulting order must remain obvious.

---

# 67. Middleware Naming

Names should describe behavior.

Good:

```text
RequestID
Recover
Authenticate
ResolveTenant
RateLimit
RequirePermission
RequestTimeout
SecurityHeaders
```

Avoid:

```text
CommonMiddleware
GlobalMiddleware
HandleStuff
PreProcess
```

---

# 68. Middleware Dependencies

Middleware dependencies must be explicit.

For example:

```text
Tenant middleware
    requires authenticated identity
```

This should be visible in composition/configuration rather than relying on hidden assumptions.

---

# 69. Database Access in Middleware

Database access inside middleware should be minimized.

Acceptable examples may include:

```text
Session lookup
Tenant membership resolution
```

when they are fundamental to request context.

Avoid middleware that performs arbitrary business queries.

Bad:

```text
Middleware
 ↓
Load User
 ↓
Load Orders
 ↓
Load Inventory
 ↓
Load Permissions
 ↓
Load Notifications
```

This creates unnecessary work for every request.

---

# 70. Caching Middleware Context

If authentication or tenant resolution uses caching, cache correctness must be explicit.

Never allow a stale cache entry to grant unauthorized access.

Security-sensitive cache entries require:

- Appropriate TTL
- Explicit invalidation
- Tenant/user scoping
- Revocation behavior

---

# 71. Middleware Performance

Middleware runs on many requests.

Therefore middleware must be:

- Lightweight
- Bounded
- Efficient
- Non-blocking where appropriate

Avoid expensive work before routing if only a small subset of routes requires it.

---

# 72. Middleware Ordering and Performance

Place cheap rejection mechanisms early where safe.

For example:

```text
Request size
 ↓
Malformed transport
 ↓
Authentication
 ↓
Application
```

There is no reason to execute expensive application logic for an obviously oversized request.

Security dependencies must still be respected.

---

# 73. Middleware and Caching

HTTP caching behavior should be configured deliberately.

Do not cache authenticated or tenant-sensitive responses accidentally.

Cache headers must consider:

```text
Authentication
Tenant
User
Resource visibility
Response sensitivity
```

Public marketing content can have very different caching behavior from authenticated API responses.

---

# 74. Middleware and Search

Search requests may require:

```text
Authentication
Tenant resolution
Rate limiting
Query limits
```

But search-specific validation should remain in the search/application layer.

Middleware should not construct OpenSearch queries.

---

# 75. Middleware and Background Jobs

HTTP middleware must not be reused blindly for workers.

Workers do not have:

```text
HTTP request
HTTP headers
Browser cookies
HTTP response
```

Instead, background jobs require an execution context containing relevant:

```text
Tenant
Identity where required
Correlation
Job metadata
Cancellation
```

Do not fake HTTP requests merely to reuse middleware.

---

# 76. Middleware and WebSockets

WebSocket connections may require authentication and tenant resolution during the handshake.

After connection establishment, authorization must continue to be enforced at the message/action level where appropriate.

Do not assume handshake authorization automatically authorizes every future action.

---

# 77. Middleware and Streaming

Streaming endpoints require special care with:

- Timeouts
- Response flushing
- Cancellation
- Connection lifetime
- Logging
- Resource cleanup

Do not apply a short request timeout that unintentionally terminates valid long-lived streams.

---

# 78. Middleware and File Downloads

Downloads must enforce:

- Authentication
- Authorization
- Tenant isolation
- Object ownership
- Appropriate content headers
- Range behavior where supported

Do not allow middleware to authorize a file merely because the user is authenticated.

The application must verify access to the specific object.

---

# 79. Testing Middleware

Each middleware should have focused tests.

Test:

```text
Successful request
Missing credentials
Invalid credentials
Expired credentials
Wrong tenant
Unauthorized permission
Rate limit exceeded
Oversized request
Timeout
Panic recovery
Malformed request
Invalid origin
Context propagation
```

---

# 80. Middleware Chain Tests

Test the actual composed chain.

Individual middleware tests are insufficient because ordering can change behavior.

Example:

```text
Authentication
 ↓
Tenant
 ↓
Authorization
```

should be tested as a complete flow.

---

# 81. Security Testing

Security middleware should be tested for:

- Authentication bypass
- Header spoofing
- Tenant spoofing
- CORS misconfiguration
- CSRF where applicable
- Rate-limit bypass
- Request smuggling considerations
- Oversized requests
- Malformed headers
- Token manipulation
- Privilege escalation

---

# 82. Concurrency Testing

Middleware must remain safe under concurrent requests.

Avoid shared mutable state unless explicitly synchronized.

Test:

```text
100 concurrent requests
```

where middleware maintains shared state such as rate-limiters or caches.

Run the Go race detector where applicable.

```bash
go test -race ./...
```

---

# 83. Middleware State

Middleware instances should generally be safe for concurrent use.

If middleware contains mutable state:

```text
Rate limiter
Cache
Counters
Configuration
```

its synchronization strategy must be explicit.

Never assume a middleware instance is called sequentially.

---

# 84. Middleware Configuration

Configuration should be injected rather than read repeatedly from environment variables.

Example:

```text
Environment
 ↓
Config
 ↓
Middleware Constructor
 ↓
Middleware
```

This improves:

- Testability
- Startup validation
- Performance
- Predictability

---

# 85. Middleware Errors

Middleware errors should map to stable transport-level errors.

Do not expose internal errors.

Example:

```text
Authentication middleware
    ↓
invalid token
    ↓
401 AUTHENTICATION_REQUIRED
```

rather than returning the raw JWT library error.

---

# 86. Middleware Documentation

Each non-trivial middleware should document:

```text
Purpose
Inputs
Outputs
Context values
Ordering requirements
Failure behavior
Security implications
Configuration
Testing
```

This is particularly important for authentication, tenant, authorization, rate limiting, and security middleware.

---

# 87. Definition of Done

Middleware is complete only when:

- Responsibility is clearly defined.
- It contains no unnecessary business logic.
- Ordering requirements are documented.
- Context propagation is correct.
- Failure behavior is explicit.
- Security behavior is tested.
- Tenant behavior is verified where applicable.
- Authentication behavior is verified where applicable.
- Authorization boundaries are clear.
- Rate limiting is appropriately scoped.
- Request limits are enforced.
- Sensitive data is not logged.
- Metrics have bounded cardinality.
- Tracing works.
- Concurrency safety is verified.
- Configuration is validated.
- Middleware chain integration tests exist.
- Documentation is updated.

---

# 88. Final Invariants

The following rules are mandatory:

```text
1. Middleware handles cross-cutting concerns, not domain business logic.

2. Middleware must remain thin and composable.

3. Middleware ordering is part of the security and correctness model.

4. Authentication establishes identity; it does not replace authorization.

5. Tenant resolution does not automatically grant tenant access.

6. Resource-level authorization belongs in the application/domain boundary.

7. Security middleware must fail closed.

8. Authentication and authorization failures must never default to access.

9. Request context must carry trusted server-established metadata,
   not arbitrary client-provided authority.

10. Request IDs and trace IDs are correlation mechanisms, not security credentials.

11. Middleware must not store request-specific state globally.

12. Middleware instances must be safe under concurrent execution.

13. Rate limiting must work across replicas when global limits are required.

14. IP-based rate limiting must not be treated as sufficient identity
    protection for authenticated operations.

15. Request size limits must be explicit and bounded.

16. CORS is not authentication.

17. Cookie-based authentication must account for CSRF.

18. Middleware must never log secrets or sensitive request/response bodies
    by default.

19. Metrics must use bounded-cardinality labels.

20. Middleware must propagate cancellation and deadlines correctly.

21. External calls from middleware should be avoided unless the middleware
    concern genuinely requires them.

22. Background workers must not pretend to be HTTP requests merely to reuse
    HTTP middleware.

23. Every middleware must have explicit failure behavior.

24. Security-sensitive middleware must never silently fail open.

25. The complete middleware chain must be tested, not only individual
    middleware functions.

26. Middleware should reject cheap, obviously invalid requests early where
    doing so does not violate security or architectural ordering.

27. If a middleware requires substantial domain logic to function,
    that logic probably belongs in the application layer.
```

## Final Request Flow

```text
                         HTTP Request
                              │
                              ▼
                    ┌──────────────────┐
                    │    Request ID    │
                    └────────┬─────────┘
                             ▼
                    ┌──────────────────┐
                    │     Tracing      │
                    └────────┬─────────┘
                             ▼
                    ┌──────────────────┐
                    │     Recovery     │
                    └────────┬─────────┘
                             ▼
                    ┌──────────────────┐
                    │ Request Limits   │
                    └────────┬─────────┘
                             ▼
                    ┌──────────────────┐
                    │ Security/CORS    │
                    └────────┬─────────┘
                             ▼
                    ┌──────────────────┐
                    │ Logging/Metrics  │
                    └────────┬─────────┘
                             ▼
                    ┌──────────────────┐
                    │ Authentication   │
                    └────────┬─────────┘
                             ▼
                    ┌──────────────────┐
                    │ Tenant Resolution│
                    └────────┬─────────┘
                             ▼
                    ┌──────────────────┐
                    │  Rate Limiting   │
                    └────────┬─────────┘
                             ▼
                    ┌──────────────────┐
                    │  Authorization   │
                    └────────┬─────────┘
                             ▼
                    ┌──────────────────┐
                    │     Handler      │
                    └────────┬─────────┘
                             ▼
                    ┌──────────────────┐
                    │   Application    │
                    └────────┬─────────┘
                             ▼
                    ┌──────────────────┐
                    │      Domain      │
                    └────────┬─────────┘
                             ▼
                    ┌──────────────────┐
                    │  Infrastructure  │
                    └──────────────────┘
```

Middleware should make the request **safe, identifiable, bounded, observable, authenticated, and correctly scoped** before it reaches application logic. It should not become the place where KAMPYN's actual business behavior lives.
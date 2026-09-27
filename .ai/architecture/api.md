# KAMPYN API Architecture

## 1. Purpose

This document defines the architectural rules for KAMPYN APIs.

The API layer is a contract between clients and the application.

It must provide:

- Stable contracts.
- Clear resource boundaries.
- Predictable behavior.
- Strong validation.
- Consistent errors.
- Secure authorization.
- Tenant isolation.
- Safe retries.
- Observable requests.
- Backward compatibility.
- Efficient data access.

The API must not become a place where business logic, database logic, and infrastructure concerns are mixed together.

---

# 2. API Architecture

KAMPYN APIs follow this general flow:

```text
Client
  ↓
API Gateway / Edge
  ↓
Authentication
  ↓
Request Validation
  ↓
Authorization
  ↓
Transport / Handler
  ↓
Application Service
  ↓
Domain Logic
  ↓
Repository / Infrastructure
  ↓
Database / External Service
```

Responses travel back through the appropriate boundaries:

```text
Database / External Service
  ↓
Infrastructure
  ↓
Domain / Application
  ↓
Response DTO
  ↓
API Handler
  ↓
HTTP Response
  ↓
Client
```

Each layer must have a clear responsibility.

---

# 3. API Layer Responsibilities

The API layer is responsible for:

- Receiving requests.
- Parsing transport-level data.
- Validating request structure.
- Establishing request context.
- Authenticating the caller.
- Enforcing authorization through application/domain boundaries.
- Calling application services.
- Mapping application results to API responses.
- Mapping known errors to HTTP responses.
- Producing API-level observability.

The API layer must not directly contain:

- Complex business workflows.
- Database queries.
- Domain persistence logic.
- Infrastructure-specific implementation.
- Long-running business processes.

---

# 4. API Styles

KAMPYN primarily uses HTTP/REST-style APIs unless a specific use case requires another protocol.

Use:

- HTTP APIs for standard application operations.
- WebSockets or equivalent real-time protocols for genuinely real-time interactions.
- Background jobs/events for asynchronous workflows.
- Internal service communication according to the established backend architecture.

Do not introduce GraphQL, gRPC, or another API style merely for novelty.

The protocol must match the problem.

---

# 5. Resource-Oriented Design

APIs should model domain resources and actions clearly.

Prefer:

```text
GET    /api/v1/food-courts
GET    /api/v1/food-courts/{foodCourtId}
POST   /api/v1/orders
GET    /api/v1/orders/{orderId}
PATCH  /api/v1/orders/{orderId}
```

over vague endpoint structures such as:

```text
POST /api/v1/doSomething
POST /api/v1/processData
POST /api/v1/manage
```

Use action endpoints when the operation represents a meaningful domain action that does not map cleanly to ordinary CRUD semantics.

Examples:

```text
POST /api/v1/orders/{orderId}/cancel
POST /api/v1/bookings/{bookingId}/confirm
POST /api/v1/payments/{paymentId}/refund
```

---

# 6. URL Structure

URLs should be:

- Predictable.
- Stable.
- Resource-oriented.
- Lowercase.
- Consistent.

Prefer:

```text
/api/v1/food-courts
/api/v1/food-courts/{foodCourtId}/vendors
```

Avoid inconsistent naming such as:

```text
/api/getFoodCourts
/api/FoodCourt
/api/foodCourtList
```

Do not expose database table names merely because they already exist.

API resources represent domain concepts, not database implementation details.

---

# 7. Versioning

Public APIs must have an explicit compatibility strategy.

When URL versioning is used:

```text
/api/v1/...
/api/v2/...
```

Breaking changes require an intentional versioning strategy.

Do not introduce a new version for every small change.

Backward-compatible additions should generally remain within the existing version.

Breaking changes may include:

- Removing fields.
- Changing field meaning.
- Changing field types.
- Changing required inputs.
- Removing endpoints.
- Changing authorization semantics.
- Changing pagination semantics incompatibly.

---

# 8. HTTP Methods

Use HTTP methods according to their semantics.

### GET

Used for retrieval.

GET requests should not cause business state changes.

### POST

Used for creation or explicit actions.

### PUT

Use for full replacement when the API contract explicitly defines replacement semantics.

### PATCH

Use for partial updates.

### DELETE

Use for deletion or removal where appropriate.

Do not use POST for every operation merely because it is convenient.

---

# 9. HTTP Status Codes

Use status codes consistently.

Typical mappings:

```text
200 OK
201 Created
202 Accepted
204 No Content

400 Bad Request
401 Unauthorized
403 Forbidden
404 Not Found
409 Conflict
422 Unprocessable Entity
429 Too Many Requests

500 Internal Server Error
502 Bad Gateway
503 Service Unavailable
504 Gateway Timeout
```

Do not return `200 OK` for every operation with an embedded error object.

The HTTP status must communicate the broad outcome.

---

# 10. Authentication

Authentication must occur before protected application operations.

The API must establish an authenticated identity containing the information required by downstream authorization.

Authentication mechanisms may include:

- Session-based authentication.
- Access tokens.
- Refresh tokens.
- Service credentials.
- Institution-specific identity providers.

The implementation must follow the security architecture defined elsewhere.

Never trust identity information supplied directly by the client without cryptographic or server-side verification.

---

# 11. Authorization

Authentication answers:

```text
Who is this?
```

Authorization answers:

```text
What may this identity do?
```

Authorization must be enforced server-side.

Check:

- User permissions.
- Roles.
- Resource ownership.
- Tenant membership.
- Administrative privileges.
- Action-specific permissions.

Do not rely on frontend controls for authorization.

---

# 12. Object-Level Authorization

Every resource identified by a client-controlled ID must be authorized.

For example:

```text
GET /api/v1/orders/{orderId}
```

must verify that the caller is allowed to access that specific order.

Do not assume:

```text
authenticated user
+
valid order ID
=
authorized access
```

This applies to:

- Orders.
- Bookings.
- Complaints.
- Hostel records.
- Payments.
- Files.
- Community resources.
- Administrative resources.

---

# 13. Multi-Tenancy

KAMPYN may serve multiple universities or organizations.

Tenant context must be established explicitly.

Every tenant-scoped API request must ensure:

```text
Authenticated identity
        ↓
Tenant membership
        ↓
Resource authorization
        ↓
Operation
```

Tenant identifiers must never be trusted merely because the client supplied them.

The server must determine or validate the caller's tenant context.

Every database query, cache operation, search operation, and background workflow must preserve tenant isolation.

---

# 14. Request Validation

Every external API request is untrusted.

Validate:

- Path parameters.
- Query parameters.
- Request bodies.
- Headers where applicable.
- File metadata.
- Pagination values.
- Filters.
- Sorting fields.
- Enum values.
- String lengths.
- Numeric ranges.

Use the project's approved validation mechanism, including Zod where appropriate.

Do not rely on TypeScript types alone for runtime input validation.

---

# 15. Unknown Fields

Request contracts should define how unknown fields are handled.

Rejecting unexpected fields is preferred when:

- Strict contracts are required.
- Silent client mistakes would be dangerous.
- Security-sensitive operations are involved.

Allowing unknown fields may be appropriate for explicitly extensible payloads.

The behavior must be intentional and consistent.

Never silently interpret unknown fields as trusted business instructions.

---

# 16. Request Size Limits

APIs must impose appropriate request limits.

Consider:

- JSON body size.
- File upload size.
- Number of array elements.
- String lengths.
- Query parameter lengths.
- Batch operation size.

Never allow clients to submit arbitrarily large payloads to ordinary synchronous endpoints.

Large operations should use appropriate asynchronous or streaming workflows.

---

# 17. Pagination

Collection endpoints must be bounded.

Never return an unbounded collection.

Example:

```text
GET /api/v1/orders?limit=50
```

The server must enforce a maximum limit.

For large or frequently changing datasets, prefer cursor/keyset pagination.

Example:

```text
GET /api/v1/orders?limit=50&cursor=eyJpZCI6...
```

Pagination behavior must be documented and stable.

---

# 18. Cursor Pagination

Cursor-based pagination should use a stable ordering.

The cursor should contain enough information to continue from the previous position without relying on mutable or ambiguous ordering.

Avoid using:

```text
ORDER BY created_at
```

alone when multiple records can share the same timestamp.

Use a deterministic tie-breaker where necessary.

Example:

```text
ORDER BY created_at DESC, id DESC
```

The cursor must not expose sensitive internal information unnecessarily.

---

# 19. Filtering

Filtering parameters must be explicit.

Prefer:

```text
GET /api/v1/orders?status=completed
```

over accepting arbitrary database expressions.

Do not expose raw database query syntax to clients.

Filters must be:

- Validated.
- Bounded.
- Authorized.
- Efficiently queryable.

---

# 20. Sorting

Sorting must use an allowlist of supported fields.

Example:

```text
sort=createdAt
order=desc
```

Do not allow clients to submit arbitrary SQL/database field expressions.

The API layer should map public API fields to internal database fields where necessary.

---

# 21. Field Selection

Field selection may be supported where payload reduction provides meaningful value.

If implemented:

```text
?fields=id,name,status
```

must use an explicit allowlist.

Never expose arbitrary database columns through a generic field-selection mechanism.

Sensitive fields must never become accessible merely because a client requests them.

---

# 22. Request DTOs

API request objects must not automatically become domain entities.

Prefer:

```text
HTTP Request
    ↓
Request DTO
    ↓
Application Command
    ↓
Domain Model
```

This prevents transport-specific concerns from leaking into the domain.

Request DTOs should represent the API contract, not database schemas.

---

# 23. Response DTOs

Database entities must not automatically become API responses.

Prefer:

```text
Domain/Application Result
    ↓
Response DTO
    ↓
HTTP Response
```

Response DTOs provide a stable public contract even when internal storage changes.

Never expose:

- Password hashes.
- Internal credentials.
- Private database fields.
- Internal service metadata.
- Unnecessary infrastructure details.

---

# 24. Response Shape

Response structures should remain consistent.

A resource response should contain only fields required by the contract.

Collection responses should have a predictable structure.

For example:

```json
{
  "data": [],
  "pagination": {
    "nextCursor": null,
    "hasMore": false
  }
}
```

The exact envelope must follow the project's established API convention.

Do not introduce multiple response conventions for equivalent resources.

---

# 25. Error Responses

Errors must use a consistent structure.

A typical structure may be:

```json
{
  "error": {
    "code": "ORDER_NOT_FOUND",
    "message": "Order not found.",
    "details": {}
  }
}
```

The exact schema must remain consistent across services.

Do not expose:

- Stack traces.
- SQL errors.
- Internal hostnames.
- Secrets.
- Internal implementation details.

---

# 26. Error Codes

Use stable machine-readable error codes.

Examples:

```text
VALIDATION_ERROR
UNAUTHORIZED
FORBIDDEN
RESOURCE_NOT_FOUND
RESOURCE_CONFLICT
RATE_LIMITED
INTERNAL_ERROR
```

Domain-specific errors may use more specific codes:

```text
ORDER_ALREADY_CANCELLED
INSUFFICIENT_INVENTORY
BOOKING_SLOT_UNAVAILABLE
```

Error codes should remain stable even if human-readable messages change.

---

# 27. Validation Errors

Validation failures should identify the affected fields where appropriate.

Example:

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Request validation failed.",
    "fields": {
      "email": "Invalid email address.",
      "quantity": "Must be greater than zero."
    }
  }
}
```

Do not expose internal validation implementation details.

---

# 28. Not Found vs Forbidden

Do not automatically reveal whether a resource exists when doing so would create an information disclosure risk.

For sensitive resources, the API may intentionally return a generalized response.

The behavior must be deliberate and consistent with the security model.

---

# 29. Idempotency

Operations that may be retried must define idempotency behavior.

This is especially important for:

- Orders.
- Payments.
- Reservations.
- Bookings.
- Inventory changes.
- Webhooks.

Where appropriate, support:

```text
Idempotency-Key: <unique-key>
```

The server must define:

- Key scope.
- Key lifetime.
- Request matching.
- Result reuse.
- Conflict behavior.
- Storage.

Do not claim an operation is idempotent merely because repeated calls usually produce the same result.

---

# 30. Retry Safety

Clients, gateways, and workers may retry requests.

API operations must define which requests are safely retryable.

GET requests should generally be safe to retry.

Non-idempotent mutations require explicit handling.

Never make a payment, reservation, or inventory operation accidentally repeatable because of automatic client retries.

---

# 31. Long-Running Operations

Do not keep synchronous HTTP requests open for unnecessarily long operations.

For long-running work use:

```text
POST /api/v1/exports
        ↓
202 Accepted
        ↓
Background Job
        ↓
GET /api/v1/exports/{exportId}
```

Appropriate asynchronous workflows may use:

- Job IDs.
- Status resources.
- Webhooks.
- WebSockets.
- Notifications.

The operation's state machine must be explicit.

---

# 32. Transactions and API Operations

API handlers must not create arbitrary transaction boundaries.

The application service should own the transaction when a complete business operation requires atomicity.

Example:

```text
HTTP Request
    ↓
Handler
    ↓
Application Service
    ↓
Transaction
    ├── Update order
    ├── Update inventory
    └── Create outbox event
    ↓
Commit
```

Do not keep database transactions open while waiting for external services.

---

# 33. External APIs

External service integrations must be isolated behind infrastructure adapters.

The API layer must not directly depend on provider-specific SDK behavior.

Prefer:

```text
Application
    ↓
Port / Interface
    ↓
Provider Adapter
    ↓
External API
```

External integrations must define:

- Timeout.
- Retry policy.
- Error mapping.
- Authentication.
- Rate limits.
- Idempotency.
- Observability.

---

# 34. API Timeouts

Every external dependency must have an appropriate timeout.

Never allow an external service to block an API request indefinitely.

Timeouts should be propagated through request context where supported.

The timeout should reflect the operation's actual latency requirements.

Do not simply increase timeouts to hide poor performance.

---

# 35. API Rate Limiting

Rate limiting should be applied according to endpoint sensitivity and expected traffic.

Consider limits for:

- Authentication.
- Password reset.
- Search.
- File uploads.
- Expensive queries.
- Public APIs.
- Administrative endpoints.
- Payment operations.

Rate limits should account for:

- User.
- Tenant.
- IP where appropriate.
- API key.
- Endpoint.

Do not use IP-only limiting as the sole mechanism when authenticated identities are available.

---

# 36. Bulk APIs

Bulk operations must be explicitly designed.

Example:

```text
POST /api/v1/orders/bulk
```

must define:

- Maximum batch size.
- Validation behavior.
- Partial failure semantics.
- Transaction semantics.
- Idempotency.
- Response structure.
- Processing time.

Do not create bulk endpoints merely to hide inefficient client behavior.

---

# 37. File Upload APIs

File uploads must define:

- Maximum size.
- Allowed types.
- Content validation.
- Storage location.
- Filename handling.
- Authorization.
- Virus/malware scanning where required.
- Processing lifecycle.

Do not trust:

- Filename extensions.
- MIME types supplied by the client.
- Client-generated paths.

Large files should generally use object storage or an appropriate upload workflow rather than passing entire files through ordinary application handlers.

---

# 38. Search APIs

Search endpoints should abstract the search implementation from clients.

Example:

```text
GET /api/v1/search?q=chicken&category=food
```

The client should not know whether results come from:

- PostgreSQL.
- MongoDB.
- OpenSearch.
- A combined search system.

Search APIs must enforce:

- Tenant isolation.
- Query limits.
- Result limits.
- Pagination.
- Authorization.
- Appropriate ranking configuration.

---

# 39. Cache Interaction

APIs may use Redis or other caches to improve performance.

Cache behavior must define:

- Key structure.
- TTL.
- Invalidation.
- Staleness tolerance.
- Tenant isolation.
- Failure behavior.

Never allow a cache hit to bypass authorization.

Authorization must remain valid even when data is served from cache.

---

# 40. Webhooks

Webhook endpoints must authenticate incoming requests.

Use mechanisms such as:

- Signature verification.
- Shared secrets.
- Provider-specific verification.

Webhook handlers should:

1. Validate the request.
2. Authenticate the sender.
3. Deduplicate events.
4. Persist required state.
5. Return promptly.
6. Process expensive work asynchronously where appropriate.

Never assume a webhook is delivered exactly once.

---

# 41. Events and Asynchronous APIs

API-triggered events must be designed for:

- At-least-once delivery.
- Duplicate delivery.
- Ordering where required.
- Retry.
- Dead-letter handling.
- Idempotent consumers.

Do not build correctness around an assumption of exactly-once delivery unless the underlying system explicitly provides and guarantees it.

---

# 42. Health Endpoints

Health endpoints should distinguish between:

### Liveness

Whether the process is running.

### Readiness

Whether the instance can safely receive traffic.

Do not make liveness checks depend on every external dependency.

A temporary database outage should not necessarily cause the process to be restarted indefinitely.

Readiness may appropriately fail when required dependencies are unavailable.

---

# 43. API Observability

Every meaningful API request should be traceable.

Where applicable capture:

- Correlation/request ID.
- Route.
- Method.
- Status.
- Latency.
- Tenant context where safe.
- User/service identity where safe.
- Error category.

Do not log sensitive request or response bodies indiscriminately.

---

# 44. API Performance

API performance should be evaluated across:

```text
Network
  ↓
Serialization
  ↓
Application logic
  ↓
Database
  ↓
External services
```

Avoid:

- Excessive round trips.
- N+1 database queries.
- Unbounded responses.
- Repeated external requests.
- Large unnecessary payloads.
- Blocking work in synchronous handlers.

Use batch operations and caching where they provide measurable value.

---

# 45. API Compatibility

Public API changes must consider existing consumers.

Backward-compatible changes generally include:

- Adding optional response fields.
- Adding new endpoints.
- Adding optional request fields where safe.

Potentially breaking changes include:

- Removing fields.
- Renaming fields.
- Changing types.
- Changing required fields.
- Changing semantics.
- Removing endpoints.
- Changing authorization behavior.

Breaking changes require an explicit migration strategy.

---

# 46. Deprecation

Deprecated endpoints or fields must have:

- Clear documentation.
- Deprecation status.
- Migration guidance.
- Removal criteria.
- Appropriate versioning strategy.

Do not leave deprecated APIs indefinitely without ownership.

---

# 47. API Documentation

Every public API must be documented sufficiently for consumers to use it correctly.

Documentation should include:

- Endpoint.
- Method.
- Authentication.
- Authorization requirements.
- Request parameters.
- Request body.
- Response.
- Errors.
- Pagination.
- Idempotency.
- Examples where useful.

OpenAPI should be used where appropriate as a machine-readable API contract.

Documentation must remain synchronized with implementation.

---

# 48. SDK Compatibility

KAMPYN may provide SDKs for institutions and third-party consumers.

API changes must consider SDK consumers.

When changing a public contract:

1. Identify affected SDKs.
2. Update generated types/client code where applicable.
3. Update documentation.
4. Preserve compatibility where possible.
5. Define migration behavior for breaking changes.

Do not silently break self-hosted or external consumers.

---

# 49. API Security Review

Before approving an API change, verify:

- Authentication.
- Authorization.
- Object-level authorization.
- Tenant isolation.
- Input validation.
- Request limits.
- Rate limiting.
- Sensitive data exposure.
- Error exposure.
- File handling.
- Idempotency.
- Webhook verification.
- Audit requirements.

Security-sensitive endpoints require additional testing.

---

# 50. API Testing

API tests should cover:

### Success
- Valid request.
- Correct response.
- Correct status code.

### Validation
- Missing fields.
- Invalid types.
- Invalid values.
- Unknown fields where relevant.
- Oversized requests.

### Authorization
- Unauthenticated request.
- Unauthorized role.
- Wrong tenant.
- Wrong resource owner.

### Concurrency
- Duplicate requests.
- Concurrent updates.
- Race-sensitive operations.

### Reliability
- Timeout.
- Dependency failure.
- Retry.
- Partial failure.

### Compatibility
- Existing clients.
- Versioned contracts.
- Deprecated behavior where required.

---

# 51. API Change Checklist

Before completing an API change:

- [ ] Resource boundary is clear.
- [ ] URL structure is consistent.
- [ ] HTTP method semantics are correct.
- [ ] Authentication is enforced.
- [ ] Authorization is enforced.
- [ ] Tenant isolation is preserved.
- [ ] Request validation exists.
- [ ] Request size is bounded.
- [ ] Pagination is bounded.
- [ ] Filtering is allowlisted.
- [ ] Sorting is allowlisted.
- [ ] Response DTO is explicit.
- [ ] Sensitive fields are excluded.
- [ ] Error contract is consistent.
- [ ] Error codes are stable.
- [ ] Idempotency is defined where required.
- [ ] Retry behavior is safe.
- [ ] External calls have timeouts.
- [ ] Rate limiting is considered.
- [ ] Database transaction boundaries are correct.
- [ ] Cache behavior is safe.
- [ ] Events/webhooks are idempotent where required.
- [ ] Observability is sufficient.
- [ ] API documentation is updated.
- [ ] SDK compatibility is considered.
- [ ] Tests cover important success and failure paths.
- [ ] Backward compatibility is considered.

---

# 52. Final API Principle

An API is a long-lived contract.

Prefer:

```text
Explicit contract
      ↓
Validated input
      ↓
Authenticated identity
      ↓
Authorized operation
      ↓
Application service
      ↓
Correct domain behavior
      ↓
Stable response
      ↓
Observable result
```

The API should expose what consumers need to accomplish their work without exposing internal implementation details
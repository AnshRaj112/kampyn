# KAMPYN Backend API Design

## 1. Purpose

This document defines the standards for designing, implementing, evolving, and consuming KAMPYN APIs.

The API is the primary contract between:

```text id="m4gkqf"
Next.js Frontend
       │
       ├── Web Application
       ├── Admin Application
       └── Other Clients
              │
              ▼
        KAMPYN API
              │
       ┌──────┼──────┐
       │      │      │
      SDK   Mobile  External
            Clients Integrations
```

The API must provide:

- predictable resource semantics
- strong validation
- explicit authorization
- tenant isolation
- consistent errors
- stable contracts
- safe retries
- efficient pagination
- observable operations
- backwards compatibility
- clear documentation

The core principle is:

> **The API is a stable contract, not a direct reflection of database tables or internal implementation.**

---

# 2. API Architecture

KAMPYN APIs follow:

```text id="g2w4h9"
Client
  │
  ▼
Edge / Gateway
  │
  ▼
Authentication
  │
  ▼
Tenant Resolution
  │
  ▼
Authorization
  │
  ▼
Request Validation
  │
  ▼
Transport Handler
  │
  ▼
Application Use Case
  │
  ▼
Domain
  │
  ▼
Repository / Integration
  │
  ▼
Infrastructure
```

The API layer must not bypass application and domain boundaries.

---

# 3. API Responsibilities

The API layer is responsible for:

- HTTP transport
- route matching
- request parsing
- request validation
- authentication context extraction
- tenant context extraction
- authorization enforcement
- calling application use cases
- response serialization
- HTTP status mapping
- error serialization
- pagination contracts
- API-level observability

The API layer is not responsible for:

- database queries
- business rules
- domain calculations
- persistence transactions
- direct OpenSearch queries
- payment-provider orchestration
- arbitrary external API calls

---

# 4. API Design Principles

KAMPYN APIs must be:

```text id="44t2rd"
Predictable
Consistent
Explicit
Typed
Secure
Tenant-aware
Idempotent where required
Observable
Backward-compatible
```

Prefer simple, explicit contracts over clever abstractions.

---

# 5. Base URL

Production APIs should use a stable API namespace.

Example:

```text id="5vkrp4"
/api/v1
```

Resources then follow:

```text id="4d0s2p"
/api/v1/orders
/api/v1/items
/api/v1/vendors
/api/v1/bookings
```

The API version should not be repeated unnecessarily inside every resource name.

Avoid:

```text id="6z6vcs"
/api/v1/v1/orders
```

---

# 6. Versioning

Externally consumed APIs must have an explicit versioning strategy.

Recommended initial structure:

```text id="cz6ydh"
/api/v1/...
```

Breaking changes require a new API version or another explicitly documented compatibility mechanism.

Non-breaking changes should generally be introduced within the existing version.

Examples of generally non-breaking changes:

```text id="kzj8tw"
adding an optional response field
adding an optional request field
adding a new endpoint
adding a new enum value only when clients tolerate unknown values
```

Potential breaking changes include:

```text id="s2c8d0"
removing fields
renaming fields
changing field meaning
changing required fields
changing response structure
changing authentication semantics
changing error semantics
```

---

# 7. Resource-Oriented Design

API resources should represent meaningful domain concepts.

Examples:

```text id="l2ykj3"
/users
/tenants
/vendors
/food-courts
/items
/orders
/payments
/bookings
/hostels
/complaints
/notifications
```

Avoid exposing internal database table names.

For example, do not automatically create:

```text id="2m5o2f"
/order_items
/order_status_history
/user_tenant_mapping
```

unless those are genuine API resources.

---

# 8. Resource Naming

Use plural nouns for collection resources.

Prefer:

```text id="3k2yxn"
/orders
/items
/vendors
```

Avoid:

```text id="q5t5zq"
/getOrders
/createOrder
/orderList
```

HTTP methods express the action.

---

# 9. Resource Identifiers

Use stable resource identifiers.

Example:

```text id="d9p6ut"
/orders/{orderId}
/users/{userId}
/items/{itemId}
```

IDs should not expose internal database implementation unnecessarily.

Avoid making clients depend on:

```text id="3s1tgm"
database row IDs
Mongo-specific object representations
internal shard IDs
```

unless those identifiers are intentionally part of the public contract.

---

# 10. HTTP Methods

Use HTTP methods consistently.

### GET

Retrieve resources.

```text id="k1p1sg"
GET /orders/{orderId}
```

### POST

Create a resource or execute a non-idempotent action where appropriate.

```text id="k6k1su"
POST /orders
```

### PUT

Replace a resource where full replacement semantics are appropriate.

```text id="9w9rpf"
PUT /users/{userId}
```

### PATCH

Partially update a resource.

```text id="r4z4t8"
PATCH /users/{userId}
```

### DELETE

Remove or deactivate a resource according to the resource's lifecycle semantics.

```text id="f2r4p4"
DELETE /users/{userId}
```

Do not use POST for every operation simply because it is convenient.

---

# 11. Actions

Some operations are not naturally represented as CRUD.

Explicit action endpoints may be appropriate.

Examples:

```text id="o4a3gl"
POST /orders/{orderId}/cancel
POST /bookings/{bookingId}/confirm
POST /payments/{paymentId}/refund
POST /users/{userId}/verify
```

Use action endpoints when the operation represents a meaningful domain transition rather than a simple field mutation.

---

# 12. State Transitions

State transitions must be validated by the domain.

Example:

```text id="bq8y7q"
Order
  │
  ├── pending
  ├── confirmed
  ├── preparing
  ├── ready
  ├── completed
  └── cancelled
```

An API must not allow arbitrary status mutation:

```text id="h7y2mz"
PATCH /orders/123
{
  "status": "completed"
}
```

if the transition requires business rules.

Prefer:

```text id="4v3v9j"
POST /orders/123/complete
```

when completion is a domain operation.

---

# 13. HTTP Status Codes

Use status codes consistently.

### 200 OK

Successful retrieval or successful operation with a response body.

### 201 Created

Successful resource creation.

### 202 Accepted

Request accepted for asynchronous processing.

### 204 No Content

Successful operation without a response body.

### 400 Bad Request

Malformed or invalid request.

### 401 Unauthorized

Authentication is missing or invalid.

### 403 Forbidden

Identity is authenticated but lacks permission.

### 404 Not Found

Requested resource does not exist or is intentionally hidden.

### 409 Conflict

Request conflicts with current state or uniqueness constraints.

### 422 Unprocessable Content

Request is structurally valid but fails semantic validation where this distinction is useful.

### 429 Too Many Requests

Rate limit exceeded.

### 500 Internal Server Error

Unexpected server failure.

### 502 Bad Gateway

Upstream dependency failure where the gateway/service acts as an intermediary.

### 503 Service Unavailable

Service or critical dependency is temporarily unavailable.

### 504 Gateway Timeout

Upstream operation exceeded its timeout.

The exact status mapping must remain consistent across the API.

---

# 14. Successful Response Structure

Responses should have predictable structures.

For example:

```json id="w3z3fl"
{
  "data": {
    "id": "order_123",
    "status": "confirmed"
  }
}
```

Collections may use:

```json id="rj3j2r"
{
  "data": [
    {
      "id": "order_123"
    },
    {
      "id": "order_124"
    }
  ],
  "pagination": {
    "next_cursor": "..."
  }
}
```

The exact envelope should be standardized rather than reinvented endpoint by endpoint.

---

# 15. Response DTOs

API responses must use explicit DTOs.

Do not return database models directly.

Bad:

```text id="2r2k9v"
Database Model
      ↓
JSON
```

Prefer:

```text id="c6r4bq"
Domain / Application Result
      ↓
Response DTO
      ↓
JSON
```

This prevents persistence changes from silently becoming API changes.

---

# 16. Request DTOs

Requests should also use explicit DTOs.

Example:

```go id="r5e5z4"
type CreateOrderRequest struct {
    Items []CreateOrderItem `json:"items"`
}
```

Request DTOs should represent the API contract, not internal domain entities.

---

# 17. Runtime Validation

TypeScript and Go compile-time types do not replace runtime validation.

Every untrusted API input must be validated.

Validate:

```text id="2f1h8n"
body
query parameters
path parameters
headers where applicable
multipart metadata
webhook payloads
external integration responses
```

For frontend/shared TypeScript contracts, Zod may be used for runtime validation.

Backend validation remains authoritative.

---

# 18. Validation Flow

The request pipeline should be:

```text id="6f1k2x"
HTTP Request
      │
      ▼
Parse
      │
      ▼
Validate
      │
      ▼
Normalize
      │
      ▼
Authenticate
      │
      ▼
Authorize
      │
      ▼
Application Use Case
```

Validation should happen before expensive operations.

---

# 19. Validation Errors

Validation failures should identify the affected fields without exposing implementation details.

Example:

```json id="5w9h4v"
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Request validation failed",
    "fields": {
      "items": "At least one item is required"
    }
  }
}
```

Error codes must remain stable even if internal validation libraries change.

---

# 20. Error Response Contract

KAMPYN should use a consistent error structure.

Example:

```json id="2f8b0x"
{
  "error": {
    "code": "ORDER_NOT_FOUND",
    "message": "Order was not found",
    "request_id": "req_123"
  }
}
```

Potential fields:

```text id="3x0h0d"
code
message
fields
request_id
details
```

Do not expose:

```text id="1v7q8x"
stack traces
SQL
database credentials
internal service topology
provider secrets
```

---

# 21. Error Codes

Error codes should be machine-readable.

Examples:

```text id="6m4d4h"
VALIDATION_ERROR
UNAUTHENTICATED
FORBIDDEN
NOT_FOUND
CONFLICT
RATE_LIMITED
ORDER_NOT_FOUND
ORDER_ALREADY_CANCELLED
INSUFFICIENT_INVENTORY
PAYMENT_FAILED
PAYMENT_PENDING
DEPENDENCY_UNAVAILABLE
INTERNAL_ERROR
```

Clients should not rely on parsing human-readable messages.

---

# 22. Error Code Ownership

Error codes should represent stable application semantics.

Avoid exposing low-level infrastructure errors such as:

```text id="2t7p1h"
PG_UNIQUE_VIOLATION_23505
MONGO_DUPLICATE_KEY
REDIS_TIMEOUT
```

Translate them into appropriate application-level errors.

---

# 23. Request IDs

Every API request should have a correlation/request ID.

Example:

```text id="8x0a4q"
X-Request-ID: req_123
```

If a trusted request ID is supplied, the system may propagate it according to the request-ID policy.

Otherwise the API should generate one.

The request ID should appear in:

```text id="7f2e6u"
logs
traces
error responses where appropriate
```

---

# 24. Distributed Trace Context

API requests should propagate trace context where supported.

```text id="c7a4kt"
Frontend
   │
   ▼
API
   │
   ▼
Application
   │
   ├── Database
   ├── Redis
   ├── OpenSearch
   └── External Provider
```

Trace context must not be treated as an authorization mechanism.

---

# 25. Authentication

Authenticated endpoints must establish identity before authorization.

```text id="8i9c7s"
Request
  │
  ▼
Authentication
  │
  ▼
Identity
```

Supported mechanisms may include:

```text id="4k5j7q"
session cookies
OAuth/OIDC
access tokens
refresh tokens
SSO
API keys
service credentials
```

The authentication architecture is defined separately in `architecture/authentication.md`.

---

# 26. Authorization

Authentication answers:

> Who are you?

Authorization answers:

> What are you allowed to do?

The API must enforce authorization server-side.

Example:

```text id="p4t1k9"
Identity
   ↓
Tenant Membership
   ↓
Role / Permission
   ↓
Resource Scope
   ↓
Action
```

Frontend visibility is not an authorization mechanism.

---

# 27. Tenant Context

Every tenant-scoped endpoint must establish tenant context.

Possible sources include:

```text id="f7z2q8"
subdomain
custom domain
authenticated membership
explicit tenant selection
deployment configuration
```

The backend must validate that the authenticated identity is allowed to operate within the resolved tenant.

---

# 28. Tenant IDs in Requests

Do not blindly trust:

```text id="p9f8a1"
tenant_id
```

provided by a client.

For example:

```text id="8s8r5x"
POST /orders
{
  "tenant_id": "tenant_B"
}
```

must not allow a user belonging only to tenant A to create an order in tenant B.

Tenant scope should be established by trusted server-side context.

---

# 29. Resource Authorization

Resource ownership must be verified.

Example:

```text id="j6x2q5"
GET /orders/order_123
```

requires checking:

```text id="j8b8l4"
authenticated identity
        +
tenant membership
        +
permission
        +
order visibility / ownership
```

Do not assume possession of an ID grants access.

---

# 30. Hidden Resources

In some security-sensitive cases, the API may intentionally return:

```text id="f5l1se"
404 Not Found
```

instead of:

```text id="6z8s4h"
403 Forbidden
```

to avoid revealing the existence of unauthorized resources.

The behavior must be consistent with the resource's security requirements.

---

# 31. Query Parameters

Query parameters should be used for retrieval controls.

Examples:

```text id="5j8p8v"
GET /orders?status=pending
GET /items?category=biryani
GET /vendors?active=true
```

Avoid using query parameters to express arbitrary commands.

---

# 32. Filtering

Filters must be explicitly supported.

Example:

```text id="w0q9x6"
/orders?status=pending&created_after=2026-01-01
```

Do not expose arbitrary database expressions.

Avoid:

```text id="r2v6sy"
/orders?where=status%3Dpending
```

---

# 33. Sorting

Sorting must use an allowlist.

Example:

```text id="8l9n8s"
/items?sort=price_asc
```

Supported values might be:

```text id="5n7n0v"
relevance
created_desc
created_asc
price_asc
price_desc
rating_desc
```

Never directly inject a client-provided field name into a database query.

---

# 34. Pagination

Collection endpoints must have bounded pagination.

Example:

```text id="n2k9fv"
/orders?limit=20&cursor=abc
```

The API should define:

```text id="3l0qk7"
default page size
maximum page size
cursor format
cursor expiration if applicable
ordering guarantees
```

Avoid unbounded collection responses.

---

# 35. Cursor Pagination

Cursor pagination is preferred for large or frequently changing datasets.

Example:

```json id="a3c9y4"
{
  "data": [],
  "pagination": {
    "next_cursor": "eyJpZCI6IjEyMyJ9",
    "has_more": true
  }
}
```

The cursor should encode sufficient state to continue the query safely.

Opaque cursors are preferable to exposing database implementation details.

---

# 36. Offset Pagination

Offset pagination may be appropriate for:

- small datasets
- administrative interfaces
- stable datasets
- simple use cases

Example:

```text id="q2t7r1"
/users?page=2&limit=50
```

Do not use large offsets for high-volume datasets when cursor pagination is more appropriate.

---

# 37. Pagination Ordering

Pagination must have deterministic ordering.

Avoid:

```text id="1e0h0m"
ORDER BY created_at
```

if `created_at` is not unique enough to guarantee stable pagination.

Prefer a stable tie-breaker:

```text id="f4g4xk"
ORDER BY created_at DESC, id DESC
```

The cursor must preserve the ordering semantics.

---

# 38. Field Selection

Field selection may be supported where payload size materially matters.

Example:

```text id="5o3l8k"
/users?fields=id,name,avatar
```

If implemented, allowed fields must be explicitly defined.

Never expose arbitrary internal database fields through field selection.

---

# 39. Expansion

Related resources should not be automatically loaded without control.

If expansion is supported:

```text id="8h0m3s"
/orders/123?include=items
```

the allowed relationships must be explicitly defined.

Avoid recursive or unbounded object expansion.

---

# 40. Nested Resources

Use nested routes when the relationship is meaningful and bounded.

Example:

```text id="j0b3j6"
/food-courts/{foodCourtId}/vendors
```

However, avoid excessively deep nesting:

```text id="q9l7m3"
/universities/{id}/food-courts/{id}/vendors/{id}/items/{id}/orders
```

Deep routes are difficult to maintain and can obscure authorization boundaries.

---

# 41. Relationship APIs

When a relationship is important but not truly subordinate, use top-level resources.

For example:

```text id="g6y6oq"
/vendors/{vendorId}
/food-courts/{foodCourtId}
```

instead of forcing every operation through nested paths.

---

# 42. Bulk APIs

Bulk operations should be explicit.

Example:

```text id="4s9w8c"
POST /admin/users/bulk-disable
```

Bulk APIs must define:

```text id="z6j6m3"
maximum batch size
authorization
partial failure behavior
idempotency
transaction semantics
rate limits
audit behavior
```

Do not allow arbitrary unbounded bulk requests.

---

# 43. Partial Failure

For bulk operations, the response must clearly communicate partial success.

Example:

```json id="3w4n1k"
{
  "data": {
    "processed": 95,
    "succeeded": 93,
    "failed": 2,
    "failures": [
      {
        "id": "user_123",
        "code": "USER_NOT_FOUND"
      }
    ]
  }
}
```

The exact structure should be standardized.

---

# 44. Idempotency

Operations that may be retried and create side effects should support idempotency.

Examples:

```text id="s5q9be"
payment creation
order submission
booking creation
refund
external webhook processing
large import initiation
```

Clients may provide:

```text id="g5d6sm"
Idempotency-Key: <unique-key>
```

The server must define:

- key scope
- retention
- request matching
- result replay behavior
- conflict behavior

---

# 45. Idempotency Semantics

For a repeated request:

```text id="6p4z1n"
same idempotency key
+
same operation
+
same request
```

the API should return the original effective result rather than performing the side effect again.

If the same key is reused with a materially different request, return a conflict.

---

# 46. Safe vs Unsafe HTTP Methods

GET should be safe and must not cause business mutations.

Avoid:

```text id="0t7m8k"
GET /orders/123/cancel
```

Use:

```text id="2t8g6m"
POST /orders/123/cancel
```

instead.

---

# 47. Conditional Requests

Conditional operations may use mechanisms such as:

```text id="f1q2d8"
ETag
If-Match
If-None-Match
```

when concurrent updates require optimistic concurrency control.

This can prevent:

```text id="z4r9c3"
Client A reads version 5
Client B updates version 6
Client A overwrites version 6
```

without detecting the conflict.

---

# 48. Optimistic Concurrency

For resources requiring version protection:

```json id="n4p7k6"
{
  "version": 5
}
```

or an HTTP conditional mechanism may be used.

The authoritative implementation must reject stale updates.

Example:

```text id="x2p8g1"
Expected version: 5
Actual version: 6

        ↓

409 Conflict
```

---

# 49. Resource State

State should be represented consistently.

Example:

```json id="q3v5a8"
{
  "id": "order_123",
  "status": "confirmed"
}
```

Avoid exposing multiple contradictory representations:

```text id="g4r9c1"
status: "confirmed"
is_confirmed: true
state_code: 2
```

unless each has a clearly documented purpose.

---

# 50. Enum Design

Enum values should be:

- stable
- explicit
- documented
- machine-readable

Example:

```text id="2x4p7m"
pending
confirmed
cancelled
completed
```

Avoid relying on numeric enum values in public APIs:

```text id="6w2z3k"
status: 2
```

unless the contract explicitly requires it.

---

# 51. Dates and Times

API timestamps should use a standard representation.

Prefer ISO 8601 / RFC 3339-style timestamps:

```text id="5b3v2x"
2026-09-28T10:30:00Z
```

Store canonical timestamps consistently and convert for user presentation at the appropriate boundary.

---

# 52. Time Zones

Requests involving local schedules should explicitly represent timezone context.

For example:

```json id="8g7m2c"
{
  "starts_at": "2026-09-28T18:00:00+05:30",
  "timezone": "Asia/Kolkata"
}
```

Do not silently assume server-local time.

---

# 53. Monetary Values

Do not use floating-point values for authoritative monetary calculations.

API representations should use a precise representation.

Example:

```json id="2r5y9q"
{
  "amount": 18000,
  "currency": "INR"
}
```

where `amount` represents the smallest currency unit if that convention is adopted.

The convention must be consistent across the API.

---

# 54. Quantities

Quantities must have explicit semantics.

For example:

```json id="5k8z4d"
{
  "quantity": 2
}
```

should have a documented unit or domain meaning where necessary.

Avoid ambiguous numeric fields.

---

# 55. File Uploads

File uploads should use dedicated endpoints or multipart requests.

Example:

```text id="8m4k7s"
POST /files
```

The API must validate:

```text id="3r7v5q"
file size
content type
extension
filename
authorization
tenant
```

Large uploads should avoid routing unnecessarily through application memory.

---

# 56. File Downloads

Downloads must enforce authorization.

Do not expose permanent public URLs for private files unless the security model explicitly allows it.

Use:

```text id="0f4n7r"
authorized endpoint
```

or appropriately scoped temporary URLs.

---

# 57. Webhooks

Webhook endpoints are different from normal user APIs.

They must verify:

```text id="7g4h2k"
provider signature
timestamp
event ID
event type
payload integrity
```

Webhook processing must be idempotent.

---

# 58. Webhook Flow

Example:

```text id="7n2m6b"
External Provider
       │
       ▼
Webhook Endpoint
       │
       ▼
Signature Verification
       │
       ▼
Schema Validation
       │
       ▼
Idempotency Check
       │
       ▼
Application Workflow
       │
       ▼
Transaction
```

Do not trust webhook payloads simply because they arrive at a known URL.

---

# 59. Webhook Responses

Webhook endpoints should acknowledge accepted events quickly.

Long-running processing should normally move to asynchronous workers.

```text id="4x7p3n"
Webhook
  │
  ▼
Validate
  │
  ▼
Persist / Queue
  │
  ▼
202 Accepted
  │
  ▼
Worker
```

The exact behavior depends on the provider's retry contract.

---

# 60. Asynchronous API Operations

Long-running operations should return an asynchronous result.

Example:

```text id="2q8m5r"
POST /exports
        │
        ▼
202 Accepted
        │
        ▼
{
  "data": {
    "job_id": "job_123",
    "status": "queued"
  }
}
```

Clients can then query:

```text id="8f3k2d"
/jobs/job_123
```

Do not keep HTTP requests open unnecessarily for long-running work.

---

# 61. Job Resources

If jobs are externally visible, their lifecycle should be explicit.

Example:

```text id="r9w3p6"
queued
running
completed
failed
cancelled
```

The API should define:

- result availability
- failure semantics
- retry behavior
- expiration
- authorization

---

# 62. API Rate Limiting

Rate limits should protect against:

```text id="z7x4k1"
abuse
accidental overload
credential attacks
expensive queries
bulk operations
search abuse
```

Rate limiting may operate at:

```text id="g8m1p4"
IP
identity
tenant
API key
endpoint
resource
```

depending on the use case.

---

# 63. Rate Limit Responses

When throttled, return:

```text id="v6k2m9"
429 Too Many Requests
```

and, where appropriate:

```text id="4y8p1q"
Retry-After
```

The client should not blindly retry immediately.

---

# 64. Request Size Limits

Every endpoint must have reasonable request size limits.

Examples:

```text id="s7j4p2"
JSON body
multipart upload
query length
header size
bulk request size
```

Large data operations should use specialized upload or asynchronous workflows.

---

# 65. Search API

Search should use a dedicated contract rather than exposing OpenSearch.

Example:

```text id="w4q8m6"
GET /api/v1/search
```

Possible parameters:

```text id="7x2n5b"
q
type
category
vendor
food_court
price_min
price_max
availability
sort
cursor
limit
```

The backend translates the request into the appropriate search query.

---

# 66. Search Authorization

Search must apply:

```text id="c4m9v2"
tenant scope
visibility
resource permissions
moderation state
```

before returning results.

Never retrieve unrestricted search results and filter unauthorized data in application memory.

---

# 67. API and Caching

HTTP caching may be used for safe public or appropriately scoped resources.

Caching must account for:

```text id="n8p3k4"
tenant
authorization
resource state
ETag
Cache-Control
staleness
```

Do not cache personalized or sensitive responses publicly.

---

# 68. API and TanStack Query

The frontend should use TanStack Query for server state.

The API should therefore provide stable semantics for:

```text id="h5q7r2"
query keys
pagination
cache invalidation
mutations
resource versions
errors
```

The API must not assume that every request results in a fresh network call.

---

# 69. API and Zustand

Zustand should not become a duplicate server-state cache.

Use Zustand for:

```text id="d2v7m5"
UI state
client preferences
navigation state
temporary client state
```

Use APIs and TanStack Query for server-owned data.

---

# 70. API Contracts and Zod

For TypeScript clients, shared schemas may use Zod where appropriate.

Example:

```text id="h3v6q8"
API Response
     │
     ▼
Zod Validation
     │
     ▼
Typed Client Data
```

Do not assume TypeScript types alone guarantee runtime correctness.

---

# 71. API Documentation

Every public API must be documented.

Documentation should include:

```text id="q4n6p8"
endpoint
method
authentication
authorization
tenant requirements
request
response
status codes
errors
pagination
idempotency
rate limits
examples
deprecation information
```

OpenAPI should be used where appropriate as the machine-readable API contract.

---

# 72. OpenAPI

The API specification should describe:

- paths
- methods
- parameters
- request bodies
- response schemas
- authentication
- errors
- pagination
- examples

The OpenAPI definition must remain synchronized with implementation.

Do not allow the specification to become stale documentation.

---

# 73. SDK Generation

SDKs should be derived from stable API contracts where practical.

```text id="p2z7m9"
OpenAPI
   │
   ▼
SDK Generation
   │
   ├── TypeScript
   ├── Go
   └── Other supported languages
```

Generated SDKs should not contain business logic that belongs on the server.

---

# 74. API Compatibility

Before changing an endpoint, determine:

```text id="v5x8k3"
frontend consumers
SDK consumers
external integrations
webhooks
mobile clients
self-hosted installations
```

A change that is safe for the current frontend may still break external consumers.

---

# 75. Deprecation

Deprecated APIs must have a documented lifecycle.

Example:

```text id="a4g6y9"
Active
  ↓
Deprecated
  ↓
Migration Period
  ↓
Retired
```

Provide:

```text id="k7x3m2"
replacement endpoint
migration guide
deprecation timeline
compatibility notes
```

Do not remove externally consumed endpoints without an explicit migration strategy.

---

# 76. API Security Headers

The edge/API stack should use appropriate security headers and transport protections.

Depending on deployment:

```text id="3g9m1x"
Strict-Transport-Security
Content-Security-Policy
X-Content-Type-Options
Referrer-Policy
```

Exact headers should be configured according to the frontend and deployment architecture.

---

# 77. CORS

CORS must be explicitly configured.

Avoid:

```text id="r3f7m1"
Access-Control-Allow-Origin: *
```

for authenticated browser APIs unless the security model explicitly permits it.

Allowed origins should be configuration-driven and environment-specific.

---

# 78. CSRF

If browser authentication uses cookies, CSRF protections must be applied to state-changing requests.

Depending on the authentication model, use appropriate mechanisms such as:

```text id="j8k3p4"
SameSite cookies
CSRF tokens
origin validation
```

Token-based authentication does not automatically eliminate every browser security concern.

---

# 79. Content Types

APIs should explicitly define supported content types.

Typical JSON APIs:

```text id="6y3m7x"
Content-Type: application/json
```

File endpoints may use:

```text id="n4v7p2"
multipart/form-data
```

Unsupported content types should receive a consistent error.

---

# 80. Compression

Response compression may be enabled where beneficial.

Do not compress data unnecessarily when:

```text id="7p2m4x"
payloads are already compressed
CPU cost outweighs benefit
security concerns require special handling
```

Compression should be handled at the appropriate edge or server layer.

---

# 81. API Performance

API performance should be measured at:

```text id="j7n2k4"
p50
p95
p99
```

Track:

```text id="q9m4s6"
request latency
database latency
cache latency
search latency
external API latency
serialization time
queue time
```

Avoid optimizing based only on total request time.

Distributed traces should identify where latency is spent.

---

# 82. N+1 API Problems

API handlers must avoid causing repeated downstream queries.

Bad:

```text id="c8v2n7"
GET /orders
   │
   ├── query orders
   ├── query vendor
   ├── query vendor
   ├── query vendor
   └── ...
```

Prefer:

```text id="u6n3q9"
API
 │
 ▼
Application
 │
 ▼
Batch / optimized repository query
```

---

# 83. Response Payload Size

Do not return unnecessarily large responses.

Use:

```text id="h7v4m2"
pagination
field selection
projections
dedicated summary endpoints
compressed responses
```

Avoid embedding large related resources automatically.

---

# 84. API Transaction Boundaries

An API request should not automatically imply a single massive database transaction.

The application workflow determines the appropriate transaction.

For example:

```text id="e3w8m5"
Request
  │
  ▼
Application Workflow
  │
  ├── read
  ├── business computation
  │
  ▼
Transaction
  ├── write order
  ├── reserve inventory
  └── outbox
```

Long-running external calls should normally remain outside transactional database locks.

---

# 85. External API Calls

When an API operation calls an external provider:

```text id="x5p7n3"
Client
  │
  ▼
KAMPYN API
  │
  ▼
Application
  │
  ▼
Integration Adapter
  │
  ▼
External Provider
```

Use:

```text id="r8k2m5"
timeouts
bounded retries
idempotency
circuit breaking where appropriate
error mapping
observability
```

---

# 86. Partial Failure

Distributed workflows must define partial failure behavior.

Example:

```text id="n7m4q2"
Order Created
     │
     ▼
Notification Provider
     │
     └── fails
```

The system must determine whether notification failure:

```text id="z5p8r1"
fails the order
```

or:

```text id="v3k6m9"
is handled asynchronously
```

For most non-critical notifications, asynchronous processing is preferable.

---

# 87. API and Events

An API request may produce events.

Example:

```text id="m8q2v5"
POST /orders
      │
      ▼
CreateOrder
      │
      ├── database mutation
      └── outbox event
              │
              ▼
        OrderCreated
```

The event must represent committed state rather than an uncommitted assumption.

---

# 88. API and Audit

Security-sensitive API operations should create audit records where required.

Examples:

```text id="s4n7p2"
admin role change
tenant configuration change
permission modification
payment adjustment
user suspension
data export
```

Audit records should be created as part of the appropriate consistency boundary.

---

# 89. API and Observability

Every important request should provide:

```text id="q7m3v5"
request ID
trace ID
service
operation
route
status
duration
tenant context
```

Do not log sensitive request bodies by default.

---

# 90. API Health Endpoints

Deployable services should expose health endpoints appropriate to their role.

Examples:

```text id="x8k2m6"
/health/live
/health/ready
```

Health endpoints must be lightweight.

Do not make liveness checks depend on every external service.

---

# 91. Internal APIs

Internal service-to-service APIs should still use explicit contracts.

Do not assume that:

```text id="m5r8q2"
internal = trusted
```

Internal requests still require:

- authentication
- authorization
- validation
- timeouts
- observability
- safe error handling

where applicable.

---

# 92. Service-to-Service Authentication

Service communication may use:

```text id="y3p6k8"
mTLS
service identity
signed tokens
short-lived credentials
```

The mechanism should be standardized rather than implemented differently by every service.

---

# 93. API Gateway vs Application

The gateway should handle infrastructure-level concerns.

```text id="v7n2m4"
Gateway
 ├── routing
 ├── TLS
 ├── coarse rate limiting
 ├── request limits
 └── edge security

Application
 ├── authentication context
 ├── authorization
 ├── validation
 ├── business rules
 └── workflows
```

Do not duplicate business logic between gateway and backend.

---

# 94. API Module Organization

A possible Go backend structure:

```text id="j4m8q2"
internal/
├── modules/
│   ├── orders/
│   │   ├── transport/
│   │   │   ├── http.go
│   │   │   ├── request.go
│   │   │   └── response.go
│   │   │
│   │   ├── application/
│   │   ├── domain/
│   │   └── infrastructure/
│   │
│   ├── inventory/
│   ├── bookings/
│   └── users/
│
└── platform/
    ├── auth/
    ├── http/
    ├── validation/
    └── observability/
```

The exact layout may vary, but API transport code should remain separated from application logic.

---

# 95. Handler Responsibilities

A handler should generally:

```text id="u5r8n2"
1. Parse request
2. Validate request
3. Obtain trusted context
4. Call application use case
5. Map result to response
6. Map errors to HTTP
```

A handler should not:

```text id="s3m7k9"
query database directly
calculate business rules
publish arbitrary events
call multiple providers
manage transactions manually
```

unless that behavior is explicitly part of the architecture.

---

# 96. Application Service Responsibilities

Application services coordinate:

```text id="p6q4m8"
authorization context
repositories
transactions
domain operations
integrations
events
```

Example:

```text id="k9m3r5"
CreateOrder
 ├── validate command
 ├── authorize
 ├── load item
 ├── apply domain rules
 ├── persist
 ├── create outbox event
 └── return result
```

---

# 97. API Naming Consistency

Use consistent terminology across:

```text id="g8p3m6"
API
domain
database
events
SDK
documentation
frontend
```

For example, do not call the same concept:

```text id="s4k8m2"
vendor
seller
merchant
restaurant
```

in different APIs unless these represent genuinely different concepts.

---

# 98. API Documentation Examples

Examples should use realistic but non-sensitive values.

Good:

```json id="q2n6m8"
{
  "name": "Campus Kitchen",
  "active": true
}
```

Do not include:

```text id="h5r9p2"
real user credentials
real access tokens
production IDs
private data
payment secrets
```

---

# 99. API Testing

Every endpoint should have appropriate tests.

### Handler Tests

Test:

```text id="v7m3q5"
request parsing
validation
status codes
response shape
error mapping
```

### Application Tests

Test:

```text id="p8k4m2"
business workflows
authorization behavior
transaction boundaries
repository interactions
```

### Integration Tests

Test:

```text id="n6r3w8"
database behavior
authentication
real HTTP routing
external contracts where practical
```

### End-to-End Tests

Test critical user journeys.

---

# 100. API Security Testing

Test:

```text id="m3q7v9"
missing authentication
invalid authentication
cross-tenant access
privilege escalation
IDOR
invalid input
SQL/NoSQL injection
rate limits
request size limits
CSRF
CORS
webhook forgery
replay attacks
```

Security tests must include both positive and negative cases.

---

# 101. API Contract Testing

API contracts should be validated against:

```text id="p4m8x2"
frontend
SDK
external consumers
OpenAPI
```

Breaking contract changes should be detected in CI where practical.

---

# 102. API Performance Testing

Critical APIs should be benchmarked under realistic workloads.

Measure:

```text id="q7n3m5"
requests per second
p50
p95
p99
error rate
CPU
memory
database load
cache hit rate
external dependency latency
```

Load testing must represent realistic tenant and dataset distributions.

---

# 103. API Change Checklist

Before modifying an endpoint:

- [ ] Identify all consumers.
- [ ] Check API version.
- [ ] Check authentication requirements.
- [ ] Check authorization requirements.
- [ ] Check tenant scope.
- [ ] Check request schema.
- [ ] Check response schema.
- [ ] Check error contract.
- [ ] Check pagination behavior.
- [ ] Check idempotency.
- [ ] Check concurrency.
- [ ] Check database impact.
- [ ] Check event impact.
- [ ] Check SDK impact.
- [ ] Check documentation.
- [ ] Add/update tests.
- [ ] Review observability.
- [ ] Review security.
- [ ] Review performance.

---

# 104. Definition of Done

An API endpoint is production-ready when:

- [ ] Resource/action semantics are explicit.
- [ ] HTTP method is appropriate.
- [ ] Authentication is defined.
- [ ] Authorization is enforced.
- [ ] Tenant scope is explicit.
- [ ] Request validation exists.
- [ ] Response DTO is defined.
- [ ] Error contract is defined.
- [ ] Status codes are correct.
- [ ] Pagination is bounded where applicable.
- [ ] Sorting/filtering are allowlisted.
- [ ] Idempotency exists where required.
- [ ] Concurrency behavior is defined where required.
- [ ] Rate limits are appropriate.
- [ ] Observability exists.
- [ ] Security tests exist.
- [ ] Integration tests exist.
- [ ] API documentation is updated.
- [ ] OpenAPI is updated where applicable.
- [ ] SDK compatibility is considered.

---

# 105. Final Principle

KAMPYN's API should be treated as a **long-lived public contract**, not merely an HTTP wrapper around backend code.

The intended flow is:

```text id="h8m3q7"
                  CLIENT
                    │
                    ▼
             API CONTRACT
                    │
                    ▼
              HTTP LAYER
                    │
        ┌───────────┼───────────┐
        ▼           ▼           ▼
 Authentication  Tenant      Validation
                    │
                    ▼
               Authorization
                    │
                    ▼
             Application Use Case
                    │
                    ▼
                  Domain
                    │
          ┌─────────┴─────────┐
          ▼                   ▼
     Repository          Integration
          │                   │
          ▼                   ▼
      Database          External Provider
```

The fundamental rules are:

> **Design APIs around domain resources and use cases, not database tables.**

> **The API contract must remain independent of internal implementation details.**

> **Authentication, authorization, tenant isolation, validation, and rate limiting are server-side responsibilities.**

> **Clients must never be trusted to enforce security boundaries.**

> **All externally supplied input must be validated at the API boundary.**

> **Collection endpoints must be bounded and use appropriate pagination.**

> **Sorting, filtering, field selection, and expansion must be explicitly allowlisted.**

> **Side-effecting operations must be idempotent where retries are possible or expected.**

> **Long-running operations should be asynchronous rather than blocking HTTP requests.**

> **Errors must be structured, stable, machine-readable, and free of sensitive implementation details.**

> **API changes must consider frontend, SDK, integrations, webhooks, mobile clients, and self-hosted installations.**

> **Every critical API must be observable, testable, secure, and performance-measurable.**

> **The API is a contract between KAMPYN and its consumers; internal implementation changes must not unnecessarily become external breaking changes.**
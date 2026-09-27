# KAMPYN Backend Integrations

## Purpose

KAMPYN interacts with external systems for capabilities such as:

- Payments
- Email
- SMS
- Push notifications
- Authentication and identity
- University systems
- Object storage
- Search infrastructure
- Maps and location services
- Analytics providers
- Cloud infrastructure
- External APIs
- Webhooks
- Data imports and exports

External integrations are inherently unreliable and must therefore be isolated from KAMPYN's core application and domain logic.

The primary architectural rule is:

```text id="6f8x9v"
KAMPYN Core
     │
     ▼
Integration Contract
     │
     ▼
Provider Adapter
     │
     ▼
External Provider
```

The domain must not depend directly on provider SDKs or provider-specific behavior.

---

# 1. Core Principles

Integrations must be:

- Isolated
- Explicit
- Replaceable
- Observable
- Secure
- Idempotent
- Timeout-aware
- Retry-aware
- Failure-tolerant
- Testable
- Configuration-driven

An external provider is a dependency, not a source of architectural authority.

---

# 2. Integration Boundary

External services must remain behind an integration boundary.

Preferred:

```text id="5r3mxy"
Application
    ↓
Integration Interface
    ↓
Provider Adapter
    ↓
Provider SDK / HTTP API
```

Avoid:

```text id="4g4p5m"
Application
    ↓
Razorpay SDK
```

or:

```text id="7xk4p0"
Domain
    ↓
Twilio SDK
```

Provider-specific implementations belong in infrastructure/integration packages.

---

# 3. Provider Independence

KAMPYN should be able to replace a provider without rewriting business logic.

Example:

```text id="8q8p2y"
Payment Application
       │
       ▼
PaymentProvider
   ┌───┴────┐
   ▼        ▼
Provider A Provider B
```

The application should depend on:

```text id="m0qj8u"
CreatePayment
VerifyPayment
RefundPayment
```

rather than:

```text id="8y2qf4"
ProviderASpecificRequest
ProviderASpecificResponse
ProviderASpecificStatus
```

---

# 4. Integration Contracts

Define interfaces around the capability KAMPYN needs.

Example:

```go id="n0r8mz"
type PaymentProvider interface {
    CreatePayment(ctx context.Context, request CreatePaymentRequest) (PaymentResult, error)
    VerifyPayment(ctx context.Context, paymentID string) (PaymentResult, error)
    RefundPayment(ctx context.Context, request RefundRequest) (RefundResult, error)
}
```

The interface should represent KAMPYN's requirements, not reproduce an external provider's entire API.

---

# 5. Consumer-Driven Interfaces

Interfaces should generally be defined near the application code that consumes them.

Example:

```text id="7gq3zy"
application/payments/
    provider.go

infrastructure/integrations/payments/
    provider_a.go
    provider_b.go
```

The application defines the contract.

The infrastructure implements it.

---

# 6. Adapter Responsibilities

A provider adapter is responsible for translating between:

```text id="0b4k0f"
KAMPYN Contract
      ↕
Provider Contract
```

It may handle:

- Provider SDK calls
- HTTP requests
- Authentication
- Request mapping
- Response mapping
- Provider-specific errors
- Provider-specific status codes
- Provider-specific retry semantics
- Provider-specific webhook formats

It must not contain KAMPYN domain business rules.

---

# 7. DTO Separation

Do not reuse provider DTOs throughout the application.

Bad:

```text id="kqz8i3"
Application
 ↓
ProviderSDKRequest
```

Preferred:

```text id="o8pxz2"
Application Request
 ↓
Provider Adapter
 ↓
Provider Request
```

This prevents provider schema changes from spreading throughout the codebase.

---

# 8. Provider Response Mapping

Provider responses must be translated into stable KAMPYN concepts.

Example:

```text id="j7h4ac"
Provider:
"authorized"
"captured"
"failed"
"pending"

        ↓

KAMPYN:
PENDING
AUTHORIZED
COMPLETED
FAILED
```

Do not expose provider-specific status strings as core domain states unless they genuinely represent domain concepts.

---

# 9. Provider Errors

Provider errors must be translated.

Example:

```text id="3q7t5d"
Provider timeout
       ↓
IntegrationTimeoutError

Provider rate limit
       ↓
IntegrationRateLimitedError

Invalid provider request
       ↓
IntegrationRequestError
```

The application should not need to understand provider-specific error formats.

---

# 10. Error Classification

Integration errors should be classified as:

```text id="i9g9z0"
Transient
Permanent
Rate Limited
Authentication Failure
Invalid Request
Unavailable
Timeout
Cancelled
Unknown
```

Classification controls:

- Retry
- Logging
- Metrics
- User response
- Circuit breaking
- Job behavior

---

# 11. Timeouts

Every external call must have an explicit timeout.

Examples:

```text id="j2g4po"
Payment API
Email provider
SMS provider
University API
Storage API
Search service
Maps API
```

No external integration should be allowed to block indefinitely.

Timeouts should propagate through `context.Context`.

---

# 12. Retries

Retries must only be used for transient failures.

Possible retry candidates:

```text id="rjv1oz"
Network timeout
Temporary provider outage
HTTP 502
HTTP 503
HTTP 429
```

Do not blindly retry:

```text id="0k6g8s"
Invalid request
Invalid credentials
Malformed payload
Rejected payment
Unauthorized operation
```

---

# 13. Exponential Backoff

Retries should use bounded exponential backoff with jitter.

Conceptually:

```text id="m11j3v"
Attempt 1
   ↓
short delay
   ↓
Attempt 2
   ↓
longer delay
   ↓
Attempt 3
   ↓
maximum delay
```

Never retry indefinitely.

---

# 14. Idempotency

Any integration that creates or mutates external state must define idempotency behavior.

Examples:

```text id="q4g6t4"
Create payment
Create refund
Send notification
Create external booking
Create university record
Upload object
```

Use stable idempotency keys whenever the provider supports them.

---

# 15. Internal Idempotency

Even when an external provider does not support idempotency, KAMPYN should maintain its own protection where necessary.

Example:

```text id="8j3p2f"
KAMPYN Operation ID
        ↓
Persist operation state
        ↓
External provider
        ↓
Persist result
```

The operation must be recoverable after a timeout or worker crash.

---

# 16. Ambiguous External Results

External calls may fail after the provider successfully processes the operation.

Example:

```text id="1l3c7y"
KAMPYN
   ↓
Provider
   ↓
Payment succeeds
   X
Response lost
```

KAMPYN must not automatically assume failure.

Use:

- Provider transaction IDs
- Status verification
- Reconciliation
- Webhooks
- Idempotency keys

depending on the integration.

---

# 17. Payment Integrations

Payment integrations require particularly strict controls.

The architecture should resemble:

```text id="qk8n0p"
Order
  ↓
Payment Application
  ↓
Payment Provider Interface
  ↓
Provider Adapter
  ↓
Payment Provider
```

Payment state must remain owned by KAMPYN's payment domain.

---

# 18. Payment State

Do not directly mirror provider states.

Example:

```text id="s4x4ef"
KAMPYN Payment

PENDING
   ↓
AUTHORIZED
   ↓
COMPLETED

or

PENDING
   ↓
FAILED

or

AUTHORIZED
   ↓
REFUNDED
```

Valid transitions must be enforced by the payment domain.

---

# 19. Payment Reconciliation

Payment state must be reconcilable against the provider.

Possible workflow:

```text id="3j2h4a"
KAMPYN Payment
      ↓
Provider Status
      ↓
Compare
      ↓
Reconcile
      ↓
Update KAMPYN State
```

Reconciliation jobs should be idempotent.

---

# 20. Payment Webhooks

Payment providers may send:

- Payment created
- Payment authorized
- Payment captured
- Payment failed
- Refund completed

Webhook processing should follow:

```text id="3x3e6d"
Webhook
   ↓
Verify Signature
   ↓
Validate Payload
   ↓
Persist Event
   ↓
Acknowledge
   ↓
Queue Processing
   ↓
Application Use Case
```

Do not perform long-running business processing directly inside the webhook request.

---

# 21. Webhook Signature Verification

Never trust an external webhook merely because it reached the endpoint.

Verify:

- Signature
- Timestamp where supported
- Provider identifier
- Event structure
- Expected credentials

Reject invalid webhooks before processing.

---

# 22. Webhook Idempotency

Webhook providers may deliver the same event multiple times.

Persist a stable provider event ID where available.

Example:

```text id="t3b8kv"
provider_event_id
+
provider
```

should be unique where appropriate.

Duplicate webhook delivery should become a safe no-op.

---

# 23. Webhook Ordering

Webhook events may arrive out of order.

Example:

```text id="c1d1o9"
COMPLETED
arrives before
AUTHORIZED
```

Do not blindly apply events without validating current state.

Use:

- Event versions
- Provider sequence numbers
- Current provider status
- State transition validation
- Reconciliation

where appropriate.

---

# 24. Email Integrations

Email should be accessed through an internal notification abstraction.

```text id="7d4r1a"
Application
   ↓
NotificationService
   ↓
EmailProvider
   ↓
Provider Adapter
```

The application should not construct provider-specific API requests.

---

# 25. Email Delivery

Email jobs should generally be asynchronous.

```text id="y8w3s0"
Application Event
   ↓
Notification Job
   ↓
Email Provider
```

Store enough information to:

- Retry
- Track status
- Diagnose failures
- Prevent duplicate delivery where required

---

# 26. SMS Integrations

SMS follows the same provider abstraction.

```text id="r7f4w2"
NotificationService
       ↓
SMSProvider
       ↓
Provider Adapter
```

Do not expose provider-specific message IDs as domain identifiers.

---

# 27. Push Notifications

Push providers should also remain behind a notification boundary.

The application should describe:

```text id="2x3x6p"
Recipient
Title
Body
Data
Priority
```

The adapter translates this into the provider-specific request.

---

# 28. Authentication Integrations

External identity providers such as OAuth/OIDC providers must remain behind the authentication boundary.

```text id="2v7v8e"
Identity Provider
       ↓
Authentication Adapter
       ↓
KAMPYN Identity
```

Do not make the core domain depend on:

```text id="7s7j5c"
GoogleUser
MicrosoftUser
ProviderSpecificClaims
```

Instead map them into KAMPYN identity concepts.

---

# 29. Identity Linking

External identities should be linked using stable provider identifiers.

Conceptually:

```text id="5g5f2w"
Provider
Provider Subject
      ↓
External Identity
      ↓
KAMPYN User
```

Do not use email alone as the permanent identity key when the provider supplies a stable subject identifier.

---

# 30. University Integrations

Universities may expose systems for:

- Student records
- Faculty records
- Hostel information
- Attendance
- Fees
- Identity
- Academic information
- Transport
- Library
- Campus services

These integrations must remain isolated behind adapters.

```text id="2p8x5b"
KAMPYN
   ↓
University Integration Contract
   ↓
University Adapter
   ↓
University System
```

---

# 31. University Data Ownership

Before synchronizing university data, define which system owns each field.

Example:

```text id="7p8z8r"
Student Name
    ↓
University System

KAMPYN Notification Preference
    ↓
KAMPYN

Hostel Assignment
    ↓
University System or KAMPYN
```

Never allow synchronization to create ambiguous ownership.

---

# 32. Synchronization Models

University integrations may use:

```text id="z5k7ad"
Pull
Push
Webhook
Scheduled Sync
Event Streaming
Manual Import
```

The synchronization strategy must be explicit.

---

# 33. Pull Synchronization

For polling:

```text id="9q6z0p"
Scheduler
   ↓
Integration Job
   ↓
University API
   ↓
Normalize
   ↓
Compare
   ↓
Persist changes
```

Use:

- Incremental sync where possible
- Pagination
- Checkpoints
- Rate limits
- Retry policies

Do not repeatedly download the entire dataset when incremental synchronization is available.

---

# 34. Push Synchronization

When KAMPYN sends data to a university system:

```text id="9j7m4z"
KAMPYN Event
   ↓
Integration Job
   ↓
University Adapter
   ↓
External System
```

Use idempotency and durable operation tracking.

---

# 35. Conflict Resolution

If both systems can modify a field, define conflict resolution.

Possible approaches:

```text id="c3i9v7"
University authoritative
KAMPYN authoritative
Latest valid version
Explicit workflow
Manual resolution
```

Do not silently use "last write wins" for every integration.

---

# 36. Integration State

Long-running integrations may require persistent state.

Example:

```text id="w4d8rx"
Integration
 ├── tenantId
 ├── provider
 ├── status
 ├── lastSuccessfulSync
 ├── lastAttempt
 ├── cursor
 └── configurationReference
```

Do not store important synchronization checkpoints only in process memory.

---

# 37. Integration Credentials

Credentials must be stored securely.

Never store provider secrets:

- In source code
- In Git
- In logs
- In job payloads
- In URLs
- In client-side code

Use the deployment secret-management mechanism.

---

# 38. Credential Rotation

Integrations must support credential rotation.

A rotation should not require changing business logic.

Provider configuration should be externalized.

---

# 39. Multi-Tenant Integrations

Each tenant may have different:

- Provider credentials
- API endpoints
- University systems
- Payment configuration
- Branding
- Notification providers
- Identity providers

Therefore integration configuration must support tenant scoping where required.

```text id="j8v4q6"
Tenant A
   ↓
Provider A credentials

Tenant B
   ↓
Provider B credentials
```

Never allow one tenant to use another tenant's integration credentials.

---

# 40. Integration Configuration

Configuration should distinguish:

```text id="x8g4u6"
Platform configuration
Tenant configuration
Provider configuration
Runtime configuration
```

Do not mix them into one unstructured configuration object.

---

# 41. Object Storage

Object storage integrations may be used for:

- User uploads
- Food images
- Documents
- Reports
- Exports
- Generated files
- Temporary processing artifacts

Application code should depend on an object-storage abstraction where provider portability matters.

---

# 42. Object Storage Security

Object storage operations must enforce:

- Tenant isolation
- Authorization
- File size limits
- Content type validation
- Secure object keys
- Access expiration
- Malware/security scanning where required

Do not expose unrestricted storage buckets.

---

# 43. Object References

Persist stable internal references rather than relying on provider-specific URLs.

Example:

```text id="r7d5kg"
Document
 ├── id
 ├── storageProvider
 ├── objectKey
 └── metadata
```

Generate access URLs when required.

---

# 44. Storage Provider Replacement

Avoid storing provider-specific URLs as the only representation of an object.

Prefer:

```text id="n4w9pz"
KAMPYN Object Reference
        ↓
Storage Adapter
        ↓
Provider
```

This allows provider migration.

---

# 45. Search Integrations

OpenSearch is an infrastructure dependency.

Search operations should remain behind the search abstraction.

```text id="h2t7jq"
Application
   ↓
Search Service
   ↓
Search Repository
   ↓
OpenSearch
```

Search failures must not corrupt the transactional source of truth.

---

# 46. Search Synchronization

Search indexes should generally be updated asynchronously.

```text id="1k2g8f"
Database Transaction
      ↓
Outbox Event
      ↓
Search Job
      ↓
OpenSearch
```

See:

```text id="h5b6v8"
architecture/search.md
architecture/events.md
```

for detailed search and event behavior.

---

# 47. Integration Rate Limits

External providers may limit:

- Requests per second
- Requests per minute
- Concurrent requests
- Daily requests
- Data transfer

KAMPYN must respect provider limits.

Use:

- Rate limiting
- Worker concurrency limits
- Backoff
- Queueing
- Provider-specific throttling

---

# 48. Circuit Breaking

If an external dependency repeatedly fails:

```text id="w6v6j9"
KAMPYN
   ↓
Provider
   X
Repeated failure
```

continued traffic may make the failure worse.

Circuit-breaking can provide:

```text id="o7j7f8"
Closed
  ↓
Failures
  ↓
Open
  ↓
Cooldown
  ↓
Half-open
  ↓
Recovery
```

Use only where the operational behavior justifies the complexity.

---

# 49. Bulk Operations

When providers support bulk APIs, use them where they improve:

- Throughput
- Rate-limit efficiency
- Cost
- Latency

However, bulk operations must define partial failure semantics.

Example:

```text id="1k0o6n"
100 records
 ├── 97 success
 └── 3 failed
```

The three failures must remain observable and recoverable.

---

# 50. Integration Jobs

Long-running integrations should generally execute through background jobs.

Examples:

```text id="z2j1ob"
university.sync
payment.reconcile
notification.send
report.export
storage.process
```

The job must call the application/integration layer rather than duplicating business logic.

---

# 51. Integration Queues

Heavy or failure-prone integrations may require dedicated queues.

Example:

```text id="3l4x8j"
Critical Queue
 ├── Payments

Standard Queue
 ├── Notifications

Heavy Queue
 ├── University Sync
 └── File Processing
```

This prevents one provider outage from consuming all worker capacity.

---

# 52. Partial Failures

Integration workflows must explicitly handle partial failure.

Example:

```text id="j9x4a1"
KAMPYN
  ↓
Provider A ✓
Provider B ✗
Provider C ✓
```

The system must persist enough state to know what succeeded.

Do not blindly restart the entire workflow if that creates duplicate external side effects.

---

# 53. Distributed Workflows

When a workflow spans multiple systems:

```text id="q1j3zz"
KAMPYN DB
   ↓
Payment Provider
   ↓
University System
   ↓
Notification Provider
```

do not attempt one giant transaction.

Use:

- Local transactions
- Durable state
- Events
- Background jobs
- Compensation
- Reconciliation

where appropriate.

---

# 54. Saga-Like Workflows

For multi-step operations:

```text id="d9f2k7"
Step A
 ↓
Step B
 ↓
Step C
```

define what happens if:

```text id="8x5g9n"
A ✓
B ✓
C ✗
```

Possible outcomes:

```text id="9k4k5a"
Retry C
Compensate B
Mark workflow failed
Manual intervention
```

The correct behavior depends on the domain.

---

# 55. Integration Status

Long-running integration operations should have explicit status.

Example:

```text id="m2z7w9"
PENDING
RUNNING
SUCCEEDED
PARTIALLY_SUCCEEDED
FAILED
CANCELLED
```

Do not infer state solely from logs.

---

# 56. Integration Observability

Every external integration should be observable.

Track:

```text id="3h8c7f"
Provider
Operation
Tenant
Duration
Success
Failure
Retry count
Timeout count
Rate limits
Request size where relevant
Response classification
```

Never log secrets or sensitive payloads.

---

# 57. Integration Metrics

Useful metrics include:

```text id="e1t8m2"
integration_requests_total
integration_failures_total
integration_timeouts_total
integration_retries_total
integration_duration_seconds
integration_rate_limited_total
integration_circuit_open_total
```

Avoid high-cardinality labels such as:

```text id="b8z1m2"
user_id
order_id
payment_id
request_id
```

---

# 58. Tracing

Distributed traces should connect:

```text id="x3z8g0"
API Request
   ↓
Application
   ↓
Integration
   ↓
External Provider
```

Use:

- Trace ID
- Span ID
- Correlation ID

where supported.

Do not expose internal trace information to untrusted external systems unless intentionally designed.

---

# 59. Provider Health

Integrations may expose provider health information internally.

Examples:

```text id="3v7x6r"
Available
Degraded
Unavailable
Rate Limited
Misconfigured
```

Health checks should not cause unnecessary provider traffic.

---

# 60. Fallbacks

Fallback behavior must be explicitly defined.

Example:

```text id="n1j7b4"
Primary provider fails
       ↓
Fallback provider
```

Do not automatically fail over every integration.

Some operations are not safe to repeat against another provider.

For financial operations especially, provider switching requires explicit reconciliation and idempotency rules.

---

# 61. Provider Selection

If multiple providers exist, provider selection should happen through configuration or a dedicated routing policy.

Example:

```text id="f4y2z8"
Tenant
 ↓
Integration Configuration
 ↓
Provider Selection
 ↓
Adapter
```

Do not scatter provider selection throughout business logic.

---

# 62. Provider-Specific Features

A provider may support capabilities that another provider does not.

Do not pollute the common interface with every provider-specific feature.

Instead:

```text id="5t3z0r"
Common capability
+
Optional provider capability
```

or expose the provider-specific functionality through an explicitly separate integration boundary.

---

# 63. Provider Migration

Provider migration should support:

```text id="5h2v8m"
Old Provider
     │
     ▼
Migration Strategy
     │
     ▼
New Provider
```

Consider:

- Existing records
- Existing transaction IDs
- Webhooks
- Historical data
- Credentials
- In-flight operations
- Retry queues
- Reconciliation
- Rollback

Do not switch providers by changing one environment variable and assuming all state is compatible.

---

# 64. Versioning

Provider APIs change.

Adapters should isolate provider API versions.

Example:

```text id="4j7x9f"
Provider API v1 Adapter
Provider API v2 Adapter
        ↓
Same KAMPYN Contract
```

This allows gradual migration.

---

# 65. Deprecation

When an integration is deprecated:

1. Mark the provider as deprecated.
2. Stop creating new operations where appropriate.
3. Migrate active workflows.
4. Reconcile existing state.
5. Remove old credentials.
6. Remove obsolete adapter code.
7. Update documentation.
8. Remove dead dependencies.

Do not leave abandoned provider integrations indefinitely.

---

# 66. SDK Usage

Third-party SDKs may be used inside adapters.

They must not leak into:

- Domain entities
- Application interfaces
- Core business logic
- Public API contracts

The adapter owns the SDK.

---

# 67. HTTP Integration Clients

Integration HTTP clients should provide:

- Base URL configuration
- Authentication
- Timeout
- Connection reuse
- Request headers
- Retry policy
- Response validation
- Error mapping
- Metrics
- Tracing

Do not duplicate these behaviors across every integration.

---

# 68. TLS

External integrations must use secure transport.

Do not disable TLS verification in production.

Any custom certificate authority or mutual TLS configuration must be explicit and securely managed.

---

# 69. Request Signing

Where providers require signed requests:

```text id="6s2h7k"
Payload
   ↓
Canonicalization
   ↓
Signature
   ↓
Request
```

Signing implementation belongs inside the provider adapter.

Never expose signing secrets to application code.

---

# 70. Webhook Security

Webhook endpoints must:

- Verify authenticity
- Validate content type
- Limit request size
- Validate payload schema
- Protect against replay where required
- Deduplicate events
- Log safe identifiers
- Return appropriate status codes

Do not trust webhook payloads simply because they originate from a known URL.

---

# 71. Replay Protection

Where a provider supports timestamps, signatures, or nonces, use them to prevent replay attacks.

Persist processed event IDs when appropriate.

---

# 72. Import Integrations

Data imports should follow:

```text id="9b4c2n"
Upload
 ↓
Validate
 ↓
Parse
 ↓
Normalize
 ↓
Validate Domain Rules
 ↓
Process
 ↓
Persist
 ↓
Report Results
```

Large imports must be asynchronous and streamed.

---

# 73. Export Integrations

Exports should:

- Respect tenant scope
- Respect authorization
- Use bounded processing
- Stream large datasets
- Store output securely
- Expire temporary files
- Record export status

Do not load entire datasets into memory.

---

# 74. Data Normalization

External systems often use different:

- Field names
- IDs
- Date formats
- Enumerations
- Units
- Statuses

Normalize them at the integration boundary.

Example:

```text id="2n4v8z"
University:
"STU_ACTIVE"

       ↓

KAMPYN:
ACTIVE
```

Do not spread normalization logic across the application.

---

# 75. External IDs

External identifiers must remain distinguishable from KAMPYN IDs.

Example:

```text id="4v7b3s"
KAMPYN User ID
+
Provider Subject ID
```

Do not assume an external ID can safely replace an internal primary key.

---

# 76. Data Mapping

Maintain explicit mapping between:

```text id="z0z7v8"
External Model
      ↓
Integration DTO
      ↓
KAMPYN Model
```

Mapping should be deterministic and tested.

---

# 77. Data Validation

Never assume external data is valid.

Validate:

- Required fields
- Types
- Ranges
- Enumerations
- IDs
- Relationships
- Timestamps
- Tenant scope

External systems are trust boundaries.

---

# 78. Integration Security Boundary

Treat every external system as an untrusted dependency.

Even if the provider is trusted operationally, external responses may be:

- Malformed
- Unexpected
- Incomplete
- Delayed
- Duplicated
- Compromised

Validate everything entering KAMPYN.

---

# 79. Integration Testing

Every adapter should have tests for:

### Success

```text id="4p2e0x"
Valid request
Valid response
```

### Failure

```text id="4u4o7d"
Timeout
Provider error
Invalid response
Rate limit
Authentication failure
```

### Retry

```text id="d7h4x8"
Transient failure
 ↓
Retry
 ↓
Success
```

### Idempotency

```text id="f9k2j0"
Same operation twice
 ↓
Safe result
```

### Malformed input

```text id="1z7q8b"
Invalid provider payload
 ↓
Rejected safely
```

---

# 80. Contract Testing

Where provider APIs are critical, maintain contract tests for:

- Request schema
- Response schema
- Authentication
- Error mapping
- Webhooks
- Status mappings

Provider behavior should not be assumed merely because an SDK compiles.

---

# 81. Integration Mocks

Mock the provider boundary, not internal implementation details.

Good:

```text id="0q7f2s"
PaymentProvider
```

Bad:

```text id="2k9m7n"
Mock every helper inside PaymentService
```

Tests should validate KAMPYN behavior.

---

# 82. Sandbox Environments

Where providers offer sandbox environments, use them for integration testing.

Never use production credentials in automated tests.

Production integration testing must be carefully controlled.

---

# 83. Failure Injection

Critical integrations should be tested under:

```text id="0h5q3k"
Timeout
Connection failure
Malformed response
HTTP 500
HTTP 429
Slow response
Duplicate webhook
Out-of-order webhook
Credential failure
Partial success
```

This validates failure handling before production incidents expose it.

---

# 84. Integration Documentation

Each significant integration should document:

```text id="y3x5k4"
Provider
Purpose
Owner
Configuration
Credentials
Endpoints
Timeouts
Retries
Rate limits
Webhooks
Idempotency
Failure behavior
Data ownership
Tenant behavior
Monitoring
Testing
Migration strategy
```

Documentation must remain synchronized with implementation.

---

# 85. Integration Ownership

Every integration should have a clear owner.

Example:

```text id="x2n8g5"
payments/
notifications/
identity/
university/
storage/
search/
```

Ownership should include:

- Code
- Configuration
- Credentials
- Monitoring
- Incident response
- Documentation
- Provider migration

---

# 86. Integration Change Process

Before modifying an integration:

1. Identify the provider boundary.
2. Identify data ownership.
3. Identify current consumers.
4. Identify retry behavior.
5. Identify idempotency requirements.
6. Identify webhook behavior.
7. Identify tenant configuration.
8. Identify operational dependencies.
9. Update adapter.
10. Update tests.
11. Update documentation.
12. Verify failure scenarios.

---

# 87. Definition of Done

An integration is complete only when:

- Provider boundary is explicit.
- Application code does not depend on provider SDKs.
- Provider DTOs are isolated.
- Errors are translated.
- Timeouts exist.
- Retry behavior is defined.
- Idempotency is defined.
- Authentication is secure.
- Credentials are externalized.
- Webhooks are verified where applicable.
- Duplicate events are safe.
- Tenant isolation is enforced.
- External data is validated.
- Provider limits are respected.
- Partial failures are handled.
- Observability exists.
- Tests cover success and failure.
- Documentation exists.
- Migration/replacement behavior is understood.

---

# 88. Final Invariants

The following rules are mandatory:

```text id="j8r1q2"
1. External providers must never become part of KAMPYN's core domain model.

2. Provider SDKs must remain inside integration adapters.

3. Application code must depend on KAMPYN-defined capabilities,
   not provider-specific APIs.

4. External data is always treated as untrusted input.

5. Every external call has a bounded timeout.

6. Retry behavior is explicit and bounded.

7. Side-effecting integrations must define idempotency behavior.

8. Ambiguous external results must be recoverable through
   status checks, webhooks, reconciliation, or equivalent mechanisms.

9. Webhooks must be authenticated and idempotent.

10. Provider-specific statuses must be mapped into stable KAMPYN concepts.

11. Provider credentials must never be stored in source code,
    logs, client applications, or job payloads.

12. Tenant-specific integrations must remain isolated by tenant.

13. External IDs must not silently replace KAMPYN identifiers.

14. Integration state that affects correctness must be persisted.

15. Long-running integrations belong in background jobs.

16. Integration failures must not corrupt KAMPYN's authoritative state.

17. Search and cache integrations remain derived infrastructure.

18. Large imports and exports must use bounded memory and streaming.

19. Provider limits must be respected through explicit concurrency
    and rate controls.

20. Distributed workflows must use local transactions and
    durable coordination rather than giant distributed transactions.

21. Provider replacement must not require rewriting core business logic.

22. Provider migrations must account for in-flight operations,
    historical state, webhooks, retries, and reconciliation.

23. Every critical integration must have failure-path tests.

24. Integration observability must make provider failures,
    latency, retries, and rate limits diagnosable.

25. If removing a provider requires modifying core domain rules,
    the integration boundary is probably in the wrong place.
```

## Integration Architecture

```text id="b6w1f4"
                       KAMPYN
                          │
                          ▼
                  Application Layer
                          │
          ┌───────────────┼────────────────┐
          │               │                │
          ▼               ▼                ▼
      Payments       Notifications     University
      Contract          Contract        Contract
          │               │                │
          ▼               ▼                ▼
      Adapter A        Adapter A        Adapter A
      Adapter B        Adapter B        Adapter B
          │               │                │
          ▼               ▼                ▼
      Provider         Provider         External
      APIs             APIs             Systems
```

The integration boundary exists to ensure that **external systems can fail, change, or be replaced without compromising KAMPYN's domain model, tenant isolation, transactional correctness, or application architecture**.
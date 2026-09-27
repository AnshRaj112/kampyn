# KAMPYN Integration Architecture

## 1. Purpose

This document defines how KAMPYN integrates with external systems and third-party services.

Integrations may include:

- Payment providers.
- Email providers.
- SMS providers.
- Push notification providers.
- Authentication and identity providers.
- University systems.
- Food vendor systems.
- Delivery or logistics systems.
- Cloud services.
- Object storage.
- Search infrastructure.
- Analytics platforms.
- External APIs.
- Webhooks.
- Institutional infrastructure.

External systems are dependencies.

KAMPYN must remain correct and maintainable even when non-critical external dependencies are unavailable.

---

# 2. Integration Principles

KAMPYN integrations follow these principles:

1. External systems must have explicit ownership.
2. External APIs must not leak throughout the application.
3. Integrations must be isolated behind adapters.
4. External failures must be expected.
5. Timeouts are mandatory.
6. Retries must be bounded and safe.
7. External operations should be idempotent where possible.
8. Credentials must be centrally managed.
9. Webhooks must be verified before processing.
10. External data must be validated.
11. Integration contracts must be documented.
12. External providers must not become accidental sources of truth.
13. Provider-specific behavior must remain isolated.
14. Optional integrations should fail independently from unrelated application functionality.

---

# 3. Integration Architecture

The general integration architecture is:

```text id="q7m3n8"
                    KAMPYN
                       │
                Application Layer
                       │
              Integration Interface
                       │
              Integration Adapter
                       │
              ┌────────┴─────────┐
              ↓                  ↓
       External Provider     External API
```

The application should depend on an internal interface.

The provider-specific implementation belongs in infrastructure.

---

# 4. Adapter Pattern

Prefer:

```text id="m4n8q2"
Application
     ↓
PaymentProvider
     ↓
RazorpayAdapter
```

over:

```text id="x7p3m8"
OrderService
     ↓
Razorpay SDK
```

This keeps provider-specific details outside business logic.

The same pattern applies to:

```text id="c8q2m5"
EmailProvider
SMSProvider
StorageProvider
IdentityProvider
SearchProvider
UniversityProvider
```

---

# 5. Integration Boundaries

Each integration should define:

- Interface.
- Adapter.
- Configuration.
- Authentication.
- Request/response mapping.
- Error mapping.
- Timeout.
- Retry behavior.
- Idempotency.
- Observability.
- Testing strategy.

A provider SDK should not become the application's domain model.

---

# 6. External Data Is Untrusted

Data received from external systems must be treated as untrusted input.

Validate:

- Structure.
- Types.
- Required fields.
- Identifiers.
- Amounts.
- Status values.
- Timestamps.
- Signatures where applicable.

Use runtime validation at integration boundaries.

Zod may be used for TypeScript integration boundaries.

Go services should use explicit validation and strongly typed integration DTOs.

---

# 7. Provider DTOs

Do not pass provider-specific response objects throughout the application.

Prefer:

```text id="n5m8q2"
External Provider Response
        ↓
Provider DTO
        ↓
Adapter Mapping
        ↓
KAMPYN Domain Model
```

This prevents external SDK types from becoming internal architectural dependencies.

---

# 8. Integration Ownership

Every integration must have an owner.

For example:

```text id="v2q8m4"
Payments
  → Payments Domain

Email
  → Notification Domain

Search
  → Search Infrastructure

Authentication
  → Identity / Authentication Domain

University ERP
  → Institutional Integration Module
```

The owning module controls the integration contract.

---

# 9. Configuration

Integration configuration must be externalized.

Typical configuration includes:

```text id="c7m3n8"
Provider
Base URL
API key
Secret
Webhook secret
Timeout
Retry configuration
Environment
Feature flags
```

Never hardcode provider credentials.

---

# 10. Secrets

Integration secrets must be stored using the platform's secret-management system.

Examples:

- API keys.
- Client secrets.
- Webhook secrets.
- Private signing keys.
- OAuth credentials.

Never commit secrets to:

- Source code.
- Git.
- Docker images.
- Frontend bundles.
- Logs.

---

# 11. Credential Rotation

Integration credentials must support rotation.

A rotation should ideally allow:

```text id="m8q4n2"
Old Credential
      +
New Credential
      ↓
Transition
      ↓
Old Credential Revoked
```

Do not require a source-code deployment merely to rotate a secret.

---

# 12. Environment Separation

External provider configuration must be separated by environment.

For example:

```text id="r3n7q8"
Development
   ↓
Provider Sandbox

Staging
   ↓
Provider Sandbox / Test Account

Production
   ↓
Production Provider
```

Never accidentally use production payment or messaging credentials in development.

---

# 13. Payment Integrations

Payment integrations are security- and correctness-sensitive.

The payment architecture should separate:

```text id="p8m2q4"
Order State
     ↓
Payment Intent
     ↓
External Payment Provider
     ↓
Payment Confirmation
     ↓
Authoritative Payment State
```

The provider response must not automatically become the application's final business state without validation.

---

# 14. Payment Idempotency

Payment operations must be protected against duplicate requests.

Use provider-supported idempotency mechanisms where available.

Conceptually:

```text id="x4n8m2"
Payment Request
    idempotency_key = X
          ↓
Provider
          ↓
Same logical payment
```

Retries must not accidentally create multiple charges.

---

# 15. Payment Webhooks

Payment webhooks must be treated as external events.

Typical flow:

```text id="q7m3n8"
Payment Provider
      ↓
Webhook
      ↓
Signature Verification
      ↓
Schema Validation
      ↓
Idempotency Check
      ↓
Persist Payment State
      ↓
Internal Event
```

Never trust a webhook solely because it came from an expected URL.

---

# 16. Payment Reconciliation

Payment systems should support reconciliation.

Potential discrepancies include:

```text id="m5q8n2"
KAMPYN says PENDING
Provider says SUCCESS
```

or:

```text id="v3n7p4"
KAMPYN says SUCCESS
Provider says FAILED
```

A reconciliation process should identify and resolve discrepancies according to the payment architecture.

---

# 17. Payment Failure Isolation

Payment provider failures should not corrupt unrelated order state.

Examples:

```text id="c8m2q7"
Provider Timeout
      ↓
Payment remains pending/retryable
```

not:

```text id="n4p7m2"
Provider Timeout
      ↓
Order incorrectly marked completed
```

Payment state transitions must be explicit.

---

# 18. Email Integration

Email should be accessed through a notification abstraction.

Prefer:

```text id="x6m3q8"
Notification Service
       ↓
EmailProvider
       ↓
Mail Provider
```

rather than application features directly invoking provider SDKs.

The provider can be changed without changing domain logic.

---

# 19. Email Delivery

Email delivery should generally be asynchronous.

Example:

```text id="q4n8m2"
OrderConfirmed
      ↓
Notification Event
      ↓
Email Worker
      ↓
Email Provider
```

Do not make an order transaction depend on successful email delivery unless explicitly required.

---

# 20. SMS Integration

SMS should follow the same abstraction:

```text id="p7m2n8"
Notification Service
      ↓
SMSProvider
      ↓
SMS Adapter
      ↓
External Provider
```

SMS failures should be isolated from unrelated transactional operations.

---

# 21. Push Notifications

Push notifications should be asynchronous where practical.

The system should track:

- Device token.
- User association.
- Platform.
- Token validity.
- Last-seen information where useful.

Invalid device tokens should be cleaned up safely.

---

# 22. Notification Preferences

Notification delivery should respect user and institutional preferences.

Potential controls include:

```text id="m8q3n2"
Email enabled
SMS enabled
Push enabled
Marketing notifications
Transactional notifications
```

Transactional notifications may have different rules from optional notifications.

---

# 23. Authentication Integrations

KAMPYN may integrate with:

- OAuth providers.
- OpenID Connect providers.
- University SSO.
- Google Workspace.
- Microsoft identity systems.
- Institutional identity providers.

The authentication architecture must isolate provider-specific identity handling.

Typical flow:

```text id="c5n8q2"
Identity Provider
      ↓
Authentication Adapter
      ↓
Validate Identity
      ↓
Map External Identity
      ↓
KAMPYN User
```

---

# 24. External Identity Mapping

Never use an external provider's email address as the sole internal identity.

Maintain an explicit relationship:

```text id="v7m3q8"
KAMPYN User
      │
      └── External Identity
            provider
            subject_id
```

The provider subject identifier should be treated as the stable external identity where supported.

---

# 25. University Integrations

University deployments may integrate with institutional systems such as:

- Student information systems.
- Staff directories.
- Hostel systems.
- Library systems.
- Identity systems.
- Payment systems.
- ERP systems.
- Campus access systems.
- Existing food/vendor systems.

These integrations must be institution-specific adapters.

Avoid embedding university-specific assumptions into the core domain.

---

# 26. Institutional Integration Layer

A useful structure is:

```text id="q2m8n4"
KAMPYN Core
      │
      ↓
Institution Integration Interface
      │
 ┌────┼──────────┐
 ↓    ↓          ↓
KIIT Adapter  University B  University C
```

The core platform remains institution-agnostic.

---

# 27. University Data Synchronization

External university data may be synchronized periodically or through events.

Examples:

```text id="n5q8m2"
Student Created
Student Updated
Student Deactivated
Staff Updated
Room Assigned
```

Synchronization must define:

- Source of truth.
- Sync direction.
- Frequency.
- Conflict handling.
- Missing records.
- Deletion behavior.
- Retry behavior.

---

# 28. External System as Source of Truth

Some institutional data may remain authoritative in the university's system.

Example:

```text id="m7n3q8"
University SIS
      ↓
Student Identity
      ↓
KAMPYN Projection
```

In such cases KAMPYN must not silently become authoritative for that data.

The integration contract must explicitly define ownership.

---

# 29. Data Synchronization Models

Possible synchronization models include:

```text id="c4m8q2"
Pull
Push
Webhook
Polling
Scheduled Sync
Event Streaming
Bidirectional Sync
```

Prefer the simplest model that satisfies freshness requirements.

Bidirectional synchronization should be introduced only when both systems genuinely need to modify the same business entity.

---

# 30. Conflict Resolution

Bidirectional integrations require explicit conflict resolution.

Possible strategies include:

- Source-of-truth precedence.
- Version numbers.
- Timestamps where semantically safe.
- Manual reconciliation.
- Domain-specific conflict rules.

Do not silently overwrite external data when conflicts occur.

---

# 31. External API Clients

External API clients should provide:

- Timeout.
- Authentication.
- Serialization.
- Validation.
- Retry policy.
- Error mapping.
- Metrics.
- Tracing.

They should not expose raw provider errors to business logic.

---

# 32. Timeouts

Every external network call must have a timeout.

Timeouts should account for:

- Expected provider latency.
- API request deadline.
- User-facing latency requirements.
- Retry budget.

Never allow an external provider to block a KAMPYN request indefinitely.

---

# 33. Retries

Retries are appropriate only for transient failures.

Potential retryable failures:

```text id="p8n3m2"
Timeout
Connection reset
Temporary service unavailable
Rate limiting with retry guidance
```

Potentially non-retryable failures:

```text id="x5q7m8"
Invalid credentials
Invalid request
Validation failure
Authorization failure
Permanent business rejection
```

Retry policies must be bounded.

---

# 34. Exponential Backoff

Retries should generally use exponential backoff.

Conceptually:

```text id="m2n8q4"
Attempt 1 → short delay
Attempt 2 → longer delay
Attempt 3 → longer delay
```

Add jitter where many workers could otherwise retry simultaneously.

---

# 35. Rate Limits

External providers may impose rate limits.

Integration clients must respect:

- Requests per second.
- Requests per minute.
- Concurrent request limits.
- Daily quotas.

Where appropriate, use:

- Queueing.
- Backpressure.
- Batching.
- Caching.
- Request coalescing.

Never solve provider rate limits by creating unlimited concurrent workers.

---

# 36. Circuit Breakers

Circuit breakers may protect KAMPYN from unstable external services.

Example:

```text id="q4m8n2"
External Provider
      ↓
Repeated failures
      ↓
Circuit Open
      ↓
Fail Fast
      ↓
Recovery attempt
```

Use them where dependency instability can otherwise cascade into KAMPYN.

---

# 37. External Dependency Isolation

Optional integrations should fail independently.

Example:

```text id="m8q2n4"
Email Provider Down
      ↓
Orders still work
Bookings still work
Inventory still works
```

An external notification provider should not become a hidden dependency for unrelated transactions.

---

# 38. Synchronous Integrations

Synchronous external calls are appropriate when the caller genuinely needs the immediate result.

Examples may include:

```text id="v5n8q2"
Payment authorization
Identity verification
Availability check
```

The request path must account for:

- Timeout.
- Failure.
- Retry.
- User-facing error.
- Provider latency.

---

# 39. Asynchronous Integrations

Use asynchronous processing when:

- Immediate response is unnecessary.
- Processing is slow.
- Provider reliability varies.
- Multiple retries may be required.
- The operation can be eventually consistent.

Example:

```text id="c7m3q8"
KAMPYN Event
      ↓
Integration Worker
      ↓
External Provider
```

---

# 40. Webhook Architecture

Inbound webhooks should follow:

```text id="q8n4m2"
External Provider
      ↓
HTTPS Endpoint
      ↓
Authenticate / Verify Signature
      ↓
Validate Payload
      ↓
Deduplicate
      ↓
Persist / Process
      ↓
Internal Event
```

The webhook handler should remain small.

Long-running work should move to asynchronous processing.

---

# 41. Webhook Idempotency

Webhook providers may retry delivery.

KAMPYN must safely handle duplicate webhook requests.

Use:

- Provider event ID.
- Idempotency records.
- Unique database constraints.
- State transition validation.

Do not assume one webhook equals one delivery attempt.

---

# 42. Webhook Response Time

Webhook endpoints should acknowledge valid requests quickly where provider semantics allow it.

Prefer:

```text id="m3q8n7"
Webhook
  ↓
Validate
  ↓
Persist event
  ↓
Acknowledge
  ↓
Async processing
```

over performing long business workflows before responding.

---

# 43. Webhook Security

Validate:

- Signature.
- Timestamp.
- Event ID.
- Provider identity.
- Expected endpoint.
- Payload schema.

Protect against:

- Replay.
- Forged requests.
- Duplicate delivery.
- Oversized payloads.

---

# 44. External Files

External integrations may provide files through:

- SFTP.
- HTTP.
- Object storage.
- Secure file exchange.
- Institutional storage.

File processing must include:

- Size limits.
- Type validation.
- Integrity checks.
- Safe filenames.
- Malware/security scanning where appropriate.
- Streaming for large files.

Never trust external filenames or paths.

---

# 45. File Import Pipelines

A typical import flow:

```text id="x8m3q2"
External System
      ↓
File Transfer
      ↓
Validation
      ↓
Staging
      ↓
Transformation
      ↓
Domain Validation
      ↓
Transactional Import
      ↓
Result / Report
```

Large imports should be asynchronous.

---

# 46. External Search Services

If KAMPYN integrates with external search infrastructure, search must remain a derived representation.

The authoritative source remains the database.

Search failures should not corrupt business state.

---

# 47. Cloud Provider Integrations

Cloud services may provide:

- Object storage.
- Secret management.
- Compute.
- Messaging.
- Monitoring.
- CDN.
- Email infrastructure.

Provider-specific code should remain isolated.

For example:

```text id="m5q8n2"
Storage Interface
      ↓
GCP Storage Adapter
```

rather than embedding cloud SDK calls throughout domain code.

---

# 48. Storage Abstraction

The application may define:

```text id="v7n3q8"
ObjectStorage
├── Put
├── Get
├── Delete
└── GenerateAccessUrl
```

The infrastructure implementation can then use:

- Google Cloud Storage.
- S3-compatible storage.
- Institution-managed object storage.

The abstraction must remain minimal and reflect actual application needs.

---

# 49. External SDKs

Third-party SDKs must remain at integration boundaries.

Avoid importing provider SDK types into:

- Domain entities.
- Core business logic.
- Shared application models.

Prefer:

```text id="q4m8n2"
Provider SDK
      ↓
Adapter
      ↓
Internal Contract
```

This reduces provider lock-in.

---

# 50. Provider Switching

A provider should be replaceable where the business requirement justifies it.

For example:

```text id="n8m3q2"
EmailProvider
     │
 ┌───┴─────────────┐
 ↓                 ↓
Provider A      Provider B
```

Do not build elaborate abstractions for hypothetical provider changes.

The abstraction should exist because it provides a real architectural boundary.

---

# 51. Integration Health

Each important integration should expose health information.

Examples:

```text id="c5q8m2"
Provider reachable
Credentials valid
API responding
Webhook processing healthy
Queue backlog acceptable
```

Health checks must distinguish between:

```text id="m3n7q8"
Application Healthy
```

and:

```text id="v8q2m4"
Optional Integration Unavailable
```

An optional integration outage should not necessarily mark the entire application unhealthy.

---

# 52. Integration Observability

Track:

- Request count.
- Success rate.
- Failure rate.
- Latency.
- Timeout count.
- Retry count.
- Rate-limit responses.
- Circuit state.
- Webhook failures.
- Queue depth.
- Provider error codes.

Metrics should identify which provider and integration operation failed.

---

# 53. Correlation and Tracing

External calls should propagate correlation and tracing information where supported.

Example:

```text id="q7m4n2"
HTTP Request
     ↓
KAMPYN API
     ↓
Payment Adapter
     ↓
Payment Provider
```

The trace should make it possible to determine where latency or failure occurred.

Do not send internal sensitive tracing information to external providers unnecessarily.

---

# 54. External Error Mapping

Provider errors must be translated into application-level errors.

Example:

```text id="p3n8m2"
Provider Timeout
      ↓
ExternalDependencyUnavailable

Provider Invalid Request
      ↓
IntegrationValidationError

Provider Rate Limit
      ↓
ExternalRateLimited
```

The application should not depend on provider-specific error strings.

---

# 55. External Status Mapping

Provider statuses must be mapped explicitly.

Example:

```text id="m8q2n4"
Provider:
AUTHORIZED
CAPTURED
FAILED
REFUNDED

KAMPYN:
PENDING
SUCCESS
FAILED
REFUNDED
```

Mappings must be documented.

Do not assume two systems use identical state semantics.

---

# 56. Integration Transactions

Do not hold database transactions open while waiting for slow external services.

Avoid:

```text id="x4q8m2"
BEGIN TRANSACTION
      ↓
External HTTP Call
      ↓
Provider waits 5 seconds
      ↓
Database COMMIT
```

Prefer:

```text id="n7m3q8"
Commit local state
      ↓
External operation
      ↓
Persist result
      ↓
State transition
```

or an explicit workflow when atomicity is required.

---

# 57. Distributed Workflows

When an integration spans multiple systems:

```text id="q5m8n2"
KAMPYN
  ↓
Payment Provider
  ↓
Inventory
  ↓
Notification
```

do not pretend the entire operation is one database transaction.

Use explicit state machines, events, retries, and compensating actions where appropriate.

---

# 58. Idempotency Keys

External operations that may be retried should use stable idempotency keys where supported.

Examples:

```text id="m3n8q2"
Payment creation
Booking creation
External order creation
File import
Webhook processing
```

The idempotency key must represent the logical operation rather than the individual network attempt.

---

# 59. Integration State

Long-running integrations should persist their state.

Example:

```text id="v7q3m8"
PENDING
   ↓
SENT
   ↓
ACKNOWLEDGED
   ↓
COMPLETED
```

Possible failure:

```text id="x4m8n2"
SENT
  ↓
RETRYABLE_FAILURE
  ↓
RETRY
```

Do not rely solely on in-memory state for important integration workflows.

---

# 60. Reconciliation

Important integrations should support reconciliation where two systems can diverge.

Examples:

- Payments.
- University student records.
- Inventory.
- Bookings.
- External orders.

A reconciliation job may:

```text id="q8m2n4"
Fetch external state
      ↓
Compare
      ↓
Detect discrepancy
      ↓
Record discrepancy
      ↓
Repair / Escalate
```

Reconciliation must be safe and auditable.

---

# 61. Integration Queues

Integration-specific queues may isolate slow external providers.

Example:

```text id="n5q8m2"
KAMPYN
   ↓
Payment Queue
   ↓
Payment Worker
   ↓
Provider
```

Separate queues may be appropriate when different integrations have very different:

- Rate limits.
- Latency.
- Retry policies.
- Priority.
- Failure characteristics.

---

# 62. Priority

Not every integration operation has equal urgency.

For example:

```text id="m7q3n8"
Payment confirmation
   > Marketing email
```

If prioritization is required, it should be explicit.

Do not allow low-value background work to exhaust resources required for critical workflows.

---

# 63. Integration Data Mapping

External data should be mapped into internal models deliberately.

Example:

```text id="c8m2q4"
External Student
   ↓
Provider DTO
   ↓
Normalization
   ↓
KAMPYN Student Model
```

Mapping should define:

- Required fields.
- Defaults.
- Unknown fields.
- Enum mappings.
- Identifier mappings.
- Timestamp conversion.
- Null handling.

---

# 64. External IDs

Store external identifiers explicitly.

Example:

```text id="v4n8m2"
KAMPYN User ID
External Provider Subject ID
```

Do not replace internal identifiers with provider identifiers.

External IDs may change depending on provider semantics.

---

# 65. Data Ownership in Integrations

For every integrated entity, define:

```text id="q7m3n8"
Who owns creation?
Who owns updates?
Who owns deletion?
Who owns identity?
Who owns status?
```

For example:

```text id="m2n8q4"
University SIS
→ Student identity

KAMPYN
→ Food ordering state

Payment Provider
→ Provider payment transaction
```

KAMPYN may maintain its own representation of external state without claiming ownership of the external system.

---

# 66. Integration Deprecation

Providers may be replaced or discontinued.

Integration code should support controlled deprecation.

Process:

```text id="n8m4q2"
Announce
   ↓
Stop new usage
   ↓
Migrate existing data/workflows
   ↓
Disable integration
   ↓
Remove credentials
   ↓
Remove adapter
```

Do not leave abandoned provider credentials or dead integration code indefinitely.

---

# 67. Integration Testing

Integration testing should include:

- Provider contract tests.
- Adapter unit tests.
- Mocked failure scenarios.
- Sandbox tests where available.
- Webhook tests.
- Retry tests.
- Timeout tests.
- Idempotency tests.
- Authentication tests.
- Rate-limit tests.
- Reconciliation tests.

Tests must not depend entirely on live third-party systems.

---

# 68. Contract Testing

External contracts should be tested independently of business logic.

For outbound requests:

```text id="q5m8n2"
KAMPYN Adapter
      ↓
Expected Provider Contract
```

For inbound webhooks:

```text id="m3n7q8"
Provider Payload
      ↓
Schema Validation
      ↓
Expected Internal Event
```

Contract tests should detect provider changes early.

---

# 69. Sandbox Environments

Use provider sandbox environments where available.

Sandbox credentials must never be mixed with production credentials.

Production testing should use carefully controlled real transactions when unavoidable.

---

# 70. Integration Documentation

Each integration should have documentation containing:

- Provider.
- Purpose.
- Owner.
- Environment configuration.
- Authentication.
- API endpoints.
- Request/response models.
- Error mapping.
- Retry policy.
- Rate limits.
- Idempotency.
- Webhooks.
- Data ownership.
- Reconciliation.
- Monitoring.
- Failure handling.
- Decommission procedure.

---

# 71. Integration Change Checklist

Before completing an integration-related change:

- [ ] Integration owner is identified.
- [ ] Provider boundary is explicit.
- [ ] Adapter exists where appropriate.
- [ ] Provider SDK types do not leak into domain code.
- [ ] Configuration is externalized.
- [ ] Secrets are protected.
- [ ] Environment separation is correct.
- [ ] External data is validated.
- [ ] Timeouts are configured.
- [ ] Retry behavior is bounded.
- [ ] Rate limits are respected.
- [ ] Idempotency is implemented where necessary.
- [ ] Webhooks are authenticated and deduplicated.
- [ ] External statuses are explicitly mapped.
- [ ] External errors are normalized.
- [ ] Transactions do not unnecessarily span external calls.
- [ ] Failure isolation is understood.
- [ ] Reconciliation exists where required.
- [ ] Observability is available.
- [ ] Contract tests exist where appropriate.
- [ ] Documentation is updated.
- [ ] Provider deprecation implications are considered.

---

# 72. Final Integration Principle

KAMPYN should treat every external system as an unreliable boundary.

The preferred architecture is:

```text id="x8m3q2"
                 KAMPYN
                    │
             Internal Contract
                    │
              Integration
                Adapter
                    │
          ┌─────────┼─────────┐
          ↓         ↓         ↓
       Payment    Email    University
       Provider  Provider    System
```

The boundary should provide:

```text id="m4n8q2"
Validation
Authentication
Timeouts
Retries
Idempotency
Error Mapping
Observability
Isolation
```

External systems may fail, change, become slow, rate-limit requests, return unexpected data, or become unavailable.

KAMPYN must therefore remain correct even when an integration does not behave as expected.

The core principle is:

```text id="q7m3n8"
External Provider
       ↓
     Adapter
       ↓
Internal Contract
       ↓
KAMPYN Domain
```

Never:

```text id="v2m8q4"
External Provider
       ↓
Provider SDK
       ↓
Everywhere in KAMPYN
```

Integrations should be replaceable where practical, observable in production, secure by default, resilient to failure, and isolated from the core business logic.
# API Security

## 1. Purpose

This document defines the mandatory security standards for designing, implementing, testing, deploying, and maintaining APIs across KAMPYN.

APIs are a critical security boundary between users, clients, internal services, third-party integrations, and infrastructure. Every API must be designed to protect sensitive information, enforce authorization, prevent abuse, preserve data integrity, and remain resilient against malicious or unexpected requests.

These standards apply to:
- Public APIs exposed to web, mobile, SDK, and third-party clients.
- Internal APIs used for communication between backend services.
- Administrative APIs used by university administrators and platform operators.
- APIs exposed to university self-hosted deployments.
- Webhooks, callbacks, and integration endpoints.
- REST, GraphQL, WebSocket handshakes, and other HTTP-based interfaces.
- Authentication, authorization, payment, file upload, search, and realtime endpoints.

API security is mandatory regardless of whether an API is publicly accessible, internally routed, or used by a single frontend.

## 2. Core Principles

All APIs MUST follow these principles:

1. **Zero trust:** Never trust a request solely because it originates from a known client, network, service, or authenticated user.
2. **Deny by default:** Requests must be explicitly authorized. Missing or ambiguous permissions must result in denial.
3. **Server-side enforcement:** Authentication, authorization, tenant isolation, validation, and business rules must be enforced on the server.
4. **Least privilege:** Users, services, tokens, and integrations must receive only the permissions they require.
5. **Defense in depth:** Security must be enforced across the gateway, application, service, database, and infrastructure layers.
6. **Validate all input:** Treat all client-provided data as untrusted, including identifiers, headers, query parameters, payloads, and uploaded files.
7. **Minimize exposure:** Return only the information required to fulfill the request.
8. **Secure by design:** Security must be considered during API design, not added after implementation.
9. **Fail securely:** Unexpected errors, dependency failures, and ambiguous authorization states must not expose sensitive data or grant access.
10. **Auditable operations:** Sensitive actions must produce sufficient audit records without logging secrets or unnecessary personal data.
11. **Tenant isolation:** Every tenant-scoped operation must enforce tenant boundaries independently of client-supplied values.
12. **Explicit trust boundaries:** Internal service calls and third-party integrations must use authenticated and authorized communication.

## 3. Threat Model

API implementations MUST consider the following threat categories:

- Broken object-level authorization (BOLA/IDOR).
- Broken function-level authorization.
- Broken authentication and session management.
- Broken object property-level authorization and mass assignment.
- Cross-tenant data access.
- Injection attacks, including SQL, NoSQL, command, and template injection.
- Cross-site request forgery (CSRF).
- Cross-origin abuse and incorrect CORS configuration.
- Cross-site scripting (XSS) through unsafe API data handling.
- Server-side request forgery (SSRF).
- Denial of service, resource exhaustion, and API abuse.
- Brute-force attacks, credential stuffing, and account enumeration.
- Replay attacks, duplicate operations, and payment manipulation.
- Insecure file uploads and downloads.
- Sensitive information disclosure.
- Unsafe redirects and untrusted callback URLs.
- Webhook forgery and replay.
- Excessive data exposure and unrestricted resource consumption.
- Misconfigured infrastructure, secrets, and transport security.
- Supply-chain vulnerabilities in API dependencies.
- Insecure administrative and internal endpoints.

Threat models MUST be reviewed when introducing new API capabilities, trust boundaries, authentication mechanisms, integrations, or sensitive data flows.

## 4. API Design and Exposure

### 4.1 API Contracts

Every API MUST have a documented contract that defines:

- HTTP method and route.
- Purpose and resource ownership.
- Authentication requirements.
- Authorization permissions.
- Tenant scope.
- Accepted parameters and payload schemas.
- Response schemas and exposed fields.
- Validation rules and limits.
- Error response format.
- Rate limits and resource-consumption constraints.
- Idempotency requirements, where applicable.
- Audit requirements for sensitive operations.

API contracts MUST be reviewed for security before implementation is considered complete.

### 4.2 Route Design

- Use explicit, resource-oriented routes.
- Use appropriate HTTP methods and semantics.
- Do not expose internal implementation details through route names.
- Do not expose database collection names, internal service topology, or infrastructure identifiers unnecessarily.
- Avoid ambiguous routes that create inconsistent authorization behavior.
- Use versioned API contracts where incompatible changes are required.
- Do not use `GET` requests for state-changing operations.
- Do not place secrets, credentials, access tokens, or sensitive personal information in URLs.

Example:

```http
GET    /api/v1/orders/{orderId}
POST   /api/v1/orders
PATCH  /api/v1/orders/{orderId}
POST   /api/v1/orders/{orderId}/cancel
```

A well-structured route does not itself guarantee authorization. Every operation must enforce its own access rules.

### 4.3 Public and Internal APIs

Public and internal APIs MUST have clearly defined trust boundaries.

- Public endpoints must be treated as directly accessible by untrusted clients.
- Internal endpoints must authenticate calling services and enforce permissions.
- Internal network placement MUST NOT be treated as sufficient authorization.
- Administrative APIs must use dedicated permissions and stronger authentication requirements.
- Debug, diagnostic, and maintenance endpoints must not be publicly exposed.
- Internal-only endpoints must be explicitly restricted through service identity, network policy, gateway configuration, or equivalent controls.

Internal APIs MUST NOT reuse user authentication as a substitute for service-to-service authentication when a distinct service identity is required.

### 4.4 API Gateway

Where an API gateway is used, it SHOULD provide centralized capabilities such as:

- TLS termination and secure transport enforcement.
- Request routing.
- Authentication integration.
- Coarse-grained access enforcement.
- Rate limiting and request throttling.
- Request size limits.
- Abuse detection.
- Correlation IDs.
- Centralized access logging.
- API version routing.

The gateway is an additional security layer, not a replacement for authorization and validation inside backend services.

Critical security checks MUST remain enforced at the service responsible for the resource or operation.

## 5. Authentication

### 5.1 Authentication Requirements

All protected endpoints MUST authenticate the caller before performing protected operations.

Authentication MUST:
- Use established, reviewed authentication protocols and libraries.
- Validate credentials server-side.
- Reject expired, revoked, malformed, or incorrectly scoped credentials.
- Avoid insecure fallback authentication.
- Avoid trusting identity information supplied through ordinary request headers.
- Support credential revocation where the credential type permits it.
- Avoid exposing whether a specific account exists through authentication error differences.

Public endpoints may remain unauthenticated only when explicitly intended and reviewed for abuse, information disclosure, and resource-consumption risks.

### 5.2 Token Security

When bearer tokens are used:

- Tokens MUST be transmitted only over HTTPS.
- Tokens MUST be stored and handled securely by the client.
- Tokens MUST NOT appear in URLs, analytics, logs, error messages, or telemetry.
- Tokens MUST have appropriate expiration and scope.
- Signature, issuer, audience, expiration, and other required claims MUST be validated.
- Algorithm selection MUST be explicit and securely configured.
- Unsigned tokens and unsupported algorithms MUST be rejected.
- Token refresh and revocation behavior MUST be documented.
- Long-lived credentials MUST NOT be used where short-lived credentials are practical.

JWTs MUST NOT be treated as encrypted data by default. Sensitive information MUST NOT be placed in readable token claims unless there is a justified need and the disclosure is acceptable.

### 5.3 Session Security

For cookie-based sessions:

- Cookies MUST use `Secure` and `HttpOnly` where applicable.
- `SameSite` MUST be explicitly configured according to the authentication flow.
- Session identifiers MUST be unpredictable and generated using a cryptographically secure random source.
- Session identifiers MUST be rotated after authentication and privilege changes where applicable.
- Logout MUST invalidate the session server-side when server-managed sessions are used.
- Session expiration MUST be enforced server-side.
- Session fixation MUST be prevented.
- Sensitive operations SHOULD require recent authentication or additional verification.

Do not store session identifiers or bearer credentials in browser-accessible persistent storage when a safer supported design is available.

### 5.4 Password Security

When KAMPYN manages passwords:

- Passwords MUST be hashed using a modern, adaptive password-hashing algorithm such as Argon2id, with parameters appropriate to the deployment environment.
- Passwords MUST NEVER be stored in plaintext or reversibly encrypted for routine authentication.
- Password reset tokens MUST be random, short-lived, single-use, and securely stored.
- Password reset endpoints MUST be rate-limited.
- Passwords MUST NOT be logged or returned by APIs.
- Authentication errors MUST avoid revealing whether an account exists.
- Password policy MUST be documented and must not rely on arbitrary complexity rules as the only defense.

### 5.5 Multi-Factor Authentication

MFA SHOULD be supported for privileged roles and sensitive operations, including platform-level administration and high-impact university administration.

Where MFA is enabled:
- Enrollment and recovery flows MUST be protected.
- Recovery codes MUST be securely generated and stored.
- MFA challenges MUST be rate-limited.
- MFA bypasses MUST be audited.
- Changes to MFA configuration MUST require appropriate reauthentication.

## 6. Authorization and Access Control

### 6.1 Server-Side Authorization

Every protected endpoint MUST enforce authorization on the server.

Authorization checks MUST verify:
- The caller's authenticated identity.
- The required action or permission.
- The target resource.
- The resource's ownership or access relationship.
- The applicable tenant or university scope.
- Relevant resource state and business constraints.

Frontend route guards, hidden buttons, disabled controls, and client-side role checks are usability mechanisms only. They MUST NOT be treated as security controls.

### 6.2 Deny by Default

- Every protected operation MUST declare its required permissions.
- Missing permissions MUST result in denial.
- Unknown roles MUST NOT receive implicit access.
- New roles MUST NOT inherit privileged access accidentally.
- Authorization failures MUST NOT reveal protected resource contents.

A missing authorization policy is a security defect, not an acceptable default.

### 6.3 Role-Based and Attribute-Based Access

KAMPYN MAY use role-based access control (RBAC), attribute-based access control (ABAC), or a combination.

Authorization decisions SHOULD consider relevant attributes such as:
- Tenant or university.
- User role.
- Resource ownership.
- Department or organizational scope.
- Resource status.
- Relationship to the resource.
- Operation type.
- Contextual restrictions.

Roles MUST map to explicit permissions. Avoid scattered role-name checks throughout business logic when a centralized permission model is practical.

### 6.4 Object-Level Authorization

Every endpoint that accepts a resource identifier MUST verify that the authenticated caller may access that specific resource.

Examples include:
- Orders.
- Food court menus.
- Hostel and guest-house bookings.
- Complaints.
- Inventory records.
- Community messages.
- User profiles.
- Shuttle bookings.
- Library reservations.
- HR records.
- Uploaded files.

Never assume that possession of an identifier grants access.

Example of an unsafe pattern:

```go
order := repository.GetByID(ctx, orderID)
return order
```

The lookup alone does not prove the caller is permitted to access the order.

A secure implementation must establish authorization in the service or repository query boundary:

```go
order, err := repository.GetByIDForTenant(
    ctx,
    orderID,
    authenticatedTenantID,
)
if err != nil {
    return mapRepositoryError(err)
}

if !authorization.CanViewOrder(principal, order) {
    return ErrForbidden
}

return order
```

The exact implementation may vary, but object-level authorization MUST be enforced for every protected resource.

### 6.5 Function-Level Authorization

Administrative and privileged operations MUST enforce explicit function-level permissions.

Examples:
- Creating or disabling university accounts.
- Changing user roles.
- Managing food court vendors.
- Editing pricing or commission rules.
- Issuing refunds.
- Modifying inventory.
- Exporting sensitive reports.
- Viewing HR information.
- Changing tenant configuration.
- Managing platform-wide settings.

Do not rely on obscured routes, UI visibility, or HTTP method restrictions to protect privileged operations.

### 6.6 Property-Level Authorization

API request and response schemas MUST define which properties a caller may read or modify.

- Use explicit request DTOs.
- Use explicit response DTOs.
- Do not bind arbitrary client JSON directly to persistence models.
- Do not serialize database documents or ORM entities indiscriminately.
- Restrict access to sensitive fields.
- Reject or ignore unauthorized properties according to the documented contract.
- Prevent users from modifying ownership, tenant IDs, roles, payment status, or other privileged fields.

Sensitive properties include:
- Password hashes.
- Authentication secrets.
- Internal flags.
- Permission assignments.
- Payment provider identifiers and secrets.
- Internal audit metadata.
- Other users' private information.

### 6.7 Authorization Changes

Role and permission changes MUST:
- Be authorized by an appropriate privileged actor.
- Be validated against tenant and platform boundaries.
- Be audited.
- Take effect according to a documented cache and token invalidation strategy.
- Avoid allowing users to grant themselves additional permissions.

High-impact privilege changes SHOULD require reauthentication or additional approval according to the risk level.

## 7. Multi-Tenant API Security

Tenant isolation is a mandatory security boundary for KAMPYN.

### 7.1 Tenant Identification

Tenant context MUST be resolved from a trusted, authenticated source, such as a validated identity claim or a server-controlled tenant mapping.

Client-provided tenant IDs, hostnames, query parameters, or headers MUST NOT independently establish tenant authority.

If a request includes a tenant identifier, the backend MUST verify that the authenticated principal is authorized to act within that tenant.

### 7.2 Tenant-Scoped Operations

Every tenant-scoped operation MUST enforce tenant isolation at the service and data-access layers.

- Tenant filters MUST be applied to reads, updates, deletes, and aggregations.
- Tenant ownership MUST be validated when creating relationships between records.
- Tenant context MUST be propagated to downstream services and background jobs.
- Cache keys MUST include tenant scope when the data is tenant-specific.
- Search queries MUST be constrained to the authorized tenant.
- File access MUST verify tenant ownership.
- Tenant-specific exports MUST not include records from other tenants.
- Tenant-scoped events MUST carry verified tenant context.

### 7.3 Cross-Tenant Access

Cross-tenant access MUST be denied unless a specific, documented platform-level operation authorizes it.

Platform operators with cross-tenant permissions MUST use dedicated permissions and auditable workflows.

Never use a user-supplied tenant ID as the sole filter for a database operation.

### 7.4 Tenant Context Propagation

Tenant context MUST be propagated explicitly through:
- HTTP middleware.
- Application services.
- Repository calls.
- Background jobs.
- Event handlers.
- Cache operations.
- Search operations.
- Service-to-service requests.

Avoid mutable global tenant state, implicit request context leakage, or shared singleton state that can cause one request's tenant context to affect another.

### 7.5 Self-Hosted Deployments

Self-hosted university deployments MUST preserve the same API security guarantees as the SaaS environment.

- Tenant or deployment boundaries MUST be clearly defined.
- Default configuration MUST be secure.
- Setup procedures MUST not require exposing internal services publicly.
- Secrets MUST be configured outside source code and container images.
- Administrative bootstrap credentials MUST be rotated or invalidated after setup.
- Operators MUST be provided with security configuration and upgrade guidance.

Self-hosting does not remove the requirement for authentication, authorization, auditability, and secure defaults.

## 8. Input Validation and Schema Enforcement

### 8.1 General Validation

All client-controlled input MUST be treated as untrusted.

Validate:
- Path parameters.
- Query parameters.
- Request headers used by application logic.
- Request bodies.
- File metadata.
- Pagination parameters.
- Sort fields.
- Search expressions.
- Filters.
- Webhook payloads.
- External integration responses before they enter trusted application logic.

Validation MUST occur at the API boundary and MUST be reinforced by domain-level rules where necessary.

### 8.2 Zod and Backend Validation

For TypeScript-based API boundaries, use Zod or an equivalent runtime schema-validation library.

For Go APIs, use explicit typed request structures and a validated schema or validation layer appropriate to the service.

- Keep schemas close to their API contracts.
- Reject malformed payloads with consistent client errors.
- Enforce string, numeric, array, and object limits.
- Reject unexpected fields for security-sensitive request DTOs.
- Distinguish syntactic validation from business-rule validation.
- Never treat TypeScript types as runtime validation.

### 8.3 Validation Requirements

Validation MUST cover:
- Required and optional fields.
- Data types and formats.
- Maximum and minimum lengths.
- Numeric ranges.
- Enum membership.
- Array item limits.
- Nested object depth and size.
- Identifier format.
- Date and time validity.
- Currency and amount constraints.
- File type and size restrictions.
- Allowed sort and filter fields.
- Business state transitions.

Validation errors MUST NOT disclose stack traces, internal schema implementation details, or sensitive submitted values.

### 8.4 Injection Prevention

- Use parameterized queries for SQL.
- Use safe query builders or properly structured query APIs.
- Never concatenate untrusted input into SQL, MongoDB operators, shell commands, or search query syntax.
- Restrict dynamic field names and sort expressions to explicit allowlists.
- Avoid evaluating user-provided expressions.
- Sanitize or safely encode data when rendering it in HTML or other active contexts.
- Use context-appropriate output encoding rather than relying on generic string sanitization.

For MongoDB, do not pass arbitrary client JSON directly as a query filter or update document. Construct permitted query operators and fields on the server.

For OpenSearch, build queries from validated, structured input. Do not accept unrestricted query DSL from ordinary clients unless explicitly designed, permissioned, and isolated.

## 9. Request Size and Resource Limits

Every API MUST enforce resource-consumption limits appropriate to its purpose.

Controls SHOULD include:
- Maximum request body size.
- Maximum header size.
- Maximum query-string length.
- Maximum URL length.
- Maximum array length and object depth.
- Maximum page size.
- Maximum upload size.
- Maximum concurrent requests per principal or tenant.
- Database query timeouts.
- Downstream request timeouts.
- Bounded background job queues.
- Maximum export size or export execution duration.

Limits MUST be configured at both the edge and application layers where appropriate.

Do not allow unbounded:
- Pagination.
- Search queries.
- Batch operations.
- Recursive payloads.
- File uploads.
- Report generation.
- Expensive aggregation requests.
- WebSocket message sizes.
- Concurrent long-running requests.

Large exports and expensive reports SHOULD use asynchronous jobs with status polling or notifications, rather than holding an API request open indefinitely.

## 10. Rate Limiting and Abuse Prevention

### 10.1 Rate Limiting

Rate limits MUST be applied to public and sensitive endpoints based on risk.

Rate-limiting dimensions MAY include:
- IP address.
- Authenticated user.
- Tenant.
- API key.
- Device or session identifier, where reliable and privacy-appropriate.
- Endpoint or operation class.

Use distributed rate limiting where multiple application instances must share enforcement state.

Rate limits MUST be configurable and observable. They SHOULD account for legitimate differences in workload and avoid relying exclusively on IP addresses.

### 10.2 Sensitive Endpoints

Apply stricter controls to:
- Login.
- Password reset.
- OTP issuance and verification.
- MFA challenges.
- Signup and invitation flows.
- Search suggestions.
- Payment and refund operations.
- Bulk imports and exports.
- File uploads.
- Administrative operations.
- Public-facing contact or complaint submission.

### 10.3 Abuse Detection

Where appropriate, detect:
- Repeated failed authentication attempts.
- Credential stuffing patterns.
- Enumeration attempts.
- Excessive resource lookups.
- Suspicious changes in request volume.
- Repeated payment or booking attempts.
- Abnormal tenant-level traffic.
- Malicious upload patterns.

Abuse controls MUST avoid creating an easy denial-of-service vector against legitimate users.

### 10.4 Rate-Limit Responses

Use HTTP `429 Too Many Requests` when a request is rate-limited.

Where suitable, return a `Retry-After` header. Error responses MUST not disclose internal rate-limiter keys, detection logic, or sensitive abuse signals.

## 11. CORS and Browser Security

### 11.1 CORS Configuration

CORS MUST be configured using an explicit allowlist of trusted origins.

- Do not use wildcard origins for credentialed requests.
- Do not reflect arbitrary `Origin` headers.
- Do not treat CORS as authentication or authorization.
- Allow only required HTTP methods and headers.
- Keep exposed response headers to the minimum required.
- Restrict credentialed requests to approved origins.
- Validate environment-specific origin configuration at startup.

Development origins MUST NOT automatically be trusted in production.

### 11.2 Credentialed Requests

When cookies or other browser credentials are used:
- Configure `Access-Control-Allow-Credentials` only when necessary.
- Use explicit allowed origins.
- Apply CSRF protections.
- Ensure cookie attributes align with the authentication design.

### 11.3 Security Headers

Browser-facing APIs and web applications SHOULD apply appropriate security headers, including:
- `Strict-Transport-Security` over HTTPS.
- `X-Content-Type-Options: nosniff`.
- `Content-Security-Policy` on browser-rendered surfaces.
- `Referrer-Policy`.
- Appropriate framing restrictions through CSP `frame-ancestors` or equivalent.

Headers MUST be configured based on the application's actual behavior and must not create false confidence or break required functionality.

## 12. CSRF Protection

Cookie-authenticated state-changing endpoints MUST implement appropriate CSRF defenses.

Controls SHOULD include:
- Anti-CSRF tokens for applicable flows.
- `SameSite` cookie settings.
- Origin or Referer validation where appropriate.
- Strict handling of content types and state-changing methods.

Do not rely exclusively on CORS to prevent CSRF.

Bearer-token APIs that do not automatically attach credentials through the browser have a different CSRF risk profile, but they still require secure token storage and XSS defenses.

## 13. HTTP and Transport Security

- All production API traffic MUST use HTTPS.
- HTTP MUST redirect to HTTPS or be rejected at the appropriate edge.
- TLS configuration MUST use modern, supported protocol versions and secure cipher configuration.
- Internal service communication SHOULD use TLS, with mutual TLS considered for sensitive service boundaries.
- Certificates MUST be renewed and monitored.
- Sensitive endpoints MUST NOT downgrade to insecure transport.
- Insecure TLS verification MUST NOT be disabled in production.
- API clients MUST validate server certificates.
- Secrets and credentials MUST NOT be sent over plaintext protocols.

Transport security requirements MUST be maintained across reverse proxies, load balancers, service meshes, and self-hosted deployments.

## 14. Sensitive Data Protection

### 14.1 Data Minimization

APIs MUST return only the data necessary for the authorized operation.

- Use explicit response DTOs.
- Exclude internal metadata unless required.
- Mask sensitive values where full disclosure is unnecessary.
- Avoid returning full user records for unrelated operations.
- Restrict bulk access to sensitive information.
- Apply data retention and privacy requirements to API outputs and logs.

### 14.2 Sensitive Information

Sensitive data MAY include:
- Authentication credentials and tokens.
- Personal contact information.
- Student and staff records.
- HR and disciplinary information.
- Payment details.
- Booking details.
- Private messages and community content.
- Complaints and reports.
- Internal administrative metadata.
- Uploaded documents and private media.

Access to sensitive information MUST be purpose-limited and authorized.

### 14.3 Error and Response Safety

API responses MUST NOT expose:
- Passwords or password hashes.
- Access or refresh tokens.
- Secret keys.
- Database connection strings.
- Internal file paths.
- Stack traces.
- Infrastructure credentials.
- Raw internal exceptions.
- Unnecessary personal information.
- Cross-tenant records.

Use stable, documented error codes and safe messages.

### 14.4 Encryption

Sensitive data MUST be protected in transit and at rest according to its classification and applicable requirements.

Encryption keys MUST be managed separately from the encrypted data. Use managed key services or an equivalent secure key-management approach where available.

Do not implement custom cryptographic algorithms.

## 15. Mass Assignment and Data Integrity

API request payloads MUST NOT be mapped directly into database models when doing so could modify protected fields.

Use dedicated request DTOs and explicit field mapping.

Fields that MUST be server-controlled where applicable include:
- User and tenant ownership.
- Role and permission assignments.
- Payment status.
- Order fulfillment state.
- Booking approval state.
- Refund state.
- Audit metadata.
- Creation and modification timestamps.
- Internal moderation or administrative flags.

Domain state transitions MUST be enforced in the backend, not accepted directly from the client.

For example, a client MUST NOT be able to mark an order as paid by submitting:

```json
{
  "status": "paid"
}
```

Payment status must be derived from a trusted payment-provider verification or an authorized, audited administrative process.

## 16. Business Logic and Workflow Security

APIs MUST enforce valid domain workflows, not just valid payload formats.

### 16.1 Orders and Payments

- Verify that the user is authorized to place or access an order.
- Validate item availability, pricing, and quantity server-side.
- Calculate totals on the server.
- Do not trust client-submitted prices, discounts, fees, or payment status.
- Verify payment callbacks using the provider's documented verification mechanism.
- Enforce idempotency for payment initiation and other retryable financial operations.
- Prevent duplicate charges, duplicate fulfillment, and unauthorized refunds.
- Audit payment and refund state changes.

### 16.2 Bookings and Reservations

For hostel, guest-house, shuttle, washing-machine, and other booking workflows:
- Verify resource availability on the server.
- Validate booking ownership.
- Enforce permitted state transitions.
- Prevent overlapping or conflicting reservations.
- Use database constraints, transactions, or concurrency-safe mechanisms to prevent double booking.
- Require appropriate permissions for cancellation, approval, and administrative overrides.
- Audit high-impact changes.

### 16.3 Inventory

- Validate stock changes against current authoritative inventory.
- Restrict adjustments to authorized roles.
- Use concurrency-safe updates for stock reservation and deduction.
- Prevent clients from directly overriding authoritative stock counts.
- Audit manual adjustments and inventory reconciliation.

### 16.4 Complaints, Reports, and Moderation

- Restrict access to complaint and report records to authorized users.
- Protect reporter identity where required.
- Prevent unauthorized changes to review or resolution status.
- Enforce moderator and administrator permissions.
- Audit sensitive moderation actions.
- Avoid exposing private report contents through broad administrative APIs.

### 16.5 Community and Messaging

- Enforce conversation membership and message visibility on every operation.
- Validate permissions for editing, deleting, reporting, and moderating messages.
- Restrict attachment access to authorized conversation participants.
- Apply message and attachment limits.
- Rate-limit high-volume actions.
- Do not assume that a message or conversation ID establishes membership.

## 17. Idempotency and Replay Protection

Operations that may be retried or duplicated MUST be designed to preserve data integrity.

Idempotency SHOULD be implemented for:
- Payment initiation.
- Order creation where duplicate submissions are harmful.
- Refund requests.
- Booking creation.
- Webhook processing.
- Other externally retried operations with non-repeatable side effects.

Where idempotency keys are used:
- Bind them to the authenticated principal and applicable tenant.
- Validate their format and length.
- Store them with a bounded retention period.
- Associate them with the request's operation and result.
- Reject conflicting reuse of the same key.
- Ensure concurrent requests with the same key cannot create duplicate effects.

Idempotency does not replace authorization, transaction safety, or payment-provider verification.

## 18. Webhook Security

### 18.1 Inbound Webhooks

Inbound webhooks MUST:
- Verify signatures using the provider's documented mechanism.
- Use constant-time comparison where comparing secret-derived values.
- Validate timestamps or equivalent freshness indicators where supported.
- Prevent replay of already-processed events.
- Validate payload schemas.
- Enforce request size and rate limits.
- Avoid trusting event contents before signature verification.
- Be idempotent.
- Record safe audit metadata.

Webhook signing secrets MUST be stored in a secure secret manager or equivalent protected configuration.

### 18.2 Outbound Webhooks

Outbound webhook functionality MUST:
- Restrict destination configuration to authorized administrators.
- Validate destination URLs and DNS resolution policy.
- Prevent SSRF and access to internal or metadata endpoints.
- Use HTTPS where supported.
- Sign outgoing payloads.
- Apply bounded retry policies with backoff.
- Avoid retrying non-retryable responses indefinitely.
- Protect delivery logs from sensitive payload exposure.

### 18.3 Webhook Processing

Webhook handlers SHOULD acknowledge verified events quickly and delegate long-running work to controlled background processing.

Processing MUST account for duplicate, delayed, out-of-order, and malformed events.

## 19. SSRF and URL Handling

Any endpoint or service that fetches a URL based on user-controlled input MUST implement SSRF protections.

- Prefer allowlisted destinations where feasible.
- Restrict allowed URL schemes.
- Reject loopback, private, link-local, multicast, and reserved IP ranges unless explicitly required.
- Prevent access to cloud instance metadata endpoints.
- Validate DNS results and account for DNS rebinding.
- Revalidate destinations after redirects.
- Restrict redirect behavior.
- Apply outbound network controls.
- Enforce timeouts and response-size limits.
- Avoid forwarding internal credentials to user-controlled destinations.

URL validation MUST be performed server-side. String-based checks alone are insufficient to establish that a destination is safe.

## 20. File Upload and Download Security

### 20.1 Uploads

File upload endpoints MUST:
- Require authorization.
- Enforce size and request limits.
- Validate file extensions, MIME types, and actual file content where practical.
- Generate server-side storage names.
- Prevent path traversal and filename-based path manipulation.
- Store files outside executable application directories.
- Restrict archive extraction and guard against archive bombs.
- Scan files for malware where appropriate to the threat model.
- Avoid trusting client-provided metadata.
- Enforce tenant and owner association.
- Apply retention and deletion policies.

File type validation MUST NOT rely solely on the filename or client-provided `Content-Type`.

### 20.2 Downloads

Download endpoints MUST:
- Authorize access to the specific file.
- Verify tenant and ownership boundaries.
- Avoid exposing direct storage paths.
- Use short-lived, scoped signed URLs when direct object-storage access is required.
- Prevent public access to private files.
- Apply safe content-disposition and content-type behavior.
- Ensure deleted or revoked files are no longer accessible according to the defined revocation model.

### 20.3 Upload Processing

File parsing and transformation SHOULD run in isolated, resource-limited workers.

Apply limits to:
- Decompressed size.
- Number of archive entries.
- Nesting depth.
- Parsing duration.
- Memory consumption.
- Image dimensions.
- Document complexity.

Do not process untrusted files in privileged application contexts.

## 21. Search API Security

KAMPYN may use OpenSearch or other search infrastructure for items, vendors, facilities, and related content.

Search endpoints MUST:
- Enforce authorization and tenant scope.
- Validate search terms, filters, sort fields, and pagination.
- Restrict query complexity and response size.
- Prevent arbitrary query DSL access unless explicitly authorized.
- Avoid leaking records through autocomplete, facets, counts, or suggestions.
- Apply rate limits to high-volume search operations.
- Avoid exposing internal index names, mappings, or backend errors unnecessarily.

Search results MUST NOT be treated as the authoritative source for permission checks. Before returning sensitive records or performing state changes, enforce authorization against the authoritative data and applicable domain rules.

## 22. WebSocket and Realtime API Security

WebSocket and realtime connections MUST apply authentication and authorization controls equivalent to protected HTTP APIs.

- Authenticate during connection establishment using a secure supported mechanism.
- Avoid long-lived credentials in URLs.
- Authorize channel or room membership server-side.
- Verify access for subscription, publish, edit, delete, and moderation actions.
- Enforce message size, frequency, and connection limits.
- Revalidate authorization when required by the session or membership model.
- Handle token expiry and revocation.
- Prevent cross-tenant room access.
- Validate event payloads.
- Avoid broadcasting sensitive data to unauthorized subscribers.
- Apply origin validation for browser-based connections where appropriate.
- Limit connection lifetime and idle duration according to operational requirements.

A successful WebSocket handshake MUST NOT imply authorization for every subsequent message or channel.

## 23. API Keys and Service-to-Service Authentication

### 23.1 API Keys

API keys MUST:
- Be generated using cryptographically secure randomness.
- Be scoped to an intended integration or client.
- Be stored securely, preferably as a verifier or hash where feasible.
- Support revocation and rotation.
- Have usage limits and monitoring.
- Be transmitted only over HTTPS.
- Never be embedded in public frontend bundles.
- Never be committed to source control.
- Be excluded from logs and error responses.

API keys identify a client or integration; they MUST NOT automatically grant user-level permissions.

### 23.2 Service Identity

Internal services SHOULD use explicit service identities and scoped credentials.

Service-to-service authentication MAY use:
- Mutual TLS.
- Short-lived signed tokens.
- Cloud workload identity.
- Another reviewed identity mechanism suitable for the deployment.

Services MUST authenticate callers and authorize individual operations. Shared credentials with broad permissions SHOULD be avoided.

### 23.3 Credential Rotation

Credential rotation procedures MUST define:
- Ownership.
- Expiration or rotation intervals where applicable.
- Emergency revocation.
- Deployment update strategy.
- Verification of successful rotation.
- Audit and incident response steps.

Avoid hardcoded credentials, fallback secrets, or credentials shared across unrelated environments.

## 24. Error Handling

### 24.1 Consistent Error Format

APIs SHOULD return a consistent error structure.

Example:

```json
{
  "error": {
    "code": "RESOURCE_NOT_FOUND",
    "message": "The requested resource was not found",
    "requestId": "request-correlation-id"
  }
}
```

The exact schema may vary by API version, but error responses MUST be documented and consistent.

### 24.2 Safe Error Semantics

Use appropriate HTTP status codes, such as:
- `400 Bad Request` for invalid request syntax or validation failures.
- `401 Unauthorized` for missing or invalid authentication.
- `403 Forbidden` for authenticated callers lacking permission.
- `404 Not Found` when a resource is unavailable or intentionally concealed.
- `409 Conflict` for state or concurrency conflicts.
- `413 Content Too Large` for request-size violations.
- `415 Unsupported Media Type` for unsupported request formats.
- `422 Unprocessable Content` where used consistently for domain validation.
- `429 Too Many Requests` for rate limiting.
- `500 Internal Server Error` for unexpected server failures.
- `503 Service Unavailable` for temporary service unavailability.

Choose response behavior that does not disclose the existence or details of protected resources.

### 24.3 Internal Error Handling

- Catch and map expected errors at appropriate boundaries.
- Do not return raw exception messages to clients.
- Do not expose stack traces in production responses.
- Preserve safe diagnostic context in protected logs.
- Avoid swallowing security failures.
- Do not retry authorization failures as transient dependency errors.

## 25. Logging, Monitoring, and Auditing

### 25.1 Security Logging

Security-relevant API events SHOULD include:
- Authentication successes and failures.
- Authorization denials.
- Privilege and role changes.
- Tenant configuration changes.
- Sensitive data exports.
- Payment and refund state changes.
- Account recovery and MFA changes.
- API key creation, revocation, and rotation.
- Webhook verification failures.
- Suspicious request patterns.
- Administrative actions.
- Significant security configuration changes.

### 25.2 Log Safety

Logs MUST NOT contain:
- Passwords.
- Access tokens or refresh tokens.
- API keys.
- Session identifiers.
- Payment credentials.
- Private cryptographic material.
- Full sensitive request bodies unless specifically justified and protected.
- Unnecessary personal data.

Use structured logging and correlation IDs. Sanitize untrusted values to prevent log injection.

### 25.3 Monitoring

Monitor:
- Request volume and latency.
- Authentication and authorization failure rates.
- Rate-limit activity.
- Server and dependency errors.
- Unusual tenant traffic.
- Resource exhaustion.
- Suspicious access patterns.
- Webhook verification and delivery failures.
- Sensitive operation frequency.

Alerts MUST be actionable and routed to the responsible operators. Monitoring systems and dashboards MUST enforce access controls and data minimization.

### 25.4 Audit Trails

Audit logs for sensitive operations SHOULD record:
- Actor identity or service identity.
- Tenant context.
- Action performed.
- Target resource.
- Timestamp.
- Outcome.
- Correlation or request identifier.
- Relevant change metadata, excluding secrets.

Audit records MUST be protected against unauthorized modification and access. Retention MUST follow documented legal, privacy, and operational requirements.

## 26. Database and Downstream Service Security

API security MUST extend to database and downstream service access.

- Use least-privilege database credentials.
- Restrict service accounts to required operations.
- Apply tenant-aware query constraints.
- Use parameterized queries.
- Enforce database constraints for critical invariants.
- Set connection and query timeouts.
- Avoid leaking database errors.
- Secure service-to-service requests.
- Validate downstream responses before trusting them.
- Propagate request deadlines and cancellation where supported.
- Avoid unbounded retries and retry storms.
- Prevent untrusted API input from controlling internal service destinations.

The API layer MUST NOT assume that downstream systems will independently enforce every business authorization rule.

## 27. Dependency and Configuration Security

- Use maintained, trusted API frameworks and libraries.
- Keep dependencies updated through a controlled process.
- Scan dependencies for known vulnerabilities.
- Review authentication, cryptography, serialization, and networking libraries carefully.
- Pin or lock dependency versions using the repository's package-management policy.
- Remove unused dependencies and endpoints.
- Validate security-sensitive configuration at startup.
- Fail startup when required secrets or security configuration are missing or invalid.
- Separate development, staging, and production credentials.
- Never use production secrets in local development or automated tests.
- Restrict access to deployment configuration and secret stores.

Production security settings MUST NOT be silently weakened by missing environment variables or invalid configuration.

## 28. API Versioning and Deprecation

- API versions MUST have documented compatibility expectations.
- Breaking changes MUST be versioned or migrated through a controlled compatibility plan.
- Deprecated endpoints MUST have an explicit removal process.
- Old API versions MUST continue to enforce current security requirements while supported.
- Unsupported versions SHOULD be disabled after the published deprecation period.
- Authentication, authorization, and tenant isolation MUST NOT be weakened to preserve legacy compatibility.

Versioning MUST NOT result in duplicated security logic that diverges across versions.

## 29. Secure API Implementation Architecture

API security responsibilities SHOULD be separated into clear layers.

### 29.1 Middleware

Middleware MAY handle cross-cutting concerns such as:
- Request IDs.
- Authentication context establishment.
- Request size limits.
- Rate limiting.
- Security headers.
- CORS.
- Request logging.
- Deadline propagation.

Middleware MUST NOT become the only place where domain-specific resource authorization is enforced.

### 29.2 Handlers

Handlers SHOULD:
- Parse and validate incoming requests.
- Resolve authenticated principal and tenant context.
- Invoke application services.
- Map application errors to safe HTTP responses.
- Avoid embedding complex business rules.
- Avoid direct persistence access where it bypasses required service controls.

### 29.3 Application Services

Application services SHOULD:
- Enforce domain-level authorization.
- Validate business rules and state transitions.
- Coordinate transactions.
- Enforce idempotency and concurrency requirements.
- Call repositories and integrations through defined interfaces.
- Produce audit events for sensitive operations.

### 29.4 Repositories

Repositories SHOULD:
- Enforce required data-scope constraints.
- Use safe, parameterized database access.
- Expose cohesive domain-oriented operations.
- Avoid returning unrestricted records to callers without a valid need.
- Support transactional operations where required.

Security MUST be enforced across layers where necessary. No layer should assume that another layer's presence guarantees every security property.

## 30. Testing Requirements

Every API MUST have security-focused tests appropriate to its risk.

### 30.1 Authentication Tests

Test:
- Missing credentials.
- Invalid credentials.
- Expired credentials.
- Revoked credentials.
- Incorrect issuer or audience.
- Invalid token signature.
- Malformed authorization headers.
- Session expiration and invalidation.
- Password reset token expiry and reuse.
- MFA verification and recovery paths, where applicable.

### 30.2 Authorization Tests

Test:
- Unauthenticated access.
- Authenticated but unauthorized access.
- Wrong role or missing permission.
- Cross-user resource access.
- Cross-tenant resource access.
- Unauthorized property modification.
- Privileged endpoint access.
- Unauthorized state transitions.
- Access after role revocation or account suspension.

Tests MUST verify denial, not just successful access by authorized users.

### 30.3 Input and Injection Tests

Test:
- Invalid types and formats.
- Oversized payloads.
- Unexpected fields.
- Malformed identifiers.
- Injection payloads.
- Invalid pagination and sorting.
- Deeply nested objects.
- Invalid file metadata.
- Malformed webhook payloads.
- Invalid URL and redirect cases.

### 30.4 Abuse and Resilience Tests

Test:
- Rate limiting.
- Brute-force protections.
- Duplicate submissions.
- Idempotency conflicts.
- Concurrent booking and inventory operations.
- Database timeouts.
- Downstream service failures.
- Oversized uploads and downloads.
- Expensive search and report requests.
- WebSocket connection and message limits.

### 30.5 Integration and Tenant Isolation Tests

Integration tests MUST verify tenant isolation across:
- Database queries.
- Search results.
- Caches.
- File storage.
- Background jobs.
- Event processing.
- WebSocket channels.
- Administrative workflows.

Use multiple synthetic tenants and users in automated tests to detect accidental data leakage.

### 30.6 Automated Security Testing

CI SHOULD include suitable automated checks such as:
- Static application security testing (SAST).
- Dependency vulnerability scanning.
- Secret scanning.
- API contract validation.
- Dynamic application security testing (DAST), where practical.
- Targeted fuzzing for complex parsers and sensitive boundaries.

Automated tools complement, but do not replace, manual security review and threat modeling.

## 31. Performance and Security Trade-offs

Security controls MUST be implemented without introducing avoidable performance or availability problems.

- Use efficient authorization checks and tenant-aware query patterns.
- Avoid repeated redundant permission lookups within a single operation.
- Cache authorization data only when invalidation and revocation semantics are safe.
- Avoid unbounded security-event processing in synchronous request paths.
- Use bounded asynchronous processing for non-blocking audit and notification work where the audit guarantee is preserved.
- Apply request deadlines and bounded concurrency.
- Avoid expensive cryptographic or validation work on unbounded input.
- Profile security-sensitive paths without disabling required controls.

Performance optimization MUST NOT remove authorization checks, weaken tenant isolation, or bypass validation.

## 32. Incident Response

API services MUST support investigation and containment of security incidents.

Procedures SHOULD cover:
- Credential compromise.
- Unauthorized data access.
- Cross-tenant exposure.
- Payment or booking manipulation.
- Webhook compromise.
- API abuse and denial of service.
- Vulnerable dependencies.
- Accidental secret disclosure.

Response capabilities SHOULD include:
- Revoking credentials.
- Disabling compromised integrations.
- Restricting or disabling affected endpoints.
- Applying emergency rate limits.
- Preserving relevant audit evidence.
- Identifying affected tenants and resources.
- Communicating through established incident procedures.
- Deploying and verifying corrective changes.

Incident response MUST follow the organization's documented security and privacy requirements.

## 33. Prohibited Practices

The following are prohibited:

- Trusting frontend authorization as the security boundary.
- Accepting client-supplied tenant IDs without verification.
- Performing resource lookups without object-level authorization.
- Binding arbitrary request payloads directly to privileged database models.
- Returning raw database entities without response-field control.
- Storing passwords in plaintext.
- Hardcoding credentials or secrets.
- Logging tokens, passwords, or secret keys.
- Using wildcard CORS for credentialed production APIs.
- Disabling TLS verification in production.
- Exposing stack traces or raw internal errors.
- Using unbounded pagination, uploads, or expensive operations.
- Trusting payment status supplied by a client.
- Trusting unsigned or unverified webhooks.
- Allowing arbitrary user-controlled URLs to be fetched server-side.
- Exposing internal or administrative endpoints without explicit protection.
- Relying on obscure route names as access control.
- Skipping authorization because an endpoint is considered internal.
- Using unrestricted query filters or operators from client payloads.
- Retrying non-idempotent operations without safeguards.
- Treating successful authentication as authorization for all resources.
- Disabling security controls to resolve performance or integration issues without an approved, documented alternative.

## 34. API Security Review Checklist

### Design
- [ ] API purpose, exposure, and trust boundary are documented.
- [ ] Authentication requirements are explicit.
- [ ] Required permissions are defined.
- [ ] Tenant scope is defined.
- [ ] Request and response schemas are documented.
- [ ] Sensitive fields and operations are identified.
- [ ] Rate limits and resource budgets are defined.
- [ ] Error behavior is documented.

### Authentication and Authorization
- [ ] Authentication is enforced where required.
- [ ] Credentials are validated securely.
- [ ] Object-level authorization is enforced.
- [ ] Function-level authorization is enforced.
- [ ] Property-level access is restricted.
- [ ] Tenant isolation is enforced.
- [ ] Privileged actions are audited.
- [ ] Revocation and session lifecycle behavior are defined.

### Input and Data
- [ ] All untrusted input is validated.
- [ ] Request payloads use explicit DTOs.
- [ ] Mass assignment is prevented.
- [ ] Database queries are injection-safe.
- [ ] Search and sort inputs are allowlisted.
- [ ] Response data is minimized.
- [ ] Sensitive information is protected.
- [ ] State transitions are validated server-side.

### Abuse and Transport
- [ ] Request and response sizes are bounded.
- [ ] Rate limiting is configured.
- [ ] Sensitive endpoints have stricter abuse protections.
- [ ] HTTPS is enforced.
- [ ] CORS uses explicit allowed origins.
- [ ] CSRF protection is implemented where applicable.
- [ ] Timeouts and concurrency limits are configured.
- [ ] SSRF and redirect risks are addressed where applicable.

### Integrations and Files
- [ ] Webhook signatures are verified.
- [ ] Replay and duplicate processing are handled.
- [ ] API keys and service identities are scoped.
- [ ] File uploads are validated and bounded.
- [ ] File downloads enforce authorization.
- [ ] External service responses are validated.
- [ ] Third-party secrets are securely managed.

### Operations and Testing
- [ ] Security events are logged safely.
- [ ] Audit records are available for sensitive actions.
- [ ] Secrets are excluded from logs.
- [ ] Authentication tests exist.
- [ ] Authorization and cross-tenant tests exist.
- [ ] Input-validation and abuse tests exist.
- [ ] Dependency and secret scanning are enabled.
- [ ] Failure and incident procedures are documented.
- [ ] Security configuration is validated for deployment.

## 35. Definition of Done

An API is considered production-ready only when:

1. Its authentication and authorization requirements are documented and enforced.
2. Every protected resource operation performs appropriate object-level authorization.
3. Tenant isolation is enforced across all relevant data and infrastructure layers.
4. All untrusted input is validated using runtime validation and domain rules.
5. Sensitive fields are explicitly controlled in request and response contracts.
6. Request sizes, execution time, concurrency, and resource consumption are bounded.
7. Rate limiting and abuse controls are applied according to risk.
8. Transport, CORS, CSRF, and browser security requirements are correctly configured where applicable.
9. Sensitive workflows enforce business rules, concurrency safety, and idempotency as required.
10. Webhooks, files, integrations, and realtime interfaces are protected according to their trust boundaries.
11. Errors and logs do not expose secrets, internal details, or unauthorized data.
12. Security-relevant events are auditable.
13. Automated tests cover authentication, authorization, tenant isolation, validation, and relevant abuse cases.
14. Security scanning and code review requirements are satisfied.
15. Deployment configuration, secrets, monitoring, and incident response are ready.
16. The API has passed the required security review for its risk level.

**No API may be considered complete solely because its functional tests pass. Security requirements are part of the API contract and Definition of Done.**
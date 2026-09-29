# Secure Coding Standards

## 1. Purpose

This document defines mandatory secure coding standards for KAMPYN to prevent vulnerabilities, protect sensitive data, maintain tenant isolation, and ensure the integrity and availability of the platform.

These standards apply to all production code, including:

- Backend services written in Go
- Frontend applications built with Next.js, React, and TypeScript
- API endpoints, middleware, and authentication systems
- Database queries, repositories, and migrations
- Background workers and scheduled jobs
- Real-time communication and community features
- SDKs, integrations, and webhooks
- Infrastructure automation and deployment tooling
- Self-hosted university installations
- Tests, scripts, and internal developer tooling that handle sensitive data

Secure coding is a mandatory engineering requirement, not an optional quality improvement.

All implementations must follow the relevant policies under `.ai/security/`, particularly authentication, authorization, API security, input validation, secrets management, dependency security, and payments.

## 2. Core Security Principles

Every implementation must follow these principles:

- **Secure by default:** Systems must use secure defaults and require explicit decisions to weaken protections.
- **Defense in depth:** No single security control should be the only barrier against a serious vulnerability.
- **Least privilege:** Users, services, processes, and integrations must receive only the permissions they require.
- **Deny by default:** Access must be explicitly authorized. Missing permissions or uncertain identity must result in denial.
- **Zero trust:** Never assume that a request, service, network, tenant, or internal component is trustworthy merely because it is inside the system.
- **Fail securely:** Errors, timeouts, invalid credentials, and unavailable security dependencies must not bypass security checks.
- **Minimize attack surface:** Avoid unnecessary endpoints, dependencies, privileges, exposed data, and complex abstractions.
- **Explicit trust boundaries:** Validate and authorize all data crossing application, service, tenant, and infrastructure boundaries.
- **Data minimization:** Collect, process, store, and expose only the information required for the operation.
- **Safe composition:** Reuse reviewed security controls instead of implementing custom cryptographic, authentication, or authorization mechanisms.
- **Auditable behavior:** Security-sensitive actions must produce appropriate audit records without exposing confidential information.
- **Tenant isolation:** No operation may access or modify another tenant's resources without explicit, validated authorization.

## 3. Secure Development Lifecycle

Security must be incorporated into every stage of software development.

| Stage | Mandatory security activities |
|---|---|
| Requirements | Identify sensitive data, security boundaries, abuse cases, and compliance requirements |
| Architecture | Threat modeling, trust boundaries, access control, and failure analysis |
| Design | Secure data flows, validation, authorization, and safe defaults |
| Implementation | Follow language-specific secure coding standards |
| Code review | Review security-sensitive logic and identify vulnerabilities |
| Testing | Unit, integration, authorization, security, and abuse-case testing |
| CI/CD | Static analysis, dependency scanning, secret scanning, and security gates |
| Deployment | Secure configuration, restricted permissions, and protected infrastructure |
| Monitoring | Audit events, suspicious activity detection, and security alerts |
| Maintenance | Patching, vulnerability remediation, reviews, and controlled deprecation |

Security requirements must be identified before implementation, rather than added after a feature is complete.

## 4. Trust Boundaries

Every application must explicitly identify its trust boundaries.

KAMPYN trust boundaries include:

- Browser to frontend server
- Frontend server to backend API
- External clients and SDKs to public APIs
- Backend services communicating with one another
- Application services to databases and caches
- Backend services to OpenSearch
- Application services to third-party integrations
- Payment providers to payment webhooks
- Background workers to message brokers and job queues
- University-managed systems to KAMPYN integrations
- One tenant's context to another tenant's context
- Administrators and operators to privileged infrastructure

All data crossing a trust boundary must be treated as untrusted until appropriately validated and authorized.

Internal network access, private endpoints, or service-to-service communication must not be considered substitutes for authentication and authorization.

## 5. Input Validation and Data Sanitization

All external input must be validated at the appropriate boundary.

### 5.1 Validation Requirements

Validate:

- HTTP request bodies
- URL parameters and query strings
- HTTP headers and cookies
- Uploaded files and metadata
- Webhook payloads
- SDK input
- User-generated content
- Event and queue messages
- Third-party API responses
- Tenant configuration
- Environment configuration

Validation must include:

- Type and structure
- Required and optional fields
- Length and size limits
- Allowed formats and enumerated values
- Numeric ranges
- Character encoding
- Business constraints
- Context-specific rules

Use Zod for TypeScript runtime validation and explicit typed request structures with appropriate validation in Go.

Validation schemas must be reusable where the same contract is shared across application boundaries, without coupling unrelated domains.

### 5.2 Reject Invalid Data

Invalid input must be rejected before it reaches sensitive business logic or persistence operations.

Do not silently coerce invalid input into privileged or security-sensitive values.

Unknown fields should be rejected or explicitly handled according to the API contract. Security-sensitive endpoints should not accept arbitrary additional properties.

### 5.3 Normalize Carefully

Normalize input only when the domain requires it.

Examples include:

- Trimming permitted whitespace
- Normalizing identifiers
- Validating Unicode
- Applying consistent case handling where appropriate

Do not normalize passwords, cryptographic material, signed payloads, or other values where exact byte representation matters.

### 5.4 Validation Is Not Authorization

Validating that an identifier is syntactically correct does not mean the caller may access the corresponding resource.

Every resource operation must separately enforce authorization and tenant ownership.

Refer to `.ai/security/input-validation.md`.

## 6. Injection Prevention

Injection vulnerabilities occur when untrusted input is interpreted as executable instructions.

### 6.1 SQL Injection

All database operations must use parameterized queries or safe ORM query APIs.

Prohibited:

```go
query := "SELECT * FROM users WHERE email = '" + email + "'"
```

Preferred:

```go
row := db.QueryRowContext(
    ctx,
    "SELECT id, email FROM users WHERE email = $1",
    email,
)
```

Requirements:

- Never concatenate untrusted input into SQL statements.
- Use parameterized queries for values.
- Validate dynamic identifiers such as table names, column names, and sort directions against strict allowlists.
- Use narrowly scoped database credentials.
- Apply tenant constraints to every tenant-scoped query.

ORM usage does not automatically guarantee safety. Raw query features must receive additional review.

### 6.2 NoSQL Injection

MongoDB queries must be constructed using typed, validated fields and trusted query operators.

Never pass arbitrary user-supplied JSON directly into database filters, updates, or aggregation pipelines.

Prohibited:

```typescript
const user = await User.findOne(req.body);
```

Preferred:

```typescript
const user = await User.findOne({
  email: validatedInput.email,
});
```

Never permit clients to control database operators, aggregation stages, or update expressions unless each operation is explicitly constrained and authorized.

### 6.3 Command Injection

Avoid invoking operating system commands from application code whenever a library or native API can perform the operation.

If process execution is unavoidable:

- Use fixed executable paths.
- Pass arguments separately rather than constructing shell commands.
- Validate every dynamic argument against strict constraints.
- Apply execution timeouts.
- Restrict process permissions.
- Limit accessible directories and resources.
- Avoid shell interpretation.

Never execute untrusted strings as shell commands.

### 6.4 Template Injection

Do not interpret user-provided content as executable template syntax.

Use context-aware template engines with automatic escaping and prevent access to arbitrary functions, objects, and runtime capabilities.

### 6.5 LDAP, XPath, and Other Query Languages

Where external input is used in specialized query languages:

- Use safe parameterization or escaping provided by trusted libraries.
- Validate inputs against strict schemas.
- Restrict available operators and expressions.
- Avoid constructing executable expressions through string concatenation.

## 7. Cross-Site Scripting (XSS)

All user-controlled content must be treated as potentially executable.

### 7.1 React and Next.js

Use React's default escaping for rendered text.

Avoid:

- `dangerouslySetInnerHTML`
- Direct DOM manipulation with untrusted content
- Unvalidated HTML rendering
- Dynamic script injection
- Unsafe URL construction
- Rendering arbitrary SVG or HTML supplied by users

If rich HTML is a required feature, sanitize it with a maintained, security-reviewed sanitizer and a strict allowlist of permitted elements, attributes, and URL schemes.

### 7.2 Context-Aware Output

Ensure output is safe for its destination:

- HTML text
- HTML attributes
- URLs
- JavaScript contexts
- CSS contexts
- Markdown rendering

HTML escaping alone is not sufficient for JavaScript, CSS, or URL contexts.

### 7.3 Content Security Policy

Deploy a restrictive Content Security Policy (CSP) appropriate to the application's architecture.

Where feasible:

- Use nonces or hashes for permitted scripts.
- Avoid unrestricted `unsafe-inline` and `unsafe-eval`.
- Restrict script, object, frame, and connection sources.
- Restrict embedding by untrusted origins.
- Review CSP changes as security-sensitive modifications.

CSP is a defense-in-depth measure and does not replace safe rendering.

### 7.4 User-Generated Content

Community posts, comments, messages, reviews, complaints, and profile content must be rendered safely.

Content moderation, HTML sanitization, and output encoding must be treated as separate controls.

## 8. Authentication Security

Authentication must use reviewed, centralized mechanisms rather than custom implementations scattered across services.

### 8.1 Password Handling

- Store passwords only as strong, salted password hashes.
- Use a modern password-hashing algorithm such as Argon2id or another approved adaptive password-hashing algorithm.
- Never store plaintext passwords.
- Never log or return passwords.
- Avoid custom password hashing algorithms.
- Enforce secure password reset and account recovery flows.
- Prevent user enumeration through authentication and recovery responses where appropriate.
- Apply rate limiting and abuse prevention to authentication endpoints.

### 8.2 Session and Token Security

- Use appropriately scoped and time-limited authentication tokens.
- Protect signing keys through approved secret management.
- Validate token signature, issuer, audience, expiry, and relevant claims.
- Reject invalid, expired, malformed, or incorrectly scoped tokens.
- Prevent token leakage through URLs, logs, or client-visible errors.
- Implement revocation or session invalidation for relevant compromise scenarios.
- Apply secure cookie attributes when cookies are used.

### 8.3 Authentication State

Authentication state must be derived from trusted server-side verification.

Never trust a client-supplied user ID, role, tenant ID, or authentication flag as proof of identity.

Refer to `.ai/security/authentication.md`.

## 9. Authorization Security

Authorization must be enforced server-side for every protected operation.

Requirements:

- Apply deny-by-default authorization.
- Enforce role-based, attribute-based, and relationship-based permissions where appropriate.
- Validate resource ownership and tenant membership.
- Check access at the object, function, and property levels.
- Prevent privilege escalation and insecure direct object references.
- Revalidate permissions for sensitive state transitions.
- Avoid relying on frontend visibility or disabled controls.
- Ensure background jobs and internal service operations enforce the required permissions or trusted service policies.

Authorization must be checked at the operation boundary, not only at the beginning of a user session.

Refer to `.ai/security/authorization.md`.

## 10. Multi-Tenant Security

Tenant isolation is a fundamental security requirement for KAMPYN.

### 10.1 Tenant Context

Tenant context must be derived from a trusted authentication and routing mechanism.

Never trust a tenant identifier solely because it was supplied in:

- Request bodies
- Query parameters
- Headers
- URL paths
- Frontend state
- SDK configuration

The server must verify that the authenticated principal is permitted to act within the requested tenant.

### 10.2 Database Isolation

Every tenant-scoped database operation must enforce tenant constraints.

Requirements:

- Include tenant scope in repository methods and query boundaries.
- Enforce tenant ownership on reads, writes, updates, and deletes.
- Prevent unscoped queries from unintentionally exposing tenant records.
- Use database-level safeguards where supported.
- Test tenant isolation across all relevant data stores.

### 10.3 Cache Isolation

Cache keys for tenant-specific data must include the tenant scope.

Never allow one tenant to receive cached data belonging to another tenant.

Cache invalidation and cache refresh operations must also respect tenant boundaries.

### 10.4 Search Isolation

All tenant-scoped OpenSearch queries must enforce tenant filtering server-side.

Tenant filters must not be removable or overridable through client-controlled query parameters.

### 10.5 Cross-Tenant Operations

Cross-tenant operations must be explicitly authorized, narrowly scoped, audited, and implemented through trusted administrative workflows.

Refer to `.ai/architecture/multi-tenancy.md` and `.ai/security/authorization.md`.

## 11. Cryptography

Cryptography must use established, maintained libraries and approved algorithms.

### 11.1 Approved Practices

- Use TLS for network communication.
- Use modern authenticated encryption for confidential data at rest where application-level encryption is required.
- Use secure cryptographic random number generators.
- Use established key derivation and password-hashing algorithms.
- Use authenticated encryption modes such as AES-GCM or an approved equivalent.
- Use cryptographic hashes only for suitable purposes.
- Use constant-time comparison for secret values where supported and relevant.
- Manage cryptographic keys through approved secret management or KMS services.

### 11.2 Prohibited Practices

Never:

- Design custom cryptographic algorithms.
- Use obsolete or broken algorithms for new security-sensitive features.
- Use MD5 or SHA-1 for password storage.
- Reuse keys across unrelated purposes.
- Hardcode encryption keys.
- Store encryption keys beside encrypted data without appropriate key protection.
- Use predictable random values for security tokens.
- Disable certificate verification in production.
- Treat Base64 encoding as encryption.

### 11.3 Sensitive Data

Sensitive data must be encrypted in transit and protected at rest according to its classification and risk.

Encryption does not replace authorization, access control, or data minimization.

Refer to `.ai/security/secrets.md`.

## 12. API Security

All public and internal APIs must enforce appropriate security controls.

Requirements:

- Authenticate protected endpoints.
- Authorize every sensitive operation.
- Validate requests and enforce size limits.
- Apply rate limits and abuse controls.
- Use safe pagination and bounded query complexity.
- Enforce tenant isolation.
- Return minimal response data.
- Use consistent and safe error responses.
- Apply appropriate CORS policies.
- Prevent mass assignment and property-level authorization flaws.
- Protect state-changing operations from CSRF where applicable.
- Ensure sensitive operations have appropriate audit records.

### 12.1 Mass Assignment

Do not bind untrusted request bodies directly to privileged database models.

Use explicit request DTOs or validated input structures that expose only fields the caller is allowed to modify.

Privileged fields such as `role`, `tenantId`, `isAdmin`, `status`, `balance`, and ownership identifiers must be assigned only through trusted business logic.

### 12.2 API Errors

Client-facing errors must not reveal:

- Internal stack traces
- SQL queries
- Database connection details
- Secret values
- Internal service topology
- Private file paths
- Sensitive account information

Use stable error codes and safe, actionable messages.

Refer to `.ai/security/api-security.md`.

## 13. Cross-Site Request Forgery (CSRF)

Applications using browser-managed credentials such as cookies must protect state-changing operations against CSRF.

Requirements:

- Use appropriate `SameSite` cookie settings.
- Validate request origin or equivalent trusted context where appropriate.
- Use CSRF tokens where required by the authentication architecture.
- Restrict credentialed CORS to explicitly trusted origins.
- Avoid state-changing operations through GET requests.
- Ensure sensitive actions require the appropriate authorization and confirmation.

CSRF protections must be evaluated in the context of the actual authentication mechanism.

## 14. Cross-Origin Resource Sharing (CORS)

CORS must be configured using explicit origin allowlists.

Requirements:

- Allow only trusted origins that require API access.
- Avoid unrestricted wildcard origins with credentials.
- Restrict permitted methods and headers.
- Avoid reflecting arbitrary request origins.
- Separate development origins from production origins.
- Review origin changes as security-sensitive configuration.

CORS is a browser access control mechanism, not a replacement for API authentication or authorization.

## 15. File Upload and Processing Security

File uploads must be treated as untrusted input.

Requirements:

- Enforce maximum file size and request limits.
- Validate file formats using content inspection rather than trusting file extensions alone.
- Generate server-controlled storage names.
- Prevent path traversal and arbitrary filesystem writes.
- Store uploaded files outside executable application directories.
- Restrict access to uploaded content.
- Scan files where risk and infrastructure justify it.
- Apply archive extraction limits and protect against archive bombs.
- Prevent malicious files from being rendered as trusted active content.
- Enforce tenant ownership for uploaded files and downloads.
- Avoid exposing internal storage paths.

For document previews, image processing, and other transformations, use isolated processing components with restricted privileges and resource limits.

## 16. Path Traversal and Filesystem Security

All filesystem operations must use controlled paths and permissions.

Requirements:

- Do not construct filesystem paths directly from untrusted input.
- Resolve paths against a known safe base directory.
- Validate the resolved path remains within the permitted directory.
- Avoid following untrusted symbolic links.
- Use least-privilege filesystem permissions.
- Avoid writing uploaded or generated files into executable locations.
- Apply file size and resource limits.
- Restrict access to secrets, configuration, and system files.

Never assume that removing `../` sequences alone provides sufficient path traversal protection.

## 17. Business Logic Security

Business logic must enforce the security and integrity requirements of each domain.

### 17.1 Food Ordering

- Calculate prices on the server.
- Verify item availability and vendor ownership.
- Validate order status transitions.
- Prevent unauthorized order modification.
- Enforce idempotency for duplicate submissions.
- Validate discounts and promotional rules server-side.
- Prevent unauthorized access to another user's order details.

### 17.2 Payments

- Never trust client-supplied payment success states.
- Verify provider responses and webhook signatures.
- Enforce idempotent payment processing.
- Validate payment and refund state transitions.
- Prevent duplicate fulfillment and double refunds.
- Maintain an auditable payment ledger.

### 17.3 Bookings

- Verify availability on the server before confirming a booking.
- Prevent concurrent allocation of the same resource.
- Enforce ownership and cancellation permissions.
- Validate booking windows and permitted transitions.
- Use transactional or concurrency-safe allocation mechanisms.

### 17.4 Inventory

- Enforce server-side stock validation.
- Prevent unauthorized inventory changes.
- Protect stock adjustment and reconciliation operations.
- Use atomic updates or transactions to avoid race conditions.
- Record sensitive inventory changes in audit logs.

### 17.5 Complaints and HR

- Restrict access to private complaints and HR records.
- Enforce field-level access for sensitive information.
- Prevent unauthorized status changes or assignment.
- Protect attachments and internal comments.
- Audit access to sensitive records where appropriate.

### 17.6 Community and Messaging

- Validate membership before allowing access to private channels.
- Enforce message visibility and moderation permissions.
- Prevent unauthorized message edits and deletions.
- Restrict attachment access by channel and tenant.
- Protect real-time events with server-side authorization.
- Prevent replay, impersonation, and unauthorized event publication.

## 18. Race Conditions and Concurrency

Security-sensitive operations must remain correct under concurrent execution.

Examples include:

- Payment confirmation
- Refund processing
- Booking allocation
- Inventory deduction
- Role and permission changes
- Account recovery
- Invitation redemption
- Coupon redemption

Requirements:

- Use transactions, atomic operations, or concurrency-safe coordination where appropriate.
- Enforce database uniqueness and integrity constraints.
- Use idempotency keys for operations vulnerable to retries.
- Avoid time-of-check-to-time-of-use vulnerabilities.
- Ensure retries cannot bypass business authorization or duplicate side effects.
- Define safe behavior for concurrent conflicting requests.

Application-level checks alone are insufficient where concurrent operations can invalidate assumptions.

## 19. Resource Exhaustion and Denial of Service

All externally reachable operations must enforce reasonable resource limits.

Requirements:

- Apply request body and file size limits.
- Enforce connection and execution timeouts.
- Use bounded concurrency.
- Apply rate limiting to sensitive and resource-intensive endpoints.
- Restrict expensive search and aggregation operations.
- Limit pagination size and query depth.
- Prevent unbounded memory allocation.
- Apply backpressure to queues and worker pools.
- Restrict decompression and archive extraction resources.
- Protect authentication, search, upload, and messaging endpoints against abuse.

Avoid algorithms and queries with unexpectedly high complexity for user-controlled input.

Prefer bounded, streaming, and efficient operations over unbounded in-memory processing.

Refer to `.ai/performance/algorithms.md` and `.ai/performance/complexity.md`.

## 20. Error Handling and Failure Security

Errors must not reveal internal implementation details or cause security controls to be bypassed.

Requirements:

- Handle errors explicitly.
- Avoid exposing raw internal exceptions to clients.
- Fail closed for authorization and authentication failures.
- Reject operations when required security dependencies cannot establish a trustworthy result.
- Use timeouts and bounded retries for external dependencies.
- Prevent retries from duplicating sensitive side effects.
- Ensure partial failures do not leave records in unauthorized or inconsistent states.
- Avoid fallback behavior that silently weakens security.

If a security-critical dependency is unavailable, the affected operation must fail safely.

## 21. Logging, Auditing, and Sensitive Data

Logging must support troubleshooting and security investigation without exposing confidential information.

### 21.1 Never Log

- Passwords
- Access or refresh tokens
- Private keys
- Secret values
- Full payment credentials
- Authentication cookies
- Sensitive personal data without a documented need
- Complete authorization headers
- Sensitive webhook signatures

### 21.2 Security Audit Events

Record relevant security-sensitive events, including:

- Authentication success and failure
- Authorization denials
- Privilege and role changes
- Administrative operations
- Secret access and rotation
- Payment state transitions
- Sensitive record access where required
- Account recovery and security setting changes
- Tenant configuration changes
- Suspicious activity and abuse controls

Audit events must contain sufficient context for investigation while minimizing personal and confidential data.

### 21.3 Log Integrity

Security audit logs must be protected from unauthorized modification and deletion.

Access must be restricted, retention must be defined, and sensitive log access must itself be controlled.

## 22. Secure Go Coding Standards

Go services must follow secure language and runtime practices.

### 22.1 Error Handling

- Handle errors explicitly.
- Do not ignore errors from security-sensitive operations.
- Wrap errors with useful context without including secret values.
- Avoid returning internal errors directly through public APIs.
- Use typed or categorized errors where they improve safe handling.

### 22.2 Context and Timeouts

- Propagate `context.Context` across request and service boundaries.
- Use context cancellation for database, network, and other long-running operations.
- Apply bounded timeouts to external dependencies.
- Avoid unbounded goroutine creation.
- Ensure request cancellation does not leave sensitive operations in inconsistent states.

### 22.3 Memory and Concurrency

- Protect shared mutable state with appropriate synchronization.
- Avoid data races in authentication, authorization, and tenant context.
- Do not store request-specific security context in unsafe global mutable variables.
- Bound worker pools and channels.
- Avoid retaining sensitive data longer than necessary.

### 22.4 HTTP Security

- Use appropriate HTTP server timeouts.
- Validate methods, paths, headers, and request bodies.
- Set safe response headers.
- Limit request sizes.
- Avoid exposing debug endpoints in production.
- Use TLS at the appropriate deployment boundary.
- Ensure proxy and forwarded headers are trusted only from configured infrastructure.

### 22.5 Database Access

- Use parameterized queries.
- Propagate request context.
- Avoid unbounded query results.
- Enforce tenant constraints.
- Use least-privilege database identities.
- Handle transaction and rollback errors correctly.

### 22.6 Unsafe Operations

The `unsafe` package must not be used in production code without explicit technical justification, security review, and documented safety guarantees.

## 23. Secure TypeScript and Next.js Coding Standards

Frontend and server-side TypeScript code must use strict typing and explicit security boundaries.

### 23.1 Type Safety

- Enable TypeScript strict mode.
- Avoid `any`.
- Prefer explicit types at trust boundaries.
- Use discriminated unions for security-sensitive states.
- Use runtime validation for external data.
- Avoid unsafe type assertions that bypass validation.
- Never treat compile-time types as proof that external input is valid.

### 23.2 Server and Client Boundaries

- Keep confidential operations server-side.
- Use server-only modules for secret-dependent functionality.
- Avoid leaking server configuration through client props.
- Enforce authorization in server-side data access and mutations.
- Treat client state as untrusted.
- Never rely on route visibility or UI controls as authorization.

### 23.3 React Security

- Prefer safe, declarative rendering.
- Avoid unsafe HTML injection.
- Clean up subscriptions, event handlers, and sensitive transient state where appropriate.
- Prevent stale authentication state from enabling unauthorized operations.
- Avoid persisting sensitive tokens in browser storage without a reviewed security design.

### 23.4 Next.js Security

- Protect Route Handlers and Server Actions with authentication and authorization.
- Validate all incoming request data.
- Avoid caching personalized responses across users or tenants.
- Review server-side caching and revalidation for sensitive information.
- Use secure cookie settings where applicable.
- Ensure redirects use validated destinations to prevent open redirect vulnerabilities.
- Avoid exposing internal server errors or secrets through rendered pages and API responses.

### 23.5 State Management

- Use TanStack Query for server state and apply correct authorization-aware cache keys.
- Use Zustand for appropriate client-side state, not as an authorization authority.
- Clear or invalidate sensitive cached state when the user logs out, changes tenant, or loses relevant permissions.
- Avoid storing secrets in persistent client-side stores.
- Prevent data from one authenticated session from appearing in another.

## 24. Secure Database Coding

Database access must preserve confidentiality, integrity, and tenant isolation.

### 24.1 Data Access Layer

- Centralize database access in cohesive repositories or domain-specific data access modules.
- Apply tenant scope consistently.
- Use parameterized queries or safe query builders.
- Enforce input validation before persistence.
- Avoid exposing internal database models directly as public API contracts.
- Restrict write operations to authorized business workflows.

### 24.2 Integrity Constraints

Use database constraints where appropriate to reinforce application-level rules:

- Unique constraints
- Foreign keys
- Check constraints
- Not-null constraints
- Tenant-scoped uniqueness
- Atomic conditional updates

### 24.3 Data Exposure

- Select only required fields.
- Avoid unrestricted population or joins that disclose private data.
- Apply field-level authorization.
- Use bounded pagination.
- Avoid exposing database identifiers as authorization credentials.
- Ensure exports and analytics enforce tenant and role restrictions.

### 24.4 Migrations

Database migrations must not weaken access control or expose sensitive data.

Migrations must be reviewed for:

- Permission changes
- Tenant isolation
- Sensitive column handling
- Data retention
- Encryption requirements
- Safe rollback or recovery
- Operational access requirements

## 25. Secure Caching

Caching must preserve the same security boundaries as the underlying data.

Requirements:

- Include tenant and user scope in cache keys when required.
- Do not cache private responses in shared public caches.
- Define appropriate TTLs for sensitive information.
- Invalidate data after relevant permission, ownership, or tenant changes.
- Avoid storing credentials or unnecessary personal data in Redis.
- Restrict Redis access using network controls and authentication.
- Protect cache refresh and invalidation operations.
- Ensure cache failures cannot bypass authorization.

Authorization must be enforced even when data is retrieved from a cache.

Refer to `.ai/performance/cache-strategy.md`.

## 26. Secure Events and Background Processing

Events and queued jobs must be treated as untrusted at consumption boundaries, even when produced internally.

Requirements:

- Validate message schemas.
- Authenticate and authorize event producers where applicable.
- Enforce tenant scope.
- Restrict event payloads to necessary data.
- Avoid embedding secrets or unnecessary personal information.
- Use idempotent processing for retryable operations.
- Validate state transitions when processing delayed events.
- Restrict access to queues, topics, and consumer identities.
- Protect dead-letter queues and replay mechanisms.
- Audit privileged or sensitive background actions.

Consumers must not assume that an event is valid solely because it came from an internal broker.

## 27. Secure Real-Time Communication

WebSocket and other real-time systems must apply authentication and authorization throughout the connection lifecycle.

Requirements:

- Authenticate connection establishment.
- Validate origin where applicable.
- Authorize channel and room membership.
- Revalidate permissions for sensitive events.
- Prevent unauthorized event publication and subscription.
- Apply message size limits and rate limiting.
- Enforce tenant isolation.
- Handle token expiry, revocation, and permission changes.
- Avoid exposing confidential information in connection errors.
- Ensure connection cleanup on logout or loss of authorization.

An authenticated connection must not be treated as permanently authorized for every subsequent operation.

## 28. Secure External Integrations

Third-party integrations must be treated as separate trust domains.

Requirements:

- Authenticate outgoing requests.
- Verify incoming webhook signatures.
- Validate external response schemas.
- Apply connection and execution timeouts.
- Restrict allowed destinations to reduce SSRF risk.
- Avoid leaking credentials in provider error messages.
- Apply retry limits and idempotency.
- Handle provider outages without bypassing business rules.
- Restrict permissions granted to external integrations.
- Monitor unusual or unexpected integration behavior.

External data must not be trusted simply because it is returned by an established provider.

## 29. Server-Side Request Forgery (SSRF)

Features that retrieve remote URLs or interact with external resources must defend against SSRF.

Requirements:

- Validate URL schemes and hostnames.
- Restrict destinations to explicitly permitted domains or destinations where feasible.
- Block loopback, link-local, private, and internal infrastructure destinations unless explicitly required and secured.
- Validate resolved IP addresses and protect against DNS rebinding.
- Restrict redirects or revalidate every redirect destination.
- Apply outbound network controls.
- Enforce request timeouts and response size limits.
- Prevent access to cloud metadata services and internal administrative endpoints.

Do not allow arbitrary user-supplied URLs to be fetched by privileged backend services without a reviewed security design.

## 30. Secure Redirects and URL Handling

Redirect destinations and user-provided URLs must be validated.

Requirements:

- Prefer relative internal paths for internal redirects.
- Use explicit allowlists for permitted external redirect destinations.
- Reject malformed, ambiguous, or unsupported URL schemes.
- Prevent `javascript:` and other dangerous schemes from being treated as safe links.
- Avoid using untrusted URLs in authentication and recovery flows.
- Validate callback and return URLs in OAuth and third-party integrations.

## 31. Identity and Privilege Management

Privileged operations must be explicitly controlled.

Examples include:

- Role assignment
- Permission changes
- Tenant configuration
- Financial adjustments
- Refund approval
- Inventory overrides
- HR record access
- Platform-level administration
- User suspension or account recovery

Requirements:

- Enforce server-side authorization.
- Validate the actor's authority for the specific action.
- Restrict self-approval for sensitive workflows where separation of duties is required.
- Record relevant audit events.
- Apply additional verification for high-impact operations.
- Avoid implicit privilege inheritance.
- Revalidate privileges when executing delayed or asynchronous operations.

## 32. Secure Defaults and Configuration

All application and infrastructure configuration must default to safe behavior.

Requirements:

- Disable debug and development endpoints in production.
- Disable permissive CORS by default.
- Reject missing security-critical configuration.
- Use secure cookie and transport settings.
- Require authentication for protected routes.
- Use restrictive filesystem and process permissions.
- Disable unnecessary features and services.
- Separate production from non-production credentials.
- Apply safe timeouts and resource limits.
- Ensure insecure settings cannot be activated accidentally through untrusted input.

Configuration changes that weaken security must receive explicit review.

## 33. Dependency and Supply Chain Security

Third-party code is part of the application attack surface.

Requirements:

- Use trusted package registries.
- Pin dependencies using supported lockfiles or reproducible version mechanisms.
- Review dependencies before introducing them.
- Minimize unnecessary runtime dependencies.
- Scan dependencies for known vulnerabilities.
- Review suspicious install scripts and transitive dependencies.
- Remove abandoned or unnecessary packages.
- Verify package provenance where supported.
- Keep critical dependencies patched.
- Generate and maintain software bills of materials where required.

Avoid implementing custom security-sensitive functionality when a maintained, reviewed library is appropriate.

Refer to `.ai/security/dependency-security.md`.

## 34. Secure File and Data Deletion

Data deletion must be implemented with appropriate authorization and lifecycle controls.

Requirements:

- Verify deletion authority.
- Apply tenant ownership checks.
- Prevent unauthorized bulk deletion.
- Respect retention and legal hold requirements.
- Handle related objects, files, indexes, caches, and events.
- Protect soft-deleted data from unauthorized retrieval.
- Define secure deletion and key retirement procedures where applicable.
- Audit sensitive deletion operations.

Deleting a database row alone may not remove its data from caches, search indexes, backups, or object storage.

## 35. Privacy by Design

Secure coding must minimize unnecessary personal and sensitive data processing.

Requirements:

- Collect only data needed for a documented purpose.
- Restrict access to sensitive personal records.
- Avoid unnecessary replication across services.
- Minimize personal information in event payloads and logs.
- Define data retention and deletion behavior.
- Protect exports and reporting features.
- Ensure tenant-specific data is not exposed through shared analytics.
- Avoid using sensitive data in test environments without appropriate safeguards.

Privacy requirements must be considered during schema design, API development, logging, and analytics implementation.

## 36. Security Testing

Security must be tested at multiple levels.

### 36.1 Unit Testing

Test security-sensitive functions, including:

- Input validation
- Authentication helpers
- Authorization policies
- Tenant scope enforcement
- Token validation
- Sensitive state transitions
- Data masking and redaction
- Error handling
- Idempotency and replay prevention

### 36.2 Integration Testing

Test interactions across:

- API and authentication middleware
- Authorization and database repositories
- Tenant-scoped data stores
- Redis and cache boundaries
- Payment and webhook integrations
- Background workers and message brokers
- File storage and upload processing
- Search and analytics services

### 36.3 Negative Testing

Security tests must verify that unauthorized operations fail.

Examples:

- User accesses another user's resource.
- Tenant A attempts to access Tenant B's records.
- A user modifies privileged request fields.
- A revoked token is reused.
- A webhook is submitted with an invalid signature.
- A request contains unexpected database operators.
- A user attempts to invoke an administrative endpoint.
- A stale cached response is requested after permission revocation.
- A file upload attempts path traversal.
- A redirect attempts to navigate to an untrusted destination.

### 36.4 Fuzz Testing

Use fuzz testing for parsers, validators, serialization, and other components that process complex or untrusted input.

Fuzz tests should target:

- Malformed JSON and XML
- Unicode edge cases
- File metadata and content parsers
- URL handling
- Query parameter parsing
- Protocol messages
- Archive processing
- Complex validation rules

### 36.5 Security Regression Tests

Every confirmed security vulnerability must result in a regression test where technically feasible.

The test must demonstrate that the vulnerability is prevented and that the fix does not introduce an alternate path to the same weakness.

## 37. Static Analysis and CI Security Gates

Security checks must be integrated into CI/CD.

Required checks should include:

- Language-specific static analysis
- Type checking
- Linting
- Dependency vulnerability scanning
- Secret scanning
- Container image scanning
- Infrastructure configuration scanning
- Relevant automated security tests

Critical security findings must block production deployment until resolved or covered by a formally approved exception.

High-risk findings must be reviewed before merging or deployment according to the project's vulnerability remediation policy.

Security tooling must be configured to avoid treating suppressed warnings as resolved vulnerabilities without justification.

## 38. Code Review Requirements

All production changes must receive code review.

Security-sensitive changes require focused review by an engineer with relevant security knowledge.

Changes requiring additional scrutiny include:

- Authentication and session management
- Authorization and permission models
- Tenant isolation
- Cryptographic operations
- Secret management
- Payment processing and refunds
- File upload and remote resource fetching
- Database query construction
- Administrative workflows
- Webhooks and external integrations
- Realtime communication
- Data exports and bulk operations
- Security middleware and infrastructure policies

Reviewers must examine both the intended behavior and potential abuse cases.

A change must not be approved solely because its tests pass.

## 39. Vulnerability Management

Identified vulnerabilities must be assessed based on:

- Severity
- Exploitability
- Exposure
- Privilege required
- Potential impact
- Affected tenants and data
- Availability of mitigations
- Evidence of active exploitation

Critical and high-risk vulnerabilities must be prioritized for immediate remediation according to `.ai/security/dependency-security.md` and the incident response process.

Security fixes must be validated, documented, and deployed through controlled procedures.

Known vulnerabilities must not be ignored merely because exploitation has not yet been observed.

## 40. Security Documentation

Security-sensitive features must have sufficient documentation for safe operation and maintenance.

Document where relevant:

- Trust boundaries
- Authentication and authorization assumptions
- Sensitive data flows
- Required permissions
- Secret dependencies
- Failure and recovery behavior
- Security configuration
- Abuse prevention controls
- Audit events
- Operational procedures
- Known limitations and approved exceptions

Documentation must never contain active credentials, private keys, access tokens, or other real secret values.

## 41. Secure Coding Anti-Patterns

The following patterns are prohibited or require explicit security justification and review:

| Anti-pattern | Risk |
|---|---|
| Trusting client-supplied roles or tenant IDs | Privilege escalation and cross-tenant access |
| Directly binding request bodies to database models | Mass assignment |
| Constructing SQL or NoSQL queries from raw input | Injection |
| Returning internal exceptions to clients | Information disclosure |
| Using frontend checks as authorization | Unauthorized access |
| Storing privileged credentials in browser storage | Credential theft |
| Caching sensitive data without proper scope | Data leakage |
| Using unrestricted HTML rendering | XSS |
| Fetching arbitrary user-supplied URLs | SSRF |
| Using unbounded request processing | Resource exhaustion |
| Performing non-atomic financial operations | Double spending or inconsistent state |
| Skipping webhook signature verification | Forged events |
| Using shared administrative credentials | Excessive privilege and poor attribution |
| Disabling TLS verification | Man-in-the-middle exposure |
| Logging complete request objects | Sensitive data exposure |
| Ignoring security-related errors | Fail-open behavior |
| Using custom cryptography | Cryptographic implementation flaws |
| Relying on hidden UI controls for security | Broken access control |
| Trusting cached authorization decisions indefinitely | Stale privilege enforcement |
| Accepting arbitrary event payloads | Internal trust boundary compromise |

## 42. Security Performance

Security controls must be implemented efficiently without compromising their effectiveness.

Requirements:

- Centralize reusable security middleware.
- Avoid duplicated authorization and validation logic.
- Use efficient data structures for permission evaluation.
- Bound expensive validation and cryptographic operations.
- Apply caching only where security context and invalidation are well defined.
- Avoid unnecessary repeated database calls while preserving access checks.
- Use rate limiting to protect expensive operations.
- Profile security-sensitive paths under realistic workloads.
- Ensure performance optimizations do not bypass security requirements.

Performance improvements must never silently weaken authentication, authorization, tenant isolation, or data integrity.

## 43. AI-Assisted Development Security

All AI-generated or AI-modified code must follow the same security standards as manually written code.

AI-generated code must not be trusted without review.

Requirements:

- Review authentication and authorization logic manually.
- Validate generated database queries and data access patterns.
- Inspect dependencies introduced by generated code.
- Check for hardcoded secrets and unsafe configuration.
- Review input validation and output encoding.
- Examine concurrency and race conditions.
- Verify tenant isolation.
- Run relevant tests and static analysis.
- Reject insecure shortcuts, deprecated APIs, and unverified assumptions.

AI coding agents must consult the relevant `.ai/security/` documentation before making substantial security-sensitive changes.

Generated code must not introduce duplicate security middleware or parallel authorization systems when an approved implementation already exists.

## 44. Security Review Checklist

Before merging or releasing a feature, verify:

### Input and Output
- [ ] All external input is validated at trust boundaries.
- [ ] Input size and complexity are bounded.
- [ ] Database and command injection risks are addressed.
- [ ] User-generated content is rendered safely.
- [ ] Sensitive output fields are explicitly controlled.

### Authentication and Authorization
- [ ] Protected operations require authentication.
- [ ] Server-side authorization is enforced.
- [ ] Object-, function-, and property-level permissions are verified.
- [ ] Tenant scope is validated and enforced.
- [ ] Privilege escalation paths are prevented.
- [ ] Session and token handling follows the authentication policy.

### Data and Secrets
- [ ] Secrets are stored and accessed securely.
- [ ] Sensitive data is protected in transit and at rest as required.
- [ ] Logs and errors do not expose confidential information.
- [ ] Database queries are safe and appropriately scoped.
- [ ] Cache and search results preserve tenant isolation.

### Application and Infrastructure
- [ ] Secure defaults are enabled.
- [ ] Rate limits and resource constraints are configured.
- [ ] File uploads and remote resource retrieval are secured.
- [ ] External integrations are authenticated and validated.
- [ ] Background jobs and realtime events enforce security requirements.
- [ ] Dependencies and build artifacts are scanned.

### Testing and Delivery
- [ ] Positive and negative security tests are included.
- [ ] Relevant integration tests pass.
- [ ] Static analysis and security scans pass.
- [ ] Security-sensitive changes received appropriate review.
- [ ] Audit and monitoring requirements are addressed.
- [ ] Deployment and recovery behavior are documented.

## 45. Definition of Done

A feature is considered secure enough for release when:

- Its security requirements and trust boundaries have been identified.
- All untrusted inputs are validated and safely processed.
- Authentication and authorization are enforced server-side.
- Tenant isolation is preserved across every relevant data store and service.
- Sensitive information and secrets are protected.
- Injection, XSS, CSRF, SSRF, path traversal, and other applicable risks are addressed.
- Database operations preserve confidentiality and integrity.
- Concurrency-sensitive operations are safe against race conditions and duplicate execution.
- Errors and logs do not disclose sensitive information.
- Secure defaults and resource limits are applied.
- Relevant automated security tests pass.
- Static analysis, dependency scanning, and secret scanning have been completed.
- Security-sensitive changes have been reviewed.
- Required monitoring, audit events, and operational documentation are in place.
- Any accepted security exception has documented approval, ownership, and an expiration date.

**Final rule:** Every component must treat external input as untrusted, enforce authorization at the point of use, protect sensitive information throughout its lifecycle, and fail securely. Security must be an inherent property of KAMPYN's architecture and implementation, not a feature added after development.
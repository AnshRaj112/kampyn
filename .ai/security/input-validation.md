# Input Validation

## 1. Purpose

This document defines the mandatory input validation and sanitization standards for KAMPYN across all applications, APIs, services, databases, integrations, and user-generated content.

KAMPYN processes data from students, faculty, vendors, university administrators, external integrations, SDKs, background jobs, and third-party systems. All externally supplied data MUST be treated as untrusted, regardless of its source or apparent structure.

The objective is to prevent vulnerabilities caused by malformed, unexpected, malicious, or unauthorized input while maintaining data integrity, predictable application behavior, and a consistent user experience.

Input validation is a security boundary, not merely a form-handling or user-experience feature.

## 2. Core Principles

All input validation MUST follow these principles:

- **Trust nothing by default:** Treat all external and user-controlled data as untrusted.
- **Validate at boundaries:** Validate input whenever it crosses into a trusted application component.
- **Use allowlists:** Define explicitly what values, formats, structures, and operations are permitted.
- **Validate types and semantics:** Validate both the structure and the business meaning of data.
- **Use strict schemas:** Prefer reusable, explicit schemas over scattered conditional checks.
- **Reject invalid input:** Fail securely when input does not meet the expected contract.
- **Normalize consistently:** Apply well-defined normalization before validation when appropriate.
- **Preserve data integrity:** Do not silently modify meaningful user input or discard invalid data without an explicit policy.
- **Separate validation from authorization:** Valid input does not mean the caller is permitted to perform the requested action.
- **Prevent injection:** Use parameterized queries, context-aware output encoding, and safe APIs.
- **Bound resource consumption:** Enforce limits on input size, nesting, complexity, and processing time.
- **Validate again at trust boundaries:** Internal services MUST NOT assume upstream validation is sufficient.

Validation MUST be deterministic, testable, and consistently applied across the system.

## 3. Scope

This policy applies to all data entering or moving through KAMPYN, including:

- HTTP request bodies, query parameters, path parameters, and headers.
- WebSocket messages and realtime events.
- Form submissions and user-generated content.
- Authentication and authorization inputs.
- API requests from external SDKs and self-hosted installations.
- Third-party webhook payloads and integration responses.
- File uploads, archives, images, documents, and imported datasets.
- Search queries, filters, sorting parameters, and pagination values.
- Database writes and inter-service communication.
- Queue messages, scheduled jobs, and event payloads.
- Environment variables and deployment configuration.
- Administrative configuration and tenant-defined settings.
- Data received from external systems and university integrations.

Input validation MUST be applied in frontend and backend systems where appropriate, with the backend remaining authoritative for security and business validation.

## 4. Validation Architecture

### 4.1 Validation Boundaries

Input MUST be validated at every relevant trust boundary.

| Boundary | Validation responsibility |
|---|---|
| Browser forms | User feedback, basic type and format validation |
| Frontend API client | Request shape and serialization consistency |
| API gateway / HTTP middleware | Request size, content type, protocol-level constraints |
| Backend handler | Request schema and endpoint contract validation |
| Application service | Business rules and domain invariants |
| Repository / persistence layer | Persistence constraints and data integrity |
| Inter-service API | Message schema, identity, and contract validation |
| Event consumer | Payload structure, version, and semantic validation |
| WebSocket gateway | Message schema, size, identity, and rate limits |
| File-processing service | File type, structure, size, and parser safety |
| Third-party integration | Response schema, integrity, and expected semantics |

Validation MUST occur before untrusted data is used in sensitive operations, database queries, filesystem operations, or downstream service calls.

### 4.2 Backend Authority

Frontend validation exists primarily to improve usability and catch mistakes early.

The backend MUST independently validate all security-relevant input, including:

- Required fields.
- Types and formats.
- Allowed values.
- Numeric and string bounds.
- Relationships between fields.
- Business invariants.
- Tenant and resource context.
- State transitions.
- File constraints.
- Sensitive operation parameters.

Frontend validation MUST NOT be treated as proof that data is safe or authorized.

### 4.3 Validation Layers

KAMPYN MUST separate validation into clear layers:

1. **Transport validation:** Protocol, content type, request size, encoding, and basic structure.
2. **Schema validation:** Required fields, types, formats, and permitted structure.
3. **Semantic validation:** Relationships between fields and valid domain values.
4. **Business validation:** Domain rules, state transitions, availability, and operational constraints.
5. **Authorization:** Whether the authenticated principal may perform the operation on the relevant resource.
6. **Persistence integrity:** Database constraints, uniqueness, referential integrity, and concurrency guarantees.

These layers MAY share schema definitions and reusable helpers, but MUST NOT be collapsed into a single generic validation function that obscures their distinct responsibilities.

## 5. Schema-Based Validation

### 5.1 General Requirements

All externally exposed APIs MUST define explicit input schemas.

Schemas MUST:

- Specify accepted fields and their types.
- Define required and optional properties.
- Enforce length, size, and numeric bounds.
- Constrain enumerations and structured values.
- Define nested object and array structure.
- Reject or explicitly handle unknown fields.
- Define safe defaults where applicable.
- Support versioning where contracts evolve.
- Be reusable across appropriate application boundaries.
- Produce structured, consistent validation errors.

Validation logic MUST NOT be duplicated unnecessarily across handlers and services.

### 5.2 Zod

For TypeScript applications and services, Zod SHOULD be used for runtime validation of untrusted data.

Zod schemas SHOULD be the source for inferred TypeScript types where suitable.

Example:

```ts
import { z } from "zod";

export const CreateOrderSchema = z.object({
  foodCourtId: z.string().uuid(),
  items: z
    .array(
      z.object({
        itemId: z.string().uuid(),
        quantity: z.number().int().min(1).max(50),
      }).strict(),
    )
    .min(1)
    .max(50),
  notes: z.string().trim().max(500).optional(),
}).strict();

export type CreateOrderInput = z.infer<
  typeof CreateOrderSchema
>;
```

The schema validates the shape and basic constraints of a request. The application service MUST still verify item availability, pricing, ownership, tenant scope, order rules, and other domain requirements.

### 5.3 Go

For Go services:

- Use explicit request structures with well-defined JSON tags.
- Validate decoded structures with an approved validation library or cohesive domain validation functions.
- Enforce strict decoding where practical.
- Reject unexpected fields for contracts that require strict compatibility.
- Validate semantic rules separately from decoding.
- Avoid relying on zero values to distinguish omitted fields from explicitly supplied values when the distinction matters.
- Use dedicated input types rather than binding external requests directly to persistence models.

Example:

```go
type CreateOrderItemInput struct {
    ItemID   string `json:"itemId"`
    Quantity int    `json:"quantity"`
}

type CreateOrderInput struct {
    FoodCourtID string                 `json:"foodCourtId"`
    Items       []CreateOrderItemInput `json:"items"`
    Notes       *string                `json:"notes,omitempty"`
}
```

Decoded values MUST undergo explicit format, bounds, and business validation before use.

### 5.4 Schema Ownership

Schemas SHOULD be colocated with the API contract, feature, or domain that owns them.

Shared schemas MUST have a clear owner and change-management process.

Do not create a global schema library containing unrelated domain rules simply to avoid local organization.

API request schemas, domain entities, and database models MUST remain conceptually distinct, even where some fields overlap.

## 6. Type, Format, and Constraint Validation

### 6.1 Strings

Every externally supplied string MUST have an explicit policy for:

- Minimum and maximum length.
- Empty and whitespace-only values.
- Unicode handling.
- Normalization, if applicable.
- Permitted character sets, where required.
- Control characters and prohibited sequences.
- Storage and rendering behavior.

String length MUST be measured consistently with the application's contract. Where byte limits and character limits differ, enforce the relevant limits explicitly.

Do not apply broad character restrictions to natural-language content without a clear business or security requirement.

### 6.2 Numbers

Numeric inputs MUST be checked for:

- Correct numeric type.
- Minimum and maximum values.
- Integer requirements, where applicable.
- Precision and scale.
- Overflow and underflow.
- Non-finite values, such as `NaN` or infinity, where the runtime permits them.
- Unit consistency.

Money MUST use an appropriate exact representation, such as integer minor units or a decimal type, rather than binary floating-point arithmetic.

Never trust client-supplied prices, discounts, tax amounts, payment totals, or account balances as authoritative.

### 6.3 Booleans

Boolean fields MUST accept only the documented boolean representation.

Do not treat arbitrary strings, numbers, or truthy values as equivalent to a valid boolean unless the API contract explicitly defines that behavior.

### 6.4 Enumerations

Fields with a limited set of values MUST use explicit allowlists.

Examples include:

- Order status.
- Booking status.
- User role.
- Complaint priority.
- Inventory movement type.
- Notification category.
- Facility type.

Unknown enum values MUST be rejected or handled through a documented forward-compatibility strategy.

Do not accept arbitrary client-defined values for privileged state or workflow transitions.

### 6.5 Identifiers

Identifiers MUST be validated against their expected format.

Examples include:

- UUIDs.
- MongoDB ObjectIds.
- Tenant IDs.
- User IDs.
- Resource IDs.
- External provider references.

A syntactically valid identifier MUST NOT be assumed to refer to an existing, accessible, or authorized resource.

Identifier validation MUST be followed by the appropriate resource lookup and authorization checks.

### 6.6 Dates and Times

Date and time inputs MUST use documented formats and time-zone semantics.

- Prefer ISO 8601 representations for API timestamps.
- Distinguish dates from timestamps.
- Require explicit time-zone interpretation where ambiguity is possible.
- Reject invalid calendar dates and malformed timestamps.
- Enforce domain-specific time ranges.
- Avoid accepting ambiguous locale-specific date strings.
- Normalize timestamps consistently for storage and comparison.

Booking and scheduling systems MUST consider daylight-saving transitions where relevant, even if most current deployments operate in a single time zone.

The backend MUST independently verify availability, scheduling conflicts, booking windows, and permitted state transitions.

### 6.7 URLs and Network Addresses

URL inputs MUST be validated against explicit requirements.

Where a feature accepts a URL:

- Allow only required schemes, normally HTTPS for externally accessible destinations.
- Reject dangerous or unsupported schemes.
- Validate hostname and port restrictions where applicable.
- Prevent access to loopback, private, link-local, and metadata-service addresses when performing server-side requests.
- Apply redirect restrictions and revalidate redirect destinations.
- Defend against DNS rebinding and time-of-check/time-of-use issues in server-side fetching.
- Enforce network egress controls for services that fetch user-supplied URLs.

URL validation alone MUST NOT be considered sufficient protection against Server-Side Request Forgery (SSRF).

## 7. Unknown and Unexpected Fields

API schemas MUST define how unknown fields are handled.

For security-sensitive requests, unknown fields SHOULD be rejected.

Unknown fields MUST NOT be silently mapped onto privileged model properties or internal configuration.

Mass assignment MUST be prevented by mapping validated request fields explicitly to permitted domain operations.

Never bind an untrusted request directly to a database document or ORM entity when doing so could modify fields that the caller is not allowed to control.

Example of prohibited behavior:

```ts
// Do not directly persist arbitrary user-controlled properties.
await userRepository.update(userId, request.body);
```

Instead, construct an explicit update object from validated and authorized fields.

## 8. Normalization and Canonicalization

### 8.1 General Rules

Normalization MUST be explicit, consistent, and performed at the correct boundary.

Normalization MAY include:

- Trimming surrounding whitespace.
- Unicode normalization where appropriate.
- Case normalization for case-insensitive identifiers.
- Canonical formatting of phone numbers.
- Standardized date and time representation.
- Consistent representation of enumerated values.

Normalization MUST NOT silently change meaningful user content or weaken validation.

### 8.2 Validate After Normalization

When normalization changes the representation of input, validate the normalized result before use.

Security-sensitive comparisons MUST use canonical representations to prevent bypasses caused by inconsistent encoding or equivalent representations.

### 8.3 Unicode

KAMPYN supports multilingual user content. Validation MUST account for Unicode safely.

- Avoid assuming that one visible character equals one code unit or byte.
- Define length semantics for multilingual content.
- Normalize identifiers where the domain requires canonical equivalence.
- Reject prohibited control characters in fields that do not support them.
- Preserve valid multilingual text in names, comments, complaints, chat messages, and other natural-language fields.
- Avoid simplistic ASCII-only validation unless the field's purpose requires it.

Unicode normalization MUST be applied carefully to avoid changing user intent or causing identifier collisions.

## 9. Injection Prevention

Validation is one layer of injection defense. It MUST be combined with safe data handling and context-specific output encoding.

### 9.1 SQL Injection

- Use parameterized queries or safe ORM query APIs.
- Never concatenate untrusted values into SQL statements.
- Allowlist dynamic identifiers such as table names, column names, and sort directions.
- Avoid raw SQL unless justified and safely parameterized.
- Review query-builder escape hatches and raw expressions.

Input validation MUST NOT be used as a substitute for parameterized SQL.

### 9.2 NoSQL Injection

- Validate document structure and permitted fields.
- Reject unexpected query operators in user-controlled objects.
- Never pass arbitrary client objects directly into MongoDB filters or update operators.
- Construct database filters explicitly.
- Avoid unsafe use of dynamic query expressions.
- Validate aggregation pipeline construction and user-defined query features.

Untrusted input MUST NOT be interpreted as a database operator or executable query structure.

### 9.3 Command Injection

- Avoid shell execution for application workflows.
- Prefer safe process APIs that pass arguments separately.
- Never concatenate untrusted strings into shell commands.
- Use strict allowlists for executable names and permitted arguments.
- Restrict process permissions and execution environments.

### 9.4 Cross-Site Scripting (XSS)

- Render untrusted content as text by default.
- Use framework-provided escaping and safe rendering APIs.
- Avoid unsafe HTML injection.
- Sanitize rich HTML with a maintained, allowlist-based sanitizer when HTML rendering is a required feature.
- Define permitted HTML elements, attributes, and URL schemes.
- Validate rich-content output after sanitization.
- Apply a restrictive Content Security Policy as defense-in-depth.

Input validation MUST NOT be treated as a replacement for context-aware output encoding.

### 9.5 Path Traversal

- Do not use raw user input as a filesystem path.
- Resolve files through server-generated identifiers or controlled storage keys.
- Enforce permitted directory boundaries.
- Reject traversal sequences and unexpected path separators where paths are accepted.
- Verify the canonical resolved path before file access.
- Avoid unsafe assumptions about URL decoding and filesystem normalization.

### 9.6 LDAP, Template, and Other Injection

Where KAMPYN integrates with LDAP, template engines, expression evaluators, or other interpreters:

- Use safe parameterized or structured APIs.
- Validate allowed input types and values.
- Avoid evaluating untrusted expressions.
- Apply context-specific escaping where applicable.
- Review any user-defined filtering, formulas, or templates as potentially executable input.

## 10. Request-Level Validation

### 10.1 HTTP Request Validation

Every API endpoint MUST define accepted:

- HTTP method.
- Content type.
- Request body format.
- Query parameters.
- Path parameters.
- Relevant headers.
- Request size limits.
- Timeout and processing constraints.

Reject unsupported content types and malformed payloads with consistent errors.

Do not assume that the `Content-Type` header accurately describes the submitted data; parsing MUST still be safe and bounded.

### 10.2 Query Parameters

Query parameters MUST be validated for:

- Type and format.
- Maximum length.
- Allowed values.
- Number of repeated parameters.
- Maximum pagination limits.
- Search complexity.
- Permitted filter and sort fields.

Dynamic sorting and filtering MUST use explicit allowlists.

Clients MUST NOT be allowed to select arbitrary database fields, internal properties, aggregation expressions, or query operators.

### 10.3 Headers

Validate security-relevant headers, including:

- Authorization.
- Content-Type.
- Content-Length, where relevant.
- Tenant-selection headers, if supported.
- Correlation and request identifiers.
- Idempotency keys.
- Forwarded headers, when accepted from trusted proxies.

Client-supplied tenant identifiers MUST be checked against the authenticated principal's authorized tenant context.

Forwarded headers MUST only be trusted from configured proxy infrastructure.

### 10.4 Request Size and Complexity

Every externally accessible endpoint MUST enforce bounded request sizes and computational complexity.

Controls SHOULD include:

- Maximum body size.
- Maximum string and array lengths.
- Maximum object nesting depth.
- Maximum number of object properties.
- Maximum number of query parameters.
- Maximum filter and sort complexity.
- Parsing and execution timeouts.
- Rate limits for expensive operations.

Limits MUST reflect the endpoint's legitimate use cases and be configurable where deployment requirements differ.

## 11. Domain-Specific Validation

### 11.1 Authentication and Account Management

Authentication-related input MUST be validated before processing.

Examples include:

- Email addresses.
- Phone numbers.
- Passwords.
- One-time codes.
- Password reset tokens.
- OAuth callback parameters.
- Session identifiers.
- Account recovery fields.

Requirements:

- Apply documented length and format constraints.
- Avoid unnecessarily restrictive password character rules.
- Do not trim or normalize passwords unless the authentication contract explicitly defines it.
- Avoid exposing whether an account exists through validation errors.
- Enforce expiration and attempt limits for verification and recovery flows.
- Treat tokens as sensitive values and never log them.
- Validate authentication data independently of user-profile update schemas.

### 11.2 Food Ordering

Order input MUST be checked for:

- Valid food court and item identifiers.
- Valid item quantities and order size.
- Permitted customizations and add-ons.
- Valid delivery or pickup options.
- Notes and special instructions length.
- Supported payment methods.
- Valid order state transitions.

The backend MUST independently resolve current item availability, authoritative prices, discounts, taxes, and totals.

Clients MUST NOT be able to set privileged order fields such as payment status, seller approval, fulfillment state, or administrative adjustments.

### 11.3 Bookings and Scheduling

Booking requests MUST validate:

- Resource identifiers.
- Start and end times.
- Time-zone interpretation.
- Permitted booking duration.
- Booking windows.
- Party size or capacity requirements.
- User eligibility.
- Allowed booking status transitions.

The service MUST recheck availability and enforce concurrency-safe booking rules during the operation.

A successful schema validation MUST NOT be interpreted as confirmation that a resource is available.

### 11.4 Inventory

Inventory input MUST validate:

- Item identifiers.
- Quantity and unit.
- Movement type.
- Location or storage identifier.
- Supplier references.
- Adjustment reasons.
- Reorder thresholds.
- Supported measurement precision.

Inventory changes MUST be checked against authorization, business rules, and concurrency constraints.

Client-provided stock balances MUST NOT directly overwrite authoritative inventory state unless the operation is an explicitly authorized reconciliation workflow.

### 11.5 Complaints and Reports

Complaint and report submissions MUST validate:

- Category.
- Description length.
- Related resource identifiers.
- Attachment count and type.
- Priority values, where user-selectable.
- Supported status transitions.

Users MUST NOT be allowed to assign privileged resolution states, internal moderation notes, or administrative ownership fields unless authorized.

### 11.6 Community and Chat

User-generated community and chat content MUST validate:

- Message size.
- Supported content type.
- Attachment metadata.
- Permitted message actions.
- Reaction and mention structures.
- Room, thread, and recipient identifiers.
- Supported edit and deletion operations.

Apply rate limits and abuse protections to reduce spam, automated flooding, and resource exhaustion.

Rich text or embedded content MUST be sanitized and rendered safely. Validation MUST NOT be used to silently rewrite the semantic meaning of messages.

### 11.7 HR and Administrative Workflows

Administrative and HR inputs MUST validate:

- Employee and department identifiers.
- Role and permission references.
- Employment-related dates and states.
- Workflow transitions.
- Supported document types.
- Allowed administrative actions.

Sensitive fields MUST be protected through authorization and audit controls in addition to validation.

### 11.8 Search and Filtering

Search input MUST enforce:

- Maximum query length.
- Permitted search operators.
- Supported filters.
- Allowed sort fields and directions.
- Pagination bounds.
- Query complexity limits.
- Tenant and authorization scope.

Never expose raw OpenSearch DSL, database expressions, or internal query syntax directly to untrusted clients unless an explicitly designed and safely constrained query interface exists.

## 12. File Upload Validation

File uploads MUST be treated as untrusted binary input.

### 12.1 File Acceptance

For every upload workflow, define:

- Permitted file extensions.
- Expected media types.
- Maximum file size.
- Maximum file count.
- Maximum decompressed size, if archives are supported.
- Maximum archive nesting depth.
- Required file structure.
- Permitted use and storage duration.

Do not trust file names, extensions, or client-supplied MIME types as proof of file content.

### 12.2 Content Verification

Where appropriate:

- Inspect file signatures or magic bytes.
- Parse content using maintained libraries.
- Validate expected document or media structure.
- Reject malformed or unsupported content.
- Detect dangerous embedded content where supported.
- Apply antivirus or malware scanning according to risk and operational requirements.

File parsers MUST run with restricted permissions and resource limits.

### 12.3 Storage and Processing

- Generate server-controlled storage names.
- Store uploaded files outside executable application paths.
- Enforce tenant and user access controls.
- Prevent uploaded content from being executed.
- Use safe download headers and content disposition.
- Apply retention and deletion policies.
- Avoid exposing internal filesystem paths.
- Ensure asynchronous processing revalidates stored file metadata and access context.

### 12.4 Archive Handling

ZIP, TAR, and other archive formats MUST be protected against:

- Path traversal.
- Archive bombs.
- Excessive file counts.
- Excessive nesting.
- Oversized decompressed content.
- Malformed headers.
- Unexpected symbolic links or special files.
- Parser vulnerabilities.

Archive extraction MUST use controlled destinations and enforce decompression limits before and during processing.

## 13. Webhooks and External Integrations

External integration payloads MUST be treated as untrusted, even when they originate from known providers.

KAMPYN MUST:

- Verify signatures or other provider-defined authenticity mechanisms.
- Validate timestamps and replay-protection fields where supported.
- Enforce payload size limits.
- Validate payload schemas and supported event types.
- Reject malformed or unsupported critical fields.
- Map external data to internal domain types explicitly.
- Avoid trusting external identifiers as internal authorization evidence.
- Handle duplicate and out-of-order events safely.
- Apply idempotency and replay protection where appropriate.
- Log validation failures without exposing secrets or sensitive payload content.

Third-party responses MUST also be validated before being used in application logic, persisted, or forwarded to other services.

## 14. Events, Queues, and Background Jobs

Messages received through queues, event buses, scheduled jobs, or internal event channels MUST be validated by their consumers.

Consumers MUST NOT assume that an event is valid merely because it was produced by another KAMPYN service.

Each event contract SHOULD define:

- Event name and version.
- Producer identity.
- Required fields.
- Field types and constraints.
- Tenant context, where applicable.
- Correlation and causation identifiers.
- Timestamp semantics.
- Idempotency requirements.
- Compatibility expectations.

Consumers MUST reject or quarantine malformed messages according to the operational policy, avoiding infinite retry loops and poison-message processing.

Background jobs MUST revalidate authorization-sensitive and state-dependent assumptions at execution time where the state may have changed since scheduling.

## 15. Database Integrity

Input validation MUST be complemented by persistence-layer integrity constraints.

Use appropriate database mechanisms, including:

- `NOT NULL` constraints.
- Unique constraints.
- Foreign keys.
- Check constraints.
- Enum or constrained-value representations where appropriate.
- Schema validation for document stores.
- Tenant-scoped uniqueness rules.
- Transactional enforcement of domain invariants.

Application validation provides understandable errors; database constraints protect integrity under concurrency, bugs, alternate write paths, and race conditions.

Do not assume that a prior validation check guarantees that a subsequent write remains valid.

Validation and persistence MUST be designed to handle concurrent requests and state changes safely.

## 16. Error Handling

### 16.1 Consistent Validation Errors

Validation failures MUST produce structured and predictable error responses.

An API error SHOULD include:

- Stable error code.
- Human-readable message.
- Field-level details where safe.
- Request or correlation identifier.
- Appropriate HTTP status code.

Example:

```json
{
  "error": {
    "code": "VALIDATION_FAILED",
    "message": "The request contains invalid fields.",
    "details": [
      {
        "field": "items[0].quantity",
        "code": "OUT_OF_RANGE",
        "message": "Quantity must be between 1 and 50."
      }
    ],
    "requestId": "req_example"
  }
}
```

The example is illustrative; actual identifiers and fields MUST be generated according to the API contract.

### 16.2 Error Disclosure

Validation errors MUST NOT reveal:

- Database schema details.
- Internal stack traces.
- Filesystem paths.
- Secrets or tokens.
- Internal service addresses.
- Sensitive account or tenant information.
- Implementation-specific parser details that enable exploitation.

Errors SHOULD provide enough detail for legitimate users and developers to correct requests without disclosing unnecessary internals.

### 16.3 Logging

Validation failures SHOULD be logged with relevant metadata, such as:

- Endpoint or event type.
- Error code.
- Request identifier.
- Tenant context, where safe.
- Authenticated principal identifier, where appropriate.
- Source service.
- Timestamp.
- Repeated-failure indicators.

Logs MUST NOT contain passwords, authentication tokens, private keys, payment credentials, or unnecessary sensitive user content.

Avoid logging full malformed payloads by default. If payload capture is operationally required, use explicit access controls, redaction, retention limits, and a documented justification.

## 17. Validation and Authorization Separation

Validation answers whether input conforms to a permitted structure and domain contract.

Authorization answers whether the principal may perform the requested operation on the target resource.

These responsibilities MUST remain distinct.

For example:

- A valid `foodCourtId` does not prove that a vendor can manage that food court.
- A valid `bookingId` does not prove that the user owns or can view the booking.
- A valid `role` value does not permit a user to assign that role.
- A valid `tenantId` does not grant access to the tenant.
- A valid inventory adjustment does not mean the caller is authorized to apply it.

Authorization MUST be enforced server-side according to `.ai/security/authorization.md`.

Validation MUST NOT be used to conceal missing authorization checks.

## 18. Rate Limits and Resource Protection

Input validation MUST consider the resource cost of processing valid input.

Endpoints and message handlers MUST apply appropriate limits to:

- Request frequency.
- Payload size.
- Array and collection sizes.
- Search complexity.
- File parsing.
- Archive decompression.
- Regex evaluation.
- Recursive or deeply nested structures.
- Expensive business operations.
- Bulk updates and imports.
- Realtime message rates.

Use bounded algorithms and safe parsers. Avoid catastrophic-backtracking regular expressions and unbounded recursive processing.

Where limits are exceeded, reject requests early and return a suitable error response.

Rate limits MUST be designed with tenant isolation and fair resource allocation in mind.

## 19. Frontend Validation

Frontend validation MUST improve usability without replacing backend enforcement.

Frontend applications SHOULD:

- Reuse appropriate shared schemas where feasible.
- Validate user input before submission.
- Show accessible field-level errors.
- Preserve user input when safe after validation failures.
- Prevent invalid form submissions where practical.
- Reflect server-side validation errors accurately.
- Avoid exposing internal implementation details.
- Support keyboard navigation and screen readers.
- Avoid expensive validation on every keystroke when it affects responsiveness.

Frontend validation MUST NOT be considered a security boundary because requests can be modified or sent outside the application.

### 19.1 State and Query Parameters

Data in URL parameters, browser storage, Zustand stores, TanStack Query caches, and client-side navigation state MUST be treated as untrusted when used as input to backend operations.

Validate values again before they influence security-sensitive actions or server requests.

## 20. API and SDK Contract Validation

All public and internal APIs MUST have explicit request and response contracts.

- Define required, optional, nullable, and defaulted fields clearly.
- Version contracts where incompatible changes may occur.
- Validate incoming requests at the receiving boundary.
- Validate critical responses from external or independently deployed services.
- Ensure SDK types match the documented API contract.
- Avoid assuming that compile-time types guarantee runtime correctness.
- Use contract tests to detect schema drift.
- Ensure validation errors remain stable enough for supported SDK consumers.

SDKs MAY provide client-side validation for convenience, but backend services MUST independently validate every request.

## 21. Multi-Tenant Validation

KAMPYN MUST enforce tenant-aware validation wherever tenant context affects data, configuration, or operations.

- Derive tenant context from trusted authentication or verified routing context.
- Validate that resource identifiers belong to the active tenant.
- Reject conflicting tenant identifiers.
- Validate tenant-specific configuration against platform constraints.
- Enforce tenant-specific upload, request, and resource limits.
- Validate cross-service tenant context and event metadata.
- Avoid accepting arbitrary tenant scope from client-controlled fields.
- Ensure validation failures do not disclose the existence of resources in other tenants.

Tenant identity and resource ownership MUST be checked independently from the syntactic validity of input.

## 22. Configuration Validation

Environment variables, deployment configuration, feature flags, and tenant-defined settings MUST be validated at startup or before activation.

Configuration validation SHOULD include:

- Required variables.
- Types and formats.
- Valid ranges.
- Supported enum values.
- URL and hostname restrictions.
- Cross-field consistency.
- Secret presence without exposing secret values.
- Compatibility with the active deployment mode.
- Resource bounds and safe defaults.

Invalid security-critical configuration MUST prevent the affected service from starting or activating the unsafe feature.

Configuration MUST NOT silently fall back to insecure defaults.

## 23. Validation Testing

Validation logic MUST have automated tests covering normal, boundary, and malicious inputs.

### 23.1 Schema Tests

Test:

- Valid minimal requests.
- Valid complete requests.
- Missing required fields.
- Unexpected fields.
- Incorrect types.
- Empty and whitespace-only strings.
- Minimum and maximum lengths.
- Values just below and above limits.
- Invalid enum values.
- Malformed identifiers.
- Invalid dates and timestamps.
- Unicode and encoding edge cases.
- Null versus omitted fields.
- Excessively nested structures.
- Oversized arrays and objects.

### 23.2 Security Tests

Test relevant attack patterns, including:

- SQL and NoSQL injection attempts.
- Command and path injection.
- XSS payloads.
- Malformed JSON and parser edge cases.
- Mass assignment.
- Unexpected object properties.
- Invalid tenant and resource identifiers.
- Forged workflow states.
- Malformed webhook signatures and replay attempts.
- Oversized requests and resource-exhaustion attempts.
- Archive traversal and decompression abuse.
- Invalid or unexpected event payloads.

Security tests MUST verify that invalid input is rejected safely and does not cause unintended state changes.

### 23.3 Integration Tests

Integration tests SHOULD verify:

- Validation middleware and endpoint contracts.
- Domain-level business validation.
- Persistence constraints.
- Transaction and concurrency behavior.
- Authorization after schema validation.
- External integration payload handling.
- Queue and event consumer validation.
- Consistent error formatting.
- Tenant isolation.
- Observability and safe logging.

### 23.4 Fuzz Testing

Fuzz testing SHOULD be used for high-risk parsers, serializers, protocol handlers, file processors, and complex validation logic.

Fuzz targets SHOULD include malformed, truncated, deeply nested, and adversarial inputs.

Any discovered crash, panic, unbounded resource consumption, or validation bypass MUST be treated as a defect and remediated.

## 24. Observability and Continuous Improvement

Validation behavior MUST be observable without unnecessarily retaining sensitive input.

Track relevant metrics, including:

- Validation failure rate by endpoint and error code.
- Repeated invalid request patterns.
- Payload-size rejection counts.
- Parser failures.
- File validation failures.
- Invalid event and webhook counts.
- Rate-limit rejections.
- Validation processing latency.
- Validation-related application errors.

Alerts SHOULD identify abnormal spikes that may indicate abuse, integration failures, client defects, or attempted exploitation.

Monitoring MUST avoid exposing sensitive user data and MUST apply appropriate tenant and access controls.

## 25. Prohibited Practices

The following practices are prohibited:

- Trusting frontend validation as the only validation layer.
- Using TypeScript types as a substitute for runtime validation.
- Accepting arbitrary object structures in sensitive endpoints.
- Passing untrusted objects directly into database filters or update operations.
- Constructing SQL, shell commands, or query expressions through unsafe string concatenation.
- Trusting client-supplied prices, permissions, roles, or workflow states.
- Treating syntactically valid identifiers as proof of authorization.
- Using client-provided MIME types or extensions as the only file checks.
- Silently coercing malformed input into privileged or security-sensitive values.
- Using unrestricted regular expressions or unbounded parsers on untrusted data.
- Returning stack traces or internal errors to clients.
- Logging credentials, tokens, or sensitive payloads.
- Disabling validation to work around integration defects.
- Relying solely on validation to prevent injection attacks.
- Allowing untrusted input to select arbitrary database fields, sort expressions, or query operators.
- Allowing invalid configuration to fall back to insecure behavior.

## 26. Review Checklist

Before merging a feature that accepts or processes input, reviewers MUST verify:

- [ ] Every trust boundary has an explicit validation strategy.
- [ ] Request schemas define accepted types, fields, and constraints.
- [ ] Unknown fields are rejected or handled by a documented policy.
- [ ] Type, format, length, and range checks are implemented.
- [ ] Semantic and business rules are validated.
- [ ] Validation and authorization remain separate.
- [ ] Tenant and resource relationships are verified.
- [ ] Database queries use safe parameterization or structured APIs.
- [ ] Untrusted values cannot trigger injection or unsafe execution.
- [ ] File and archive inputs are safely bounded and parsed.
- [ ] External integration responses and events are validated.
- [ ] Resource-consumption limits are enforced.
- [ ] Errors are structured and do not expose sensitive internals.
- [ ] Logs are useful, access-controlled, and appropriately redacted.
- [ ] Frontend validation is backed by server-side validation.
- [ ] Tests cover valid, invalid, boundary, and adversarial inputs.
- [ ] API and SDK contracts remain consistent.
- [ ] Relevant documentation is updated.

## 27. Definition of Done

Input validation is complete only when:

- Every relevant input boundary has been identified.
- Explicit schemas or validation rules are implemented.
- Structural, semantic, and business validation are handled at the appropriate layers.
- Authorization and tenant-isolation checks remain independently enforced.
- Injection prevention uses safe APIs and context-aware output handling.
- Payload size, complexity, and processing limits are enforced.
- File, event, webhook, and external integration inputs are safely processed.
- Database constraints protect persistent data integrity.
- Errors are predictable and do not expose sensitive implementation details.
- Validation behavior is covered by automated tests.
- Security-relevant monitoring is in place where appropriate.
- API and SDK contracts are documented and consistent.
- No known validation bypass or unsafe fallback remains unresolved.

## 28. Final Rule

Every value originating outside a trusted component MUST be considered untrusted until it has been validated for its intended use.

Validation MUST be explicit, bounded, consistent, and enforced at the appropriate trust boundary. Valid input MUST still pass independent authorization, business-rule, and persistence-integrity checks before it can affect KAMPYN state.
# KAMPYN Backend Validation

## Purpose

Validation protects KAMPYN from invalid, malformed, unsafe, inconsistent, or unauthorized input.

Validation is not one operation performed in one layer.

KAMPYN uses layered validation:

```text id="v8m2q4"
External Input
      ↓
Transport Validation
      ↓
Application Validation
      ↓
Domain Invariants
      ↓
Persistence Constraints
      ↓
External Provider Validation
```

Each layer validates what it owns.

The goal is not to validate the same thing everywhere.

The goal is to ensure that no untrusted or invalid state crosses a boundary without appropriate protection.

---

# 1. Core Principles

Validation must be:

- Explicit
- Layered
- Deterministic
- Type-safe
- Runtime-enforced
- Security-conscious
- Reusable
- Close to the boundary it protects
- Consistent across services
- Observable when failures matter

Validation must not:

- Replace authorization.
- Replace business logic.
- Replace database constraints.
- Trust TypeScript types as runtime validation.
- Trust client-side validation.
- Be duplicated unnecessarily.
- Become hidden inside unrelated utility functions.

---

# 2. Validation Layers

KAMPYN should use the following validation model:

```text id="c4r7x2"
┌──────────────────────────────┐
│ External Request             │
└──────────────┬───────────────┘
               ↓
┌──────────────────────────────┐
│ Transport Validation         │
│ Syntax / Shape / Encoding    │
└──────────────┬───────────────┘
               ↓
┌──────────────────────────────┐
│ Application Validation       │
│ Use-case Preconditions       │
└──────────────┬───────────────┘
               ↓
┌──────────────────────────────┐
│ Domain Validation            │
│ Business Invariants          │
└──────────────┬───────────────┘
               ↓
┌──────────────────────────────┐
│ Persistence Constraints      │
│ DB Integrity                 │
└──────────────┬───────────────┘
               ↓
┌──────────────────────────────┐
│ External Provider Boundary   │
│ Provider Contract            │
└──────────────────────────────┘
```

---

# 3. Transport Validation

Transport validation protects the application from malformed external input.

Examples:

```text id="n5x8q3"
Malformed JSON
Missing required field
Invalid UUID
Invalid enum
Invalid query parameter
Invalid pagination value
Invalid content type
Oversized request
Invalid file metadata
```

Transport validation should happen before application logic executes.

---

# 4. Application Validation

Application validation verifies that the request can meaningfully execute a use case.

Examples:

```text id="r7m3k9"
Required resource exists
Tenant is active
Requested operation is supported
Required workflow state exists
Referenced entities belong to the same tenant
Requested quantity is allowed
```

Application validation may require repositories or other application dependencies.

---

# 5. Domain Validation

Domain validation protects business invariants.

Examples:

```text id="q6x2m8"
Order cannot transition from Completed → Pending
Inventory quantity cannot become negative
Booking cannot overlap
Refund cannot exceed captured amount
A cancelled resource cannot be completed
```

These rules must remain enforceable regardless of how the domain is accessed.

---

# 6. Persistence Constraints

The database provides the final integrity boundary.

Examples:

```text id="m8r4x2"
NOT NULL
UNIQUE
FOREIGN KEY
CHECK
PRIMARY KEY
UNIQUE COMPOSITE KEY
```

Application validation should provide good user-facing errors, but database constraints remain necessary for concurrency and integrity.

---

# 7. External Provider Validation

External responses are untrusted too.

Do not assume a provider response matches its documented schema.

Validate important external data before using it.

Example:

```text id="x3q7m1"
Payment Provider
      ↓
External Response
      ↓
Runtime Validation
      ↓
Integration DTO
      ↓
Application
```

---

# 8. Client Validation Is Not Security

Frontend validation improves UX.

It does not protect the backend.

Never assume:

```text id="k5n2v8"
React validation
=
Backend validation
```

A malicious client can bypass all browser-side validation.

The backend must independently validate every externally controlled value.

---

# 9. Type Checking Is Not Runtime Validation

This is not validation:

```ts id="w7m3q2"
const request = input as CreateOrderRequest;
```

TypeScript types disappear at runtime.

External data must be parsed or validated.

---

# 10. TypeScript Runtime Validation

KAMPYN uses Zod on the TypeScript side.

Zod should be used at appropriate runtime trust boundaries.

Example:

```ts id="x8r4m1"
const result = CreateOrderSchema.safeParse(input);

if (!result.success) {
  // reject invalid input
}
```

Prefer parsing at the boundary rather than repeatedly validating the same trusted object internally.

---

# 11. Go Runtime Validation

Go's static type system does not eliminate runtime validation.

External values still require validation for:

```text id="p6m2x9"
HTTP requests
JSON
Webhooks
Environment variables
Files
Messages
Database records
External APIs
```

Use explicit validation mechanisms appropriate to the repository.

---

# 12. Validation Ownership

Each validation rule must have an owner.

Example:

```text id="g3x7m2"
"Is this a valid UUID?"
        ↓
Transport / Value Type

"Does this order belong to the tenant?"
        ↓
Application / Authorization

"Can this order transition?"
        ↓
Domain

"Can two rows have the same unique identifier?"
        ↓
Database
```

Do not place every rule inside one validator.

---

# 13. Validation vs Authorization

Validation asks:

```text id="q8m3x6"
"Is this input structurally and semantically valid?"
```

Authorization asks:

```text id="r5x7k2"
"Is this actor allowed to perform this operation?"
```

Example:

```text id="h4m8q1"
status = "CANCELLED"
```

may be a valid value.

Whether the current user is allowed to cancel the order is authorization/business policy.

Do not confuse the two.

---

# 14. Validation vs Business Rules

Validation may reject malformed or impossible input.

Business rules determine domain behavior.

Example:

```text id="v7r2m5"
quantity = -1
```

is invalid input.

```text id="c9x4k8"
quantity = 5
but only 2 units are available
```

is an application/domain business condition.

Both may reject the operation, but they belong to different layers.

---

# 15. Validation vs Database Constraints

Application validation:

```text id="z6m3q9"
Provides understandable errors.
```

Database constraints:

```text id="f8r2x5"
Guarantee integrity under concurrency.
```

Both are required where appropriate.

Do not remove a critical database constraint simply because the service validates the same condition first.

---

# 16. Validation Pipeline

A typical KAMPYN request should follow:

```text id="t4m8x2"
HTTP Request
    ↓
Content-Type Validation
    ↓
Request Size Validation
    ↓
Authentication
    ↓
Tenant Resolution
    ↓
Schema Validation
    ↓
Authorization
    ↓
Application Validation
    ↓
Domain Rules
    ↓
Persistence Constraints
```

The exact ordering may vary depending on the endpoint.

---

# 17. Validate Early

Reject obviously invalid input as early as practical.

Examples:

```text id="p7x3m8"
Malformed UUID
Invalid JSON
Unsupported content type
Invalid enum
Oversized request
Invalid pagination cursor
```

Do not perform expensive database or external API operations before rejecting clearly invalid input.

---

# 18. Do Not Validate Everything Early

Not every rule belongs at the transport boundary.

Avoid loading the entire database just to validate an HTTP request.

For example:

```text id="j5m8q2"
"Does this booking conflict?"
```

may require a transactional application/domain check.

Validation should occur at the layer that has the required knowledge.

---

# 19. Schema Validation

Request schemas should describe the external contract.

Example:

```text id="c8r4m7"
CreateOrderRequest
 ├── items
 ├── deliveryAddress
 └── paymentMethod
```

The schema should validate:

- Required fields.
- Types.
- Basic formats.
- Bounds.
- Allowed values.
- Structural relationships.

---

# 20. Required vs Optional Fields

Do not confuse:

```text id="n7x2m4"
optional
```

with:

```text id="v3k8q6"
nullable
```

These have different meanings.

For example:

```text id="a5r9m1"
field omitted
```

may mean:

```text id="x6q2k8"
"leave existing value unchanged"
```

while:

```text id="h3m7p4"
field = null
```

may mean:

```text id="w8r2c5"
"explicitly clear the value"
```

The API contract must define this behavior.

---

# 21. Unknown Fields

KAMPYN APIs should deliberately decide how unknown request fields are handled.

Strict schemas may reject unexpected fields.

Permissive schemas may ignore them.

The choice must be consistent with the API contract and security requirements.

Do not accidentally accept dangerous fields such as:

```text id="f4x7m2"
role
tenantId
isAdmin
permissions
ownerId
```

merely because the schema permits arbitrary objects.

---

# 22. Mass Assignment

Never blindly map client input into domain/database objects.

Bad:

```text id="q7m2x5"
Object.assign(user, request.body)
```

This may allow fields such as:

```text id="c3r8n1"
role
permissions
tenantId
verified
status
```

to be modified by clients.

Use explicit field mapping.

---

# 23. Allowlisted Fields

Prefer explicit input structures:

```go id="m8x3r7"
type UpdateProfileInput struct {
    DisplayName string
    Bio         string
}
```

rather than accepting arbitrary maps.

---

# 24. Strings

String validation should consider:

- Length.
- Encoding.
- Whitespace.
- Unicode.
- Normalization where necessary.
- Allowed characters where meaningful.
- Security context.

Do not impose arbitrary restrictions without a reason.

---

# 25. Trimming

Determine whether whitespace is meaningful before trimming.

For example:

```text id="r4m8x2"
displayName
```

may reasonably be normalized.

But:

```text id="q7x3m5"
password
```

should not automatically be trimmed because changing the value changes the credential.

---

# 26. Unicode

KAMPYN is a university platform and may contain multilingual data.

Validation must not assume ASCII unless the field explicitly requires it.

Consider:

```text id="k5m8r2"
Unicode names
Multilingual content
Emoji
International addresses
Non-English search terms
```

---

# 27. Unicode Normalization

Where identifiers or security-sensitive comparisons require canonical representation, normalization should be explicit.

Do not silently normalize fields where exact user-provided content must be preserved.

---

# 28. Email Validation

Email validation should verify reasonable structural correctness.

Do not attempt to fully reproduce every detail of email standards with an enormous regex.

For account workflows:

```text id="m2x7q4"
Syntax validation
        ↓
Verification email
        ↓
Verified identity
```

Email syntax alone does not prove ownership.

---

# 29. Password Validation

Password validation should enforce the configured security policy.

Do not:

- Log passwords.
- Store plaintext passwords.
- Return passwords in responses.
- Put passwords into job payloads.
- Send passwords through analytics.

Password hashing belongs to the authentication/security boundary.

---

# 30. UUID / Identifier Validation

Identifiers must be validated according to the actual ID format.

Do not accept arbitrary strings merely because a database driver might reject them later.

Prefer typed identifiers where practical.

---

# 31. Enum Validation

External enum values must be allowlisted.

Example:

```text id="p8r3x6"
PENDING
CONFIRMED
CANCELLED
```

Do not accept arbitrary strings and cast them into enum types.

Unknown enum values from external providers must be handled safely.

---

# 32. Boolean Validation

Do not treat arbitrary strings as booleans.

Avoid:

```text id="w5m2q8"
if value {
    ...
}
```

when `value` may originate from:

```text id="j4x7m1"
"false"
"0"
"no"
```

Use explicit parsing.

---

# 33. Numeric Validation

Numeric inputs should validate:

- Type.
- Range.
- Precision.
- Sign.
- Integer vs decimal requirements.

Examples:

```text id="x8r2m5"
quantity ≥ 0
percentage ∈ [0, 100]
pageSize ≤ configured maximum
```

---

# 34. Floating-Point Values

Do not use floating-point values for exact monetary amounts.

Validation should reject values that cannot be represented safely by the domain's money model.

---

# 35. Money Validation

Money input should validate:

```text id="g7m3x9"
currency
amount
precision
non-negative constraints where applicable
maximum permitted amount
```

The domain must still enforce financial rules.

---

# 36. Quantities

Inventory and booking quantities must be validated against:

```text id="q5x8m2"
minimum
maximum
integer/decimal requirements
domain constraints
```

Do not allow negative quantities unless explicitly meaningful.

---

# 37. Dates

Validate dates according to their semantic meaning.

Examples:

```text id="r8m3x6"
date
datetime
time
duration
date range
```

Do not silently interpret ambiguous date formats.

Prefer standardized representations at API boundaries.

---

# 38. Time Zones

Time-zone-sensitive input must preserve its intended timezone semantics.

Avoid silently assuming:

```text id="c4x7m2"
local machine timezone
server timezone
browser timezone
```

as the authoritative timezone.

---

# 39. Date Ranges

Validate:

```text id="n8m2r5"
start <= end
```

where applicable.

Also validate maximum range length when large ranges could cause expensive queries.

---

# 40. Pagination Validation

Pagination inputs must have explicit limits.

Example:

```text id="j3x8q7"
pageSize >= 1
pageSize <= MAX_PAGE_SIZE
```

Never allow an arbitrary client to request millions of records in one response.

---

# 41. Cursor Validation

Cursor values should be:

- Structurally valid.
- Correctly decoded.
- Version-aware where necessary.
- Bound to the expected pagination semantics.

Do not directly expose arbitrary database query fragments as cursors.

---

# 42. Sorting Validation

Sort fields must be allowlisted.

Example:

```text id="f7m2x8"
Allowed:
createdAt
name
price
rating
```

Do not accept:

```text id="q3r8m1"
sort=DROP TABLE...
```

or arbitrary database expressions.

---

# 43. Filtering Validation

Filters must be represented using explicit schemas.

Example:

```text id="m5x7q2"
status
category
minPrice
maxPrice
createdAfter
```

Each filter should have defined:

- Type.
- Range.
- Allowed values.
- Maximum complexity.

---

# 44. Search Validation

Search queries require validation against abuse and expensive execution.

Validate:

```text id="x8m3r5"
query length
number of filters
number of facets
page size
sort options
field selection
```

Do not allow clients to submit arbitrary OpenSearch queries.

---

# 45. Search Complexity

A request should have bounded complexity.

For example:

```text id="k7x2m4"
100 filters
+
20 aggregations
+
huge page size
```

should not be accepted merely because each individual field is syntactically valid.

---

# 46. File Validation

File uploads require multiple layers of validation:

```text id="p3r8x5"
Request Size
    ↓
Content Type
    ↓
Extension
    ↓
Magic Bytes / File Signature
    ↓
File Size
    ↓
Content Validation
    ↓
Malware/Security Scanning where required
    ↓
Storage
```

Never trust the client-provided filename or MIME type alone.

---

# 47. File Names

File names must not be allowed to control arbitrary filesystem paths.

Protect against:

```text id="m8x2q7"
../
..\ 
absolute paths
special device paths
```

Prefer generated object keys.

---

# 48. Object Storage Validation

Before storing an uploaded object:

- Validate size.
- Validate content type.
- Validate ownership.
- Validate tenant.
- Validate file purpose.
- Validate storage destination.

Do not let a client choose arbitrary object-storage buckets or prefixes.

---

# 49. JSON Validation

JSON must be validated before application use.

Consider:

```text id="q5m8x2"
Maximum body size
Maximum nesting depth
Required fields
Allowed fields
Types
```

Avoid accepting arbitrary deeply nested objects that can cause excessive processing.

---

# 50. XML Validation

XML input requires additional security controls.

Where XML is accepted:

- Disable unsafe external entity resolution.
- Bound document size.
- Bound nesting/complexity.
- Validate against the expected schema where applicable.

Protect against XML-based denial-of-service and entity expansion attacks.

---

# 51. CSV / Tabular Imports

KAMPYN import validation should include:

```text id="v7r3m8"
File size
Encoding
Delimiter
Header structure
Column count
Required columns
Column types
Row limits
Duplicate handling
Malformed rows
```

Large files must be processed incrementally.

---

# 52. Webhook Validation

Webhook input must be validated in stages:

```text id="g2x8m5"
Receive
 ↓
Size Limit
 ↓
Signature Verification
 ↓
Replay Protection
 ↓
Schema Validation
 ↓
Idempotency
 ↓
Application Processing
```

Do not parse and trust webhook content before verifying authenticity where the provider supports signatures.

---

# 53. External API Response Validation

External API responses are not trusted merely because they came from a known provider.

Validate:

```text id="r5m7x2"
Required fields
Types
Enum values
Nested structures
Provider status
Identifiers
Amounts
Timestamps
```

before mapping into domain/application types.

---

# 54. Database Data Validation

Data read from a database should generally be trusted as persistence data, but compatibility migrations and legacy data may produce unexpected values.

Important boundaries should still protect against:

```text id="x3q8m1"
corrupt records
legacy states
unexpected nulls
invalid enum values
migration inconsistencies
```

Do not assume old data always matches the newest domain model.

---

# 55. Configuration Validation

Configuration is an external input boundary.

Validate at startup:

```text id="m8x4r2"
Required environment variables
URLs
Ports
Durations
Database configuration
Feature flags
Security settings
Provider credentials
Resource limits
```

Fail fast when required configuration is invalid.

---

# 56. Configuration Must Be Typed

Avoid scattering string parsing throughout the application.

Preferred:

```text id="k5x7m3"
Environment
 ↓
Config Schema
 ↓
Typed Configuration
 ↓
Application
```

---

# 57. Environment-Specific Validation

Configuration may differ between:

```text id="q8m3x5"
development
test
staging
production
self-hosted
```

Required fields must be defined per environment.

Do not weaken production validation because development does not require a value.

---

# 58. Event Validation

Events crossing service boundaries must be validated.

Validate:

```text id="f7x2m8"
event type
event version
event ID
aggregate ID
tenant ID
timestamp
payload
metadata
```

Consumers must safely handle unknown or newer fields.

---

# 59. Event Version Validation

Events should use explicit versions where schema evolution requires them.

Example:

```text id="r3m8x2"
OrderCreated.v1
OrderCreated.v2
```

Consumers should reject or safely handle unsupported versions.

---

# 60. Message Validation

Queue messages require the same trust-boundary treatment as HTTP requests.

Do not assume:

```text id="x7m2q5"
"internal queue = trusted"
```

Messages can be:

- Duplicated.
- Delayed.
- Corrupted.
- Produced by old versions.
- Produced by buggy services.

---

# 61. Job Payload Validation

Before executing a background job:

```text id="m4x8r2"
Deserialize
 ↓
Validate
 ↓
Check version
 ↓
Establish tenant context
 ↓
Execute
```

Do not blindly cast job payloads into expected types.

---

# 62. Tenant Validation

Tenant IDs require special treatment.

A request may contain:

```text id="q8m3x5"
tenantId
```

but that does not establish tenant authority.

The trusted tenant context must come from:

```text id="f7r2m8"
authenticated membership
host/domain resolution
service identity
trusted routing
```

depending on the deployment model.

---

# 63. Cross-Tenant Validation

Whenever multiple tenant-scoped resources interact, validate that they belong to compatible tenant contexts.

Example:

```text id="v3x8m2"
Order.TenantID
Inventory.TenantID
User.TenantID
```

must not silently refer to different tenants.

---

# 64. Resource Ownership

Validation may establish that a resource exists.

Authorization determines whether the actor can access it.

Avoid leaking resource existence across authorization boundaries.

For sensitive resources, unauthorized access may need to appear as:

```text id="r7m3x8"
not found
```

rather than exposing existence.

---

# 65. State Validation

Services should validate expected state before executing transitions.

Example:

```text id="k5x2m7"
CancelOrder
 ↓
Load Order
 ↓
Check current state
 ↓
Domain transition
```

Do not blindly set:

```text id="q8r3m5"
status = CANCELLED
```

---

# 66. Cross-Field Validation

Some validation requires multiple fields.

Examples:

```text id="m7x2r8"
startDate < endDate
password confirmation matches password
minPrice <= maxPrice
currency matches payment method
```

These should be validated at the layer that owns the relationship.

---

# 67. Conditional Validation

Some fields are required only under certain conditions.

Example:

```text id="x5m8q2"
paymentMethod = CARD
        ↓
card-specific information required
```

Do not make unrelated fields globally required.

---

# 68. Validation Ordering

A useful ordering is:

```text id="r3x7m5"
1. Transport safety
2. Structural schema
3. Authentication context
4. Tenant context
5. Authorization
6. Application preconditions
7. Domain invariants
8. Persistence constraints
9. External provider constraints
```

The exact order may change when one validation depends on information produced by another layer.

---

# 69. Validation and Transactions

Validation that depends on mutable state may need to occur inside the transaction.

Example:

```text id="m8x2q5"
Check inventory
    ↓
Reserve inventory
```

If performed outside the transaction, another request may change inventory between the two operations.

---

# 70. TOCTOU

Avoid time-of-check/time-of-use bugs.

Bad:

```text id="q4x8m2"
Check permission
      ↓
Wait
      ↓
Perform operation
```

when the relevant state can change.

Use appropriate transactions, locks, or atomic operations.

---

# 71. Validation and Concurrency

Validation alone does not guarantee correctness.

Example:

```text id="k7m3x8"
Request A → inventory = 1
Request B → inventory = 1
```

Both may validate:

```text id="p5x2r7"
requested quantity = 1
```

Only database-level atomicity can guarantee that inventory is not oversold.

---

# 72. Validation and Idempotency

Repeated requests must not bypass correctness.

Example:

```text id="x8m4q2"
CreatePayment
```

may pass validation twice.

Idempotency mechanisms must prevent duplicate effects.

Validation and idempotency solve different problems.

---

# 73. Validation Errors

Validation errors should have stable machine-readable codes.

Example:

```json id="r3x7m8"
{
  "code": "INVALID_FIELD",
  "field": "quantity",
  "message": "Quantity must be greater than zero."
}
```

The exact KAMPYN API error contract is defined by the API architecture.

---

# 74. Multiple Validation Errors

Where useful, validation may return multiple field-level errors.

Example:

```text id="m7x2q5"
email
  → invalid format

quantity
  → must be positive

items
  → required
```

Do not expose internal implementation details.

---

# 75. Error Messages

Error messages should be:

- Clear.
- Stable enough for users.
- Free of sensitive information.
- Appropriate to the audience.

Do not expose:

```text id="q8r3m5"
SQL queries
stack traces
filesystem paths
database hostnames
provider secrets
internal object identifiers
```

---

# 76. Localization

User-facing validation messages may eventually require localization.

Therefore, machine-readable error codes should remain stable.

Prefer:

```text id="x5m8r2"
INVALID_QUANTITY
```

over making clients depend on:

```text id="p7x3m8"
"Quantity must be greater than zero."
```

as a programmatic identifier.

---

# 77. Validation and API Contracts

Validation rules that affect API consumers must be reflected in API documentation.

For example:

```text id="k4m8x2"
pageSize maximum
required fields
enum values
date format
error codes
```

must remain synchronized with OpenAPI or the authoritative contract.

---

# 78. Validation and SDKs

SDKs may perform client-side validation for developer experience.

However:

```text id="r7x3m5"
SDK validation
      ↓
UX improvement
```

does not replace:

```text id="m8x2q4"
Backend validation
      ↓
Security and correctness
```

The backend remains authoritative.

---

# 79. Validation Reuse

Reuse validation schemas when they represent the same boundary and semantics.

Do not duplicate:

```text id="q5m8x2"
same regex
same enum
same limits
same parsing
```

across unrelated files.

However, do not force one schema to serve different boundaries if their requirements differ.

---

# 80. Shared Validation

Shared validation is appropriate for genuinely shared concepts.

Examples:

```text id="x7m3r8"
UUID parsing
Email structure
Pagination parameters
Money representation
Tenant ID
```

Shared validation must remain small and domain-appropriate.

---

# 81. Validation Helpers

Helpers should be used when they improve clarity.

Good:

```text id="m4x8q2"
parseUUID()
parsePagination()
validateMoney()
```

Avoid:

```text id="q7m3r5"
validateEverything()
commonValidator()
universalValidation()
```

---

# 82. Validation Packages

Validation packages should have clear ownership.

Avoid a single:

```text id="k8x2m4"
validation/
```

directory containing every domain's business rules.

Prefer domain-oriented schemas where appropriate:

```text id="r5m7x2"
orders/validation
bookings/validation
users/validation
```

with genuinely shared primitives separated appropriately.

---

# 83. Validation and Domain Objects

Constructors or factory functions may enforce domain invariants.

Example:

```go id="x3m8q7"
NewMoney(...)
NewBooking(...)
NewOrder(...)
```

This ensures invalid domain objects cannot easily be constructed.

---

# 84. Value Object Validation

Value objects should validate their own invariants.

Examples:

```text id="m7x2r8"
Email
Money
Quantity
PhoneNumber
TenantID
OrderID
```

Once constructed, they should represent valid values.

---

# 85. Primitive Obsession

Where invalid primitive values are common, prefer meaningful domain types.

Instead of:

```go id="q8m3x5"
func Process(id string)
```

consider:

```go id="r7x2m4"
func Process(id OrderID)
```

when the domain benefits from that distinction.

---

# 86. Validation and Serialization

Serialization boundaries must validate decoded data.

For example:

```text id="x5m8q2"
JSON
 ↓
Decode
 ↓
Validate
 ↓
Typed Object
```

Do not treat successful JSON decoding as successful validation.

---

# 87. Validation and Deserialization

Deserialization may fail because:

```text id="m8x3r5"
invalid syntax
wrong type
missing field
unknown version
invalid encoding
```

These should be distinguished where useful.

---

# 88. Validation and Database Writes

Never assume that validation before a write guarantees the write will succeed.

Writes may still fail because of:

```text id="q4m8x2"
unique constraints
foreign keys
deadlocks
concurrent changes
connection failures
```

The service must handle these outcomes.

---

# 89. Validation and Deletes

Deletion requests require validation of:

```text id="x7m3r8"
resource identifier
resource state
authorization
tenant scope
deletion policy
```

Bulk deletion requires especially strict filtering.

---

# 90. Validation and PATCH

PATCH-like operations require careful semantics.

Distinguish:

```text id="m5x8q2"
field omitted
field = null
field = new value
```

The API contract must define how each is interpreted.

---

# 91. Validation and PUT

PUT-style replacement operations should validate the complete representation expected by the API.

Do not accidentally treat PUT as partial update unless explicitly defined.

---

# 92. Validation and Bulk APIs

Bulk operations must validate:

```text id="r8x3m5"
maximum item count
individual item validity
cross-item constraints
duplicate items
tenant consistency
authorization
partial failure semantics
```

Never allow unbounded bulk requests.

---

# 93. Partial Failure

Bulk operations must define whether:

```text id="q7m2x8"
all succeed
all fail
partial success is allowed
```

Validation must not hide the distinction.

---

# 94. Import Validation

Large imports should distinguish:

```text id="m4x8r2"
file-level errors
row-level errors
field-level errors
system errors
```

The user should be able to identify invalid rows without exposing internal stack traces.

---

# 95. Validation Limits

Every externally controllable expensive operation should have limits.

Examples:

```text id="x8m3q5"
Maximum body size
Maximum file size
Maximum rows
Maximum array items
Maximum nesting depth
Maximum query length
Maximum pagination size
Maximum filters
Maximum batch size
```

Limits protect both correctness and availability.

---

# 96. Denial-of-Service Considerations

Validation must help prevent resource exhaustion.

Attackers may submit syntactically valid but computationally expensive input.

Examples:

```text id="k5r8x2"
Huge arrays
Deep JSON
Expensive search filters
Large regex inputs
Large file decompression
Massive bulk requests
```

Validation must consider computational cost, not just syntax.

---

# 97. Regular Expressions

Regex validation should be designed carefully.

Avoid catastrophic backtracking patterns.

Do not use unnecessarily complex regexes for fields that can be validated more safely through simpler parsing.

---

# 98. ReDoS

Regular expressions operating on attacker-controlled input must be reviewed for denial-of-service risk.

Prefer:

```text id="q7m3x8"
simple regex
bounded input
dedicated parser
allowlist
```

where possible.

---

# 99. Validation and Decompression

Compressed files require validation before and during decompression.

Protect against:

```text id="m8x2r4"
decompression bombs
unexpected archive size
path traversal
excessive file counts
nested archives
```

Resource limits must apply to the decompressed output as well as the compressed input.

---

# 100. Archive Validation

For ZIP/TAR uploads:

```text id="x5m8q2"
Archive
 ↓
Validate entries
 ↓
Validate paths
 ↓
Validate count
 ↓
Validate extracted size
 ↓
Extract safely
```

Never blindly extract user-provided archives.

---

# 101. Validation of External URLs

If KAMPYN accepts URLs:

```text id="r7x3m8"
Parse URL
 ↓
Validate scheme
 ↓
Validate hostname
 ↓
Apply allowlist/blocklist policy
 ↓
Perform bounded request
```

This is especially important for server-side fetching.

---

# 102. SSRF Validation

Validation must not treat:

```text id="q4m8x2"
https://example.com
```

as inherently safe.

A malicious URL may resolve to:

```text id="k7x2m5"
localhost
private network
cloud metadata service
internal service
```

URL validation must consider the actual network policy.

---

# 103. Validation and Permissions

Permission values must be validated against the known permission model.

Do not allow:

```text id="m8x3r7"
permissions: ["anything"]
```

to become a valid internal permission set.

Unknown permissions should be rejected or handled explicitly.

---

# 104. Validation and Roles

Role assignment requires more than validating that the role string exists.

The application must additionally verify:

```text id="x5m7q2"
actor permission
target tenant
target user
role assignment policy
```

Validation establishes shape; authorization establishes authority.

---

# 105. Validation and Admin Operations

Administrative APIs require strict validation and authorization.

Sensitive operations should validate:

```text id="r8m3x5"
target resource
target tenant
requested state
reason where required
confirmation requirements
```

Audit requirements must be explicit.

---

# 106. Validation and Community Content

Community input may require validation for:

```text id="k4x8m2"
length
content type
attachments
mentions
URLs
formatting
moderation constraints
```

Content validation does not replace moderation or authorization.

---

# 107. Validation and Private Chat

Chat messages should validate:

```text id="m7x2r8"
conversation ID
sender identity
membership
message size
content type
attachments
```

Authorization must verify that the sender belongs to the conversation.

---

# 108. Validation and HTML

If user-generated content supports HTML or rich text, validation alone is insufficient.

Sanitization must remove unsafe content according to the rendering context.

Do not assume:

```text id="q8m3x5"
"valid HTML"
=
"safe HTML"
```

---

# 109. Validation and XSS

Any user-controlled content rendered in a browser must be handled according to its output context.

Use framework escaping by default.

Do not bypass escaping simply because validation accepted the input.

---

# 110. Validation and CSRF

CSRF protection is not input validation.

For cookie-authenticated state-changing requests, appropriate CSRF defenses remain necessary.

Validation cannot substitute for CSRF protection.

---

# 111. Validation and Authentication

Authentication credentials must be validated before attempting authentication.

Examples:

```text id="x5m8r2"
token structure
OTP format
email structure
password presence
```

But malformed credentials should not produce detailed information that assists account enumeration.

---

# 112. Validation and Rate Limiting

Some validation endpoints are security-sensitive.

Examples:

```text id="r7m3x8"
Login
OTP
Password reset
Email verification
Search
File upload
Report generation
```

Validation and rate limiting should work together.

---

# 113. Validation and Account Enumeration

Error responses should avoid unnecessarily revealing:

```text id="k4x8m2"
whether an account exists
whether an email is registered
whether a reset token exists
```

when the workflow is sensitive to enumeration.

---

# 114. Validation and Secrets

Never validate secrets by logging them.

Bad:

```text id="m8x2r5"
logger.Info("received token", token)
```

Validation failures must not expose secret values.

---

# 115. Validation and Logging

Safe validation failure logs may include:

```text id="x7m3r8"
requestId
route
tenantId where safe
error code
field name where safe
```

Do not log complete request bodies by default.

---

# 116. Validation and Observability

Track validation failures where useful:

```text id="q5m8x2"
validation_failure_total
```

with bounded dimensions such as:

```text id="r7m3x8"
endpoint
validation_code
```

Avoid high-cardinality labels such as:

```text id="k4x8m2"
email
userId
requestId
raw input
```

---

# 117. Validation Testing

Every validation boundary should have tests for:

### Valid input

```text id="m8x2r5"
minimum valid
maximum valid
normal valid
```

### Invalid input

```text id="x7m3q8"
missing
wrong type
out of range
malformed
unexpected
oversized
```

### Security input

```text id="r4x8m2"
injection
path traversal
oversized input
malicious URL
unexpected fields
```

---

# 118. Boundary Tests

Test validation at actual boundaries.

For HTTP:

```text id="k7m3x8"
HTTP
 ↓
Decode
 ↓
Validate
```

For events:

```text id="q5x8m2"
Message
 ↓
Decode
 ↓
Validate
```

For configuration:

```text id="r7m3x8"
Environment
 ↓
Parse
 ↓
Validate
```

---

# 119. Property-Based Testing

Property-based testing may be useful for validation-heavy components.

Examples:

```text id="m8x2r5"
Pagination parser
Money parser
Identifier parser
Date-range validation
Input normalization
```

The exact tooling is repository-defined.

---

# 120. Fuzz Testing

Fuzz testing is especially useful for parsers and untrusted input.

Potential targets:

```text id="x7m3q8"
JSON parsing
CSV parsing
URL parsing
file metadata
archive parsing
pagination cursors
webhook payloads
```

A parser should fail safely rather than crash the service.

---

# 121. Validation and Performance

Validation itself must not become a denial-of-service vector.

Avoid:

```text id="r4x8m2"
expensive database query
```

for every syntactically invalid request.

Perform cheap validation first.

---

# 122. Validation and Database Queries

When validation requires database state:

```text id="k7m3x8"
Cheap syntax validation
        ↓
Authentication
        ↓
Authorization
        ↓
Database validation
```

Do not query the database for obviously malformed identifiers.

---

# 123. Validation and Caching

Do not cache validation results blindly when underlying state can change.

For example:

```text id="q5x8m2"
"User can perform action X"
```

may become invalid after:

```text id="r7m3x8"
role change
tenant suspension
resource ownership change
```

Authorization and validation caches must have explicit consistency semantics.

---

# 124. Validation and Search

Search filters must respect authorization and tenant boundaries.

Do not rely on:

```text id="m8x2r5"
validation that tenantId is syntactically valid
```

to establish that the user may search that tenant.

---

# 125. Validation and Events

Consumers should validate event metadata before trusting:

```text id="x7m3q8"
tenantId
aggregateId
eventType
eventVersion
```

A malformed event must not cause cross-tenant processing.

---

# 126. Validation and Jobs

Jobs should validate their payload before execution.

If the payload is invalid:

```text id="r4x8m2"
Do not execute
 ↓
Record failure
 ↓
Dead-letter / alert according to policy
```

Do not repeatedly retry permanently invalid jobs.

---

# 127. Validation and Retries

Differentiate:

```text id="k7m3x8"
Invalid Input
    ↓
Non-retryable

Temporary Dependency Failure
    ↓
Potentially Retryable
```

Retrying invalid input wastes resources and can create queue pressure.

---

# 128. Validation and Dead Letters

Dead-letter records should preserve enough metadata to diagnose the issue without storing unnecessary sensitive payloads.

Include where appropriate:

```text id="q5x8m2"
job/event ID
type
version
tenant context
failure code
attempt count
timestamp
```

---

# 129. Validation and API Error Codes

Validation errors should remain consistent across endpoints.

For example:

```text id="r7m3x8"
INVALID_REQUEST
INVALID_FIELD
INVALID_FORMAT
INVALID_ENUM
OUT_OF_RANGE
MISSING_FIELD
PAYLOAD_TOO_LARGE
```

The exact canonical KAMPYN error taxonomy should be defined centrally.

---

# 130. Validation and Internationalization

Machine-readable validation codes should remain language-neutral.

Human-readable messages may be localized at the appropriate presentation boundary.

---

# 131. Validation and Documentation

Validation rules that affect consumers must be documented.

Documentation should include:

```text id="m8x2r5"
required fields
allowed values
limits
formats
nullable fields
error codes
examples
```

Keep documentation synchronized with schemas.

---

# 132. Validation and Generated Contracts

If OpenAPI or generated schemas are used:

```text id="x7m3q8"
Canonical Contract
       ↓
Generated Types / SDK
       ↓
Runtime Validation
```

Do not manually maintain multiple contradictory versions.

---

# 133. Validation and API Evolution

When changing validation rules, consider compatibility.

Changing:

```text id="r4x8m2"
maximum length: 500
```

to:

```text id="k7m3x8"
maximum length: 100
```

may break existing clients.

Validation changes are API changes when they affect consumers.

---

# 134. Backward Compatibility

Prefer additive changes where practical.

For stricter validation:

```text id="q5x8m2"
Identify existing clients
 ↓
Assess compatibility
 ↓
Introduce migration path
 ↓
Enforce new rule
```

Do not silently break clients.

---

# 135. Validation and Versioning

If incompatible validation semantics are required, use the API/versioning strategy defined by KAMPYN.

Do not silently change the meaning of an existing endpoint.

---

# 136. Service-Level Validation

Application services may validate use-case-specific preconditions.

Example:

```text id="r7m3x8"
CreateBooking
 ↓
Validate requested dates
 ↓
Validate tenant
 ↓
Check resource availability
 ↓
Domain booking rule
```

The service should not repeat basic JSON shape validation already performed at the transport boundary.

---

# 137. Domain-Level Validation

Domain constructors and methods should protect invariants.

Example:

```text id="m8x2r5"
NewQuantity(-1)
      ↓
Error
```

or:

```text id="x7m3q8"
order.Cancel()
      ↓
InvalidState
```

This protects the domain regardless of caller.

---

# 138. Database-Level Validation

The database should protect critical integrity even if application validation fails.

Examples:

```text id="r4x8m2"
unique tenant + email
foreign key
positive quantity
unique booking slot
```

The exact constraints depend on the domain model.

---

# 139. Validation and Defense in Depth

KAMPYN should intentionally use multiple validation layers:

```text id="k7m3x8"
Client
   ↓
Transport
   ↓
Application
   ↓
Domain
   ↓
Database
```

Each layer protects against different failure modes.

Do not remove a deeper protection simply because an earlier layer validates the same concept.

---

# 140. Avoid Duplicate Validation

Do not repeatedly execute expensive or identical validation.

For example:

```text id="q5x8m2"
HTTP
 ↓
validate UUID
 ↓
Service
 ↓
validate UUID again
 ↓
Repository
 ↓
validate UUID again
```

A trusted typed `UUID` value should generally be passed after the boundary validation.

---

# 141. Trusted Internal Values

Once a value has crossed a validated boundary and has not been transformed into an untrusted representation, downstream code may rely on its established type/invariant.

For example:

```text id="r7m3x8"
string
 ↓
Parse
 ↓
OrderID
```

Downstream code should receive `OrderID` rather than repeatedly parsing the original string.

---

# 142. Validation State

Do not create ambiguous booleans such as:

```text id="m8x2r5"
isValidated
```

to represent complex validation state.

Prefer types that make invalid states harder to represent.

---

# 143. Validation Results

Validation functions should have predictable semantics.

For example:

```go id="x7m3q8"
value, err := ParseOrderID(input)
```

is preferable to:

```go id="r4x8m2"
if Validate(input) {
    ...
}
```

when the validated result itself is needed.

---

# 144. Parser vs Validator

Use parsing when transforming untrusted representation into a trusted type.

Example:

```text id="k7m3x8"
"123e4567..."
       ↓
Parse
       ↓
OrderID
```

Use validation when checking an already structured value.

Example:

```text id="q5x8m2"
Order
 ↓
ValidateState()
```

The distinction should remain clear.

---

# 145. Normalization

Normalize only when the domain requires canonical representation.

Examples:

```text id="r7m3x8"
Email casing policy
Phone number representation
Unicode normalization
Whitespace handling
```

Do not normalize arbitrary content destructively.

---

# 146. Validation and Canonicalization

Canonicalization can affect security.

If two representations should be treated as equivalent, define the canonical form explicitly before comparison.

This is particularly important for:

```text id="m8x2r5"
identifiers
URLs
paths
email addresses
Unicode text
```

---

# 147. Validation and Authorization Order

Do not leak sensitive resource information through validation errors.

For example, if a resource is inaccessible:

```text id="x7m3q8"
Authorization / visibility decision
        ↓
then detailed validation
```

may be preferable to revealing that the resource exists.

The exact ordering depends on the endpoint's security model.

---

# 148. Validation and Rate Limits

Do not let attackers bypass rate limits simply by submitting malformed input.

Security-sensitive endpoints should apply appropriate rate limits before expensive validation where necessary.

---

# 149. Validation and Request Size

Request size limits should be enforced before expensive parsing.

Preferred:

```text id="r4x8m2"
Network / HTTP limit
 ↓
Body limit
 ↓
Parser
 ↓
Schema validation
```

Do not parse gigabytes of JSON just to discover it is invalid.

---

# 150. Validation and Compression

If HTTP compression is accepted, enforce limits on the decompressed payload as well.

Compressed size alone is not a sufficient safety limit.

---

# 151. Validation and Response Data

Validation is also relevant to output.

Before returning sensitive or externally exposed data, ensure that the response contains only fields allowed by the API contract.

Do not serialize entire persistence objects blindly.

---

# 152. Response Validation

Important integrations and public APIs may use response schemas to detect accidental contract drift.

Example:

```text id="k7m3x8"
Application Result
 ↓
Response DTO
 ↓
Schema / Serialization
 ↓
Client
```

Sensitive internal fields must never leak accidentally.

---

# 153. Validation and Serialization Boundaries

Every serialization boundary should define:

```text id="q5x8m2"
Allowed fields
Field types
Nullability
Enum values
Version
Encoding
```

This applies to:

```text id="r7m3x8"
HTTP
Events
Jobs
Cache
Database
External providers
```

---

# 154. Validation and Cache Data

Cached values may outlive application versions.

When cache schemas change:

```text id="m8x2r5"
Version keys
 ↓
Invalidate
or
Support compatibility
```

Do not assume Redis contains only the newest representation.

---

# 155. Validation and Event Replay

Events may be replayed months later.

Consumers should validate event versions and payloads without assuming today's schema existed when the event was created.

---

# 156. Validation and Data Migration

Schema migrations may temporarily create mixed representations.

Validation should account for the migration strategy.

Do not deploy code that assumes the new representation before the database rollout makes that representation available.

---

# 157. Validation and Legacy Data

Legacy data should be handled deliberately.

Possible strategies:

```text id="x7m3q8"
Migration
Compatibility mapper
Default value
Explicit invalid-state handling
Quarantine
```

Do not silently invent values that change business meaning.

---

# 158. Validation and Backfills

Backfills must validate generated values before writing them.

Large backfills should:

- Process bounded batches.
- Be restart-safe.
- Track progress.
- Avoid violating new constraints.
- Be observable.

---

# 159. Validation and Data Imports

Imports should never directly bypass domain validation unless the import architecture explicitly defines a controlled trusted path.

Preferred:

```text id="r4x8m2"
Import
 ↓
Parse
 ↓
Validate
 ↓
Normalize
 ↓
Application Use Case
 ↓
Domain
 ↓
Persistence
```

---

# 160. Validation and Admin Imports

Even privileged administrators should not bypass critical integrity validation.

Administrative authority changes **who may perform an operation**, not whether invalid state is acceptable.

---

# 161. Validation and System Operations

Internal tools and CLI commands are still trust boundaries.

Validate:

```text id="k7m3x8"
CLI arguments
configuration
file input
IDs
environment
database targets
```

Do not assume internal tooling is always used correctly.

---

# 162. Validation and Self-Hosted Configuration

Self-hosted deployments may have different configurations.

Startup validation must clearly identify:

```text id="q5x8m2"
missing configuration
invalid configuration
unsupported combination
```

The service should fail fast rather than operate with dangerous defaults.

---

# 163. Safe Defaults

When configuration is omitted, defaults must be safe.

Examples:

```text id="r7m3x8"
reasonable request limits
secure cookies
TLS expectations
disabled dangerous debug modes
bounded concurrency
```

Do not choose permissive defaults merely to simplify setup.

---

# 164. Validation and Feature Flags

Feature flag values should be validated.

Do not accept arbitrary strings where a known set of states exists.

For example:

```text id="m8x2r5"
enabled
disabled
percentage
targeted
```

must have explicit semantics.

---

# 165. Validation and Feature Compatibility

When enabling a feature, validate that its dependencies are available.

Example:

```text id="x7m3q8"
Enable Payment Provider
        ↓
Provider Configuration Valid?
        ↓
Required Credentials?
        ↓
Supported Currency?
```

Do not allow configurations that cannot operate.

---

# 166. Validation and Startup

Critical validation should happen during startup rather than first request.

Examples:

```text id="r4x8m2"
invalid database URL
missing secret
invalid service URL
invalid feature configuration
unsupported environment
```

Failing early is safer than discovering the issue under production traffic.

---

# 167. Validation and Health Checks

Readiness should reflect required configuration and dependencies.

A service with invalid configuration should not report itself ready.

---

# 168. Validation and Security Boundaries

Validation must occur whenever data crosses from:

```text id="k7m3x8"
Browser → Server
Service → Service
Queue → Worker
Provider → Service
File → Parser
Database → Domain
CLI → Service
```

Every boundary must define what it trusts.

---

# 169. Validation Checklist

Before accepting a new input boundary, ask:

### Structure

- Is the format valid?
- Are required fields present?
- Are types correct?
- Are unknown fields handled deliberately?

### Limits

- Is size bounded?
- Is complexity bounded?
- Are arrays bounded?
- Are ranges bounded?

### Security

- Can this cause injection?
- Can this cause SSRF?
- Can this cause path traversal?
- Can this cause XSS?
- Can this cause resource exhaustion?

### Semantics

- Are cross-field rules defined?
- Are enums explicit?
- Are dates unambiguous?
- Are identifiers valid?

### Authorization

- Is the actor allowed?
- Is tenant context trusted?
- Is resource ownership verified?

### Persistence

- Are database constraints present?
- Can concurrency invalidate the validation?
- Is a transaction required?

### Reliability

- Is the operation idempotent?
- Can it be safely retried?
- What happens on timeout?

---

# 170. Definition of Done

Validation is complete only when:

- Every external trust boundary has explicit validation.
- Transport schemas are defined.
- Application preconditions are explicit.
- Domain invariants are enforced by domain logic.
- Critical persistence invariants are backed by database constraints.
- Tenant context is validated and trusted.
- Authorization is separate from basic validation.
- Input size and complexity are bounded.
- Dynamic queries use allowlists.
- Files are validated safely.
- External responses are validated.
- Webhooks are authenticated before processing.
- Jobs/events are schema-validated.
- Configuration is validated at startup.
- Errors use stable machine-readable semantics.
- Sensitive values are not logged.
- Validation failures are observable where useful.
- Tests cover valid, invalid, boundary, and malicious inputs.
- API documentation reflects validation rules.
- Compatibility implications have been reviewed.

---

# 171. Final Invariants

```text id="k3x8m5"
1. Validation is a layered responsibility.

2. Every external trust boundary must validate its input.

3. Client-side validation never replaces backend validation.

4. TypeScript types never replace runtime validation.

5. Transport validation protects request shape.

6. Application validation protects use-case preconditions.

7. Domain validation protects business invariants.

8. Database constraints protect persistence integrity.

9. External provider responses must be treated as untrusted input.

10. Validation is not authorization.

11. Validation is not business workflow logic.

12. Validation is not a substitute for database constraints.

13. Critical invariants must remain protected under concurrency.

14. Check-then-act validation is unsafe when shared state can change.

15. Tenant identifiers supplied by clients do not establish tenant authority.

16. Tenant isolation must remain enforced beyond validation.

17. Dynamic database fields, sorting, filtering, and queries must use
    explicit allowlists.

18. Request size and computational complexity must be bounded.

19. Unbounded arrays, files, queries, and batch operations are prohibited.

20. Large data must be validated and processed using bounded resources.

21. File type validation must not rely solely on filename or MIME type.

22. Webhook authenticity must be verified before trusting webhook data.

23. Job and event payloads must be validated before execution.

24. Configuration must be validated before the service becomes ready.

25. Invalid permanent jobs/events must not be retried indefinitely.

26. Validation errors must have stable machine-readable semantics.

27. Error responses must not expose secrets or internal infrastructure.

28. Validation must not leak sensitive resource existence.

29. Validation must not create unnecessary database or external calls.

30. Expensive validation must be bounded and observable.

31. Validation schemas should be reused where semantics are genuinely shared.

32. Validation must not be duplicated merely for appearance.

33. Trusted values should be converted into explicit types where useful.

34. Domain value objects should prevent invalid states from being constructed.

35. Validation changes that affect API consumers are compatibility changes.

36. API documentation must remain synchronized with validation behavior.

37. Security-sensitive validation must be tested with malicious input.

38. Validation must remain correct after retries, concurrency, restarts,
    migrations, and partial failures.

39. Validation should make invalid states harder to reach, not merely
    report them after they occur.

40. The strongest validation is layered defense:
    boundary validation + application rules + domain invariants +
    persistence constraints.
```

## KAMPYN Validation Model

```text id="n7m3x8"
                         EXTERNAL INPUT
                              │
                 ┌────────────┴────────────┐
                 │                         │
              HTTP                      EVENT/JOB
                 │                         │
                 └────────────┬────────────┘
                              ↓
                    TRANSPORT VALIDATION
                 Schema / Size / Encoding
                              │
                              ↓
                    AUTHENTICATION CONTEXT
                              │
                              ↓
                       TENANT CONTEXT
                              │
                              ↓
                        AUTHORIZATION
                              │
                              ↓
                   APPLICATION VALIDATION
                 Preconditions / References
                              │
                              ↓
                       DOMAIN VALIDATION
                  Business Invariants
                              │
                              ↓
                   TRANSACTION / DATABASE
                 Constraints / Atomicity
                              │
                              ↓
                   EXTERNAL INTEGRATIONS
                              │
                              ↓
                    RESPONSE / EVENT / JOB
                              │
                              ↓
                    OUTPUT CONTRACT
```

The governing principle is:

```text id="x5r8m2"
Validate at the boundary that owns the rule.

Validate external data before trusting it.

Use the domain to protect business invariants.

Use the database to guarantee critical integrity.

Never confuse validation with authorization.

Never assume validation alone makes a concurrent operation safe.

Never let validation become an unbounded source of computation.

The objective is not "validate everything everywhere."

The objective is to make invalid, unsafe, unauthorized,
and inconsistent states difficult to enter and impossible
to persist when they violate KAMPYN's invariants.
```
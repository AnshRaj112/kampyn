# Tenant Isolation & Multi-Tenant Security

## 1. Purpose

This document defines the mandatory security standards for isolating tenants across KAMPYN's applications, APIs, databases, caches, search infrastructure, background workers, integrations, and deployment environments.

KAMPYN is a multi-tenant university hospitality platform designed to support multiple universities, each with its own users, vendors, food courts, facilities, bookings, inventory, administrative operations, and institutional configurations.

Tenant isolation ensures that one university cannot access, modify, infer, or interfere with another university's data, resources, operations, or configurations without explicitly authorized platform-level access.

This policy applies to:

- All backend services written in Go
- Next.js and TypeScript frontend applications
- Public and internal APIs
- PostgreSQL, MongoDB, Redis, and OpenSearch
- Authentication and authorization systems
- Background workers and scheduled jobs
- Events, queues, and real-time communication
- File and object storage
- Payment and third-party integrations
- Analytics, reporting, and monitoring
- SDKs, APIs, and university integrations
- Docker, Kubernetes, and cloud infrastructure
- University self-hosted deployments
- Administrative and support tooling

Tenant isolation is a mandatory security boundary and must be enforced consistently across every component that processes tenant-scoped data.

## 2. Core Principles

All tenant-aware implementations must follow these principles:

- **Default isolation:** Tenant data must be inaccessible to other tenants unless an explicitly authorized cross-tenant operation is performed.
- **Server-derived tenant context:** Tenant identity must be established and validated by trusted server-side mechanisms.
- **Deny by default:** Missing, invalid, ambiguous, or unauthorized tenant context must result in denial.
- **Defense in depth:** Tenant isolation must be enforced across application logic, data access, storage, caching, search, and infrastructure.
- **Least privilege:** Users, services, workers, and administrators must receive only the tenant access required for their responsibilities.
- **Explicit tenant scoping:** Every tenant-scoped operation must have an explicit, validated tenant context.
- **No implicit trust:** Internal services, queues, caches, and databases must not be trusted to preserve tenant boundaries automatically.
- **Consistent enforcement:** The same tenant access rules must apply to reads, writes, updates, deletes, exports, and asynchronous operations.
- **Data minimization:** Services must process only the tenant data necessary for their responsibilities.
- **Auditable access:** Cross-tenant and privileged tenant operations must be traceable.
- **Isolation under failure:** Errors, retries, cache misses, stale tokens, and partial failures must never cause tenant boundaries to be bypassed.
- **Secure extensibility:** New tenants, services, integrations, and deployment models must preserve the established isolation guarantees.

## 3. Tenant Security Model

A tenant represents an independently administered university or institution operating within KAMPYN.

Each tenant has its own logical security boundary, including its users, resources, configurations, and authorized operations.

<text>Examples of tenant-scoped resources include:</text>

- Students, faculty, staff, and administrators
- Food courts, vendors, menus, and orders
- Hostel and guest-house bookings
- Washing-machine schedules
- Library availability and reservations
- Shuttle schedules and bookings
- Inventory and stock records
- Complaints and support tickets
- Community spaces and messages
- HR records and workflows
- Tenant-level analytics and reports
- Payment configurations and transaction records
- Notifications and preferences
- Files, attachments, and exports

### 3.1 Tenant Identity

Every tenant must have a stable, immutable, globally unique identifier.

Use a canonical identifier such as:

```text
tenant_id
```

The tenant identifier must not be treated as a secret or as proof of authorization.

Tenant IDs must not be reused after a tenant is deleted or decommissioned.

### 3.2 Tenant Ownership

Every tenant-scoped resource must have an explicit and enforceable ownership relationship.

For example:

```text
Tenant
 ├── Users
 ├── Food Courts
 │    ├── Vendors
 │    ├── Menus
 │    ├── Orders
 │    └── Inventory
 ├── Facilities
 │    ├── Hostel Bookings
 │    ├── Guest House Bookings
 │    └── Washing Schedules
 ├── Library
 ├── Shuttle
 ├── Complaints
 ├── Community
 ├── HR
 └── Analytics
```

The ownership model must be defined in domain schemas and enforced by application and data-access layers.

### 3.3 Tenant Membership

A user may belong to one or more tenants, depending on the approved product design.

Membership must explicitly define:

- User identity
- Tenant identity
- Membership status
- Assigned roles
- Permissions
- Membership validity
- Relevant organizational scope

Membership in one tenant must not imply membership in another tenant.

Tenant membership must be checked whenever a user accesses tenant-scoped resources.

## 4. Tenant Context Resolution

Tenant context determines the tenant under which an operation is performed.

Tenant context must be established through trusted authentication, validated routing, or an explicitly authorized service identity.

### 4.1 Trusted Context Sources

Tenant context may be resolved using:

- A validated authenticated user's active tenant membership
- A tenant-specific, verified domain or routing configuration
- A trusted service-to-service identity with an authorized tenant scope
- An explicitly authorized platform-level administrative operation

The source and precedence of tenant context must be documented for each request type.

### 4.2 Untrusted Tenant Identifiers

Never trust tenant IDs supplied solely through:

- Request bodies
- Query parameters
- URL parameters
- Custom headers
- Frontend state
- Browser storage
- SDK configuration
- Event payloads
- Client-controlled cookies

These values may be used as selectors or routing hints, but must be validated against the authenticated principal and trusted tenant configuration before use.

### 4.3 Tenant Resolution Flow

Every tenant-scoped request must follow a controlled sequence:

1. Authenticate the caller or validate the trusted service identity.
2. Resolve the requested tenant context from permitted sources.
3. Verify that the tenant exists and is active.
4. Verify that the caller or service is authorized for that tenant.
5. Construct an immutable tenant context for the operation.
6. Pass the validated context through service and repository boundaries.
7. Enforce tenant scope on every resource access.
8. Record security-sensitive actions where required.

If tenant resolution fails, the request must be rejected.

### 4.4 Immutable Tenant Context

Once established, tenant context must not be arbitrarily overwritten by downstream components.

In Go, pass an explicit tenant-aware request context or typed domain context through service boundaries. Context values may be used for request-scoped propagation, but must not be treated as a substitute for explicit authorization or type-safe function contracts.

Avoid mutable global tenant variables and shared singleton state containing request-specific tenant information.

### 4.5 Tenant Context Propagation

Tenant context must be preserved across:

- Middleware
- Controllers and handlers
- Domain services
- Repository methods
- Database transactions
- Cache operations
- Search requests
- Event publication
- Queue messages
- Background workers
- WebSocket connections
- External integration calls

Each boundary must validate the tenant context according to its trust level.

## 5. Tenant-Aware Architecture

Tenant isolation must be a cross-cutting architecture concern rather than a collection of isolated checks.

### 5.1 Application Layers

Tenant context must be enforced at multiple layers.

| Layer | Responsibility |
|---|---|
| Edge / Gateway | Validate trusted routing context and apply request-level protections |
| Authentication | Establish authenticated identity and tenant memberships |
| Authorization | Verify access to the requested tenant and resource |
| Middleware | Build and propagate validated tenant context |
| Domain services | Enforce tenant-aware business rules |
| Repositories | Scope data access to the validated tenant |
| Database | Enforce integrity and isolation safeguards where supported |
| Cache | Scope keys and operations to the correct tenant |
| Search | Apply server-controlled tenant filters |
| Events and jobs | Preserve and validate tenant scope |
| Storage | Restrict file access to the owning tenant |
| Observability | Prevent cross-tenant data leakage in logs and telemetry |

No single layer may be treated as the sole tenant isolation boundary.

### 5.2 Service Boundaries

Every service that handles tenant-scoped information must declare:

- Whether it is tenant-aware
- Which tenant-scoped resources it owns
- Which tenant context it accepts
- Which tenant permissions it requires
- Which cross-tenant operations, if any, it supports
- Which data stores and external systems it accesses

Services must not assume that upstream validation guarantees correct tenant scope throughout the entire operation.

### 5.3 Service-to-Service Communication

Internal services must authenticate one another and enforce tenant-scoped authorization.

Service identity must not automatically grant access to all tenant data.

Use narrowly scoped service identities, explicit tenant context, and validated service-to-service contracts.

A service that acts on behalf of a user must preserve the original user's authorized tenant scope where applicable.

## 6. Authentication and Tenant Membership

Authentication establishes identity. It does not independently establish unrestricted access to a tenant.

### 6.1 Membership Validation

Before accessing tenant-scoped resources, the system must verify:

- The user is authenticated.
- The tenant is active.
- The user has a valid membership in the tenant.
- The membership is not suspended or revoked.
- The requested operation is within the user's permissions.
- Any required organizational or resource-level restrictions are satisfied.

### 6.2 Active Tenant Selection

Where users can access multiple tenants, active tenant selection must be explicitly resolved and validated.

Switching tenants must not preserve unauthorized access to resources or cached information from the previous tenant.

On tenant switching:

- Revalidate membership and permissions.
- Update the server-side tenant context.
- Clear or partition client-side tenant-specific state.
- Invalidate relevant cached queries.
- Reestablish tenant-specific real-time subscriptions.
- Ensure outstanding operations cannot silently continue under stale tenant context.

### 6.3 Token Claims

If tokens contain tenant claims, those claims must be verified and treated as bounded authorization context.

A token's tenant claim must not override current membership status or permissions when immediate revocation or dynamic authorization checks are required.

Long-lived tokens must not create indefinite tenant access after membership removal.

### 6.4 Account Recovery and Invitations

Invitation, account recovery, and membership acceptance flows must be tenant-bound.

Tokens must be validated for:

- Intended tenant
- Intended recipient or identity
- Permitted action
- Expiration
- Single-use or replay constraints where applicable
- Revocation status

A valid invitation for one tenant must not grant access to another tenant.

## 7. Authorization and Tenant Isolation

Tenant isolation is enforced through authorization, ownership validation, and scoped data access.

### 7.1 Mandatory Authorization

Every tenant-scoped operation must check:

1. The authenticated principal or service identity.
2. The active tenant context.
3. The resource's tenant ownership.
4. The permission required for the operation.
5. Additional attribute or relationship constraints.

### 7.2 Object-Level Authorization

Every resource lookup must verify tenant ownership before returning or modifying the resource.

For example, retrieving an order must validate that the order belongs to the active tenant and that the user is authorized to access it.

Knowledge of a resource ID must never grant access.

### 7.3 Function-Level Authorization

Access to a function must be checked independently of resource access.

Examples include:

- Viewing tenant analytics
- Exporting order records
- Managing food courts
- Adjusting inventory
- Assigning HR roles
- Approving refunds
- Configuring payment integrations
- Managing tenant membership

### 7.4 Property-Level Authorization

Sensitive fields must be protected independently.

Examples include:

- User roles
- Membership status
- Payment details
- Internal complaint notes
- HR information
- Inventory adjustment records
- Administrative configuration
- Integration credentials

Users must not gain access to restricted fields merely because they can access the containing record.

### 7.5 Cross-Tenant Administrative Access

Platform-level operations that legitimately access multiple tenants must be explicitly authorized.

Requirements:

- Dedicated permissions
- Restricted administrative identities
- Strong authentication
- Justification for sensitive access where appropriate
- Audit logging
- Narrowly scoped queries and exports
- Appropriate approval or time-limited access for high-risk operations

Platform administrators must not receive unrestricted tenant-data access merely because they have operational responsibilities.

## 8. PostgreSQL Tenant Isolation

PostgreSQL must enforce tenant-aware access through application-level scoping and database-level safeguards where practical.

### 8.1 Tenant Columns

Tenant-scoped relational tables should include a non-null tenant identifier.

Example:

```sql
CREATE TABLE orders (
    id UUID PRIMARY KEY,
    tenant_id UUID NOT NULL,
    user_id UUID NOT NULL,
    status TEXT NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

The tenant identifier must be populated from trusted server-side context, not accepted blindly from client input.

### 8.2 Tenant-Scoped Queries

Every tenant-scoped query must include the tenant constraint.

Example:

```sql
SELECT id, status, created_at
FROM orders
WHERE tenant_id = $1
  AND id = $2;
```

The tenant ID must come from validated context.

Avoid repository methods that permit tenant-scoped queries without a tenant parameter.

### 8.3 Tenant-Scoped Relationships

Foreign key relationships must prevent invalid cross-tenant associations wherever practical.

For tenant-owned entities, consider composite uniqueness and foreign key constraints using both resource ID and tenant ID.

Example:

```sql
ALTER TABLE food_courts
ADD CONSTRAINT food_courts_tenant_id_id_unique
UNIQUE (tenant_id, id);

ALTER TABLE orders
ADD CONSTRAINT orders_food_court_tenant_fk
FOREIGN KEY (tenant_id, food_court_id)
REFERENCES food_courts (tenant_id, id);
```

This prevents an order from referencing a food court owned by a different tenant, even if the application layer contains a defect.

### 8.4 Row-Level Security

PostgreSQL Row-Level Security (RLS) should be considered as an additional defense-in-depth control for tenant-scoped tables.

Where enabled:

- Define explicit tenant access policies.
- Set tenant context securely for each transaction or database session.
- Ensure connection pooling does not leak tenant context between requests.
- Reset session-level context safely.
- Use restrictive policies for reads and writes.
- Test behavior for missing or invalid tenant context.
- Ensure privileged database roles do not bypass RLS unintentionally.

Example policy pattern:

```sql
ALTER TABLE orders ENABLE ROW LEVEL SECURITY;
ALTER TABLE orders FORCE ROW LEVEL SECURITY;

CREATE POLICY orders_tenant_isolation
ON orders
USING (
    tenant_id = current_setting('app.tenant_id', true)::uuid
)
WITH CHECK (
    tenant_id = current_setting('app.tenant_id', true)::uuid
);
```

The application must establish the tenant context using trusted values inside the appropriate transaction.

RLS must not be treated as a replacement for application-level authorization. Database owners, roles with `BYPASSRLS`, and other privileged execution paths require explicit review.

### 8.5 Connection Pool Safety

Tenant context must not leak across pooled connections.

Requirements:

- Prefer transaction-scoped tenant settings.
- Set tenant context within the transaction that uses it.
- Avoid persistent session settings unless they are reliably reset.
- Roll back failed transactions.
- Test reuse of connections across different tenant requests.
- Ensure pooled connections cannot retain stale authorization state.

### 8.6 Database Roles

Separate:

- Runtime application roles
- Migration roles
- Administrative roles
- Reporting or read-only roles
- Backup and recovery roles

Runtime roles must not have unnecessary schema, ownership, or bypass privileges.

### 8.7 Tenant-Scoped Uniqueness

Uniqueness must reflect the actual business rule.

If an identifier is unique only within a tenant, use a tenant-scoped unique constraint.

Example:

```sql
CREATE UNIQUE INDEX vendors_tenant_slug_unique
ON vendors (tenant_id, slug);
```

Do not impose global uniqueness where it is not required by the domain, and do not omit tenant scope where it is necessary to preserve isolation.

## 9. MongoDB Tenant Isolation

MongoDB collections containing tenant-scoped records must enforce explicit tenant ownership.

### 9.1 Document Structure

Tenant-scoped documents must include a tenant identifier.

Example:

```json
{
  "_id": "order-id",
  "tenantId": "tenant-id",
  "userId": "user-id",
  "status": "pending"
}
```

The tenant ID must be derived from validated server-side context.

### 9.2 Tenant-Scoped Queries

Every tenant-scoped query must include a tenant filter.

Example:

```typescript
const order = await Order.findOne({
  _id: orderId,
  tenantId: tenantContext.tenantId,
});
```

Avoid queries that use only a document ID when the resource is tenant-scoped.

### 9.3 Update and Delete Operations

Updates and deletes must include tenant constraints.

Example:

```typescript
const result = await Order.updateOne(
  {
    _id: orderId,
    tenantId: tenantContext.tenantId,
  },
  {
    $set: {
      status: validatedStatus,
    },
  }
);
```

Do not fetch a resource, perform an independent authorization check, and later update it by ID alone when the tenant or ownership could change between operations.

Use atomic conditional updates or transactions where required.

### 9.4 Aggregation Pipelines

Tenant filtering must be applied at the earliest safe stage of an aggregation pipeline.

Requirements:

- Apply validated tenant filters before sensitive transformations.
- Validate lookup relationships.
- Prevent client-controlled aggregation operators.
- Ensure `$lookup`, `$unionWith`, and related stages cannot introduce cross-tenant data.
- Review grouping and export pipelines for tenant leakage.
- Restrict access to raw aggregation interfaces.

### 9.5 Indexes

Tenant-scoped indexes must reflect actual query patterns.

Examples:

```javascript
db.orders.createIndex({
  tenantId: 1,
  createdAt: -1
});

db.vendors.createIndex(
  {
    tenantId: 1,
    slug: 1
  },
  {
    unique: true
  }
);
```

Indexes improve query efficiency but do not enforce authorization by themselves.

### 9.6 Transactions and Sharding

When tenant-scoped operations span documents or collections:

- Use appropriate atomic operations or transactions.
- Validate tenant ownership for every participating record.
- Avoid cross-tenant references unless explicitly supported.
- Design shard keys with tenant access patterns and isolation requirements in mind.
- Avoid tenant identifiers being inferred from untrusted document fields.

## 10. Redis Tenant Isolation

Redis is shared infrastructure and must enforce tenant-aware key design and access policies.

### 10.1 Key Naming

Tenant-specific keys must include the tenant identifier.

Example:

```text
kampyn:tenant:{tenant_id}:orders:{order_id}
kampyn:tenant:{tenant_id}:inventory:{item_id}
kampyn:tenant:{tenant_id}:sessions:{session_id}
kampyn:tenant:{tenant_id}:analytics:{metric}
```

The format must be consistent across services.

### 10.2 Tenant Scope

All tenant-specific reads, writes, deletes, and invalidation operations must be scoped to the validated tenant context.

Never allow a client to submit arbitrary Redis keys or commands.

### 10.3 Cache Isolation

- Include tenant scope in cache keys.
- Include user or permission scope when the cached value is user-specific.
- Avoid caching sensitive data globally.
- Prevent tenant IDs from being omitted in shared cache abstractions.
- Ensure cache invalidation cannot remove or alter unrelated tenant data.
- Revalidate authorization before serving protected cache results.

### 10.4 Sessions

Session records must be bound to the authenticated identity and relevant tenant context.

Where sessions can operate across multiple tenants, tenant selection must be separately validated.

Revoked memberships and sessions must not retain access through stale Redis records.

### 10.5 Redis Administration

Application services must use narrowly scoped Redis access.

Where practical:

- Separate Redis users by service.
- Restrict command access.
- Apply network-level controls.
- Restrict administrative operations.
- Avoid shared administrative credentials.

Redis key prefixes are organizational conventions, not an authorization boundary. Access to the Redis server itself must also be restricted.

## 11. OpenSearch Tenant Isolation

OpenSearch must preserve tenant boundaries for indexing, querying, aggregations, and document retrieval.

### 11.1 Document Ownership

Every tenant-scoped indexed document must include a validated tenant identifier.

Example:

```json
{
  "documentId": "item-id",
  "tenantId": "tenant-id",
  "name": "Example Item",
  "category": "Food"
}
```

### 11.2 Query Enforcement

Tenant filters must be injected server-side into every tenant-scoped query.

Requirements:

- Never rely on frontend-provided filters.
- Prevent client-controlled query structures from removing tenant constraints.
- Apply tenant filtering to aggregations and nested queries.
- Validate tenant scope during direct document retrieval.
- Apply tenant scope to autocomplete, suggestions, and faceted search.
- Validate tenant scope during exports and bulk operations.

### 11.3 Indexing

Indexing workers must validate the tenant ownership of each source record.

Documents must not be indexed under a tenant ID taken from an untrusted event or payload without verification.

### 11.4 Updates and Deletes

OpenSearch update and delete operations must validate document ownership.

Bulk operations must preserve tenant scope for every document in the batch.

### 11.5 Search Security

Search must not reveal another tenant's:

- Item names
- Vendor details
- Order metadata
- Complaint information
- User profiles
- Analytics
- Internal operational records

Search relevance, facets, counts, autocomplete, and error behavior must also be evaluated for potential information leakage.

### 11.6 Index-Level Isolation

Separate indexes or index aliases may be used for specific tenant isolation requirements, scale, or operational boundaries.

Index separation does not remove the need for authorization and access controls.

## 12. Object Storage and File Isolation

All tenant-owned files must have enforceable tenant ownership and access restrictions.

### 12.1 Object Naming

Object paths should use controlled tenant-scoped namespaces.

Example:

```text
tenants/{tenant_id}/users/{user_id}/profile/{file_id}
tenants/{tenant_id}/orders/{order_id}/attachments/{file_id}
tenants/{tenant_id}/complaints/{complaint_id}/{file_id}
```

Object names must not be treated as authorization credentials.

### 12.2 Upload Security

- Validate tenant membership and upload permissions.
- Derive tenant scope from trusted server context.
- Generate server-controlled object identifiers.
- Validate content type and file format.
- Enforce file size and resource limits.
- Prevent path traversal.
- Restrict write permissions to authorized services.
- Ensure metadata cannot override tenant ownership.

### 12.3 Download Security

Before serving or generating access to a file:

- Validate the authenticated principal.
- Verify tenant membership.
- Verify ownership or access permissions.
- Confirm that the file belongs to the expected tenant.
- Check the requested operation's permissions.

### 12.4 Signed URLs

Signed URLs must:

- Be short-lived.
- Be scoped to a specific object and operation.
- Be generated only after authorization.
- Avoid exposing broad storage permissions.
- Use restrictive content-disposition and content-type settings where appropriate.
- Be revoked or invalidated where supported if access is compromised.

### 12.5 Deletion

File deletion must verify tenant ownership and must account for relevant database references, caches, and derived objects.

## 13. API Tenant Isolation

Every tenant-aware API must enforce tenant context and resource ownership.

### 13.1 Route Design

Tenant-aware routes may include tenant identifiers for routing or clarity, but those identifiers must be validated against the authenticated caller's permitted scope.

Example:

```text
/api/v1/tenants/{tenantId}/orders/{orderId}
```

The path's `tenantId` must never independently authorize access.

### 13.2 Request Validation

API validation must explicitly separate:

- Tenant selection
- Authentication
- Membership verification
- Resource ownership
- Operation permissions
- Business validation

### 13.3 Response Filtering

Responses must include only the fields the caller is permitted to access.

Avoid returning entire database documents or internal service models.

### 13.4 Error Consistency

Unauthorized cross-tenant requests must not reveal whether sensitive resources exist in another tenant.

Use safe and consistent responses, such as a generic not-found response where appropriate to the API's disclosure model.

### 13.5 Bulk APIs

Bulk endpoints must validate tenant scope for every item.

A single authorized item must not cause unrelated unauthorized items to be processed.

Bulk failure behavior must be clearly defined to prevent partial unauthorized modifications.

## 14. Frontend Tenant Isolation

The frontend must preserve tenant context for user experience while treating all client-side state as untrusted.

### 14.1 Tenant Selection

Tenant selection must be validated by the server.

The frontend must not treat a selected tenant ID as proof of membership or permission.

### 14.2 Routing

Tenant-specific routes must not expose protected content before server-side authorization.

Dynamic route segments must be validated and passed to trusted server-side authorization logic.

### 14.3 TanStack Query

Tenant-scoped query keys must include tenant identity and any other required authorization scope.

Example:

```typescript
const queryKey = [
  "tenant",
  tenantId,
  "orders",
  filters,
];
```

Requirements:

- Use validated tenant context for protected requests.
- Prevent query key collisions across tenants.
- Invalidate or remove sensitive cached data when switching tenants.
- Clear relevant caches on logout.
- Revalidate protected data after permission changes.
- Avoid rendering stale data while tenant context is unresolved.

### 14.4 Zustand

Zustand may store selected tenant UI state, but it must not be treated as an authorization authority.

Requirements:

- Do not store privileged credentials in persistent Zustand state.
- Clear tenant-specific state when the active tenant changes.
- Avoid sharing private state across authenticated sessions.
- Prevent stale tenant context from controlling protected mutations.

### 14.5 Server Components and Caching

Next.js server-side rendering and caching must not share personalized or tenant-specific content across tenants.

Requirements:

- Verify authorization before rendering protected content.
- Use appropriate request-aware cache policies.
- Avoid caching personalized responses as globally reusable data.
- Include tenant and permission scope in explicitly tenant-aware caching designs.
- Review `fetch` caching, route caching, revalidation, and static rendering behavior for data leakage.
- Ensure server-side errors do not reveal tenant data.

### 14.6 UI Restrictions

Hiding buttons, navigation links, or pages based on roles is a usability feature only.

The backend must independently authorize every protected operation.

## 15. Tenant Isolation in Events and Queues

Asynchronous systems must preserve tenant scope beyond the original request lifecycle.

### 15.1 Event Metadata

Tenant-scoped events should include explicit tenant metadata.

Example:

```json
{
  "eventId": "event-id",
  "eventType": "order.created",
  "tenantId": "tenant-id",
  "aggregateId": "order-id",
  "occurredAt": "2026-01-01T12:00:00Z"
}
```

The event schema must be validated at publication and consumption boundaries.

### 15.2 Producer Validation

Event producers must:

- Derive tenant context from a trusted source.
- Validate resource ownership.
- Publish only authorized event types.
- Avoid including unnecessary personal or confidential information.
- Prevent tenant metadata from being arbitrarily overridden.

### 15.3 Consumer Validation

Consumers must:

- Validate event structure.
- Verify trusted producer identity where applicable.
- Establish tenant context safely.
- Confirm source-resource ownership when necessary.
- Enforce tenant scope on database and external operations.
- Handle retries and duplicate delivery idempotently.

### 15.4 Dead-Letter Queues

Dead-letter queues may contain tenant data and must be treated as sensitive.

Access must be restricted, retention defined, and replay operations authorized.

### 15.5 Event Replay

Replay tools must preserve the original tenant scope and validate current permissions before executing sensitive side effects.

An event must not be replayed under an arbitrary tenant context.

## 16. Background Workers and Scheduled Jobs

Background workers must explicitly establish and validate tenant scope.

### 16.1 Tenant-Specific Jobs

Each tenant-specific job must carry a validated tenant identifier and the minimum required operation context.

Job payloads must not be treated as proof of authorization.

### 16.2 Scheduled Jobs

For jobs processing multiple tenants:

- Discover eligible tenants through trusted platform-level access.
- Process each tenant within an explicit tenant context.
- Isolate failures between tenants where practical.
- Prevent shared mutable state from leaking tenant data.
- Apply tenant-specific limits and configuration.
- Record outcomes using tenant-safe observability.

### 16.3 Retry Safety

Retries must preserve tenant scope and must not duplicate sensitive side effects.

Workers must validate that the tenant and resource remain eligible for processing before executing delayed operations.

### 16.4 Worker Identity

Each worker must use a dedicated service identity with the minimum required permissions.

Platform-wide jobs must receive explicit platform-level authorization rather than bypassing tenant checks implicitly.

## 17. Real-Time Communication Isolation

WebSocket connections, subscriptions, and realtime event delivery must enforce tenant boundaries.

### 17.1 Connection Establishment

On connection:

- Authenticate the user or service.
- Resolve the active tenant context.
- Verify membership and connection permissions.
- Establish a server-controlled tenant scope.
- Apply connection limits and rate controls.

### 17.2 Channels and Rooms

Tenant-specific rooms must use server-controlled identifiers.

Before subscribing or publishing:

- Validate tenant context.
- Verify room ownership.
- Confirm membership or resource access.
- Check the required action permission.

### 17.3 Permission Changes

Membership suspension, tenant switching, role changes, and access revocation must trigger appropriate revalidation or connection cleanup.

### 17.4 Event Delivery

Events must be delivered only to authorized recipients within the intended tenant and resource scope.

Never broadcast tenant-private events to globally shared channels.

## 18. Tenant Isolation in Payments

Payment processing must preserve tenant boundaries across orders, payment intents, refunds, settlements, and financial reporting.

Requirements:

- Associate each payment record with its tenant.
- Validate order ownership and tenant scope before payment initiation.
- Resolve merchant or provider configuration from trusted tenant settings.
- Prevent client-supplied tenant IDs from selecting unauthorized payment credentials.
- Verify payment provider callbacks and webhook signatures.
- Ensure webhook events are associated with the expected tenant and transaction.
- Scope refunds to authorized tenant-owned transactions.
- Prevent cross-tenant settlement or commission calculations.
- Isolate financial reports and exports by tenant.
- Audit privileged financial operations.

Payment credentials must be stored and accessed according to `.ai/security/secrets.md` and `.ai/security/payments.md`.

## 19. Tenant Isolation in Analytics and Reporting

Analytics and reporting systems must not combine or disclose tenant data unintentionally.

### 19.1 Tenant-Scoped Analytics

Every tenant-specific analytical query must be scoped to the validated tenant.

This includes:

- Order volume
- Revenue
- Vendor performance
- Inventory movement
- Booking utilization
- Complaints
- Facility usage
- User engagement
- Operational metrics

### 19.2 Aggregation

Cross-tenant aggregation must be limited to explicitly authorized platform-level operations.

Where aggregate reporting is permitted, it must avoid exposing tenant-specific sensitive details unless specifically authorized.

### 19.3 Exports

Exports must enforce the same authorization rules as interactive queries.

Requirements:

- Verify export permissions.
- Scope every exported record.
- Restrict file access.
- Apply retention limits.
- Audit sensitive exports.
- Prevent cross-tenant records from being combined accidentally.

### 19.4 Data Warehousing

Replicated analytical datasets must preserve tenant ownership metadata and access boundaries.

ETL pipelines must validate source tenant scope and ensure transformations do not introduce cross-tenant leakage.

## 20. Tenant Isolation in Logs and Observability

Tenant metadata is useful for tracing and investigation, but observability systems must not become a cross-tenant data exposure path.

### 20.1 Tenant Metadata

Logs, traces, and metrics may include a tenant identifier where operationally necessary.

Tenant metadata must be derived from validated context, not blindly copied from user input.

### 20.2 Sensitive Data

Do not log:

- Authentication tokens
- Passwords
- Secret values
- Payment credentials
- Private messages unless explicitly required and properly protected
- Unnecessary personal information
- Confidential tenant configuration

### 20.3 Access Controls

Observability access must be restricted according to operational responsibilities.

Tenant administrators must not receive access to other tenants' private logs, traces, or diagnostic records.

### 20.4 Metrics

Avoid high-cardinality tenant labels that could create operational instability.

Tenant-level metrics must be collected and exposed according to a documented privacy and access model.

## 21. Tenant Configuration Security

Each tenant may have configurable settings for its operational environment.

Examples include:

- Branding
- Food court configuration
- Facility policies
- Booking rules
- Notification preferences
- Integration endpoints
- Payment configuration
- Feature availability
- User access policies

Requirements:

- Validate all tenant configuration inputs.
- Enforce tenant-specific administrative permissions.
- Separate platform-wide configuration from tenant-level settings.
- Restrict secrets and sensitive integration values.
- Audit high-impact configuration changes.
- Prevent one tenant from modifying another tenant's configuration.
- Validate configuration before applying it to runtime services.
- Avoid using tenant-controlled configuration to disable platform security controls.

Tenant-specific feature flags must not grant access to restricted functionality without corresponding authorization.

## 22. Tenant Lifecycle and Deprovisioning

Tenant isolation must remain enforced during onboarding, suspension, reactivation, and deletion.

### 22.1 Onboarding

Tenant provisioning must:

- Create a unique tenant identifier.
- Initialize required configuration and resources.
- Establish authorized initial administrators.
- Create tenant-scoped data structures where required.
- Configure tenant integrations securely.
- Ensure no inherited credentials or records from another tenant are exposed.

### 22.2 Suspension

Suspended tenants must have their access restricted according to documented business and legal requirements.

Suspension must consider:

- User authentication and membership
- API access
- Active sessions
- Realtime connections
- Background jobs
- External integrations
- Payment and settlement workflows
- Data retention and export requirements

Suspension must not unintentionally expose tenant data or delete records that must be retained.

### 22.3 Reactivation

Reactivation must validate tenant status, configuration, memberships, and credentials before restoring access.

Expired or revoked integrations must not be reactivated automatically.

### 22.4 Deprovisioning

Tenant deprovisioning must:

- Revoke tenant-specific credentials and integrations.
- Disable memberships and sessions as required.
- Stop tenant-specific jobs.
- Remove or invalidate tenant-specific caches.
- Remove tenant-specific search data where required.
- Restrict access to retained records.
- Apply documented data retention and deletion requirements.
- Preserve necessary audit records.
- Ensure backups and recovery systems continue to respect the tenant's data lifecycle.

Deletion must account for all relevant data stores and derived datasets.

## 23. Cross-Tenant Platform Operations

Some platform-level services legitimately process information across multiple tenants.

Examples include:

- Tenant provisioning
- Global service health
- Platform-wide security monitoring
- Tenant billing administration
- Platform-level analytics
- Tenant lifecycle management
- Authorized incident investigation

These operations must use dedicated platform-level permissions.

Requirements:

- Define the purpose and permitted data scope.
- Use dedicated service identities or administrative roles.
- Restrict data access to the minimum required.
- Separate operational access from routine application identities.
- Record sensitive operations in audit logs.
- Avoid unrestricted data access for convenience.
- Prevent platform-level privileges from leaking into ordinary tenant workflows.

Cross-tenant functionality must be implemented as an explicit capability, not as an accidental consequence of omitting tenant filters.

## 24. Tenant-Aware SDK Design

KAMPYN SDKs must make tenant context explicit while preventing clients from controlling privileged authorization decisions.

### 24.1 Client SDKs

Browser and mobile SDKs must:

- Use public configuration only.
- Obtain authentication credentials through supported secure flows.
- Select tenants only within the authenticated user's authorized memberships.
- Never contain platform-wide secrets.
- Avoid treating local tenant state as authorization proof.
- Handle tenant switching by clearing or invalidating sensitive local state.

### 24.2 Server SDKs

Server-side SDKs may support privileged tenant-scoped operations through secure credential configuration.

Requirements:

- Keep privileged credentials server-side.
- Validate tenant scope for every operation.
- Use dedicated service identities.
- Restrict administrative capabilities.
- Provide typed tenant-aware interfaces.
- Avoid global mutable tenant context.
- Document cross-tenant operations separately.

### 24.3 SDK Contracts

SDK methods must make tenant scope clear in their contract, either through an explicitly validated client context or a required tenant-aware request context.

The backend must always independently verify the caller's access.

## 25. Self-Hosted University Isolation

KAMPYN supports university self-hosting, where an institution may operate its own deployment.

A standalone university installation may represent a single logical tenant, but must still preserve isolation between users, roles, departments, vendors, and restricted operational domains.

Requirements:

- Clearly define the installation's tenant model.
- Prevent users from changing tenant identifiers to access unauthorized data.
- Apply the same authorization and resource ownership checks used in SaaS deployments.
- Protect secrets using the university's approved secret management system.
- Restrict administrative access.
- Enforce secure defaults in installation templates.
- Document backup, restore, and deprovisioning procedures.
- Provide secure configuration for integrations and institutional identity systems.

If a university operates multiple campuses or organizational divisions as separate security scopes, their relationships and permitted cross-scope access must be explicitly configured.

Self-hosting must not disable core tenant isolation controls.

## 26. Deployment and Infrastructure Isolation

Infrastructure must reinforce tenant isolation where practical.

### 26.1 SaaS Infrastructure

Use appropriate infrastructure boundaries for:

- Runtime identities
- Service accounts
- Database access
- Cache access
- Search access
- Object storage
- Network connectivity
- Secrets
- Monitoring systems
- Deployment pipelines

Infrastructure separation may be logical or physical depending on tenant risk, scale, compliance, and contractual requirements.

### 26.2 Kubernetes

Where Kubernetes is used:

- Separate workloads by namespace or another documented isolation model.
- Apply least-privilege RBAC.
- Restrict service account permissions.
- Apply network policies where supported.
- Avoid sharing secrets unnecessarily.
- Restrict privileged containers.
- Apply resource quotas and limits.
- Secure admission and deployment controls.
- Restrict access to cluster administration.
- Prevent tenant-controlled workloads from gaining platform privileges.

Namespaces alone are not a complete security boundary.

### 26.3 Network Isolation

Network controls should limit service communication to required destinations.

Requirements:

- Restrict database and cache access to approved workloads.
- Restrict administrative endpoints.
- Apply outbound network controls where feasible.
- Prevent tenant-controlled input from reaching internal infrastructure through server-side request forgery.
- Protect management and monitoring interfaces.
- Separate production and non-production networks where practical.

### 26.4 Resource Isolation

Tenant-level resource limits should be applied where needed to reduce noisy-neighbor risks.

Consider limits for:

- API requests
- Concurrent operations
- File storage
- Upload sizes
- Background job throughput
- Search workloads
- Realtime connections
- Analytics queries

Resource controls must prevent one tenant's workload from significantly degrading other tenants' availability.

## 27. Isolation Strategy and Deployment Models

KAMPYN may use different isolation strategies depending on the deployment model and workload.

| Strategy | Description | Considerations |
|---|---|---|
| Shared application, shared database | Tenants share application instances and database structures with tenant-scoped records | Efficient, requires strong query and authorization controls |
| Shared application, separate schemas | Tenants share application instances but have separate database schemas | More structural separation, increased migration complexity |
| Shared application, separate databases | Tenants share application instances but use dedicated databases | Stronger data separation, increased operational complexity |
| Dedicated application and infrastructure | Tenant receives isolated application and infrastructure resources | Higher isolation potential, greater cost and operational overhead |
| University self-hosted | Institution operates its own deployment | Institution controls infrastructure and operational security |

The chosen strategy must be documented and justified based on security requirements, scale, operational constraints, data residency, and contractual commitments.

No deployment model eliminates the need for authentication, authorization, secure configuration, and correct tenant context handling.

## 28. Tenant-Aware Repository Design

Repositories must make tenant scoping difficult to omit accidentally.

### 28.1 Explicit Tenant Parameters

Tenant-scoped repository methods should require a validated tenant context or tenant identifier from a trusted internal boundary.

Example:

```go
type OrderRepository interface {
    GetByID(
        ctx context.Context,
        tenantID TenantID,
        orderID OrderID,
    ) (Order, error)
}
```

The repository must not accept tenant IDs directly from untrusted request data.

### 28.2 Scoped Repository Interfaces

Where practical, provide tenant-scoped repository abstractions that bind a validated tenant context to a controlled data-access scope.

These abstractions must not become a way to bypass authorization or introduce mutable cross-request state.

### 28.3 Avoid Unscoped Methods

Avoid public repository methods such as:

```go
GetOrderByID(ctx, orderID)
```

for tenant-scoped resources unless they are explicitly restricted to a narrowly authorized platform-level use case.

Cross-tenant administrative queries must be separated and named to make their privileged behavior obvious.

### 28.4 Centralized Enforcement

Shared repository helpers and middleware should reduce repeated tenant filtering logic.

However, centralization must remain transparent enough for reviewers to verify that each query is correctly scoped.

Do not hide tenant enforcement inside overly generic abstractions that make security behavior difficult to reason about.

## 29. Data Migration and Tenant Reassignment

Tenant-scoped data migrations must preserve ownership and isolation.

### 29.1 Migration Requirements

- Include tenant ownership in schema changes.
- Backfill tenant IDs using trusted mapping sources.
- Reject ambiguous ownership.
- Validate tenant references before migration.
- Verify record counts and ownership after migration.
- Preserve constraints during migration where practical.
- Prevent partially migrated records from becoming accessible.
- Maintain a documented recovery strategy.

### 29.2 Tenant Reassignment

Moving a resource from one tenant to another is a privileged operation.

Requirements:

- Explicit authorization.
- Verification of the source and destination tenants.
- Validation of related resource ownership.
- Impact assessment for references, caches, search indexes, events, and files.
- Transactional or controlled multi-step execution.
- Audit logging.
- Revalidation of permissions after reassignment.

Tenant reassignment must not be exposed as an ordinary editable field in generic update endpoints.

## 30. Cache Invalidation and Permission Changes

Tenant isolation must remain correct when permissions or memberships change.

Relevant changes include:

- User removal
- Role modification
- Tenant suspension
- Resource ownership change
- Integration revocation
- Tenant switching
- Administrative permission updates

The system must invalidate or revalidate relevant:

- TanStack Query caches
- Zustand tenant-specific state
- Redis records
- Server-side caches
- Search access state
- WebSocket subscriptions
- Background job eligibility

Cache invalidation must be designed for eventual consistency, and sensitive operations must perform fresh authorization checks when required.

## 31. Concurrency and Tenant Isolation

Concurrency must not allow tenant context or authorization state to leak between operations.

Requirements:

- Avoid mutable shared tenant state.
- Bind tenant context to each request, transaction, job, or connection.
- Prevent context reuse across goroutines or asynchronous tasks without explicit propagation.
- Ensure database transactions use the correct tenant scope.
- Validate ownership within atomic updates where necessary.
- Prevent tenant context from changing during sensitive operations.
- Ensure worker pools reset request-specific state between jobs.
- Test concurrent operations involving multiple tenants.

Shared connection pools, worker pools, and application instances must be assumed to process requests from multiple tenants concurrently.

## 32. Tenant Isolation Testing

Tenant isolation must be tested as a first-class security property.

### 32.1 Cross-Tenant Access Tests

For each tenant-scoped resource, verify that:

- Tenant A can access its authorized resources.
- Tenant A cannot read Tenant B's resources.
- Tenant A cannot modify Tenant B's resources.
- Tenant A cannot delete Tenant B's resources.
- Tenant A cannot infer sensitive existence or metadata beyond the documented disclosure model.
- Tenant A cannot access Tenant B's files, search results, caches, events, or analytics.
- Tenant A cannot use Tenant B's integration credentials.
- Tenant A cannot modify Tenant B's configuration.

### 32.2 Authentication and Membership Tests

Test:

- Users without tenant membership.
- Suspended users.
- Revoked memberships.
- Expired invitations.
- Invalid tenant selection.
- Stale tokens after membership changes.
- Users with multiple tenant memberships.
- Users switching tenants.
- Service identities with limited tenant scope.

### 32.3 Data Store Tests

Test tenant isolation independently across:

- PostgreSQL
- MongoDB
- Redis
- OpenSearch
- Object storage
- Analytics datasets
- Search indexes
- Backups and exports, where applicable

### 32.4 API Tests

Test:

- URL tenant ID tampering
- Header tenant ID tampering
- Query tenant ID tampering
- Body tenant ID tampering
- Mass assignment of tenant fields
- Bulk operations containing mixed-tenant records
- Unauthorized administrative endpoints
- Unscoped repository access

### 32.5 Asynchronous Tests

Test:

- Forged or malformed event tenant metadata
- Cross-tenant job payload manipulation
- Retry behavior
- Dead-letter queue replay
- Delayed jobs after membership revocation
- Concurrent workers processing different tenants
- Event delivery to unauthorized realtime subscribers

### 32.6 Cache and Search Tests

Test:

- Cache key collisions
- Tenant switching with stale frontend data
- Permission changes with cached data
- Tenant filter removal attempts
- Aggregation and facet leakage
- Autocomplete leakage
- Bulk search and export isolation

### 32.7 Concurrency Tests

Run concurrent operations across multiple tenants to detect:

- Context leakage
- Incorrect transaction scoping
- Cache collisions
- Shared mutable state
- Race conditions in authorization
- Cross-tenant updates
- Worker state reuse

### 32.8 Regression Tests

Every confirmed cross-tenant vulnerability must result in a regression test where feasible.

Tests must demonstrate that the vulnerable access path is blocked and that related data-access paths remain tenant-scoped.

## 33. Monitoring and Detection

Tenant isolation violations must be detectable through security monitoring.

Monitor for:

- Repeated cross-tenant authorization failures
- Unusual tenant switching activity
- Access to inactive or suspended tenants
- Unexpected platform-level queries
- Abnormal tenant data export volume
- Suspicious tenant-specific secret access
- Search requests with invalid tenant scope
- Unexpected changes in tenant ownership
- Unauthorized membership or role changes
- Cross-tenant event publication or delivery failures
- Repeated attempts to manipulate tenant identifiers

Alerts should provide enough context for investigation without exposing confidential tenant information.

## 34. Incident Response for Tenant Isolation Violations

Any suspected cross-tenant access must be treated as a security incident.

Response steps:

1. Identify the affected tenant boundaries and resources.
2. Determine the vulnerable endpoint, service, query, cache, or processing path.
3. Contain the issue by restricting or disabling the affected operation where appropriate.
4. Prevent further unauthorized access.
5. Review access logs, audit events, traces, and relevant data-store records.
6. Determine the potential scope of data exposure or modification.
7. Identify affected tenants and data categories.
8. Restore data integrity where required.
9. Apply and test the permanent fix.
10. Rotate affected credentials if relevant.
11. Notify responsible stakeholders according to incident procedures and applicable obligations.
12. Add regression tests and monitoring to prevent recurrence.
13. Document root cause, impact, remediation, and follow-up actions.

Do not assume that an authorization error indicates no data exposure. Investigate the underlying access path and any potentially affected derived systems.

## 35. Performance and Scalability

Tenant isolation must remain enforceable as KAMPYN scales.

Requirements:

- Use tenant-aware indexes aligned with actual access patterns.
- Avoid unbounded cross-tenant scans.
- Apply pagination and resource limits.
- Use efficient authorization evaluation.
- Avoid unnecessary repeated membership lookups while maintaining appropriate freshness.
- Use safe, tenant-aware caching.
- Isolate expensive tenant workloads where necessary.
- Apply backpressure to tenant-specific jobs and event processing.
- Monitor tenant-level resource consumption.
- Test large tenants and high-concurrency multi-tenant workloads.

Performance optimizations must not remove tenant filters or weaken authorization guarantees.

## 36. Tenant Isolation and Data Residency

Where contractual or institutional requirements mandate specific data residency or processing boundaries, the deployment architecture must enforce them.

Requirements:

- Document where tenant data is stored and processed.
- Identify replication and backup locations.
- Restrict integrations to permitted destinations.
- Ensure analytics and observability follow applicable data boundaries.
- Apply tenant-specific infrastructure isolation where required.
- Validate residency assumptions during deployment and migration.

Logical tenant isolation must not be represented as physical or geographic isolation unless the infrastructure actually provides it.

## 37. Security Review Requirements

Changes that affect tenant isolation require focused security review.

Examples include:

- Tenant context resolution
- Authentication and membership changes
- Authorization policies
- Repository interfaces
- Database schemas and queries
- RLS policies
- Cache key design
- OpenSearch mappings and query builders
- File storage and signed URLs
- Event schemas and consumers
- Worker scheduling
- WebSocket channels
- Analytics and export pipelines
- Platform-level administrative features
- Tenant lifecycle workflows
- Infrastructure and deployment isolation

Reviewers must verify both the intended tenant access model and the failure modes that could bypass it.

## 38. Prohibited Practices

The following practices are strictly prohibited:

- Trusting a client-supplied tenant ID as proof of authorization.
- Performing tenant-scoped queries without validated tenant context.
- Returning resources based only on globally unique IDs without ownership checks.
- Using shared mutable tenant context across concurrent requests.
- Allowing frontend controls to act as the only tenant access restriction.
- Omitting tenant filters from cache or search operations.
- Allowing client-controlled search filters to override tenant scope.
- Processing tenant-specific events without validating tenant context.
- Replaying background jobs under arbitrary tenant identities.
- Sharing tenant secrets across unrelated tenants without an approved design.
- Returning private tenant data through logs, analytics, or error messages.
- Using unrestricted platform-level access for routine tenant operations.
- Performing tenant reassignment through generic update endpoints.
- Assuming namespaces, key prefixes, or unique identifiers alone provide tenant security.
- Allowing stale cached authorization to bypass required permission checks.
- Disabling tenant isolation controls for convenience, testing, or performance without an approved and tightly controlled procedure.
- Deleting tenant data without validating ownership and lifecycle requirements.

## 39. Exceptions

Tenant isolation exceptions must be rare, explicit, and time-limited.

Every exception must document:

- Business or technical justification
- Affected tenants and resources
- Data classification
- Security and privacy impact
- Access scope
- Compensating controls
- Monitoring and audit requirements
- Responsible owner
- Required approval
- Expiration date
- Remediation plan

Cross-tenant access must never be introduced implicitly through shared service logic or unscoped database queries.

Exceptions must be reviewed before expiration, and high-risk exceptions must receive additional security oversight.

## 40. Ownership and Responsibilities

| Responsibility | Owner |
|---|---|
| Tenant isolation architecture | Architecture / Engineering |
| Tenant identity and membership | Identity / Backend |
| Authorization policy | Security / Backend |
| Tenant-scoped repositories | Backend Engineering |
| Database isolation controls | Database / Platform Engineering |
| Cache and search isolation | Backend / Platform Engineering |
| Tenant-aware frontend state | Frontend Engineering |
| Event and worker isolation | Backend / Platform Engineering |
| Infrastructure boundaries | Platform / DevOps |
| Tenant lifecycle security | Platform / Backend |
| Security testing | Engineering / Security |
| Incident response | Security / Operations |
| Self-hosted deployment controls | Deployment administrator |

Every tenant-aware service must have an identified owner responsible for maintaining its isolation guarantees.

## 41. Review Checklist

### Tenant Identity and Authorization
- [ ] Tenant context is resolved from a trusted source.
- [ ] Tenant membership and status are validated.
- [ ] Every sensitive operation has server-side authorization.
- [ ] Resource ownership is verified.
- [ ] Tenant switching invalidates or refreshes relevant access state.
- [ ] Privileged cross-tenant operations are explicitly authorized and audited.

### Data Stores
- [ ] PostgreSQL queries are tenant-scoped.
- [ ] Database relationships prevent invalid cross-tenant references where practical.
- [ ] RLS is implemented where appropriate and correctly configured.
- [ ] MongoDB queries, updates, deletes, and aggregations are tenant-scoped.
- [ ] Redis keys and operations preserve tenant scope.
- [ ] OpenSearch queries and bulk operations enforce tenant filtering.
- [ ] Object storage access verifies tenant ownership.
- [ ] Analytics, exports, and backups follow tenant access and lifecycle rules.

### Application and Infrastructure
- [ ] Tenant context is explicitly propagated.
- [ ] Shared state cannot leak tenant information.
- [ ] Events and background jobs preserve tenant scope.
- [ ] Realtime subscriptions enforce tenant membership.
- [ ] API errors do not reveal sensitive cross-tenant information.
- [ ] Frontend caches and state are cleared or invalidated appropriately.
- [ ] Service identities are scoped to required operations.
- [ ] Infrastructure permissions and network boundaries are appropriately restricted.

### Testing and Operations
- [ ] Cross-tenant read, write, update, and delete tests pass.
- [ ] Tenant switching and membership revocation tests pass.
- [ ] Concurrent multi-tenant tests pass.
- [ ] Cache and search isolation tests pass.
- [ ] Event and worker isolation tests pass.
- [ ] Security monitoring is configured.
- [ ] Tenant lifecycle and recovery procedures are documented.
- [ ] Security-sensitive changes received appropriate review.

## 42. Definition of Done

Tenant isolation is considered complete when:

- Every tenant has a stable and immutable identity.
- Tenant context is resolved and validated through trusted mechanisms.
- Every tenant-scoped resource has an enforceable ownership relationship.
- Authentication and membership are verified before tenant access.
- Authorization is enforced at object, function, and property levels.
- Tenant scope is enforced across PostgreSQL, MongoDB, Redis, OpenSearch, and object storage.
- Frontend state, server-side caching, and realtime communication preserve tenant boundaries.
- Background jobs, events, integrations, and exports validate tenant context.
- Platform-level cross-tenant access is explicit, restricted, and auditable.
- Concurrency and connection pooling cannot leak tenant context.
- Tenant lifecycle operations preserve isolation and data retention requirements.
- Cross-tenant security tests and regression tests pass.
- Monitoring and incident response procedures are in place.
- Deployment-specific isolation requirements are documented.
- Relevant security reviews have been completed.

**Final rule:** Every tenant-scoped operation must be performed under a validated tenant context and independently enforce the caller's authorization and resource ownership. No identifier, cache entry, internal service, or infrastructure boundary may be trusted as a substitute for tenant isolation.
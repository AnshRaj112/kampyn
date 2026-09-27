# KAMPYN Multi-Tenancy Architecture

## 1. Purpose

This document defines the multi-tenancy architecture for KAMPYN.

KAMPYN is designed to support multiple universities or institutions from a common platform while maintaining strict isolation between tenants.

A tenant represents an independently operated KAMPYN institution or university.

The architecture must guarantee:

- Tenant isolation.
- Data isolation.
- Authorization isolation.
- Cache isolation.
- Search isolation.
- Storage isolation.
- Configuration isolation.
- Operational isolation where required.
- Clear tenant ownership.
- Safe tenant lifecycle management.

A tenant boundary is a security boundary.

---

# 2. Core Principle

The fundamental multi-tenancy model is:

```text
KAMPYN Platform
      │
      ├── Tenant A
      │     ├── Users
      │     ├── Food Courts
      │     ├── Orders
      │     ├── Bookings
      │     └── Configuration
      │
      ├── Tenant B
      │     ├── Users
      │     ├── Food Courts
      │     ├── Orders
      │     ├── Bookings
      │     └── Configuration
      │
      └── Tenant C
            ├── Users
            ├── Food Courts
            ├── Orders
            ├── Bookings
            └── Configuration
```

Tenant A must never be able to access Tenant B's data unless an explicitly authorized platform-level operation permits it.

---

# 3. Tenant Definition

A tenant is an independently governed KAMPYN environment.

A tenant may represent:

- University.
- College.
- Educational institution.
- Campus group.
- Institution-managed deployment.

A tenant should have a stable internal identifier.

Example:

```text
tenant_id
```

The tenant identifier must not depend on a mutable value such as:

- University name.
- Domain name.
- Email domain.
- Display name.

---

# 4. Tenant Identity

Tenant identity must be represented explicitly.

Example:

```text
Tenant
├── id
├── name
├── status
├── configuration
├── created_at
└── updated_at
```

The internal tenant ID should remain stable even when:

- Branding changes.
- Domain changes.
- Institution name changes.
- Authentication provider changes.

---

# 5. Tenant Context

Every tenant-scoped request should have an explicit tenant context.

Conceptually:

```text
Request
   ↓
Authentication
   ↓
Tenant Resolution
   ↓
Tenant Membership Validation
   ↓
Authorization
   ↓
Application
```

The tenant context should be established before tenant-owned business operations execute.

---

# 6. Tenant Resolution

Tenant resolution may use:

- Authenticated membership.
- Hostname.
- Subdomain.
- Explicit API tenant context.
- Institution configuration.

Example:

```text
kiit.kampyn.com
      ↓
Tenant Resolution
      ↓
KIIT Tenant
```

However, resolving a tenant from a hostname does not itself grant authorization.

The authenticated identity must still be authorized for that tenant.

---

# 7. Tenant Resolution Order

A typical request may follow:

```text
Incoming Request
      ↓
Identify Tenant Context
      ↓
Authenticate User
      ↓
Validate User Membership
      ↓
Authorize Action
      ↓
Execute Business Operation
```

The exact ordering may vary depending on the authentication mechanism.

The final authorization decision must always bind identity and tenant context together.

---

# 8. Tenant Context Object

The backend should maintain an explicit tenant context rather than passing raw tenant IDs through arbitrary function parameters.

Conceptually:

```text
TenantContext
├── tenant_id
├── user_id
├── membership_id
└── authorization_scope
```

Only the fields required by the current application architecture should be included.

Tenant context should be immutable during a request.

---

# 9. Tenant Context Trust

Tenant context must come from a trusted server-side resolution process.

Never trust:

```text
X-Tenant-ID
```

or:

```text
?tenant_id=...
```

merely because the client supplied it.

Client-provided tenant identifiers are input.

The backend must validate that the authenticated identity can access the requested tenant.

---

# 10. Authentication and Tenancy

Authentication identifies the user.

Multi-tenancy determines which institutional context the user is operating within.

The relationship is:

```text
Identity
   ↓
Tenant Membership
   ↓
Tenant Context
   ↓
Authorization
```

A user may belong to:

```text
Tenant A
Tenant B
```

if the platform explicitly supports multi-tenant memberships.

Membership must be explicit.

---

# 11. Tenant Membership

A membership represents a user's relationship with a tenant.

Conceptually:

```text
User
 │
 ├── Membership → Tenant A
 │
 └── Membership → Tenant B
```

Membership may contain:

```text
user_id
tenant_id
role
status
created_at
```

The exact permission model is defined by the authorization architecture.

---

# 12. Active Tenant

If a user belongs to multiple tenants, the application must have an explicit active tenant.

Example:

```text
User
 ├── KIIT
 ├── University B
 └── University C

Active Tenant
      ↓
KIIT
```

Switching tenants must trigger a new authorization context.

---

# 13. Tenant Switching

Tenant switching must:

1. Validate membership.
2. Establish the new tenant context.
3. Reset tenant-sensitive client state.
4. Refresh tenant-scoped server state.
5. Re-evaluate permissions.
6. Avoid leaking previous tenant data.

Cached frontend queries must include tenant identity where necessary.

---

# 14. Tenant Isolation Layers

Tenant isolation must exist at multiple layers.

```text
Authentication
      ↓
Authorization
      ↓
Application
      ↓
Database
      ↓
Cache
      ↓
Search
      ↓
Object Storage
      ↓
Events / Workers
```

A single missing layer must not become an easy path to cross-tenant access.

---

# 15. Database Tenant Isolation

Tenant-owned relational data should generally include an explicit tenant boundary.

Example:

```text
orders
├── id
├── tenant_id
├── user_id
├── status
└── created_at
```

Queries must always operate within the correct tenant scope.

Prefer repositories that require tenant context rather than relying on individual developers to remember filters.

---

# 16. Repository Tenant Scoping

Prefer:

```text
OrderRepository
    ↓
ListByTenant(ctx, tenantID, ...)
```

or a repository initialized with a trusted tenant context.

Avoid unrestricted repository methods such as:

```text
GetOrder(id)
```

when the resource is inherently tenant-owned and the method makes tenant isolation easy to bypass.

---

# 17. Database-Level Isolation

KAMPYN may use different database isolation models depending on deployment requirements.

Possible models include:

```text
Shared Database
    ↓
Shared Tables
    ↓
tenant_id
```

or:

```text
Shared Database
    ↓
Separate Schemas
    ↓
Tenant
```

or:

```text
Separate Database
    ↓
Tenant
```

The appropriate model depends on:

- Scale.
- Security requirements.
- Institutional requirements.
- Operational cost.
- Self-hosting requirements.
- Compliance requirements.

---

# 18. Default Isolation Model

For a shared KAMPYN SaaS deployment, the default architecture should prefer logical tenant isolation through explicit tenant ownership and strong application/database controls unless a stronger physical isolation model is required.

The architecture must make it possible to move high-isolation tenants to dedicated infrastructure when justified.

---

# 19. Dedicated Tenant Infrastructure

Some institutions may require dedicated resources.

Possible topology:

```text
KAMPYN Platform
      │
      ├── Shared Tenant Infrastructure
      │
      ├── Tenant A
      │     └── Dedicated Database
      │
      └── Tenant B
            └── Dedicated Infrastructure
```

The application should not assume every tenant shares the same physical database.

Tenant routing must be abstracted from business logic.

---

# 20. Tenant Database Routing

If tenants may use different database infrastructure, introduce a database routing layer.

Conceptually:

```text
Tenant Context
      ↓
Tenant Database Resolver
      ↓
┌─────┴─────────┐
↓               ↓
Shared DB     Dedicated DB
```

The domain should not know which physical database hosts the tenant.

---

# 21. Tenant Configuration

Tenant-specific configuration should be stored separately from application-wide configuration.

Examples:

- Branding.
- Institution name.
- Timezone.
- Currency.
- Enabled features.
- Food court settings.
- Notification preferences.
- Authentication providers.
- Operational policies.

Conceptually:

```text
Platform Configuration
        +
Tenant Configuration
        ↓
Effective Configuration
```

---

# 22. Configuration Precedence

Where applicable:

```text
Platform Defaults
       ↓
Tenant Overrides
       ↓
Runtime Context
```

The precedence must be explicit.

Tenant configuration must not override security-critical platform constraints.

---

# 23. Tenant Feature Flags

Features may be enabled per tenant.

Example:

```text
Tenant A
  Community → enabled

Tenant B
  Community → disabled
```

Feature flags must not be treated as authorization.

A disabled feature should affect availability and UI behavior, while authorization remains separately enforced.

---

# 24. Tenant Branding

Tenant-specific branding may include:

- Logo.
- Colors.
- Name.
- Domain.
- Favicon.
- Email branding.
- Public content.

Branding must remain presentation/configuration data.

It must not affect authorization or tenant identity semantics.

---

# 25. Tenant Domains

KAMPYN may support:

```text
tenant.kampyn.com
```

or institution-controlled domains:

```text
kampyn.university.edu
```

Domain-to-tenant mappings must be stored explicitly.

Do not derive tenant identity from arbitrary string manipulation.

---

# 26. Tenant Domain Verification

Before an institution-controlled domain becomes active:

1. Domain ownership must be verified.
2. DNS configuration must be validated.
3. Tenant mapping must be created.
4. TLS must be configured.
5. Routing must be tested.

An unverified domain must not be allowed to claim an existing tenant.

---

# 27. Tenant Authentication Providers

Different tenants may use different identity providers.

Example:

```text
Tenant A
  → Google Workspace

Tenant B
  → Microsoft Entra ID

Tenant C
  → University OIDC
```

The authentication architecture must map external identities into the correct KAMPYN tenant membership.

---

# 28. Tenant Identity Linking

An external identity should be associated with:

```text
Provider
Provider Subject
Tenant
KAMPYN User
```

Do not assume the same email address automatically means the same identity across institutions.

---

# 29. Tenant Authorization

Authorization must always evaluate tenant scope.

Conceptually:

```text
Identity
   ↓
Tenant Membership
   ↓
Role / Permission
   ↓
Resource Tenant
   ↓
Action
   ↓
Decision
```

A valid permission in Tenant A must not automatically grant the same permission in Tenant B.

---

# 30. Tenant Administrators

Tenant administrators operate within their own institution.

A tenant administrator should generally be able to manage:

```text
Their tenant
```

but not:

```text
Other tenants
```

unless they also possess an explicitly defined platform-level administrative role.

---

# 31. Platform Administrators

Platform administrators may have cross-tenant capabilities.

These permissions must be:

- Explicit.
- Rare.
- Auditable.
- Strongly protected.
- Separately authorized.

Do not implement cross-tenant access by simply allowing:

```text
tenant_id = *
```

without an explicit authorization model.

---

# 32. Cross-Tenant Operations

Cross-tenant operations should be exceptional.

Examples:

- Platform-wide analytics.
- Platform administration.
- Infrastructure management.
- Billing administration.
- System health.

Such operations must explicitly identify:

```text
Actor
Purpose
Tenant Scope
Action
```

where auditability is required.

---

# 33. Tenant Data Access

Every data access path must preserve tenant scope.

This includes:

- API endpoints.
- Repositories.
- Background jobs.
- Event consumers.
- Search.
- Cache.
- File storage.
- Reports.
- Exports.
- Analytics.
- Administrative tools.

A tenant boundary must not disappear simply because the operation is asynchronous.

---

# 34. Tenant-Aware APIs

Tenant-owned API resources should resolve tenant context from trusted authentication and routing information.

Example:

```text
GET /api/v1/orders
```

may automatically operate within the active tenant.

Avoid requiring users to repeatedly provide tenant IDs for normal tenant-scoped operations when the authenticated context already establishes the tenant.

---

# 35. Explicit Tenant APIs

Some platform-level APIs may legitimately operate across tenants.

These APIs should make their elevated scope explicit.

Example:

```text
GET /api/v1/platform/tenants
```

rather than pretending the endpoint is an ordinary tenant-scoped resource.

---

# 36. Cache Isolation

Tenant-specific cache keys must include tenant scope.

Example:

```text
tenant:{tenant_id}:menu:{food_court_id}
```

not:

```text
menu:{food_court_id}
```

when identifiers may overlap between tenants.

See:

```text
architecture/caching.md
```

for detailed caching architecture.

---

# 37. Frontend Cache Isolation

TanStack Query keys must include tenant context when the result is tenant-specific.

Example:

```text
[
  "orders",
  tenantId,
  filters
]
```

When switching tenants:

- Invalidate tenant-specific queries.
- Prevent stale data display.
- Refresh tenant configuration.
- Reset tenant-sensitive Zustand state where required.

---

# 38. Search Isolation

OpenSearch documents must include tenant scope.

Example:

```text
{
  "tenant_id": "...",
  "food_court_id": "...",
  "name": "..."
}
```

Search queries must enforce tenant filtering.

Tenant filtering must not depend solely on frontend-provided filters.

---

# 39. Search Index Architecture

Tenant isolation may be implemented through:

```text
Shared Index
    ↓
tenant_id field
```

or:

```text
Separate Index
    ↓
Tenant
```

or:

```text
Separate Index Namespace
```

The choice depends on:

- Scale.
- Search volume.
- Isolation requirements.
- Operational complexity.

---

# 40. OpenSearch Security

Search access must be authorized independently from the database.

A user must not gain access to another tenant's indexed data simply because an OpenSearch query omitted a filter.

Search authorization must remain server-side.

---

# 41. Object Storage Isolation

Tenant-owned files should have tenant-scoped object keys.

Example:

```text
tenants/
  {tenant_id}/
    users/
    food/
    complaints/
    reports/
```

Avoid generic keys such as:

```text
uploads/{filename}
```

when tenant ownership matters.

---

# 42. Object Access

Object storage access should be authorized before generating access URLs.

A signed URL must not become an authorization bypass.

The backend should verify:

```text
User
   ↓
Tenant Membership
   ↓
Resource Ownership
   ↓
Object Access
```

before issuing access to a private object.

---

# 43. File Names

Original filenames must not be trusted as storage paths.

Use generated object identifiers.

Example:

```text
tenants/{tenant_id}/food/{object_id}
```

Store the original filename as metadata if required.

---

# 44. Events and Tenant Isolation

Tenant-scoped events should carry tenant context.

Example:

```text
OrderPlaced
├── event_id
├── tenant_id
├── aggregate_id
└── payload
```

Consumers must preserve tenant scope when:

- Updating search.
- Invalidating cache.
- Sending notifications.
- Writing analytics.
- Updating projections.

---

# 45. Event Consumers

A background worker must never assume that an event's tenant context is safe merely because the event came from internal infrastructure.

Validate:

- Event schema.
- Tenant ID.
- Resource ownership.
- Consumer authorization assumptions.

---

# 46. Tenant-Scoped Jobs

Background jobs should carry tenant context where applicable.

Example:

```text
GenerateReport
├── tenant_id
├── report_id
└── parameters
```

Workers must not accidentally process a tenant-scoped job using another tenant's configuration or credentials.

---

# 47. Scheduled Jobs

Platform-wide scheduled jobs may iterate through tenants.

Example:

```text
Daily Reconciliation
       ↓
Tenant A
Tenant B
Tenant C
```

Each tenant execution should have an explicit context.

Avoid one giant operation that mixes all tenant data without clear boundaries.

---

# 48. Tenant Isolation in Analytics

Analytics architecture must define whether metrics are:

```text
Tenant-specific
Platform-wide
```

Tenant-specific analytics must retain tenant scope.

Platform-wide analytics should be explicitly authorized and aggregated intentionally.

Avoid exposing cross-tenant raw data to tenant administrators.

---

# 49. Tenant Exports

Exports are security-sensitive.

Before generating an export:

```text
User
 ↓
Tenant Membership
 ↓
Export Permission
 ↓
Tenant Scope
 ↓
Generate Export
```

Generated files must inherit tenant ownership.

---

# 50. Tenant Reports

Reports should operate within the same tenant boundary as the underlying data.

If a report requires cross-tenant aggregation, it must use a platform-level authorization context.

Do not reuse ordinary tenant-scoped endpoints for platform-wide reporting.

---

# 51. Tenant Notifications

Notifications must preserve tenant context.

Example:

```text
OrderConfirmed
    ↓
Tenant A
    ↓
Tenant A notification configuration
    ↓
Tenant A user
```

Do not use another tenant's notification configuration or provider credentials accidentally.

---

# 52. Tenant-Specific Integrations

A tenant may have its own:

- Payment provider configuration.
- Email provider.
- SMS provider.
- Identity provider.
- University API.
- Object storage.
- Domain.

Integration configuration must therefore support tenant scope.

Example:

```text
Payment Provider
   ├── Tenant A credentials
   └── Tenant B credentials
```

See:

```text
architecture/integrations.md
```

for integration architecture.

---

# 53. Tenant Secrets

Tenant-specific credentials must be isolated.

Examples:

- University SSO secrets.
- Payment credentials.
- SMTP credentials.
- API keys.
- Webhook secrets.

Tenant administrators should not automatically receive raw secrets through ordinary application APIs.

---

# 54. Tenant-Specific Infrastructure

High-isolation tenants may use dedicated:

- Database.
- Redis.
- OpenSearch.
- Object storage.
- Event infrastructure.
- Kubernetes namespace.
- Entire deployment.

The platform should represent infrastructure placement as configuration rather than embedding it into domain logic.

---

# 55. Tenant Routing to Infrastructure

Conceptually:

```text
Tenant Context
      ↓
Infrastructure Resolver
      ↓
┌───────────────┬────────────────┐
↓               ↓
Shared          Dedicated
Infrastructure  Infrastructure
```

Application code should continue using the same domain interfaces.

---

# 56. Tenant Lifecycle

A tenant should have an explicit lifecycle.

Example:

```text
PROVISIONING
      ↓
ACTIVE
      ↓
SUSPENDED
      ↓
DECOMMISSIONING
      ↓
DELETED / ARCHIVED
```

Valid transitions must be defined.

Do not allow arbitrary status mutation.

---

# 57. Tenant Provisioning

Tenant provisioning may include:

```text
Create tenant
      ↓
Create configuration
      ↓
Configure domain
      ↓
Configure authentication
      ↓
Configure storage
      ↓
Configure search
      ↓
Configure integrations
      ↓
Create initial administrator
      ↓
Activate tenant
```

Provisioning should be repeatable and observable.

---

# 58. Tenant Provisioning Idempotency

Provisioning operations must be idempotent.

If provisioning fails halfway:

```text
Retry
   ↓
Continue / Repair
```

rather than:

```text
Retry
   ↓
Duplicate resources
```

Each provisioning step should have explicit completion state where necessary.

---

# 59. Tenant Suspension

A suspended tenant should have clearly defined behavior.

Potentially:

- Block normal user access.
- Block new transactions.
- Preserve data.
- Continue required system jobs.
- Allow authorized administrators to recover the tenant.

Suspension must not accidentally delete tenant data.

---

# 60. Tenant Decommissioning

Tenant decommissioning should be deliberate.

Potential process:

```text
Suspend
   ↓
Export / Backup
   ↓
Disable Integrations
   ↓
Stop New Writes
   ↓
Delete / Archive Derived Data
   ↓
Handle Object Storage
   ↓
Handle Database Data
   ↓
Remove Infrastructure
```

The exact sequence depends on retention requirements.

---

# 61. Tenant Deletion

Deletion must account for:

- Authoritative database data.
- MongoDB documents.
- Redis keys.
- OpenSearch documents.
- Object storage.
- Events.
- Analytics.
- Backups.
- External integrations.

Some records may need to be retained for legal or operational reasons.

Retention rules must be explicit.

---

# 62. Tenant Data Export

Before tenant deletion where required, provide an export process.

The export should include the data categories the institution is entitled to receive.

Exports must be:

- Authorized.
- Auditable.
- Tenant-scoped.
- Integrity-checked.
- Securely delivered.

---

# 63. Tenant Isolation Testing

Tenant isolation requires dedicated tests.

At minimum test:

```text
Tenant A user
      ↓
Cannot read Tenant B resource

Tenant A administrator
      ↓
Cannot modify Tenant B resource

Tenant A search
      ↓
Cannot return Tenant B result

Tenant A cache
      ↓
Cannot return Tenant B data

Tenant A file access
      ↓
Cannot access Tenant B object
```

---

# 64. Cross-Tenant Regression Tests

Every security-sensitive multi-tenant feature should include negative tests.

Test cases should intentionally attempt:

- Wrong tenant ID.
- Wrong resource ID.
- Valid resource ID from another tenant.
- Cross-tenant search.
- Cross-tenant cache lookup.
- Cross-tenant file access.
- Cross-tenant event processing.
- Unauthorized tenant switching.

The expected result should be rejection or absence of the unauthorized resource.

---

# 65. Tenant Data Leakage

Potential leakage can occur through:

- API responses.
- Error messages.
- Logs.
- Metrics.
- Search.
- Cache.
- Browser storage.
- Analytics.
- Exports.
- Notifications.
- Emails.
- URLs.
- Object storage.

Tenant isolation reviews must consider all of these surfaces.

---

# 66. Error Responses

Errors must not reveal another tenant's data.

For example, an unauthorized request for:

```text
/orders/{otherTenantOrderId}
```

should not reveal sensitive details about that order.

Resource existence leakage should be considered where security requirements demand it.

---

# 67. Logs

Logs containing tenant information should include tenant context where operationally useful.

Example:

```text
tenant_id
user_id
request_id
correlation_id
```

Do not log sensitive tenant data unnecessarily.

Logs must also be protected from unauthorized cross-tenant access.

---

# 68. Metrics

Metrics should avoid exposing raw tenant-sensitive information.

Where tenant-level metrics are required, ensure access is restricted appropriately.

High-cardinality tenant labels should be used carefully because they can significantly increase monitoring cost.

---

# 69. Rate Limiting by Tenant

Rate limits may be scoped by:

```text
User
Tenant
IP
API Key
Endpoint
```

Tenant-level limits can protect one institution from consuming disproportionate shared infrastructure resources.

Rate limits should account for legitimate differences in tenant size.

---

# 70. Resource Quotas

KAMPYN may define tenant-level quotas for:

- API requests.
- Storage.
- File size.
- Users.
- Food vendors.
- Search requests.
- Event volume.
- Notifications.
- Reports.

Quotas must be explicit and observable.

---

# 71. Noisy Neighbor Protection

Shared infrastructure introduces noisy-neighbor risk.

One tenant generating excessive traffic should not destabilize other tenants.

Potential controls include:

- Rate limiting.
- Queue isolation.
- Worker concurrency limits.
- Resource quotas.
- Database connection controls.
- Search limits.
- Storage quotas.

---

# 72. Tenant Performance Isolation

For high-volume tenants, dedicated infrastructure may become appropriate.

Possible escalation:

```text
Shared Infrastructure
        ↓
Resource Limits
        ↓
Dedicated Resources
        ↓
Dedicated Deployment
```

The decision should be based on actual workload and institutional requirements.

---

# 73. Tenant-Aware Caching Strategy

Caching should consider:

```text
Tenant
Resource
Authorization Scope
Version
```

Example:

```text
tenant:{tenant}:food:{food_id}:v2
```

Tenant-aware cache design must remain consistent across API instances.

---

# 74. Tenant-Aware Search Strategy

Search queries should include mandatory tenant filtering.

Conceptually:

```text
Search Query
    +
tenant_id filter
    ↓
OpenSearch
```

Tenant filters should be injected by trusted server-side code rather than accepted as arbitrary client filters.

---

# 75. Tenant-Aware Database Transactions

Transactions must operate within one tenant context unless a platform-level cross-tenant transaction is explicitly required.

Avoid transactions that accidentally combine unrelated tenants.

For example:

```text
Tenant A Order
+
Tenant B Order
```

should not become one ordinary business transaction.

---

# 76. Tenant-Aware Events

Events must preserve tenant scope from producer to consumer.

Example:

```text
OrderPlaced
   ↓
tenant_id = A
   ↓
Search Consumer
   ↓
Tenant A index
```

A consumer must never drop tenant context when creating derived data.

---

# 77. Tenant-Aware APIs for SDKs

KAMPYN SDKs must make tenant context explicit.

For example:

```text
Client
  ↓
Authenticated Session
  ↓
Tenant Context
  ↓
Resource API
```

The SDK should make it difficult to accidentally issue tenant-ambiguous requests.

Tenant authorization must still be enforced by the server.

---

# 78. Self-Hosted Multi-Tenancy

A university self-hosting KAMPYN may use:

```text
Single Institution
      ↓
Single Tenant
      ↓
Dedicated Infrastructure
```

In this deployment model, the tenant boundary still exists logically even if there is only one tenant.

This keeps the application architecture consistent between SaaS and self-hosted deployments.

---

# 79. Single-Tenant Deployments

A single-tenant deployment should not require removing tenant logic from the application.

Instead:

```text
Deployment
    ↓
One Tenant
```

The same tenant-aware application architecture is retained.

This prevents divergent SaaS and self-hosted codebases.

---

# 80. Tenant Migration

A tenant may need to move between infrastructure environments.

Example:

```text
Shared Database
      ↓
Export
      ↓
Transform / Validate
      ↓
Dedicated Database
      ↓
Verify
      ↓
Switch Routing
```

Tenant migration must preserve:

- IDs.
- Relationships.
- Files.
- Search data.
- Configuration.
- Integrations.
- Audit records.

---

# 81. Tenant Migration Safety

Before migration:

- Back up data.
- Verify destination capacity.
- Validate schema compatibility.
- Test migration.
- Establish rollback strategy.

After migration:

- Verify record counts.
- Verify relationships.
- Verify search.
- Verify files.
- Verify integrations.
- Verify authentication.
- Verify tenant isolation.

---

# 82. Tenant Data Consistency

Derived systems may temporarily lag after tenant provisioning, migration, or restoration.

The architecture must provide reconciliation mechanisms.

For example:

```text
Database Restored
      ↓
Rebuild OpenSearch
      ↓
Warm / Repopulate Cache
      ↓
Validate Derived State
```

---

# 83. Tenant Recovery

Tenant recovery should be possible without unnecessarily affecting other tenants.

If shared infrastructure is used, recovery procedures must distinguish:

```text
Tenant A Recovery
```

from:

```text
Entire Platform Recovery
```

Dedicated tenants may have independent recovery procedures.

---

# 84. Tenant Backup Strategy

Backup architecture should define whether backups are:

```text
Platform-wide
Tenant-specific
Both
```

Tenant-specific restoration is valuable when institutions require independent recovery.

The selected strategy must balance:

- Recovery requirements.
- Storage cost.
- Operational complexity.
- Isolation requirements.

---

# 85. Tenant Security Boundary

The tenant boundary must be treated as a security invariant.

Every new feature must answer:

```text
What tenant owns this data?
How is tenant context established?
How is tenant access authorized?
Where is tenant scope enforced?
How is tenant scope preserved asynchronously?
```

If these questions cannot be answered, the feature is not multi-tenant safe.

---

# 86. Multi-Tenancy Change Checklist

Before completing a multi-tenant change:

- [ ] Tenant ownership is explicitly defined.
- [ ] Tenant identity is stable.
- [ ] Tenant context is established server-side.
- [ ] Membership is validated.
- [ ] Authorization includes tenant scope.
- [ ] Database queries are tenant-scoped.
- [ ] Database constraints support isolation where appropriate.
- [ ] Cache keys include tenant scope.
- [ ] Frontend query keys include tenant scope where necessary.
- [ ] Search queries enforce tenant filtering.
- [ ] Object storage paths are tenant-scoped.
- [ ] Events carry tenant context.
- [ ] Background jobs preserve tenant context.
- [ ] Reports and exports preserve tenant scope.
- [ ] Integrations use the correct tenant configuration.
- [ ] Logs and metrics are appropriately scoped.
- [ ] Rate limits and quotas are considered.
- [ ] Cross-tenant access is explicitly authorized.
- [ ] Tenant switching is safe.
- [ ] Tenant lifecycle behavior is defined.
- [ ] Isolation tests exist.
- [ ] Negative cross-tenant tests exist.
- [ ] Self-hosted implications are considered.
- [ ] Documentation is updated.

---

# 87. Final Multi-Tenancy Principle

Tenant isolation must be an architectural property, not a convention developers are expected to remember.

The preferred request model is:

```text
                    Request
                       │
                       ↓
                Authentication
                       │
                       ↓
                Tenant Resolution
                       │
                       ↓
              Membership Validation
                       │
                       ↓
                 Authorization
                       │
                       ↓
              Tenant-Scoped Domain
                       │
          ┌────────────┼────────────┐
          ↓            ↓            ↓
       Database      Cache       Search
          │            │            │
          └────────────┼────────────┘
                       ↓
                 Tenant Data
```

The same tenant boundary must continue through asynchronous and derived systems:

```text
Tenant Context
     ↓
Database
     ↓
Event
     ↓
Worker
     ↓
Cache / Search / Storage / Notification
```

The fundamental invariant is:

```text
A tenant may access only resources that belong to
that tenant or resources explicitly authorized for
cross-tenant/platform access.
```

The architecture should optimize for:

```text
Isolation
    ↓
Security
    ↓
Correctness
    ↓
Operational Clarity
    ↓
Scalability
    ↓
Performance
```

KAMPYN should maintain one consistent tenant-aware architecture across:

```text
SaaS
Self-Hosted
Single-Tenant
Multi-Tenant
Dedicated-Tenant
```

The deployment model may change.

The tenant security boundary must not.
# Authorization

## 1. Purpose

This document defines the mandatory authorization standards for KAMPYN, including access control, permissions, roles, resource ownership, tenant isolation, privileged operations, policy enforcement, and authorization lifecycle management.

Authorization determines **what an authenticated identity is permitted to do**. Authentication verifies identity; authorization evaluates whether that identity may perform a specific action on a specific resource under the current conditions.

These standards apply to:
- Students, faculty, staff, vendors, and university administrators.
- Platform-level administrators and internal operators.
- SaaS and self-hosted deployments.
- Public, private, and internal APIs.
- Web, mobile, and SDK clients.
- Background jobs and service-to-service operations.
- Food ordering, inventory, bookings, facilities, HR, community, messaging, complaints, search, analytics, and integrations.

Authorization MUST be enforced server-side and MUST be consistently applied across all services and data access paths.

This document complements:
- `.ai/security/api-security.md`
- `.ai/security/authentication.md`

## 2. Core Principles

All authorization implementations MUST follow these principles:

1. **Deny by default:** Access is denied unless an explicit policy permits the requested action.
2. **Least privilege:** Every identity receives only the permissions necessary for its responsibilities.
3. **Server-side enforcement:** Authorization decisions MUST be made by trusted backend components.
4. **Every request is independent:** A previously authorized request does not automatically authorize subsequent requests.
5. **Object-level authorization:** Access to an identifier does not imply access to the referenced resource.
6. **Tenant isolation:** Tenant-scoped resources MUST remain isolated unless an explicitly authorized cross-tenant operation exists.
7. **Explicit permissions:** Privileges MUST be represented by documented permissions rather than scattered assumptions.
8. **Separation of duties:** Sensitive operations SHOULD require distinct permissions or approval paths where appropriate.
9. **Context-aware decisions:** Authorization MUST account for ownership, membership, resource state, and other relevant conditions.
10. **Fail securely:** Missing policies, unavailable authorization dependencies, or ambiguous decisions MUST NOT grant access.
11. **Auditable decisions:** Sensitive authorization outcomes and privileged actions MUST be traceable.
12. **Consistent enforcement:** The same security rules MUST apply regardless of client, API route, or deployment type.

## 3. Authentication vs. Authorization

Authentication and authorization are separate security responsibilities.

| Authentication | Authorization |
|---|---|
| Verifies identity | Determines permitted actions |
| Establishes a trusted principal | Evaluates access policies |
| Validates credentials | Checks permissions and resource relationships |
| Establishes authentication context | Enforces tenant and ownership boundaries |
| Handles sessions and tokens | Controls access to operations and data |

A successful login MUST NOT grant unrestricted access to application resources.

For example, a student may be authenticated successfully but still be unauthorized to:
- Access another student's private booking.
- Modify a food court's inventory.
- Approve a hostel reservation.
- Read confidential HR records.
- Access another university's data.
- Perform a platform-level administrative operation.

Every protected operation MUST perform its own authorization checks.

## 4. Authorization Architecture

Authorization SHOULD be implemented as a shared, reusable capability integrated into the application architecture.

<escape>
Incoming Request
       |
       v
Authentication Middleware
       |
       v
Trusted Principal Context
       |
       v
Tenant Context Resolution
       |
       v
Authorization Policy Evaluation
       |
       +---- Denied ----> Safe Error Response
       |
       v
Application Service
       |
       v
Resource-Level Validation
       |
       v
Repository / Domain Operation
       |
       v
Protected Resource
</escape>

### 4.1 Responsibilities

The authorization subsystem MUST provide:

- Permission definitions.
- Role and permission mappings.
- Resource-level access evaluation.
- Tenant-aware access enforcement.
- Ownership and relationship checks.
- Privileged operation controls.
- Policy evaluation interfaces.
- Authorization failure handling.
- Audit events for sensitive decisions and actions.
- Policy testing and validation.

### 4.2 Separation of Responsibilities

| Layer | Responsibility |
|---|---|
| Authentication | Establishes verified identity |
| Principal context | Carries validated identity attributes |
| Tenant resolver | Establishes authorized tenant context |
| Authorization policy | Evaluates permissions and access conditions |
| Application service | Enforces domain rules and workflow constraints |
| Repository | Applies required data-scope constraints |
| Audit system | Records relevant security-sensitive actions |

No single layer should be assumed to provide complete authorization for all operations.

### 4.3 Centralized Policy, Distributed Enforcement

Authorization policies SHOULD be centrally defined and reusable, while enforcement MUST occur at the appropriate service and resource boundaries.

- Avoid duplicated permission definitions across modules.
- Avoid embedding policy logic in frontend components.
- Avoid relying exclusively on API gateway authorization.
- Ensure application services enforce domain-specific access requirements.
- Ensure repositories cannot accidentally bypass mandatory tenant or ownership constraints.
- Use shared policy interfaces to reduce inconsistencies.

Centralization MUST NOT create a single unrestricted authorization bypass or a dependency that causes all services to grant access when unavailable.

## 5. Authorization Model

KAMPYN SHOULD use a combination of Role-Based Access Control (RBAC) and Attribute-Based Access Control (ABAC), supported by resource ownership and relationship-based checks where necessary.

### 5.1 Role-Based Access Control (RBAC)

RBAC assigns permissions to roles, and roles to identities.

Example:

```text
Role: FOOD_COURT_MANAGER
    |
    +-- menu.read
    +-- menu.update
    +-- inventory.read
    +-- inventory.update
    +-- orders.read
    +-- orders.manage
```

Users MUST receive roles through an authorized process. Roles MUST NOT be self-assigned through ordinary user requests.

### 5.2 Attribute-Based Access Control (ABAC)

ABAC evaluates attributes of:
- The principal.
- The requested action.
- The target resource.
- The environment or context.

Example:

```text
Allow access if:
    principal is authenticated
    AND principal has orders.read
    AND principal.tenant_id == resource.tenant_id
    AND principal is permitted to view resource
```

ABAC SHOULD be used where a role alone is insufficient to establish access, such as ownership, department, booking status, or resource assignment.

### 5.3 Relationship-Based Access

Relationship-based checks MAY be used where permissions depend on relationships between users and resources.

Examples:
- A user belongs to a conversation.
- A student owns a booking.
- A faculty member supervises a department.
- A vendor manages a particular food court.
- A moderator is assigned to a community.

Relationship checks MUST be evaluated using trusted server-side data, not client-submitted relationship claims.

### 5.4 Authorization Decision

An authorization decision SHOULD consider:

```text
principal
    + action
    + resource
    + tenant context
    + resource relationships
    + resource state
    + applicable policy
    = allow or deny
```

Policies MUST be deterministic for the same inputs and policy state, unless explicitly designed to use contextual risk signals.

Unknown roles, permissions, resources, or policy conditions MUST NOT result in implicit access.

## 6. Principal and Authorization Context

### 6.1 Principal

The authenticated principal MUST be created from verified authentication data.

A principal MAY contain:

- Internal identity ID.
- Identity type.
- Authentication method.
- Session ID or credential identifier.
- Verified tenant membership references.
- Assigned roles or permission references.
- Authentication time.
- MFA assurance information.
- Account status.

Only validated attributes may be used for authorization decisions.

### 6.2 Trusted Attributes

The following MUST NOT be trusted when supplied directly by the client:

- User ID representing the authenticated caller.
- Role names.
- Permission lists.
- Tenant IDs.
- Ownership claims.
- Administrative flags.
- MFA-completed flags.
- Account status.
- Resource approval status.

These values MUST be derived from trusted identity, membership, policy, or persistence data.

### 6.3 Context Propagation

Authorization context MUST be propagated explicitly through:
- Middleware.
- Application services.
- Repository operations.
- Background jobs.
- Event handlers.
- Service-to-service requests.
- Realtime communication handlers.

Avoid:
- Mutable global authorization state.
- Implicit tenant context.
- Shared request state between concurrent operations.
- Unverified context copied from headers.
- Authorization decisions based on stale identity information without a documented policy.

## 7. Roles and Permissions

### 7.1 Permission Definitions

Permissions MUST represent specific, understandable actions on defined resource categories.

Use a consistent naming convention:

```text
resource.action
```

Examples:

```text
users.read
users.update
users.suspend

orders.read
orders.create
orders.cancel
orders.manage

inventory.read
inventory.update
inventory.adjust

bookings.read
bookings.create
bookings.cancel
bookings.approve

complaints.read
complaints.assign
complaints.resolve

analytics.read
analytics.export
```

Permission names MUST be:
- Unique.
- Explicit.
- Documented.
- Stable across the relevant API contract.
- Narrow enough to avoid accidental privilege escalation.

Avoid vague permissions such as `admin.access`, `all.manage`, or `system.full` for ordinary roles.

### 7.2 Role Definitions

Roles SHOULD group permissions according to operational responsibilities.

Possible KAMPYN role categories include:

| Role | Typical scope |
|---|---|
| Student | Own profile, orders, bookings, and permitted community actions |
| Faculty | Assigned academic or university-related functions |
| Staff | Assigned operational responsibilities |
| Vendor Staff | Assigned vendor operations |
| Food Court Manager | Managed food court operations |
| Hostel Manager | Assigned hostel operations |
| Facility Manager | Assigned facility operations |
| HR Manager | Authorized HR workflows |
| University Administrator | Approved university-level administration |
| Platform Administrator | Explicit platform-level operational responsibilities |
| Service Identity | Specific machine-to-machine permissions |

These are conceptual roles. Each deployment MUST define its actual roles and permissions based on its operational model.

### 7.3 Role Assignment

Role assignment MUST:
- Be performed by an authorized actor or approved provisioning process.
- Validate the target user's identity and tenant membership.
- Restrict assignment to roles the assigning actor is allowed to grant.
- Prevent users from elevating their own privileges.
- Record relevant audit information.
- Apply changes according to the defined session and authorization-cache invalidation policy.

A role MUST NOT be granted solely because a client submits a role name in a registration or profile-update request.

### 7.4 Role Hierarchies

Role hierarchies SHOULD be avoided unless there is a clear, documented need.

If hierarchies are used:
- Inheritance MUST be explicit.
- Effective permissions MUST be calculable and reviewable.
- Privileged permissions MUST NOT propagate unintentionally.
- Cyclic inheritance MUST be rejected.
- Changes to inheritance MUST be audited and tested.

### 7.5 Custom Roles

If universities can define custom roles:
- Custom roles MUST be scoped to the relevant tenant.
- Permissions MUST be selected from an approved permission registry.
- Tenants MUST NOT define arbitrary platform-level permissions.
- Custom role changes MUST be audited.
- Role assignment MUST verify that the target user is within the tenant.
- Changes MUST invalidate or refresh relevant cached authorization state.

Custom roles MUST NOT weaken platform-level security boundaries.

## 8. Least Privilege

Every user, role, service, and integration MUST receive only the permissions required for its responsibilities.

Implement least privilege by:
- Defining narrow permissions.
- Limiting roles to operational scope.
- Restricting resource access to assigned resources.
- Separating read, create, update, delete, approve, and export capabilities.
- Restricting administrative access.
- Reviewing unused or excessive permissions.
- Revoking access when responsibilities change.
- Limiting service identities to specific APIs and operations.

Avoid granting broad permissions as a workaround for missing access-control design.

## 9. Object-Level Authorization

### 9.1 Mandatory Resource Checks

Every operation involving a specific resource MUST verify that the principal may perform the requested action on that resource.

This applies to:
- Read operations.
- Create operations involving parent resources.
- Updates.
- Deletes.
- State transitions.
- Exports.
- File downloads.
- Realtime subscriptions.
- Administrative overrides.

Resource IDs MUST NOT be treated as proof of ownership or access.

### 9.2 Ownership

Where resources are user-owned, authorization MUST validate ownership using trusted server-side data.

Example:

```text
Allow if:
    principal.id == order.owner_id
    AND principal.tenant_id == order.tenant_id
```

Ownership alone may not be sufficient. Additional role, status, and business rules may apply.

### 9.3 Resource Relationships

When access depends on a relationship, the relationship MUST be validated at the time of the operation or through a documented consistency mechanism.

Examples:
- Conversation membership.
- Vendor assignment.
- Department membership.
- Facility management assignment.
- Booking ownership.
- Staff responsibility for a complaint.

### 9.4 Safe Resource Queries

Where possible, constrain data access at the query boundary.

Illustrative pattern:

```go
order, err := repository.GetByIDForTenant(
    ctx,
    orderID,
    principal.TenantID,
)
```

The application service MUST then verify any additional ownership, permission, and state conditions not enforced by the repository query.

Do not retrieve unrestricted records and assume that a frontend filter will prevent unauthorized access.

### 9.5 Resource Enumeration

APIs MUST prevent unauthorized users from enumerating resource identifiers or inferring protected data through:
- Predictable IDs.
- Different error messages.
- Unrestricted counts.
- Search suggestions.
- Pagination metadata.
- Bulk endpoints.
- Timing differences, where practical.

Returning `404 Not Found` instead of `403 Forbidden` MAY be appropriate when the existence of a resource is sensitive. The behavior MUST be consistent and documented.

## 10. Function-Level Authorization

Every API operation MUST define the required permission or policy.

Examples of protected functions:
- Creating or disabling accounts.
- Assigning roles.
- Changing permissions.
- Managing food court configuration.
- Updating inventory.
- Approving bookings.
- Processing refunds.
- Exporting analytics.
- Accessing HR records.
- Moderating community content.
- Changing tenant configuration.
- Managing integrations.
- Performing platform-level operations.

Function-level access MUST be enforced regardless of whether the operation is exposed through:
- REST.
- GraphQL.
- WebSocket.
- Background jobs.
- Internal service calls.
- Administrative interfaces.

Do not protect privileged operations solely through route naming, UI visibility, or gateway configuration.

## 11. Property-Level Authorization

### 11.1 Request Properties

Request schemas MUST explicitly define which properties a principal may submit or modify.

Privileged fields MUST be controlled by the server, including where applicable:
- User ownership.
- Tenant ID.
- Roles and permissions.
- Payment state.
- Refund state.
- Approval state.
- Audit fields.
- Internal moderation flags.
- Creation and modification timestamps.

Use explicit DTOs and allowlisted field mapping.

### 11.2 Response Properties

Response schemas MUST expose only properties the caller is authorized to view.

Sensitive fields MAY include:
- Personal contact information.
- HR and staff records.
- Payment metadata.
- Private complaint information.
- Authentication metadata.
- Internal administrative notes.
- Private messages.
- Other users' personal information.

### 11.3 Field-Level Rules

Field-level authorization SHOULD be applied when:
- Different roles may access different fields.
- Sensitive information has additional restrictions.
- Data visibility depends on ownership or workflow state.
- A resource contains both public and confidential properties.

Avoid returning full persistence models and removing a few known sensitive fields afterward. Prefer explicit response contracts.

## 12. Multi-Tenant Authorization

Tenant isolation is a mandatory authorization boundary in KAMPYN.

### 12.1 Tenant Scope

Every tenant-scoped authorization decision MUST validate:
- The principal's verified membership.
- The requested tenant context.
- The resource's tenant ownership.
- The action's tenant-specific permission.
- Any additional resource relationship or business constraints.

### 12.2 Tenant Context Resolution

Tenant context MUST be established using trusted authentication claims, server-side membership records, or another explicitly approved mechanism.

Client-supplied tenant identifiers MUST NOT independently establish authorization.

If a client requests an operation for a specific tenant, the backend MUST verify that the principal is allowed to operate within that tenant.

### 12.3 Tenant-Scoped Permissions

Permissions SHOULD be evaluated within a tenant scope.

For example:

```text
orders.read
```

must not mean that a user can read orders from every university. Its effective access MUST be constrained by verified tenant membership and resource scope.

### 12.4 Cross-Tenant Access

Cross-tenant access MUST be denied by default.

Where platform operators require cross-tenant access:
- Use dedicated platform-level permissions.
- Require an explicit operational purpose.
- Restrict the action to the minimum necessary scope.
- Record the actor, target tenant, action, and outcome.
- Apply additional approval or reauthentication where appropriate.
- Avoid granting persistent cross-tenant access when temporary scoped access is sufficient.

### 12.5 Tenant Isolation Across Systems

Authorization MUST be enforced consistently across:
- PostgreSQL.
- MongoDB.
- Redis.
- OpenSearch.
- Object storage.
- Background workers.
- Event handlers.
- WebSocket rooms.
- Internal service APIs.
- Analytics and reporting.

A resource that is hidden in the main API but accessible through search, cache, file storage, or realtime interfaces is an authorization defect.

### 12.6 Self-Hosted Deployments

Self-hosted deployments MUST retain tenant and administrative boundaries according to their configured operating model.

Deployment administrators MUST NOT automatically receive access to every application user's private data unless that access is explicitly authorized and documented.

## 13. Authorization for KAMPYN Domains

### 13.1 Food Ordering

Authorization MUST ensure:
- Students can view and manage their own orders.
- Vendor staff can access orders for assigned vendors or food courts.
- Food court managers can manage only their assigned operations.
- Pricing and payment-state changes are restricted.
- Refund operations require explicit permissions.
- Order transitions follow approved workflows.
- Administrative overrides are auditable.

Order ownership MUST be verified on every sensitive operation, including viewing details, cancelling, and requesting refunds.

### 13.2 Food Courts and Vendors

Authorization MUST ensure:
- Vendor staff can manage only assigned vendor resources.
- Food court managers are restricted to their assigned food courts.
- Vendor onboarding and verification require authorized workflows.
- Vendor ownership and assignment changes are privileged operations.
- Cross-vendor access is denied by default.
- Sensitive commercial and configuration data is restricted.

### 13.3 Inventory

Authorization MUST ensure:
- Inventory reads are restricted to permitted users.
- Stock changes are restricted to authorized operational roles.
- Manual adjustments require explicit permissions.
- Inventory ownership and tenant scope are validated.
- High-impact stock adjustments are audited.
- Clients cannot directly override authoritative inventory state.

### 13.4 Hostel and Guest-House Bookings

Authorization MUST ensure:
- Users can access their own bookings.
- Hostel or guest-house staff can manage only assigned facilities.
- Approval and rejection permissions are explicit.
- Cancellation permissions respect ownership and workflow rules.
- Sensitive guest details are restricted.
- Administrative overrides are logged.
- Booking state transitions are validated.

### 13.5 Shuttle and Facility Scheduling

Authorization MUST ensure:
- Users can access their own reservations.
- Facility managers can manage only assigned facilities.
- Staff permissions are limited to relevant operations.
- Resource availability and booking conflicts are enforced by domain logic.
- Changes to schedules and capacity are restricted.
- Administrative actions are audited.

### 13.6 Library Vacancy and Reservations

Authorization MUST ensure:
- Users can access permitted availability information.
- Users can manage their own reservations.
- Staff can perform only assigned library operations.
- Administrative controls are limited to authorized roles.
- Private reservation information is not exposed through public availability endpoints.

### 13.7 Complaints and Reports

Authorization MUST ensure:
- Users can access their own submitted complaints and permitted status information.
- Assigned staff can access complaints within their responsibility.
- Sensitive complaint content is restricted.
- Reporter identity is protected where required.
- Assignment and resolution operations require explicit permissions.
- Moderation and administrative overrides are audited.

### 13.8 HR and Staff Management

HR data MUST be treated as sensitive.

Authorization MUST ensure:
- HR access is granted only to designated roles.
- Staff can access their own permitted information.
- Managers access only information within their authorized organizational scope.
- Salary, disciplinary, personal, and employment records receive appropriate restrictions.
- Bulk HR exports require explicit permissions.
- Role and employment-status changes are audited.

### 13.9 Community and Messaging

Authorization MUST ensure:
- Only authorized conversation members can read private messages.
- Membership is verified for every sensitive conversation operation.
- Message edits and deletions follow explicit ownership and moderation rules.
- Moderation permissions are scoped appropriately.
- Private attachments are accessible only to authorized participants.
- Cross-tenant communication is denied unless explicitly supported and authorized.
- Room and channel membership is validated server-side.

### 13.10 Analytics and Reporting

Authorization MUST ensure:
- Reports are scoped to authorized tenants and organizational units.
- Sensitive aggregates are protected from inference where appropriate.
- Export permissions are separate from ordinary read permissions when necessary.
- Bulk exports are auditable.
- Personal or confidential fields are restricted.
- Search, filter, and aggregation parameters cannot bypass authorization.

### 13.11 Administration

Administrative access MUST be separated into clear scopes:
- Platform-level administration.
- University-level administration.
- Department-level administration.
- Vendor or facility administration.
- Operational staff access.

Administrative interfaces MUST use the same backend authorization policies as other clients.

## 14. Privileged Access Management

### 14.1 Privileged Permissions

Privileged permissions SHOULD be narrow and explicitly defined.

Examples include:
- `users.roles.assign`
- `users.suspend`
- `tenant.settings.update`
- `payments.refund`
- `analytics.export`
- `security.policies.update`
- `platform.tenants.manage`

Avoid a single unrestricted administrator permission where more specific controls are practical.

### 14.2 Privileged Account Requirements

Privileged accounts SHOULD:
- Require MFA.
- Use strong authentication and secure session lifetimes.
- Have narrowly scoped permissions.
- Be reviewed periodically.
- Avoid sharing credentials.
- Use separate administrative identities where justified.
- Be monitored for sensitive actions.
- Require reauthentication for high-risk operations.

### 14.3 Just-in-Time Access

For high-impact platform or operational tasks, consider temporary, time-limited elevation rather than persistent broad permissions.

Temporary access MUST:
- Have an explicit purpose.
- Be approved by an authorized actor where required.
- Have a defined expiration.
- Be scoped to specific operations or resources.
- Be logged.
- Be revoked automatically or reliably when it expires.

### 14.4 Break-Glass Access

Emergency access MUST be explicitly designed and documented.

Break-glass mechanisms MUST:
- Be limited to genuine emergencies.
- Use strong identity verification.
- Be auditable.
- Notify responsible operators where appropriate.
- Have bounded scope and duration.
- Be reviewed after use.
- Not become an ordinary administrative workflow.

## 15. Separation of Duties

Sensitive operations SHOULD separate the ability to initiate an action from the ability to approve or finalize it.

Examples MAY include:
- Initiating and approving high-value refunds.
- Requesting and approving sensitive HR changes.
- Creating and approving privileged role assignments.
- Initiating and authorizing high-impact tenant configuration changes.

Where separation of duties is required:
- Define incompatible permission combinations.
- Enforce restrictions server-side.
- Prevent self-approval where prohibited.
- Record the initiator and approver.
- Ensure approval applies to the exact action and resource.
- Invalidate approvals when relevant action details change.

The required separation level MUST be based on documented risk and operational needs.

## 16. Authorization for Background Jobs and Events

Background jobs and event handlers MUST execute under an explicit service identity or securely delegated authorization context.

### 16.1 Service Permissions

Each service identity MUST have:
- A defined purpose.
- Explicit permissions.
- A permitted resource scope.
- A documented owner.
- A revocation mechanism.

Avoid broad permissions shared across unrelated workers.

### 16.2 User-Initiated Jobs

For jobs initiated by a user:
- Record the initiating identity.
- Preserve verified tenant context.
- Validate the requested operation at submission time.
- Recheck authorization at execution time when permissions, ownership, or resource state may have changed.
- Restrict delegated authority to the intended job.
- Audit sensitive completion or failure outcomes.

### 16.3 Event Consumers

Event consumers MUST NOT treat event payloads as proof of authorization.

- Validate the event source.
- Authenticate service-to-service communication where applicable.
- Verify tenant context.
- Enforce consumer-specific permissions.
- Validate resource relationships.
- Prevent replay or unauthorized event injection.
- Avoid processing stale authorization claims without a defined consistency model.

## 17. Authorization for Realtime APIs

WebSocket and realtime interfaces MUST enforce authorization for every protected channel and operation.

- Authenticate the connection.
- Verify subscription permissions.
- Verify channel or room membership.
- Authorize each privileged message operation.
- Enforce tenant isolation.
- Revalidate access after relevant permission or membership changes.
- Remove revoked users from protected channels according to the revocation policy.
- Avoid broadcasting confidential data to unauthorized subscribers.
- Enforce authorization for file attachments and message history.

An authorized connection MUST NOT imply access to every channel or event.

## 18. Authorization for Search and Caching

### 18.1 Search

Search results MUST be restricted to resources the principal is permitted to access.

- Apply tenant and access filters to search queries.
- Validate filter and sort fields.
- Prevent unrestricted access to internal search indexes.
- Protect autocomplete, facets, counts, and suggestions.
- Recheck authorization before sensitive resource operations.
- Ensure index updates and deletion events preserve access boundaries.

Search results MUST NOT be treated as an authoritative authorization source.

### 18.2 Cache

Authorization-sensitive cache entries MUST be scoped by all attributes necessary to preserve access boundaries.

- Include tenant and relevant identity or permission scope in cache keys.
- Avoid sharing user-specific responses across principals.
- Invalidate or expire affected entries after permission changes.
- Avoid caching privileged data in public or shared caches.
- Define behavior for stale authorization state.
- Do not let cache availability failures grant access.

Cache invalidation MUST account for changes to roles, permissions, resource ownership, tenant membership, and account status.

## 19. Authorization Policy Implementation

### 19.1 Policy Definition

Policies SHOULD be defined in a consistent, reviewable format.

A policy SHOULD specify:
- Principal type.
- Required permission.
- Resource type.
- Tenant scope.
- Ownership or relationship requirements.
- Contextual conditions.
- Allowed state transitions.
- Denial behavior.
- Audit requirements.

### 19.2 Policy Evaluation

Authorization evaluation SHOULD:
- Accept an explicit principal.
- Accept a clearly identified action.
- Accept the target resource or its trusted attributes.
- Evaluate applicable policies.
- Return an explicit allow or deny decision.
- Avoid side effects.
- Produce safe diagnostic context for internal observability without leaking protected data.

An authorization evaluator MUST NOT default to allow when an input or policy is missing.

### 19.3 Policy Consistency

- Avoid implementing the same permission with conflicting semantics in different modules.
- Maintain a central permission registry or equivalent source of truth.
- Version policy changes where necessary.
- Review changes to privileged permissions.
- Ensure all API entry points use the applicable policy.
- Maintain tests for shared policy behavior.

### 19.4 Policy Changes

Policy changes MUST:
- Be reviewed.
- Be tested against expected allowed and denied cases.
- Identify affected roles and users.
- Consider cached permission state.
- Consider active sessions and long-lived tokens.
- Be deployed through a controlled process.
- Be auditable when they affect production access.

## 20. Database-Level Authorization

Application-level authorization MUST be supported by safe data-access patterns.

- Use tenant-scoped repository methods for tenant-owned resources.
- Use explicit ownership filters where appropriate.
- Prevent unrestricted updates and deletes.
- Use database constraints to preserve important ownership relationships.
- Consider PostgreSQL Row-Level Security (RLS) for additional defense in depth where suitable.
- Configure database roles with least privilege.
- Ensure background and administrative database operations have explicit scope.
- Prevent tenant context from leaking through connection pooling or reused sessions.

If RLS is used, the application MUST still maintain correct identity and authorization logic. RLS configuration, connection context, and transaction handling MUST be tested for pooled connections and failure scenarios.

MongoDB and other datastores MUST apply equivalent tenant and ownership safeguards through their supported query and access-control mechanisms.

## 21. Authorization Failures

### 21.1 Denial by Default

If authorization cannot be reliably evaluated, the request MUST be denied.

Examples:
- Principal context is missing or invalid.
- Tenant membership cannot be verified.
- Required permissions are unknown.
- Policy evaluation fails.
- Resource ownership cannot be established.
- Required authorization data is unavailable.

### 21.2 HTTP Responses

Use appropriate responses according to the API contract:
- `401 Unauthorized` when authentication is missing or invalid.
- `403 Forbidden` when an authenticated principal lacks permission.
- `404 Not Found` when resource existence must be concealed.
- `409 Conflict` when an operation is disallowed by current resource state or concurrency conditions.

Responses MUST NOT expose sensitive resource information or internal policy implementation details.

### 21.3 Dependency Failures

When authorization depends on a policy service, database, or other dependency:
- Fail closed for protected operations if authorization cannot be verified.
- Avoid converting timeouts into implicit approval.
- Apply bounded retries only where appropriate.
- Use safe error handling.
- Record relevant operational failures.
- Preserve availability for unrelated operations where safe.

## 22. Authorization Auditing

### 22.1 Events to Audit

Sensitive authorization-related events SHOULD include:
- Role assignment and removal.
- Permission changes.
- Privileged access grants.
- Tenant membership changes.
- Cross-tenant administrative access.
- Sensitive resource exports.
- Administrative overrides.
- Approval and rejection actions.
- Access-policy changes.
- Break-glass access.
- Repeated authorization denials that indicate suspicious activity.

### 22.2 Audit Record Fields

Audit records SHOULD contain:
- Actor identity.
- Actor type.
- Tenant context.
- Action.
- Target resource type and identifier.
- Authorization outcome.
- Timestamp.
- Request or correlation ID.
- Reason or approved purpose, where required.
- Relevant before-and-after metadata, excluding secrets.

### 22.3 Audit Integrity

Audit records MUST:
- Be protected from unauthorized modification.
- Have access controls.
- Follow documented retention requirements.
- Avoid unnecessary personal information.
- Avoid storing credentials or secrets.
- Be available for authorized incident investigation.

Do not use ordinary application logs as the only record of high-impact privileged actions.

## 23. Performance and Scalability

Authorization MUST remain correct as KAMPYN scales across tenants, services, and users.

- Use efficient permission evaluation.
- Avoid repeated redundant policy lookups within the same operation.
- Batch relationship checks where safe.
- Use tenant-scoped queries.
- Cache policy data only with explicit invalidation and revocation semantics.
- Avoid unbounded role inheritance or recursive relationship evaluation.
- Set timeouts for external policy dependencies.
- Use bounded concurrency.
- Monitor authorization latency and denial rates.

Performance optimization MUST NOT bypass mandatory authorization checks or weaken tenant isolation.

## 24. Frontend Authorization

Frontend authorization exists to improve user experience and MUST NOT be considered a security boundary.

Frontend applications MAY:
- Hide controls the user cannot access.
- Display actions based on permission information.
- Prevent invalid interactions.
- Show appropriate access-denied states.
- Render role-specific navigation.

Frontend applications MUST NOT:
- Be the only enforcement point.
- Trust local storage or Zustand state as proof of permissions.
- Trust client-supplied role or tenant values.
- Assume that hidden controls prevent direct API calls.
- Expose sensitive data in HTML or client-side bundles merely because UI controls are hidden.

### 24.1 Next.js

- Protected server-rendered data MUST be authorized on the server.
- Server Actions and Route Handlers MUST independently enforce authorization.
- Server Components MUST NOT expose data based solely on client-side role state.
- Shared caching MUST not leak protected responses across identities or tenants.
- Authentication or permission changes MUST clear or refresh relevant client-side state.

### 24.2 TanStack Query and Zustand

- TanStack Query MUST be used for server-state management, not as the authority for permission decisions.
- Query keys MUST include the necessary identity and tenant scope.
- User-scoped cached data MUST be cleared or invalidated on logout, account switching, or tenant switching.
- Zustand MAY store UI-level permission hints, but backend services MUST make final access decisions.
- Optimistic updates MUST not bypass server-side authorization or domain-state validation.

## 25. Authorization for SDKs and Integrations

SDKs and third-party integrations MUST follow the same authorization model as first-party clients.

- Use documented authentication mechanisms.
- Request only required permissions.
- Scope credentials to the intended integration.
- Validate resource ownership and tenant context server-side.
- Avoid exposing privileged tokens in client applications.
- Enforce authorization for webhook configuration and delivery management.
- Revoke credentials when integrations are disabled or compromised.
- Document permission requirements for each integration capability.

An SDK MUST NOT provide a hidden bypass around standard API authorization.

## 26. Testing Requirements

Authorization MUST be tested for both permitted and denied scenarios.

### 26.1 Role and Permission Tests

Test:
- Each role's intended permissions.
- Missing permissions.
- Unknown roles.
- Permission removal.
- Role changes.
- Privileged permission assignment.
- Incompatible role combinations.
- Custom-role constraints, where supported.

### 26.2 Object-Level Tests

Test:
- Resource owner access.
- Non-owner access.
- Assigned staff access.
- Unassigned staff access.
- Access to nonexistent resources.
- Cross-user resource access.
- Unauthorized updates and deletes.
- Unauthorized state transitions.
- Unauthorized file downloads.

### 26.3 Tenant Isolation Tests

Use multiple synthetic tenants to verify:
- Cross-tenant reads are denied.
- Cross-tenant writes are denied.
- Cross-tenant deletes are denied.
- Cross-tenant search results are excluded.
- Cache entries are isolated.
- Files are tenant-scoped.
- Realtime channels enforce membership.
- Background jobs preserve tenant boundaries.
- Events cannot inject unauthorized tenant context.
- Administrative cross-tenant access requires explicit permissions.

### 26.4 Property-Level Tests

Test:
- Unauthorized role changes.
- Unauthorized tenant changes.
- Unauthorized payment-state changes.
- Unauthorized approval-state changes.
- Sensitive-field exposure.
- Mass-assignment attempts.
- Unauthorized access to HR, complaint, and private messaging fields.

### 26.5 Workflow Tests

Test authorization throughout:
- Order creation, cancellation, and refunds.
- Booking creation, approval, and cancellation.
- Inventory adjustments.
- Complaint assignment and resolution.
- Community moderation.
- Staff and role management.
- Tenant configuration changes.
- Analytics and data exports.

### 26.6 Failure and Concurrency Tests

Test:
- Authorization-service timeouts.
- Missing policy configuration.
- Stale permission caches.
- Permission revocation during active sessions.
- Concurrent role changes and sensitive operations.
- Tenant membership removal.
- Database and cache failures.
- Background job execution after authorization changes.

All security-critical denial cases MUST be automated where practical.

## 27. Monitoring and Alerting

Monitor:
- Authorization allow and deny rates.
- Repeated denied requests.
- Privileged operations.
- Role and permission changes.
- Cross-tenant access attempts.
- Authorization-service latency and failures.
- Policy evaluation errors.
- Access after credential or membership revocation.
- Unusual bulk reads and exports.
- Administrative overrides.

Alerts SHOULD be generated for suspicious patterns and high-impact access-control changes.

Monitoring MUST avoid leaking protected resource data or unnecessary personal information.

## 28. Incident Response

Authorization controls MUST support investigation and containment of access-control incidents, including:
- Broken object-level authorization.
- Privilege escalation.
- Cross-tenant data exposure.
- Unauthorized administrative access.
- Incorrect role assignment.
- Stale permission or tenant caches.
- Unauthorized exports.
- Access through overlooked APIs or background jobs.

Incident procedures SHOULD include:
- Revoking excessive permissions.
- Suspending affected credentials or accounts.
- Invalidating relevant authorization caches.
- Restricting vulnerable endpoints.
- Identifying affected tenants and resources.
- Preserving audit evidence.
- Assessing data exposure.
- Applying corrective changes.
- Reviewing authorization tests and policies to prevent recurrence.

## 29. Prohibited Practices

The following practices are prohibited:

- Relying on frontend checks as the sole authorization control.
- Treating authentication as proof of resource access.
- Trusting client-supplied roles, permissions, ownership, or tenant identifiers.
- Granting access by default when a policy is missing.
- Performing object lookups without verifying resource access.
- Exposing database models without field-level controls.
- Allowing unrestricted mass assignment.
- Using broad administrator permissions where narrower controls are practical.
- Allowing users to assign themselves privileged roles.
- Allowing cross-tenant access without explicit authorization.
- Relying solely on API gateways for domain authorization.
- Treating internal network location as authorization.
- Sharing authorization caches across users or tenants without proper scope.
- Allowing stale permission data to bypass revocation requirements.
- Executing background jobs with implicit or excessive privileges.
- Trusting event payloads as authorization proof.
- Exposing private resources through search, analytics, cache, files, or realtime interfaces.
- Skipping authorization tests for denied scenarios.
- Disabling authorization controls to resolve integration or performance issues without an approved alternative.
- Logging sensitive resource contents unnecessarily during authorization failures.

## 30. Authorization Review Checklist

### Architecture
- [ ] Authentication and authorization responsibilities are separate.
- [ ] Authorization policies have a clear source of truth.
- [ ] Enforcement occurs at the appropriate service and resource boundaries.
- [ ] Principal context contains only verified attributes.
- [ ] Authorization fails closed when evaluation is unavailable.

### Roles and Permissions
- [ ] Permissions are explicit and documented.
- [ ] Roles are mapped to approved permissions.
- [ ] Role assignment is authorized and audited.
- [ ] Least privilege is enforced.
- [ ] Custom roles cannot grant platform-level privileges.
- [ ] Privileged operations use narrow permissions.
- [ ] Separation of duties is applied where required.

### Resource and Tenant Access
- [ ] Object-level authorization is enforced.
- [ ] Function-level authorization is enforced.
- [ ] Property-level authorization is enforced.
- [ ] Ownership and relationships are verified server-side.
- [ ] Tenant context is trusted and verified.
- [ ] Cross-tenant access is denied by default.
- [ ] Database queries enforce relevant data scopes.
- [ ] Files, search, caches, and realtime channels preserve isolation.

### Operations
- [ ] Background jobs use explicit service or delegated identities.
- [ ] Integrations follow the same authorization model.
- [ ] Permission changes trigger the correct invalidation behavior.
- [ ] Sensitive operations are audited.
- [ ] Authorization failures expose no protected data.
- [ ] Administrative access is reviewed and monitored.

### Testing and Monitoring
- [ ] Positive and negative authorization tests exist.
- [ ] Cross-user and cross-tenant tests exist.
- [ ] Privileged-operation tests exist.
- [ ] Property-level and mass-assignment tests exist.
- [ ] Concurrency and revocation behavior are tested.
- [ ] Monitoring covers unusual access patterns.
- [ ] Incident response procedures are documented.

## 31. Definition of Done

An authorization implementation is considered production-ready only when:

1. All protected operations have explicit authorization requirements.
2. Authentication and authorization responsibilities are clearly separated.
3. Roles and permissions are documented and follow least privilege.
4. Object-level, function-level, and property-level authorization are enforced as applicable.
5. Tenant isolation is validated across every relevant system.
6. Ownership, membership, and resource relationships are verified server-side.
7. Privileged operations have appropriate safeguards and audit trails.
8. Background jobs, integrations, and realtime interfaces enforce explicit authorization.
9. Permission changes and revocation follow documented invalidation rules.
10. Authorization failures deny access safely.
11. Automated tests cover both allowed and denied access, including cross-tenant scenarios.
12. Monitoring and incident-response capabilities are ready.
13. The implementation has passed the required code review and security review.

**Authorization is a mandatory backend security boundary. No user, service, role, or client may access a resource or perform an operation without an explicit, verified, and appropriately scoped authorization decision.**
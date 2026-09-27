# KAMPYN Authorization Architecture

## 1. Purpose

This document defines the authorization architecture for KAMPYN.

Authorization determines whether an authenticated identity is permitted to perform a specific action on a specific resource within a specific tenant and context.

Authorization must provide:

- Explicit permission boundaries.
- Strong tenant isolation.
- Resource-level access control.
- Role and permission management.
- Protection against privilege escalation.
- Consistent enforcement across APIs and services.
- Auditable administrative actions.
- Secure defaults.
- Clear separation from authentication.

Authorization must always be enforced server-side.

---

# 2. Authentication vs Authorization

Authentication establishes identity.

Authorization establishes permissions.

```text id="7j3r8n"
Authentication
    ↓
Who is this?
    ↓
Authenticated Identity
    ↓
Authorization
    ↓
What may this identity do?
    ↓
Resource Access
```

Never treat:

```text id="8j2x6s"
Authenticated
=
Authorized
```

A valid session or token does not automatically grant access to a resource or operation.

---

# 3. Authorization Model

KAMPYN should use a layered authorization model:

```text id="b2e6q9"
Identity
   ↓
Tenant Membership
   ↓
Role / Permission
   ↓
Resource Ownership / Scope
   ↓
Action
   ↓
Authorization Decision
```

A request should be allowed only when all applicable authorization requirements are satisfied.

For example:

```text id="u5r8f1"
User
  ↓
Member of University A
  ↓
Has vendor-management permission
  ↓
Vendor belongs to University A
  ↓
Action = update vendor
  ↓
ALLOW
```

---

# 4. Authorization Is a Server-Side Boundary

Frontend authorization is for user experience.

Backend authorization is the security boundary.

The frontend may:

- Hide unavailable actions.
- Disable controls.
- Display permitted navigation.
- Display role-specific interfaces.

The backend must independently verify:

- Identity.
- Tenant membership.
- Permission.
- Resource access.
- Action.

Never rely on:

```text id="q7r1fa"
hidden button
disabled button
client-side role
client-side route guard
```

for security.

---

# 5. Default Deny

Authorization must follow a default-deny model.

If the system cannot establish that an action is permitted:

```text id="8u7v0p"
DENY
```

Do not interpret missing permission information as permission.

Avoid:

```text id="c2w5y4"
if permission exists:
    check it
else:
    allow
```

Prefer:

```text id="7u1p9x"
if permission exists:
    evaluate
else:
    deny
```

---

# 6. Explicit Authorization Decisions

Authorization decisions should be explicit.

Conceptually:

```text id="e7m3r5"
Can(
    principal,
    action,
    resource,
    context
) → Allow / Deny
```

The decision should consider the relevant:

- Identity.
- Tenant.
- Role.
- Permission.
- Resource.
- Resource ownership.
- Resource state.
- Context.

Avoid authorization logic scattered across unrelated handlers.

---

# 7. Roles

Roles group permissions around meaningful responsibilities.

Examples may include:

```text id="v0q4a6"
Student
Faculty
Staff
Vendor
Vendor Manager
Hostel Manager
Library Manager
Food Court Manager
University Admin
System Admin
```

The exact role model must reflect the actual KAMPYN deployment.

Do not create roles merely because two users currently have different UI experiences.

Roles should represent meaningful authorization responsibilities.

---

# 8. Permissions

Permissions represent specific capabilities.

Prefer permissions such as:

```text id="v6q2p1"
orders.read
orders.create
orders.cancel

bookings.read
bookings.create
bookings.approve
bookings.cancel

inventory.read
inventory.update

vendors.read
vendors.create
vendors.update
vendors.delete
```

over relying exclusively on broad role checks such as:

```text id="h5k3t9"
if user.role == "admin"
```

Fine-grained permissions make authorization more explicit and easier to evolve.

---

# 9. Permission Naming

Permissions should follow a consistent structure.

Prefer:

```text id="4g6f0j"
<resource>.<action>
```

Examples:

```text id="b7f4k2"
users.read
users.update
orders.read
orders.create
orders.cancel
inventory.read
inventory.update
reports.export
```

For more complex domains, additional scopes may be appropriate.

Do not create arbitrary inconsistent permission names.

---

# 10. Role-to-Permission Mapping

Roles should map to permissions explicitly.

Conceptually:

```text id="2j9q8m"
Role
  ↓
Permissions
  ↓
Authorization Decision
```

Avoid hardcoding large permission matrices throughout application code.

Prefer centralized or domain-owned authorization definitions.

---

# 11. Resource-Level Authorization

Permission to perform an action does not automatically mean permission to perform it on every resource.

For example:

```text id="4q5j1z"
orders.update
```

does not mean a user can update every order in KAMPYN.

The system must also evaluate:

```text id="r5g8w2"
Does this user have access to this specific order?
```

This is especially important for:

- Orders.
- Bookings.
- Complaints.
- Payments.
- User records.
- Files.
- Hostel records.
- Administrative resources.

---

# 12. Ownership

Some resources are controlled by their owner.

Example:

```text id="n8c4q2"
Student
  ↓
Owns
  ↓
Order
```

Authorization may permit:

```text id="p7v5s1"
order.ownerId == authenticatedUser.id
```

where the domain requires ownership.

Never assume resource IDs provide ownership.

---

# 13. Tenant Isolation

Tenant isolation is mandatory for tenant-scoped resources.

The authorization decision must establish:

```text id="e1m5k7"
Authenticated User
       ↓
Tenant Membership
       ↓
Resource Tenant
       ↓
Match
```

A user belonging to University A must not access University B resources unless explicitly authorized across tenants.

Tenant filtering must be enforced in the backend.

---

# 14. Tenant Context

Tenant context may originate from:

- Authenticated identity.
- Session.
- Trusted deployment configuration.
- Institution domain.
- Explicit tenant selection validated against membership.

Never trust a client-supplied tenant ID as authorization.

For example, this is insufficient:

```text id="j6h0z3"
X-Tenant-ID: university-b
```

The server must verify that the authenticated principal is allowed to operate within that tenant.

---

# 15. Cross-Tenant Access

Cross-tenant operations should be exceptional.

If cross-tenant administration is required:

- Define the permission explicitly.
- Define the allowed scope.
- Audit the action.
- Require appropriate administrative privileges.
- Avoid silently bypassing tenant isolation.

Do not implement cross-tenant access through a generic "admin bypass."

---

# 16. Administrative Access

Administrative access must be explicitly scoped.

Avoid a single unrestricted:

```text id="j3f7m1"
isAdmin = true
```

model when the system requires differentiated administrative responsibilities.

Prefer scoped permissions such as:

```text id="p8w2r4"
users.manage
vendors.manage
orders.manage
inventory.manage
reports.export
tenant.settings.manage
```

Administrative actions should be auditable.

---

# 17. System Administrators

System administrators may require access across tenant boundaries.

Such access must be explicitly defined as a separate authorization capability.

System-level permissions must not automatically be granted to ordinary university administrators.

Distinguish:

```text id="f2n6a9"
University Administrator
        ≠
System Administrator
```

unless the deployment explicitly defines otherwise.

---

# 18. Permission Scope

Permissions should be scoped to the smallest meaningful boundary.

Possible scopes include:

```text id="k5q8x0"
Global
Tenant
Campus
Food Court
Hostel
Vendor
Resource
```

Use the narrowest scope that satisfies the business requirement.

Avoid granting global access when tenant-level access is sufficient.

---

# 19. Contextual Authorization

Some permissions depend on runtime context.

Examples:

- A vendor may edit only their own menu.
- A hostel manager may modify only assigned hostels.
- A student may cancel only their own booking.
- A manager may approve only requests within their assigned department.
- An administrator may export reports only for their tenant.

Authorization should evaluate these contextual conditions explicitly.

---

# 20. State-Based Authorization

Some actions depend on resource state.

Example:

```text id="r8c2v1"
Order
  ├── Pending → Cancel allowed
  ├── Preparing → Cancel may be restricted
  ├── Completed → Cancel denied
  └── Cancelled → Cancel denied
```

The authorization decision may therefore depend on:

```text id="m4q9s2"
Identity
+
Permission
+
Resource
+
Resource State
```

Do not implement state restrictions solely in the frontend.

---

# 21. Separation of Permission and Business Rules

Authorization and domain business rules are related but distinct.

Authorization determines:

```text id="g5v7t1"
May this actor perform this operation?
```

Domain logic determines:

```text id="q6r3m8"
Is this operation valid according to business rules?
```

Both may be required.

Example:

```text id="e8p4x2"
User has orders.cancel
        ↓
Authorization passes
        ↓
Order state is Completed
        ↓
Domain rule rejects cancellation
```

Do not use authorization as a replacement for domain validation.

---

# 22. Authorization Placement

Authorization should be enforced at the application boundary where the action is known.

Typical flow:

```text id="c7m5x1"
HTTP Request
    ↓
Authentication
    ↓
Application Service
    ↓
Authorization Policy
    ↓
Domain Operation
    ↓
Repository
```

Generic middleware may perform broad authorization checks, but resource-specific authorization should remain close to the use case.

---

# 23. Authorization Policies

Complex authorization rules should be represented through explicit policies or dedicated authorization services.

Example:

```text id="s4j8n0"
CanCancelOrder(principal, order)
CanEditVendor(principal, vendor)
CanApproveBooking(principal, booking)
```

Policies should remain:

- Testable.
- Deterministic where possible.
- Domain-aware.
- Explicit.

Avoid embedding authorization logic across dozens of handlers.

---

# 24. Avoid Scattered Role Checks

Avoid repeatedly writing:

```go id="9w0c6k"
if user.Role == "ADMIN" {
    ...
}
```

throughout the backend.

Prefer:

```text id="d4m7q2"
Authorization Policy
       ↓
CanManageInventory(...)
```

Role checks may still be appropriate for simple infrastructure-level decisions, but domain authorization should use meaningful permissions and policies.

---

# 25. Authorization Middleware

Middleware may establish broad requirements such as:

```text id="g3j6m8"
RequiresAuthentication
RequiresPermission("orders.read")
```

However, middleware alone is insufficient for resource-level authorization.

For:

```text id="n4v8q1"
/orders/{orderId}
```

the application must still determine whether the authenticated identity may access that specific order.

---

# 26. API Authorization

Every protected API endpoint must define its authorization requirements.

For each endpoint determine:

- Authentication required?
- Required permission?
- Tenant scope?
- Resource scope?
- Ownership requirement?
- State restrictions?
- Administrative override?
- Audit requirement?

Avoid endpoints where authorization behavior is ambiguous.

---

# 27. Database Authorization Boundaries

Application authorization remains the primary authorization mechanism.

Where appropriate, database-level access controls may provide defense-in-depth.

Do not rely solely on database filtering if the application architecture requires richer authorization logic.

Queries should still enforce tenant/resource boundaries where applicable to reduce accidental data exposure.

---

# 28. Search Authorization

Search results must obey the same authorization boundaries as ordinary API resources.

Never assume:

```text id="s2h7n5"
user can search
=
user can see every indexed document
```

Search queries must enforce:

- Tenant scope.
- Resource permissions.
- Visibility rules.

OpenSearch must not become an authorization bypass.

---

# 29. Cache Authorization

Cached data must remain authorization-safe.

Cache keys must include all authorization-relevant scope where necessary.

Never serve:

```text id="q8f3m1"
Tenant A cached result
```

to:

```text id="n5w7r2"
Tenant B
```

because the cache key omitted tenant context.

Authorization must not be bypassed merely because data came from Redis.

---

# 30. Background Jobs

Background workers must preserve authorization context where the authorization decision remains relevant.

Do not assume:

```text id="v4m9x1"
background job
=
system administrator
```

If a job performs an operation on behalf of a user or tenant, preserve enough context to enforce:

- Tenant ownership.
- Permission scope.
- Resource scope.
- Audit requirements.

System-level jobs may use explicit service identities instead.

---

# 31. Events

Event consumers must not assume that the event producer's authorization automatically grants the consumer unlimited access.

Consumers should operate using:

- Explicit service identity.
- Defined permissions.
- Event-specific trust boundaries.

Do not propagate arbitrary user privileges through events.

---

# 32. Service-to-Service Authorization

Internal services must authenticate and authorize one another where multiple services exist.

Do not assume:

```text id="p8c4r0"
internal network
=
trusted network
```

Service identities should have:

- Explicit permissions.
- Limited scope.
- Rotatable credentials.
- Auditable access.

---

# 33. Service Accounts

Service accounts must follow least privilege.

A service should receive only the permissions required for its workload.

Avoid:

```text id="f9w3q2"
service-admin
```

when the service only needs:

```text id="t5n7m4"
orders.read
inventory.read
```

---

# 34. Privilege Escalation

Review all operations that modify:

- Roles.
- Permissions.
- Tenant membership.
- Administrative status.
- Service credentials.
- API keys.

These operations must require appropriate authorization and should generally be audited.

Never allow users to modify their own authorization claims merely by submitting a request field such as:

```json id="4z8x1m"
{
  "role": "ADMIN"
}
```

---

# 35. Role Assignment

Role assignment should be treated as a privileged operation.

The system should verify:

```text id="m6k8q1"
Actor
  ↓
May manage roles?
  ↓
Target user
  ↓
Target tenant
  ↓
Requested role
  ↓
Allowed?
```

Do not permit administrators to assign permissions beyond their own authorized scope unless explicitly designed.

---

# 36. Permission Delegation

If KAMPYN supports delegated administration, delegation must have explicit boundaries.

Define:

- Who may delegate.
- What may be delegated.
- To whom.
- For which tenant/resource.
- For how long.
- Whether further delegation is allowed.

Avoid unrestricted permission inheritance.

---

# 37. Temporary Access

Temporary elevated access should have:

- Explicit purpose.
- Start time.
- Expiration.
- Scope.
- Audit trail.
- Revocation.

Do not create permanent privileges for temporary operational requirements.

---

# 38. Break-Glass Access

Emergency administrative access, if required, must be explicit and auditable.

A break-glass mechanism should define:

- Who can activate it.
- What access it grants.
- How activation is logged.
- How long it lasts.
- How it is revoked.
- How post-use review occurs.

Do not create hidden administrator accounts as an emergency mechanism.

---

# 39. Authorization Caching

Authorization decisions may be cached only when the consistency model is understood.

If permissions change, cached decisions may become stale.

For sensitive authorization:

- Prefer current authoritative state.
- Use short TTLs where caching is necessary.
- Invalidate permission caches on changes where practical.

Never allow stale authorization to persist indefinitely.

---

# 40. Permission Changes

Changes to roles and permissions must account for active sessions.

Determine whether:

- Permissions are evaluated on every request.
- Claims are embedded in tokens.
- Sessions require refresh.
- Existing tokens remain valid until expiration.
- Permission changes trigger session revocation.

The behavior must be explicit.

---

# 41. Token Claims

Do not place excessive authorization state into long-lived tokens.

Claims such as:

```text id="r3x7m5"
role=admin
```

may become stale after a role change.

Where authorization is security-critical, the backend must use an authoritative permission source or a carefully designed short-lived claim model.

---

# 42. Resource Visibility

Not every resource must be visible to every authenticated user.

Define visibility separately from mutation permissions where appropriate.

Examples:

```text id="g5m2n8"
Public
Tenant-visible
Role-visible
Owner-visible
Private
Administrative
```

Search, listing, direct access, and mutation must all respect the visibility model.

---

# 43. Read vs Write Permissions

Do not assume write permission automatically implies every read permission.

For sensitive resources, separate:

```text id="q8v4m1"
resource.read
resource.create
resource.update
resource.delete
resource.export
```

Export permissions may require additional authorization beyond read access.

---

# 44. Export Authorization

Data export is a high-impact operation.

Export permissions should consider:

- Tenant scope.
- Dataset scope.
- User role.
- Data sensitivity.
- Volume.
- Audit requirements.

A user who can view individual records does not necessarily need permission to export the entire dataset.

---

# 45. Bulk Authorization

Bulk operations must authorize every affected resource or establish a safe scoped authorization rule.

Do not authorize:

```text id="n6x8p2"
bulk update
```

solely because the user is allowed to modify one resource.

For example:

```text id="b4m7q9"
100 orders
    ↓
Verify authorization scope
    ↓
Process only authorized resources
```

---

# 46. File Authorization

File access must verify authorization independently of the file URL.

Do not treat possession of an object-storage path as authorization.

Use controlled access mechanisms such as:

- Authenticated download endpoints.
- Short-lived signed URLs.
- Resource-level authorization before issuing access.

File permissions must respect tenant boundaries.

---

# 47. Notification Authorization

Notifications must not leak information across users or tenants.

Verify:

- Recipient identity.
- Tenant.
- Notification ownership.
- Visibility.

Do not allow a client to retrieve another user's notifications merely by changing an ID.

---

# 48. Community and Chat Authorization

For KAMPYN community features, authorization must distinguish:

```text id="n8j4q1"
Platform
    ↓
Tenant
    ↓
Community
    ↓
Channel
    ↓
Conversation
    ↓
Message
```

Permissions may include:

```text id="y4c7p8"
community.read
community.create
channel.manage
message.create
message.delete
member.manage
moderation.manage
```

Access must be evaluated at the appropriate scope.

---

# 49. Moderation Authorization

Moderation actions require explicit permissions.

Examples:

- Delete message.
- Remove member.
- Lock channel.
- Review report.
- Suspend user.
- Restore content.

Moderation privileges should be scoped to the communities or tenants the moderator manages.

---

# 50. Auditability

Security-sensitive authorization actions should produce audit records.

Examples:

- Role assignment.
- Permission changes.
- Tenant membership changes.
- Administrative access.
- Data exports.
- Account suspension.
- Moderation actions.
- Break-glass access.

Audit records should capture enough context to establish:

```text id="h3m6r8"
Who
What
When
Which tenant
Which resource
Which action
Result
```

Do not include unnecessary sensitive data.

---

# 51. Authorization Errors

Use appropriate responses for authorization failures.

Typical semantics:

```text id="j7q5m2"
401 Unauthorized
→ Authentication is missing or invalid.

403 Forbidden
→ Authentication exists, but access is not permitted.
```

For sensitive resources, the system may intentionally return `404 Not Found` to avoid revealing resource existence.

The behavior must be deliberate.

---

# 52. Authorization Testing

Authorization must be tested independently from authentication.

Test:

### Identity
- Unauthenticated user.
- Authenticated user.
- Suspended user.

### Roles
- Each relevant role.
- Missing role.
- Multiple roles.

### Permissions
- Permission granted.
- Permission denied.
- Permission revoked.

### Resources
- Own resource.
- Another user's resource.
- Same-tenant resource.
- Different-tenant resource.

### Scope
- Correct campus.
- Wrong campus.
- Correct department.
- Wrong department.

### State
- Allowed state.
- Forbidden state.
- Transition boundary.

### Privilege escalation
- Role manipulation.
- Tenant manipulation.
- ID manipulation.
- Permission manipulation.

---

# 53. Authorization Review Checklist

Before completing an authorization-related change:

- [ ] Authentication is established first.
- [ ] Authorization is enforced server-side.
- [ ] Default-deny behavior is preserved.
- [ ] Required permission is explicit.
- [ ] Resource-level authorization is enforced.
- [ ] Ownership rules are enforced where applicable.
- [ ] Tenant isolation is preserved.
- [ ] Administrative scope is explicit.
- [ ] Cross-tenant access is explicitly controlled.
- [ ] State-based restrictions are enforced.
- [ ] Search access is authorized.
- [ ] Cache access is authorization-safe.
- [ ] Background jobs preserve required scope.
- [ ] Service-to-service permissions are explicit.
- [ ] Role changes are protected.
- [ ] Privilege escalation paths were reviewed.
- [ ] Temporary access has an expiration.
- [ ] Authorization caching does not create unsafe staleness.
- [ ] Export permissions are considered.
- [ ] Bulk operations are safely scoped.
- [ ] File access is independently authorized.
- [ ] Sensitive actions are auditable.
- [ ] Authorization failure behavior is consistent.
- [ ] Tests cover cross-user and cross-tenant access.
- [ ] Documentation reflects the current authorization model.

---

# 54. Final Authorization Principle

Authorization should answer one precise question:

```text id="v7m3k2"
May this authenticated principal
perform this action
on this resource
within this tenant
in this state
under this context?
```

The decision flow should be:

```text id="c9x4q7"
Authenticated Identity
        ↓
Tenant Membership
        ↓
Permission
        ↓
Resource Scope
        ↓
Ownership / Context
        ↓
Resource State
        ↓
Authorization Decision
        ↓
ALLOW / DENY
```

Prefer explicit permissions over implicit trust.

Prefer least privilege over broad access.

Prefer server-side enforcement over client assumptions.

Prefer auditable decisions over hidden administrative behavior.

Authorization must make unauthorized access difficult to introduce accidentally and easy to detect when it is attempted.
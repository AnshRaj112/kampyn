# KAMPYN Engineering Constitution

## Mission

KAMPYN exists to build reliable, secure, scalable, and maintainable infrastructure for universities and the people who depend on it.

The engineering mission is:

> **Build KAMPYN as a dependable platform that makes complex university operations simple for users, administrators, vendors, and institutions while remaining correct, secure, scalable, observable, and maintainable as the system grows.**

Engineering decisions must serve this mission.

Technology exists to support the product.

Architecture exists to protect the system.

Code exists to implement the architecture.

Processes exist to preserve correctness.

No individual technology, abstraction, pattern, or implementation is more important than the mission.

---

# 1. The KAMPYN Engineering Goal

KAMPYN must be engineered so that it can grow from:

```text
One University
        ↓
Multiple Universities
        ↓
Large Multi-Tenant SaaS
        ↓
Self-Hosted University Deployments
```

without requiring the system to be fundamentally rewritten.

The architecture must therefore prioritize:

```text
Correctness
Security
Maintainability
Scalability
Performance
Observability
Portability
Operability
```

These properties are not optional optimizations.

They are part of the product.

---

# 2. User Trust Is the Highest Technical Priority

KAMPYN handles workflows that users and institutions may depend on for:

```text
Food ordering
Payments
Bookings
Inventory
Hostels
Transportation
Complaints
Community
Communication
Institutional operations
```

A technically elegant implementation that produces incorrect results is a failure.

A fast system that loses data is a failure.

A scalable system that leaks tenant data is a failure.

A feature-rich system that cannot be operated reliably is a failure.

Therefore:

> **Correctness and trust take precedence over feature velocity.**

---

# 3. Correctness Over Convenience

Engineering decisions must favor correct behavior over the easiest implementation.

Do not choose an implementation merely because it:

- Requires fewer lines.
- Is faster to write.
- Avoids tests.
- Avoids migrations.
- Avoids architectural review.
- Uses a familiar shortcut.
- Hides complexity instead of solving it.

A shortcut is acceptable only when its consequences are understood and its behavior remains correct.

---

# 4. Security Is a Product Requirement

Security is not a final review step.

Security must exist throughout:

```text
Architecture
 ↓
Design
 ↓
Implementation
 ↓
Testing
 ↓
Deployment
 ↓
Operations
```

Every component must assume that:

```text
Input can be malicious.
Clients can be modified.
Requests can be replayed.
Credentials can be compromised.
Networks can fail.
Services can behave unexpectedly.
```

Security boundaries must therefore be explicit.

---

# 5. Tenant Isolation Is Non-Negotiable

KAMPYN is designed to support multiple universities.

A tenant boundary is therefore a security boundary.

The system must ensure:

```text
Tenant A
   X
Tenant B
```

for:

- Data
- APIs
- Authorization
- Cache
- Search
- Files
- Events
- Jobs
- Notifications
- Integrations
- Analytics
- Administrative operations

A failure of tenant isolation is a critical architectural failure.

---

# 6. The Backend Must Have Clear Ownership

KAMPYN's primary backend runtime is Go.

The architecture must avoid accidental duplication of backend responsibilities across:

```text
Go
Node.js
Next.js
Workers
Scripts
```

Each component must have a defined responsibility.

The system must have authoritative owners for:

```text
Business rules
Persistence
Authentication
Authorization
Search
Caching
Events
Integrations
```

No component should silently become a second source of truth.

---

# 7. One Source of Truth

Every important piece of state must have a clear authoritative owner.

For example:

```text
Transactional State
        ↓
PostgreSQL / MongoDB
```

```text
Cache
        ↓
Redis
        ↓
Optimization
```

```text
Search
        ↓
OpenSearch
        ↓
Derived Read Model
```

```text
Events
        ↓
Facts About State Changes
```

Derived systems must never silently become competing sources of truth.

---

# 8. Architecture Before Implementation

Substantial implementation decisions must begin with understanding the existing architecture.

Before changing the system:

```text
Understand
   ↓
Inspect
   ↓
Search
   ↓
Identify ownership
   ↓
Identify contracts
   ↓
Design
   ↓
Implement
   ↓
Verify
```

Do not begin by writing code.

---

# 9. Existing Code Is Evidence

The repository is the first source of truth for implementation behavior.

Before creating something new:

```text
Search existing code.
Search existing abstractions.
Search existing utilities.
Search existing services.
Search existing contracts.
Search existing tests.
```

Reuse existing behavior when it already satisfies the requirement.

Do not create a duplicate implementation simply because the existing implementation was not immediately discovered.

---

# 10. Reuse Over Duplication

KAMPYN must aggressively avoid unnecessary duplication.

Prefer:

```text
One implementation
        ↓
Multiple consumers
```

over:

```text
Consumer A → Implementation A
Consumer B → Implementation B
Consumer C → Implementation C
```

However, reuse must not create inappropriate coupling.

The objective is:

> **Reuse behavior where the responsibility is genuinely shared, while preserving clear ownership boundaries.**

---

# 11. Simplicity Is Architectural

Simple systems are easier to:

- Understand
- Test
- Debug
- Secure
- Operate
- Scale
- Modify

Do not introduce complexity merely because a more sophisticated architecture is possible.

Examples of unnecessary complexity include:

```text
Premature microservices
Unnecessary abstractions
Generic frameworks
Excessive indirection
Unneeded distributed coordination
Over-engineered factories
Duplicate state systems
```

Complexity must earn its place.

---

# 12. Complexity Must Be Justified

Every significant architectural complexity should answer:

```text
Why does this exist?
What problem does it solve?
What failure mode does it address?
What operational cost does it introduce?
```

If those questions cannot be answered clearly, the complexity should be reconsidered.

---

# 13. Modularity

KAMPYN must be modular.

Modules should have:

- Clear ownership.
- Clear boundaries.
- Explicit dependencies.
- Cohesive responsibilities.
- Minimal coupling.

A module should be understandable without understanding the entire system.

---

# 14. Domain Ownership

Business capabilities should have explicit ownership.

Examples include:

```text
Identity
Tenancy
Orders
Inventory
Food Courts
Bookings
Payments
Notifications
Community
Chat
Search
Files
Reports
```

Ownership must determine:

```text
Who owns the state?
Who owns the rules?
Who owns the API?
Who owns the events?
Who owns the migrations?
```

---

# 15. Dependency Direction

Dependencies must point toward stable abstractions and domain/application boundaries.

The preferred model is:

```text
Presentation
     ↓
Application
     ↓
Domain
     ↓
Infrastructure
```

Infrastructure must not dictate business rules.

Frameworks must not dictate domain design.

Database structure must not automatically become application architecture.

---

# 16. Business Logic Must Have an Owner

Business rules must exist in one authoritative location.

Do not duplicate the same rule across:

```text
Frontend
API
Service
Repository
Worker
Database
```

Client-side checks may improve UX.

Database constraints may enforce integrity.

But the business rule must have an authoritative domain/application owner.

---

# 17. APIs Are Contracts

An API is not merely an HTTP endpoint.

It is a contract between:

```text
Client
Server
SDK
Services
Integrations
```

Contracts must define:

- Inputs
- Outputs
- Errors
- Authentication
- Authorization
- Tenant scope
- Pagination
- Compatibility
- Versioning
- Idempotency

Changing a contract requires deliberate consideration of its consumers.

---

# 18. Data Integrity

Data integrity must be protected at multiple levels.

```text
Application Rules
        +
Domain Invariants
        +
Database Constraints
        +
Transactions
        +
Concurrency Controls
```

No single layer should be assumed to be sufficient for critical data.

---

# 19. Concurrency Is Part of Correctness

KAMPYN must be correct when multiple operations happen simultaneously.

Examples:

```text
Two users buy the last item.
Two users book the same resource.
Two workers process the same job.
Two administrators update the same record.
Two payment callbacks arrive.
```

Concurrency must be designed explicitly.

Do not assume sequential execution.

---

# 20. Performance Is a Design Property

Performance must be considered before implementation rather than after the system becomes slow.

Consider:

```text
Algorithmic complexity
Database queries
Indexes
Network calls
Serialization
Memory
Concurrency
Caching
Search
File processing
```

Avoid unnecessary:

```text
O(n²)
N+1 queries
Unbounded loops
Unbounded concurrency
Unbounded memory
Repeated remote calls
```

---

# 21. Scale Through Boundaries

Scaling KAMPYN should primarily come from clear boundaries rather than indiscriminately adding infrastructure.

Examples:

```text
Stateless APIs
Bounded workers
Database partitioning where justified
Caching
Search read models
Asynchronous processing
Horizontal scaling
```

Do not use infrastructure to compensate for poor application design.

---

# 22. Reliability Over Availability at Any Cost

Not every operation should remain available if doing so would corrupt state.

For critical workflows:

```text
Correct failure
```

is preferable to:

```text
Successful but incorrect operation
```

Examples:

```text
Payment
Inventory
Booking
Authorization
Tenant isolation
```

---

# 23. Failure Is Normal

KAMPYN must assume that:

```text
Databases fail.
Networks fail.
Providers fail.
Pods restart.
Requests timeout.
Messages duplicate.
Events arrive late.
Jobs retry.
Users retry requests.
```

The architecture must define behavior for these cases.

---

# 24. Idempotency

Operations that may be retried must define idempotency where required.

Especially:

```text
Payments
Orders
Bookings
Inventory
Webhooks
Jobs
Event consumers
Notifications
Imports
Exports
```

A retry must not accidentally create duplicate effects.

---

# 25. Eventual Consistency Must Be Explicit

If a system is eventually consistent, that must be intentional.

Examples:

```text
Database
   ↓
Event
   ↓
OpenSearch
```

or:

```text
Database
   ↓
Event
   ↓
Notification
```

Users and developers must not be given the false impression that derived systems are immediately consistent.

---

# 26. Observability Is Part of Correctness

A system that cannot explain what happened cannot be reliably operated.

Important workflows must provide appropriate:

```text
Logs
Metrics
Traces
Audit records
Health information
```

Observability must respect privacy and security.

---

# 27. Errors Must Be Meaningful

Errors should answer:

```text
What failed?
Why did it fail?
Can it be retried?
Who should handle it?
```

Do not hide failures behind:

```text
Something went wrong.
```

Internally, errors should preserve useful context.

Externally, errors must avoid leaking sensitive information.

---

# 28. Testing Is Part of Implementation

Code is not complete merely because it compiles.

Appropriate changes require appropriate verification:

```text
Unit Tests
Integration Tests
API Tests
Contract Tests
Security Tests
Concurrency Tests
Performance Tests
End-to-End Tests
```

The required level depends on the change.

---

# 29. Verification Must Be Honest

Never claim:

```text
Tests passed.
Build succeeded.
Lint passed.
Deployment succeeded.
```

unless it was actually verified.

If verification could not be performed, state that clearly.

---

# 30. Documentation Is Part of the System

Architecture that exists only in someone's memory is not reliable architecture.

Important decisions must be documented.

Documentation should explain:

```text
Purpose
Ownership
Boundaries
Contracts
Tradeoffs
Failure behavior
Operational requirements
```

Code and documentation must evolve together.

---

# 31. Small Changes

Prefer the smallest coherent change that fully solves the problem.

Do not mix:

```text
Feature
+
Unrelated refactor
+
Dependency upgrade
+
Formatting migration
+
Architecture rewrite
```

unless there is a compelling reason.

Small changes are easier to review, test, deploy, and revert.

---

# 32. No Speculative Engineering

Do not implement functionality merely because:

```text
"We might need it later."
```

Future flexibility should come from clean boundaries, not unused complexity.

Build what is required.

Design the boundary so future evolution remains possible.

---

# 33. Technology Is Replaceable

KAMPYN must not become dependent on a particular vendor or framework where a meaningful abstraction is required.

Examples:

```text
Payment Provider
Email Provider
Cloud Storage
Identity Provider
Search Provider
Database Driver
```

Provider-specific implementation belongs behind appropriate boundaries.

---

# 34. Abstractions Must Earn Their Cost

Do not introduce an abstraction simply because abstraction is considered good architecture.

A useful abstraction should provide:

```text
Isolation
Reuse
Testability
Provider independence
Complexity reduction
Clear ownership
```

An abstraction that only adds indirection should be removed.

---

# 35. Security Boundaries Must Be Explicit

Every boundary must define:

```text
Who is calling?
What are they allowed to do?
Which tenant are they operating in?
What data can they access?
What input can they control?
```

This applies to:

```text
HTTP
WebSocket
Events
Jobs
Repositories
Integrations
Admin tools
CLI tools
Self-hosted deployments
```

---

# 36. Least Privilege

Every component should receive only the access it requires.

Examples:

```text
Service Account
Database User
API Key
Cloud IAM Role
Kubernetes Service Account
```

must have minimum necessary permissions.

---

# 37. Secrets Are Never Code

Secrets must never be committed to:

```text
Source code
Git history
Logs
Client bundles
Documentation
Tests
Example configuration
```

unless the value is explicitly fake and clearly non-sensitive.

---

# 38. User Data Is Not Debug Data

Production user data must not be treated as convenient debugging material.

Avoid unnecessary collection or logging of:

```text
Passwords
Tokens
Private messages
Payment information
Sensitive personal information
```

---

# 39. Privacy by Design

Collect, process, retain, and expose only what is necessary.

Every data field should have a reason to exist.

Data retention and deletion must be deliberate.

---

# 40. Operational Simplicity

KAMPYN must remain operable by humans.

Every production capability should have:

```text
Health checks
Logs
Metrics
Alerts where appropriate
Runbooks for critical failures
Recovery procedures
```

A system that requires the original developer to operate it is not sufficiently mature.

---

# 41. Deployment Safety

Deployments must be designed around:

```text
Backward compatibility
Graceful shutdown
Migration safety
Health checks
Rollback
Observability
```

Never assume a deployment is atomic across the entire system.

---

# 42. Database Changes Are Production Changes

Database migrations must be treated as high-impact changes.

They must consider:

```text
Existing data
Existing application versions
Rollback
Lock duration
Migration duration
Backfills
Indexes
Constraints
Zero-downtime deployment
```

---

# 43. Self-Hosting Is a First-Class Requirement

KAMPYN may be deployed by universities independently.

Therefore architecture must consider:

```text
Configuration
Secrets
Database setup
Object storage
Search
Networking
DNS
TLS
Backups
Upgrades
Migrations
Observability
Resource requirements
```

Self-hosting must not require undocumented developer knowledge.

---

# 44. SaaS and Self-Hosted Must Share Core Architecture

The core application should remain consistent between:

```text
KAMPYN Cloud
```

and:

```text
KAMPYN Self-Hosted
```

Deployment topology may differ.

Core domain semantics should not.

---

# 45. SDKs Must Follow the API Contract

SDKs are clients of KAMPYN.

They should not become alternate implementations of backend behavior.

The architecture should be:

```text
SDK
 ↓
API Contract
 ↓
KAMPYN Backend
 ↓
Domain
```

---

# 46. AI Must Follow the Constitution

AI coding agents are implementation assistants.

They must not become architectural authorities.

AI must:

```text
Read the constitution.
Read applicable `.ai/` rules.
Inspect existing code.
Search before creating.
Preserve ownership.
Verify changes.
Report uncertainty.
```

AI must never invent architecture merely because it appears convenient.

---

# 47. Human Ownership

Final architectural decisions remain human decisions.

AI-generated code must be reviewable.

The system should be understandable by engineers who did not generate the original code.

---

# 48. Engineering Discipline

Every change should follow:

```text
Understand
   ↓
Inspect
   ↓
Search
   ↓
Design
   ↓
Implement
   ↓
Test
   ↓
Verify
   ↓
Review
   ↓
Document
```

Skipping steps requires a reason appropriate to the change.

---

# 49. The 200-Line Principle

Production source files should normally remain below:

```text
200 lines
```

This is a design signal, not a mathematical law.

A file exceeding the guideline should trigger review.

Do not split files artificially merely to satisfy the number.

The real objective is:

```text
High cohesion
Low coupling
Clear responsibility
Readable implementation
```

---

# 50. No Redundant Code

Redundancy must be actively challenged.

Before creating new code:

```text
Search.
```

Before creating a helper:

```text
Search.
```

Before creating an abstraction:

```text
Search.
```

Before creating a service:

```text
Search.
```

The question is:

> **Does KAMPYN already have something that owns or solves this responsibility?**

---

# 51. No Hidden Architecture

Architecture must be discoverable from:

```text
Repository structure
Documentation
Contracts
Types
Dependency boundaries
Tests
Configuration
```

Critical behavior must not depend on undocumented conventions.

---

# 52. No Magic Ownership

Every important component should have an identifiable owner.

For:

```text
Database
Repository
Service
Event
API
Integration
Search index
Cache
Job
```

it should be possible to determine:

```text
Who owns it?
Who may change it?
Who depends on it?
What contract does it expose?
```

---

# 53. Prefer Explicitness

When correctness matters, explicit code is preferred over clever code.

Prefer:

```text
Explicit dependency
Explicit transaction
Explicit authorization
Explicit tenant
Explicit state transition
Explicit error
Explicit retry
Explicit timeout
```

over implicit behavior that developers must discover indirectly.

---

# 54. Avoid Cleverness

Code should optimize for:

```text
Understanding
```

rather than:

```text
Showing technical sophistication
```

A straightforward implementation is preferred when it satisfies the same requirements.

---

# 55. Optimize for Change

KAMPYN will evolve.

Architecture should make common changes safe:

```text
New university
New food vendor
New payment provider
New notification provider
New feature
New API client
New deployment environment
New search capability
```

without requiring unrelated parts of the system to change.

---

# 56. Optimize for Failure

Good architecture makes failure understandable.

For every important workflow, engineers should be able to answer:

```text
What if the database fails?

What if the provider times out?

What if the request is retried?

What if the event is duplicated?

What if the worker crashes?

What if the cache is unavailable?

What if the search index is stale?

What if two users act simultaneously?

What if a deployment happens mid-operation?
```

---

# 57. Optimize for Recovery

Correctness includes recovery.

The system should support:

```text
Retry
Replay
Reconciliation
Rollback where appropriate
Rebuild
Reindex
Restore
Backfill
Dead-letter processing
```

where required by the workflow.

---

# 58. Architecture Must Be Observable

If an architectural boundary exists, its behavior should be observable enough to diagnose failures.

For example:

```text
API
 ↓
Service
 ↓
Repository
 ↓
Database
```

should be traceable when a production request fails.

---

# 59. Architecture Must Be Testable

A boundary that cannot be tested independently is often a sign of excessive coupling.

Important boundaries should support appropriate:

```text
Unit testing
Integration testing
Contract testing
Failure testing
```

---

# 60. Architecture Must Be Reversible Where Practical

When choosing between two approaches with similar value, prefer the one that keeps future changes possible without large migrations.

Do not build irreversible infrastructure casually.

---

# 61. But Avoid Abstracting the Future

Future flexibility must not become speculative architecture.

Prefer:

```text
Clean boundary today
```

over:

```text
Huge framework for hypothetical requirements
```

---

# 62. Technical Debt Must Be Visible

When a shortcut is intentionally accepted:

```text
Document it.
```

Identify:

```text
Why it exists.
Risk.
Impact.
Expected follow-up.
```

Hidden technical debt compounds faster than documented technical debt.

---

# 63. Cleanup Is Engineering

When modifying an area, remove obsolete code when safe.

Do not leave:

```text
Dead functions
Unused dependencies
Obsolete feature flags
Old APIs
Temporary hacks
Unused configuration
```

without a reason.

---

# 64. Compatibility Has a Cost

Backward compatibility is valuable but not free.

When maintaining compatibility:

```text
Document the compatibility requirement.
Define the migration path.
Define the removal condition.
```

Do not maintain obsolete behavior indefinitely without purpose.

---

# 65. Performance Must Be Evidence-Based

Do not optimize based solely on assumptions.

Use:

```text
Profiling
Benchmarks
Query plans
Metrics
Tracing
Load testing
Memory measurements
```

when performance is important.

---

# 66. Security Must Be Evidence-Based

Security-sensitive changes should consider:

```text
Threat model
Trust boundaries
Attack surface
Authentication
Authorization
Input validation
Data exposure
Secrets
Dependencies
Deployment
```

Do not declare something secure merely because it looks secure.

---

# 67. Engineering Tradeoffs Must Be Explicit

There is rarely one perfect architecture.

When tradeoffs exist, document:

```text
Decision
Alternatives
Reason
Risks
Consequences
```

Use ADRs where appropriate.

---

# 68. Constitution Hierarchy

This constitution defines the highest-level engineering intent.

Lower-level rules should refine it rather than contradict it.

The intended hierarchy is:

```text id="2p4y8m"
CONSTITUTION
    ↓
AGENTS.md
    ↓
.ai/ Architecture / Engineering Policies
    ↓
Technology-Specific Rules
    ↓
Implementation
```

If a lower-level rule conflicts with the constitution, the conflict must be surfaced and resolved.

---

# 69. Rules Must Remain Consistent

Rules should not be interpreted independently when they create contradictory behavior.

For example:

```text
Performance
```

must not silently override:

```text
Security
```

and:

```text
Developer convenience
```

must not override:

```text
Data integrity
```

When principles conflict, the higher-impact system property must be considered explicitly.

---

# 70. Engineering Priority

When tradeoffs are unavoidable, KAMPYN should generally reason in this order:

```text
1. Safety / Security
2. Data Integrity / Correctness
3. Availability / Reliability
4. Maintainability
5. Performance / Scalability
6. Developer Convenience
7. Implementation Speed
```

This is a decision framework, not permission to ignore performance, usability, or delivery requirements.

---

# 71. Product Reality Matters

Engineering must serve real users.

Do not optimize architecture at the expense of:

```text
Usability
Accessibility
Reliability
Understandability
Operational simplicity
```

Technical excellence that creates an unusable product is not success.

---

# 72. Engineering Quality Is a Product Feature

Users may never see:

```text
Repository boundaries
Database indexes
Event schemas
Concurrency controls
Architecture documents
```

But they experience their consequences through:

```text
Speed
Reliability
Security
Correctness
Recovery
```

Therefore engineering quality directly contributes to product quality.

---

# 73. Long-Term Objective

The objective is not merely:

```text
Make KAMPYN work.
```

It is:

```text
Make KAMPYN work correctly
        ↓
Make it understandable
        ↓
Make it secure
        ↓
Make it observable
        ↓
Make it scalable
        ↓
Make it operable
        ↓
Make it changeable
```

---

# 74. Definition of Mission Success

The engineering mission is successful when KAMPYN can:

```text
Serve users reliably.
Protect tenant data.
Preserve transactional correctness.
Scale without architectural collapse.
Recover from failures.
Support multiple universities.
Support self-hosted deployments.
Evolve without uncontrolled coupling.
Be understood by new engineers.
Be safely modified by humans and AI agents.
```

without requiring constant architectural reinvention.

---

# 75. Final Constitutional Principles

```text id="7m3x8q"
1. KAMPYN exists to solve real university problems reliably.

2. User trust is more important than implementation convenience.

3. Correctness comes before speed of development.

4. Security is part of the architecture, not a final checklist.

5. Tenant isolation is a critical security invariant.

6. Every important state must have one authoritative owner.

7. Business rules must have one authoritative implementation.

8. Architecture must be understood before implementation begins.

9. Existing code must be searched before new code is created.

10. Reuse is preferred over unnecessary duplication.

11. Abstractions must earn their complexity.

12. Modules must have clear ownership and boundaries.

13. Dependencies must have deliberate direction.

14. APIs are contracts, not merely endpoints.

15. Database constraints remain necessary even when application validation exists.

16. Concurrency is part of correctness.

17. Idempotency is required wherever retries can create duplicate effects.

18. Failure is normal and must be designed for.

19. Recovery is part of reliability.

20. Observability is part of production correctness.

21. Performance must be considered during design.

22. Performance optimization must be evidence-based.

23. Unbounded computation, memory, concurrency, and database access are prohibited.

24. Sensitive data must be minimized, protected, and never casually logged.

25. Production systems must be operable without relying on undocumented knowledge.

26. Self-hosting is a first-class architectural concern.

27. SDKs must consume backend contracts rather than duplicate backend logic.

28. Node.js is a supporting runtime unless explicitly given an architectural responsibility.

29. Go remains the primary KAMPYN backend runtime.

30. AI agents are assistants, not architectural authorities.

31. AI-generated changes must remain understandable and reviewable by humans.

32. Tests and verification are part of implementation, not optional cleanup.

33. Documentation is part of the system.

34. Technical debt must be visible rather than hidden.

35. Changes should be small, coherent, and reversible where practical.

36. Speculative architecture is prohibited.

37. Cleverness is never a substitute for clarity.

38. The system should optimize for long-term change, not short-term convenience.

39. Every major engineering decision should improve KAMPYN's ability to
    remain correct, secure, maintainable, scalable, and operable.

40. The ultimate measure of engineering success is a KAMPYN system that
    users and institutions can trust.
```

## Constitutional Statement

```text id="m8r3x7"
KAMPYN shall be engineered as a system that people can trust.

It shall protect their data.

It shall preserve the correctness of their transactions.

It shall remain understandable as it grows.

It shall fail predictably and recover deliberately.

It shall scale without sacrificing its architectural integrity.

It shall support both managed and self-hosted deployments.

It shall favor explicit ownership over hidden coupling,
correctness over convenience, and durable engineering
over short-lived implementation speed.

Every architectural rule, engineering policy, technology choice,
and implementation decision exists in service of this mission.
```
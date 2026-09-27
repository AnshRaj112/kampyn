# KAMPYN Engineering Instructions

> **Purpose:** Define the engineering standards, decision-making principles, and operating rules that apply to all AI-assisted and human development within the KAMPYN repository.
>
> **Scope:** Repository-wide.
>
> **Source of truth:** Detailed policies live under `.ai/`. This file defines the governing principles and points agents toward the appropriate policy.

---

## 1. Source of Truth

The canonical engineering policies live under:

```text
.ai/
```

`AGENTS.md` is intentionally kept at the repository level and should not become a copy of every engineering policy.

Before making a substantial change, consult the relevant `.ai/` documentation.

Typical policy areas include:

```text
.ai/
├── constitution/
├── architecture/
├── standards/
├── performance/
├── security/
├── frontend/
├── backend/
├── data/
├── testing/
├── workflows/
└── skills/
```

Tool-specific instruction files such as:

```text
CLAUDE.md
.cursor/rules/*
```

must not become competing sources of truth.

They should reference or summarize the relevant `.ai/` policies instead of duplicating them.

When project policies change, update `.ai/` first.

---

# 2. Engineering Principles

These principles govern implementation decisions throughout KAMPYN.

## 2.1 Correctness Before Cleverness

- Preserve existing behavior unless the requirement explicitly changes it.
- Never trade correctness for brevity.
- Prefer simple, explicit, maintainable implementations.
- Avoid clever abstractions whose value cannot be clearly explained.
- Handle expected and unexpected failure paths deliberately.
- Do not optimize code by making its behavior harder to understand unless the performance benefit is justified.

Correct code that is understandable is preferable to clever code that requires extensive explanation.

---

## 2.2 Understand Before Modifying

Before changing code:

1. Understand the requirement.
2. Inspect the surrounding architecture.
3. Identify the affected domain.
4. Search for existing implementations.
5. Identify dependencies and consumers.
6. Identify relevant tests.
7. Determine the smallest coherent change.

Do not begin implementation based solely on the filename, task description, or assumptions about how the repository works.

---

## 2.3 Search Before Creating

Before creating a new:

- component
- hook
- service
- repository
- utility
- validator
- type
- API client
- formatter
- middleware
- helper
- database abstraction
- infrastructure abstraction

search the repository for existing functionality.

Search by:

- behavior
- domain concept
- exported symbol
- API contract
- related types
- surrounding implementation

Do not rely solely on filenames.

If equivalent functionality already exists, reuse or extend it.

---

## 2.4 Reuse Before Abstraction

KAMPYN should maintain a strong preference for reuse.

- Do not duplicate business logic.
- Maintain a single source of truth for shared behavior.
- Prefer existing project abstractions when they correctly represent the requirement.
- Extend an established abstraction when appropriate.
- Do not create a second implementation of an existing concept.

However, reuse does not mean forcing unrelated concepts into one abstraction.

Prefer:

```text
well-understood duplication
```

over:

```text
premature abstraction
```

when the common behavior has not yet been sufficiently understood.

---

## 2.5 Cohesion and Responsibility

Each module should have one clear and coherent responsibility.

Avoid:

- god files
- god services
- god components
- catch-all utility modules
- mixed infrastructure and business logic
- mixed presentation and persistence logic
- unrelated domain concepts in the same module

Code should be split around **meaningful responsibilities**, not arbitrary line counts.

---

# 3. File and Module Size

## 3.1 Production Source

The default target for production source files is:

```text
≤ 200 lines
```

200 LOC is an architectural pressure threshold, not a mechanical rule.

Files exceeding 200 lines should be reviewed for decomposition.

Prefer extracting:

- independently meaningful responsibilities
- reusable domain logic
- cohesive utilities
- separate presentation concerns
- separate infrastructure concerns

Do not split files artificially merely to satisfy the line count.

The following may reasonably exceed the threshold when generated or inherently declarative:

- generated code
- database migrations
- snapshots
- generated clients
- machine-produced files
- schema definitions where splitting would reduce clarity

---

## 3.2 Functions

Functions should:

- perform one coherent operation
- have explicit inputs
- have explicit outputs
- avoid unnecessary side effects
- avoid excessive parameters
- remain understandable without excessive comments

When a function requires many parameters, consider a typed options/input object.

Do not split a function into meaningless fragments merely to reduce its line count.

---

# 4. Type Safety

KAMPYN should remain strongly typed.

- Prefer explicit, meaningful types.
- Avoid `any`.
- `any` requires explicit justification.
- Prefer `unknown` when data is genuinely unknown.
- Narrow unknown values before use.
- Do not weaken types merely to satisfy the compiler.
- Avoid unnecessary type assertions.
- Keep domain types reusable and consistent.
- Do not maintain multiple conflicting representations of the same domain concept without a documented reason.

Type safety is part of system correctness.

---

# 5. Validation and Trust Boundaries

All externally controlled data is untrusted.

Validate data at system boundaries, including where applicable:

- HTTP requests
- query parameters
- route parameters
- form submissions
- uploaded files
- webhooks
- queue messages
- external API responses
- environment configuration
- SDK inputs

Never rely exclusively on client-side validation.

After successful validation, internal code should be able to rely on the validated contract.

Do not repeatedly perform identical validation in every internal layer unless the boundary or security model requires it.

---

# 6. Security

Security is a design requirement.

## 6.1 Secrets

Never:

- commit secrets
- hard-code credentials
- expose server secrets to clients
- print tokens
- log passwords
- log private keys
- log payment credentials
- commit local environment files containing secrets

Use the project's approved secret and configuration mechanisms.

---

## 6.2 Authorization

Authentication determines identity.

Authorization determines permission.

Authorization must be enforced server-side.

Never rely on:

- hidden UI elements
- disabled buttons
- client-side role checks
- route visibility
- client-controlled identifiers

for security decisions.

---

## 6.3 Input Security

Validate and sanitize externally controlled data.

Consider relevant threats including:

- injection
- privilege escalation
- unauthorized resource access
- tenant isolation failures
- malformed payloads
- replay
- abuse
- excessive requests

Security-sensitive changes require explicit review.

---

# 7. Performance and Algorithms

Performance decisions must be based on expected scale and workload.

Prefer algorithms and data structures appropriate to the expected input size.

Avoid unnecessary:

```text
O(n²)
O(n³)
repeated linear scans
N+1 queries
repeated network calls
unbounded operations
```

when an appropriate alternative exists.

Prefer, where appropriate:

```text
O(1)
O(log n)
O(n)
O(n log n)
hash-based lookup
index-based lookup
batching
streaming
pagination
caching
precomputation
```

An O(n²) algorithm is not automatically forbidden.

It may be acceptable when:

- input is demonstrably bounded
- the operation is not performance-sensitive
- the alternative introduces disproportionate complexity
- the behavior is explicitly justified

Do not optimize prematurely.

Do not sacrifice correctness or maintainability for theoretical performance.

Performance-sensitive changes should include measurable verification where practical.

---

# 8. Database Engineering

Database access belongs in the appropriate data-access layer.

Never place database implementation details inside:

- UI components
- controllers
- HTTP handlers
- presentation code

unless the architecture explicitly requires it.

Before adding or modifying a database query, consider:

- query frequency
- expected cardinality
- indexes
- joins
- filtering
- pagination
- transaction boundaries
- locking
- consistency requirements
- round trips
- caching opportunities

Avoid:

- N+1 queries
- unnecessary round trips
- unbounded queries
- accidental full-table scans
- excessive indexes
- database access inside loops when batching is possible

Never assume database ordering unless ordering is explicitly requested.

Migrations must be:

- versioned
- tested
- safe
- backwards-compatible where required
- reversible where practical
- evaluated against realistic data volumes

---

# 9. API and Contract Stability

Public contracts include, where applicable:

- HTTP APIs
- SDK interfaces
- events
- queues
- webhooks
- database contracts
- shared types

Do not silently change public contracts.

Prefer backwards-compatible changes.

Breaking changes require:

1. explicit justification
2. identification of affected consumers
3. migration strategy
4. appropriate documentation
5. updated tests

Externally consumed contracts should be versioned when required by the architecture.

Do not expose internal database models directly as public API contracts.

---

# 10. Error Handling

Errors must be deliberate.

- Never silently swallow errors.
- Preserve useful diagnostic context.
- Distinguish expected business errors from unexpected system failures.
- Use the project's standard error model.
- Do not expose internal implementation details to clients.
- Do not hide failures through arbitrary retries or delays.
- Do not use errors/exceptions as ordinary control flow when a clearer mechanism exists.

Every error path should answer:

```text
What failed?
Why did it fail?
Can the caller recover?
What should be logged?
What should the client see?
```

---

# 11. Observability

Production-critical behavior should have appropriate observability.

Use the project's supported mechanisms for:

- structured logging
- metrics
- tracing
- error reporting

Logs should contain enough context to diagnose failures without exposing sensitive information.

Avoid:

- secrets in logs
- excessive logging in high-frequency paths
- duplicate logs for the same failure
- unstructured diagnostic noise

Observability should be designed alongside critical workflows rather than added only after failures occur.

---

# 12. Dependencies

Do not add a dependency without a concrete reason.

Before introducing a package, determine:

1. Does the project already solve this problem?
2. Can the existing stack solve it?
3. Can the platform or standard library solve it?
4. Is the dependency actively maintained?
5. What security risks does it introduce?
6. What transitive dependencies does it introduce?
7. What bundle/runtime cost does it add?
8. What licensing implications exist?
9. Does it complicate deployment?
10. Is it justified by the expected value?

Prefer fewer dependencies when they provide equivalent capability.

Never introduce a dependency simply because it is convenient for a small piece of functionality.

---

# 13. Documentation

The repository should be understandable to an engineer joining the project today.

Document:

- public APIs
- exported behavior where non-obvious
- architectural decisions
- complex algorithms
- important business rules
- external integrations
- non-obvious operational requirements

Architectural decisions belong under:

```text
.ai/decisions/
```

or the project's designated ADR location.

Do not use comments to compensate for unclear code.

Prefer:

```text
clear naming
+
clear structure
+
small modules
```

over excessive comments.

Comments should explain **why**, constraints, trade-offs, or non-obvious behavior.

They should not merely restate what the code already says.

---

# 14. Testing

Testing is part of implementation, not a final cleanup step.

Every production change requires appropriate verification.

Tests should primarily verify behavior and contracts rather than implementation details.

## Required Principles

- Add regression tests for bug fixes.
- Test meaningful business rules.
- Prefer deterministic tests.
- Do not weaken tests merely to make a change pass.
- Do not delete useful tests because they expose a defect.
- Avoid unnecessary mocking of internal implementation.
- Critical workflows require meaningful coverage.

The centralized testing strategy lives under:

```text
.ai/testing/
```

The repository-level test structure lives under:

```text
tests/
```

Do not create scattered test directories without architectural justification.

---

# 15. Scope Control

Every task should produce the smallest coherent change that fully satisfies the requirement.

Do not:

- reformat unrelated files
- rename unrelated variables
- reorganize unrelated directories
- upgrade unrelated dependencies
- refactor unrelated modules
- change coding conventions during feature implementation
- fix unrelated bugs inside a feature diff

If an unrelated issue blocks the requested work:

1. document the issue
2. determine whether it genuinely blocks implementation
3. avoid silently expanding scope

If a broader refactor is necessary, separate it conceptually and explain why.

---

# 16. Cleanup

Do not leave behind:

- dead code
- unused imports
- unused variables
- temporary debugging statements
- commented-out implementations
- obsolete configuration
- obsolete dependencies
- unnecessary feature flags
- TODOs representing work required by the current task

Remove artifacts introduced by the change when they are no longer needed.

Do not remove existing code merely because it appears unused without understanding whether it is externally consumed.

---

# 17. Architectural Consistency

Do not introduce a new pattern when an established project pattern already solves the problem.

When multiple existing patterns conflict, prefer:

1. documented architecture
2. current, non-deprecated patterns
3. the pattern used consistently by the surrounding module
4. the smallest compatible approach

If ambiguity remains and the decision materially affects architecture, do not invent a convention silently.

Document the decision or request clarification.

When intentionally introducing a new pattern, create an architectural decision record.

---

# 18. Dependency Direction

Dependencies should move toward stable abstractions.

The preferred direction is generally:

```text
Presentation
      ↓
Application
      ↓
Domain
      ↓
Infrastructure
```

The exact implementation may differ by subsystem, but architectural boundaries must remain explicit.

Examples:

- UI must not directly access databases.
- Controllers/handlers must not contain persistence implementation details.
- Domain logic should not unnecessarily depend on framework-specific infrastructure.
- Infrastructure implementations should not leak into unrelated layers.
- One domain should not directly manipulate another domain's internal state.

Cross-domain interaction should occur through an intentional boundary such as:

- public service
- interface
- API
- event
- command

depending on the architecture.

---

# 19. Concurrency

Concurrency must be intentional.

For Go services:

- Do not introduce goroutines without understanding their lifecycle.
- Every goroutine must have a clear termination path.
- Respect context cancellation.
- Avoid goroutine leaks.
- Do not share mutable state without appropriate synchronization.
- Protect shared resources explicitly.
- Use channels when they clarify ownership or communication.
- Do not introduce concurrency merely because it appears faster.

For asynchronous processing in any subsystem, define:

```text
ownership
lifecycle
failure behavior
retry behavior
cancellation behavior
resource limits
```

---

# 20. Idempotency

Operations that may be retried must be designed with retry behavior in mind.

This is particularly important for:

- payments
- orders
- bookings
- reservations
- inventory mutations
- webhook processing
- asynchronous jobs
- external API calls with side effects

Never assume a network operation executes exactly once.

Where appropriate, use:

- idempotency keys
- unique constraints
- deduplication
- transactional state transitions
- safe retry semantics

---

# 21. Transaction Boundaries

Transactions should be:

- explicit
- short-lived
- limited to operations requiring atomicity

Do not hold database transactions across:

- external API calls
- network requests
- long-running computation
- user interaction
- arbitrary asynchronous work

Keep transactional boundaries aligned with actual consistency requirements.

---

# 22. No Speculative Implementation

Implement what the current requirement needs.

Do not create functionality merely because it may be useful later.

Avoid creating:

- unused abstractions
- unused interfaces
- unused configuration
- unused database fields
- unused endpoints
- unused feature flags
- unused utilities
- placeholder architecture
- speculative infrastructure

Future flexibility should come from clean boundaries, not unused code.

---

# 23. Abstraction Policy

Do not create an abstraction solely because two pieces of code look similar.

Before introducing an abstraction, verify that:

- it represents a real concept
- it has a clear responsibility
- the behavior is genuinely shared
- reuse is expected
- the abstraction reduces complexity
- the abstraction does not hide meaningful differences

Prefer a small amount of understandable duplication over a premature abstraction.

---

# 24. Evidence Before Assumption

Do not assume:

- an API exists
- a package is installed
- a database field exists
- an endpoint has a particular contract
- an environment variable exists
- a function behaves a certain way
- a dependency supports a feature
- a test is sufficient
- infrastructure behaves in a particular way

Inspect:

- source code
- types
- configuration
- documentation
- tests
- dependency manifests
- runtime behavior

before relying on an assumption.

If evidence cannot be established, state the uncertainty rather than inventing an answer.

---

# 25. AI-Specific Behavior

AI agents working in this repository must behave as engineering assistants, not autonomous architects.

## Before Coding

The agent must:

1. Read the relevant repository instructions.
2. Inspect the affected code.
3. Search for existing implementations.
4. Identify applicable `.ai/` policies.
5. Understand dependencies and contracts.
6. Determine the smallest coherent change.

## During Coding

The agent should:

- follow existing patterns
- reuse existing functionality
- preserve contracts
- keep changes focused
- maintain type safety
- maintain testability
- avoid speculative implementation
- avoid unrelated refactoring

## After Coding

The agent must:

1. Review the changed files.
2. Review the diff.
3. Check for accidental changes.
4. Add or update tests.
5. Run applicable formatting.
6. Run linting.
7. Run type checking.
8. Run relevant tests.
9. Review complexity.
10. Review security implications.
11. Review performance implications.
12. Update documentation where required.

An agent must never claim that a check passed unless it actually ran that check.

An agent must never claim that a requirement is satisfied without verifying it.

---

# 26. Change Workflow

Every meaningful implementation should follow this general lifecycle:

```text
Requirement
    ↓
Understand
    ↓
Discover Existing Code
    ↓
Identify Architecture
    ↓
Identify Contracts
    ↓
Plan Smallest Coherent Change
    ↓
Implement
    ↓
Test
    ↓
Validate
    ↓
Review
    ↓
Document
    ↓
Complete
```

### Understand

Determine exactly what is being requested.

### Discover

Search the repository before creating anything new.

### Plan

Identify:

- affected modules
- dependencies
- contracts
- data flow
- security implications
- testing requirements

### Implement

Make the smallest complete change.

### Test

Add appropriate tests before considering the work complete.

### Validate

Run the relevant:

- formatter
- linter
- type checker
- unit tests
- integration tests
- contract tests
- E2E tests
- build

depending on the change.

### Review

Review:

- correctness
- architecture
- reuse
- complexity
- security
- performance
- maintainability
- scope

### Document

Update documentation when behavior, architecture, contracts, or operational requirements change.

---

# 27. Definition of Done

A change is complete only when all applicable conditions are satisfied.

## Correctness

- [ ] Requirement is implemented.
- [ ] Existing behavior is preserved where required.
- [ ] Edge cases are considered.
- [ ] Failure paths are handled.

## Architecture

- [ ] Correct module/layer was modified.
- [ ] Existing patterns were reused where appropriate.
- [ ] No unnecessary abstraction was introduced.
- [ ] Dependency direction remains valid.

## Code Quality

- [ ] No unnecessary duplication.
- [ ] Naming is clear.
- [ ] Modules remain cohesive.
- [ ] File size remains reasonable.
- [ ] No dead code remains.
- [ ] No temporary debugging code remains.

## Security

- [ ] External input is validated.
- [ ] Authorization is enforced.
- [ ] Tenant boundaries are preserved where applicable.
- [ ] No secrets are exposed.
- [ ] Sensitive information is not logged.

## Performance

- [ ] Algorithmic complexity is appropriate.
- [ ] No accidental O(n²) or worse behavior exists where avoidable.
- [ ] No N+1 database queries were introduced.
- [ ] Relevant indexes were considered.
- [ ] No unnecessary network calls were introduced.

## Testing

- [ ] Appropriate unit tests exist.
- [ ] Appropriate integration tests exist.
- [ ] Contract tests are updated where required.
- [ ] E2E coverage exists for critical user flows where applicable.
- [ ] Regression tests exist for bug fixes.
- [ ] Relevant tests pass.

## Documentation

- [ ] Public behavior is documented where necessary.
- [ ] Architecture documentation is updated where necessary.
- [ ] ADR created when a meaningful architectural decision was introduced.

## Verification

- [ ] Formatting passes.
- [ ] Linting passes.
- [ ] Type checking passes.
- [ ] Build passes where applicable.
- [ ] Final diff contains no unrelated changes.

---

# 28. Final Rule

When in doubt:

```text
Understand before changing.
Search before creating.
Reuse before duplicating.
Prefer simplicity before abstraction.
Validate before trusting.
Measure before optimizing.
Test before declaring complete.
Document before introducing architectural precedent.
```

KAMPYN should remain understandable, maintainable, secure, performant, and approachable to an engineer who joins the project years after the original implementation.
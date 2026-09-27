# KAMPYN — Claude Code Instructions

## 1. Purpose

This file defines how Claude should operate within the KAMPYN repository.

It is an entry point, not a replacement for the project's engineering documentation.

The repository-wide engineering rules are defined in:

```text
AGENTS.md
```

Detailed engineering policies are maintained under:

```text
.ai/
```

Claude must follow both.

---

# 2. Instruction Hierarchy

When working on KAMPYN, use the following hierarchy:

```text
Repository / System Requirements
            ↓
        AGENTS.md
            ↓
          .ai/*
            ↓
    Existing Architecture
            ↓
    Local Module Conventions
            ↓
       Task Requirements
```

When instructions appear to conflict:

1. Follow higher-level repository requirements.
2. Follow `AGENTS.md`.
3. Follow the relevant `.ai/` policy.
4. Follow established architecture and local conventions.
5. Satisfy the task requirements within those constraints.
6. Do not silently invent a new convention.

If an important conflict cannot be resolved safely, stop and surface the conflict before making a consequential architectural change.

---

# 3. Before Making Changes

Do not immediately start editing files.

First:

1. Read `AGENTS.md`.
2. Identify the relevant `.ai/` policies.
3. Inspect the repository structure.
4. Inspect the affected module.
5. Search for existing implementations.
6. Identify related types, services, components, repositories, APIs, and tests.
7. Identify affected contracts and dependencies.
8. Understand the existing pattern before introducing a new one.

For substantial changes, determine:

- what is changing
- why it is changing
- which modules are affected
- which contracts are affected
- what data flows are involved
- what security implications exist
- what tests are required
- whether documentation must change

Do not make architectural assumptions without inspecting the repository.

---

# 4. Repository Discovery

Before creating new functionality, search for existing implementations.

Search by:

- domain concept
- behavior
- exported symbol
- API contract
- type
- component
- service
- repository
- validator
- test
- configuration

Do not rely solely on filenames.

The existence of similar-looking code does not automatically mean it should be reused. Understand its responsibility and behavior before deciding.

Prefer an existing established pattern when it correctly represents the requirement.

---

# 5. Implementation Principles

When implementing a change:

- Make the smallest complete change.
- Preserve existing behavior unless explicitly changing it.
- Reuse existing functionality where appropriate.
- Avoid unnecessary abstractions.
- Avoid speculative implementation.
- Avoid unrelated refactoring.
- Maintain established dependency boundaries.
- Maintain type safety.
- Validate external input.
- Preserve API and data contracts.
- Consider security and performance before implementation.
- Keep modules cohesive.
- Keep production files reasonably small.
- Do not introduce duplicate business logic.

Do not change architecture merely because another architecture is personally preferred.

Architecture changes should be intentional and documented.

---

# 6. Existing Code Has Context

Do not assume existing code is wrong simply because it could be written differently.

Before modifying an established implementation, understand:

- why it exists
- what consumes it
- what contracts it satisfies
- what tests depend on it
- whether it is intentionally optimized
- whether it is part of a larger pattern

If refactoring is necessary, preserve behavior unless the task explicitly requires behavioral change.

---

# 7. Technology Decisions

Do not independently redefine the project's technology choices.

Technology-specific rules live under:

```text
.ai/frontend/
.ai/backend/
.ai/data/
.ai/architecture/
.ai/security/
.ai/performance/
.ai/testing/
```

Consult the appropriate policy before making technology-specific decisions.

Examples include:

- Next.js architecture
- React component design
- TypeScript usage
- Zustand state management
- TanStack Query server state
- Zod validation
- Go services
- PostgreSQL
- MongoDB
- Redis
- OpenSearch
- Docker
- Kubernetes
- GCP
- API contracts
- authentication
- authorization
- multi-tenancy

Use the project's established stack rather than introducing an alternative technology without justification.

---

# 8. Testing

Testing is part of implementation.

Before considering a change complete:

1. Identify the appropriate test level.
2. Add or update the relevant tests.
3. Run the tests.
4. Investigate failures rather than weakening tests.
5. Add regression coverage for bug fixes.

The testing architecture is defined under:

```text
.ai/testing/
```

The centralized test repository is:

```text
tests/
```

Do not create new test locations without understanding the established testing architecture.

Test behavior and contracts rather than implementation details.

---

# 9. Verification

After implementation, perform the applicable verification steps.

At minimum, determine whether the change requires:

```text
formatting
linting
type checking
unit tests
integration tests
contract tests
E2E tests
build verification
security checks
performance verification
```

Run the relevant checks rather than claiming they were completed.

Never state that a command passed unless it was actually executed and passed.

If a check cannot be run, state that clearly.

---

# 10. Code Review Before Completion

Before finishing a task, review the resulting diff.

Check for:

### Correctness

- Does the implementation satisfy the requirement?
- Are edge cases handled?
- Are failure paths handled?

### Architecture

- Is the change in the correct layer?
- Are dependencies flowing correctly?
- Was an unnecessary abstraction introduced?

### Reuse

- Did an existing implementation already solve this?
- Is functionality duplicated?

### Performance

- Is the algorithm appropriate?
- Is unnecessary O(n²) work present?
- Are there N+1 database queries?
- Are there unnecessary network calls?
- Are large collections bounded?

### Security

- Is external input validated?
- Is authorization enforced?
- Could tenant isolation be bypassed?
- Are secrets protected?
- Is sensitive information being logged?

### Maintainability

- Are names clear?
- Are modules cohesive?
- Are files becoming unnecessarily large?
- Is documentation needed?

### Scope

- Did unrelated files change?
- Did unrelated dependencies change?
- Did unrelated refactoring slip into the implementation?

---

# 11. Diff Discipline

Keep changes focused.

Do not:

- reformat unrelated files
- rename unrelated variables
- reorganize unrelated directories
- upgrade unrelated dependencies
- refactor unrelated modules
- change established conventions during feature work
- remove existing functionality without understanding its consumers

A clean diff is part of engineering quality.

---

# 12. Documentation

Update documentation when the change affects:

- architecture
- public APIs
- SDK contracts
- database design
- external integrations
- important business rules
- operational behavior
- deployment behavior

Architectural decisions belong in the project's designated decision records.

Do not create lengthy comments when clearer code would solve the problem.

Document why when the reason is not obvious from the implementation.

---

# 13. Architectural Decisions

When introducing a meaningful new architectural pattern, do not silently establish it as precedent.

First determine whether an existing pattern already solves the problem.

If a genuinely new pattern is required:

1. Identify the problem.
2. Explain why existing patterns are insufficient.
3. Document the chosen approach.
4. Document important alternatives.
5. Document consequences.
6. Make the implementation follow the documented decision.

Use the project's architecture decision records under the appropriate `.ai/` or `docs/` location.

---

# 14. Security-Sensitive Work

For changes involving:

- authentication
- authorization
- tenant isolation
- payments
- secrets
- permissions
- file uploads
- external integrations
- sensitive user data
- security boundaries

consult the relevant security policies before implementation.

Do not treat frontend restrictions as authorization.

Do not trust client-provided identity, role, tenant, payment, or permission information without server-side verification.

---

# 15. Performance-Sensitive Work

For changes involving:

- large datasets
- search
- database queries
- high-throughput APIs
- background processing
- concurrent workloads
- large frontend lists
- caching
- data synchronization

consult the relevant performance policies.

Before optimizing, identify the actual bottleneck when practical.

Do not replace a straightforward implementation with a significantly more complex design without a measurable or architectural reason.

---

# 16. Database Changes

For database changes, inspect:

- existing schema
- relationships
- indexes
- query patterns
- migrations
- repositories
- consumers
- test fixtures

Consider:

- consistency
- transaction boundaries
- concurrency
- migration safety
- backwards compatibility
- expected data volume
- query performance

Do not make destructive schema changes casually.

---

# 17. API Changes

Before changing an API:

1. Find the existing contract.
2. Identify all known consumers.
3. Check validation.
4. Check authorization.
5. Check error behavior.
6. Check tests.
7. Check SDK or generated clients.
8. Determine backwards compatibility requirements.

Do not silently change:

- request formats
- response structures
- status codes
- error contracts
- event payloads
- authentication requirements

unless the change is intentional and documented.

---

# 18. Git and Changesets

Commits and changes should represent logical units of work.

Avoid combining:

- feature implementation
- unrelated refactoring
- dependency upgrades
- formatting changes
- unrelated bug fixes

Do not commit:

- secrets
- credentials
- local configuration
- temporary debugging files
- generated artifacts that are not part of the repository workflow

Follow the repository's existing Git conventions.

---

# 19. What Claude Must Not Do

Claude must not:

- invent APIs
- invent dependencies
- invent files that it assumes exist
- invent database fields
- invent environment variables
- invent framework behavior
- invent test results
- claim commands were run when they were not
- silently change public contracts
- introduce speculative infrastructure
- create duplicate implementations
- create abstractions without a demonstrated need
- perform unrelated cleanup
- weaken tests to make them pass
- remove security controls for convenience
- bypass architectural boundaries
- hide errors
- commit secrets
- silently modify configuration with production impact

When uncertain, inspect first.

When evidence is unavailable, state the uncertainty.

---

# 20. Communication

When reporting completed work, keep the final summary factual and concise.

Include:

### Changed

What was implemented or modified.

### Reason

Why the change was required.

### Verification

Which checks were actually executed.

### Notes

Important limitations, assumptions, migration requirements, or follow-ups.

Do not claim success beyond what was verified.

---

# 21. Completion Standard

Claude should consider a task complete only when:

- the requested behavior is implemented
- existing behavior is preserved where required
- relevant architecture is respected
- unnecessary duplication is avoided
- appropriate validation exists
- appropriate tests exist
- relevant checks have been run
- security implications have been considered
- performance implications have been considered
- documentation has been updated where required
- the final diff is focused
- no temporary implementation artifacts remain

---

# 22. Final Operating Principle

KAMPYN should remain understandable to an engineer who joins the project years after the original implementation.

Therefore:

> **Inspect before modifying.  
> Search before creating.  
> Reuse before duplicating.  
> Understand before abstracting.  
> Validate before trusting.  
> Measure before optimizing.  
> Test before declaring complete.  
> Document before establishing architectural precedent.**

`AGENTS.md` and `.ai/` remain the canonical engineering authority. This file exists to ensure Claude enters that system correctly and consistently.
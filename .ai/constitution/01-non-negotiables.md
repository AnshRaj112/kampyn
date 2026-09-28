# KAMPYN Engineering Constitution: Non-Negotiables

> **Status:** Constitutional  
> **Scope:** All KAMPYN repositories, services, applications, SDKs, infrastructure, and deployments  
> **Authority:** Subordinate only to `constitution/00-mission.md` and applicable legal or safety requirements  
> **Audience:** All engineers, contributors, reviewers, and AI coding agents

---

## 1. Purpose

This document defines the engineering principles that must never be violated during the design, implementation, review, deployment, or maintenance of KAMPYN.

These rules are not optional preferences, stylistic recommendations, or suggestions for future improvement. They are mandatory constraints governing how KAMPYN is built and operated.

Every implementation must comply with these non-negotiables unless an explicitly documented exception is approved through the appropriate human-led architectural decision process.

**No deadline, convenience, performance target, feature request, or AI-generated recommendation automatically justifies violating these rules.**

---

## 2. Security Is Mandatory

Security must be incorporated into every layer of KAMPYN rather than added after implementation.

- Never expose credentials, secrets, private keys, access tokens, or sensitive configuration in source code, logs, client bundles, public repositories, or error responses.
- Never trust client-provided identity, roles, permissions, tenant identifiers, ownership claims, or authorization decisions.
- Every protected operation must enforce server-side authentication and authorization.
- Every resource access must verify the requesting identity's permission to access that specific resource.
- Every tenant-scoped operation must enforce tenant isolation at the appropriate application and data-access boundaries.
- Never weaken authentication, authorization, encryption, validation, or security controls to make a feature work.
- Never introduce known exploitable vulnerabilities without an explicitly approved, time-bounded mitigation plan.
- Never store passwords, authentication secrets, or other sensitive credentials in plaintext.
- Never expose sensitive data through APIs, search indexes, caches, analytics, exports, events, or logs without an explicit access and data-handling policy.
- Never allow untrusted input to directly control database queries, shell commands, file paths, dynamic code execution, or other sensitive operations.
- Never disable security checks merely to make tests, builds, or deployments pass.

Security-sensitive changes must include an appropriate threat assessment, verification, and documentation.

---

## 3. Tenant Isolation Is Absolute

KAMPYN is designed to support multiple universities and institutions. Tenant isolation is a foundational system invariant.

- A tenant's data must never be accessible to another tenant without an explicitly authorized platform-level operation.
- Tenant context must be established through a trusted server-side process.
- Client-supplied tenant identifiers must never be treated as proof of tenant membership or access.
- Tenant boundaries must be enforced in APIs, application services, repositories, database queries, caches, search, object storage, background jobs, events, analytics, and integrations wherever tenant-scoped data is handled.
- Cross-tenant operations must be explicitly identified, narrowly scoped, authorized, and auditable.
- Tenant-specific configuration, branding, identity providers, and feature settings must not leak across tenants.
- Background jobs and asynchronous events must preserve and validate tenant context.
- Tenant switching must trigger appropriate membership and authorization checks.
- Platform administrators must not receive unrestricted access to tenant data by default.
- Self-hosted deployments must preserve the same isolation and authorization guarantees wherever their deployment model requires them.

**No feature may be released if it creates an unmitigated cross-tenant data exposure or authorization path.**

---

## 4. Data Integrity Cannot Be Compromised

KAMPYN manages operational data that universities, vendors, staff, and students rely on. Data correctness is mandatory.

- Every domain must have a clearly defined source of truth for its authoritative data.
- Data must not be silently discarded, corrupted, duplicated, or inconsistently updated.
- Critical business invariants must be enforced at the appropriate domain and persistence boundaries.
- Database constraints must be used where they provide essential integrity guarantees.
- Transactions must protect operations that require atomicity.
- Multi-step operations must explicitly account for partial failure and recovery.
- Concurrent operations must not violate inventory, booking, payment, account, or other domain invariants.
- Destructive data changes must have an explicit authorization, validation, and recovery strategy.
- Migrations must preserve existing data unless data removal is an explicit, approved part of the change.
- Data transformations and imports must validate their input and report rejected or incomplete records.
- Retries must not create unintended duplicate side effects.
- Eventual consistency must be intentional, bounded where feasible, and observable.
- Derived data must be rebuildable from authoritative sources where the architecture requires it.

**A successful API response must not be treated as proof that a business operation was correctly or durably completed.**

---

## 5. The Backend Owns Business Authority

The backend is the authoritative enforcement layer for KAMPYN business operations.

- Business rules must not exist exclusively in frontend code.
- The frontend must not be trusted to enforce permissions, pricing, payment state, inventory availability, booking eligibility, or other business-critical decisions.
- Every state-changing request must be validated and authorized by the backend.
- Application services must coordinate use cases while domain logic enforces business invariants.
- Persistence operations must respect domain ownership and established transaction boundaries.
- API responses must not reveal internal data that the caller is not authorized to access.
- Background jobs must follow the same business rules and authorization context applicable to their operations.
- SDKs and client applications must use documented contracts rather than bypassing server-side enforcement.
- Internal service calls must not be treated as inherently trusted when they cross a security or ownership boundary.

Frontend validation may improve user experience, but it must never replace backend validation or authorization.

---

## 6. Architecture Must Be Preserved

KAMPYN must remain modular, understandable, and capable of evolving without uncontrolled coupling.

- Every module must have a clear responsibility and ownership boundary.
- Dependencies must follow the approved architectural direction.
- Domain logic must not depend directly on transport, UI, or infrastructure implementation details.
- Cross-domain operations must use explicit contracts, application interfaces, or approved event-driven mechanisms.
- Shared modules must not become unstructured collections of unrelated functionality.
- Circular dependencies must not be introduced.
- Infrastructure details must be replaceable without requiring unnecessary changes to business logic.
- New architectural patterns must solve a demonstrated problem rather than introduce speculative complexity.
- Existing architectural conventions must be inspected before introducing new ones.
- Significant architectural changes must be documented through the appropriate decision records.
- A change must not silently undermine established system boundaries.

**No implementation may bypass an architectural boundary merely because doing so is faster or easier.**

---

## 7. No Redundant Code or Unjustified Duplication

Code reuse is required when it improves correctness, consistency, and maintainability.

- Search for existing implementations before creating new ones.
- Reuse established utilities, services, components, contracts, and patterns where they genuinely fit.
- Do not duplicate business rules across modules or layers.
- Do not create multiple sources of truth for the same concept.
- Do not maintain parallel implementations of the same behavior without a documented reason.
- Do not introduce abstractions solely to eliminate a small amount of harmless duplication.
- Do not force unrelated behaviors into a shared abstraction simply because they appear superficially similar.
- Shared code must have a clear owner, stable contract, and cohesive responsibility.
- Refactoring must preserve behavior unless behavior changes are explicitly intended and verified.

The objective is not maximum abstraction. It is minimum unnecessary complexity and consistent behavior.

---

## 8. Correctness Before Convenience

Correctness is more important than implementation speed, superficial simplicity, or convenience.

- Requirements must be understood before implementation begins.
- Assumptions that affect correctness must be verified or explicitly documented.
- Edge cases must be considered for business-critical operations.
- Invalid states must be rejected or made unrepresentable wherever reasonably possible.
- Errors must not be silently ignored when they affect correctness, security, or data integrity.
- APIs must use explicit and consistent contracts.
- Time, money, quantities, identifiers, and state transitions must have well-defined semantics.
- Concurrency-sensitive behavior must be designed for actual concurrent execution.
- Partial failures must be handled deliberately.
- Changes must preserve existing behavior unless a change is explicitly required.
- Unknown behavior must not be presented as verified behavior.

When correctness cannot be established, the uncertainty must be surfaced rather than hidden behind assumptions.

---

## 9. Validation Is Required at Trust Boundaries

All untrusted data must be validated before it is relied upon.

- Validate request structure, types, formats, ranges, and required fields.
- Validate data at external integration boundaries.
- Validate events and background-job payloads before processing them.
- Validate configuration during startup.
- Validate imported files and records before they affect authoritative data.
- Enforce domain invariants independently of transport-level validation.
- Reject unexpected or unauthorized fields where accepting them could create a security or integrity risk.
- Apply appropriate size, complexity, and resource limits to untrusted inputs.
- Use runtime validation where static types alone cannot guarantee correctness.
- Return structured, actionable errors without disclosing sensitive implementation details.

Validation must be consistent with the established contracts and must not be used as a substitute for authorization.

---

## 10. No Uncontrolled Complexity

Every implementation must use appropriate data structures, algorithms, and resource-management strategies.

- Choose algorithms based on expected input size, workload, and operational constraints.
- Avoid unnecessary quadratic or worse time complexity.
- Any intentionally expensive algorithm must have a documented justification and bounded use case.
- Avoid repeated work that can be safely performed once or incrementally.
- Avoid unbounded memory consumption and uncontrolled concurrency.
- Use streaming, batching, pagination, or chunking for large datasets when appropriate.
- Avoid unnecessary network calls, database queries, serialization, and repeated computation.
- Bound queues, worker pools, retries, and resource usage.
- Prevent unbounded goroutine creation and uncontrolled parallel processing.
- Apply query limits, timeouts, and cancellation where appropriate.
- Performance optimizations must preserve correctness and maintainability.

Performance claims must be supported by measurements or clearly identified estimates, not assumptions.

---

## 11. Reliability and Failure Handling Are Mandatory

KAMPYN must behave predictably when components, networks, providers, or dependencies fail.

- External calls must have appropriate timeouts and cancellation.
- Retries must be bounded and use an appropriate backoff strategy.
- Retryable operations must be designed for idempotency where repeated execution is possible.
- Critical workflows must account for partial completion and recovery.
- Background jobs must have defined failure, retry, and terminal-state behavior.
- External provider failures must not silently corrupt internal business state.
- Critical operations must distinguish between success, failure, and uncertain outcomes.
- Services must release resources and shut down gracefully where applicable.
- Health checks must reflect meaningful service health.
- Critical failure modes must be observable.
- Recovery procedures must be documented for important operational workflows.
- Fallback behavior must not bypass security or integrity requirements.

**A failure must not be disguised as a successful operation.**

---

## 12. Concurrency Must Be Explicitly Managed

Concurrent requests, jobs, services, and users must not be assumed to execute sequentially.

- Identify shared mutable state and define its synchronization strategy.
- Prefer database constraints and transaction guarantees for persistent invariants.
- Use locking, optimistic concurrency, atomic operations, or distributed coordination only where justified.
- Prevent race conditions, duplicate processing, and conflicting state transitions.
- Ensure idempotency for operations that may be retried or delivered more than once.
- Define resource ownership and lifecycle for goroutines, channels, workers, and connections.
- Prevent goroutine leaks, deadlocks, uncontrolled contention, and unsafe shared-memory access.
- Account for out-of-order event delivery and concurrent background processing.
- Ensure that concurrency controls work across multiple service instances when distributed execution is possible.
- Test critical concurrency behavior rather than relying solely on code inspection.

Concurrency must be designed as part of the operation, not added as a last-minute patch.

---

## 13. Authentication and Authorization Must Fail Safely

Access-control failures must not result in unintended access.

- Protected operations must deny access when identity or authorization cannot be established.
- Authorization must be checked at the resource and action level where applicable.
- Roles and permissions must be explicit and consistently interpreted.
- Privileged operations must require appropriately scoped authorization.
- Authorization decisions must not rely on hidden frontend controls or client state.
- Expired, revoked, invalid, or malformed credentials must not be accepted.
- Permission caches must account for revocation and relevant changes.
- Sensitive administrative actions must have appropriate auditability.
- Service-to-service access must use explicit identity and permission boundaries.
- Authentication and authorization errors must not leak sensitive details.

Any intentional fail-open behavior requires a documented, narrowly scoped exception and must never compromise tenant isolation or critical data.

---

## 14. Secrets and Sensitive Data Must Be Protected

Sensitive data must be handled according to its classification and purpose.

- Never commit secrets to source control.
- Never include credentials in logs, analytics, exception messages, or client-side bundles.
- Use approved secret-management mechanisms for deployed environments.
- Limit access to sensitive information using least privilege.
- Use encryption in transit and appropriate encryption at rest.
- Avoid collecting or retaining personal data that is not needed for a defined purpose.
- Apply appropriate retention and deletion policies.
- Redact sensitive information from logs, traces, telemetry, and support diagnostics.
- Protect backups, exports, object storage, and generated reports.
- Prevent sensitive information from leaking through search indexes, caches, notifications, or events.
- Rotate compromised or exposed credentials promptly.
- Ensure self-hosted deployment guidance does not encourage insecure secret handling.

Data access must be purpose-limited, authorized, and auditable where required.

---

## 15. APIs Are Contracts

Every public API is a contract with consumers, including KAMPYN applications, SDKs, integrations, and self-hosted deployments.

- API behavior must be explicit, documented, and consistent.
- Request and response schemas must be defined and validated.
- HTTP methods and status codes must reflect the intended operation.
- Errors must use a consistent, machine-readable structure.
- Authentication, authorization, and tenant requirements must be explicit.
- Pagination, filtering, sorting, and idempotency behavior must be predictable.
- Breaking changes must not be introduced silently.
- API compatibility and deprecation must be managed deliberately.
- Internal implementation details must not become accidental public contracts.
- OpenAPI specifications and SDKs must remain consistent with actual API behavior.
- Public contracts must be tested at appropriate boundaries.

A contract change must include an impact assessment for existing consumers.

---

## 16. Database Changes Must Be Deliberate

Database changes can affect data integrity, availability, and compatibility across deployments.

- Every database must have a defined responsibility and data ownership boundary.
- Schema changes must be reviewed for compatibility, migration safety, and operational impact.
- Production migrations must not assume that existing data is clean or uniform.
- Destructive migrations must have an explicit approval and recovery strategy.
- Database constraints must protect essential integrity rules.
- Queries must be reviewed for correctness, authorization scope, and expected performance.
- Database transactions must have clearly defined boundaries.
- Long-running migrations and backfills must be designed to limit operational impact.
- Index changes must account for write overhead, storage, and deployment requirements.
- Database credentials and privileges must follow least privilege.
- Backups and restore procedures must be considered for critical data.
- Application versions and schema versions must remain compatible during rolling deployments where required.

No database change is complete until its deployment and recovery implications are understood.

---

## 17. Source of Truth Must Be Unambiguous

Every important piece of data must have one authoritative owner.

- Domain ownership must be explicit.
- Derived data must not silently become authoritative.
- Caches must not be treated as durable sources of truth.
- Search indexes must be treated as derived read models.
- Analytics and reporting projections must not override transactional records.
- Duplicate representations must have documented synchronization rules.
- Replication and eventual consistency must have defined semantics.
- Data ownership must be respected across modules, services, and integrations.
- Updates must be performed through the authoritative domain boundary.
- Rebuildable projections must have a documented reconstruction strategy.

When data is inconsistent, the system must have a clear method for identifying and resolving the authoritative value.

---

## 18. Testing Is a Release Requirement

Tests are required to establish that changes behave as intended and preserve important system guarantees.

- New behavior must have appropriate tests.
- Bug fixes must include regression coverage where practical.
- Critical business invariants must be tested.
- Authorization and tenant-isolation behavior must be tested.
- Database constraints, transactions, and concurrency-sensitive behavior must be verified.
- API contracts and external integration boundaries must be tested.
- Failure paths and partial failures must be covered where they materially affect reliability.
- Tests must be deterministic, isolated, and maintainable.
- Mocks must not conceal important integration behavior.
- Tests must not be weakened or deleted merely to make a change pass.
- Performance-sensitive changes must be benchmarked or profiled when appropriate.
- Security-sensitive changes must receive security-focused verification.

A passing test suite does not automatically prove correctness, security, or production readiness. Verification must be proportionate to the risk.

---

## 19. Verification Must Be Honest

All claims about implementation and verification must reflect what was actually done.

- Never claim that tests passed if they were not run.
- Never claim that a build, lint, type check, migration, deployment, or security scan succeeded without evidence.
- Clearly distinguish completed work from proposed work.
- Report skipped, failed, blocked, or unavailable checks.
- Do not conceal known defects or unresolved risks.
- Review the final diff for unintended changes.
- Identify assumptions and unverified behavior.
- Do not fabricate test results, logs, files, dependencies, APIs, or deployment outcomes.
- If verification is incomplete, state what remains to be checked.

Engineering trust depends on accurate reporting, not optimistic summaries.

---

## 20. Observability Is Not Optional for Critical Behavior

Critical system behavior must be diagnosable without relying on guesswork.

- Use structured logging where appropriate.
- Propagate request, trace, and correlation identifiers across relevant operations.
- Record meaningful errors and operational failures.
- Measure critical service and workflow behavior.
- Monitor background-job outcomes and queue health.
- Observe database, cache, search, and external-provider dependencies.
- Avoid high-cardinality or sensitive telemetry.
- Apply redaction to logs and traces.
- Define useful health indicators for deployed services.
- Ensure important operational alerts are actionable.
- Document relevant dashboards, alerts, and runbooks.

Observability must support diagnosis while respecting privacy, security, and resource constraints.

---

## 21. No Silent Failure or Error Suppression

Errors must be handled intentionally and transparently.

- Never silently ignore errors that affect business correctness, security, data integrity, or reliability.
- Never return fabricated success when an operation failed or has an uncertain outcome.
- Never swallow exceptions, errors, or failed results without a justified handling strategy.
- Translate errors at appropriate system boundaries without losing useful context.
- Do not expose stack traces, credentials, internal queries, or sensitive details to clients.
- Preserve enough diagnostic context for authorized operators.
- Distinguish validation errors, authorization failures, conflicts, dependency failures, and internal errors.
- Ensure cleanup failures and compensation failures are visible where they matter.
- Use fallback behavior only when its safety and semantics are understood.

Error handling must improve system behavior, not merely silence compiler warnings or logs.

---

## 22. Configuration Must Be Explicit and Validated

Configuration must not introduce hidden or environment-dependent behavior.

- Avoid hardcoded environment-specific values.
- Keep configuration separate from business logic.
- Validate required configuration during startup.
- Fail clearly when critical configuration is missing or invalid.
- Use secure defaults.
- Make feature flags and operational toggles explicit.
- Document configuration names, purposes, valid values, and sensitive status.
- Avoid hidden dependencies on developer machines or local environment state.
- Keep SaaS and self-hosted configuration behavior documented and consistent.
- Never use configuration to bypass essential security or data-integrity guarantees.

Configuration errors should be detected as early as practical.

---

## 23. Dependencies Must Be Justified

Third-party dependencies introduce security, maintenance, operational, and licensing considerations.

- Inspect existing dependencies before adding new ones.
- Prefer established project capabilities when they meet the requirement.
- Add dependencies only when the benefit justifies the added cost and risk.
- Avoid overlapping libraries with redundant responsibilities.
- Review maintenance status, compatibility, security, and licensing.
- Pin or constrain versions according to the repository's dependency-management policy.
- Keep dependencies updated through a deliberate process.
- Do not introduce abandoned, untrusted, or unnecessary packages into production paths.
- Avoid using dependencies to hide fundamental architectural problems.
- Document significant dependencies and their purpose where appropriate.

A dependency must have a clear reason to exist and a reasonable maintenance strategy.

---

## 24. Production Code Must Remain Maintainable

KAMPYN must be understandable by engineers who did not author the original implementation.

- Prefer clear, explicit code over clever or unnecessarily compressed implementations.
- Keep modules small and cohesive.
- Avoid unnecessary nesting, indirection, hidden side effects, and implicit behavior.
- Use meaningful names for types, functions, variables, APIs, and domain concepts.
- Keep functions focused on a clear responsibility.
- Keep production source files at or below 200 lines unless a documented architectural justification supports exceeding the threshold.
- Split large files by responsibility, not arbitrarily.
- Remove dead code and unused abstractions when safe.
- Keep comments focused on intent, constraints, and non-obvious reasoning.
- Avoid comments that merely restate the code.
- Keep formatting and project conventions consistent.
- Ensure public APIs, exported types, and important behavior are documented.

The 200-line guideline is a maintainability constraint, not a reason to fragment cohesive logic into meaningless files.

---

## 25. Changes Must Be Scoped and Reviewable

Every change must have a clear purpose and a controlled impact.

- Understand the requested outcome before modifying code.
- Inspect the relevant architecture, policies, and existing implementation.
- Search for reusable code and existing contracts.
- Avoid unrelated refactoring within feature changes.
- Avoid opportunistic dependency upgrades or broad formatting changes.
- Preserve unrelated user modifications and local work.
- Review staged and unstaged changes before completion.
- Identify migration, compatibility, security, and deployment impacts.
- Keep changes as small as reasonably possible without sacrificing correctness.
- Document meaningful deviations and trade-offs.
- Do not create files, modules, or abstractions without a clear need.

Large changes must be divided into coherent, verifiable steps whenever practical.

---

## 26. Documentation Must Match Reality

Documentation is part of the system's operational and engineering contract.

- Update documentation when behavior, architecture, configuration, or public contracts change.
- Keep setup instructions aligned with the actual repository.
- Document required environment variables and deployment dependencies.
- Keep API and SDK documentation consistent with actual behavior.
- Document non-obvious business rules and important invariants.
- Document migrations, operational procedures, and recovery requirements where relevant.
- Avoid unsupported claims, stale examples, and invented functionality.
- Clearly label planned, experimental, and implemented features.
- Keep architecture decisions discoverable.
- Ensure self-hosting documentation accurately reflects supported deployment behavior.

Documentation must help users and engineers understand what the system actually does.

---

## 27. Privacy and User Trust Must Be Preserved

KAMPYN must handle university, student, staff, vendor, and institutional data responsibly.

- Collect only data required for a legitimate product or operational purpose.
- Limit data access to authorized roles and workflows.
- Avoid unnecessary retention of personal or sensitive information.
- Respect applicable deletion, retention, and access requirements.
- Avoid exposing private user activity through search, analytics, notifications, or community features.
- Protect private messages and restricted community content according to their access model.
- Ensure exports, reports, and administrative tools enforce authorization.
- Keep audit trails for sensitive actions where appropriate.
- Avoid using production personal data in development and testing unless explicitly authorized and appropriately protected.
- Treat privacy risks as architectural concerns, not merely UI concerns.

User trust must not be traded for convenience or unnecessary data collection.

---

## 28. Self-Hosting Must Not Be an Afterthought

KAMPYN's self-hosted deployments must remain a supported architectural consideration.

- Avoid unnecessary dependencies on proprietary control planes for core application behavior.
- Separate deployment-specific configuration from application logic.
- Document required infrastructure, secrets, dependencies, and operational responsibilities.
- Keep service startup, health checks, upgrades, and recovery procedures explicit.
- Provide versioned and compatible deployment artifacts according to the release model.
- Make external provider dependencies configurable where the architecture supports alternatives.
- Avoid leaking SaaS-specific assumptions into tenant-independent domain logic.
- Ensure self-hosted deployments receive appropriate security updates and migration guidance.
- Document limitations and differences between managed and self-hosted deployment modes.
- Do not claim self-hosting support for configurations that have not been verified.

Self-hosting must preserve the relevant security, correctness, and data-integrity guarantees of KAMPYN.

---

## 29. SDKs Must Respect System Boundaries

SDKs are consumer interfaces, not alternate authorities.

- SDKs must use documented and supported API contracts.
- SDKs must not bypass server-side authorization or validation.
- Public SDK behavior must be predictable and versioned.
- SDKs must handle errors, retries, and timeouts according to documented semantics.
- SDKs must not expose secrets or internal implementation details.
- SDK types and schemas must remain aligned with supported API contracts.
- Breaking changes must be managed through compatibility and deprecation practices.
- SDKs must support the documented authentication and tenant-context model.
- SDK examples must use secure configuration patterns.
- SDK behavior must be tested against the relevant API contract.

SDK convenience must never weaken backend authority or security.

---

## 30. AI Agents Must Follow Human-Owned Governance

AI agents may assist with engineering tasks but must not become an independent source of architectural authority.

- Treat the engineering constitution as mandatory.
- Follow the repository's `AGENTS.md`, `.ai/` policies, and relevant technology-specific rules.
- Inspect existing code and documentation before proposing or making changes.
- Never invent APIs, dependencies, files, architectural decisions, or test outcomes.
- Never bypass security, validation, authorization, tenant isolation, or testing requirements.
- Never make broad speculative changes without an explicit requirement.
- Surface conflicts between instructions, existing code, and architectural policies.
- Identify assumptions and uncertainties.
- Do not silently redefine product behavior or domain rules.
- Do not claim that changes were verified when checks were not performed.
- Escalate consequential architectural, security, privacy, and data-integrity decisions for human review.

AI-generated output must be treated as proposed engineering work until reviewed and verified according to the repository's standards.

---

## 31. Exceptions Require Explicit Approval

These non-negotiables must not be silently bypassed.

An exception may be considered only when:

- A genuine technical or operational constraint makes compliance impractical.
- The impact and risks are understood.
- The exception does not violate applicable law or create an unacceptable safety, security, privacy, or data-integrity risk.
- A safer alternative has been considered.
- The exception is documented with its scope, rationale, owner, mitigation, and review or expiry condition.
- The appropriate human decision-maker has explicitly approved it.

Exceptions must be narrow, time-bounded where possible, and revisited when the constraint changes.

**No exception may authorize concealed risk, fabricated verification, or unmitigated cross-tenant data exposure.**

---

## 32. Non-Negotiable Review Checklist

Every substantial change must be reviewed against the following questions:

- [ ] Does it preserve the security boundaries of the system?
- [ ] Does it preserve tenant isolation?
- [ ] Does it protect data integrity and domain invariants?
- [ ] Does the backend remain authoritative for business rules?
- [ ] Does it respect module ownership and dependency direction?
- [ ] Does it avoid redundant code and unjustified abstractions?
- [ ] Are untrusted inputs validated?
- [ ] Are concurrency and idempotency handled where relevant?
- [ ] Are failures, retries, and partial outcomes handled deliberately?
- [ ] Are API and database contracts preserved or deliberately versioned?
- [ ] Are performance and resource limits appropriate for expected workloads?
- [ ] Are critical behaviors observable?
- [ ] Are appropriate tests added and verification honestly reported?
- [ ] Are documentation and operational requirements updated?
- [ ] Does the change preserve privacy and user trust?
- [ ] Is the change scoped, reviewable, and maintainable?
- [ ] Have any exceptions been explicitly documented and approved?

A checklist is a review aid, not a substitute for engineering judgment or evidence.

---

## 33. Enforcement and Conflict Resolution

These rules apply across all KAMPYN codebases and environments, including local development, testing, staging, production, managed SaaS, and self-hosted deployments.

When instructions or implementation choices conflict, use the following hierarchy:

1. Applicable law, safety, and mandatory security requirements.
2. `constitution/00-mission.md` and this constitution.
3. Root `AGENTS.md`.
4. Relevant `.ai/` architecture and engineering policies.
5. Technology-specific rules and repository conventions.
6. Local implementation choices.

A lower-level instruction must not override a higher-level requirement. If a genuine conflict remains unresolved, stop the affected work, explain the conflict, and request a human decision rather than silently choosing a path that may violate the constitution.

Reviewers and AI agents must identify material violations and require correction or an explicitly approved exception before the affected change is considered complete.

---

## 34. Final Principle

KAMPYN must never sacrifice security, tenant isolation, data integrity, correctness, or user trust for speed or convenience.

Every engineering decision must preserve the system's ability to remain understandable, reliable, recoverable, maintainable, and safe as it evolves.

**These non-negotiables define the minimum acceptable standard for engineering KAMPYN.**
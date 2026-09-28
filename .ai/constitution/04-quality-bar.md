# KAMPYN Quality Bar

## 1. Purpose

This document defines the minimum quality standard that every part of KAMPYN must meet before it is considered complete, reliable, maintainable, secure, and ready for use.

The quality bar applies to:

- Application code
- APIs and SDKs
- Frontend interfaces
- Backend services
- Databases and data pipelines
- Authentication and authorization
- Integrations and background jobs
- Infrastructure and deployment
- Tests, documentation, and operational processes
- AI-generated and human-written contributions

Quality is not defined by whether code compiles, a feature works in a happy-path demonstration, or a task is marked complete.

A change is considered complete only when it meets the requirements of its intended use, preserves existing system guarantees, handles expected failures, and can be safely maintained.

## 2. Core Quality Standard

Every KAMPYN change MUST be:

1. **Correct** — Produces the intended result and preserves business and system invariants.
2. **Secure** — Protects identities, data, tenants, transactions, and system boundaries.
3. **Reliable** — Handles expected failures and avoids uncontrolled system behavior.
4. **Maintainable** — Can be understood, tested, modified, and extended without unnecessary complexity.
5. **Efficient** — Uses appropriate algorithms, data structures, queries, memory, and network resources.
6. **Consistent** — Follows established architecture, conventions, contracts, and engineering policies.
7. **Testable** — Has appropriate evidence that its behavior is correct.
8. **Observable** — Provides sufficient diagnostic information without exposing sensitive data.
9. **Documented** — Makes important behavior, contracts, configuration, and operational requirements understandable.
10. **Deployable** — Can be integrated, configured, released, and operated safely.

A change MUST NOT be considered high quality merely because it satisfies some of these requirements while violating a critical one.

## 3. Quality Priority

When engineering requirements conflict, use the following priority order:

1. Safety and security
2. Data integrity and correctness
3. Reliability and recoverability
4. Maintainability and architectural integrity
5. Performance and scalability
6. Developer experience and convenience
7. Implementation speed

Lower-priority goals MUST NOT undermine higher-priority guarantees without an explicit, documented, and approved architectural decision.

Temporary trade-offs MUST have a clear scope, owner, risk assessment, and remediation plan where applicable.

## 4. Correctness

### 4.1 Functional correctness

Every feature MUST:

- Fulfill its documented requirements.
- Produce deterministic and expected results for equivalent inputs, unless nondeterminism is explicitly part of the design.
- Handle valid, invalid, incomplete, and boundary inputs appropriately.
- Preserve existing behavior unless a change is intentional.
- Enforce business rules at the correct authoritative layer.
- Define expected behavior for failures and partial completion.
- Avoid relying on undocumented assumptions.

### 4.2 Business invariants

Business invariants MUST be explicitly enforced and protected.

Examples include:

- A user cannot access another tenant's resources without explicit authorization.
- An order cannot be marked paid solely because a client reports successful payment.
- Inventory cannot be oversold because of concurrent requests.
- A booking cannot be confirmed when its required resource is unavailable.
- A completed transaction cannot be silently duplicated by a retried request.
- A user cannot perform an operation outside their permitted role or resource scope.
- A deleted or revoked resource cannot remain accessible through stale authorization paths.

Invariants MUST be enforced by the appropriate combination of domain logic, authorization, transactions, database constraints, and idempotency controls.

### 4.3 State correctness

Stateful features MUST:

- Define valid states and transitions.
- Reject invalid transitions.
- Handle repeated requests safely.
- Define behavior for concurrent modifications.
- Preserve valid state when operations fail.
- Avoid exposing partially committed state as complete.
- Record important state transitions where auditability is required.

Critical workflows MUST use explicit state machines or an equally clear representation of valid transitions.

## 5. Security and Privacy

Security is a mandatory quality requirement, not an optional enhancement.

Every change MUST:

- Validate and normalize untrusted input.
- Enforce authentication and authorization on protected operations.
- Apply tenant isolation consistently.
- Follow least-privilege principles.
- Protect secrets, credentials, tokens, and sensitive information.
- Avoid leaking sensitive data through logs, errors, URLs, analytics, or responses.
- Use secure transport and appropriate security controls.
- Handle untrusted files, external payloads, and provider responses safely.
- Consider abuse cases, resource exhaustion, and privilege escalation.
- Preserve applicable privacy, retention, and deletion requirements.

Security-sensitive changes MUST include relevant negative tests, such as unauthorized access, cross-tenant access, invalid credentials, replayed requests, and malformed input.

No feature may rely solely on frontend restrictions to enforce security.

## 6. Data Integrity

Data integrity MUST be preserved across application logic, persistence, events, integrations, and background processing.

Every data-related change MUST:

- Identify the authoritative source of truth.
- Define ownership of the data.
- Use appropriate constraints and validation.
- Preserve transactional boundaries.
- Handle concurrent writes safely.
- Prevent unintended duplication, loss, or corruption.
- Define consistency expectations for derived data.
- Consider migration, rollback, recovery, and retention.
- Preserve tenant boundaries in storage and retrieval.

### 6.1 Persistence quality

Database changes MUST:

- Use appropriate data models and types.
- Define required uniqueness and integrity constraints.
- Use indexes that support actual query patterns.
- Avoid unbounded queries and uncontrolled scans.
- Include safe migrations for existing data.
- Consider rollback and compatibility with deployed application versions.
- Avoid introducing undocumented dependencies between domains.

### 6.2 Derived data

Caches, OpenSearch indices, analytics, and other derived representations MUST NOT silently become competing sources of truth.

Derived data MUST have:

- A defined source of truth.
- A synchronization or refresh strategy.
- A known consistency model.
- A recovery or rebuild procedure where applicable.
- A defined behavior when stale, unavailable, or incomplete.

## 7. Architecture and Design Quality

Every change MUST preserve the architectural principles defined in the KAMPYN constitution and supporting architecture documentation.

Code MUST:

- Respect domain ownership and module boundaries.
- Follow dependency direction.
- Keep business logic out of transport and presentation layers.
- Avoid unnecessary coupling between domains.
- Use explicit contracts for module communication.
- Prefer cohesive modules and small, focused components.
- Reuse established patterns when appropriate.
- Avoid unnecessary abstractions and premature distribution.
- Keep technology-specific concerns behind appropriate boundaries.

### 7.1 Complexity

Complexity MUST be justified by a concrete requirement.

Avoid:

- Redundant implementations.
- Duplicate business logic.
- Unnecessary abstraction layers.
- Overly generic frameworks for narrow use cases.
- Hidden side effects.
- Circular dependencies.
- Excessive configuration.
- Unnecessary microservices.
- Overly broad modules with unclear ownership.

Prefer the simplest design that satisfies the real requirements while preserving security, correctness, reliability, and future maintainability.

### 7.2 File and function size

Production source files SHOULD remain within the established 200-line guideline.

Functions, components, services, and modules SHOULD remain small and focused.

When a file exceeds the guideline:

- Review whether it contains multiple responsibilities.
- Extract cohesive responsibilities where doing so improves clarity.
- Avoid splitting code mechanically into arbitrary files.
- Document architectural justification when a larger file is genuinely appropriate.

Line count is a maintainability signal, not a substitute for sound design.

## 8. Code Quality

All production code MUST be readable, consistent, and maintainable.

Code MUST:

- Use clear and meaningful names.
- Have explicit responsibilities.
- Prefer strong types over loosely structured data.
- Validate data at trust boundaries.
- Handle errors explicitly.
- Avoid unnecessary mutation and hidden side effects.
- Avoid dead code, unused dependencies, and unexplained constants.
- Follow established formatting and linting standards.
- Use comments to explain intent, constraints, or non-obvious decisions.
- Avoid comments that merely restate the code.
- Keep public interfaces deliberate and stable.
- Avoid exposing internal implementation details unnecessarily.

Code SHOULD be understandable to another engineer without requiring the original author to explain its intent.

## 9. Performance and Resource Efficiency

Performance MUST be considered during design and implementation, particularly for high-volume or frequently used operations.

Every change MUST:

- Use an appropriate algorithm and data structure.
- Avoid unnecessary repeated computation.
- Avoid unbounded memory consumption.
- Avoid unnecessary network and database calls.
- Avoid N+1 queries and uncontrolled data retrieval.
- Use pagination, batching, streaming, or chunking where appropriate.
- Bound concurrency and resource usage.
- Apply timeouts to external and potentially blocking operations.
- Consider workload size, frequency, and growth.

Expected complexity SHOULD be understood and justified. Quadratic or otherwise expensive operations MUST NOT be introduced casually where efficient alternatives are practical.

Performance-sensitive changes SHOULD be supported by benchmarks, profiling, query analysis, load tests, or other relevant measurements.

Optimization MUST NOT compromise correctness, security, or maintainability without an explicit, reviewed trade-off.

## 10. Reliability and Failure Handling

Features MUST be designed for expected failures, not only successful execution.

Every change MUST consider:

- Database unavailability.
- Network timeouts and connection failures.
- External provider errors.
- Invalid or incomplete responses.
- Duplicate or delayed events.
- Worker restarts and interrupted jobs.
- Concurrent operations.
- Partial workflow completion.
- Resource exhaustion.
- Deployment and configuration failures.

Failure handling MUST be explicit and appropriate to the operation.

Use retries only when safe, bounded, and justified. Operations that may be repeated MUST have suitable idempotency protections.

Critical workflows MUST define how they recover, reconcile, or safely expose incomplete states.

Failures MUST NOT be silently swallowed or falsely reported as success.

## 11. API and Contract Quality

Every public API, event, SDK interface, and integration contract MUST be treated as a product boundary.

Contracts MUST:

- Use clear, consistent naming.
- Define request and response structures.
- Validate untrusted input.
- Use appropriate status and error semantics.
- Enforce authentication, authorization, and tenant scope.
- Define pagination and filtering behavior where applicable.
- Handle idempotency and concurrency where required.
- Avoid exposing internal database models directly.
- Document compatibility expectations.
- Include appropriate tests.

Breaking changes MUST be deliberate, documented, and managed through an approved compatibility or versioning strategy.

SDKs MUST reflect the supported API contract and MUST NOT become an alternate source of business rules or authorization.

## 12. Frontend Quality

Frontend features MUST be functionally correct, accessible, responsive, secure, and consistent with the established design system.

Every frontend change MUST:

- Use appropriate TypeScript types.
- Validate runtime data received from untrusted sources.
- Respect server and client component boundaries.
- Keep server state and client state responsibilities clear.
- Handle loading, success, empty, error, and disabled states.
- Prevent unauthorized actions from being presented as authoritative.
- Handle expired sessions and failed requests appropriately.
- Avoid unnecessary rerenders and expensive client-side processing.
- Support keyboard navigation and accessible interaction.
- Provide appropriate responsive layouts.
- Avoid exposing secrets or sensitive information to the browser.
- Follow established styling and component conventions.

Frontend validation improves usability but MUST NOT replace backend validation, authorization, or business-rule enforcement.

## 13. Testing and Verification

Every change MUST have verification appropriate to its risk, scope, and complexity.

Testing SHOULD cover, where applicable:

- Expected behavior.
- Boundary conditions.
- Invalid inputs.
- Authorization failures.
- Tenant isolation.
- State transitions.
- Data integrity.
- Concurrent operations.
- Idempotency.
- External service failures.
- Retry and recovery behavior.
- Regression scenarios.
- Performance-sensitive behavior.

### 13.1 Verification requirements

Before a change is marked complete:

- Relevant tests MUST be run.
- Relevant formatting, linting, and type checks MUST be run where configured.
- Build or compilation checks MUST be run where applicable.
- Failures MUST be investigated and accurately reported.
- The final diff MUST be reviewed for unintended changes.
- Documentation MUST be updated when affected.

Tests MUST NOT be weakened, removed, skipped, or rewritten solely to make a change pass without a valid and documented reason.

A test passing is evidence for the behavior it checks; it is not proof that the entire system is correct.

### 13.2 Honest verification

Engineers and AI agents MUST accurately distinguish between:

- Tests that were run and passed.
- Tests that were run and failed.
- Tests that were not run.
- Behavior that was manually inspected.
- Behavior that remains unverified.

No implementation may be described as fully tested or verified when the relevant checks were not performed.

## 14. Observability and Diagnostics

Production behavior MUST be diagnosable without exposing sensitive data.

Relevant changes SHOULD provide appropriate:

- Structured logs.
- Metrics.
- Distributed traces.
- Request and correlation identifiers.
- Error context.
- Health signals.
- Audit records for sensitive operations.

Observability MUST support investigation of failures, latency, resource utilization, workflow progress, and external dependency health.

Logs and telemetry MUST avoid credentials, tokens, secrets, and unnecessary personal information.

High-cardinality labels and excessive logging MUST be avoided.

Critical operational failures MUST be visible to the appropriate monitoring and alerting systems.

## 15. Documentation Quality

Documentation MUST be updated whenever a change materially affects:

- Architecture or module ownership.
- Public APIs or SDK contracts.
- Data models or migrations.
- Authentication or authorization.
- Configuration or environment variables.
- Deployment or infrastructure.
- Integrations and provider behavior.
- Background jobs and event contracts.
- Operational procedures.
- User-facing workflows.

Documentation MUST be accurate, consistent with implementation, and specific enough to be useful.

Examples MUST reflect supported behavior. Configuration examples MUST NOT contain real secrets.

Documentation MUST NOT claim capabilities, guarantees, or verification that the system does not provide.

## 16. Dependency and Supply Chain Quality

New dependencies MUST have a clear technical justification.

Before introducing a dependency, consider:

- Whether existing project capabilities already solve the problem.
- Maintenance activity and support status.
- Security history and update process.
- License compatibility.
- Runtime and build impact.
- Transitive dependency risks.
- Operational complexity.
- Compatibility with SaaS and self-hosted deployments.

Dependencies MUST be pinned or constrained according to repository policy and updated through a controlled process.

Unnecessary dependencies MUST NOT be introduced for trivial functionality.

## 17. Infrastructure and Deployment Quality

Infrastructure and deployment changes MUST be reproducible, secure, observable, and recoverable.

Changes MUST consider:

- Environment-specific configuration.
- Secrets management.
- Least-privilege access.
- Resource requests and limits.
- Health checks and graceful shutdown.
- Deployment ordering and compatibility.
- Rollback or recovery procedures.
- Monitoring and alerting.
- Backup and restore implications.
- Cost and capacity impact.
- SaaS and self-hosted deployment requirements where relevant.

Production configuration MUST NOT rely on undocumented manual steps.

Infrastructure changes MUST avoid creating hidden dependencies between services or environments.

## 18. Multi-Tenancy Quality

Every tenant-scoped feature MUST preserve tenant isolation across all applicable system layers.

This includes:

- Authentication context.
- Authorization decisions.
- Application services.
- Database queries.
- Cache keys.
- Search queries and indices.
- Files and object storage.
- Events and background jobs.
- Integrations.
- Analytics and exports.
- Logs and audit records where tenant context applies.

Tenant identity MUST be derived from a trusted server-side context and verified against the caller's access.

Cross-tenant access MUST be explicitly authorized for narrowly scoped platform operations and MUST be auditable where appropriate.

Tenant isolation MUST be tested, not assumed.

## 19. Compatibility and Change Safety

Changes MUST account for existing consumers, stored data, deployed versions, and operational workflows.

Before changing an established contract or data structure:

- Identify affected consumers.
- Review compatibility requirements.
- Plan migrations and deployment ordering.
- Consider mixed-version operation.
- Define rollback or forward-recovery behavior.
- Update tests and documentation.
- Communicate required operational steps.

Destructive changes MUST NOT be performed implicitly.

Feature flags MAY be used to control rollout when appropriate, but MUST have clear ownership, lifecycle, and cleanup requirements.

## 20. AI-Assisted Engineering Quality

AI-generated code and documentation MUST meet the same quality bar as manually authored work.

AI agents MUST:

- Inspect relevant repository structure and existing implementations.
- Read applicable constitution, `AGENTS.md`, and `.ai/` policies.
- Use established project conventions.
- Avoid inventing APIs, dependencies, configuration, or architecture.
- Prefer minimal, cohesive changes.
- Preserve unrelated user work.
- Validate assumptions against available code and documentation.
- Add or update relevant tests.
- Run available verification steps.
- Review the resulting diff.
- Report limitations and unverified behavior honestly.

AI agents MUST NOT treat generated code as inherently correct or bypass review because an implementation was produced by a model.

Human maintainers remain responsible for reviewing and accepting changes.

## 21. Quality Gates

A change MUST pass the applicable quality gates before it is merged or released.

| Gate | Required standard |
|---|---|
| Requirements | Scope and expected behavior are understood |
| Architecture | Boundaries, ownership, and dependencies are respected |
| Correctness | Business rules and invariants are preserved |
| Security | Trust boundaries and access controls are enforced |
| Data integrity | Persistence and consistency guarantees are maintained |
| Code quality | Implementation is clear, cohesive, and maintainable |
| Performance | Resource use and complexity are appropriate |
| Reliability | Expected failures and recovery are considered |
| Testing | Relevant verification is performed |
| Observability | Important behavior can be diagnosed |
| Documentation | Affected contracts and operations are documented |
| Compatibility | Existing consumers and data are considered |
| Diff review | No unintended or unrelated changes remain |
| Completion | Results and outstanding limitations are reported honestly |

Not every gate requires a separate artifact or test suite. The evidence required MUST be proportionate to the change's risk and impact.

## 22. Risk-Based Quality Requirements

The depth of review and verification MUST reflect the consequences of failure.

### Low-risk changes

Examples:

- Copy updates.
- Non-functional styling changes.
- Internal refactoring with unchanged behavior.

Expected controls:

- Scope review.
- Relevant linting or formatting.
- Appropriate tests.
- Diff inspection.

### Medium-risk changes

Examples:

- New UI workflows.
- New API endpoints.
- Non-critical integrations.
- Changes to ordinary data retrieval.

Expected controls:

- Architecture and contract review.
- Relevant unit and integration tests.
- Input and error-path verification.
- Authorization and tenant checks where applicable.
- Documentation updates.

### High-risk changes

Examples:

- Authentication and authorization.
- Payments and financial operations.
- Inventory and booking concurrency.
- Tenant isolation.
- Database migrations affecting critical data.
- Sensitive information handling.
- Production infrastructure and deployment.
- Event-driven workflows with critical state transitions.

Expected controls:

- Explicit threat and failure analysis.
- Detailed architecture and data integrity review.
- Positive and negative tests.
- Concurrency, idempotency, and recovery verification where relevant.
- Migration and rollback or recovery planning.
- Observability and audit review.
- Additional human review before release.

Risk classification MUST be based on potential impact, not implementation size.

## 23. Definition of Done

A task is complete only when all applicable conditions are satisfied:

- [ ] Requirements and acceptance criteria are understood.
- [ ] The implementation follows the approved architecture.
- [ ] Existing behavior and business invariants are preserved.
- [ ] Security, authorization, and tenant isolation are addressed.
- [ ] Data integrity and concurrency risks are addressed.
- [ ] Code is cohesive, readable, and maintainable.
- [ ] Algorithmic and resource costs are appropriate.
- [ ] Expected failure cases are handled.
- [ ] Relevant tests are added or updated.
- [ ] Relevant tests and verification checks are run.
- [ ] No known critical regression remains unresolved.
- [ ] Logs, metrics, traces, or audit records are updated where necessary.
- [ ] Documentation is updated where necessary.
- [ ] Compatibility and deployment implications are reviewed.
- [ ] The final diff contains no unintended changes.
- [ ] Remaining limitations, risks, and unverified areas are disclosed.
- [ ] Required review and approval are completed.

A task MUST NOT be marked complete simply because the main happy path works.

## 24. Quality Exceptions

Exceptions to this quality bar MUST be rare, explicit, and justified.

An exception MUST identify:

- The requirement being waived.
- The reason the requirement cannot currently be met.
- The risks and affected system boundaries.
- The compensating controls.
- The approving owner or reviewer.
- The expected duration.
- The remediation plan, where applicable.

Exceptions MUST NOT be used to normalize poor engineering practices or bypass critical security and data integrity requirements.

Any exception affecting security, tenant isolation, critical data, or financial correctness requires appropriate elevated review.

## 25. Continuous Quality Improvement

Quality is an ongoing responsibility.

The team SHOULD regularly review:

- Production incidents and root causes.
- Repeated defects and regression patterns.
- Performance bottlenecks.
- Security findings.
- Operational complexity.
- Testing gaps.
- Documentation drift.
- Dependency risks.
- Architectural friction.
- Accumulated technical debt.

Lessons from incidents and reviews SHOULD lead to improvements in engineering practices, tests, architecture, observability, or operational controls.

Technical debt MUST be visible and prioritized according to its impact, likelihood, and cost of delay.

## 26. Final Quality Principle

KAMPYN quality is measured by the system's ability to behave correctly, protect users and institutions, preserve data, withstand expected failures, remain understandable, and evolve safely.

A feature is not complete because it works once. It is complete when its behavior is understood, its risks are addressed, its contracts are respected, its failures are handled, and there is credible evidence that it meets its intended requirements.

**KAMPYN MUST optimize for dependable outcomes over superficial completion, durable engineering over unnecessary complexity, and user trust over implementation convenience.**
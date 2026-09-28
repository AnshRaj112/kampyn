# KAMPYN Definition of Done

## 1. Purpose

This document defines the mandatory completion criteria for every engineering task, feature, bug fix, architectural change, integration, deployment, and documentation update in KAMPYN.

The Definition of Done (DoD) establishes a shared standard for determining when work is genuinely complete, rather than merely implemented.

A task MUST NOT be marked as done simply because:
- The code has been written.
- The feature works on the developer's machine.
- The happy path has been demonstrated.
- The application compiles.
- An AI agent has generated an implementation.
- The task has been moved to a completed status.

Work is complete only when its requirements are satisfied, applicable quality and security standards are met, relevant verification has been performed, and remaining risks or limitations are communicated.

This document applies to:
- Features and enhancements.
- Bug fixes and regressions.
- Refactoring and code cleanup.
- APIs, SDKs, and integrations.
- Database and infrastructure changes.
- Authentication and authorization.
- Multi-tenant functionality.
- Background jobs, events, and real-time systems.
- Frontend and backend applications.
- Documentation, testing, and operational work.
- Human-written and AI-assisted contributions.

## 2. Definition of Complete

A task is considered done when it is:

1. **Understood** — Requirements, scope, dependencies, and acceptance criteria are clear.
2. **Designed** — The implementation respects approved architecture and domain ownership.
3. **Implemented** — The intended behavior is complete without unrelated changes.
4. **Correct** — Business rules, data integrity, and existing system invariants are preserved.
5. **Secure** — Applicable authentication, authorization, privacy, and tenant isolation requirements are satisfied.
6. **Resilient** — Expected failure, retry, concurrency, and recovery scenarios are handled.
7. **Tested** — Relevant automated and manual verification has been performed.
8. **Efficient** — Algorithmic complexity and resource consumption are appropriate.
9. **Observable** — Important behavior and failures can be diagnosed where necessary.
10. **Documented** — Relevant contracts, configuration, behavior, and operational requirements are recorded.
11. **Reviewed** — The final changes have been inspected and required approvals obtained.
12. **Deliverable** — The work can be safely integrated, deployed, operated, and maintained.

Not every task requires a separate artifact for every item. The depth of implementation and verification MUST be proportionate to the risk and impact of the change.

## 3. Requirements and Scope

Before implementation, the responsible engineer or agent MUST understand the task.

### 3.1 Requirements

- [ ] The objective of the task is understood.
- [ ] Expected behavior is defined.
- [ ] Acceptance criteria are clear or reasonable assumptions are explicitly documented.
- [ ] Relevant edge cases are identified.
- [ ] Dependencies and affected modules are understood.
- [ ] Existing behavior that must be preserved is identified.
- [ ] Out-of-scope work is understood.
- [ ] Relevant architecture and engineering policies have been reviewed.

Unclear requirements MUST be clarified when they materially affect correctness, security, data integrity, or architecture.

Assumptions MUST NOT be presented as confirmed requirements.

### 3.2 Scope control

- [ ] The implementation addresses the requested problem.
- [ ] Unrelated refactoring is avoided.
- [ ] Unnecessary dependencies are not introduced.
- [ ] Existing user changes are preserved.
- [ ] Changes are limited to the necessary files and modules.
- [ ] Any unavoidable scope expansion is justified and communicated.

The smallest appropriate change SHOULD be preferred, provided it satisfies the required quality bar.

## 4. Architecture and Design

Every change MUST comply with the KAMPYN constitution, `AGENTS.md`, `.ai/` policies, and applicable architecture documentation.

- [ ] The correct domain or module owns the behavior.
- [ ] Dependency direction is respected.
- [ ] Business logic is located in the appropriate layer.
- [ ] Module boundaries and public contracts are respected.
- [ ] Existing implementations have been searched for before creating new ones.
- [ ] Reuse is preferred where it improves cohesion and maintainability.
- [ ] No unnecessary abstraction or architectural complexity is introduced.
- [ ] No circular or inappropriate dependencies are introduced.
- [ ] Data ownership and authoritative sources are clear.
- [ ] Cross-module communication uses established contracts.
- [ ] Architectural decisions with significant long-term impact are documented.

The default architectural approach SHOULD remain a modular system with explicit domain boundaries. Distributed components MUST be introduced only when justified by actual requirements.

## 5. Functional Correctness

Every feature or fix MUST satisfy its intended behavior.

- [ ] Acceptance criteria are met.
- [ ] Expected inputs produce the correct outputs.
- [ ] Invalid inputs are handled safely.
- [ ] Boundary conditions are considered.
- [ ] Business rules are enforced at the appropriate authoritative layer.
- [ ] Existing behavior is preserved unless intentionally changed.
- [ ] State transitions are valid.
- [ ] Partial failures do not produce misleading success states.
- [ ] Duplicate requests are handled appropriately.
- [ ] Error conditions are explicit and predictable.
- [ ] Regression risks have been considered.

### 5.1 Business invariants

All relevant domain invariants MUST remain true.

Examples include:
- Orders cannot be marked paid based only on client-reported status.
- Inventory cannot be oversold through concurrent operations.
- Bookings cannot be confirmed when required resources are unavailable.
- Unauthorized users cannot modify protected resources.
- Users cannot access another tenant's resources without explicit authorization.
- Critical operations cannot be silently duplicated by retries.
- Invalid state transitions cannot be accepted.

Where a change affects a critical invariant, its enforcement and verification MUST be reviewed explicitly.

## 6. Security and Privacy

Every task MUST satisfy the applicable requirements in `constitution/05-security-bar.md`.

- [ ] Trust boundaries are identified.
- [ ] Authentication requirements are satisfied.
- [ ] Server-side authorization is enforced.
- [ ] Resource ownership and access scope are verified.
- [ ] Tenant isolation is preserved.
- [ ] Inputs and external payloads are validated.
- [ ] Injection and unsafe parsing risks are addressed.
- [ ] Sensitive information is protected.
- [ ] Secrets are not exposed or committed.
- [ ] Files and uploads are handled safely where applicable.
- [ ] Errors and logs do not expose sensitive data.
- [ ] Rate limiting and abuse protections are applied where necessary.
- [ ] Security-sensitive operations are auditable where required.
- [ ] Relevant negative security tests are included.

Security requirements MUST NOT be waived merely because a task is small or appears to affect only the frontend.

## 7. Data Integrity and Persistence

Changes involving data MUST preserve authoritative ownership, consistency, and recoverability.

- [ ] The source of truth is identified.
- [ ] Data ownership is respected.
- [ ] Appropriate validation and constraints are applied.
- [ ] Transactions are used where required.
- [ ] Concurrent updates are handled safely.
- [ ] Duplicate writes and retries are handled where applicable.
- [ ] Database queries are scoped and authorized.
- [ ] Migrations are safe for existing data.
- [ ] Indexes and query patterns are reviewed where necessary.
- [ ] Cache and search consistency are considered.
- [ ] Data deletion and retention implications are reviewed.
- [ ] Backup and recovery implications are considered for critical changes.

Derived data, including caches, search indexes, and analytics, MUST NOT silently become an alternative source of truth.

### 7.1 Database changes

Database-related tasks MUST additionally verify, where applicable:

- [ ] Schema changes are compatible with existing application versions.
- [ ] Migration and deployment ordering is considered.
- [ ] Constraints and indexes match intended access patterns.
- [ ] Large backfills or transformations are bounded and recoverable.
- [ ] Rollback or forward-recovery behavior is understood.
- [ ] Sensitive data is protected.
- [ ] Data integrity is tested against representative cases.

Destructive migrations MUST NOT be introduced without an explicit impact assessment and approved migration plan.

## 8. API and Contract Completion

API, event, and SDK changes MUST satisfy the applicable contract requirements.

- [ ] Request and response contracts are defined.
- [ ] Runtime validation is implemented at appropriate boundaries.
- [ ] HTTP methods and status codes are appropriate.
- [ ] Error responses follow established conventions.
- [ ] Authentication, authorization, and tenant scoping are enforced.
- [ ] Pagination, filtering, and sorting are bounded where applicable.
- [ ] Idempotency and concurrency controls are implemented where needed.
- [ ] Internal data models are not unnecessarily exposed.
- [ ] Compatibility requirements are respected.
- [ ] OpenAPI or other contract documentation is updated where applicable.
- [ ] SDKs and consumers remain compatible or receive an explicit migration path.
- [ ] Contract tests are updated where appropriate.

Public contract changes MUST be intentional, documented, and reviewed before release.

## 9. Backend Completion

Backend tasks MUST satisfy the established backend architecture and implementation standards.

- [ ] Transport logic remains separate from application and domain logic.
- [ ] Application services coordinate use cases without absorbing unrelated responsibilities.
- [ ] Repositories own persistence interactions according to established boundaries.
- [ ] Domain invariants are enforced independently of transport details.
- [ ] Context and cancellation are propagated where relevant.
- [ ] Goroutines, workers, and other concurrent operations have bounded lifecycles.
- [ ] Database and external calls use appropriate timeouts.
- [ ] Retries are bounded and safe.
- [ ] Resources are released correctly.
- [ ] Errors are handled and mapped consistently.
- [ ] Configuration and secrets are managed safely.
- [ ] Startup and shutdown behavior is appropriate.
- [ ] Relevant health and observability signals are available.

Backend tasks MUST NOT introduce uncontrolled goroutines, unbounded queues, hidden global mutable state, or unnecessary cross-domain dependencies.

## 10. Frontend Completion

Frontend tasks MUST satisfy the established frontend architecture and quality standards.

- [ ] TypeScript types are accurate and appropriate.
- [ ] Runtime data validation is used where required.
- [ ] Server and client component boundaries are respected.
- [ ] Server state and client state are separated appropriately.
- [ ] TanStack Query is used consistently for server state where applicable.
- [ ] Zustand is used only for suitable client-side shared state.
- [ ] URL state is used where it improves navigation and shareability.
- [ ] Forms validate inputs and handle submission states.
- [ ] Loading, success, empty, error, and disabled states are implemented where relevant.
- [ ] API failures are handled without misleading the user.
- [ ] Authentication and session behavior are handled correctly.
- [ ] The interface is responsive and accessible.
- [ ] Untrusted content is rendered safely.
- [ ] Performance and unnecessary rerenders are considered.
- [ ] Sensitive information is not unnecessarily exposed in the browser.
- [ ] Relevant frontend tests are updated.

Frontend restrictions MUST NOT be treated as a substitute for backend authorization or business-rule enforcement.

## 11. Multi-Tenancy Completion

Tenant-scoped changes MUST verify tenant isolation at every relevant boundary.

- [ ] Tenant context is derived from a trusted source.
- [ ] Membership and access to the tenant are verified.
- [ ] Application operations enforce tenant scope.
- [ ] Database reads and writes are tenant-scoped.
- [ ] Cache keys and values respect tenant boundaries.
- [ ] Search queries and results respect tenant boundaries.
- [ ] File storage and access respect tenant boundaries.
- [ ] Events and background jobs preserve trusted tenant context.
- [ ] Integrations and notifications use the correct tenant context.
- [ ] Analytics, reports, and exports cannot expose unauthorized tenant data.
- [ ] Cross-tenant operations require explicit authorization.
- [ ] Unauthorized cross-tenant attempts are tested.

Tenant isolation MUST be verified through negative tests wherever the task can affect tenant-scoped access.

## 12. Integrations and External Dependencies

Integration tasks MUST handle external systems as independent and potentially unreliable trust boundaries.

- [ ] Provider interfaces and responsibilities are clear.
- [ ] Credentials are stored securely.
- [ ] External inputs and responses are validated.
- [ ] Timeouts are configured.
- [ ] Retries are bounded and appropriate.
- [ ] Idempotency is implemented where necessary.
- [ ] Rate limits and provider quotas are considered.
- [ ] Errors are mapped into safe application-level errors.
- [ ] Partial failures and ambiguous results are handled.
- [ ] Webhooks are authenticated and protected against replay where applicable.
- [ ] Provider-specific behavior does not leak unnecessarily into domain logic.
- [ ] Relevant contract or integration tests are included.
- [ ] Monitoring and operational troubleshooting are considered.

External provider success MUST NOT be assumed when the result is unknown or unverified.

## 13. Events and Background Jobs

Event-driven and asynchronous tasks MUST be safe under retries, delays, duplication, and partial failures.

- [ ] Event or job ownership is clear.
- [ ] Payloads are validated.
- [ ] Tenant and resource context are preserved securely.
- [ ] Idempotency is implemented where needed.
- [ ] Duplicate delivery is handled.
- [ ] Out-of-order processing is considered.
- [ ] Retry and backoff behavior is bounded.
- [ ] Dead-letter or failure handling is defined where applicable.
- [ ] Concurrency and resource consumption are bounded.
- [ ] Cancellation and shutdown behavior are considered.
- [ ] Sensitive payloads are protected.
- [ ] Relevant observability and tests are provided.

Critical state transitions MUST remain correct when events or jobs are delayed, retried, or processed more than once.

## 14. Performance and Resource Usage

Every task MUST use reasonable algorithms and resource patterns.

- [ ] Algorithmic complexity is understood.
- [ ] Appropriate data structures are used.
- [ ] Unnecessary repeated work is avoided.
- [ ] Database queries are efficient and bounded.
- [ ] N+1 queries are avoided.
- [ ] Pagination, batching, streaming, or chunking is used where appropriate.
- [ ] Memory consumption is controlled.
- [ ] Concurrency is bounded.
- [ ] External calls are limited and use timeouts.
- [ ] Large inputs cannot cause uncontrolled resource consumption.
- [ ] Frontend rendering and data fetching are appropriate.
- [ ] Performance-sensitive behavior is measured when necessary.

Expensive algorithms or queries MUST be justified by concrete requirements and MUST NOT be introduced casually when practical alternatives exist.

## 15. Reliability and Recovery

Changes MUST consider expected failures and their effects on the system.

- [ ] Failure modes have been considered.
- [ ] Timeouts are applied where appropriate.
- [ ] Retry behavior is bounded and safe.
- [ ] Partial completion is handled explicitly.
- [ ] State remains valid after failure.
- [ ] Resources are cleaned up.
- [ ] Recovery or reconciliation is defined for critical workflows.
- [ ] External dependency failures are handled safely.
- [ ] The system does not falsely report success.
- [ ] Relevant alerts or health signals are updated where necessary.

Critical workflows SHOULD have clear recovery procedures and must not rely on manual guesswork to restore valid state.

## 16. Testing and Verification

Testing MUST be proportionate to risk and provide meaningful evidence of correctness.

### 16.1 Required verification

For every code change, applicable checks MUST be performed.

- [ ] Relevant unit tests are added or updated.
- [ ] Relevant integration tests are added or updated.
- [ ] Relevant API or contract tests are added or updated.
- [ ] Relevant authorization and tenant tests are added or updated.
- [ ] Relevant failure-path tests are added or updated.
- [ ] Formatting checks are run where configured.
- [ ] Linting checks are run where configured.
- [ ] Type checks are run where configured.
- [ ] Build or compilation checks are run where applicable.
- [ ] Relevant existing tests are run.
- [ ] Test failures are investigated and reported accurately.

Not all checks are applicable to every task. Skipped checks MUST NOT be represented as passed.

### 16.2 Test quality

Tests SHOULD:

- Verify externally observable behavior.
- Cover relevant edge cases and failure modes.
- Avoid unnecessary dependence on implementation details.
- Be deterministic and isolated.
- Use representative fixtures.
- Avoid excessive mocking of the behavior under test.
- Verify security denials as well as successful operations.
- Protect against regressions in critical workflows.

Tests MUST NOT be weakened solely to accommodate a defective implementation.

### 16.3 Manual verification

Manual verification MAY supplement automated testing when appropriate.

Manual checks SHOULD record:
- What was tested.
- The environment or configuration used.
- The observed outcome.
- Any limitations.

Manual verification MUST NOT be described as automated test coverage.

## 17. Observability and Auditability

Changes MUST provide appropriate diagnostic information for their operational risk.

- [ ] Errors are traceable where necessary.
- [ ] Relevant logs are available.
- [ ] Sensitive data is not unnecessarily logged.
- [ ] Request or correlation identifiers are propagated where applicable.
- [ ] Metrics are updated where meaningful.
- [ ] Distributed tracing is considered for cross-service workflows.
- [ ] Security-sensitive operations have appropriate audit records.
- [ ] Operational failures are visible to relevant monitoring systems.

Observability MUST help diagnose behavior without creating additional privacy or security risks.

## 18. Documentation

Documentation MUST be updated when behavior, architecture, contracts, configuration, or operations change.

- [ ] Public API documentation is updated where applicable.
- [ ] SDK documentation is updated where applicable.
- [ ] Architecture documentation is updated for significant design changes.
- [ ] Database and migration documentation is updated where needed.
- [ ] Configuration and environment documentation is updated.
- [ ] Security guidance is updated for security-sensitive changes.
- [ ] Deployment or operational instructions are updated where required.
- [ ] Examples reflect actual supported behavior.
- [ ] Outdated or contradictory documentation is corrected.

Documentation MUST accurately describe implemented behavior and MUST NOT claim unsupported guarantees.

## 19. Dependencies and Supply Chain

Dependency changes MUST be deliberate and reviewed.

- [ ] A clear need for the dependency exists.
- [ ] Existing capabilities were considered first.
- [ ] Compatibility with the project is verified.
- [ ] Maintenance and security risks are considered.
- [ ] License requirements are reviewed where applicable.
- [ ] Unnecessary transitive dependencies are avoided.
- [ ] Lockfiles or equivalent dependency records are updated.
- [ ] Relevant vulnerability checks are performed where available.
- [ ] No secrets or credentials are exposed in dependency or build configuration.

New dependencies MUST NOT be added solely for trivial functionality that can be implemented safely with existing project capabilities.

## 20. Infrastructure and Deployment

Infrastructure and deployment changes MUST be operationally safe.

- [ ] Configuration is explicit and validated.
- [ ] Secrets are handled through approved mechanisms.
- [ ] Access permissions follow least privilege.
- [ ] Network exposure is reviewed.
- [ ] Resource requests and limits are appropriate.
- [ ] Health checks and graceful shutdown are considered.
- [ ] Deployment ordering and compatibility are considered.
- [ ] Rollback or forward-recovery procedures are understood.
- [ ] Monitoring and alerting are appropriate.
- [ ] Backup and recovery implications are reviewed.
- [ ] SaaS and self-hosted implications are considered where applicable.
- [ ] Manual operational steps are documented.

Production deployments MUST NOT depend on undocumented configuration or unsafe defaults.

## 21. Compatibility and Migration

Changes MUST consider all affected consumers and deployed versions.

- [ ] Existing API and event consumers are identified.
- [ ] Compatibility requirements are reviewed.
- [ ] Database migration impact is understood.
- [ ] Mixed-version operation is considered where applicable.
- [ ] Deployment ordering is safe.
- [ ] Rollback or forward-recovery behavior is defined.
- [ ] SDKs and client applications have a supported migration path.
- [ ] Deprecated behavior is documented where applicable.
- [ ] Breaking changes receive the required approval.

Destructive or incompatible changes MUST NOT be introduced accidentally.

## 22. Code Review and Diff Hygiene

Before completion, the final changes MUST be inspected.

- [ ] The diff has been reviewed.
- [ ] No unrelated files or changes are included.
- [ ] No secrets, debug artifacts, or temporary files are included.
- [ ] No generated files are modified unnecessarily.
- [ ] No dead code or unused dependencies remain.
- [ ] Error handling and edge cases have been reviewed.
- [ ] Security-sensitive paths have been inspected.
- [ ] Tests reflect the final implementation.
- [ ] Documentation matches the final behavior.
- [ ] Review feedback has been addressed or explicitly resolved.

Review MUST focus on correctness, security, data integrity, architecture, performance, maintainability, and compatibility, rather than formatting alone.

## 23. Risk-Based Completion Requirements

The depth of the Definition of Done MUST reflect the potential impact of failure.

### 23.1 Low-risk changes

Examples:
- Copy changes.
- Non-functional styling updates.
- Internal refactoring that does not change behavior.

Minimum expectations:
- Scope and diff review.
- Relevant formatting or lint checks.
- Appropriate tests where behavior could be affected.
- Documentation updates where necessary.

### 23.2 Medium-risk changes

Examples:
- New user-facing workflows.
- New non-critical API endpoints.
- Ordinary data retrieval changes.
- Non-critical external integrations.

Additional expectations:
- Architecture and contract review.
- Relevant unit and integration tests.
- Validation and error-path verification.
- Authorization and tenant checks where applicable.
- Compatibility and documentation review.

### 23.3 High-risk changes

Examples:
- Authentication and authorization.
- Tenant isolation.
- Payments and financial operations.
- Inventory and booking concurrency.
- Critical database migrations.
- Sensitive data processing.
- Production infrastructure.
- Event-driven critical state transitions.

Additional expectations:
- Explicit risk and threat analysis.
- Detailed architecture and data integrity review.
- Positive and negative security tests.
- Concurrency and idempotency verification where relevant.
- Failure, recovery, and reconciliation testing where applicable.
- Migration and deployment planning.
- Observability and audit review.
- Required elevated human approval.

Risk MUST be determined by potential impact and exposure, not by the number of lines changed.

## 24. AI-Assisted Task Completion

AI agents MUST follow the same Definition of Done as human contributors.

AI agents MUST:

- Read relevant project instructions before implementation.
- Inspect existing code and architecture.
- Search for reusable implementations.
- Avoid inventing APIs, dependencies, or project conventions.
- Make the smallest appropriate change.
- Preserve unrelated user work.
- Implement relevant tests.
- Run applicable verification checks.
- Inspect the final diff.
- Update affected documentation.
- Report what was changed and what was verified.
- Clearly disclose skipped checks, unresolved failures, and remaining risks.

AI agents MUST NOT claim a task is complete merely because code was generated or a partial implementation was produced.

Human maintainers retain responsibility for reviewing and accepting AI-generated work.

## 25. Completion Report

When a task is completed, the responsible engineer or agent SHOULD provide a concise completion report containing:

### Summary
- What was implemented or changed.
- Which requirements or defects were addressed.

### Technical impact
- Relevant modules, contracts, data models, or infrastructure affected.
- Important architectural decisions, if any.

### Verification
- Tests and checks actually run.
- Results of those checks.
- Manual verification performed, if any.

### Outstanding items
- Known limitations.
- Checks not performed.
- Unresolved risks or follow-up work.
- Required deployment or migration steps.

The completion report MUST distinguish implemented behavior from intended or unverified behavior.

## 26. Exceptions and Incomplete Work

If a task cannot meet all applicable completion criteria, it MUST NOT be reported as fully done.

The remaining work MUST be identified with:
- The unmet requirement.
- The reason it remains unmet.
- The impact and risk.
- Any temporary mitigation.
- The next action and responsible owner, where known.

Exceptions MUST be explicit, approved at the appropriate level, and consistent with the constitution's security and correctness priorities.

Critical security, tenant isolation, and data integrity requirements MUST NOT be casually waived to satisfy a delivery deadline.

## 27. Final Definition of Done Checklist

{@body const sections = [
  {title:"Requirements and architecture",items:["Requirements and acceptance criteria are clear","Relevant instructions and architecture have been reviewed","Domain ownership and dependencies are correct","Scope is controlled and unrelated changes are excluded"]},
  {title:"Implementation and correctness",items:["Expected behavior is implemented","Business invariants and existing behavior are preserved","Errors, edge cases, and state transitions are handled","Code is readable, cohesive, and maintainable"]},
  {title:"Security and data integrity",items:["Authentication and authorization are correct","Tenant isolation is preserved","Inputs, secrets, and sensitive data are protected","Persistence, concurrency, and consistency are safe"]},
  {title:"Reliability and performance",items:["Failure and recovery paths are considered","Retries and idempotency are safe where applicable","Algorithms and resource usage are appropriate","Timeouts, cancellation, and cleanup are handled"]},
  {title:"Verification and delivery",items:["Relevant tests and checks are run","Results and limitations are reported honestly","Observability and audit requirements are met","Documentation and compatibility are addressed","Final diff is reviewed and approvals are complete"]}
]}
{@body const [checked,setChecked] = DIL.useState([])}
<box border radius="lg" padding={3} gap={3}>
  <row align="center" justify="between">
    **Task completion checklist**
    <text color="secondary" size="sm" tabularNums>{checked.length} / {sections.reduce((n,s)=>n+s.items.length,0)} complete</text>
  </row>
  <box background="surface-tertiary" height="6px" radius="full" clip>
    <box background="#16a34a" width={(checked.length/sections.reduce((n,s)=>n+s.items.length,0)*100)+"%"} height="6px" radius="full"/>
  </box>
  {#each sections as section,i}
    <box gap={2}>
      <text weight="semibold">{section.title}</text>
      {#each section.items as item,j}
        <checkbox checked={checked.includes(i+"-"+j)} onChange={v=>setChecked(prev=>v?[...prev,i+"-"+j]:prev.filter(x=>x!==i+"-"+j))}>{item}</checkbox>
      {/each}
    </box>
    {#if i<sections.length-1}<divider/>{/if}
  {/each}
  {#if checked.length===sections.reduce((n,s)=>n+s.items.length,0)}
    <badge color="success">All checklist items marked complete</badge>
    <text color="secondary" size="xs">Confirm that each item has been verified and that any required approvals are complete before marking the task done.</text>
  {/if}
  <row justify="end">
    <button size="sm" variant="outline" color="secondary" onClick={()=>setChecked([])}>Reset checklist</button>
  </row>
</box>

## 28. Final Principle

The Definition of Done exists to ensure that KAMPYN delivers dependable engineering outcomes rather than superficial task completion.

A task is done when the implementation satisfies its requirements, respects the system's architecture, protects users and institutional data, handles expected failures, and has been verified to a level appropriate for its risk.

**KAMPYN MUST define completion by evidence, preserve quality under delivery pressure, and never confuse implemented code with production-ready work.**
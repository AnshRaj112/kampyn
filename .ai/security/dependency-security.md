# Dependency Security

## 1. Purpose

This document defines the mandatory security standards for selecting, installing, updating, auditing, and maintaining third-party dependencies across KAMPYN.

Dependencies introduce external code, transitive risk, licensing obligations, supply-chain exposure, and potential vulnerabilities. Every dependency must have a clear purpose, an accountable owner, and a maintenance strategy.

The goal is to minimize the dependency attack surface while keeping the platform secure, maintainable, and compatible with SaaS and self-hosted deployments.

## 2. Core Principles

All dependency management MUST follow these principles:

- **Minimize dependencies:** Prefer built-in platform capabilities and existing approved dependencies over adding new packages.
- **Verify before adoption:** Assess the security, maintenance, provenance, and licensing of a dependency before introducing it.
- **Pin and reproduce:** Ensure dependency installation produces predictable, reviewable builds.
- **Audit continuously:** Detect known vulnerabilities in direct and transitive dependencies throughout their lifecycle.
- **Patch promptly:** Prioritize remediation according to exploitability, severity, exposure, and business impact.
- **Protect the supply chain:** Treat packages, registries, maintainers, build tools, and update automation as part of the trusted computing base.
- **Maintain accountability:** Every production dependency must have a defined purpose and an owner responsible for its lifecycle.
- **Fail securely:** Dependency failures, compromised packages, and unavailable registries must not cause security controls to be bypassed.
- **Preserve reproducibility:** Builds must be traceable to reviewed source, dependency versions, and build configuration.

Security MUST NOT depend solely on automated vulnerability scanners.

## 3. Scope

This policy applies to all dependencies used by:

- Next.js and React frontend applications.
- Go backend services and libraries.
- Node.js services, tooling, and SDKs.
- Python utilities and background workers.
- PostgreSQL, MongoDB, Redis, and OpenSearch integrations.
- Authentication, authorization, encryption, and security components.
- API clients, payment integrations, notification providers, and external SDKs.
- Docker images, operating system packages, and container base images.
- Kubernetes operators, Helm charts, and deployment tooling.
- CI/CD workflows, build systems, code generators, and developer tools.
- Testing, linting, formatting, monitoring, and observability tools.
- Self-hosted university deployments and customer-facing SDKs.

This policy covers direct, transitive, development, build-time, test-time, runtime, and infrastructure dependencies.

## 4. Dependency Classification

Every dependency MUST be classified according to its purpose and exposure.

| Category | Description | Examples |
|---|---|---|
| Runtime | Required by production application execution | HTTP routers, database drivers |
| Security-critical | Handles security-sensitive operations | Authentication, cryptography, token validation |
| Build-time | Required to compile or bundle applications | Compilers, bundlers |
| Development | Used by developers locally | Linters, formatters |
| Testing | Used for automated tests | Test frameworks, mocks |
| Infrastructure | Required by deployment or runtime infrastructure | Base images, controllers |
| SDK / Client | Distributed to external consumers | KAMPYN API SDKs |
| Transitive | Installed indirectly through another dependency | Nested package dependencies |

Security-critical dependencies MUST receive additional scrutiny because vulnerabilities in them can directly affect authentication, authorization, data confidentiality, integrity, or availability.

Development-only dependencies MUST NOT be assumed harmless. Build tools, plugins, and code generators can execute arbitrary code and may affect production artifacts.

## 5. Dependency Approval

### 5.1 Before Adding a Dependency

Before introducing a new dependency, developers MUST:

1. Identify the exact problem the dependency solves.
2. Check whether the language, framework, standard library, or an existing approved package already provides the required functionality.
3. Assess whether the dependency introduces unnecessary complexity or overlapping functionality.
4. Review its maintenance history, release activity, and support status.
5. Review known security advisories and vulnerability history.
6. Verify its official source, repository, package identity, and distribution channel.
7. Review its direct and transitive dependency footprint.
8. Check its license and compatibility with KAMPYN's distribution model.
9. Assess its permissions, network access, file-system access, and execution behavior where relevant.
10. Identify an owner and a strategy for updates and eventual removal.

A dependency MUST NOT be added solely because it is popular, convenient, or commonly used.

### 5.2 Approval Requirements

Dependencies that affect authentication, authorization, cryptography, payments, tenant isolation, secrets, or sensitive data MUST receive explicit security review before production adoption.

Dependencies with native code, install scripts, privileged execution, or broad system access MUST receive additional scrutiny.

The reviewer MUST consider whether the same functionality can be implemented safely with an existing dependency or a platform-native capability.

### 5.3 Dependency Decision Record

For significant or security-critical dependencies, record:

- Package name and ecosystem.
- Exact version or versioning policy.
- Official source and repository.
- Intended functionality.
- Reason for adoption.
- Alternatives considered.
- License.
- Security and maintenance assessment.
- Direct and transitive dependency impact.
- Responsible owner.
- Update and replacement strategy.

The record SHOULD be maintained alongside the relevant service or architecture documentation.

## 6. Trusted Sources and Package Provenance

Dependencies MUST be obtained from trusted, expected sources.

Developers MUST:

- Use official package registries or approved internal mirrors.
- Verify package names, publishers, repository links, and namespaces.
- Prefer packages with verifiable release provenance where supported.
- Verify checksums or cryptographic signatures when provided and supported by the ecosystem.
- Use approved registry configurations in CI/CD.
- Review unexpected changes in package ownership, repository URLs, maintainers, or release patterns.
- Avoid downloading and executing packages from arbitrary URLs.
- Avoid unverified binaries and unofficial package mirrors.

### 6.1 Typosquatting and Dependency Confusion

Developers MUST check package names carefully before installation.

For scoped packages and internal modules:

- Reserve and protect internal package namespaces.
- Use explicit registry configuration for private scopes.
- Prevent public registry fallback for internal package names.
- Verify that internal package names cannot be impersonated through public registries.
- Restrict publishing permissions to authorized maintainers.
- Require review for changes to registry and package-source configuration.

Internal packages MUST have clearly defined ownership and controlled release permissions.

### 6.2 Package Installation Scripts

Packages that execute scripts during installation, compilation, or setup MUST be reviewed for necessity and behavior.

Where feasible:

- Disable lifecycle scripts for installations that do not require them.
- Restrict build environments and execution permissions.
- Review changes to installation scripts during dependency updates.
- Prevent untrusted package scripts from accessing production secrets.
- Keep CI jobs that install dependencies isolated from privileged deployment jobs.

Installation scripts MUST NOT be granted unrestricted access to credentials, production environments, or sensitive files.

## 7. Versioning and Lockfiles

### 7.1 Version Pinning

All applications and services MUST use deterministic dependency resolution.

- Commit lockfiles for supported package managers.
- Keep lockfiles synchronized with dependency manifests.
- Pin container base images to immutable digests for production builds where practical.
- Use explicit versions for critical infrastructure dependencies.
- Avoid floating production dependency references such as `latest`.
- Review version-range policies for libraries distributed to external consumers.

Version ranges in manifests MAY be used where ecosystem conventions require them, but the lockfile MUST capture the exact resolved versions for application builds.

### 7.2 Lockfile Integrity

Lockfiles MUST:

- Be committed to version control.
- Be changed through approved package-manager operations.
- Be reviewed as part of dependency updates.
- Be checked for unexpected registry URLs or package source changes.
- Be included in reproducible CI builds.

Manual lockfile editing SHOULD be avoided unless the package manager cannot safely perform the required correction.

Unexpected large lockfile changes MUST be investigated before merging.

### 7.3 Reproducible Builds

CI/CD MUST install dependencies using the repository's committed lockfiles and frozen or equivalent installation modes.

Examples include:

- npm: `npm ci`
- pnpm: `pnpm install --frozen-lockfile`
- Yarn: immutable installation mode
- Go: module verification and controlled module resolution
- Python: locked requirements or a lockfile-backed environment
- Container builds: pinned base-image digests and controlled package versions

The exact command MUST match the package manager and repository configuration.

Production builds MUST NOT silently update dependency versions during installation.

## 8. Vulnerability Scanning

### 8.1 Automated Scanning

All actively maintained repositories MUST have automated dependency vulnerability scanning.

Scanning SHOULD be integrated into:

- Pull requests.
- CI pipelines.
- Scheduled repository audits.
- Container image builds.
- Release workflows.
- Software Bill of Materials generation.
- Production image and artifact monitoring.

Scanners SHOULD detect:

- Known vulnerabilities in direct dependencies.
- Known vulnerabilities in transitive dependencies.
- Vulnerable container base images.
- Vulnerable operating system packages.
- Unsupported or end-of-life dependencies.
- Malicious or suspicious packages where tooling supports detection.
- License-policy violations where applicable.

### 8.2 Vulnerability Sources

Security findings SHOULD be correlated with authoritative advisory sources and ecosystem-specific databases, including:

- GitHub Security Advisories.
- OSV (Open Source Vulnerabilities).
- NVD (National Vulnerability Database).
- Official vendor security advisories.
- Language-specific package security advisories.
- Container image and operating system vendor advisories.

A scanner result is a signal requiring triage, not an automatic determination of exploitability.

### 8.3 Scan Coverage

Scanning MUST cover production and development dependency graphs.

Special attention MUST be given to:

- Authentication and authorization libraries.
- Cryptographic libraries.
- Database drivers and ORM components.
- API gateways and HTTP frameworks.
- File parsing and upload processing libraries.
- Serialization and deserialization packages.
- Image, document, archive, and media processing libraries.
- Search and caching clients.
- Build plugins and code-generation tools.
- Container base images and system packages.

## 9. Vulnerability Severity and Remediation

### 9.1 Risk Assessment

Vulnerabilities MUST be assessed using more than a severity score.

Consider:

- CVSS severity, where available.
- Whether a fix is available.
- Whether the vulnerable code is reachable.
- Whether the vulnerable feature is enabled.
- Whether authentication is required for exploitation.
- Whether exploitation is known or actively occurring.
- Exposure to the public internet.
- Tenant isolation and cross-tenant impact.
- Access to sensitive data or privileged operations.
- Potential impact on confidentiality, integrity, and availability.
- Compensating controls and deployment context.

A vulnerability with a lower severity score MAY require urgent action if it is actively exploited or exposes sensitive KAMPYN operations.

### 9.2 Remediation Targets

The following are default internal targets. A stricter contractual, regulatory, or incident-specific deadline takes precedence.

| Priority | Typical condition | Target |
|---|---|---|
| Critical | Active exploitation, remote code execution, or severe exposure of sensitive data | Immediate containment; remediation target within 24 hours |
| High | Significant exploitable vulnerability affecting exposed or security-critical components | Within 7 calendar days |
| Medium | Meaningful vulnerability with limited exploitability or exposure | Within 30 calendar days |
| Low | Limited security impact or difficult-to-exploit issue | Within 90 calendar days |

These are maximum target windows, not waiting periods. Remediation SHOULD happen sooner whenever practical.

If a vulnerability cannot be fixed within its target:

- Record the reason.
- Identify affected services and tenants.
- Document the risk and compensating controls.
- Assign an accountable owner.
- Define a remediation deadline.
- Obtain security-owner approval for the exception.
- Reassess the exception regularly.

Critical or actively exploited vulnerabilities MUST be escalated immediately, and affected functionality or deployments MUST be isolated, disabled, or otherwise mitigated when necessary.

### 9.3 Remediation Options

Preferred remediation order:

1. Upgrade to a vendor-supported patched version.
2. Upgrade to a compatible secure release.
3. Apply an official vendor patch.
4. Replace the dependency with an approved alternative.
5. Disable the vulnerable feature or remove the dependency.
6. Apply a documented temporary mitigation while preparing a permanent fix.

Forking a dependency to maintain a security patch MUST be treated as an exception with explicit ownership and an upstream contribution or replacement plan.

## 10. Dependency Updates

### 10.1 Routine Updates

Dependencies MUST be reviewed regularly for:

- Security patches.
- Supported releases.
- Bug fixes.
- End-of-life notices.
- Breaking changes.
- Compatibility with the supported runtime and deployment environments.

Automated update tooling MAY create pull requests, but updates MUST pass the same review, testing, and security requirements as other code changes.

### 10.2 Update Types

| Update type | Required handling |
|---|---|
| Patch | Automated tests and compatibility checks |
| Minor | Tests, changelog review, behavior compatibility review |
| Major | Migration review, breaking-change analysis, integration and regression tests |
| Security fix | Risk-based priority, focused tests, expedited review |
| Runtime or base-image update | Build, compatibility, security, and deployment validation |

Major version upgrades MUST NOT be merged solely because automated tests pass. Review migration notes, deprecated APIs, changed defaults, and security-related behavior.

### 10.3 Update Pull Requests

Dependency update pull requests MUST:

- Identify the dependency and version change.
- State the reason for the update.
- Include relevant release notes or advisory references.
- Show lockfile changes.
- Pass required automated checks.
- Be reviewed for unexpected dependency additions or removals.
- Be checked for changes to installation scripts, package sources, and permissions.

Updates that modify authentication, cryptography, authorization, payment processing, tenant isolation, or data handling MUST receive focused security review.

### 10.4 Update Cadence

As a baseline:

- Security alerts MUST be triaged promptly.
- Automated dependency update checks SHOULD run at least weekly.
- Full dependency audits SHOULD run at least monthly.
- End-of-life and unsupported dependencies MUST be tracked until replaced or explicitly approved for continued use.

A release freeze MUST NOT prevent remediation of a critical security vulnerability.

## 11. Ecosystem-Specific Requirements

### 11.1 JavaScript and TypeScript

For Next.js, React, Node.js services, SDKs, and tooling:

- Use the repository's selected package manager consistently.
- Commit the relevant lockfile.
- Use deterministic installation in CI.
- Review `postinstall`, `preinstall`, and other lifecycle scripts.
- Avoid unnecessary packages that duplicate native browser, Node.js, or framework capabilities.
- Review package changes affecting server/client boundaries.
- Keep frontend and backend dependencies separated where their runtime requirements differ.
- Prevent server-only dependencies and secrets from entering client bundles.
- Validate security-critical runtime inputs independently of TypeScript types.
- Audit both production and development dependency trees.

### 11.2 Go

For Go services and SDKs:

- Use Go modules and commit `go.mod` and `go.sum`.
- Verify module integrity using Go's supported mechanisms.
- Review module origin, maintainers, release history, and transitive dependencies.
- Avoid replacing modules with untrusted forks.
- Review `replace` directives and private module configuration.
- Use supported Go toolchains and update them regularly.
- Run vulnerability analysis using Go-compatible tooling.
- Review native dependencies and CGO usage where applicable.

### 11.3 Python

For Python utilities and workers:

- Use supported Python versions.
- Pin and lock production dependencies.
- Prefer isolated environments and reproducible builds.
- Review packages that execute setup or build code.
- Assess native extensions and binary wheels.
- Scan direct and transitive dependencies.
- Avoid installing packages dynamically from arbitrary URLs in production.
- Separate runtime dependencies from development and test dependencies.

### 11.4 Containers and Operating System Packages

Container images MUST:

- Use trusted and maintained base images.
- Prefer minimal images that contain only required runtime components.
- Pin production base images to immutable digests where practical.
- Scan operating system and application packages.
- Avoid unnecessary root privileges and build tools in runtime images.
- Keep build-time and runtime stages separate when feasible.
- Rebuild images when base-image security updates are released.
- Track image provenance and build inputs.
- Avoid embedding secrets or private registry credentials in image layers.

Production images MUST NOT contain unnecessary package managers, compilers, debug tools, or development dependencies without documented justification.

### 11.5 Infrastructure Dependencies

Kubernetes components, Helm charts, operators, CI actions, and infrastructure modules MUST be treated as software dependencies.

- Use trusted sources and pinned versions.
- Review permissions and requested capabilities.
- Pin third-party CI actions to immutable commit SHAs where supported.
- Review changes to deployment privileges and secret access.
- Scan container images and infrastructure artifacts.
- Track supported versions and end-of-life dates.
- Test upgrades in a representative environment before production rollout.

## 12. Security-Critical Dependencies

Dependencies involved in security-sensitive operations MUST receive enhanced controls.

This includes libraries for:

- Password hashing.
- Authentication and session management.
- Authorization and policy evaluation.
- Cryptographic operations.
- Token creation and validation.
- TLS and certificate handling.
- Secrets management.
- Payment and transaction processing.
- File upload and content parsing.
- Identity federation and OAuth/OIDC.
- Audit logging and security monitoring.

Requirements:

- Prefer established, actively maintained implementations.
- Do not create custom cryptographic primitives.
- Use secure, documented defaults.
- Review configuration changes during upgrades.
- Add focused tests for security-sensitive behavior.
- Review known vulnerabilities and supported versions.
- Ensure fallback behavior cannot weaken security.
- Document the dependency's role and security assumptions.

Security-critical dependency upgrades MUST be validated against relevant authentication, authorization, tenant-isolation, and data-protection tests.

## 13. Software Bill of Materials (SBOM)

KAMPYN production releases SHOULD generate an SBOM for each deployable artifact.

The SBOM SHOULD:

- Identify direct and transitive dependencies.
- Include package names, versions, and ecosystems.
- Identify licenses where available.
- Include package identifiers and provenance metadata where supported.
- Be associated with the build artifact or release.
- Be retained for the supported lifetime of the release.
- Be available to authorized university self-hosting customers where contractually appropriate.

Use a recognized machine-readable format such as SPDX or CycloneDX.

SBOM generation MUST NOT expose secrets, private source code, internal credentials, or sensitive infrastructure details.

SBOMs SHOULD be used to identify affected releases when new vulnerabilities are disclosed.

## 14. Supply-Chain Integrity

### 14.1 Build Isolation

Build environments MUST be isolated from production systems and sensitive runtime data.

- CI jobs MUST receive only the permissions required for their tasks.
- Pull-request builds from untrusted contributions MUST NOT receive production secrets.
- Dependency installation and build scripts MUST run with restricted privileges.
- Release signing and publishing credentials MUST be isolated from ordinary test jobs.
- Build artifacts MUST be promoted through controlled release stages rather than rebuilt inconsistently for each environment.

### 14.2 Artifact Provenance

Production releases SHOULD maintain verifiable records of:

- Source revision.
- Build workflow and toolchain.
- Dependency lockfiles.
- Container base images.
- Build artifact identifiers.
- SBOM.
- Release approvals.
- Signing and verification metadata, where supported.

Where available, adopt signed artifacts and provenance attestations to strengthen supply-chain traceability.

### 14.3 Registry and Publishing Security

Private package registries and image registries MUST:

- Enforce authenticated access.
- Restrict publishing and deletion permissions.
- Protect release credentials.
- Use TLS.
- Maintain audit logs.
- Support controlled retention and cleanup.
- Prevent unauthorized overwrites of released artifacts where supported.

Published KAMPYN packages and images MUST use controlled release processes. Secrets and sensitive configuration MUST never be published as part of packages or container layers.

## 15. Dependency Security for Multi-Tenancy

Dependency vulnerabilities can affect all tenants sharing the same deployment.

KAMPYN MUST consider tenant-wide impact when evaluating dependency risk.

- Shared runtime dependencies MUST be assessed for cross-tenant exposure.
- Tenant-specific integrations MUST be isolated from unrelated tenant data and credentials.
- Dependency update and rollback processes MUST account for all affected tenants.
- Self-hosted installations MUST receive actionable security advisories for relevant vulnerabilities.
- Tenant configuration MUST NOT allow an insecure dependency or unsupported component to bypass platform security requirements.
- Security fixes MUST be applied consistently across supported deployment variants.

Dependencies used in search, caching, background jobs, storage, messaging, and realtime communication MUST preserve tenant boundaries during both normal operation and failure conditions.

## 16. Open-Source License Compliance

Every dependency MUST have a known license or a documented legal review before production use.

- Maintain a record of direct and transitive dependency licenses.
- Identify obligations related to attribution, notices, source distribution, and modifications.
- Review copyleft and network-copyleft implications against KAMPYN's SaaS and self-hosted distribution models.
- Include required license notices in distributed artifacts.
- Do not assume that a publicly available package is unrestricted for commercial use.
- Escalate unclear, custom, or incompatible licenses for legal review.

Dependencies with unknown, conflicting, or prohibited licenses MUST NOT be introduced into production without an approved exception.

## 17. Unsupported and Abandoned Dependencies

Dependencies MUST be monitored for maintenance and support status.

Warning signs include:

- No meaningful maintenance over an extended period.
- Unresolved security issues.
- End-of-life notices.
- Incompatibility with supported runtimes.
- Unresponsive maintainers.
- Archived or read-only repositories.
- Unexplained ownership changes.
- Untrusted or suspicious release activity.

An unsupported dependency MUST be assessed for replacement, internal maintenance, or removal.

Security-critical dependencies MUST NOT remain unsupported without an explicit, time-limited risk exception and a documented migration plan.

## 18. Dependency Exceptions

Exceptions MUST be rare, justified, and time-bound.

Each exception MUST include:

- Dependency name and version.
- Affected applications and services.
- Known vulnerability or policy requirement.
- Reason remediation is currently infeasible.
- Risk assessment and potential tenant impact.
- Compensating controls.
- Responsible owner.
- Approval by the designated security owner.
- Expiry date and remediation plan.

Exceptions MUST be reviewed before expiry and closed once the dependency is upgraded, replaced, or removed.

An exception MUST NOT be used to silently suppress critical or actively exploited vulnerabilities.

## 19. Incident Response

If a dependency is suspected to be compromised or affected by a severe vulnerability:

1. Identify all repositories, builds, services, and releases that use the dependency.
2. Determine affected versions, transitive paths, and runtime exposure.
3. Assess whether the vulnerable code was reachable or executed.
4. Check available vendor advisories and trusted threat intelligence.
5. Contain affected builds, deployments, credentials, or functionality as necessary.
6. Upgrade, replace, isolate, or remove the affected dependency.
7. Rebuild affected artifacts from trusted source and verified dependencies.
8. Review logs, build records, registry access, and deployment history for evidence of compromise.
9. Rotate credentials if exposure is plausible or confirmed.
10. Notify affected internal stakeholders and customers where required.
11. Record the incident, remediation, and lessons learned.
12. Update detection, approval, and prevention controls.

For suspected malicious package releases, stop using the affected versions and avoid relying on the same potentially compromised distribution source until it is assessed.

Incident handling MUST follow the broader KAMPYN incident response and vulnerability management policies.

## 20. Monitoring and Reporting

Dependency security MUST be observable and auditable.

Track at minimum:

- Number of known vulnerabilities by severity.
- Time from advisory detection to triage.
- Time to remediation by severity.
- Vulnerability backlog age.
- Percentage of repositories with automated scanning.
- Percentage of production artifacts with an SBOM.
- Number of unsupported dependencies.
- Number of active exceptions and their expiry dates.
- Dependency update failure rates.
- Supply-chain policy violations.
- Critical dependency exposure across deployed versions.

Security findings MUST have an owner, status, priority, affected components, and remediation or exception record.

Reports SHOULD distinguish confirmed exploitable vulnerabilities from scanner findings that require further investigation.

## 21. Testing Requirements

Dependency changes MUST pass the relevant quality and security checks.

Required validation SHOULD include:

- Dependency installation from a clean environment.
- Lockfile consistency checks.
- Unit and integration tests.
- Build and type checks.
- Static analysis and vulnerability scans.
- License checks.
- Container image scans, when applicable.
- API contract and SDK compatibility tests.
- Authentication and authorization regression tests for security-sensitive updates.
- Tenant-isolation tests where data access or request handling may be affected.
- Performance and load checks for changes to high-traffic components.

Security-sensitive dependency upgrades MUST include tests for relevant failure modes, including invalid input, malformed data, rejected credentials, expired tokens, and denied access.

Tests MUST validate the behavior of the integrated application, not only the dependency in isolation.

## 22. CI/CD Enforcement

CI/CD MUST enforce dependency security as part of the standard delivery process.

At minimum:

- Validate lockfile consistency.
- Run automated vulnerability scans.
- Scan production container images.
- Detect prohibited licenses where configured.
- Block release for unapproved critical vulnerabilities according to the vulnerability policy.
- Require review for changes to dependency manifests, lockfiles, registries, and build configuration.
- Prevent untrusted pull requests from accessing privileged secrets.
- Retain security scan results and artifact metadata.

CI checks MUST fail clearly when a mandatory security requirement is violated.

Temporary exceptions MUST be explicit, approved, traceable, and time-limited. Suppression rules MUST include a reason and an expiry or review date.

## 23. KAMPYN-Specific Dependency Controls

### 23.1 Food Ordering and Payments

Dependencies handling order placement, pricing, payment callbacks, refunds, or reconciliation MUST receive security-focused review.

- Validate payment-provider signatures using supported mechanisms.
- Ensure idempotency and transaction integrity are enforced by KAMPYN services.
- Review dependency changes that affect decimal handling, serialization, signatures, or callback verification.
- Never rely on frontend dependency behavior as the source of payment authorization.

### 23.2 Bookings and Resource Scheduling

Dependencies supporting hostel, guest-house, shuttle, washing-machine, or facility bookings MUST preserve concurrency and authorization guarantees.

Updates MUST be checked for changes to date handling, time zones, locking, validation, and transaction behavior.

### 23.3 Community and Realtime Communication

Dependencies supporting chat, notifications, WebSockets, or realtime events MUST be assessed for:

- Authentication and connection authorization.
- Tenant and room isolation.
- Input validation and message handling.
- Rate limiting and resource exhaustion.
- File and attachment processing.
- Reconnection and session lifecycle behavior.

### 23.4 Search and Analytics

OpenSearch clients, analytics libraries, data processors, and exporters MUST be reviewed for access-control implications, query safety, sensitive-data exposure, and resource consumption.

Search indexes and analytics outputs MUST NOT become an unintended source of cross-tenant data exposure.

### 23.5 Self-Hosted Deployments and SDKs

Dependencies distributed through KAMPYN SDKs, Docker images, Helm charts, or self-hosted packages MUST have clear supported-version policies.

- Document runtime requirements and compatible versions.
- Publish security advisories for supported releases.
- Provide upgrade guidance for security fixes.
- Avoid exposing internal service dependencies unnecessarily through public SDKs.
- Keep supported deployment artifacts reproducible and traceable.
- Define the support lifecycle for externally distributed components.

## 24. Prohibited Practices

The following practices are prohibited:

- Installing dependencies from unverified sources.
- Committing credentials or secrets in package manifests, lockfiles, build scripts, or artifacts.
- Ignoring critical or actively exploited vulnerabilities without containment and an approved exception.
- Disabling vulnerability scanning to make a pipeline pass.
- Suppressing security findings without a documented reason and review date.
- Using abandoned security-critical dependencies without an approved risk exception.
- Using unpinned production artifacts where this prevents reliable traceability.
- Allowing arbitrary package scripts to access production secrets.
- Publishing internal packages to public registries unintentionally.
- Using unreviewed forks for security-critical functionality.
- Including unnecessary development tooling in production images.
- Treating frontend package checks as a substitute for backend security enforcement.
- Updating dependencies directly in production without controlled validation.
- Assuming that a package is safe merely because it is widely adopted.

## 25. Review Checklist

Before merging a dependency change, reviewers MUST verify:

- [ ] The dependency has a clear and necessary purpose.
- [ ] Existing approved dependencies or platform capabilities were considered.
- [ ] The package source and identity are verified.
- [ ] The license is compatible with KAMPYN's distribution model.
- [ ] Maintenance and support status have been assessed.
- [ ] Direct and transitive dependency changes have been reviewed.
- [ ] Installation scripts and permissions have been assessed where relevant.
- [ ] Lockfiles are updated and reproducible.
- [ ] Vulnerability and license scans have passed or have approved exceptions.
- [ ] Tests and builds pass.
- [ ] Security-sensitive behavior has received focused review.
- [ ] Runtime and container artifacts remain minimal and traceable.
- [ ] Documentation and ownership records are updated where necessary.
- [ ] The dependency has a clear update and removal strategy.

## 26. Definition of Done

A dependency change is complete only when:

- The dependency is justified, approved, and correctly classified.
- Its source, version, license, and maintenance status have been assessed.
- Direct and transitive risks have been reviewed.
- Lockfiles and build configuration are reproducible.
- Required vulnerability and license checks have passed.
- Relevant application, security, and compatibility tests pass.
- Security findings have been remediated or formally excepted.
- Ownership and lifecycle responsibilities are clear.
- Required SBOM and provenance information is generated or updated.
- Documentation and deployment guidance are updated where applicable.
- No secrets or unintended privileges have been introduced.
- The change complies with KAMPYN's architecture, security, and quality standards.

## 27. Final Rule

Every dependency is part of KAMPYN's software supply chain and MUST be treated as a potential security boundary.

A dependency MUST be necessary, trusted, maintained, traceable, and continuously monitored. When its security can no longer be established, it MUST be upgraded, replaced, isolated, or removed.
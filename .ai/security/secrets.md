# Secrets Management & Protection

## 1. Purpose

This document defines how KAMPYN handles, stores, accesses, rotates, distributes, and audits secrets across development, testing, staging, production, self-hosted deployments, CI/CD pipelines, and third-party integrations.

Secrets are sensitive values that grant access to systems, data, infrastructure, or privileged operations. Improper handling can lead to account compromise, data breaches, financial loss, service disruption, and cross-tenant exposure.

This policy applies to all KAMPYN applications, services, infrastructure, developers, operators, automated agents, SDKs, deployment pipelines, and university self-hosted installations.

## 2. Core Principles

All secrets management must follow these principles:

- **Never hardcode secrets:** Secrets must not be embedded in source code, configuration files committed to version control, container images, frontend bundles, or documentation.
- **Least privilege:** Every service, user, and workload must have access only to the secrets it requires.
- **Centralized management:** Production secrets must be managed through an approved secrets manager or secure deployment platform.
- **Environment isolation:** Development, testing, staging, and production secrets must be separate.
- **Short-lived credentials:** Prefer temporary, automatically issued credentials over long-lived static credentials.
- **Secure by default:** Missing, invalid, or inaccessible secrets must cause secure startup failure for services that require them.
- **Rotation:** Secrets must support controlled rotation and revocation without unnecessary service disruption.
- **Auditable access:** Secret access and administrative operations must be traceable.
- **No unnecessary exposure:** Secrets must never be logged, returned through APIs, exposed in error messages, or unnecessarily copied between systems.
- **Tenant isolation:** Tenant-specific credentials must be scoped and protected against cross-tenant access.
- **Defense in depth:** Secret managers, identity controls, encryption, network restrictions, and monitoring must work together.

## 3. Secret Classification

Secrets must be classified according to their impact if exposed.

| Classification | Examples | Protection |
|---|---|---|
| Critical | Root credentials, production signing keys, database superuser credentials, payment provider master credentials | Strictly controlled access, strong audit, managed storage, rapid rotation |
| High | Production database credentials, JWT signing keys, OAuth client secrets, encryption keys | Secrets manager, least privilege, rotation, restricted access |
| Medium | Third-party API keys with limited scope, service credentials, webhook secrets | Managed storage, scoped access, rotation |
| Low | Local development credentials for isolated environments | Local secure storage, never committed |

Classification must consider the privileges a secret grants, the sensitivity of the accessible data, and the impact of misuse.

A secret must be handled according to the highest classification applicable to its use.

## 4. Types of Secrets

KAMPYN may use the following categories of secrets:

### 4.1 Application Secrets

- JWT signing and verification keys
- Session signing keys
- Password reset and verification token secrets
- Internal API credentials
- Service-to-service authentication credentials
- Encryption keys
- Cryptographic salts where applicable

### 4.2 Database Secrets

- PostgreSQL credentials
- MongoDB credentials
- Redis authentication credentials
- OpenSearch credentials
- Database migration credentials
- Backup and restoration credentials

### 4.3 Infrastructure Secrets

- Cloud provider credentials
- Kubernetes service account credentials
- Container registry credentials
- Infrastructure provisioning credentials
- TLS private keys
- Deployment automation credentials
- Monitoring and observability credentials

### 4.4 Third-Party Integration Secrets

- Payment gateway credentials
- Email delivery credentials
- SMS provider credentials
- OAuth client secrets
- Cloud storage credentials
- Webhook signing secrets
- External university integration credentials

### 4.5 Tenant-Specific Secrets

- University-specific integration credentials
- Tenant-specific webhook secrets
- Tenant-managed service credentials
- Tenant-specific encryption keys, where supported

Tenant-specific secrets must be isolated so that access granted to one tenant or integration cannot expose another tenant's credentials.

## 5. Approved Secret Storage

### 5.1 Production

Production secrets must be stored in a dedicated secrets management solution, such as:

- Google Cloud Secret Manager
- HashiCorp Vault
- Kubernetes Secrets backed by an approved external secrets manager
- A cloud provider's equivalent managed secrets service

The chosen solution must support access control, encryption at rest, access auditing, versioning or controlled rotation, and secure integration with workloads.

Production secrets must not be stored in plain-text files, source repositories, container images, or unencrypted deployment manifests.

### 5.2 Development

Developers must use local environment variables, ignored environment files, or an approved local secrets manager.

Example:

```text
.env.local
```

Local secret files must be excluded from version control using `.gitignore`.

Example:

```gitignore
.env
.env.*
!.env.example
```

Local secrets must not be copied from production unless there is a formally approved, narrowly scoped need and a secure procedure for doing so. Prefer synthetic or sanitized development data.

### 5.3 CI/CD

CI/CD systems must retrieve secrets from an approved secret store or use short-lived identity-based credentials.

Secrets must not be:

- Hardcoded in workflow definitions
- Printed in pipeline logs
- Exposed as build arguments in container builds
- Stored in public or broadly accessible pipeline artifacts
- Shared between unrelated repositories or environments without justification

Where supported, use workload identity federation or equivalent mechanisms instead of long-lived cloud access keys.

### 5.4 Self-Hosted University Deployments

Universities self-hosting KAMPYN are responsible for protecting their deployment secrets in accordance with this policy and the deployment security guide.

KAMPYN must provide secure configuration mechanisms, documented secret requirements, and clear rotation procedures.

Self-hosted deployments must not require universities to embed secrets in frontend configuration, source code, or publicly accessible deployment files.

Where a university operates its own secret manager, KAMPYN services should integrate with it through documented mechanisms.

## 6. Environment Variable Management

Environment variables may be used to provide secrets to application processes at runtime, but they are not themselves a secret management system.

### 6.1 Naming Conventions

Environment variables must use consistent, descriptive names.

Examples:

```env
DATABASE_URL=
MONGODB_URI=
REDIS_URL=
JWT_SIGNING_KEY=
PAYMENT_PROVIDER_SECRET=
WEBHOOK_SIGNING_SECRET=
```

Names must clearly identify the secret's purpose without revealing the secret value.

### 6.2 Configuration Separation

Separate configuration from secret values.

Non-sensitive configuration may include:

```env
APP_ENV=production
LOG_LEVEL=info
API_PORT=8080
```

Sensitive configuration may include:

```env
DATABASE_URL=<injected-at-runtime>
JWT_SIGNING_KEY=<injected-at-runtime>
```

Actual secret values must never be included in examples, documentation, test fixtures, or committed configuration files.

### 6.3 Validation

Applications must validate required secrets during startup.

Validation must check:

- Presence of required secrets
- Correct format and encoding
- Minimum cryptographic strength where applicable
- Compatibility with the configured environment
- Expiration or validity where supported

Applications must fail startup when a required secret is missing, invalid, or insecure.

Error messages must identify the missing configuration key without disclosing its value.

### 6.4 Runtime Access

Secrets must be loaded only by components that need them.

Avoid passing secrets through unrelated services, global state, broad configuration objects, or client-facing data structures.

Secrets must not be included in application responses or frontend hydration payloads.

## 7. Frontend Secret Protection

Frontend applications run in an environment controlled by the user and must be treated as publicly inspectable.

The following must never be exposed to the frontend:

- Database credentials
- JWT signing keys
- Internal service credentials
- Payment provider secret keys
- Cloud provider credentials
- Encryption keys
- Private API tokens
- Webhook secrets
- Privileged administrative credentials

Only explicitly designed public identifiers or public keys may be included in frontend bundles.

For Next.js:

- Only intentionally public configuration may use `NEXT_PUBLIC_` variables.
- Never prefix a secret with `NEXT_PUBLIC_`.
- Server-side secrets must remain within server-only modules.
- Do not return secrets through Server Components, Server Actions, Route Handlers, or API responses.
- Avoid importing server-only configuration into client components.

Frontend access controls are not a substitute for server-side authorization.

## 8. Secret Access Control

### 8.1 Identity-Based Access

Secret access must be granted to authenticated identities, including:

- Individual developers
- Production operators
- CI/CD workloads
- Backend services
- Infrastructure automation
- Approved support personnel

Each identity must have only the permissions required for its specific responsibilities.

### 8.2 Role Separation

Secret access must be separated by operational responsibility.

| Role | Permitted Access |
|---|---|
| Developer | Approved development secrets |
| CI/CD workload | Secrets required for its specific build or deployment |
| Service identity | Secrets required by that service |
| Production operator | Approved operational secrets |
| Security administrator | Controlled secret administration and audit |
| Tenant administrator | Only tenant-scoped secrets explicitly delegated to them |

A role must not automatically receive access to every secret in its environment.

### 8.3 Service Identity

Each service must use a dedicated identity where practical.

For example:

- Ordering service
- Booking service
- Notification service
- Payment integration service
- Search indexing worker
- Analytics worker

A service must not use shared administrative credentials when a narrower identity can be created.

### 8.4 Privileged Access

Privileged access must require stronger authentication and additional controls, such as:

- Multi-factor authentication
- Just-in-time access
- Time-limited permissions
- Approval for highly sensitive operations
- Audited administrative sessions

Emergency or break-glass access must be restricted, logged, reviewed, and revoked after use.

### 8.5 Access Reviews

Secret access permissions must be reviewed periodically and whenever:

- An employee changes roles
- A contractor's engagement ends
- A service is decommissioned
- A deployment identity is replaced
- A security incident occurs
- A tenant integration is removed

Unused or excessive permissions must be removed promptly.

## 9. Secret Generation

Secrets must be generated using cryptographically secure random number generators.

Do not use:

- Predictable timestamps
- Usernames or email addresses
- Sequential identifiers
- General-purpose random functions that are not cryptographically secure
- Hardcoded default credentials
- Reused passwords or keys

Cryptographic keys must use approved algorithms and adequate key lengths for their intended purpose.

Secrets must be generated using trusted tools, libraries, or managed secret services.

Generated secrets must be transferred directly to approved storage or the intended secure provisioning process. Avoid copying them through chat, email, issue trackers, or unsecured notes.

## 10. Cryptographic Key Management

Cryptographic keys require stronger controls than ordinary configuration values.

### 10.1 Key Separation

Different cryptographic purposes must use different keys.

For example:

- JWT signing keys
- Data encryption keys
- Webhook verification secrets
- Session signing keys
- Password-reset token signing keys

A key must not be reused across unrelated cryptographic operations.

### 10.2 Key Storage

Keys must be stored in an approved secret manager or key management service.

Where possible, use a managed Key Management Service (KMS) for encryption keys and cryptographic operations.

Private keys must not be embedded in source code, container images, or client-side applications.

### 10.3 Encryption at Rest

Sensitive data encryption must use managed keys or securely managed application keys.

Where envelope encryption is used:

- Data encryption keys (DEKs) encrypt application data.
- Key encryption keys (KEKs) protect DEKs.
- KEKs must be stored and managed separately from encrypted data.
- Key rotation must account for existing encrypted records.

### 10.4 Key Rotation

Key rotation must be supported without unnecessarily invalidating all active sessions or making encrypted data unrecoverable.

For signing keys, use a controlled transition period where old keys may verify existing tokens while new tokens are signed with the current key, where the protocol and risk model permit it.

For encryption keys, retain access to prior key versions for decryption until data has been safely re-encrypted or the retention requirement expires.

Key retirement must be deliberate, documented, and tested.

## 11. Secret Rotation and Revocation

Secrets must have documented rotation and revocation procedures.

Rotation frequency must be based on:

- Secret classification
- Exposure risk
- Provider capabilities
- Credential lifetime
- Regulatory or contractual requirements
- Operational impact

Short-lived credentials should be preferred wherever supported.

### 11.1 Rotation Requirements

Rotation procedures must:

1. Generate or request a new secret securely.
2. Store the new secret in the approved secret manager.
3. Update the relevant workload or integration.
4. Verify that the new credential works.
5. Revoke the old credential when safe.
6. Confirm that dependent services continue operating.
7. Record the rotation event without recording the secret value.

### 11.2 Zero-Downtime Rotation

Where supported, rotation should use overlapping validity periods.

For example:

1. Issue a new credential.
2. Allow both credentials during a controlled transition.
3. Update workloads to use the new credential.
4. Verify successful operation.
5. Revoke the old credential.

If overlapping credentials are not supported, use a coordinated deployment or maintenance procedure with an explicit rollback plan.

### 11.3 Immediate Revocation

Secrets must be revoked immediately when:

- Exposure is confirmed or reasonably suspected
- A credential is accidentally committed
- An unauthorized party may have accessed a secret
- A service identity is compromised
- An employee or contractor with privileged access leaves unexpectedly
- A third-party integration is terminated

Revocation must be accompanied by investigation, impact assessment, and replacement of affected credentials.

## 12. Secret Handling in Source Control

Secrets must never be committed to Git repositories.

This includes:

- `.env` files containing actual secrets
- Private keys and certificates
- API tokens
- Database connection strings
- Access credentials
- Secret-bearing test fixtures
- Configuration backups containing credentials

### 12.1 Safe Examples

Provide a sanitized `.env.example` file:

```env
APP_ENV=development
API_PORT=8080

DATABASE_URL=
REDIS_URL=
JWT_SIGNING_KEY=
PAYMENT_PROVIDER_SECRET=
```

Examples must use placeholders rather than realistic-looking credentials that could accidentally be accepted by a production service.

### 12.2 Secret Scanning

Repositories must use automated secret scanning in:

- Local development workflows where available
- Pull requests
- CI pipelines
- Main branches
- Periodic repository scans

Secret scanning must include current files and, where supported, repository history.

### 12.3 Pre-Commit Protection

Developers should use pre-commit secret scanning to detect accidental exposure before code is committed.

CI must independently scan changes because local hooks can be bypassed.

### 12.4 Accidental Exposure

If a secret is committed:

1. Treat it as compromised.
2. Revoke or rotate it immediately.
3. Identify the affected systems and data.
4. Review repository history and access logs.
5. Remove the exposed value from active source and, where feasible, repository history.
6. Assess forks, clones, caches, build artifacts, and logs.
7. Document the incident and corrective actions.

Removing a secret from the latest commit does not make the credential safe again. Rotation is mandatory.

## 13. Secret Handling in CI/CD and Build Systems

CI/CD pipelines must minimize secret exposure during builds and deployments.

Requirements:

- Use short-lived workload identities where supported.
- Scope secrets to the specific repository, workflow, environment, or deployment.
- Restrict access to protected branches and production deployment environments.
- Require review or approval for sensitive production deployments.
- Avoid exposing production secrets to untrusted pull requests or forked repositories.
- Prevent secrets from appearing in command output, debug logs, or test reports.
- Do not place secrets in Docker build arguments or image layers.
- Use secure runtime injection for containerized workloads.
- Restrict access to build artifacts and deployment logs.
- Revoke credentials for retired or compromised pipelines.

Production deployment credentials must be separate from routine development and testing credentials.

## 14. Containers and Kubernetes

### 14.1 Container Images

Container images must not contain secrets in:

- Image layers
- Build arguments
- Environment defaults
- Copied configuration files
- Entrypoint scripts
- Embedded certificates or private keys

Images must be built without requiring production credentials.

### 14.2 Kubernetes

Kubernetes workloads must retrieve secrets using an approved secrets integration or a properly secured Kubernetes Secret mechanism.

Requirements:

- Enable encryption at rest for Kubernetes Secrets.
- Apply strict RBAC policies.
- Restrict secret access by namespace and service account.
- Avoid mounting secrets into containers that do not require them.
- Prefer read-only secret mounts where practical.
- Prevent secrets from being exposed through pod specifications, logs, or debugging endpoints.
- Restrict access to Kubernetes API and control-plane credentials.
- Rotate service account credentials and application secrets according to policy.

Base64 encoding is not encryption. Kubernetes Secret objects must be treated as sensitive data.

### 14.3 Runtime Injection

Secrets should be injected at runtime using secure integrations rather than baked into images or manifests.

Workloads must support secret updates through a documented reload, restart, or redeployment process.

## 15. Database Credentials

Database access must use dedicated identities with the minimum required privileges.

Requirements:

- Separate application credentials from migration and administrative credentials.
- Avoid using database superuser accounts in application services.
- Scope credentials to the required database, schema, collection, or operation.
- Use TLS for database connections where supported.
- Restrict network access to trusted workloads.
- Rotate credentials and remove obsolete accounts.
- Never expose connection strings through client APIs or logs.
- Use separate credentials for development, staging, and production.

Where supported, prefer IAM-based or short-lived database authentication over static passwords.

Database migration credentials must not be available to normal application request handlers.

## 16. Third-Party Integration Secrets

Third-party integration credentials must be isolated by provider and purpose.

For each integration, document:

- Provider and integration purpose
- Owning service or team
- Secret classification
- Permissions granted
- Storage location
- Rotation procedure
- Expiration date, where applicable
- Revocation process
- Recovery and fallback procedure

Examples include payment processing, email delivery, SMS delivery, cloud storage, identity providers, and university systems.

Credentials must be scoped to the minimum permissions and environments required.

Production and sandbox credentials must never be interchanged.

Webhook signing secrets must be distinct from API credentials and must be validated according to the integration's verification protocol.

## 17. Tenant-Specific Secret Isolation

KAMPYN is a multi-tenant platform. Tenant-specific credentials must be protected from cross-tenant access.

Requirements:

- Associate each tenant secret with an immutable tenant identifier.
- Enforce tenant authorization before retrieving or using tenant-specific credentials.
- Avoid relying solely on client-provided tenant identifiers.
- Restrict tenant secrets to the integration or service that needs them.
- Ensure cache keys and secret references include the correct tenant scope.
- Prevent tenant secrets from appearing in shared logs, analytics, or error messages.
- Audit administrative access to tenant-specific secrets.
- Revoke tenant credentials when integrations are disabled or tenants are deprovisioned.

Tenant administrators must not be able to retrieve platform-wide secrets or credentials belonging to another tenant.

Where tenant-managed keys are supported, the ownership, recovery, rotation, and deletion responsibilities must be documented clearly.

## 18. Logging and Monitoring

Secret values must never be written to logs.

This includes application logs, access logs, tracing spans, metrics labels, crash reports, analytics events, and audit messages.

### 18.1 Redaction

Sensitive values must be redacted before logging.

Redaction must cover:

- Authorization headers
- Cookies and session tokens
- API keys
- Database connection strings
- Passwords
- Private keys
- Webhook signatures
- Secret-bearing request fields
- Sensitive environment configuration

Redaction must be implemented centrally where practical and reinforced at individual logging boundaries.

### 18.2 Access Auditing

Secret management systems should record:

- Identity accessing the secret
- Secret identifier
- Access timestamp
- Access result
- Operation performed
- Relevant environment or workload context

Audit logs must not contain secret values.

### 18.3 Alerting

Alert on suspicious activities, including:

- Unusual secret retrieval volume
- Access from unexpected identities or environments
- Repeated access failures
- Unauthorized secret version changes
- Unexpected privilege escalation
- Secret access outside normal deployment windows
- Access to revoked or deprecated credentials
- Attempts to access secrets outside an authorized tenant scope

Alerts must be actionable and routed to the responsible operational or security team.

## 19. Error Handling

Applications must not expose secrets in error responses.

Prohibited examples include:

- Returning complete database connection strings
- Returning provider error messages containing credentials
- Printing environment configuration during startup failures
- Returning raw authentication headers
- Including tokens in stack traces exposed to clients

Errors should identify the failed operation and provide a safe correlation identifier where useful.

Detailed diagnostic information must be available only through appropriately restricted internal observability systems, with secrets redacted.

## 20. Backups and Disaster Recovery

Secret stores and cryptographic keys are critical recovery dependencies.

Requirements:

- Define backup and recovery procedures for secret metadata and key material, where supported.
- Protect backups using encryption and strict access control.
- Ensure recovery procedures do not expose secrets to unauthorized operators.
- Test recovery processes periodically.
- Maintain appropriate key version history for encrypted data recovery.
- Document dependencies between restored services, secret stores, and encryption systems.
- Define recovery responsibilities for SaaS and self-hosted deployments.

A database backup is not sufficient for disaster recovery if the keys needed to decrypt its contents cannot be recovered.

Key loss scenarios must be included in disaster recovery planning.

## 21. Secret Lifecycle Management

Every production secret must have an identifiable lifecycle.

### 21.1 Provisioning

- Define the secret's purpose and owner.
- Classify its sensitivity.
- Generate or obtain it securely.
- Store it in an approved secret manager.
- Assign narrowly scoped access.
- Record metadata without storing the secret value in documentation.

### 21.2 Active Use

- Monitor access.
- Review permissions.
- Rotate according to risk and policy.
- Track expiry where applicable.
- Maintain an up-to-date owner and dependency record.

### 21.3 Decommissioning

When a service or integration is retired:

- Revoke related credentials.
- Remove secret access policies.
- Delete unused secret versions when retention requirements permit.
- Remove references from deployment configurations.
- Confirm no active workload depends on the retired credential.
- Retain only necessary audit metadata.

## 22. Incident Response

Suspected or confirmed secret exposure must be treated as a security incident.

### 22.1 Response Procedure

1. **Identify:** Determine which secret may have been exposed, where it appeared, and its classification.
2. **Contain:** Restrict access, disable affected identities, or revoke the credential.
3. **Rotate:** Issue replacement credentials and update dependent services.
4. **Investigate:** Review secret access logs, system logs, deployments, and potential unauthorized activity.
5. **Assess impact:** Identify potentially affected tenants, services, data, and integrations.
6. **Recover:** Restore normal operation using verified credentials and configurations.
7. **Notify:** Follow applicable contractual, legal, and internal incident notification requirements.
8. **Remediate:** Address the root cause, improve controls, and add tests or detection mechanisms.
9. **Review:** Document the incident, timeline, decisions, and lessons learned.

### 22.2 Incident Priorities

Critical and high-impact secret exposure must receive immediate attention.

Examples include:

- Production signing key exposure
- Production database credential compromise
- Cloud administrator credential compromise
- Payment credential exposure
- Cross-tenant secret disclosure
- Exposure of encryption keys protecting sensitive data

Incident priority must be based on the privileges granted, the data accessible, the exposure scope, and evidence of misuse.

### 22.3 Communication

Incident communication must avoid reproducing the exposed secret.

Share only the information necessary for response and ensure sensitive incident records are access-controlled.

## 23. Application-Specific Requirements

### 23.1 Authentication

- Keep token signing keys server-side.
- Use distinct secrets for different token purposes.
- Rotate signing keys using a controlled transition procedure.
- Avoid storing raw passwords or reusable authentication secrets.
- Revoke compromised credentials and invalidate affected sessions as appropriate.

### 23.2 Payments

- Keep provider secret keys on trusted backend services.
- Use separate credentials for sandbox and production.
- Restrict access to payment credentials to the services that require them.
- Verify webhook signatures using the correct integration secret.
- Rotate compromised payment credentials immediately.
- Do not log payment authorization headers or secret-bearing provider responses.

### 23.3 Notifications

- Restrict email and SMS credentials to the notification services that need them.
- Use provider-level permissions and rate limits where supported.
- Keep provider credentials separate from user notification preferences.
- Prevent secrets from appearing in delivery errors or message metadata.

### 23.4 Community and Realtime Services

- Keep signing and service credentials server-side.
- Use short-lived connection credentials where practical.
- Avoid sharing internal service credentials with browser clients.
- Restrict service identities that publish or consume internal events.
- Ensure secret access does not bypass channel or tenant authorization.

### 23.5 Search and Analytics

- Restrict OpenSearch and analytics credentials to required operations.
- Separate indexing, querying, and administrative identities where practical.
- Do not expose internal search credentials to browser clients.
- Prevent secret values from entering indexed documents, search logs, or analytics events.

### 23.6 Background Workers

- Give each worker access only to the secrets required for its workload.
- Ensure retries and dead-letter queues do not contain secret values.
- Avoid passing secrets in job payloads.
- Retrieve credentials securely at runtime.
- Revoke credentials for retired workers.

## 24. SDK and API Requirements

KAMPYN SDKs must never require developers to embed privileged platform secrets in client-side applications.

Requirements:

- Distinguish public configuration from confidential credentials.
- Provide secure server-side initialization for privileged SDK capabilities.
- Document safe handling of access tokens.
- Avoid logging credentials in SDK debug mode.
- Support credential replacement without requiring unsafe code changes.
- Ensure SDK errors do not reveal secrets.
- Document tenant-scoped credential configuration for self-hosted and enterprise integrations.

Public API keys must have explicitly limited capabilities and must not be treated as confidential credentials.

## 25. Self-Hosted Secret Configuration

Self-hosted installations must have a documented secret provisioning process.

Requirements:

- Provide a complete list of required secret names and purposes.
- Provide sanitized configuration templates.
- Clearly identify secrets that must be generated uniquely per installation.
- Document supported secret managers and secure runtime injection.
- Provide rotation and recovery procedures.
- Separate initial bootstrap credentials from ongoing runtime credentials.
- Avoid shipping default production passwords or shared private keys.
- Require operators to replace any temporary bootstrap credentials before production use.

Installation tooling must not print secrets to terminal output, logs, or generated public configuration.

## 26. Testing Requirements

Secret management controls must be tested as part of application and infrastructure validation.

### 26.1 Automated Tests

Tests should verify:

- Application startup fails when required secrets are missing.
- Invalid secret formats are rejected.
- Secrets are not returned in API responses.
- Secrets are redacted from logs and errors.
- Unauthorized identities cannot access secret values.
- Tenant-scoped secret access is enforced.
- Secret rotation works as designed.
- Revoked credentials are rejected.
- CI pipelines prevent unauthorized secret exposure.
- Secret-dependent services recover correctly after credential replacement.

### 26.2 Security Testing

Security reviews must include:

- Repository secret scanning
- Container image inspection
- CI/CD configuration review
- Secret manager access policy review
- Environment separation validation
- Credential rotation testing
- Access log and alert review
- Tenant isolation testing
- Backup and key recovery exercises

### 26.3 Test Data

Tests must use synthetic, isolated credentials.

Production secrets must not be copied into automated test environments or developer machines for convenience.

## 27. Prohibited Practices

The following practices are strictly prohibited:

- Hardcoding secrets in source code.
- Committing real secrets to Git.
- Embedding privileged credentials in frontend applications.
- Storing production secrets in plaintext configuration files.
- Logging secret values or returning them in API responses.
- Sharing production credentials among unrelated services.
- Using production credentials in routine local development.
- Using default or shared credentials in production.
- Storing secrets in Docker images or build artifacts.
- Passing secrets through URLs, query parameters, or job payloads.
- Sending secrets through unsecured chat, email, or issue trackers.
- Using long-lived administrator credentials where scoped or short-lived credentials are available.
- Treating Base64 encoding as encryption.
- Reusing cryptographic keys for unrelated purposes.
- Assuming deletion from source control removes the risk of an exposed secret.
- Allowing tenant administrators to access platform-wide secrets.
- Disabling secret scanning to bypass a failing pipeline without an approved exception.

## 28. Exceptions

Exceptions must be rare, documented, and time-limited.

Every exception must include:

- Business or technical justification
- Affected services and environments
- Secret classification
- Risk assessment
- Compensating controls
- Responsible owner
- Approval from the appropriate security or engineering authority
- Expiration date
- Remediation plan

Exceptions must be reviewed before expiration. High-risk exceptions require stronger approval and continuous monitoring.

No exception may authorize exposing privileged production secrets to public clients or knowingly committing active production credentials to public source control.

## 29. Ownership and Responsibilities

| Responsibility | Owner |
|---|---|
| Secret classification and ownership | Service owner |
| Secure secret storage | Platform / Infrastructure |
| Application-level secret handling | Backend and frontend engineering |
| CI/CD secret protection | DevOps / Platform |
| Access policy management | Security / Platform |
| Rotation and revocation | Secret owner and operations |
| Incident response | Security and incident response team |
| Self-hosted secret configuration | University deployment administrator |
| Compliance and periodic review | Security leadership |

Each production secret must have a responsible owner and a documented purpose.

## 30. Review Checklist

Before approving a change involving secrets, verify:

- [ ] No secret is hardcoded or committed.
- [ ] All secrets are stored in an approved location.
- [ ] Environment-specific credentials are separated.
- [ ] Secret access follows least privilege.
- [ ] Frontend bundles and public APIs expose no confidential values.
- [ ] Secrets are not included in logs, traces, metrics, or error messages.
- [ ] CI/CD access is restricted and appropriately scoped.
- [ ] Container images and artifacts contain no secrets.
- [ ] Rotation and revocation procedures are documented.
- [ ] Tenant-specific credentials are isolated.
- [ ] Secret scanning is enabled and passing.
- [ ] Recovery and incident response procedures are defined.
- [ ] Required tests cover secret handling and access control.
- [ ] Ownership and review requirements are documented.

## 31. Definition of Done

Secret management is considered complete when:

- All required secrets are identified and classified.
- Production secrets are stored in an approved secrets manager.
- Secret access is identity-based and least-privilege.
- Development, staging, production, and self-hosted configurations are appropriately isolated.
- No secrets are present in source code, frontend bundles, container images, or build artifacts.
- Secret values are excluded from logs and error responses.
- Rotation and revocation procedures are implemented and tested.
- CI/CD pipelines protect secrets and enforce scanning.
- Tenant-specific secret access is isolated and tested.
- Monitoring and auditing are enabled for sensitive secret operations.
- Incident response, backup, and recovery procedures are documented.
- Relevant security tests and reviews have passed.

**Final rule:** A secret is only protected when its storage, access, use, rotation, logging, and revocation are all controlled. Secret management is an ongoing operational responsibility, not a one-time configuration task.
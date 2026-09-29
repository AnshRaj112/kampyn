# Authentication Security

## 1. Purpose

This document defines the mandatory authentication standards for KAMPYN, covering identity verification, credential management, sessions, tokens, multi-factor authentication, account recovery, service identities, and authentication lifecycle management.

Authentication establishes **who or what is making a request**. It does not establish what that identity is permitted to do. Authorization MUST be enforced separately according to `.ai/security/api-security.md`.

These standards apply to:
- Student, faculty, staff, vendor, and administrator accounts.
- University and platform-level administrators.
- SaaS and self-hosted deployments.
- Web, mobile, and SDK clients.
- Internal services and background workers.
- Third-party integrations and API clients.
- Authentication, registration, verification, login, logout, recovery, and credential-management workflows.

Authentication MUST be designed as a centralized, reusable capability rather than implemented independently by individual modules.

## 2. Core Principles

All authentication implementations MUST follow these principles:

1. **Identity verification:** Every protected request must be associated with a verified identity or explicitly authorized service identity.
2. **Least privilege:** Authentication credentials must grant identity, not unrestricted access.
3. **Secure defaults:** Authentication must fail securely when required configuration, credentials, or identity information is missing or invalid.
4. **Credential protection:** Passwords, tokens, session identifiers, API keys, and recovery credentials must be protected throughout their lifecycle.
5. **Short-lived credentials:** Prefer short-lived access credentials with a controlled renewal process.
6. **Revocability:** Provide mechanisms to revoke credentials and terminate compromised sessions.
7. **Explicit identity context:** Every authenticated request must have a validated principal and clearly defined authentication context.
8. **Defense in depth:** Use layered protections, including secure credential storage, rate limiting, MFA where appropriate, and anomaly detection.
9. **No implicit trust:** Do not trust identity information merely because it comes from a frontend, internal network, or previously authenticated service.
10. **Auditability:** Security-sensitive authentication events must be logged safely and be traceable.
11. **Tenant awareness:** Identity and tenant membership must be verified independently.
12. **Standards over custom cryptography:** Use established authentication protocols and maintained cryptographic libraries.

## 3. Authentication Architecture

KAMPYN MUST have a clearly defined authentication boundary responsible for verifying credentials and establishing a trusted principal.

<escape>
Client (Web / Mobile / SDK)
          |
          v
    HTTPS Request
          |
          v
 Authentication Middleware
          |
          v
 Credential Validation
          |
          v
 Identity & Session Resolution
          |
          v
 Principal Context
          |
          v
 Authorization Layer
          |
          v
 Application Services
          |
          v
 Protected Resources
</escape>

### 3.1 Authentication Responsibilities

The authentication subsystem MUST be responsible for:

- User identity verification.
- Credential validation.
- Session creation and lifecycle management.
- Access-token issuance and validation, where used.
- Refresh-token rotation and revocation, where used.
- Password hashing and verification.
- Email or other identity verification workflows.
- Password reset and account recovery.
- MFA enrollment, challenge, and recovery.
- Authentication event logging.
- Account lockout or throttling mechanisms.
- Service identity verification.
- Authentication configuration and policy enforcement.

Authentication MUST NOT be responsible for making every domain-level access decision. Permissions, ownership, tenant boundaries, and resource-level access remain the responsibility of the authorization and application layers.

### 3.2 Centralized Identity

- Use a shared authentication service or module for common identity operations.
- Avoid duplicating password verification, token parsing, or session logic across domain services.
- Define a consistent authenticated principal contract.
- Keep authentication interfaces stable for consuming modules.
- Avoid direct access to authentication internals from unrelated domain modules.
- Separate identity verification from profile, role, and business-domain data.

A domain service MUST NOT implement its own incompatible login or token-validation mechanism without an approved architectural reason.

### 3.3 Authentication Context

After successful authentication, the application MUST establish a trusted principal context containing only validated information necessary for the request.

A principal MAY contain:
- Internal user identifier.
- Identity type.
- Authentication method.
- Authentication time.
- Session or credential identifier.
- Tenant context, when verified.
- MFA completion status or assurance level, where relevant.
- Credential expiry information.

The principal MUST NOT be constructed directly from untrusted request fields.

Avoid placing sensitive credential material, such as raw tokens or passwords, in the principal context.

## 4. Identity and Account Model

### 4.1 Identity Separation

KAMPYN SHOULD separate authentication identity from domain-specific user profiles.

An identity record may contain:
- Internal identity ID.
- Canonical login identifier.
- Credential verifier or external identity reference.
- Authentication status.
- Verification status.
- Credential version or session invalidation version, if used.
- Creation and update timestamps.
- Security-related metadata.

A user profile may contain:
- Display name.
- Profile image.
- Contact information.
- Academic or employment details.
- Tenant membership.
- Domain-specific attributes.

Authentication MUST NOT depend on loading or exposing the entire user profile.

### 4.2 Unique Identifiers

- Use stable, server-generated internal identifiers.
- Enforce uniqueness of canonical login identifiers according to the account policy.
- Normalize identifiers consistently before comparison.
- Do not assume that case-sensitive and case-insensitive identifiers are equivalent unless the policy explicitly defines this.
- Avoid using mutable display names as authentication identifiers.
- Avoid exposing sequential internal IDs where doing so creates unnecessary information disclosure.

### 4.3 Account States

Accounts SHOULD use explicit, validated states, such as:

- `PENDING_VERIFICATION`
- `ACTIVE`
- `SUSPENDED`
- `LOCKED`
- `DISABLED`
- `RECOVERY_REQUIRED`

Allowed transitions MUST be documented and enforced server-side.

For example:
- A pending account must complete the required verification before accessing protected functionality.
- A suspended or disabled account must not authenticate for ordinary access.
- A locked account must follow the defined unlock or recovery process.
- A recovery-required account may be restricted to approved recovery operations.

Do not rely on loosely defined booleans that can produce contradictory account states.

### 4.4 Account Membership

A single identity may have access to one or more universities or deployments where the product model supports it.

- Tenant membership MUST be explicitly represented.
- Authentication MUST establish the identity, while tenant selection and membership verification establish the applicable tenant context.
- Users MUST NOT gain access to a tenant simply by submitting its identifier.
- Tenant membership changes MUST follow authorized workflows.
- Removing membership MUST invalidate or restrict relevant sessions and credentials according to the revocation policy.

## 5. Password Security

### 5.1 Password Storage

If KAMPYN manages passwords directly:

- Passwords MUST be hashed using a modern adaptive password-hashing algorithm such as Argon2id.
- Use parameters appropriate to the production environment and review them periodically.
- Use a unique salt per password, generated by a maintained hashing library.
- Store only the password hash and necessary algorithm parameters.
- Never store plaintext passwords.
- Never store passwords using reversible encryption for routine login verification.
- Never use fast general-purpose hashes such as SHA-256, SHA-1, or MD5 as password hashes.
- Never design a custom password-hashing algorithm.

Password verification MUST use a maintained library with safe comparison behavior.

### 5.2 Password Policy

Password policy MUST be documented and consistently enforced.

- Support long passphrases.
- Set a reasonable minimum length.
- Permit password managers and paste operations.
- Avoid arbitrary composition rules as the only protection.
- Reject known-compromised or commonly used passwords where a suitable mechanism is available.
- Prevent passwords from being truncated silently.
- Apply a documented maximum input length to protect against resource exhaustion while supporting long passwords.
- Avoid forcing routine password changes without a risk-based or policy-based reason.

The exact minimum and maximum lengths MUST be defined in the product's authentication configuration.

### 5.3 Password Verification

During login:
1. Validate the login identifier and password input format.
2. Apply rate limiting and abuse controls.
3. Locate the identity using the canonical identifier.
4. Verify the supplied password against the stored hash using the approved library.
5. Check account status and applicable authentication requirements.
6. Perform MFA or other required verification.
7. Create a session or issue credentials only after all required checks succeed.
8. Record the authentication event safely.

Authentication failures MUST return generic messages that do not reveal whether the identifier exists, the password is incorrect, or the account is in a restricted state.

### 5.4 Password Rehashing

When password-hashing parameters change:
- Detect hashes that no longer meet the approved policy.
- Rehash passwords after successful authentication when appropriate.
- Replace the stored hash atomically.
- Avoid requiring unnecessary password resets solely due to a parameter update.
- Track and monitor the proportion of hashes that remain below the current standard.

### 5.5 Password Changes

Password-change endpoints MUST:
- Require an authenticated session.
- Require current-password verification or an equivalent step-up mechanism, where appropriate.
- Validate the new password against the current policy.
- Prevent accidental changes from stale or unauthorized sessions.
- Invalidate or rotate relevant credentials according to the session policy.
- Notify the user through a verified communication channel where appropriate.
- Record the event without logging the password.

Changing a password MUST NOT silently preserve compromised refresh tokens or long-lived sessions unless an explicit, reviewed exception exists.

## 6. Login and Authentication Flows

### 6.1 Login

Login endpoints MUST:
- Accept credentials only over HTTPS.
- Validate input before expensive processing.
- Apply rate limiting and abuse detection.
- Avoid account enumeration.
- Avoid unnecessary differences in timing or response structure that reveal account state.
- Check account status.
- Apply required MFA or step-up verification.
- Issue credentials only after all required checks succeed.
- Record safe audit information.

Do not log credentials or return tokens in error messages.

### 6.2 Registration

Registration MUST:
- Validate the registration payload.
- Enforce unique canonical identity rules.
- Prevent unauthorized role assignment.
- Prevent users from selecting privileged account types without an authorized process.
- Apply anti-abuse protections.
- Require verification where configured.
- Create accounts in an appropriate initial state.
- Avoid exposing sensitive account-existence information.

New users MUST NOT be able to self-assign administrator, moderator, vendor-owner, or other privileged permissions through registration payloads.

### 6.3 Email and Identity Verification

When email verification is used:
- Verification tokens MUST be generated using cryptographically secure randomness.
- Tokens MUST be single-use and expire within a documented period.
- Tokens MUST be securely stored, preferably as a verifier or hash.
- Verification endpoints MUST be rate-limited.
- Tokens MUST be bound to the intended account and verification purpose.
- Successful verification MUST atomically update the relevant verification state.
- Repeated use of a consumed token MUST fail.
- Resending verification messages MUST not invalidate accounts in unexpected ways.

Email verification establishes control of an email inbox at a point in time. It does not independently prove a person's legal identity or university affiliation.

### 6.4 University-Specific Authentication

If KAMPYN supports university-specific authentication, such as institutional SSO:

- Use established protocols such as OpenID Connect or SAML through maintained libraries or identity providers.
- Validate issuer, audience, signature, expiry, nonce, and state according to the selected protocol.
- Use exact, approved redirect URIs.
- Validate the identity provider's response before creating a local session.
- Map external identities to internal identities using stable provider identifiers.
- Do not automatically trust email-domain membership as proof of university authorization.
- Enforce tenant membership and account status after successful external authentication.
- Define account linking and unlinking procedures.
- Prevent account takeover through unsafe automatic identity linking.

Institutional SSO configuration MUST be isolated by deployment or tenant as required by the identity model.

## 7. Session Management

### 7.1 Session Requirements

For session-based authentication:
- Session IDs MUST be generated using cryptographically secure randomness.
- Session identifiers MUST be unpredictable and sufficiently long.
- Session identifiers MUST be invalidated on logout.
- Sessions MUST expire according to configured idle and absolute lifetimes.
- Sessions MUST be rotated after authentication and relevant privilege changes.
- Session validity MUST be enforced server-side.
- Sessions MUST be bound to the correct identity and applicable tenant context.
- Session state MUST not be shared accidentally across users or tenants.

### 7.2 Session Cookies

For browser-based applications using cookies:

- Set `Secure` in production.
- Set `HttpOnly` for session cookies.
- Set `SameSite` explicitly based on the authentication flow.
- Set an appropriate `Path`.
- Avoid broad `Domain` attributes unless explicitly required and reviewed.
- Use a suitable expiration or session-cookie policy.
- Use HTTPS for all session-bearing requests.
- Implement CSRF defenses for cookie-authenticated state-changing operations.

Do not expose session cookies to JavaScript unless there is a documented, reviewed requirement that cannot be satisfied through a safer design.

### 7.3 Session Expiration

Session policy MUST define:
- Idle timeout.
- Absolute lifetime.
- Reauthentication requirements.
- Session renewal behavior.
- Revocation behavior.
- Handling of suspended or disabled accounts.

Privileged accounts and high-risk sessions SHOULD have stricter expiration and reauthentication requirements.

### 7.4 Concurrent Sessions

KAMPYN SHOULD support session visibility and management where practical.

Users SHOULD be able to:
- View active sessions or authenticated devices.
- Revoke individual sessions.
- Revoke all other sessions.
- Review relevant recent authentication activity.

Session management APIs MUST enforce ownership and must not expose other users' session details.

### 7.5 Logout

Logout MUST invalidate the current session or refresh-token family according to the authentication design.

For cookie-based sessions:
- Invalidate the server-side session.
- Expire the corresponding cookie using matching cookie attributes.

For token-based authentication:
- Revoke the refresh token or token family where supported.
- Clear client-side credentials.
- Ensure the documented access-token expiration or revocation model is followed.

A client-side redirect or deletion of local UI state alone is not sufficient to invalidate a server-managed session.

## 8. Access Tokens

### 8.1 Token Requirements

Where access tokens are used:
- Use a well-reviewed token format and maintained library.
- Use cryptographically secure signing or authenticated issuance mechanisms.
- Set a short, documented lifetime appropriate to the risk.
- Validate issuer, audience, expiry, signature, and required claims.
- Enforce supported algorithms explicitly.
- Reject malformed, unsigned, expired, revoked, or incorrectly scoped tokens.
- Avoid including unnecessary personal or sensitive information.
- Use token scopes or permissions only where their meaning and revocation behavior are well defined.

### 8.2 JWT Security

When JWTs are used:
- Validate the signature using trusted keys.
- Validate the expected algorithm using server configuration.
- Validate `iss`, `aud`, `exp`, and `nbf` where applicable.
- Validate `iat` when issuance-time restrictions are part of the policy.
- Validate required application-specific claims.
- Reject unexpected token types and unsupported key identifiers.
- Prevent algorithm-confusion and key-selection vulnerabilities.
- Do not trust claims before signature and standard-claim validation.
- Do not use a decoded JWT as proof of authenticity.

JWT payloads are generally readable by whoever holds the token. Do not put secrets or sensitive personal data in claims unless they are appropriately protected and required.

### 8.3 Access-Token Lifetime

Access-token lifetimes MUST be defined centrally and aligned with:
- Risk of the protected operations.
- Client type.
- Revocation requirements.
- Reauthentication requirements.
- User experience.
- Operational constraints.

Prefer short-lived access tokens and avoid unnecessarily long-lived bearer credentials.

### 8.4 Token Revocation

The system MUST define how access is revoked when:
- A user logs out.
- A password changes.
- An account is suspended or disabled.
- A user's permissions change.
- A tenant membership is removed.
- A credential is suspected to be compromised.
- An administrator terminates a session.

If access tokens are not individually revocable, the system MUST document the residual exposure window and use compensating controls, such as short lifetimes, token-version checks, or session-backed validation, where appropriate.

## 9. Refresh Tokens

### 9.1 Refresh-Token Security

Refresh tokens MUST:
- Be generated using cryptographically secure randomness.
- Be transmitted only over HTTPS.
- Be stored securely on the client.
- Be associated with an identity and session or token family.
- Have a documented expiration policy.
- Support revocation.
- Be protected against replay.
- Never be logged or exposed through analytics.

Refresh tokens SHOULD be opaque unless there is a specific, reviewed reason to use another design.

### 9.2 Refresh-Token Rotation

Where refresh tokens are used:
- Rotate the refresh token after successful use.
- Invalidate the previous token when rotation succeeds.
- Detect reuse of an invalidated token.
- Revoke the affected token family or session when reuse indicates possible compromise.
- Handle concurrent refresh requests safely.
- Ensure rotation is atomic to prevent duplicate active credentials.

### 9.3 Refresh-Token Storage

Store refresh-token verifiers or hashes server-side where applicable, rather than raw reusable tokens.

Records SHOULD include:
- Token or family identifier.
- Identity ID.
- Session ID.
- Token verifier.
- Issuance timestamp.
- Expiration timestamp.
- Revocation status.
- Rotation or replacement reference.
- Relevant client metadata, minimized according to privacy requirements.

### 9.4 Refresh Failure

If refresh fails due to invalidity, expiry, revocation, or detected reuse:
- Do not issue new credentials.
- Invalidate affected credentials as required by policy.
- Return a safe authentication error.
- Require reauthentication where necessary.
- Record a security event without logging the token.

## 10. Multi-Factor Authentication

### 10.1 MFA Policy

MFA SHOULD be supported for all user groups where appropriate and MUST be required for designated privileged or high-risk operations according to KAMPYN's deployment policy.

MFA requirements SHOULD consider:
- Platform administrator access.
- University administrator access.
- Financial operations.
- Sensitive HR access.
- Bulk data exports.
- Credential and permission management.
- Account recovery.
- Other high-impact actions.

### 10.2 Supported Factors

Depending on deployment capabilities, MFA MAY support:
- TOTP authenticator applications.
- WebAuthn/passkeys or security keys.
- Other reviewed phishing-resistant authentication methods.

SMS-based OTP SHOULD NOT be treated as phishing-resistant MFA and SHOULD NOT be the only factor protecting highly privileged accounts where stronger methods are available.

### 10.3 MFA Enrollment

MFA enrollment MUST:
- Require an authenticated account.
- Verify the new factor before activating it.
- Protect enrollment secrets during setup.
- Avoid exposing MFA secrets in logs or analytics.
- Provide a secure recovery process.
- Notify users of important enrollment or removal events where appropriate.

### 10.4 MFA Challenges

MFA challenges MUST:
- Be associated with the correct identity and authentication transaction.
- Expire within a documented period.
- Be single-use where applicable.
- Be rate-limited.
- Prevent brute-force attempts.
- Avoid revealing unnecessary account information.
- Be verified server-side.

### 10.5 Recovery Codes

Recovery codes MUST:
- Be generated securely.
- Be sufficiently unpredictable.
- Be single-use.
- Be stored securely, preferably as verifiers or hashes.
- Be shown to the user only through a protected enrollment flow.
- Be invalidated after use or regeneration.
- Be covered by an audited recovery process.

### 10.6 MFA Reset

MFA removal or reset MUST require a sufficiently strong identity-verification process.

Privileged-account MFA resets SHOULD require additional verification or an approved administrator-assisted recovery workflow.

MFA recovery MUST NOT become an easier route to account takeover than normal authentication.

## 11. Account Recovery

### 11.1 Recovery Principles

Account recovery MUST restore legitimate access without creating an authentication bypass.

Recovery flows MUST:
- Verify control of an approved recovery channel or use another approved identity-proofing process.
- Use short-lived, single-use recovery credentials.
- Apply rate limiting and abuse detection.
- Avoid account enumeration.
- Invalidate recovery credentials after use.
- Record recovery events.
- Notify the user of sensitive account changes where appropriate.

### 11.2 Password Reset

Password-reset tokens MUST:
- Be generated with cryptographically secure randomness.
- Be linked to the intended account and recovery purpose.
- Have a documented short expiration period.
- Be stored as a verifier or hash where feasible.
- Be single-use.
- Be invalidated after successful reset.
- Be invalidated or superseded when a new reset request is issued according to the defined policy.
- Be excluded from logs, analytics, and third-party tracking.

Reset pages MUST use safe URL handling and must not leak tokens through unnecessary redirects, external resources, or referrer transmission.

### 11.3 Recovery Completion

After successful account recovery:
- Update the credential securely.
- Invalidate relevant reset tokens.
- Revoke or rotate existing sessions and refresh tokens according to the compromise-risk policy.
- Require MFA re-verification where applicable.
- Notify the user through an established channel where appropriate.
- Record an audit event.

### 11.4 Support-Assisted Recovery

Support-assisted recovery MUST follow a documented identity-verification procedure.

Support staff MUST NOT:
- Request users' passwords.
- Ask users to disclose full active tokens or recovery codes.
- Bypass identity verification based solely on a support request.
- Grant themselves permanent access to user accounts.
- Perform unauthorized credential resets.

High-risk recovery actions SHOULD require approval or additional verification.

## 12. Account Protection and Abuse Prevention

### 12.1 Rate Limiting

Authentication-related endpoints MUST have risk-appropriate rate limits.

Apply protections to:
- Login attempts.
- Registration attempts.
- Verification-token requests.
- Password reset requests.
- MFA challenges.
- Recovery-code attempts.
- Refresh-token requests.
- Session creation.
- Credential-management operations.

Rate limiting SHOULD consider multiple dimensions, such as account identifier, IP address, session, and deployment or tenant, while avoiding unnecessary denial of service against legitimate users.

### 12.2 Brute-Force and Credential Stuffing

The authentication system SHOULD detect and mitigate:
- Repeated failed logins.
- Credential stuffing.
- Password spraying.
- Automated account creation.
- OTP guessing.
- Recovery-flow abuse.
- Repeated attempts to validate account existence.

Controls MAY include progressive delays, throttling, risk-based challenges, temporary restrictions, or other reviewed mechanisms.

### 12.3 Account Lockout

Account lockout policy MUST balance security and availability.

- Lockout rules MUST be documented.
- Permanent lockout MUST NOT occur solely due to unauthenticated requests.
- Temporary restrictions SHOULD have bounded durations or a verified recovery process.
- Lockout state MUST be maintained securely.
- Account recovery MUST remain available to legitimate users.
- Lockout mechanisms MUST not enable trivial targeted denial of service.

### 12.4 Suspicious Authentication

The system SHOULD monitor for suspicious authentication activity, including:
- Unusual login frequency.
- Repeated failed attempts.
- Refresh-token reuse.
- Unexpected credential changes.
- Unusual session creation patterns.
- Access from anomalous environments where detection is supported.

Risk signals MUST be used carefully and should not be treated as conclusive proof of malicious activity.

## 13. Reauthentication and Step-Up Authentication

Some actions require stronger or more recent identity verification than an ordinary authenticated session.

Step-up authentication SHOULD be required for high-impact operations such as:
- Changing passwords or recovery channels.
- Enrolling or removing MFA.
- Changing privileged permissions.
- Creating or revoking API credentials.
- Initiating sensitive financial actions.
- Issuing high-value refunds.
- Exporting particularly sensitive information.
- Changing critical tenant security settings.

The system MUST define:
- Which actions require step-up authentication.
- Which factors satisfy the requirement.
- How recent authentication must be.
- How the step-up result is associated with the intended operation.
- How the requirement is enforced across clients and backend services.

A frontend-only confirmation dialog MUST NOT be considered step-up authentication.

## 14. External Identity Providers and SSO

### 14.1 Protocol Requirements

When using external identity providers, use maintained implementations of recognized protocols such as OpenID Connect or SAML.

- Validate protocol responses according to the protocol specification.
- Verify signatures, issuers, audiences, and relevant time claims.
- Use state and nonce protections where applicable.
- Validate exact redirect URIs.
- Protect authorization-code exchanges.
- Use PKCE for applicable OAuth authorization-code flows, particularly public clients.
- Avoid insecure implicit-flow designs.
- Protect client secrets and signing keys.

### 14.2 Identity Linking

Identity linking MUST:
- Require a verified existing session or a controlled account-recovery process.
- Verify control of the external identity.
- Use stable provider-specific subject identifiers.
- Prevent linking an external identity to an account merely because an email address matches.
- Record linking and unlinking events.
- Provide a secure unlinking process that avoids leaving an account without a usable authentication method.

### 14.3 Provider Availability

The application MUST define behavior when an identity provider is unavailable.

- Do not silently bypass SSO requirements.
- Do not accept stale or unverified identity assertions.
- Avoid issuing new credentials without successful authentication.
- Preserve safe error handling and observability.
- Support only documented fallback mechanisms.

## 15. Service-to-Service Authentication

### 15.1 Service Identity

Every protected internal service MUST authenticate the calling service using a recognized service-identity mechanism.

Suitable mechanisms MAY include:
- Mutual TLS.
- Cloud workload identity.
- Short-lived signed service tokens.
- Other reviewed identity mechanisms appropriate to the environment.

### 15.2 Service Credential Requirements

Service credentials MUST:
- Be scoped to the calling service and intended audience.
- Have an explicit expiration or rotation policy.
- Be stored outside source code and container images.
- Be revocable.
- Be excluded from logs.
- Be restricted to required operations.
- Be separated between development, staging, and production.

### 15.3 Background Workers

Background workers MUST use service identities or securely delegated, scoped credentials.

Jobs MUST NOT rely on a user's old access token as a substitute for an explicit execution identity.

Where a job acts on behalf of a user:
- Preserve the initiating user's identity as audit context.
- Preserve verified tenant context.
- Revalidate authorization when execution occurs if permissions or resource state may have changed.
- Restrict the job's delegated authority to the intended operation.

### 15.4 Internal Trust

A request originating from an internal network MUST NOT be considered authenticated solely because of its network location.

Every service MUST validate the caller's identity and authorize the requested operation according to the defined service contract.

## 16. API Keys and Machine Credentials

API keys are intended for machine clients and integrations, not as a substitute for user authentication.

- Generate keys using cryptographically secure randomness.
- Use scoped permissions.
- Associate keys with an explicit owner or integration.
- Store key verifiers securely where feasible.
- Display secrets only at creation time where possible.
- Support revocation and rotation.
- Apply rate limits.
- Monitor usage.
- Avoid embedding keys in browser bundles, public repositories, or mobile application source.
- Never treat possession of an API key as proof of a particular human identity.

Sensitive actions initiated through API keys MUST follow the integration's explicit permission model.

## 17. Authentication for Web, Mobile, and SDK Clients

### 17.1 Web Clients

Web clients MUST:
- Use HTTPS.
- Follow the approved session or token design.
- Avoid exposing long-lived credentials to JavaScript.
- Apply CSRF protections when using automatically attached browser credentials.
- Clear local authentication state after logout or account changes.
- Avoid caching sensitive authentication responses in shared caches.
- Handle expiration and revocation safely.

### 17.2 Mobile Clients

Mobile clients SHOULD:
- Use platform-supported secure storage for credentials.
- Avoid storing long-lived secrets in ordinary application preferences.
- Use secure transport.
- Avoid embedding shared confidential API keys.
- Support credential revocation and session expiration.
- Avoid leaking tokens through logs, crash reports, clipboard operations, or analytics.
- Handle device loss and account compromise through server-side revocation.

A mobile application binary MUST be treated as an untrusted client. Any embedded secret can potentially be extracted.

### 17.3 SDK Clients

KAMPYN SDKs MUST:
- Use documented authentication interfaces.
- Avoid implementing independent, incompatible token-validation rules.
- Keep credentials out of logs and exception messages.
- Support token renewal and error handling according to the official contract.
- Avoid persisting secrets unless explicitly configured by the integrator.
- Provide secure examples and documentation.
- Make the distinction between user credentials and service credentials explicit.

SDKs MUST NOT hardcode privileged credentials or attempt to bypass backend authentication controls.

## 18. Authentication and Tenant Isolation

Authentication identifies the principal; tenant selection and membership determine the applicable organizational context.

- Resolve tenant membership from trusted identity and server-side membership records.
- Validate tenant membership for each tenant-scoped session or operation as required.
- Do not trust arbitrary tenant headers or request-body fields.
- Bind sessions and credentials to the intended identity.
- Ensure tenant switching requires verified membership and an explicit server-side operation.
- Revalidate access after tenant membership changes.
- Prevent shared browser state from carrying one tenant's credentials into another tenant's context.
- Include tenant scope in session records or equivalent context where applicable.
- Avoid issuing tenant-scoped credentials with ambiguous or excessive access.

Authentication caches MUST be scoped and invalidated so that identity or membership updates do not leave unintended access paths.

## 19. Credential and Secret Management

### 19.1 Secret Storage

Authentication secrets MUST be stored in an approved secret manager or protected deployment configuration.

Secrets include:
- JWT signing keys.
- Session-signing keys.
- OAuth client secrets.
- SSO credentials.
- MFA encryption keys, where applicable.
- Password-reset signing or verification secrets, where applicable.
- API keys.
- Service credentials.

Secrets MUST NOT be:
- Committed to source control.
- Included in frontend bundles.
- Written into Docker images.
- Printed in application logs.
- Included in test fixtures with production values.
- Shared across unrelated environments.

### 19.2 Key Management

Cryptographic keys MUST:
- Have a defined owner and purpose.
- Use appropriate key sizes and algorithms.
- Be generated using secure mechanisms.
- Be access-controlled.
- Have a rotation and revocation strategy.
- Be backed up securely where recovery is required.
- Be monitored for unexpected access.

Key rotation MUST account for verification of previously issued credentials during the planned transition window without accepting revoked or untrusted keys.

### 19.3 Environment Separation

Development, staging, production, and self-hosted deployments MUST use separate credentials and key material.

Production authentication secrets MUST NOT be copied into local environments or shared development systems.

### 19.4 Emergency Rotation

The authentication system MUST document how to:
- Rotate signing keys.
- Revoke compromised credentials.
- Invalidate sessions.
- Revoke refresh-token families.
- Disable compromised identity providers or integrations.
- Restore service safely after emergency credential rotation.

## 20. Authentication Caching

Authentication-related caching MUST preserve correctness, tenant isolation, and revocation requirements.

- Cache only data with a defined ownership and invalidation strategy.
- Use short, explicit TTLs for security-sensitive cached state.
- Avoid caching raw passwords, access tokens, refresh tokens, or recovery credentials.
- Include identity, tenant, and credential scope in cache keys as applicable.
- Invalidate relevant cache entries after account suspension, credential reset, role changes, or membership removal.
- Define behavior when the cache is unavailable.
- Avoid relying exclusively on stale cached authorization or account status for high-risk operations.

Caching MUST NOT create an undocumented period during which revoked credentials continue to provide access.

## 21. Authentication Failure Handling

Authentication failures MUST be handled consistently and safely.

- Return generic external messages where account enumeration is a risk.
- Use appropriate HTTP status codes.
- Avoid exposing internal authentication-provider errors.
- Avoid revealing credential-validation details.
- Apply rate limiting to repeated failures.
- Record safe security events.
- Do not issue partial credentials when authentication fails.
- Avoid redirecting to untrusted destinations after authentication errors.
- Fail securely when required identity-provider validation or cryptographic verification is unavailable.

Unexpected internal failures MUST NOT be interpreted as successful authentication.

## 22. Authentication Logging and Auditing

### 22.1 Events to Record

Record relevant events, including:
- Successful login.
- Failed login.
- Logout.
- Session creation and revocation.
- Refresh-token rotation and reuse detection.
- Password changes.
- Password-reset requests and completions.
- Email or identity verification.
- MFA enrollment, challenge failures, and reset.
- Recovery-code use.
- Account lockout, suspension, or reactivation.
- SSO identity linking and unlinking.
- API-key creation, rotation, and revocation.
- Privileged authentication events.
- Authentication policy changes.

### 22.2 Safe Metadata

Authentication logs MAY contain:
- Internal user or service ID.
- Tenant ID where applicable.
- Event type.
- Timestamp.
- Success or failure outcome.
- Authentication method.
- Session or credential identifier that is not itself a usable secret.
- Correlation ID.
- Coarse client metadata when justified.

Collect IP addresses, device information, and other personal data only as necessary, with documented access and retention controls.

### 22.3 Prohibited Logging

Never log:
- Passwords.
- Raw access tokens.
- Raw refresh tokens.
- Session cookies.
- API keys.
- MFA secrets.
- Recovery codes.
- Password-reset tokens.
- Private keys.
- Full authentication request bodies containing credentials.

Logs MUST be protected against unauthorized access and tampering.

## 23. Privacy and Data Retention

Authentication data MUST be collected and retained only for documented security, operational, and legal purposes.

- Minimize personal data stored in authentication records.
- Define retention periods for sessions, authentication events, and recovery metadata.
- Restrict access to authentication logs.
- Securely delete expired credentials and unnecessary metadata.
- Avoid using authentication data for unrelated profiling.
- Document any device or risk telemetry collected.
- Apply relevant privacy requirements to SaaS and self-hosted deployments.

Authentication telemetry MUST NOT unnecessarily expose sensitive personal information to unrelated KAMPYN modules.

## 24. Authentication in Self-Hosted Deployments

Self-hosted KAMPYN installations MUST retain secure authentication defaults.

Deployment documentation MUST explain:
- Initial administrator provisioning.
- Secret and signing-key configuration.
- Session and token settings.
- Identity-provider configuration.
- Password and MFA policy.
- Email or notification requirements for verification and recovery.
- Credential rotation.
- Session revocation.
- Backup and recovery considerations.
- Authentication monitoring.

Self-hosted installations MUST NOT ship with shared production credentials, permanent default passwords, or publicly known signing keys.

Bootstrap credentials MUST be unique to the deployment, securely delivered, and rotated or invalidated after initial setup.

## 25. Authentication Configuration

Authentication settings MUST be centralized, typed, validated, and environment-aware.

Configuration SHOULD include:
- Token issuer and audience.
- Token lifetimes.
- Session idle and absolute timeouts.
- Refresh-token policy.
- Password-hashing parameters.
- Login and recovery rate limits.
- MFA requirements.
- Account lockout or throttling policy.
- Allowed identity providers.
- Redirect URI allowlists.
- Cookie security attributes.
- Key references and rotation metadata.
- Verification-token lifetime.
- Recovery-token lifetime.

Configuration MUST:
- Be validated at startup.
- Reject invalid or insecure production values.
- Avoid silent fallback to weak defaults.
- Separate environment-specific settings.
- Avoid logging secret values.
- Be covered by tests.

Security-sensitive defaults MUST be conservative, documented, and reviewed before production deployment.

## 26. Framework and Implementation Standards

### 26.1 General

- Use maintained authentication libraries and protocol implementations.
- Avoid custom cryptographic primitives.
- Keep authentication code modular and cohesive.
- Centralize token issuance and validation.
- Centralize password hashing and verification.
- Avoid duplicating authentication rules across handlers.
- Keep authentication interfaces explicit.
- Separate identity, session, and authorization responsibilities.
- Use dependency injection or equivalent explicit dependencies for cryptographic and identity services.
- Avoid global mutable authentication state.

### 26.2 Go Backend

For Go services:
- Use maintained libraries for cryptographic and protocol operations.
- Use `context.Context` for request-scoped identity propagation where appropriate.
- Avoid storing request identity in mutable global variables.
- Ensure authentication middleware does not bypass service-level authorization.
- Use typed principal structures.
- Use constant-time comparison for custom secret-derived comparisons where applicable.
- Apply request deadlines and bounded resource usage.
- Handle concurrency safely during session and token rotation.

### 26.3 TypeScript and Next.js Clients

For TypeScript and Next.js clients:
- Use strict TypeScript settings.
- Validate external authentication responses at runtime.
- Keep credentials out of client bundles and server-rendered HTML.
- Use server-side authentication checks for protected server operations.
- Avoid trusting client-managed role or tenant state.
- Clear TanStack Query and other user-scoped cached data on identity changes.
- Reset user-scoped Zustand state when the authenticated identity changes.
- Avoid leaking authenticated data through shared caching or static rendering.
- Keep server-only authentication logic in server-only modules.

Frontend state MUST never be treated as the source of truth for authentication or permissions.

## 27. Testing Requirements

Authentication code MUST have automated tests covering normal, adversarial, and failure scenarios.

### 27.1 Unit Tests

Test:
- Password hashing and verification.
- Token issuance and validation.
- Expiration and claim validation.
- Session lifecycle.
- Refresh-token rotation.
- Credential revocation.
- Account-state transitions.
- Verification-token consumption.
- Recovery-token expiry and reuse.
- MFA challenge validation.
- Configuration validation.
- Principal construction and propagation.

### 27.2 Integration Tests

Test:
- Registration and verification.
- Successful and failed login.
- Logout and session invalidation.
- Password change and recovery.
- Refresh-token rotation and replay detection.
- MFA enrollment and challenges.
- SSO callback validation.
- Service-to-service authentication.
- Account suspension and reactivation.
- Tenant membership changes.
- Credential revocation propagation.

### 27.3 Security Tests

Test:
- Invalid and expired tokens.
- Invalid signatures and unsupported algorithms.
- Wrong issuer or audience.
- Token substitution across clients or tenants.
- Account enumeration behavior.
- Brute-force and rate-limit enforcement.
- Session fixation and rotation.
- Refresh-token reuse.
- Recovery-flow abuse.
- Unauthorized privilege escalation.
- MFA bypass attempts.
- Credential leakage through errors or logs.
- Authentication behavior during dependency failures.

### 27.4 Concurrency Tests

Test concurrent:
- Login and session creation.
- Refresh-token rotation.
- Logout and refresh.
- Password reset and active-session revocation.
- MFA challenge verification.
- Account suspension and authenticated requests.

Concurrent operations MUST not produce duplicate valid credentials, bypass revocation, or leave inconsistent account state.

### 27.5 Test Data

- Use synthetic identities and credentials.
- Never use real production passwords or tokens in tests.
- Avoid reusable static secrets that could be mistaken for production credentials.
- Keep test signing keys separate from production key material.
- Ensure tests do not leak credentials through snapshots, reports, or logs.

## 28. Monitoring and Alerting

Authentication monitoring SHOULD cover:
- Login success and failure rates.
- Repeated failures by account or source.
- Refresh-token reuse.
- Session creation and revocation rates.
- Password-reset and recovery activity.
- MFA failure rates.
- Privileged login activity.
- SSO validation failures.
- API-key usage anomalies.
- Authentication-service latency and errors.
- Identity-provider availability.
- Key rotation and credential-revocation events.

Alerts MUST be actionable, appropriately prioritized, and routed to responsible operators.

Monitoring MUST avoid including raw credentials or unnecessary personal data.

## 29. Incident Response

Authentication services MUST support containment and recovery for:
- Stolen credentials.
- Compromised user accounts.
- Compromised administrator accounts.
- Signing-key compromise.
- Refresh-token theft.
- Session hijacking.
- Malicious or compromised identity providers.
- API-key leakage.
- Recovery-flow exploitation.

Incident procedures SHOULD define:
- Credential revocation.
- Session termination.
- Refresh-token-family invalidation.
- Signing-key rotation.
- Account suspension.
- MFA reset.
- Identity-provider disconnection.
- User notification.
- Audit evidence preservation.
- Impact assessment across tenants.
- Secure service restoration.

After an incident, review the root cause, affected authentication paths, revocation effectiveness, monitoring gaps, and necessary policy changes.

## 30. Prohibited Practices

The following practices are prohibited:

- Storing plaintext passwords.
- Using fast general-purpose hashes as password hashes.
- Implementing custom cryptographic algorithms.
- Trusting unverified JWT claims.
- Accepting unsigned tokens.
- Using unrestricted or unvalidated redirect URIs.
- Exposing access tokens, refresh tokens, or secrets in URLs.
- Storing long-lived credentials in insecure client storage.
- Embedding privileged API keys in frontend or mobile application code.
- Logging passwords, tokens, cookies, or recovery codes.
- Treating authentication as authorization.
- Trusting client-supplied identity, role, or tenant claims.
- Allowing self-assignment of privileged roles.
- Using account recovery as an authentication bypass.
- Allowing unlimited login, OTP, or recovery attempts.
- Issuing credentials after partial or failed authentication.
- Silently accepting expired, revoked, or malformed credentials.
- Disabling token signature or TLS verification in production.
- Using shared default credentials across deployments.
- Relying on frontend state to establish authenticated identity.
- Bypassing MFA for privileged operations without an approved exception.
- Keeping compromised sessions active without a documented risk decision.
- Using internal network location as the only proof of service identity.

## 31. Authentication Review Checklist

### Identity and Architecture
- [ ] Authentication responsibilities are clearly separated from authorization.
- [ ] Identity and profile data are modeled separately where appropriate.
- [ ] Principal context is created only from validated identity information.
- [ ] Account states and transitions are explicit.
- [ ] Tenant membership is verified independently of authentication.

### Credentials
- [ ] Passwords use an approved adaptive hashing algorithm.
- [ ] Password policy is documented and enforced.
- [ ] Tokens use maintained, reviewed implementations.
- [ ] Token claims and signatures are validated.
- [ ] Access-token lifetimes are appropriate.
- [ ] Refresh-token rotation and replay detection are implemented where applicable.
- [ ] Credential revocation is supported.
- [ ] Secrets are managed securely and separated by environment.

### Sessions and MFA
- [ ] Session IDs are unpredictable.
- [ ] Session expiration and rotation are enforced.
- [ ] Cookies have appropriate security attributes.
- [ ] Logout invalidates server-side credentials as required.
- [ ] MFA is required for designated high-risk access.
- [ ] MFA enrollment and recovery are protected.
- [ ] Step-up authentication is implemented for required operations.

### Recovery and Abuse
- [ ] Registration and login are rate-limited.
- [ ] Authentication failures avoid account enumeration.
- [ ] Password reset uses short-lived, single-use credentials.
- [ ] Recovery flows have strong identity verification.
- [ ] Lockout and throttling avoid trivial denial of service.
- [ ] Suspicious authentication activity is monitored.

### Integrations and Clients
- [ ] SSO responses are validated.
- [ ] Identity linking is secure.
- [ ] Service identities are explicit and scoped.
- [ ] Mobile and SDK clients follow approved credential-handling rules.
- [ ] API keys are restricted, revocable, and monitored.
- [ ] Tenant context is validated and propagated safely.

### Operations and Testing
- [ ] Authentication events are logged without secrets.
- [ ] Security-sensitive events are auditable.
- [ ] Configuration is typed and validated.
- [ ] Automated tests cover credential lifecycle and failure scenarios.
- [ ] Concurrency and revocation behavior are tested.
- [ ] Monitoring and alerting are configured.
- [ ] Incident response procedures are documented.
- [ ] Self-hosted setup uses secure defaults.

## 32. Definition of Done

Authentication functionality is considered production-ready only when:

1. Identity verification is implemented using approved mechanisms.
2. Authentication responsibilities are separated from authorization.
3. Passwords and other credentials are securely generated, stored, verified, and revoked.
4. Sessions and tokens have documented lifetimes and lifecycle controls.
5. Refresh-token rotation and replay protection are implemented where applicable.
6. Account recovery and identity verification flows resist abuse.
7. MFA and step-up authentication are implemented according to risk and deployment policy.
8. Tenant membership and service identities are independently validated.
9. Authentication failures are handled securely without exposing sensitive details.
10. Secrets and keys are managed, rotated, and separated by environment.
11. Authentication events are auditable without exposing credential material.
12. Automated tests cover successful, failed, adversarial, and concurrent scenarios.
13. Monitoring and incident response procedures are ready.
14. Self-hosted deployments can configure authentication securely.
15. Authentication changes have passed the required code and security reviews.

**Authentication is a critical trust boundary in KAMPYN. No protected operation may rely on an unverified identity, and no authentication mechanism may be considered complete without secure credential lifecycle management, revocation, testing, and operational readiness.**
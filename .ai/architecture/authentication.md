# KAMPYN Authentication Architecture

## 1. Purpose

This document defines the authentication architecture for KAMPYN.

Authentication establishes the identity of a user, service, or trusted system interacting with KAMPYN.

Authentication must provide:

- Strong identity verification.
- Secure credential handling.
- Secure session management.
- Token lifecycle management.
- Multi-tenant identity context.
- Account recovery.
- Session revocation.
- Protection against credential abuse.
- Clear separation between authentication and authorization.
- Compatibility with self-hosted university deployments.

Authentication must never be treated as a frontend-only concern.

---

# 2. Authentication vs Authorization

Authentication answers:

```text
Who are you?
```

Authorization answers:

```text
What are you allowed to do?
```

The systems must remain conceptually separate.

```text
Authentication
    ↓
Identity
    ↓
Tenant Membership
    ↓
Authorization
    ↓
Resource Access
```

A successfully authenticated user is not automatically authorized to perform every operation.

---

# 3. Authentication Architecture

The general authentication flow is:

```text
Client
  ↓
Authentication Request
  ↓
Authentication Service
  ↓
Credential / Identity Provider Verification
  ↓
Identity Resolution
  ↓
Session / Token Issuance
  ↓
Authenticated Request
  ↓
Authentication Middleware
  ↓
Identity Context
  ↓
Authorization
  ↓
Application
```

Authentication infrastructure must remain separate from business-domain logic.

---

# 4. Identity Model

A KAMPYN identity should have a stable internal identifier.

The identity may represent:

- Student.
- Faculty member.
- Staff member.
- Administrator.
- Vendor/operator.
- Institution-managed user.
- Service account where explicitly supported.

Do not use email addresses as the permanent internal identity identifier.

Email addresses may change.

Prefer:

```text id="n5cxk2"
userId = stable internal identifier
email  = mutable identity attribute
```

---

# 5. User Identity

The user identity should contain only the information required for authentication and identity resolution.

Typical attributes may include:

- Internal user ID.
- Email.
- Phone number where required.
- Display name.
- Account status.
- Authentication provider.
- Provider subject identifier.
- Email verification state.
- Security metadata.
- Created/updated timestamps.

Do not store unnecessary personal information in the authentication model.

---

# 6. Tenant Identity

KAMPYN may serve multiple universities or organizations.

Authentication must establish not only:

```text
Who is the user?
```

but, where applicable:

```text
Which institution does this identity belong to?
```

Tenant membership must be resolved server-side.

Do not trust a client-provided tenant ID as proof of membership.

The authenticated context should conceptually contain:

```text id="7l7t8x"
User Identity
    +
Tenant Membership
    +
Authentication Method
    +
Session Context
```

---

# 7. Authentication Providers

KAMPYN may support multiple authentication mechanisms.

Potential mechanisms include:

- Email/password.
- Institution-managed identity providers.
- OAuth/OIDC providers.
- Enterprise SSO.
- University identity systems.
- Service-to-service authentication.

Each provider must map its external identity to a stable internal KAMPYN identity.

Do not allow provider-specific identifiers to leak throughout the domain.

Prefer:

```text id="d6uv6m"
External Provider
      ↓
Provider Adapter
      ↓
Internal Identity
```

---

# 8. OAuth and OpenID Connect

When OAuth/OIDC is used, follow the provider's protocol requirements.

The application must validate:

- Issuer.
- Audience.
- Signature.
- Expiration.
- Nonce where applicable.
- State where applicable.
- Redirect URI.
- Provider subject.

Never trust claims merely because they were supplied by the browser.

Provider configuration must be explicit.

---

# 9. External Identity Linking

An external identity should map to an internal user through a controlled identity-linking mechanism.

Conceptually:

```text
Provider
  ↓
issuer + subject
  ↓
External Identity Record
  ↓
Internal User
```

The combination of provider identity attributes must be unique.

Do not identify users solely by email when the provider supplies a stable subject identifier.

---

# 10. Password Authentication

When password authentication is supported:

- Passwords must never be stored in plaintext.
- Passwords must use an appropriate password hashing algorithm.
- Password hashes must never be returned through APIs.
- Password verification must occur server-side.
- Password requirements must be documented.
- Password reset must use a separate secure flow.

Use a modern password hashing algorithm appropriate for password storage.

Do not invent custom password hashing schemes.

---

# 11. Password Requirements

Password requirements should provide meaningful protection without unnecessary complexity.

Consider:

- Minimum length.
- Password reuse.
- Common/compromised passwords.
- Rate limiting.
- Credential stuffing protection.

Do not rely exclusively on arbitrary complexity requirements such as requiring a specific combination of character classes.

Long, unique passwords are generally preferable.

---

# 12. Credential Stuffing Protection

Authentication endpoints are high-value abuse targets.

Protect:

- Login.
- Password reset.
- Verification.
- OTP verification.
- Account recovery.

Use appropriate:

- Rate limiting.
- Progressive delays.
- Abuse detection.
- Temporary restrictions.
- Monitoring.

Do not reveal whether a particular account exists when doing so would enable account enumeration.

---

# 13. Account Enumeration

Authentication responses should avoid unnecessarily revealing account existence.

For sensitive flows such as:

```text
Forgot password
Account recovery
Email verification
```

prefer responses that do not reveal whether an account exists.

Do not create different user-visible behavior solely based on whether an email exists unless the security model explicitly permits it.

---

# 14. Session Architecture

Authenticated sessions must have:

- A clear owner.
- An expiration policy.
- Revocation capability.
- Secure storage.
- Appropriate rotation behavior.
- Device/session visibility where required.

The exact implementation may use secure server-side sessions or appropriately designed tokens.

The architecture must define which model is authoritative.

Do not mix session and token semantics inconsistently.

---

# 15. Cookies

When cookies are used for authentication, configure them securely.

Authentication cookies should generally use:

```text id="d5h6d1"
HttpOnly
Secure
SameSite
```

with values appropriate to the deployment architecture.

Do not store sensitive authentication credentials in ordinary JavaScript-accessible storage when secure cookies can provide the required security model.

---

# 16. CSRF Protection

Cookie-based authentication must account for CSRF.

Use appropriate:

- SameSite policies.
- CSRF tokens where required.
- Origin checks where appropriate.
- Server-side request validation.

Do not assume that `HttpOnly` alone prevents CSRF.

`HttpOnly` protects cookie access from JavaScript; it does not by itself prevent cross-site requests.

---

# 17. Token Architecture

If token-based authentication is used, tokens must have clearly defined purposes.

Distinguish between:

- Access tokens.
- Refresh tokens.
- Verification tokens.
- Password-reset tokens.
- API keys.
- Service credentials.

Do not use one token type for every authentication workflow.

---

# 18. Access Tokens

Access tokens should:

- Have limited lifetime.
- Contain only necessary claims.
- Be audience-bound where appropriate.
- Be issuer-bound.
- Be validated server-side.
- Not contain unnecessary sensitive information.

Do not put secrets or large mutable datasets into access tokens.

---

# 19. Refresh Tokens

Refresh tokens provide longer-lived authentication and therefore require stronger protection.

Where refresh tokens are used:

- Store or handle them securely.
- Use rotation where appropriate.
- Detect reuse where supported.
- Provide revocation.
- Associate them with a session/device where appropriate.
- Limit lifetime.

A refresh token must not become an unbounded permanent credential.

---

# 20. Token Rotation

Token rotation should be used where appropriate to reduce replay risk.

A typical flow:

```text id="yq5vcp"
Refresh Token A
      ↓
Refresh
      ↓
Access Token B
+
Refresh Token C
      ↓
Invalidate A
```

If token reuse is detected, revoke the affected session/token family according to the security architecture.

---

# 21. Session Revocation

Users must be able to invalidate authentication sessions where appropriate.

Revocation should be possible for:

- Logout.
- Password change.
- Security compromise.
- Administrative action.
- Account suspension.

Do not assume deleting a browser cookie automatically invalidates server-side credentials.

---

# 22. Logout

Logout should invalidate the relevant authentication state.

For cookie sessions:

```text
Client logout
    ↓
Server session invalidation
    ↓
Cookie expiration
```

For refresh-token systems:

```text
Client logout
    ↓
Refresh token/session revocation
    ↓
Credential removal
```

Access tokens that are already issued may remain valid until expiration unless the architecture supports immediate revocation.

The behavior must be documented.

---

# 23. Session Management

Where session records are stored, they should contain enough information to manage security without storing unnecessary sensitive data.

Potential metadata includes:

- Session ID.
- User ID.
- Tenant.
- Created time.
- Last-used time.
- Expiration.
- Revocation status.
- Device metadata where appropriate.
- Security event metadata.

Do not store raw authentication secrets unnecessarily.

---

# 24. Multiple Devices

A user may have multiple active sessions.

Sessions should be independently identifiable where practical.

The architecture may support:

```text id="h6n5jj"
User
 ├── Laptop Session
 ├── Mobile Session
 └── Tablet Session
```

Revoking one session should not unintentionally revoke unrelated sessions unless required by the security policy.

Security-sensitive events may revoke all sessions.

---

# 25. Account States

Authentication must distinguish relevant account states.

Examples:

```text id="vxtw2y"
Pending
Active
Suspended
Disabled
Locked
Deleted
```

The exact states must be defined by the domain.

Authentication must not allow disabled or suspended accounts to obtain new authenticated sessions.

---

# 26. Email Verification

If email verification is required:

```text id="q7e6m4"
Account Creation
    ↓
Verification Token
    ↓
Email
    ↓
Token Validation
    ↓
Verified Identity
```

Verification tokens must:

- Be unpredictable.
- Expire.
- Be single-use where appropriate.
- Be invalidated after successful use.
- Not expose sensitive information.

Do not treat a user-controlled email field as verified merely because it has a valid email format.

---

# 27. Password Reset

Password reset must be a separate security workflow.

Typical flow:

```text id="m7r4ac"
Reset Request
    ↓
Generic Response
    ↓
Secure Reset Token
    ↓
User Opens Reset Link
    ↓
Token Validation
    ↓
New Password
    ↓
Password Update
    ↓
Session Invalidation
```

Reset tokens must:

- Be unpredictable.
- Expire quickly.
- Be single-use.
- Be stored securely or represented by a secure hash.
- Be invalidated after use.

Do not send passwords through email.

---

# 28. Account Recovery

Account recovery is a security-sensitive operation.

Recovery mechanisms must not provide a weaker authentication path than the account they protect.

Consider:

- Identity verification.
- Recovery factors.
- Existing sessions.
- Recovery token lifetime.
- Rate limiting.
- Audit logging.

Do not add recovery mechanisms merely for convenience without considering account takeover risk.

---

# 29. OTP and Verification Codes

If OTPs are used:

- Generate them securely.
- Limit attempts.
- Expire them quickly.
- Prevent unlimited resend requests.
- Prevent brute-force verification.
- Invalidate after successful use.

Do not log OTP values.

Do not use predictable counters or timestamps as security codes.

---

# 30. Authentication Context

After successful authentication, create a trusted server-side identity context.

Conceptually:

```text id="qklz10"
Authentication
    ↓
AuthenticatedPrincipal
    ├── User ID
    ├── Tenant ID / memberships
    ├── Authentication method
    └── Session metadata
```

Downstream services should consume this trusted context rather than repeatedly parsing raw credentials.

---

# 31. Authentication Middleware

Authentication middleware should be responsible for:

- Extracting credentials.
- Validating credentials.
- Resolving identity.
- Establishing request context.
- Rejecting invalid authentication.

It should not contain:

- Business workflows.
- Resource-specific authorization logic.
- Database mutations unrelated to authentication.

---

# 32. Authorization Boundary

After authentication:

```text id="rrx6gb"
Authentication Middleware
        ↓
Identity Context
        ↓
Authorization Policy
        ↓
Application Service
```

Resource authorization belongs near the application/domain operation where the resource and action are known.

Do not place every authorization rule inside generic authentication middleware.

---

# 33. Tenant Resolution

Tenant resolution may come from:

- Authenticated identity.
- Session.
- Trusted host/domain mapping.
- Explicit institution selection validated against membership.

Never trust an arbitrary:

```text
X-Tenant-ID
```

header as authorization.

If a client supplies tenant context, it must be validated against the authenticated identity.

---

# 34. Administrative Authentication

Administrative accounts require stronger protection.

Consider:

- Strong authentication.
- MFA.
- Shorter session lifetimes.
- Reauthentication for sensitive actions.
- Detailed audit logging.
- Separate administrative roles.
- Session visibility.

Do not treat administrative users as ordinary users with a boolean `isAdmin` flag when the authorization model is more complex.

---

# 35. Multi-Factor Authentication

MFA should be supported where the deployment/security requirements justify it.

Potential factors include:

- Authenticator applications.
- Hardware security keys.
- Institution-managed authentication.
- Other approved strong authentication mechanisms.

MFA state must be enforced server-side.

A frontend-controlled:

```text
mfaCompleted = true
```

must never be trusted.

---

# 36. Reauthentication

Sensitive operations may require recent authentication.

Examples:

- Changing password.
- Changing MFA.
- Removing security factors.
- Accessing highly sensitive data.
- Financially sensitive actions.
- Changing administrative privileges.

The backend must determine whether reauthentication is required.

---

# 37. Service Authentication

Service-to-service authentication must be separate from human-user authentication where appropriate.

Service credentials should:

- Have explicit identity.
- Have limited permissions.
- Be rotated.
- Be stored securely.
- Be auditable.

Do not reuse a human user's credentials for backend services.

---

# 38. API Keys

API keys may be used for institution integrations or server-to-server consumers where appropriate.

API keys must:

- Have an explicit owner.
- Have limited scope.
- Be revocable.
- Be rotated.
- Be stored securely.
- Be auditable.
- Never be exposed to browser clients when they grant privileged access.

Store hashes of API keys where practical rather than plaintext secrets.

---

# 39. Self-Hosted Authentication

KAMPYN may be self-hosted by universities.

Authentication architecture must therefore avoid undocumented dependence on EXSOLVIA infrastructure.

A self-hosted deployment should be able to configure:

- Authentication providers.
- Institution domains.
- OAuth/OIDC settings.
- Session configuration.
- Email provider.
- Token settings.
- MFA configuration where supported.

Configuration must be explicit and validated during startup.

---

# 40. Institution SSO

University deployments may use institution-managed SSO.

The integration should follow:

```text id="11hm96"
University IdP
      ↓
OIDC / SAML Adapter
      ↓
KAMPYN Identity Mapping
      ↓
KAMPYN User
      ↓
Tenant Membership
```

Provider-specific implementation must remain behind an authentication adapter.

Do not spread provider-specific claims throughout business code.

---

# 41. Domain-Based Identity

Institution domains may assist with identity routing.

For example:

```text
student@university.edu
```

may indicate an institution.

However, email domain alone must not be treated as sufficient authorization.

Verify the identity through the configured authentication provider and institution membership.

---

# 42. Authentication Events

Important authentication events should be auditable.

Examples:

- Login success.
- Login failure where useful.
- Logout.
- Password change.
- Password reset.
- MFA changes.
- Session creation.
- Session revocation.
- Account suspension.
- Provider linking/unlinking.
- API key creation/revocation.

Do not log credentials or tokens.

---

# 43. Brute-Force Protection

Authentication endpoints must have abuse controls.

Consider:

- Per-account limits.
- Per-IP limits.
- Per-device/session signals.
- Progressive delays.
- Temporary restrictions.
- CAPTCHA or equivalent mechanisms where justified.

Do not rely on a single IP-based limit.

---

# 44. Credential Security

Never:

- Log passwords.
- Log access tokens.
- Log refresh tokens.
- Return password hashes.
- Store plaintext passwords.
- Include secrets in URLs.
- Commit authentication secrets.
- Send credentials to analytics systems.

Credentials must remain within the smallest possible trust boundary.

---

# 45. Redirect Security

Authentication redirects must validate destination URLs.

Never blindly redirect to a user-supplied URL after login.

Use an allowlist or controlled relative paths where appropriate.

Protect against:

- Open redirects.
- Malicious callback URLs.
- OAuth redirect manipulation.

---

# 46. Browser Security

Authentication-related browser behavior must account for:

- CSRF.
- XSS.
- Clickjacking where applicable.
- Secure cookies.
- SameSite behavior.
- HTTPS.
- Origin validation.

Do not treat frontend JavaScript as a trusted security boundary.

---

# 47. Transport Security

Authentication credentials must be transmitted over encrypted transport.

Production authentication endpoints must require HTTPS.

Do not send:

- Passwords.
- Tokens.
- Session identifiers.

over unencrypted HTTP in production.

Secure internal service communication according to the deployment's trust model.

---

# 48. Token Leakage

Tokens must not appear in:

- URLs.
- Query parameters.
- Logs.
- Analytics.
- Error messages.
- Browser history.

Prefer secure headers or cookies according to the chosen authentication model.

---

# 49. Authentication Error Handling

Authentication errors should reveal only what is necessary.

Avoid responses such as:

```text id="0x6gyn"
Email exists but password is incorrect.
```

when this would enable account enumeration.

Prefer consistent external responses while retaining detailed internal observability.

---

# 50. Clock and Expiration Handling

Authentication systems depend on time.

Account for:

- Clock skew.
- Token expiration.
- Session expiration.
- Verification expiration.
- Reset-token expiration.

Use server-controlled time.

Do not rely on browser time to determine whether credentials are valid.

---

# 51. Authentication Testing

Authentication must be tested across:

### Login
- Valid credentials.
- Invalid credentials.
- Disabled account.
- Suspended account.
- Rate limiting.

### Sessions
- Creation.
- Expiration.
- Revocation.
- Logout.
- Multiple sessions.

### Tokens
- Expiration.
- Invalid signature.
- Wrong issuer.
- Wrong audience.
- Replay where applicable.

### Recovery
- Password reset.
- Verification.
- OTP.
- Account recovery.

### Authorization boundary
- Correct identity context.
- Wrong tenant.
- Unauthorized resource.
- Privilege escalation attempts.

### Security
- CSRF.
- Enumeration.
- Brute force.
- Token leakage.
- Redirect manipulation.

---

# 52. Authentication Change Checklist

Before completing an authentication-related change:

- [ ] Authentication and authorization remain separate.
- [ ] Identity has a stable internal ID.
- [ ] Credentials are securely handled.
- [ ] Sessions/tokens have defined lifetimes.
- [ ] Revocation behavior is defined.
- [ ] Authentication cookies are secure where used.
- [ ] CSRF protection is appropriate.
- [ ] Account enumeration is considered.
- [ ] Rate limiting is applied where required.
- [ ] Password reset is secure.
- [ ] Verification tokens are secure.
- [ ] MFA behavior is server-enforced where applicable.
- [ ] Tenant resolution is trusted and validated.
- [ ] Provider-specific logic is isolated.
- [ ] Administrative authentication is appropriately protected.
- [ ] Service credentials are separate from human credentials.
- [ ] Secrets are not logged or committed.
- [ ] Authentication events are auditable where required.
- [ ] Self-hosted configuration is documented.
- [ ] Authentication tests cover failure paths.
- [ ] Documentation reflects the current authentication flow.

---

# 53. Final Authentication Principle

Authentication is the foundation of KAMPYN's trust boundary.

The architecture should establish:

```text
Credential
    ↓
Verified Identity
    ↓
Trusted Authentication Context
    ↓
Tenant Membership
    ↓
Authorization
    ↓
Resource Access
```

Never trust identity information merely because it came from the client.

Never treat authentication as authorization.

Never make credential recovery weaker than the account it protects.

Prefer short-lived, revocable credentials, explicit identity boundaries, secure defaults, and auditable authentication flows.

The authentication system should make secure identity handling the default path rather than an optional concern.
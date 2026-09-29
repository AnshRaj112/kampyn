# Payment Security

## 1. Purpose

This document defines the mandatory security standards for designing, implementing, processing, storing, transmitting, and monitoring payments across KAMPYN.

KAMPYN supports transactions involving students, faculty, vendors, food courts, university departments, and other campus service providers. Payment workflows may include food orders, bookings, refunds, cancellations, vendor settlements, commissions, and transaction reconciliation.

Payment processing is a security-critical component. Any weakness can result in financial loss, unauthorized transactions, duplicate charges, fraudulent refunds, data exposure, or loss of transaction integrity.

The objective is to ensure that every payment is authenticated, authorized, verified, recorded, traceable, and reconciled while minimizing KAMPYN's exposure to sensitive financial data.

## 2. Core Principles

All payment-related functionality MUST follow these principles:

- **Never trust the client:** Payment amounts, statuses, provider responses, and transaction references supplied by the frontend MUST NOT be treated as authoritative.
- **Use trusted payment providers:** Delegate sensitive payment instrument handling to approved payment gateways wherever possible.
- **Verify every transaction:** Payment success MUST be established through verified provider communication and authoritative server-side checks.
- **Prevent duplicate processing:** All payment operations MUST be designed for idempotency and safe retries.
- **Preserve transaction integrity:** Financial state changes MUST be atomic where possible and recoverable where distributed transactions are involved.
- **Enforce authorization:** Users may only access and initiate payment operations they are authorized to perform.
- **Maintain auditability:** Every financial state transition MUST be traceable to its source, actor, and associated transaction.
- **Minimize sensitive data:** KAMPYN MUST NOT collect or store payment instrument data unless explicitly required and approved.
- **Secure provider communication:** All payment-provider communication MUST use authenticated, encrypted channels.
- **Reconcile independently:** Internal financial records MUST be periodically reconciled against payment-provider records.
- **Fail securely:** Uncertain payment outcomes MUST NOT be treated as successful or automatically charged again.
- **Separate financial concerns:** Payment processing, order fulfillment, refunds, and vendor settlement MUST have clearly defined responsibilities.

Payment status MUST be based on verified financial evidence, not on the user's browser state or an unverified callback.

## 3. Scope

This policy applies to:

- Online food ordering and checkout.
- Hostel, guest-house, and facility bookings.
- Shuttle bookings and other paid campus services.
- Payment initiation and authorization.
- Payment capture and confirmation.
- Payment provider webhooks and callbacks.
- Refunds, reversals, and cancellations.
- Vendor payouts and settlements.
- Platform commissions and service fees.
- Discounts, coupons, and promotional adjustments.
- Payment reconciliation and financial reporting.
- Payment-related APIs and SDKs.
- Background payment jobs and scheduled reconciliation tasks.
- Administrative financial operations.
- Payment integrations and self-hosted university deployments.
- Payment-related databases, queues, logs, caches, and analytics.

The policy applies to every environment where real payment credentials, financial transactions, or production payment-provider accounts are involved.

## 4. Payment Architecture

### 4.1 Architectural Requirements

KAMPYN MUST maintain a clear separation between:

- **Frontend:** Displays checkout, payment instructions, transaction status, and receipts.
- **Backend payment service:** Authoritatively manages payment intents, provider communication, transaction states, refunds, and reconciliation.
- **Payment provider:** Processes payment instruments and provides authoritative payment transaction evidence.
- **Order or booking service:** Manages the domain operation associated with a payment.
- **Ledger or financial records:** Maintains an auditable record of financial movements.
- **Webhook processor:** Receives, authenticates, validates, and processes provider events.
- **Reconciliation worker:** Compares KAMPYN records with provider transaction records.
- **Administrative interface:** Provides authorized financial oversight and operational controls.

Payment functionality SHOULD be isolated behind a dedicated payment module or service interface, even when deployed within a modular monolith.

A separate microservice MUST NOT be introduced solely for the sake of separation if a well-defined module within the backend satisfies the architectural requirements.

### 4.2 High-Level Payment Flow

```text
Student / User
      |
      v
KAMPYN Frontend
      |
      | Create payment request
      v
KAMPYN Backend
      |
      | Validate identity, tenant, order,
      | amount and payment eligibility
      v
Payment Service
      |
      | Create provider transaction
      v
Payment Gateway
      |
      | Hosted checkout / secure payment UI
      v
Student / User
      |
      | Completes payment
      v
Payment Gateway
      |
      | Signed webhook / provider event
      v
Webhook Processor
      |
      | Verify authenticity and transaction
      v
Payment Service
      |
      | Persist verified state
      v
Order / Booking Service
      |
      | Confirm eligible domain operation
      v
Database / Financial Records
      |
      v
Reconciliation and Audit
```

The frontend MAY receive immediate payment status information from the provider's checkout interface, but it MUST NOT directly mark an order or booking as paid.

### 4.3 Payment Provider Integration

KAMPYN SHOULD use an established payment gateway with appropriate security controls, documentation, compliance capabilities, and operational support.

The provider integration MUST:

- Use official, maintained server-side SDKs or secure HTTPS APIs.
- Verify the provider's current API and webhook specifications.
- Use separate credentials for test and production environments.
- Support secure transaction creation and status verification.
- Support idempotency or an equivalent duplicate-prevention mechanism where available.
- Support authenticated webhook verification.
- Support transaction lookup and reconciliation.
- Define timeout, retry, and failure behavior.
- Document the provider's transaction lifecycle and status semantics.

Payment-provider functionality MUST be wrapped in a KAMPYN-owned adapter or interface so that provider-specific details do not leak throughout the domain layer.

Example interface:

```go
type PaymentProvider interface {
    CreatePayment(
        ctx context.Context,
        request CreatePaymentRequest,
    ) (ProviderPayment, error)

    GetPayment(
        ctx context.Context,
        providerPaymentID string,
    ) (ProviderPayment, error)

    CreateRefund(
        ctx context.Context,
        request CreateRefundRequest,
    ) (ProviderRefund, error)

    VerifyWebhook(
        payload []byte,
        headers map[string]string,
    ) (ProviderEvent, error)
}
```

This is an illustrative interface. Actual types and methods MUST reflect the selected provider's supported capabilities and KAMPYN's domain requirements.

## 5. Payment Data Classification

Payment-related data MUST be classified according to sensitivity.

| Data type | Classification | Handling |
|---|---|---|
| Payment instrument details | Highly sensitive | Avoid collecting or storing |
| Card number (PAN) | Highly sensitive | Do not store unless formally approved and compliant |
| CVV / CVC | Highly sensitive | Never store after authorization |
| PIN or OTP | Highly sensitive | Never store or log |
| Provider secret keys | Critical secret | Secret manager only |
| Webhook signing secrets | Critical secret | Secret manager only |
| Provider transaction ID | Sensitive financial metadata | Access-controlled storage |
| Internal payment ID | Sensitive financial metadata | Access-controlled storage |
| Payment amount and currency | Financial data | Integrity-protected records |
| Refund and settlement records | Financial data | Access-controlled, auditable storage |
| Payment status | Financial data | Server-authoritative state |
| Masked payment method | Limited sensitive data | Store only when required |
| Payment timestamps | Financial metadata | Controlled retention |
| Reconciliation reports | Confidential financial data | Restricted access |
| Public receipt reference | Limited sensitive data | Avoid exposing internal identifiers |

KAMPYN MUST NOT store raw card numbers, CVV/CVC values, PINs, OTPs, or payment authentication secrets in application databases, logs, analytics systems, caches, or event payloads.

If a business requirement introduces the handling of payment card data, it MUST undergo formal security, legal, and compliance review before implementation.

## 6. Payment Instrument Security

### 6.1 Hosted Checkout

KAMPYN SHOULD prefer provider-hosted checkout pages, hosted payment fields, or other provider-approved methods that minimize the exposure of payment instrument data to KAMPYN infrastructure.

Where hosted checkout is used:

- Redirect destinations MUST be obtained from trusted provider responses.
- Checkout sessions MUST be associated with a server-created payment record.
- Session expiry MUST be enforced.
- Payment completion MUST be verified independently.
- Return URLs MUST NOT determine payment success.
- Checkout session identifiers MUST be handled as sensitive values.
- Redirect and callback flows MUST be protected against tampering.

### 6.2 Payment Tokenization

Where supported, provider-issued tokens MAY be used instead of storing payment instrument data.

Tokens MUST:

- Be obtained through an approved provider flow.
- Be scoped to the intended user, merchant, or payment context where supported.
- Be stored only when required.
- Be protected from unauthorized access.
- Never be treated as authorization to perform unrelated financial actions.
- Be revoked or deleted when no longer required, where supported.

Provider tokens MUST NOT be assumed to be harmless simply because they do not contain raw card data.

### 6.3 Payment Authentication

Payment authentication requirements, such as additional customer verification or regulatory authentication, MUST be handled through provider-supported flows.

KAMPYN MUST NOT:

- Bypass provider-required authentication.
- Store authentication codes or one-time passwords.
- Expose provider authentication secrets to unrelated services.
- Treat an initiated authentication flow as completed payment.

The backend MUST verify the final transaction outcome independently.

## 7. Payment Initiation

### 7.1 Server-Side Payment Creation

All payment transactions MUST be initiated by the backend.

Before creating a payment, the backend MUST verify:

- The user is authenticated.
- The user is authorized to pay for the specified order or booking.
- The resource belongs to the active tenant.
- The order or booking exists.
- The resource is in a payable state.
- The order or booking has not already been paid.
- The payment amount is calculated from authoritative server-side data.
- The currency is supported.
- Discounts, fees, taxes, and commissions are calculated according to the approved pricing rules.
- The request is within the permitted payment workflow.

The backend MUST NOT accept a client-provided amount as the authoritative payment value.

### 7.2 Amount Calculation

Payment amounts MUST be calculated from trusted domain data.

For an order, the amount MAY include:

- Item prices.
- Approved item customizations.
- Applicable taxes.
- Delivery or service charges.
- Valid discounts.
- Platform fees, where applicable.
- Other documented adjustments.

The backend MUST calculate the final payable amount using a defined, auditable pricing process.

Use integer minor units or an exact decimal representation for financial calculations.

Floating-point arithmetic MUST NOT be used for authoritative financial calculations.

The amount submitted to the payment provider MUST match the authoritative amount stored in KAMPYN's payment record.

### 7.3 Price Changes

If prices, item availability, taxes, or discounts change between order creation and payment initiation, KAMPYN MUST apply a documented pricing policy.

Possible policies include:

- Recalculate the amount and request user confirmation.
- Honor a previously locked quote until its expiry.
- Reject the payment attempt and require checkout to be refreshed.

The policy MUST be consistent, documented, and enforced server-side.

### 7.4 Payment Intent

Every payment attempt MUST be associated with a unique internal payment identifier.

A payment record SHOULD include:

- Internal payment ID.
- Tenant ID.
- User or initiating principal ID.
- Related order or booking ID.
- Provider name.
- Provider payment ID, when available.
- Amount and currency.
- Payment status.
- Idempotency key or request reference.
- Creation and update timestamps.
- Provider verification details, where appropriate.
- Correlation and audit references.

Payment records MUST NOT contain raw payment credentials.

### 7.5 Idempotent Initiation

Payment initiation MUST be idempotent.

For each logical payment request:

- Generate or accept a scoped idempotency key.
- Associate the key with the authenticated principal, tenant, and intended resource.
- Store the original operation outcome.
- Return the existing operation result for a legitimate retry.
- Reject reuse of the same key with conflicting request parameters.
- Prevent concurrent duplicate payment creation for the same logical attempt.

Where the provider supports idempotency, KAMPYN MUST use the provider's mechanism as an additional safeguard.

Idempotency keys MUST have defined uniqueness, retention, and expiry behavior.

## 8. Payment State Management

### 8.1 State Machine

Payment states MUST be explicitly defined and enforced.

A baseline state model MAY include:

| State | Meaning |
|---|---|
| `CREATED` | Internal payment record created |
| `PENDING` | Provider transaction initiated, outcome not final |
| `REQUIRES_ACTION` | Additional customer action required |
| `AUTHORIZED` | Provider confirms authorization, where supported |
| `CAPTURE_PENDING` | Capture has been requested but is not confirmed |
| `SUCCEEDED` | Payment is verified as successfully completed |
| `FAILED` | Payment is confirmed as failed |
| `CANCEL_PENDING` | Cancellation has been requested but is not confirmed |
| `CANCELLED` | Payment cancellation is verified |
| `REFUND_PENDING` | Refund requested but not yet confirmed |
| `PARTIALLY_REFUNDED` | Some amount has been refunded |
| `REFUNDED` | Full refund is verified |
| `DISPUTED` | Payment is under dispute or chargeback review |
| `REVIEW_REQUIRED` | Conflicting or uncertain transaction evidence requires investigation |

The final state model MUST be adapted to the selected provider's actual transaction lifecycle.

### 8.2 Valid State Transitions

State transitions MUST be explicitly controlled.

- Only permitted transitions may be applied.
- Terminal states MUST NOT be overwritten by stale events.
- Repeated events MUST be idempotently processed.
- Invalid or conflicting transitions MUST be rejected or routed for investigation.
- State transitions MUST be auditable.
- The transition MUST be tied to verified provider evidence or an authorized internal operation.

A payment MUST NOT transition to `SUCCEEDED` based only on a frontend redirect, client response, or unverified webhook.

### 8.3 Uncertain Outcomes

When a provider request times out or the response is ambiguous:

- Do not assume that the payment failed.
- Do not automatically create a new payment attempt without checking the existing attempt.
- Query the provider using the known transaction reference when supported.
- Retain the payment in a pending or review state until the outcome is established.
- Reconcile uncertain transactions through controlled background processing.

An uncertain transaction MUST NOT trigger duplicate charges or duplicate order fulfillment.

## 9. Provider Webhook Security

### 9.1 Webhook Authenticity

Webhooks MUST be verified before they can affect payment state.

KAMPYN MUST:

- Use the provider's documented signature-verification mechanism.
- Verify signatures against the exact payload representation required by the provider.
- Use constant-time comparison where applicable.
- Protect signing secrets.
- Reject invalid or missing signatures.
- Enforce supported timestamp or replay-protection checks.
- Validate provider account, merchant, and environment context.
- Avoid trusting event metadata until authenticity is established.

Webhook verification MUST use the provider's official security specification.

### 9.2 Webhook Payload Validation

After authenticity verification, webhook payloads MUST be validated against an explicit schema.

Validation MUST include:

- Event type.
- Event identifier.
- Provider payment identifier.
- Merchant or account context.
- Amount and currency, where provided.
- Payment status.
- Relevant timestamps.
- Required related references.
- Supported schema version, where applicable.

Unexpected or malformed events MUST NOT change financial state.

### 9.3 Replay and Duplicate Protection

Webhook processing MUST be idempotent.

- Store provider event identifiers or an equivalent deduplication reference.
- Prevent duplicate processing of the same event.
- Reject replayed events where provider semantics allow reliable replay detection.
- Ensure repeated delivery does not trigger repeated refunds, captures, order confirmations, or settlement operations.
- Define a retention period for deduplication records based on provider retry and dispute windows.

### 9.4 Event Ordering

Webhook events may be delivered more than once or out of order.

KAMPYN MUST NOT assume that event arrival order matches transaction chronology.

When events conflict:

- Verify the current payment state through the provider API where appropriate.
- Apply only valid state transitions.
- Preserve the event history.
- Route unresolved conflicts to `REVIEW_REQUIRED`.
- Avoid regressing verified terminal states based on stale events.

### 9.5 Webhook Processing Flow

```text
Provider Webhook
      |
      v
Enforce Request Size and Method
      |
      v
Verify Provider Signature
      |
      v
Validate Event Schema
      |
      v
Check Provider / Merchant Context
      |
      v
Check Event Deduplication
      |
      v
Resolve Internal Payment
      |
      v
Verify Amount, Currency and References
      |
      v
Validate State Transition
      |
      v
Persist Payment State and Event Record
      |
      v
Trigger Idempotent Domain Action
      |
      v
Return Provider-Compatible Acknowledgement
```

The implementation MUST follow the selected provider's acknowledgement and retry requirements.

### 9.6 Webhook Availability

Webhook endpoints MUST:

- Be exposed only through secure HTTPS.
- Enforce request-size and method restrictions.
- Use appropriate timeouts.
- Avoid performing slow external operations synchronously when the provider expects rapid acknowledgement.
- Persist accepted events reliably before acknowledging them, where required by the processing model.
- Support safe retries and recovery.
- Be monitored for failures, invalid signatures, and processing backlogs.

Webhook processing MUST NOT be considered successful merely because an HTTP request was received.

## 10. Payment Confirmation and Order Fulfillment

### 10.1 Verified Confirmation

An order or booking MUST be marked as paid only after KAMPYN has verified the payment outcome using trusted provider evidence.

Verification MAY use:

- A valid signed webhook.
- An authenticated provider status query.
- Another documented provider verification mechanism.

For high-risk or ambiguous transactions, KAMPYN SHOULD independently query the provider before applying final financial state.

### 10.2 Domain State Consistency

Payment confirmation and domain fulfillment MUST be coordinated safely.

Examples include:

- Confirming a food order.
- Confirming a hostel booking.
- Reserving a guest-house room.
- Confirming a shuttle booking.
- Unlocking a paid service.

The system MUST ensure that:

- An order is not fulfilled twice due to duplicate payment events.
- A payment cannot be associated with an unrelated order or booking.
- A successful payment does not bypass domain eligibility checks.
- Failed or cancelled payments do not accidentally confirm unpaid resources.
- Inventory and booking capacity are updated according to the domain's consistency model.

Where payment state and order state are stored in separate services or datastores, use idempotent event processing, an outbox pattern, or a documented saga to manage eventual consistency.

### 10.3 Reservation and Inventory Coordination

For limited-capacity resources, KAMPYN MUST define how payment delays affect reservations and inventory.

The system SHOULD:

- Use explicit reservation holds with expiry.
- Revalidate availability before final confirmation.
- Define what happens if payment succeeds after a hold expires.
- Prevent overselling through concurrency-safe domain operations.
- Provide a controlled recovery process for paid but unfulfilled transactions.

A payment success MUST NOT silently override a capacity conflict or inventory constraint.

## 11. Refund Security

### 11.1 Refund Authorization

Refunds MUST be initiated only through authorized backend operations.

Before processing a refund, KAMPYN MUST verify:

- The principal has the required refund permission.
- The payment exists and belongs to the relevant tenant.
- The payment is eligible for a refund.
- The requested amount does not exceed the refundable balance.
- The refund reason is documented where required.
- The refund does not conflict with another pending refund.
- Any required approval or separation-of-duties rule is satisfied.

Users MUST NOT be able to set a payment's status to refunded through ordinary order or booking update APIs.

### 11.2 Refund Limits

The backend MUST calculate the refundable amount from authoritative payment and refund records.

For partial refunds:

- Track the original payment amount.
- Track all successful refunds.
- Track pending refunds to prevent concurrent over-refunding.
- Enforce the remaining refundable balance.
- Maintain currency consistency.
- Preserve an audit trail for each refund.

Refund calculations MUST use exact financial representations.

### 11.3 Refund Idempotency

Refund requests MUST be idempotent.

- Each logical refund MUST have a unique internal reference.
- Retries MUST return the existing refund operation where appropriate.
- Conflicting requests using the same idempotency key MUST be rejected.
- Provider idempotency MUST be used where supported.
- Duplicate webhook events MUST NOT create duplicate refund records.

### 11.4 Refund State

Refunds SHOULD be represented as separate financial records linked to the original payment.

A refund MUST NOT be marked as completed until verified provider evidence confirms the refund outcome.

The payment's overall status MUST be derived or updated consistently from the original payment and its refund records.

### 11.5 Administrative Refunds

Administrative refund operations MUST:

- Require explicit permissions.
- Require stronger authentication for sensitive actions where appropriate.
- Enforce approval thresholds where configured.
- Record the initiating administrator and approver.
- Record the amount, currency, reason, and related transaction.
- Support investigation and reconciliation.
- Prevent self-approval where separation of duties is required.

## 12. Cancellations and Payment Reversals

Cancellation, void, reversal, and refund are distinct financial operations and MUST be modeled separately.

KAMPYN MUST:

- Follow provider-specific transaction semantics.
- Validate the payment's current state before requesting a cancellation or reversal.
- Avoid treating a cancellation request as a confirmed cancellation.
- Verify the final provider outcome.
- Handle late success events and conflicting responses.
- Maintain a traceable operation record.
- Define how domain resources are released or retained during pending cancellation.

A cancelled order or booking MUST NOT automatically imply that the associated payment has been reversed or refunded.

## 13. Vendor Settlements and Platform Commissions

### 13.1 Settlement Integrity

Vendor settlements and platform commissions MUST be calculated from authoritative financial records.

Settlement logic MUST define:

- Eligible transactions.
- Commission rules.
- Tax and fee treatment.
- Refund and chargeback adjustments.
- Settlement periods.
- Currency handling.
- Payout status.
- Reconciliation requirements.

Vendor-facing interfaces MUST NOT be allowed to directly modify authoritative settlement balances.

### 13.2 Settlement Records

Each settlement SHOULD contain:

- Internal settlement ID.
- Tenant ID.
- Vendor or beneficiary ID.
- Settlement period.
- Currency.
- Gross eligible amount.
- Refund and adjustment totals.
- Commission and fees.
- Net payable amount.
- Related payment and refund references.
- Provider payout reference, where applicable.
- Status and timestamps.
- Initiator and approver, where applicable.

Settlement records MUST be auditable and reproducible from the underlying financial activity.

### 13.3 Payout Authorization

Payout operations MUST enforce:

- Authorized initiators.
- Beneficiary verification according to provider requirements.
- Approval thresholds.
- Idempotency.
- Provider status verification.
- Separation of duties where appropriate.
- Audit logging and reconciliation.

Payout destination changes MUST receive additional authorization and verification controls.

### 13.4 Settlement Reconciliation

KAMPYN MUST reconcile settlement records with provider payout reports and internal financial records.

Unmatched or inconsistent settlements MUST be flagged for investigation and MUST NOT be silently adjusted to force agreement.

## 14. Financial Ledger and Audit Trail

### 14.1 Financial Records

KAMPYN MUST maintain durable records of payment-related financial operations.

A payment record SHOULD be distinct from the associated order, booking, or service record.

Where financial reporting, vendor settlement, or accounting requirements justify it, KAMPYN SHOULD maintain an append-only financial ledger or equivalent immutable transaction history.

Ledger entries SHOULD record:

- Unique entry identifier.
- Related payment, refund, reversal, or settlement.
- Tenant and relevant account references.
- Debit or credit direction, where applicable.
- Amount and currency.
- Entry type.
- Source operation.
- Creation timestamp.
- Correlation identifier.
- Relevant actor or service identity.

The ledger design MUST define balancing, correction, and reversal behavior. Historical financial records MUST NOT be silently overwritten to correct errors.

### 14.2 Audit Events

Sensitive payment operations MUST produce audit events, including:

- Payment creation.
- Provider transaction association.
- Payment confirmation.
- Payment failure.
- Cancellation requests and confirmations.
- Refund initiation and completion.
- Settlement generation and payout.
- Payment state corrections.
- Administrative overrides.
- Reconciliation discrepancies.
- Payment configuration changes.

Audit events MUST be access-controlled, tamper-resistant to the extent practical, and retained according to applicable requirements.

### 14.3 Financial Corrections

Financial corrections MUST use explicit compensating records, reversals, or adjustments.

Directly changing historical payment amounts or deleting completed transaction records to conceal errors is prohibited.

Any manual correction MUST include an authorized actor, reason, supporting evidence, and audit reference.

## 15. Database and Transaction Security

### 15.1 Persistence

Payment data MUST be stored in appropriately protected datastores with controlled access.

Requirements:

- Enforce tenant-aware data access.
- Use database constraints for critical financial invariants.
- Use transactions for related state changes where supported.
- Avoid broad write permissions.
- Separate application roles from migration and administration roles.
- Encrypt data in transit and at rest according to the applicable infrastructure policy.
- Protect backups and exports with appropriate access controls.
- Define retention and deletion requirements for financial records.

### 15.2 Atomicity and Concurrency

Payment processing MUST account for concurrent requests, retries, webhook delivery, and background jobs.

Use:

- Unique constraints for transaction references and idempotency keys.
- Transactions for local atomic state changes.
- Conditional state updates or optimistic concurrency controls.
- Appropriate row-level or distributed locking where justified.
- Outbox or durable event delivery for cross-service workflows.

Do not hold database transactions open while waiting for external payment-provider network calls.

### 15.3 Datastore Separation

KAMPYN MAY use PostgreSQL as the authoritative store for payment and financial records.

Redis MAY be used for short-lived coordination, idempotency assistance, or caching, but MUST NOT be the sole durable source of truth for completed financial transactions.

MongoDB or other datastores MAY be used only where the architecture has an explicit ownership and consistency model.

Search indexes, analytics stores, and caches MUST be treated as derived representations rather than authoritative financial records.

## 16. Secrets and Credential Management

Payment-provider credentials are critical secrets.

KAMPYN MUST:

- Store provider API keys, webhook secrets, and signing credentials in an approved secret manager or secure deployment secret mechanism.
- Separate credentials across development, testing, staging, and production.
- Restrict access using least privilege.
- Avoid embedding secrets in source code, frontend bundles, SDKs, container images, logs, or documentation.
- Rotate credentials according to risk, provider requirements, and incident response procedures.
- Support revocation and replacement of compromised credentials.
- Audit privileged access to payment secrets.
- Avoid exposing production secrets to ordinary CI jobs or untrusted pull-request builds.

Frontend applications MUST never receive private provider API keys or webhook signing secrets.

If a credential is suspected to be compromised, it MUST be rotated or revoked promptly and the affected transactions and integrations investigated.

## 17. API Security

Payment APIs MUST follow `.ai/security/api-security.md`, `.ai/security/authentication.md`, `.ai/security/authorization.md`, and `.ai/security/input-validation.md`.

Payment endpoints MUST:

- Require authentication except for specifically designed provider callbacks.
- Enforce server-side authorization.
- Enforce tenant isolation.
- Validate request schemas.
- Enforce rate limits and request-size limits.
- Use idempotency for financial operations.
- Avoid leaking payment details through error messages.
- Prevent mass assignment of payment status, amounts, ownership, and provider identifiers.
- Protect against CSRF where cookie-based authentication and browser-originated state changes make it relevant.
- Apply appropriate CORS policies.
- Avoid using GET requests for financial state changes.
- Require explicit authorization for sensitive administrative operations.

Payment status endpoints MUST expose only information the authenticated principal is authorized to view.

## 18. Frontend and Checkout Security

The frontend MUST be treated as an untrusted client.

- Do not calculate authoritative payment totals exclusively in the browser.
- Do not mark orders or bookings paid based on query parameters or redirect results.
- Do not store payment secrets in local storage, session storage, Zustand, or other browser-accessible state.
- Avoid exposing provider credentials in JavaScript bundles.
- Validate return-route parameters and prevent open redirects.
- Ensure checkout and payment status interfaces are protected against unauthorized access.
- Clear sensitive transient state when the user signs out or changes tenant context.
- Display pending and uncertain states honestly rather than assuming success or failure.
- Avoid leaking sensitive transaction metadata in URLs, analytics, or client-side error reports.

The frontend MAY poll or subscribe to payment status updates, but MUST use backend-authoritative status.

## 19. WebSocket and Event Security

Payment-related realtime notifications MUST NOT be used as authoritative payment evidence.

- Authenticate realtime connections.
- Authorize access to payment status channels.
- Scope events to the correct user, tenant, and transaction.
- Validate event schemas.
- Avoid including sensitive payment instrument data.
- Prevent replayed or forged client events from changing payment state.
- Ensure downstream consumers process financial events idempotently.

Realtime events SHOULD notify clients that a payment status may have changed. The client MUST obtain authoritative status from the backend.

## 20. Rate Limiting and Fraud Controls

Payment endpoints MUST apply rate limits and abuse controls appropriate to the operation.

Controls SHOULD include:

- Payment initiation limits.
- Repeated payment attempt detection.
- Refund request limits.
- Webhook abuse protection.
- Account and device-level risk signals where lawful and appropriate.
- Per-tenant financial operation limits.
- Administrative operation monitoring.
- Provider-specific fraud and risk controls.

Fraud detection MAY consider unusual transaction frequency, repeated failures, anomalous amounts, or suspicious changes to account and payment context.

Automated fraud controls MUST be designed to avoid unjustified denial of legitimate transactions and SHOULD provide a review path for ambiguous cases.

Do not rely on a single client-controlled attribute as proof of fraud or legitimacy.

## 21. Payment Reconciliation

### 21.1 Purpose

Reconciliation is mandatory for identifying mismatches between KAMPYN's financial records and the payment provider's records.

KAMPYN MUST implement a documented reconciliation process for supported payment flows.

### 21.2 Reconciliation Inputs

Depending on provider capabilities, reconciliation MAY use:

- Provider transaction APIs.
- Settlement and payout reports.
- Refund reports.
- Dispute and chargeback reports.
- Internal payment records.
- Internal refund and adjustment records.
- Order and booking state.
- Ledger entries.

### 21.3 Reconciliation Rules

Reconciliation MUST compare appropriate fields, including:

- Provider transaction identifier.
- Internal payment identifier.
- Amount and currency.
- Transaction outcome.
- Capture status.
- Refund totals.
- Settlement or payout status.
- Relevant timestamps.
- Merchant or account context.

### 21.4 Mismatch Handling

Reconciliation discrepancies MUST be categorized, recorded, and assigned for resolution.

Examples include:

- Provider reports success but KAMPYN remains pending.
- KAMPYN records success but the provider reports failure.
- Duplicate transactions.
- Missing transactions.
- Amount or currency mismatch.
- Refund mismatch.
- Settlement discrepancy.
- Late or out-of-order events.

The system MUST NOT silently modify financial records to make mismatches disappear.

Automated correction MAY be performed only when the reconciliation rule is deterministic, authorized, auditable, and safe for the relevant transaction state.

### 21.5 Reconciliation Frequency

Reconciliation frequency MUST reflect transaction volume, provider capabilities, financial exposure, and contractual obligations.

KAMPYN SHOULD perform:

- Near-real-time verification for uncertain high-impact transactions where supported.
- Regular scheduled reconciliation of recent transactions.
- Settlement-period reconciliation.
- Periodic review of unresolved discrepancies.

Unresolved financial discrepancies MUST remain visible until formally resolved.

## 22. Refunds, Disputes, and Chargebacks

KAMPYN MUST support controlled handling of disputes and chargebacks where applicable to its payment providers.

- Associate dispute records with the original payment.
- Preserve provider references and relevant timelines.
- Restrict dispute actions to authorized personnel.
- Record evidence submissions and administrative decisions.
- Track chargeback amounts, fees, and outcomes.
- Reflect verified financial impacts in settlement and reconciliation records.
- Prevent dispute handling from silently modifying original transaction history.

Evidence MUST be collected and retained only as necessary, with access controls and retention limits appropriate to its sensitivity.

## 23. Logging and Monitoring

Payment activity MUST be observable without exposing sensitive financial credentials.

### 23.1 Required Logging

Where appropriate, logs SHOULD capture:

- Internal payment ID.
- Provider name.
- Provider transaction reference, if safe.
- Tenant ID.
- Relevant order or booking reference.
- Operation type.
- State transition.
- Result category.
- Correlation ID.
- Processing duration.
- Error category.
- Initiating service or authorized actor.

Logs MUST NOT include:

- Full card numbers.
- CVV/CVC.
- PINs or OTPs.
- Authentication secrets.
- Provider API keys.
- Webhook signing secrets.
- Raw payment credentials.
- Sensitive authentication headers.
- Unredacted provider payloads containing prohibited sensitive data.

### 23.2 Monitoring

Monitor at minimum:

- Payment initiation success and failure rates.
- Provider API latency and error rates.
- Pending transaction age.
- Uncertain transaction count.
- Webhook verification failures.
- Webhook processing backlog.
- Duplicate event detections.
- Refund success and failure rates.
- Settlement discrepancies.
- Reconciliation backlog.
- Payment state transition errors.
- Administrative payment actions.
- Suspected fraud and abuse indicators.

Alerts MUST be configured for severe or sustained anomalies.

Monitoring dashboards and alerts MUST enforce access controls and avoid unnecessary exposure of personal or financial data.

## 24. Failure Handling and Resilience

### 24.1 Provider Unavailability

If a payment provider is unavailable:

- Do not claim payment success without verification.
- Preserve the payment attempt and its known state.
- Apply bounded retries only where safe.
- Use idempotency for retryable operations.
- Avoid uncontrolled retry storms.
- Communicate pending or unavailable states accurately.
- Allow reconciliation or provider status verification after recovery.

### 24.2 Timeout Handling

A network timeout does not prove that a financial operation failed.

For timed-out initiation, capture, cancellation, refund, or payout requests:

- Preserve the operation reference.
- Check provider status before repeating the operation.
- Retry only according to documented provider semantics.
- Route unresolved outcomes to a recoverable state.
- Prevent duplicate financial effects.

### 24.3 Circuit Breakers and Backpressure

Provider integrations SHOULD use appropriate:

- Timeouts.
- Bounded retries with backoff and jitter.
- Circuit breakers.
- Concurrency limits.
- Queue-based processing.
- Backpressure.
- Dead-letter handling for persistent failures.

Resilience mechanisms MUST NOT bypass transaction verification or create duplicate charges.

### 24.4 Recovery

Payment recovery procedures MUST define:

- How to find pending and uncertain transactions.
- How to query provider state.
- How to reprocess verified events safely.
- How to resolve conflicting states.
- How to recover delayed order or booking fulfillment.
- How to reconcile refunds and settlements.
- Who is authorized to perform manual corrections.

Recovery operations MUST be idempotent and auditable.

## 25. Multi-Tenant Payment Security

KAMPYN MUST enforce tenant isolation across all payment-related operations.

- Every internal payment record MUST be associated with the correct tenant where applicable.
- Tenant context MUST be derived from trusted authentication or verified service context.
- Resource ownership MUST be verified before initiating payment.
- Provider accounts and merchant contexts MUST be mapped explicitly to their authorized tenants.
- Webhook events MUST be associated with the correct provider account and tenant.
- Refunds and settlements MUST be scoped to the correct tenant and beneficiary.
- Payment lookups MUST enforce tenant-aware authorization.
- Cache keys, queue messages, and event payloads MUST preserve tenant context.
- Reconciliation MUST identify tenant mismatches and cross-tenant references.
- Administrative cross-tenant access MUST require explicit permissions and auditing.

A valid provider transaction ID MUST NOT grant access to a payment record or associated order.

For university self-hosted deployments, the deployment MUST define the payment provider's credential ownership, financial record ownership, webhook configuration, and reconciliation responsibilities.

## 26. Compliance and Regulatory Requirements

Payment processing MUST comply with applicable laws, regulations, provider contracts, and security standards.

Where card data is involved, the relevant PCI DSS scope and obligations MUST be formally assessed.

KAMPYN MUST:

- Minimize cardholder-data exposure.
- Use provider-supported payment collection and processing methods.
- Determine applicable compliance responsibilities with the payment provider and qualified advisors.
- Avoid making unsupported claims of PCI DSS compliance or certification.
- Maintain required transaction records according to applicable legal and contractual obligations.
- Follow relevant consumer protection, privacy, tax, and financial recordkeeping requirements.
- Ensure university deployments understand their respective compliance responsibilities.

Regulatory obligations MAY differ by payment method, provider, merchant arrangement, jurisdiction, and deployment model. These MUST be confirmed for the actual operating context.

## 27. Testing Requirements

Payment-related changes MUST receive comprehensive testing before release.

### 27.1 Unit Tests

Test:

- Amount calculation.
- Currency handling.
- Fee and commission calculations.
- Payment state transitions.
- Idempotency behavior.
- Refund limits.
- Authorization decisions.
- Tenant-context validation.
- Provider status mapping.
- Error classification.

### 27.2 Integration Tests

Test:

- Provider request construction.
- Provider response parsing.
- Signature verification.
- Webhook processing.
- Payment status verification.
- Transaction persistence.
- Outbox and event delivery.
- Refund creation and confirmation.
- Reconciliation logic.
- Database constraints.
- Retry and recovery behavior.

Use provider sandbox environments or controlled test doubles according to the test objective. Test doubles MUST NOT replace provider integration testing entirely.

### 27.3 Security Tests

Test at minimum:

- Forged payment success responses.
- Invalid webhook signatures.
- Replayed webhook events.
- Duplicate payment initiation.
- Duplicate refund requests.
- Unauthorized refund attempts.
- Cross-tenant payment access.
- Amount and currency tampering.
- Payment status mass assignment.
- Forged provider identifiers.
- Stale or out-of-order provider events.
- Unauthorized settlement operations.
- Sensitive data exposure through logs or APIs.
- Abuse through repeated payment attempts.
- Provider timeout and ambiguous outcome handling.

### 27.4 Concurrency Tests

Test concurrent:

- Payment initiation for the same order.
- Payment status updates and webhook delivery.
- Refund requests against the same payment.
- Cancellation and payment completion.
- Settlement generation.
- Inventory or booking confirmation after payment.

Tests MUST verify that concurrent operations do not create duplicate charges, over-refunds, double fulfillment, or inconsistent financial state.

### 27.5 End-to-End Tests

End-to-end tests SHOULD cover:

- Successful payment.
- Failed payment.
- Pending payment.
- Additional authentication.
- Delayed webhook delivery.
- Duplicate and out-of-order webhooks.
- Refund and partial refund.
- Cancellation and reversal.
- Provider outage.
- Recovery of uncertain outcomes.
- Order or booking fulfillment after verified payment.
- Reconciliation of mismatched transactions.

## 28. Deployment and Release Security

Payment-related production changes MUST use controlled deployment procedures.

- Separate test and production provider accounts and credentials.
- Validate webhook endpoints and secret configuration before activation.
- Restrict production payment access to authorized services.
- Verify migrations and financial data compatibility.
- Ensure backward-compatible webhook handling during rolling deployments.
- Monitor payment behavior after deployment.
- Maintain rollback or mitigation procedures.
- Avoid replaying historical financial events without deduplication safeguards.
- Validate configuration for self-hosted installations.
- Maintain clear operational ownership for provider integration incidents.

Changes affecting payment state models, amount calculation, idempotency, webhook verification, refunds, or settlements MUST receive focused review and staged validation.

## 29. Administrative and Operational Controls

Sensitive payment administration MUST be restricted and auditable.

Administrative functions MAY include:

- Viewing transaction details.
- Reviewing pending and uncertain payments.
- Initiating authorized refunds.
- Reviewing reconciliation discrepancies.
- Managing provider configuration.
- Generating settlement reports.
- Resolving verified financial exceptions.

Requirements:

- Apply least-privilege access.
- Require stronger authentication for high-impact operations where appropriate.
- Enforce separation of duties for configured thresholds.
- Record all administrative actions.
- Restrict exports of sensitive financial data.
- Require a reason for manual adjustments.
- Prevent direct editing of completed financial history.
- Review privileged access periodically.
- Provide an auditable break-glass procedure for exceptional incidents.

Administrative interfaces MUST NOT bypass the backend payment state machine or provider verification requirements.

## 30. Backup and Disaster Recovery

Payment and financial records MUST be included in appropriate backup and recovery procedures.

- Encrypt backups at rest and in transit.
- Restrict backup access.
- Define retention and deletion policies.
- Test restoration procedures.
- Preserve transaction identifiers and relationships.
- Maintain reconciliation capabilities after restoration.
- Prevent duplicate financial processing during recovery.
- Revalidate uncertain provider outcomes after restoring application state.
- Ensure outbox and event replay mechanisms remain idempotent.

Recovery objectives MUST be defined according to the business impact of payment data loss or unavailability.

Restoring KAMPYN databases MUST NOT cause historical payment events to be processed as new financial operations.

## 31. Prohibited Practices

The following practices are prohibited:

- Trusting the frontend to declare payment success.
- Trusting unverified webhooks or provider callbacks.
- Accepting client-provided payment amounts as authoritative.
- Storing raw card data, CVV/CVC, PINs, or OTPs in KAMPYN systems.
- Exposing private provider credentials to frontend applications.
- Marking transactions successful based only on redirect parameters.
- Repeating financial operations without idempotency or provider-state checks.
- Processing the same webhook event more than once with financial effects.
- Allowing users to modify payment status or provider transaction references.
- Initiating refunds without authorization and balance validation.
- Allowing refunds to exceed the refundable amount.
- Silently overwriting completed financial records.
- Using floating-point arithmetic for authoritative financial calculations.
- Treating a cancellation request as a confirmed reversal or refund.
- Using Redis or a search index as the sole durable financial source of truth.
- Logging secrets, payment credentials, or prohibited sensitive payment data.
- Allowing cross-tenant access to payment or settlement records.
- Bypassing provider-required payment authentication.
- Ignoring reconciliation discrepancies.
- Performing unreviewed manual financial corrections.
- Disabling payment verification or security controls to resolve operational delays.

## 32. Review Checklist

Before merging any payment-related change, reviewers MUST verify:

- [ ] Payment initiation is server-side and authorized.
- [ ] Tenant and resource ownership are verified.
- [ ] Amounts and currency are calculated authoritatively.
- [ ] Financial calculations use exact representations.
- [ ] The provider integration uses trusted APIs and credentials.
- [ ] Payment instrument data is not unnecessarily collected or stored.
- [ ] Payment attempts are idempotent.
- [ ] Payment states and transitions are explicit.
- [ ] Provider responses and webhooks are authenticated and validated.
- [ ] Replay, duplicate, and out-of-order events are handled safely.
- [ ] Payment success is independently verified.
- [ ] Order or booking fulfillment is idempotent and consistent.
- [ ] Refunds and cancellations have separate authorization and state handling.
- [ ] Financial records are durable and auditable.
- [ ] Tenant isolation is enforced across all payment data paths.
- [ ] Errors and logs do not expose sensitive data.
- [ ] Rate limits and abuse controls are implemented.
- [ ] Provider outages and uncertain outcomes are recoverable.
- [ ] Reconciliation and discrepancy handling are defined.
- [ ] Administrative operations are restricted and audited.
- [ ] Relevant unit, integration, security, concurrency, and end-to-end tests pass.
- [ ] Deployment and rollback procedures are documented.
- [ ] Compliance responsibilities have been assessed.

## 33. Definition of Done

A payment-related feature is complete only when:

- The payment architecture and provider responsibilities are documented.
- All payment operations are initiated and controlled by the backend.
- Payment amounts and eligibility are calculated from authoritative domain data.
- Payment instrument data is minimized and protected.
- Authentication, authorization, and tenant isolation are enforced.
- Idempotency and transaction state management are implemented.
- Provider responses and webhook events are authenticated and verified.
- Order, booking, refund, and settlement workflows are consistent and recoverable.
- Financial records are durable, auditable, and reconciliable.
- Errors, timeouts, retries, and provider outages are handled safely.
- Monitoring, alerting, and operational recovery procedures are in place.
- Security and concurrency tests cover expected and adversarial behavior.
- Required compliance responsibilities are assessed and documented.
- No known critical payment-security defect remains unresolved.

## 34. Final Rule

A payment MUST be treated as successful only when KAMPYN has verified authoritative evidence from the payment provider and safely recorded the resulting financial state.

Every payment, refund, cancellation, reversal, and settlement MUST be authorized, idempotent, tenant-scoped, auditable, and recoverable. No frontend response, unverified event, or manual status change may override these requirements.
# Frontend Validation Standards

## 1. Purpose

This document defines the validation standards for KAMPYN's frontend, ensuring that user input, API responses, form data, URL parameters, environment variables, and other runtime values are validated consistently, safely, and predictably.

KAMPYN serves students, faculty, university administrators, vendors, and staff across workflows such as food ordering, hostel and guest-house booking, library availability, shuttle scheduling, inventory management, complaints, community interactions, and payments. Reliable validation is essential to protect data integrity, improve user experience, and prevent invalid or malformed data from propagating through the application.

Frontend validation is a usability and data-quality layer. It does not replace backend validation, authorization, or business-rule enforcement.

### Core objectives

- Validate user input before submitting it to the backend.
- Use Zod for runtime validation at appropriate application boundaries.
- Maintain type safety across forms, components, and API integrations.
- Provide clear, accessible, and actionable validation feedback.
- Prevent malformed data from entering frontend application state.
- Validate API responses when runtime trust boundaries require it.
- Avoid duplicated validation logic and inconsistent rules.
- Ensure validation is consistent across web, mobile, and shared SDK interfaces where applicable.
- Protect sensitive information and avoid exposing internal validation details.
- Keep validation logic maintainable, reusable, testable, and performant.

---

## 2. Non-Negotiable Principles

1. **Runtime validation is mandatory at trust boundaries:** TypeScript types alone are insufficient to validate external or runtime data.
2. **Zod is the standard validation library:** Use Zod for frontend runtime schemas unless a documented architectural decision specifies otherwise.
3. **The backend remains authoritative:** Frontend validation must never be treated as proof of data integrity, authorization, or business-rule compliance.
4. **Validate at boundaries:** Validate external data when it enters the frontend application, not repeatedly in every component.
5. **Reuse schemas:** Shared validation rules must have a single clear source of truth wherever possible.
6. **Keep validation separate from presentation:** Schemas and validation logic should not be tightly coupled to individual UI components.
7. **Provide actionable feedback:** Validation messages must help users understand how to correct invalid input.
8. **Never trust client-controlled values:** User IDs, tenant IDs, roles, prices, permissions, and other sensitive values must be verified by the backend.
9. **Avoid redundant validation:** Do not repeatedly validate already trusted internal data without a clear reason.
10. **Preserve type safety:** Derive TypeScript types from Zod schemas wherever practical to prevent schema and type drift.
11. **Validate asynchronous data safely:** API responses, URL parameters, browser storage, and realtime messages must be treated as runtime values.
12. **Do not weaken validation to make requests pass:** Invalid data must be corrected at its source or handled through an explicit, documented compatibility strategy.

---

## 3. Validation Architecture

Frontend validation must follow clear boundaries and responsibilities.

<escape>
User Input
    |
    v
UI Component / Form
    |
    v
Client-Side Validation (Zod)
    |
    +---- Invalid ----> Accessible Validation Feedback
    |
    v
Valid Form Data
    |
    v
API Client / Mutation
    |
    v
Backend Validation (Authoritative)
    |
    +---- Invalid ----> Structured API Error
    |
    v
Domain Logic / Persistence
    |
    v
Validated API Response
    |
    v
Frontend Response Validation (where required)
    |
    v
Typed Application State / UI
</escape>

### 3.1 Validation layers

| Layer | Responsibility |
|---|---|
| UI components | Display validation state and guide user interaction |
| Form layer | Coordinate field values, submission, and validation timing |
| Zod schemas | Define reusable runtime validation rules |
| API client | Validate request and response boundaries where required |
| TanStack Query | Manage server state and validated response data |
| Backend | Enforce authoritative validation, authorization, and business rules |
| Domain services | Enforce domain invariants and workflow constraints |

Each layer must have a clearly defined responsibility. Validation must not be duplicated across multiple layers without a specific purpose.

### 3.2 Client-side and backend validation

Client-side validation exists to improve usability, reduce avoidable requests, and catch common mistakes early.

Backend validation exists to protect the system and enforce authoritative rules.

The backend must independently validate:

- Required fields.
- Data types and formats.
- Length and range constraints.
- Business rules.
- Tenant ownership.
- Authentication and authorization.
- Resource availability.
- Financial calculations.
- State transitions.
- Referential integrity.
- Duplicate or conflicting operations.

A valid frontend payload may still be rejected by the backend because the authoritative state may have changed or the request may violate a business rule.

---

## 4. Zod as the Standard

Zod is the standard library for frontend runtime validation.

### 4.1 Schema-first validation

Define a schema first and derive the corresponding TypeScript type from it.

```ts
import { z } from "zod";

export const emailSchema = z
  .string()
  .trim()
  .min(1, "Email is required.")
  .email("Enter a valid email address.");

export type Email = z.infer<typeof emailSchema>;
```

This ensures that runtime validation and compile-time types are derived from the same definition.

### 4.2 Schema requirements

Every reusable schema must:

- Have a clear and descriptive name.
- Represent one well-defined data contract.
- Define required and optional fields explicitly.
- Enforce appropriate type, format, length, and range constraints.
- Avoid unnecessary transformations.
- Use consistent error-message conventions.
- Be covered by tests for valid and invalid inputs.
- Be located in the appropriate domain or feature module.

### 4.3 Schema reuse

Shared validation rules must be reused rather than independently reimplemented.

For example, if multiple forms use the same email validation, they should import a shared schema instead of defining slightly different email rules.

However, reuse must not create oversized schemas that combine unrelated domains.

Prefer composing smaller schemas into larger schemas when necessary.

### 4.4 Schema ownership

Schemas should be owned by the domain or feature that defines the relevant frontend contract.

Examples:

- Authentication schemas belong to the authentication feature.
- Food ordering schemas belong to the ordering feature.
- Booking schemas belong to the booking feature.
- Profile schemas belong to the user profile feature.
- Shared primitive schemas belong to an appropriate shared validation module.

A suitable structure:

```text
src/
├── lib/
│   └── validation/
│       ├── primitives.ts
│       ├── common.ts
│       └── errors.ts
│
├── features/
│   ├── auth/
│   │   └── validation/
│   │       ├── login.schema.ts
│   │       ├── signup.schema.ts
│   │       └── password.schema.ts
│   │
│   ├── orders/
│   │   └── validation/
│   │       ├── cart.schema.ts
│   │       └── order.schema.ts
│   │
│   └── bookings/
│       └── validation/
│           ├── booking.schema.ts
│           └── availability.schema.ts
│
└── components/
    └── forms/
```

The exact structure may follow the repository's established conventions, but schema ownership must remain clear.

---

## 5. Type Safety and Schema Inference

### 5.1 Infer types from schemas

Use `z.infer` for types corresponding to validated schema output.

```ts
import { z } from "zod";

export const profileSchema = z.object({
  name: z.string().trim().min(2).max(100),
  email: z.string().trim().email(),
});

export type ProfileInput = z.input<typeof profileSchema>;
export type ProfileData = z.output<typeof profileSchema>;
```

Use input and output types when transformations cause them to differ.

### 5.2 Avoid duplicated type definitions

Do not manually define a TypeScript interface that duplicates an existing Zod schema unless there is a documented reason, such as a distinct external contract.

Avoid maintaining separate, manually synchronized definitions of the same data structure.

### 5.3 Unknown data

Treat external and untrusted values as `unknown` until they have been validated.

```ts
function parseProfile(value: unknown) {
  return profileSchema.safeParse(value);
}
```

Do not use `any` to bypass validation or type checking.

### 5.4 Optional and nullable fields

Optional and nullable values represent different contracts.

- Optional means a field may be absent.
- Nullable means a field may explicitly contain `null`.

Define each deliberately and avoid silently converting between them.

Use `exactOptionalPropertyTypes` consistently with the project's TypeScript standards.

### 5.5 Discriminated unions

Use discriminated unions to model values that have multiple valid shapes.

```ts
import { z } from "zod";

export const paymentMethodSchema = z.discriminatedUnion("type", [
  z.object({
    type: z.literal("card"),
    paymentToken: z.string().min(1),
  }),
  z.object({
    type: z.literal("upi"),
    upiId: z.string().min(1),
  }),
]);

export type PaymentMethod = z.infer<typeof paymentMethodSchema>;
```

Avoid broad optional-field objects when the valid structure depends on a specific state or type.

---

## 6. Form Validation

Forms must use consistent validation behavior and provide clear feedback without unnecessarily interrupting the user.

### 6.1 Form schema

Each form with meaningful constraints should have a dedicated schema or use an appropriate shared schema.

Example:

```ts
import { z } from "zod";

export const signupSchema = z.object({
  name: z
    .string()
    .trim()
    .min(2, "Name must contain at least 2 characters.")
    .max(100, "Name must not exceed 100 characters."),

  email: z
    .string()
    .trim()
    .email("Enter a valid email address."),

  password: z
    .string()
    .min(12, "Password must contain at least 12 characters."),
});

export type SignupInput = z.infer<typeof signupSchema>;
```

The example illustrates schema structure. Actual password, email, and account requirements must follow KAMPYN's authentication and security policies.

### 6.2 Validation timing

Choose validation timing based on the form's interaction needs.

| Strategy | Appropriate use |
|---|---|
| On submit | Long or multi-step forms where early validation is disruptive |
| On blur | Field-level feedback after a user finishes editing |
| On change | Immediate feedback for simple or previously invalid fields |
| On input | Specific interactions requiring immediate feedback |
| Async validation | Remote checks such as availability, with debouncing and cancellation |

Recommended behavior:

- Avoid showing errors for untouched fields on initial render unless the context requires it.
- Validate required fields on submission.
- After a field has an error, revalidate it at a suitable interaction point.
- Avoid expensive validation on every keystroke.
- Keep the form's validation behavior predictable and consistent.

### 6.3 Field-level validation

Each field must:

- Have a visible or programmatically associated label.
- Display an understandable validation message when invalid.
- Expose invalid state accessibly.
- Preserve the user's entered value when safe.
- Avoid relying on color alone to indicate errors.
- Identify how the user can correct the problem.

### 6.4 Form-level validation

Use form-level validation for constraints involving multiple fields or the entire form.

Examples:

- Password and confirm-password matching.
- Start date preceding end date.
- Booking dates falling within permitted ranges.
- Cart quantities meeting ordering constraints.
- Mutually exclusive form options.

Keep cross-field rules in the relevant schema or validation layer rather than scattering them across individual UI components.

### 6.5 Submission validation

Before submitting:

1. Validate the current form values.
2. Prevent submission if local validation fails.
3. Display actionable errors.
4. Prevent accidental duplicate submission.
5. Submit the validated payload through the API client.
6. Handle structured backend validation errors.
7. Update UI state only according to the authoritative response.

Client-side validation must not be considered sufficient to confirm a successful operation.

---

## 7. React Hook Form Integration

When React Hook Form is used, integrate Zod through the appropriate resolver rather than manually duplicating schema rules.

Example:

```tsx
import { zodResolver } from "@hookform/resolvers/zod";
import { useForm } from "react-hook-form";

import {
  signupSchema,
  type SignupInput,
} from "./signup.schema";

export function SignupForm() {
  const form = useForm<SignupInput>({
    resolver: zodResolver(signupSchema),
    mode: "onBlur",
    reValidateMode: "onChange",
    defaultValues: {
      name: "",
      email: "",
      password: "",
    },
  });

  const onSubmit = form.handleSubmit(async (values) => {
    // Submit validated values through the feature API client.
  });

  return (
    <form onSubmit={onSubmit} noValidate>
      {/* Accessible fields and submission controls */}
    </form>
  );
}
```

Standards:

- Use schema resolvers for Zod-backed forms.
- Define appropriate default values.
- Use explicit validation modes.
- Avoid unnecessary whole-form subscriptions.
- Use field-level subscriptions where appropriate.
- Keep submission handlers focused on orchestration.
- Keep API and domain operations outside the form component.
- Avoid duplicating validation rules in input components.

The example is illustrative; the actual form must implement accessible field labels, error associations, pending states, and API error handling.

---

## 8. Input Normalization and Transformation

Normalization may be applied where it is well-defined, safe, and consistent with the backend contract.

Examples include:

- Trimming accidental surrounding whitespace from names.
- Normalizing case for identifiers when the domain contract specifies case-insensitive behavior.
- Converting a user-entered numeric string into a number at a clearly defined boundary.
- Normalizing supported date input formats before submission.

Standards:

- Use Zod transformations only when the expected input and output are clearly defined.
- Avoid silently altering meaningful user input.
- Do not normalize passwords unless explicitly required by the authentication contract.
- Do not assume all identifiers are case-insensitive.
- Do not perform lossy transformations without explicit domain approval.
- Keep formatting concerns separate from canonical data representation.

For example, a formatted phone number may be displayed in the UI, but the submitted representation must follow the backend's documented contract.

### 8.1 Coercion

Use coercion deliberately.

Avoid permissive coercion that can turn malformed strings, empty inputs, or unexpected values into misleading valid values.

Validate the source representation and the transformed output whenever the conversion could introduce ambiguity.

### 8.2 Dates and times

Date and time validation must respect the domain contract.

- Distinguish date-only values from timestamps.
- Define timezone expectations.
- Avoid assuming the user's local timezone is the backend's timezone.
- Validate date ranges and permitted values.
- Avoid relying solely on browser-native date input constraints.
- Ensure booking and scheduling values are interpreted consistently by the backend.

A locally valid date does not establish that a booking slot is available.

---

## 9. API Request Validation

The frontend must construct requests that conform to documented API contracts.

### 9.1 Request schemas

Define request schemas where they provide meaningful runtime assurance, particularly for:

- Complex form payloads.
- Data assembled from multiple UI states.
- Requests that cross shared frontend modules.
- Persisted drafts or restored browser state.
- Critical user workflows.

Use types derived from schemas to ensure request construction remains type-safe.

### 9.2 Request boundaries

Validate untrusted or dynamically assembled request data before sending it.

Examples include:

- Values restored from browser storage.
- URL-derived filters.
- Dynamic form data.
- Imported data.
- User-generated content.
- Realtime-derived state used to construct subsequent requests.

Avoid repeatedly parsing fully trusted internal values when the same data has already been validated and has not crossed a new trust boundary.

### 9.3 Sensitive fields

Never rely on frontend validation for sensitive or security-critical request fields.

The backend must independently determine or verify:

- Authenticated user identity.
- Tenant identity and membership.
- User role and permissions.
- Prices and totals.
- Payment status.
- Inventory availability.
- Booking eligibility.
- Administrative privileges.
- Ownership of resources.

Client-provided values for these fields must not be treated as authoritative.

---

## 10. API Response Validation

TypeScript declarations do not validate JSON received over the network.

Treat API responses as runtime data and validate them at appropriate boundaries.

### 10.1 Response schemas

Use response schemas for:

- Untrusted or external API integrations.
- Critical workflows where malformed data could cause incorrect UI behavior.
- Dynamic or loosely typed payloads.
- Persisted data restored from a previous application version.
- Realtime messages.
- Data consumed across independently versioned services.

For strongly controlled internal APIs, use generated or shared contracts where available, and apply runtime parsing where the risk justifies it.

### 10.2 API client boundary

Response validation should be centralized in the API client or feature data-access layer rather than repeated in every component.

Illustrative example:

```ts
import { z } from "zod";

const orderSchema = z.object({
  id: z.string().min(1),
  status: z.enum([
    "pending",
    "confirmed",
    "preparing",
    "completed",
    "cancelled",
  ]),
  total: z.number().nonnegative(),
});

export type Order = z.infer<typeof orderSchema>;

export function parseOrder(value: unknown): Order {
  return orderSchema.parse(value);
}
```

This example demonstrates the boundary pattern. The actual status values and monetary representation must match KAMPYN's canonical API contract.

### 10.3 Invalid responses

When a response fails validation:

- Do not silently cast it to the expected type.
- Do not continue processing malformed data as trusted application state.
- Return a typed error or controlled failure state.
- Log safe diagnostic information where appropriate.
- Avoid exposing raw response bodies or internal schema details to end users.
- Provide a user-facing message that does not disclose sensitive implementation details.
- Use a suitable retry policy only if the failure is potentially transient.

### 10.4 Forward and backward compatibility

API contracts may evolve independently across services and university deployments.

- Use explicit versioning or compatibility policies where required.
- Handle additive optional fields intentionally.
- Avoid assuming unknown fields always indicate a fatal error.
- Reject missing or invalid required fields.
- Document compatibility expectations.
- Test supported API versions and response variants.

Schema strictness should be a deliberate contract decision, not a default reaction to every additional field.

---

## 11. URL and Route Parameter Validation

URL parameters are runtime inputs and must be validated before they influence application behavior.

Validate:

- Dynamic route parameters.
- Query parameters.
- Search filters.
- Sort fields and directions.
- Pagination values.
- Date ranges.
- Identifiers.
- Redirect destinations.
- Values used to construct API requests.

### 11.1 Query parameter schemas

Example:

```ts
import { z } from "zod";

export const searchParamsSchema = z.object({
  q: z.string().trim().max(200).optional(),
  page: z.coerce.number().int().min(1).default(1),
  pageSize: z.coerce.number().int().min(1).max(100).default(20),
  sort: z.enum(["name", "createdAt"]).default("name"),
});

export type SearchParams = z.infer<typeof searchParamsSchema>;
```

The allowed sort fields and page-size limits must match the backend API contract.

### 11.2 Route handling

- Validate route parameters before using them in data-fetching operations.
- Handle malformed identifiers with controlled not-found or invalid-input states.
- Do not assume a route parameter refers to a resource the user is authorized to access.
- Keep navigation and filtering state consistent with the validated parameter values.
- Avoid passing raw query strings directly into backend requests without validation.

### 11.3 Redirect validation

Redirect targets derived from query parameters or other user-controlled values must be checked against an explicit allowlist or safe same-origin policy.

Do not permit arbitrary untrusted redirect destinations.

---

## 12. Environment Variable Validation

Environment variables are runtime configuration and must be validated before they are used.

### 12.1 Environment schemas

Validate required configuration such as:

- Public application URLs.
- Public feature flags.
- Analytics configuration.
- API base URLs.
- Runtime environment indicators.

Use separate schemas for server-only and client-exposed environment variables.

### 12.2 Client exposure

Only variables intentionally exposed through the `NEXT_PUBLIC_` prefix should be included in client-side bundles.

Never expose:

- Private API keys.
- Database credentials.
- Signing secrets.
- Internal service credentials.
- Privileged backend tokens.

### 12.3 Failure behavior

- Fail early for missing or invalid required configuration.
- Use safe defaults only when they are explicitly defined.
- Avoid silently substituting invalid production values.
- Ensure configuration errors do not reveal secrets.
- Validate URL formats and permitted protocols.

Environment validation must follow the frontend and infrastructure configuration policies.

---

## 13. Browser Storage Validation

Browser storage is not a trusted source of application data.

Validate values retrieved from:

- `localStorage`.
- `sessionStorage`.
- IndexedDB.
- Persisted Zustand stores.
- Cached form drafts.
- Client-side preferences.

### 13.1 Storage requirements

- Treat stored values as `unknown`.
- Validate them before use.
- Handle missing, malformed, or outdated records.
- Provide versioning for persisted structures where necessary.
- Reset invalid or incompatible state safely.
- Avoid persisting sensitive data without explicit approval.
- Do not treat storage values as evidence of identity, authorization, or tenant membership.

### 13.2 Persisted state compatibility

Persisted data may outlive the code version that created it.

- Include a version when persisted structure may evolve.
- Migrate known compatible versions deliberately.
- Discard unsupported versions safely.
- Prevent stale drafts from overwriting newer authoritative server data.
- Keep migration logic small, deterministic, and tested.

---

## 14. Authentication and Authorization Validation

Authentication and authorization are security boundaries and must never rely on frontend validation alone.

### 14.1 Authentication forms

Validate:

- Required credentials.
- Input format.
- Password requirements for relevant account operations.
- Verification code format and length.
- Recovery and reset form structure.

Do not reveal whether an account exists when the authentication flow is designed to prevent account enumeration.

### 14.2 Authorization-dependent UI

Frontend checks may be used to:

- Hide controls the current user cannot use.
- Display relevant navigation.
- Improve interaction flow.
- Avoid presenting unavailable actions unnecessarily.

However:

- The backend must enforce all permissions.
- Client-side roles and permission flags are informational only.
- A modified client must not gain access to protected operations.
- Resource ownership must be verified server-side.
- Tenant membership must be verified server-side.

### 14.3 Session and tenant changes

When the authenticated identity or tenant changes:

- Clear or isolate user- and tenant-scoped query caches.
- Reset relevant transient state.
- Prevent stale asynchronous responses from updating the new session.
- Re-establish relevant subscriptions under the new authorized context.
- Avoid showing previous-tenant data while the new context is loading.

---

## 15. Domain-Specific Validation

KAMPYN contains multiple operational domains. Validation must reflect the relevant domain contract while keeping final enforcement on the backend.

### 15.1 Food ordering

Frontend validation may check:

- Item selection.
- Quantity format and permitted client-side range.
- Required customization selections.
- Cart structure.
- Delivery or pickup selection.
- Required user input.

The backend must verify:

- Current item availability.
- Actual item prices.
- Discounts and taxes.
- Vendor and food court eligibility.
- Order acceptance rules.
- Inventory constraints.
- Final payable amount.
- Valid order state transitions.

A locally valid cart is not proof that an order can be fulfilled.

### 15.2 Payments

Frontend validation may check:

- Required payment method selection.
- Expected input formats.
- Required fields for the selected payment flow.
- Client-side request structure.

The backend and payment provider integration must determine:

- Transaction status.
- Payment authorization and capture.
- Amount and currency.
- Idempotency.
- Refund eligibility.
- Payment confirmation.

Never treat client-side success callbacks or form validation as authoritative proof of payment.

### 15.3 Hostel and guest-house bookings

Frontend validation may check:

- Required dates.
- Date ordering.
- Guest count format.
- Required booking details.
- Selection of an available option shown to the user.

The backend must verify:

- Actual availability.
- Booking eligibility.
- Current rates.
- Capacity.
- Conflicting bookings.
- University-specific rules.
- Final reservation status.

### 15.4 Library availability

Frontend validation may check:

- Search parameters.
- Location or section selection.
- Filter and pagination values.

The backend must provide authoritative availability data and enforce access restrictions.

### 15.5 Shuttle booking

Frontend validation may check:

- Route selection.
- Schedule selection.
- Passenger input.
- Required booking details.

The backend must verify:

- Route availability.
- Current departure schedule.
- Capacity.
- Booking eligibility.
- Seat or reservation status.

### 15.6 Inventory management

Frontend validation may check:

- Numeric input format.
- Required item fields.
- Supported units.
- Request structure.

The backend must enforce:

- Stock constraints.
- Authorized adjustments.
- Unit consistency.
- Concurrent update handling.
- Audit requirements.
- Valid inventory transitions.

### 15.7 Complaints and reports

Frontend validation may check:

- Required subject and description.
- Input length.
- Category selection.
- Attachment type and size constraints.
- Structured submission fields.

The backend must enforce:

- Attachment safety.
- Upload limits.
- User eligibility.
- Tenant ownership.
- Submission rules.
- Rate limits and abuse prevention.

### 15.8 Community and chat

Frontend validation may check:

- Message structure.
- Input length.
- Supported attachment metadata.
- Valid interaction payloads.

The backend must enforce:

- Membership and authorization.
- Message permissions.
- Rate limits.
- Attachment validation and scanning.
- Moderation and reporting rules.
- Tenant isolation.

Client-side content filtering may improve usability but must not be treated as the security or moderation boundary.

---

## 16. File and Upload Validation

File selection and metadata supplied by the browser are untrusted.

### 16.1 Client-side checks

Where applicable, validate:

- File presence.
- File size.
- Allowed extension.
- Declared MIME type.
- Number of files.
- Required metadata.
- Upload form structure.

Client-side checks should provide early feedback and prevent avoidable uploads.

### 16.2 Backend enforcement

The backend must independently validate:

- Actual file content and format.
- File size and upload limits.
- MIME type using appropriate inspection.
- Filename safety.
- Malware or unsafe content according to the platform's security policy.
- Storage authorization.
- Tenant ownership.
- Access permissions.

Do not trust file extensions or browser-supplied MIME types as proof of file safety.

### 16.3 Upload UX

- Show progress when supported.
- Handle cancellation where possible.
- Prevent accidental duplicate submissions.
- Preserve recoverable form data when safe.
- Communicate rejected files clearly.
- Avoid exposing raw storage errors or internal paths.

---

## 17. Asynchronous and Remote Validation

Some validations require a server request, such as checking a username or verifying a resource's current status.

### 17.1 Remote validation rules

- Use remote validation only when it provides meaningful user value.
- Debounce high-frequency requests.
- Cancel obsolete requests where supported.
- Handle network failure separately from invalid input.
- Avoid presenting stale responses as current.
- Do not expose sensitive account or resource existence information.
- Revalidate critical conditions during final backend submission.

### 17.2 Race conditions

When users change a value while a remote validation request is pending:

- Ensure only the latest relevant response can update the field state.
- Use request cancellation or request identity tracking.
- Ignore obsolete responses.
- Keep loading indicators scoped to the relevant field or action.

### 17.3 Availability validation

Availability checks for bookings, shuttles, inventory, and other constrained resources are informational until the backend confirms the operation.

A previously available slot may become unavailable before submission. The UI must handle this response clearly and allow the user to select another option.

---

## 18. Error Handling and Validation Messages

Validation errors must be consistent, safe, and understandable.

### 18.1 Error message principles

Messages should be:

- Clear and concise.
- Specific to the input or action.
- Written in plain language.
- Actionable where possible.
- Consistent across related workflows.
- Accessible to assistive technologies.
- Free from sensitive implementation details.

Prefer:

- "Enter a valid email address."
- "Choose a booking date."
- "Quantity must be at least 1."
- "This time slot is no longer available."

Avoid:

- Raw Zod error objects.
- Stack traces.
- Internal database or service messages.
- Unexplained error codes.
- Messages that blame or shame users.
- Technical details that do not help the user correct the problem.

### 18.2 Error categories

Distinguish between:

| Error category | Example | Handling |
|---|---|---|
| Client validation | Invalid email format | Display field-level feedback |
| Backend validation | Quantity exceeds permitted limit | Map safe API error to the relevant field |
| Authentication | Session expired | Trigger appropriate reauthentication flow |
| Authorization | User lacks permission | Show a safe access-denied state |
| Conflict | Slot became unavailable | Explain the conflict and offer recovery |
| Network | Request timed out | Provide retry where safe |
| Server | Internal operation failed | Show a generic failure and safe recovery path |
| Response validation | Malformed API payload | Stop unsafe processing and report a controlled error |

### 18.3 Mapping backend errors

- Map documented backend validation codes to user-friendly messages.
- Associate errors with fields only when the API contract identifies the relevant field.
- Handle unknown error codes safely.
- Avoid displaying raw backend messages without review.
- Keep error mapping centralized where possible.
- Do not suppress errors that indicate data-integrity or security concerns.

### 18.4 Error summaries

Long or multi-step forms may benefit from a summary of validation errors.

Error summaries should:

- Identify fields requiring attention.
- Link or focus the relevant field when appropriate.
- Avoid duplicating large amounts of text.
- Remain accessible to keyboard and screen-reader users.
- Update correctly after errors are resolved.

---

## 19. Accessibility Requirements

Validation must comply with KAMPYN's accessibility standards.

- Associate every field with a programmatic label.
- Associate validation messages with the relevant control.
- Set `aria-invalid` appropriately.
- Use `aria-describedby` or an equivalent association for field instructions and errors.
- Ensure errors are not communicated by color alone.
- Preserve keyboard accessibility.
- Move focus to an error summary or invalid field only when appropriate.
- Announce asynchronous validation results accessibly.
- Avoid announcing the same error repeatedly on every keystroke.
- Ensure validation messages remain visible and readable at supported zoom levels.

Example:

```tsx
<label htmlFor="email">Email address</label>

<input
  id="email"
  type="email"
  aria-invalid={Boolean(error)}
  aria-describedby={error ? "email-error" : undefined}
/>

{error && (
  <p id="email-error" role="alert">
    {error}
  </p>
)}
```

Use the project's shared form components where available to ensure consistent accessibility behavior.

---

## 20. Validation Performance

Validation must not introduce avoidable latency or rendering overhead.

- Prefer simple, deterministic schema rules.
- Avoid repeatedly parsing large objects on every keystroke.
- Validate at appropriate interaction points.
- Scope form subscriptions to the fields that need updates.
- Avoid unnecessary revalidation of unaffected fields.
- Debounce expensive asynchronous checks.
- Avoid blocking the main thread with expensive synchronous processing.
- Keep schemas cohesive and reasonably sized.
- Profile complex validation paths before introducing optimization.

Do not weaken required validation rules simply to improve rendering performance.

For large imported datasets or complex client-side validation, consider chunking or moving independent CPU-intensive work off the main thread where justified.

---

## 21. Internationalization and Localization

Validation messages must support KAMPYN's localization strategy.

- Avoid scattering hardcoded messages throughout reusable components.
- Use centralized message definitions or translation keys where localization is supported.
- Keep schema validation rules independent of presentation language where practical.
- Translate messages at the appropriate UI boundary.
- Ensure date, time, number, and currency formatting respects the applicable locale.
- Avoid parsing localized display strings as canonical values without explicit conversion rules.
- Ensure translated messages remain accessible and actionable.

Validation codes or structured issues may be preferable to embedding presentation-specific text in shared contracts.

---

## 22. Validation Testing

Validation schemas and form behavior must be tested independently and through relevant user workflows.

### 22.1 Schema unit tests

Every important reusable schema should have tests for:

- Valid input.
- Missing required fields.
- Invalid data types.
- Empty values.
- Boundary lengths.
- Boundary numeric values.
- Invalid formats.
- Optional and nullable behavior.
- Cross-field constraints.
- Transformation behavior.
- Unexpected input structures.

Test both accepted and rejected values.

### 22.2 Form tests

Test:

- Field-level errors.
- Submission blocking on invalid data.
- Successful submission with valid data.
- Error clearing after correction.
- Validation timing.
- Pending and disabled states.
- Duplicate-submission prevention.
- Backend validation error mapping.
- Accessible labels and error associations.
- Keyboard interaction.

### 22.3 API boundary tests

Test:

- Valid response parsing.
- Malformed response handling.
- Missing required response fields.
- Unexpected values.
- Compatibility with supported response versions.
- Safe behavior when parsing fails.
- Appropriate query or mutation error states.

### 22.4 Integration and end-to-end tests

Critical workflows should include validation coverage for:

- Signup and login.
- Profile updates.
- Food ordering and checkout.
- Payment initiation and confirmation states.
- Hostel and guest-house bookings.
- Shuttle bookings.
- Complaint submissions.
- Inventory updates.
- Community messaging and reporting.
- Administrative forms.

Tests must confirm that invalid client data is rejected by the UI and that backend failures are handled safely.

### 22.5 Property-based testing

Property-based testing may be used for complex schemas, transformations, and boundary-sensitive validation.

It is especially useful when input combinations are too numerous for a small set of hand-written examples.

Use it when it adds meaningful confidence, not as a mandatory dependency for every simple field.

---

## 23. Validation Anti-Patterns

The following practices are prohibited unless a documented exception exists.

- Treating TypeScript types as runtime validation.
- Using `any` to bypass schema validation.
- Maintaining duplicate schemas for the same contract without justification.
- Reimplementing the same rules independently across components.
- Trusting client-side validation as a security boundary.
- Trusting client-provided identity, tenant, price, or permission values.
- Parsing untrusted API responses using type assertions alone.
- Silently accepting malformed data.
- Displaying raw internal errors to end users.
- Performing expensive validation on every keystroke without need.
- Repeatedly validating unchanged trusted data.
- Using permissive coercion that hides malformed inputs.
- Silently modifying meaningful user input.
- Assuming that a valid booking request means availability is guaranteed.
- Treating client-side payment state as authoritative.
- Using browser storage as a trusted source.
- Failing to handle stale asynchronous validation responses.
- Implementing remote validation without race-condition handling.
- Ignoring accessibility in validation feedback.
- Duplicating backend business rules in the frontend as if they were authoritative.
- Making schema changes without updating relevant tests and contracts.
- Weakening validation to suppress errors instead of correcting the underlying contract.

---

## 24. Documentation and Schema Governance

Validation schemas are part of the frontend's application contract and must be maintained accordingly.

For each important schema:

- Document its domain responsibility.
- Identify its owning feature.
- Define the expected input and output.
- Document meaningful transformations.
- Identify backend contract dependencies.
- Record compatibility assumptions where relevant.
- Add tests for important constraints.
- Update related form and API integrations when it changes.

When a schema changes:

1. Identify all consumers.
2. Review API contract compatibility.
3. Update derived types and related code.
4. Update tests and fixtures.
5. Verify affected user workflows.
6. Check for breaking changes to persisted data or shared interfaces.
7. Update relevant documentation.

Avoid creating a universal schema module that becomes a dumping ground for unrelated feature rules.

---

## 25. Code Quality Standards

- Keep schemas small, cohesive, and domain-oriented.
- Prefer composition over repeated definitions.
- Use descriptive schema and type names.
- Keep parsing and error mapping in appropriate boundaries.
- Avoid embedding business workflows inside validation schemas.
- Avoid side effects during validation.
- Keep validation deterministic wherever possible.
- Use explicit transformations.
- Avoid unnecessary custom validation helpers.
- Reuse shared primitives only when the rules genuinely match.
- Keep production source files within the project's normal 200-line limit unless architectural justification is documented.
- Ensure validation changes pass TypeScript, linting, and relevant test checks.

Validation schemas must not become a replacement for domain services or API contracts.

---

## 26. Validation Review Checklist

### Schema design
- [ ] Zod is used for runtime validation where required.
- [ ] Schemas have clear ownership and naming.
- [ ] Types are derived from schemas where practical.
- [ ] Required, optional, and nullable fields are intentional.
- [ ] Constraints reflect documented contracts.
- [ ] Transformations are explicit and safe.
- [ ] Shared rules are reused appropriately.

### Forms
- [ ] Form validation timing is appropriate.
- [ ] Field-level and cross-field rules are implemented.
- [ ] Errors are clear and actionable.
- [ ] Submission is blocked for invalid data.
- [ ] Duplicate submissions are prevented.
- [ ] Backend validation errors are handled.
- [ ] Form state is preserved safely where appropriate.

### Runtime boundaries
- [ ] API responses are validated where required.
- [ ] URL parameters are validated.
- [ ] Browser storage is treated as untrusted.
- [ ] Environment variables are validated.
- [ ] Realtime payloads are validated at the appropriate boundary.
- [ ] Unknown external data is not blindly cast.

### Security
- [ ] Backend validation remains authoritative.
- [ ] Authorization is enforced by the backend.
- [ ] Tenant and identity values are not trusted from the client.
- [ ] Sensitive values are not exposed in validation messages.
- [ ] Redirect destinations are checked.
- [ ] File uploads are independently validated by the backend.

### Accessibility and performance
- [ ] Inputs have accessible labels.
- [ ] Errors are programmatically associated with fields.
- [ ] Keyboard and screen-reader behavior is supported.
- [ ] Validation does not cause unnecessary rerenders.
- [ ] Expensive asynchronous checks are controlled.
- [ ] Stale validation responses cannot overwrite current state.

### Testing and maintenance
- [ ] Schema unit tests cover valid and invalid cases.
- [ ] Form tests cover submission and error behavior.
- [ ] API boundary tests cover malformed data.
- [ ] Critical workflows have integration or end-to-end coverage.
- [ ] Contract changes are documented.
- [ ] TypeScript and lint checks pass.

---

## 27. Definition of Done

A frontend feature is not complete until:

1. All relevant user inputs have explicit validation rules.
2. Zod schemas are defined or reused at the appropriate boundaries.
3. TypeScript types are consistent with validated data.
4. Validation feedback is clear, actionable, and accessible.
5. Invalid form data cannot be submitted through the intended UI flow.
6. API request and response boundaries are handled safely.
7. Backend validation and authorization remain authoritative.
8. Sensitive and tenant-specific values are never trusted solely because they passed frontend validation.
9. Asynchronous validation handles failures, cancellation, and stale responses.
10. Validation does not introduce unnecessary rendering or network overhead.
11. Relevant schema, form, and integration tests pass.
12. Contract changes and compatibility assumptions are documented.
13. No malformed or untrusted data is silently treated as trusted application state.

---

## 28. Guiding Principle

Frontend validation exists to help users submit correct information and to protect the application from malformed runtime data. It must remain predictable, accessible, reusable, and consistent with KAMPYN's contracts.

Zod provides runtime validation, TypeScript provides compile-time guarantees, and the backend enforces authoritative business rules, data integrity, and security.

**Validate at boundaries, guide users clearly, trust no unverified runtime data, and always enforce critical rules on the backend.**
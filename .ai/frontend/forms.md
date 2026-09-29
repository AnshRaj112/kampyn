# Frontend Forms

## Purpose

This document defines the engineering standards for building, validating, managing, submitting, and testing forms across KAMPYN.

Forms are a critical interface between users and the platform. They handle authentication, food ordering, bookings, payments, complaints, university administration, HR workflows, inventory management, and other operations where incorrect or incomplete input can affect business operations or user data.

Every form must be:

- **Correct:** Prevent invalid data from reaching business logic.
- **Accessible:** Usable with keyboards, screen readers, and assistive technologies.
- **Predictable:** Provide clear validation, submission, and error states.
- **Secure:** Treat all client input as untrusted.
- **Maintainable:** Use reusable patterns without introducing unnecessary abstractions.
- **Performant:** Avoid excessive re-renders, unnecessary validation, and duplicated state.
- **Consistent:** Follow shared design-system, validation, and interaction standards.

This document is the source of truth for frontend form engineering. It complements the component, accessibility, state-management, data-fetching, security, and testing policies.

---

## 1. Core Principles

All forms must follow these principles:

1. Use **React Hook Form** for complex or multi-field forms where it provides meaningful benefits.
2. Use controlled React state for small, simple forms when a form library would add unnecessary complexity.
3. Use **Zod** for runtime validation and typed form schemas.
4. Reuse shared validation schemas when the same business rules apply across multiple interfaces.
5. Treat frontend validation as a usability feature, never as a security boundary.
6. Validate data on the backend independently of frontend validation.
7. Keep form state local unless there is a clear requirement to share it.
8. Use TanStack Query for server interactions and server-state synchronization, not as a replacement for local form state.
9. Use Zustand only when form-related state genuinely needs to be shared across distant components or workflows.
10. Display validation errors close to the relevant fields and provide actionable feedback.
11. Prevent duplicate submissions and handle retries safely.
12. Preserve user input when recoverable errors occur.
13. Do not expose sensitive information in URLs, logs, analytics, or browser storage.
14. Build forms from accessible, reusable components and consistent design tokens.
15. Keep form components cohesive, small, and testable.

Do not introduce a form library, schema abstraction, custom hook, or shared component without a concrete need.

---

## 2. Form Technology

### 2.1 React Hook Form

Use React Hook Form for forms that have multiple fields, nested data, conditional sections, complex validation, dynamic arrays, or workflows that benefit from controlled submission and field registration.

Typical use cases include:

- User registration and profile management
- Vendor onboarding
- Food court and menu configuration
- Hostel and guest house bookings
- HR and university administration
- Inventory reporting
- Complaints and support requests
- Multi-step forms
- Complex search and filter forms

Use the library's registration and validation mechanisms rather than manually maintaining state for every field.

### 2.2 Simple React State

For small forms with very limited interaction, local React state may be sufficient.

Examples include:

- A simple search field
- A single feedback input
- A lightweight inline edit
- A small filter with one or two values

Do not introduce React Hook Form solely for a single input if native state is simpler and equally maintainable.

### 2.3 Zod

Use Zod to define runtime validation schemas and infer TypeScript types from those schemas.

```ts
import { z } from "zod";

export const profileSchema = z.object({
  fullName: z.string().trim().min(2).max(100),
  email: z.email(),
});

export type ProfileFormValues = z.infer<typeof profileSchema>;
```

Use the project's established Zod version and compatible syntax. Keep schema definitions aligned with the validation conventions in the repository.

When using React Hook Form, integrate Zod through the project's approved resolver.

```ts
const form = useForm<ProfileFormValues>({
  resolver: zodResolver(profileSchema),
  defaultValues: {
    fullName: "",
    email: "",
  },
});
```

Do not duplicate the same validation rules in field components, submit handlers, and schema definitions.

---

## 3. Form Architecture

Forms should follow a clear separation of responsibilities.

### 3.1 Recommended Structure

```text
src/
├── components/
│   └── ui/
│       ├── input/
│       ├── textarea/
│       ├── select/
│       ├── checkbox/
│       ├── radio-group/
│       ├── date-picker/
│       └── form-field/
│
├── features/
│   └── bookings/
│       ├── components/
│       │   └── BookingForm.tsx
│       ├── schemas/
│       │   └── booking.schema.ts
│       ├── hooks/
│       │   └── useBookingForm.ts
│       ├── services/
│       │   └── booking.service.ts
│       └── types/
│           └── booking.types.ts
│
└── lib/
    └── validation/
        ├── common.schema.ts
        └── validation.utils.ts
```

This structure is illustrative. Follow the existing feature organization in the repository rather than creating empty directories or splitting every small form into multiple files.

### 3.2 Responsibility Boundaries

| Layer | Responsibility |
|---|---|
| Form component | Layout, field composition, interaction, submission presentation |
| UI field component | Accessible input behavior, labels, descriptions, visual states |
| Zod schema | Input shape and client-side validation rules |
| Form hook | Reusable form configuration or workflow-specific form logic |
| Service/API client | Communicate with backend endpoints |
| TanStack Query | Mutation lifecycle, cache invalidation, server-state synchronization |
| Backend | Authoritative validation, authorization, business rules, persistence |

Form components must not directly implement database logic, authorization decisions, or business-critical calculations.

### 3.3 Feature Ownership

Keep feature-specific schemas and form logic within their owning feature unless they are genuinely shared.

Shared schemas belong in a shared validation module only when multiple features rely on the same rules and changes should be coordinated.

Do not create a universal form abstraction that attempts to represent every form in KAMPYN.

---

## 4. Schema Design and Validation

### 4.1 Schema-First Validation

Define validation rules in Zod schemas and infer TypeScript types wherever practical.

```ts
import { z } from "zod";

export const complaintSchema = z.object({
  category: z.enum(["food", "hostel", "transport", "other"]),
  description: z.string().trim().min(10).max(2000),
});

export type ComplaintFormValues = z.infer<
  typeof complaintSchema
>;
```

Schemas should be:

- Explicit about required and optional fields.
- Strict about accepted data types.
- Clear about minimum and maximum lengths.
- Consistent with backend contracts.
- Designed around business requirements rather than UI implementation details.
- Easy to test independently.

### 4.2 Required and Optional Fields

Required fields must be explicit. Optional values must have clearly defined semantics.

Distinguish between:

- A missing field
- An empty string
- `null`
- An omitted optional value
- A value that is present but invalid

Do not silently convert invalid values into valid defaults unless the business contract explicitly allows it.

Avoid making a field optional simply because the frontend currently does not collect it.

### 4.3 Normalization

Normalize input only when it is safe and consistent with the business contract.

Examples include:

- Trimming leading and trailing whitespace.
- Normalizing case for case-insensitive identifiers where appropriate.
- Removing formatting characters from phone numbers when required.
- Converting user-facing numeric input into a defined numeric representation.

Do not alter user input in ways that change its meaning.

Never normalize passwords by trimming, changing case, or otherwise modifying them unless the authentication contract explicitly requires it.

### 4.4 Cross-Field Validation

Use schema-level validation for rules that depend on multiple fields.

Examples include:

- Booking end date must be after the start date.
- Password confirmation must match the password.
- Quantity must not exceed a permitted limit.
- A conditional field becomes required when a specific option is selected.
- Check-out must follow check-in.

Cross-field errors should be associated with the relevant field or clearly presented at the form level.

### 4.5 Conditional Validation

Conditional fields must be validated according to the selected workflow.

For example, if a user selects a particular complaint category that requires a reference number, that reference number must be required for that category.

When a conditional field becomes hidden:

- Define whether its value should be cleared, retained, or ignored.
- Prevent stale hidden values from being submitted unintentionally.
- Ensure the schema reflects the active form state.
- Keep keyboard focus and screen-reader behavior predictable.

Do not rely only on hiding a field to exclude its value from submission.

### 4.6 Validation Timing

Choose validation timing according to the interaction.

| Situation | Preferred behavior |
|---|---|
| Initial form load | Avoid showing errors before the user interacts |
| Field blur | Validate fields where early feedback is useful |
| Field change | Revalidate corrected fields where appropriate |
| Submit | Validate all applicable fields |
| Server rejection | Map relevant errors to fields or show a form-level message |

Avoid aggressive validation that interrupts typing or repeatedly announces errors to screen readers.

### 4.7 Validation Messages

Validation messages must be:

- Human-readable
- Specific to the error
- Actionable where possible
- Free from internal implementation details
- Consistent across the application

Prefer:

- “Enter a valid email address.”
- “The description must contain at least 10 characters.”
- “Choose a check-out date after your check-in date.”

Avoid:

- “Invalid input.”
- “ZodError.”
- “Validation failed with code 400.”
- Raw stack traces or backend exception messages.

Do not reveal whether a sensitive account identifier exists during authentication or recovery flows.

---

## 5. Form State Management

### 5.1 Local State by Default

Form state belongs close to the form.

Use React Hook Form or local component state for:

- Current field values
- Dirty and touched state
- Client validation errors
- Submission state
- Conditional field visibility
- Step-specific values

Do not store ordinary form values in Zustand or a global context without a concrete cross-component requirement.

### 5.2 Server State

Use TanStack Query for:

- Loading selectable options from the server
- Submitting mutations
- Tracking mutation state
- Invalidating relevant queries
- Refreshing server-derived information

Do not duplicate server data in local form state unless the user is intentionally editing a local draft.

When editing server data, initialize the form with a deliberate mapping from the response to form values. Avoid continuously resetting the form whenever a background query refreshes, as this can overwrite unsaved edits.

### 5.3 Derived State

Do not store values that can be calculated from existing form values.

For example, totals, validation flags, conditional visibility, and formatted summaries should be derived where possible.

Keep calculations pure, predictable, and consistent with the backend's authoritative calculations.

### 5.4 Multi-Step Forms

For multi-step workflows:

- Define a clear step sequence and completion criteria.
- Keep the form's source of truth consistent across steps.
- Validate the current step before allowing progression.
- Validate the complete payload before final submission.
- Preserve entered values when navigating backward.
- Restore drafts only when the workflow explicitly supports it.
- Provide progress indicators and clear navigation controls.
- Prevent submission from an incomplete or invalid step.
- Define what happens when a user refreshes, leaves, or loses connectivity.

Do not persist sensitive form values to local storage or session storage by default.

### 5.5 Reset Behavior

Reset forms only in response to deliberate user actions or well-defined workflow transitions.

Examples include:

- Successful creation followed by a new-entry workflow.
- Explicit cancellation.
- Selecting a different record to edit.
- A deliberate reset action.

Do not clear a form automatically after a failed submission.

Warn users before discarding meaningful unsaved work when the loss could be consequential.

---

## 6. Reusable Form Components

### 6.1 Shared Field Components

Build shared field components when they establish consistent behavior, accessibility, styling, or error handling.

Examples include:

- `FormField`
- `TextInput`
- `Textarea`
- `Select`
- `Checkbox`
- `RadioGroup`
- `DatePicker`
- `FileUpload`
- `PasswordInput`
- `FormError`
- `FormDescription`

Shared components should integrate with the established design system and support controlled or form-library registration patterns where required.

### 6.2 Field Responsibilities

A field component may own:

- Label rendering
- Input rendering
- Required indication
- Description text
- Error presentation
- Disabled and read-only states
- Accessibility attributes
- Styling and variants

It should not own:

- API requests
- Business validation
- Authorization
- Feature-specific persistence
- Unrelated global state
- Complex multi-field workflows

### 6.3 Component APIs

Use explicit, strongly typed props.

```ts
type FormFieldProps = {
  id: string;
  label: string;
  description?: string;
  error?: string;
  required?: boolean;
  children: React.ReactNode;
};
```

Avoid overly generic props, excessive boolean combinations, and APIs that expose internal implementation details.

Use composition where it makes the field clearer and easier to maintain.

### 6.4 File Size and Cohesion

Follow the repository's general rule that production source files should normally remain at or below 200 lines.

Split a form when responsibilities naturally separate, such as:

- Schema and form UI
- Field groups
- Multi-step sections
- Reusable validation utilities
- Feature-specific submission logic

Do not split a simple form into many files solely to meet a line count. Any exception must have a clear architectural justification.

---

## 7. Accessibility

All forms must follow the project's accessibility policy and target WCAG 2.2 AA.

### 7.1 Labels

Every input must have a programmatically associated label.

Use visible labels for most forms. Placeholder text is not a replacement for a label.

Labels must clearly describe the expected input.

### 7.2 Descriptions and Errors

Associate help text and validation errors with their corresponding controls using appropriate accessibility attributes such as:

- `aria-describedby`
- `aria-invalid`
- `aria-required` when applicable

Ensure IDs are unique within the rendered document.

Do not rely on color alone to communicate an error.

### 7.3 Keyboard Support

Every form must be operable using a keyboard.

Ensure:

- Logical tab order
- Visible focus indicators
- Keyboard-operable custom controls
- No keyboard traps
- Predictable focus after validation or submission
- Accessible operation of date pickers, menus, and dialogs

### 7.4 Error Announcements

Validation errors must be available to assistive technologies.

Use appropriate live-region behavior for submission summaries and asynchronous server errors. Avoid announcing every keystroke or repeatedly reading unchanged messages.

After a failed submission, consider focusing the first invalid field or an accessible error summary, depending on form complexity.

### 7.5 Grouping

Use semantic grouping for related fields.

Examples include:

- Address details
- Booking dates
- Contact information
- Payment details
- Notification preferences

Use `fieldset` and `legend` where they provide meaningful grouping, especially for radio buttons and related choices.

### 7.6 Accessible Submission Feedback

Loading, success, and failure states must be perceivable without relying only on visual changes.

Buttons must communicate their action and disabled state clearly. Submission status should be announced appropriately when asynchronous.

---

## 8. Submission Lifecycle

Every form submission must follow a predictable lifecycle.

### 8.1 Standard Lifecycle

1. User submits the form.
2. Client-side validation runs.
3. Invalid fields display relevant feedback.
4. A valid payload is mapped to the API contract.
5. The submit action is triggered.
6. Duplicate submission is prevented where appropriate.
7. The backend validates authorization, input, and business rules.
8. The client handles success, validation failure, conflict, network failure, or unknown outcomes.
9. The UI updates or preserves the user's work based on the outcome.

### 8.2 Submission State

Represent the submission lifecycle clearly:

- Idle
- Validating
- Submitting
- Success
- Validation error
- Conflict
- Recoverable failure
- Unrecoverable failure, where applicable

Use the state-management capabilities of the selected form and mutation libraries instead of duplicating submission flags unnecessarily.

### 8.3 Duplicate Submission

Prevent accidental duplicate submissions through appropriate UI state and backend safeguards.

- Disable or otherwise guard the submit action while an equivalent request is in flight.
- Provide clear progress feedback.
- Use idempotency keys for operations where duplicate execution could cause financial, booking, inventory, or other business impact.
- Do not assume disabling a button guarantees exactly-once execution.

### 8.4 Error Handling

Handle API errors according to their meaning.

| Error category | Expected behavior |
|---|---|
| Field validation | Display actionable field-level errors |
| Business rule rejection | Explain the constraint without exposing internal details |
| Authentication failure | Follow the authentication flow |
| Authorization failure | Display an appropriate access message |
| Conflict | Explain the conflict and provide a recovery path |
| Rate limit | Communicate that the action is temporarily limited |
| Network failure | Preserve input and allow a safe retry |
| Server failure | Show a generic message and preserve user work |
| Unknown outcome | Avoid unsafe resubmission for non-idempotent actions |

Never display raw backend exceptions, database errors, tokens, internal service names, or stack traces.

### 8.5 Server Validation

Backend validation is mandatory even when a form uses Zod.

The server must independently validate:

- Payload structure and types
- Required fields and limits
- Business rules
- Tenant ownership
- Authentication and authorization
- Resource availability
- State transitions
- Idempotency and concurrency requirements

Frontend and backend schemas may share definitions where the architecture supports it, but this must not create an unsafe dependency or weaken backend ownership of validation.

---

## 9. API Contract and Payload Handling

### 9.1 Explicit Mapping

Map form values to API payloads explicitly when form shape and API shape differ.

Do not submit arbitrary component state or the entire server response as a mutation payload.

### 9.2 Runtime Validation

Validate external data at runtime where required, especially when populating forms from API responses or URL parameters.

Treat server responses as untrusted at the UI boundary.

### 9.3 Avoid UI-Coupled Contracts

Do not let presentation-specific details leak into API contracts.

For example, formatted currency strings, localized dates, display labels, and component state flags should be converted to the appropriate API representation before submission.

### 9.4 Payload Minimization

Submit only fields required by the operation.

Do not send hidden values, stale conditional fields, unnecessary personal information, or server-owned fields such as authoritative prices, permissions, tenant ownership, and calculated totals.

---

## 10. Security and Privacy

All user-entered form data must be treated as untrusted.

### 10.1 Input Security

- Never rely on frontend validation to prevent malicious input.
- Avoid unsafe HTML rendering of user-provided values.
- Do not use client-supplied tenant IDs or ownership fields as proof of authorization.
- Validate file types, sizes, and content on the server.
- Enforce authorization at the API boundary.
- Apply appropriate CSRF protections where relevant to the authentication architecture.
- Use secure transport for sensitive form submissions.

### 10.2 Sensitive Fields

Sensitive information includes passwords, authentication codes, payment information, private contact data, health-related data, and confidential university or HR records.

For sensitive fields:

- Do not log values.
- Do not expose them in error messages.
- Do not place them in URLs.
- Do not persist them in browser storage without an approved security design.
- Clear them when the workflow requires it.
- Limit their visibility to authorized users and services.

### 10.3 Authentication Forms

Authentication and account-recovery forms must avoid revealing sensitive account existence or internal authentication state.

Use secure password input behavior, suitable autocomplete attributes, clear feedback, and safe handling of verification codes.

Do not implement authentication rules solely in the frontend.

### 10.4 File Uploads

File upload forms must:

- Enforce client-side file size and type guidance.
- Validate file contents and actual type on the server.
- Use secure upload authorization.
- Avoid trusting filenames or MIME types supplied by the browser.
- Show upload progress and failure states where applicable.
- Prevent accidental duplicate uploads when harmful.
- Define retention and deletion behavior for uploaded content.
- Avoid exposing private object-storage URLs.

---

## 11. Domain-Specific Form Requirements

KAMPYN contains workflows where invalid input can affect other users or campus operations. Forms must reflect their business consequences.

### 11.1 Food Ordering

- Validate item selection and quantity limits.
- Clearly present selected options and modifiers.
- Reconfirm availability and pricing on the backend.
- Do not treat client-calculated totals as authoritative.
- Handle menu or inventory changes between selection and submission.
- Use idempotent order creation where appropriate.
- Preserve the cart or provide recovery feedback after recoverable failures.

### 11.2 Bookings

Applies to hostel, guest house, washing machine, shuttle, and other reservation workflows.

- Validate date and time ranges.
- Display timezone assumptions where relevant.
- Check availability on the backend at submission time.
- Handle conflicting bookings and stale availability.
- Clearly communicate cancellation or modification constraints.
- Prevent duplicate reservations through backend idempotency and integrity controls.

### 11.3 Complaints and Support

- Use clear categories and descriptions.
- Define length limits and required contextual details.
- Support attachments only through approved secure upload flows.
- Avoid exposing private complaint information to unauthorized users.
- Communicate submission status and reference identifiers safely.

### 11.4 HR and Administration

- Enforce role-based access on the backend.
- Separate editable fields from server-controlled or privileged fields.
- Require explicit confirmation for consequential changes.
- Record audit information on the server.
- Avoid retaining confidential information in browser storage or telemetry.

### 11.5 Inventory and Menu Management

- Validate quantities, units, identifiers, and allowed state transitions.
- Handle concurrent edits and stale data.
- Recheck stock or menu constraints on the backend.
- Communicate conflicts rather than silently overwriting newer values.
- Use clear confirmation for bulk or destructive changes.

---

## 12. Async Data and Dynamic Fields

Forms may depend on asynchronous server data such as campus locations, food courts, menu items, rooms, time slots, departments, or user roles.

### 12.1 Loading States

- Display a meaningful loading state for dependent fields.
- Prevent invalid selection while required options are unavailable.
- Distinguish between loading, empty, and failed states.
- Avoid presenting stale values as confirmed current availability.

### 12.2 Dependent Fields

When one field controls another:

- Reset or reconcile dependent values when the parent changes.
- Prevent submission of incompatible combinations.
- Handle asynchronous response ordering safely.
- Avoid unnecessary requests and repeated option loading.
- Validate the final combination on the backend.

### 12.3 Dynamic Arrays

For repeatable field groups:

- Use stable identifiers for rendered items.
- Support adding, removing, and reordering where appropriate.
- Validate item-level and collection-level constraints.
- Preserve focus and accessible labels when the collection changes.
- Avoid using array indexes as identity when items can be reordered or removed.

---

## 13. Date, Time, Number, and Currency Inputs

### 13.1 Dates and Time

- Define whether a value represents a calendar date, local date-time, or absolute timestamp.
- Handle timezone conversion explicitly.
- Avoid parsing ambiguous date strings.
- Validate chronological relationships.
- Revalidate availability on the backend for time-sensitive workflows.
- Display localized formats without changing the underlying contract.

### 13.2 Numbers

- Define permitted precision, range, and units.
- Avoid treating formatted strings as authoritative numeric values.
- Handle empty and partially entered values without unexpected coercion.
- Prevent invalid `NaN`, infinity, or out-of-range values from entering submission payloads.

### 13.3 Currency

- Use integer minor units or the approved precise representation for monetary values.
- Avoid floating-point arithmetic for authoritative financial calculations.
- Display currency and formatting consistently with the active locale.
- Treat backend-calculated prices, fees, taxes, discounts, and totals as authoritative.

---

## 14. Unsaved Changes and Drafts

Forms with meaningful user effort should define behavior for interrupted work.

- Warn before leaving when unsaved changes could be lost.
- Do not block navigation unnecessarily for trivial inputs.
- Save drafts only when explicitly supported by the feature.
- Define draft ownership, expiration, synchronization, and deletion.
- Avoid persisting sensitive values in local storage.
- Handle stale drafts and server-side changes.
- Make it clear when a draft has been saved versus when it exists only locally.

For multi-step workflows, communicate progress and whether the current step has been saved.

---

## 15. Performance

Forms should remain responsive on low-end devices and unreliable campus networks.

- Avoid unnecessary component re-renders.
- Subscribe to only the form state needed by each component.
- Use field-level subscriptions where supported.
- Avoid expensive validation on every keystroke for large forms.
- Debounce remote validation only where it is safe and useful.
- Cancel or ignore stale asynchronous validation results.
- Avoid loading large option lists unnecessarily.
- Use pagination, search, or virtualization for very large selectable datasets.
- Keep transformations and derived calculations efficient.
- Avoid introducing memoization without evidence of a performance issue.

Performance improvements must not compromise accessibility, correctness, or predictable form behavior.

---

## 16. Internationalization and Localization

KAMPYN may serve universities with different languages, regions, and formats.

Forms must:

- Keep user-facing labels and messages localization-ready.
- Avoid hardcoding locale-dependent date, number, and currency formats.
- Support Unicode input.
- Avoid assuming names, addresses, or identifiers follow one regional pattern.
- Preserve user-entered text unless normalization is explicitly required.
- Ensure translated labels and errors remain accessible and understandable.
- Avoid embedding translated text in shared validation logic when the application has a localization layer.

Validation should return stable error identifiers or structured issues where needed so the UI can present localized messages.

---

## 17. Testing

Forms require tests for validation, interaction, accessibility, API behavior, and failure recovery.

### 17.1 Schema Tests

Test:

- Valid and invalid inputs
- Required and optional fields
- Boundaries and maximum lengths
- Cross-field rules
- Conditional validation
- Empty and null values
- Normalization behavior
- Malformed external data

### 17.2 Component Tests

Test:

- Correct labels and accessible names
- Field descriptions and error associations
- User typing and selection
- Required and disabled states
- Touched and dirty behavior
- Conditional fields
- Keyboard interaction
- Error announcements where testable
- Reset and cancellation behavior

### 17.3 Submission Tests

Test:

- Valid submission
- Invalid submission
- Loading state
- Duplicate submission prevention
- Successful response
- Field-level server validation errors
- Authorization and business-rule rejection
- Network failure and retry
- Conflict handling
- Unknown outcome behavior for consequential operations
- Correct payload mapping

### 17.4 Workflow Tests

For multi-step and domain-specific forms, test:

- Step progression and backward navigation
- Preservation of values
- Step-specific validation
- Final payload validation
- Dynamic and dependent fields
- Draft recovery where supported
- Concurrent or stale availability handling

### 17.5 Accessibility Tests

Use automated accessibility testing as part of the test strategy, supplemented by manual keyboard and assistive-technology checks.

Automated tests do not replace manual verification of focus order, error announcements, and complex custom controls.

### 17.6 End-to-End Tests

Critical workflows should have end-to-end coverage, especially:

- Registration and authentication
- Food ordering
- Payments
- Hostel and guest house bookings
- Scheduling
- Complaints
- Administrative operations
- Inventory and menu changes

Tests must avoid using production personal data or real payment operations.

---

## 18. Observability and Analytics

Form telemetry must help identify failures without collecting unnecessary personal data.

Track suitable operational signals such as:

- Submission success and failure rates
- Validation failure categories
- API latency
- Abandonment at an aggregate level where approved
- Repeated conflict or availability failures
- Upload failure rates

Never log raw form values, passwords, verification codes, payment credentials, private complaint content, or confidential HR data.

Use approved analytics events and ensure consent, privacy, and retention requirements are respected.

---

## 19. Error Recovery and Resilience

Forms must be designed for unreliable connectivity, delayed responses, and backend state changes.

- Preserve user-entered values after recoverable errors.
- Clearly distinguish client validation from server rejection.
- Provide safe retry behavior.
- Avoid assuming a timed-out request was not processed.
- Use idempotency for consequential operations.
- Reconcile stale reference data before submission where necessary.
- Communicate when the user must refresh or review updated information.
- Avoid silent data loss when asynchronous updates arrive.

For high-impact workflows, define recovery behavior before implementation.

---

## 20. Anti-Patterns

The following are prohibited unless an explicit architectural exception is approved:

- Duplicating validation rules across components and handlers.
- Treating client-side validation as a security boundary.
- Storing all form values in global state by default.
- Using TanStack Query as a replacement for form state.
- Resetting user input automatically after a failed request.
- Submitting stale hidden or conditional values unintentionally.
- Using placeholder text instead of labels.
- Showing raw backend or database errors.
- Trusting client-provided prices, permissions, tenant IDs, or availability.
- Using floating-point arithmetic for authoritative monetary calculations.
- Making API calls directly inside low-level input components.
- Creating a universal form builder without a proven product requirement.
- Overusing controlled inputs where registration or native behavior is simpler.
- Triggering expensive validation on every keystroke without justification.
- Persisting sensitive form values in browser storage.
- Using array indexes as identity for reorderable dynamic fields.
- Allowing repeated financial or booking submissions without idempotency safeguards.
- Clearing fields on every background server refresh.
- Creating excessive custom hooks or wrappers that obscure standard form-library behavior.
- Ignoring keyboard navigation, focus management, or screen-reader feedback.

---

## 21. Documentation Requirements

Complex or consequential forms must document:

- Purpose and owning feature
- Schema and field constraints
- API contract and payload mapping
- Business rules and validation ownership
- Conditional field behavior
- Submission lifecycle
- Error and recovery behavior
- Sensitive data handling
- Draft and unsaved-change behavior
- Accessibility considerations
- Tests and known limitations

Documentation should be concise and located near the owning feature or in the relevant `.ai/` policy.

---

## 22. Form Review Checklist

### Architecture
- [ ] The form belongs to the correct feature.
- [ ] Responsibilities are separated appropriately.
- [ ] Shared abstractions are justified.
- [ ] The implementation follows existing repository conventions.
- [ ] Files remain cohesive and normally within the 200-line limit.

### Validation
- [ ] Zod schemas define applicable client validation.
- [ ] Types are inferred or otherwise kept in sync.
- [ ] Required and optional fields are explicit.
- [ ] Cross-field and conditional rules are covered.
- [ ] Validation messages are actionable.
- [ ] Backend validation remains authoritative.

### State and Submission
- [ ] Form state is local by default.
- [ ] Server state is managed with the approved data-fetching approach.
- [ ] Submission states are clear.
- [ ] Duplicate submissions are handled.
- [ ] API payloads are explicitly mapped.
- [ ] Recoverable failures preserve user input.
- [ ] Consequential operations use appropriate idempotency and conflict handling.

### Accessibility
- [ ] Every input has an accessible label.
- [ ] Descriptions and errors are programmatically associated.
- [ ] Keyboard navigation works.
- [ ] Focus behavior is predictable.
- [ ] Errors and submission status are perceivable to assistive technologies.
- [ ] Custom controls meet accessibility requirements.

### Security
- [ ] All input is treated as untrusted.
- [ ] No sensitive values are logged or exposed.
- [ ] Authorization is enforced by the backend.
- [ ] Client-provided business-critical values are not trusted.
- [ ] File uploads follow approved security requirements.
- [ ] Browser persistence is justified and safe.

### Quality
- [ ] Schema and component tests exist.
- [ ] Submission success and failure paths are tested.
- [ ] Critical workflows have end-to-end coverage.
- [ ] Dynamic and asynchronous fields handle stale data.
- [ ] Performance is acceptable for expected form size.
- [ ] Documentation is sufficient for future maintenance.

---

## 23. Definition of Done

A form is complete only when:

- Its structure and responsibilities follow the frontend architecture.
- Its fields use appropriate shared components and design tokens.
- Its Zod validation is explicit and tested.
- Its backend contract and business validation are independently enforced.
- Its state and submission lifecycle are predictable.
- Its errors are understandable and accessible.
- It preserves user work through recoverable failures.
- It protects sensitive information.
- It handles asynchronous data and stale state safely.
- Its performance is suitable for expected devices and workloads.
- Its tests cover normal, boundary, failure, and recovery paths.
- Its documentation and review checklist are complete where required.

**A form is not complete merely because it accepts input and submits a request. It is complete when users can enter, correct, submit, and recover their information safely, accessibly, and predictably.**
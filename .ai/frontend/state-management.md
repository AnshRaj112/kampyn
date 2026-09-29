# Frontend State Management

## Purpose

This document defines the standards for managing state across the KAMPYN frontend, built with Next.js, React, TypeScript, TanStack Query, Zustand, and Zod.

The goal is to ensure that state is predictable, consistent, maintainable, performant, secure, and scalable across KAMPYN's modules, including food ordering, payments, bookings, inventory, community, notifications, and administration.

State management must prioritize:
- Clear ownership of every piece of state.
- The simplest solution that satisfies the requirements.
- Separation of server state, client state, form state, and URL state.
- Minimal duplication and unnecessary synchronization.
- Strong typing and runtime validation at system boundaries.
- Tenant and user isolation.
- Predictable updates and explicit state transitions.
- Testability and maintainability.

State management must not become a substitute for sound component architecture, backend validation, database consistency, or domain design.

---

## 1. Core Principles

### 1.1 Assign One Source of Truth

Every piece of state must have a clearly defined source of truth.

- Server-owned data must remain authoritative on the backend.
- TanStack Query must manage cached server state on the client.
- Zustand must manage shared client-side state that genuinely needs a global or cross-component lifecycle.
- React component state must manage local UI state.
- Form libraries or controlled form state must manage form inputs and validation state.
- URL search parameters must represent shareable, navigable page state where appropriate.

Do not store the same authoritative value independently in multiple state systems.

### 1.2 Choose State by Ownership

Before introducing state, identify:
- Who owns the value?
- Where does it originate?
- Which components need it?
- How long must it live?
- Does it need persistence?
- Can it be derived?
- Is it sensitive?
- What invalidates or changes it?

Choose the state mechanism based on those answers rather than familiarity or convenience.

### 1.3 Prefer Derived State

Do not store values that can be calculated from existing state unless there is a measurable performance or lifecycle reason.

Avoid redundant state such as:
- Storing both a list and its item count.
- Storing a filtered list when it can be derived from the original list and filters.
- Storing `isEmpty` when it can be calculated from the collection.
- Duplicating a query result into Zustand without a specific requirement.
- Storing multiple boolean flags for a single workflow when one explicit status can represent it.

Derived values should be calculated close to where they are consumed, using memoization only when necessary.

### 1.4 Keep State Local by Default

State should live in the narrowest scope that satisfies its consumers.

Prefer:
1. Derived values.
2. Component-local state.
3. Shared state through props or composition.
4. URL state for navigable state.
5. TanStack Query for server state.
6. Zustand for genuinely shared client state.

Do not move state into a global store simply to avoid passing a small number of props.

### 1.5 Keep Business Rules on the Backend

Frontend state is for presentation, interaction, and client-side coordination.

The backend remains authoritative for:
- Authentication and authorization.
- Tenant access.
- Pricing and discounts.
- Order totals and payment status.
- Inventory availability.
- Booking eligibility and confirmation.
- Resource ownership.
- Moderation and reporting decisions.
- HR permissions and workflows.
- Any operation that changes business-critical records.

Frontend state must never be treated as proof that a business operation is valid or authorized.

---

## 2. State Classification

Every state variable must belong to an identifiable category.

| State category | Examples | Preferred mechanism |
|---|---|---|
| Server state | Menus, orders, bookings, inventory, user profile | TanStack Query |
| Global client state | Active cart draft, UI preferences, transient app-wide selections | Zustand |
| Local UI state | Modal visibility, expanded sections, active tab | React `useState` |
| Derived state | Filtered items, computed totals for display, item counts | Derived expressions or `useMemo` |
| Form state | Input values, touched fields, submission errors | Form library or controlled React state |
| URL state | Search query, filters, sorting, page number, selected category | URL search parameters |
| Persistent preference state | Theme, density, dismissed non-critical onboarding | Explicit persistence layer |
| Workflow state | Multi-step client-side interaction progress | Local state or a focused state machine |
| Real-time state | Presence, typing indicators, transient live updates | Dedicated real-time layer with controlled client state |
| Temporary optimistic state | Pending UI representation of an in-flight mutation | TanStack Query mutation/cache lifecycle |

A value must not be placed in multiple categories without an explicit synchronization contract.

---

## 3. TanStack Query — Server State

TanStack Query is the primary client-side mechanism for fetching, caching, synchronizing, and mutating server-owned data.

Use it for:
- Food court listings and menus.
- Search results returned by the backend.
- Orders and order history.
- Payment status.
- Hostel and guest house bookings.
- Washing machine schedules.
- Library vacancy information.
- Shuttle schedules and reservations.
- Inventory and stock availability.
- User and staff profiles.
- Community posts and conversations retrieved from the server.
- Notifications and read status.
- Administrative dashboards and reports.
- Any backend-owned entity displayed in the UI.

### 3.1 Do Not Duplicate Query Data

Do not copy TanStack Query results into Zustand or local state solely for convenient access.

Avoid:

```ts
const { data } = useQuery({
  queryKey: ["orders"],
  queryFn: getOrders,
});

const setOrders = useOrderStore((state) => state.setOrders);

useEffect(() => {
  if (data) {
    setOrders(data);
  }
}, [data, setOrders]);
```

This creates two client-side representations of the same server-owned data and introduces synchronization problems.

Prefer consuming the query result directly:

```ts
const { data: orders, isPending, isError } = useOrders();
```

If a user is editing server-owned data, maintain a separate edit draft with a clear save or discard lifecycle. Do not silently treat that draft as the authoritative server record.

### 3.2 Query Ownership

Each domain must define its own query functions, query keys, hooks, and mutation behavior.

Suggested structure:

```text
src/
├── features/
│   ├── orders/
│   │   ├── api/
│   │   │   ├── get-orders.ts
│   │   │   └── update-order.ts
│   │   ├── queries/
│   │   │   ├── order-keys.ts
│   │   │   ├── use-orders.ts
│   │   │   └── use-order.ts
│   │   ├── mutations/
│   │   │   └── use-update-order.ts
│   │   ├── components/
│   │   └── types/
│   └── bookings/
│       ├── api/
│       ├── queries/
│       ├── mutations/
│       ├── components/
│       └── types/
└── shared/
    └── query/
        ├── query-client.ts
        ├── query-provider.tsx
        └── query-defaults.ts
```

Feature modules must own domain-specific query logic. Shared query infrastructure must not become a repository for unrelated business logic.

### 3.3 Query Key Standards

Query keys must be:
- Deterministic.
- Structured.
- Typed where practical.
- Hierarchical.
- Specific to the resource and relevant parameters.
- Tenant-aware where data is tenant-scoped.
- User-aware where data is user-specific.

Example:

```ts
export const orderKeys = {
  all: ["orders"] as const,

  tenant: (tenantId: string) =>
    [...orderKeys.all, tenantId] as const,

  lists: (tenantId: string) =>
    [...orderKeys.tenant(tenantId), "list"] as const,

  list: (
    tenantId: string,
    filters: OrderFilters,
  ) => [...orderKeys.lists(tenantId), filters] as const,

  details: (tenantId: string) =>
    [...orderKeys.tenant(tenantId), "detail"] as const,

  detail: (tenantId: string, orderId: string) =>
    [...orderKeys.details(tenantId), orderId] as const,
};
```

Avoid vague keys such as:

```ts
["data"]
["list"]
["details"]
```

Do not use non-serializable values, functions, class instances, or unstable object references in query keys.

Query keys must include all parameters that can change the returned data, such as:
- Tenant ID.
- User ID, when the result is user-specific.
- Resource ID.
- Filters.
- Sorting.
- Pagination.
- Search query.
- Relevant locale or currency context, where the response varies by it.

Never rely on a query key as an authorization mechanism. Backend authorization remains mandatory.

### 3.4 Query Functions

Query functions must:
- Be independently testable.
- Use the shared API client.
- Accept an `AbortSignal` where supported.
- Validate untrusted response data at appropriate boundaries.
- Return typed values.
- Avoid hidden side effects.
- Avoid direct manipulation of unrelated caches.
- Normalize errors consistently.

Example:

```ts
export async function getOrder(
  orderId: string,
  signal?: AbortSignal,
): Promise<Order> {
  const response = await apiClient.get(
    `/orders/${orderId}`,
    { signal },
  );

  return orderSchema.parse(response.data);
}
```

Do not make query functions responsible for UI notifications, navigation, or unrelated state changes.

### 3.5 Query Lifecycle

Every query-driven screen must intentionally handle:
- Initial loading.
- Background fetching.
- Success with data.
- Success with no results.
- Recoverable errors.
- Authentication expiration.
- Tenant or permission changes.
- Refetching and stale data.
- Cancellation and unmounting.

Do not render an indefinite loading state for every background refetch. Where safe, retain usable stale data while refreshing.

Do not confuse an empty result with a failed request.

### 3.6 Freshness and Caching

Set `staleTime`, `gcTime`, retry behavior, and refetch behavior according to the domain's consistency requirements.

Illustrative defaults only:

| Data | Typical freshness strategy |
|---|---|
| Static configuration | Long stale time |
| Food court catalogue | Moderate stale time |
| Menu availability | Shorter stale time during service hours |
| User profile | Moderate stale time |
| Orders | Moderate stale time with mutation-driven invalidation |
| Payment status | Explicit verification or controlled polling |
| Inventory | Short stale time with backend enforcement |
| Booking availability | Short stale time; revalidate before confirmation |
| Presence | Real-time lifecycle rather than long-lived query caching |
| Analytics | Longer stale time where reporting delay is acceptable |

These are starting points, not universal fixed values. Each feature must document its consistency and freshness needs.

Never use frontend cache freshness as a guarantee that inventory, payments, or booking slots remain available.

### 3.7 Mutation Standards

Mutations must:
- Use domain-specific hooks.
- Validate inputs before submission where appropriate.
- Rely on backend validation and authorization.
- Provide pending, success, and error states.
- Prevent unintended duplicate submissions.
- Invalidate or reconcile affected queries.
- Handle cancellation and retries safely.
- Avoid exposing sensitive backend errors directly to users.

Example:

```ts
export function useCancelOrder() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: cancelOrder,

    onSuccess: (_, variables) => {
      void queryClient.invalidateQueries({
        queryKey: orderKeys.tenant(variables.tenantId),
      });
    },
  });
}
```

Mutation success must only be shown after the backend confirms the operation. A locally updated button or optimistic UI must not be represented as a completed payment, confirmed booking, or finalized order.

### 3.8 Cache Invalidation

Invalidation must be deliberate and scoped.

- Invalidate the smallest reliable query-key prefix.
- Reconcile related list and detail views.
- Avoid refetching the entire application after every mutation.
- Avoid broad cache clearing unless required by an identity or tenant lifecycle transition.
- Document cross-feature invalidation dependencies.

When one mutation affects multiple domains, define the affected query keys explicitly.

### 3.9 Optimistic Updates

Optimistic updates are permitted only when:
- The expected result is predictable.
- The action is reversible or failure can be reconciled.
- The UI can clearly represent pending state.
- Rollback behavior is implemented.
- Concurrent updates are handled safely.
- The backend remains authoritative.

Use extra caution for:
- Orders.
- Payments.
- Inventory.
- Bookings.
- Shuttle reservations.
- HR approvals.
- Moderation and reports.
- Any irreversible or financial operation.

Do not optimistically claim that a payment succeeded, stock was reserved, or a booking was confirmed.

For a suitable low-risk interaction, snapshot previous cache data, apply the temporary update, restore on failure, and reconcile with the backend response on success.

### 3.10 Retries and Offline Behavior

Retries must be based on operation semantics.

- Retry safe transient reads selectively.
- Avoid aggressive retry loops.
- Do not automatically retry non-idempotent writes without an idempotency strategy.
- Surface connectivity state when it materially affects user actions.
- Do not present locally queued work as server-confirmed.
- Define which user actions may be queued offline and how conflicts are resolved.

Payment, order submission, and reservation operations require backend-supported idempotency and explicit result verification before safe retry.

---

## 4. Zustand — Shared Client State

Zustand is used for shared client-side state that:
- Is not authoritative server data.
- Is required by multiple components or routes.
- Has a meaningful client-side lifecycle.
- Cannot be represented more clearly through local state, URL state, or TanStack Query.

Suitable examples:
- Active cart draft before submission.
- App-wide navigation drawer state, if genuinely shared.
- Temporary user interface preferences.
- Multi-step interaction state that spans multiple components.
- Selected campus resource context, if it is a client selection rather than a trusted authorization source.
- Client-side notification presentation state.
- Temporary draft composition state.

Zustand must not become a second server cache or a dumping ground for arbitrary frontend values.

### 4.1 Store Design

Stores must be:
- Small and cohesive.
- Named after a specific domain or responsibility.
- Strongly typed.
- Split by ownership and lifecycle.
- Designed with explicit actions.
- Free from backend business-rule duplication.

Suggested structure:

```text
src/
└── stores/
    ├── cart/
    │   ├── cart-store.ts
    │   ├── cart-types.ts
    │   └── cart-selectors.ts
    ├── ui/
    │   └── ui-store.ts
    └── preferences/
        └── preferences-store.ts
```

Feature-local stores may live inside the corresponding feature when they are not shared across the application.

Do not create one massive global store containing unrelated domains.

### 4.2 Store State and Actions

Keep state and actions explicit.

```ts
interface CartState {
  items: CartDraftItem[];
  addItem: (item: CartDraftItem) => void;
  removeItem: (itemId: string) => void;
  updateQuantity: (itemId: string, quantity: number) => void;
  clear: () => void;
}
```

Actions should:
- Have one clear responsibility.
- Preserve invariants of the client-side draft.
- Avoid unrelated side effects.
- Avoid direct network calls unless a well-defined architecture specifically requires it.
- Never bypass backend validation.

Prefer selectors for derived values instead of storing duplicate computed fields.

### 4.3 Selectors and Rendering

Subscribe components only to the state they need.

Prefer:

```ts
const itemCount = useCartStore(
  (state) => state.items.length,
);
```

Avoid subscribing a component to the entire store when it only uses one property.

Use stable selectors and shallow comparison where multiple selected values require it and the selected object would otherwise cause unnecessary renders.

Do not introduce memoization or selector abstractions without a concrete performance or reuse benefit.

### 4.4 Store Boundaries

Zustand must not own:
- Canonical orders or order history.
- Confirmed payments.
- Server-owned booking records.
- Canonical inventory.
- Authoritative user permissions.
- Server-side session validity.
- Tenant authorization decisions.
- Search result collections that are already managed by TanStack Query.

If a server-owned record is being edited, maintain a clearly scoped draft. On successful save, reconcile with the server response and update or invalidate the relevant query cache.

### 4.5 Cart Drafts

The cart may use Zustand for an active client-side draft.

Cart state may contain:
- Item identifiers.
- Selected options.
- Draft quantities.
- User-selected notes.
- Current food court or vendor context.
- Client-side grouping for presentation.

The backend must recalculate:
- Prices.
- Taxes.
- Discounts.
- Fees.
- Availability.
- Stock.
- Vendor eligibility.
- Final order totals.

A cart draft is not an order and must never be treated as proof that an order has been accepted.

When the tenant, user, or food court context changes, define whether the cart must be cleared, preserved as a separate draft, or revalidated. Never silently submit a cart under a different context.

### 4.6 Store Persistence

Persistence must be opt-in and justified.

Before persisting state, determine:
- Whether it is safe to store on the device.
- Whether it is necessary after reload.
- Whether it is scoped to a tenant or user.
- How stale values are invalidated.
- What happens on logout or account switching.
- Whether the value can be reconstructed from the backend.

Never persist:
- Access tokens or refresh tokens in ordinary Zustand persistence.
- Passwords or authentication secrets.
- Payment credentials.
- Sensitive personal or HR data without a specifically approved secure design.
- Authoritative permissions.
- Data that should be fetched afresh for security or consistency.

If persistence is necessary:
- Use a versioned schema.
- Validate persisted data at runtime.
- Handle malformed and outdated records.
- Scope keys appropriately.
- Clear user-scoped data on logout or identity changes.
- Avoid treating persisted data as trusted.

Persistence must not be enabled globally by default.

### 4.7 Zustand and Next.js

In Next.js, stores must be designed carefully to prevent cross-request state leakage.

- Do not use a shared mutable server singleton for user-specific state.
- Do not access browser-only storage during server rendering.
- Use per-provider or per-client store instances where the architecture requires request or tree isolation.
- Keep server components independent of browser-only Zustand APIs.
- Hydrate persisted state only through a deliberate client lifecycle.
- Prevent hydration mismatches caused by different server and client initial values.

Server-rendered output must not depend on untrusted or unavailable browser state.

---

## 5. React Local State

Use React state for values whose lifecycle belongs to one component or a small local subtree.

Examples:
- Modal open/closed state.
- Accordion expansion.
- Active tab in a local component.
- Temporary hover or focus presentation state.
- Local selection within a non-shareable widget.
- Small transient interaction state.

Use `useState` for straightforward values and `useReducer` when state transitions are related, multi-step, or easier to reason about as explicit actions.

Do not create a global store for state that is naturally local.

Do not duplicate props into local state unless the component intentionally creates an editable or independently managed draft.

When a prop change should reset local state, define that lifecycle explicitly rather than relying on incidental effects.

---

## 6. URL State

Use URL search parameters for state that should be:
- Shareable.
- Bookmarkable.
- Restored on refresh.
- Navigable through browser history.
- Accessible through deep links.

Examples:
- Search text.
- Filters.
- Sorting.
- Pagination.
- Selected catalogue category.
- Dashboard date range.
- Selected non-sensitive view mode.

URL state must be:
- Parsed and validated.
- Normalized consistently.
- Encoded using stable names.
- Compatible with back/forward navigation.
- Free of secrets and sensitive personal data.

Avoid keeping the same filter or search value independently in URL state, Zustand, and local state.

The URL should be the source of truth when the state is intentionally navigable. Components may derive their view state from the parsed URL.

Do not place authentication tokens, private identifiers, confidential search terms, or sensitive information in URLs.

---

## 7. Form State

Form state belongs to the form lifecycle and must not be managed through a global store by default.

Forms may use a suitable form library or controlled React state, according to complexity.

Form state includes:
- Current field values.
- Touched and dirty state.
- Client-side validation errors.
- Submission state.
- Field-level server errors.
- Multi-step progress, when part of the form.

Use Zod for runtime validation at appropriate boundaries and for reusable schemas where suitable.

### 7.1 Form Ownership

- Keep field state inside the form.
- Share form state only when multiple components genuinely participate in the same form.
- Do not copy form fields into Zustand merely to access them elsewhere.
- Keep unsaved edits distinct from persisted server data.
- Define reset, cancel, submit, and navigation behavior.

### 7.2 Validation

Client validation improves user experience but does not replace backend validation.

- Validate types and constraints before submission.
- Reuse schemas when the same contract applies.
- Avoid duplicating backend business rules as if client validation were authoritative.
- Map backend field errors into the form where possible.
- Present safe, actionable error messages.
- Avoid exposing internal exceptions or implementation details.

### 7.3 Long-Lived Drafts

If a form draft must survive route changes or reloads, define an explicit draft lifecycle and persistence policy.

Drafts involving sensitive personal information, HR records, private messages, or financial details require additional safeguards and must not be persisted casually.

---

## 8. State Machines and Complex Workflows

Use explicit state machines or reducer-based transitions when a workflow has:
- Multiple valid states.
- Restricted transitions.
- Asynchronous operations.
- Recovery or cancellation paths.
- Concurrent or dependent steps.
- Significant consequences if the UI enters an invalid state.

Examples:
- Multi-step onboarding.
- Complex booking flows.
- Payment initiation and result display.
- Moderation workflows.
- Multi-stage administrative actions.

Represent mutually exclusive states using a discriminated union rather than a collection of unrelated booleans.

Prefer:

```ts
type SubmissionState =
  | { status: "idle" }
  | { status: "submitting" }
  | { status: "success"; referenceId: string }
  | { status: "error"; message: string };
```

Avoid contradictory state such as `isLoading`, `isSuccess`, and `isError` all being independently settable.

For server-owned workflows, the backend remains the authoritative source of workflow status. A client state machine may coordinate the interaction but must reconcile with backend responses.

---

## 9. Authentication and Tenant Lifecycle

Authentication and tenant changes are critical state boundaries.

On login, logout, account switching, tenant switching, or session expiration:
- Update the authenticated application context through the approved authentication flow.
- Clear or isolate user-scoped cached data.
- Prevent previous-user data from rendering under the new identity.
- Clear sensitive transient client state.
- Reset or revalidate user-specific drafts according to policy.
- Cancel in-flight requests where appropriate.
- Ensure query keys are identity- and tenant-aware.
- Re-fetch data that must be verified for the new context.

Do not rely only on component unmounting to protect cached data.

### 9.1 Tenant Isolation

KAMPYN is a multi-tenant platform. Client state must respect tenant boundaries.

- Include tenant context in relevant query keys.
- Scope client-side drafts to their tenant context.
- Clear or isolate tenant-specific UI state on tenant switch.
- Never reuse tenant-specific data under another tenant.
- Do not infer tenant authorization from a locally selected tenant ID.
- Validate tenant access on the backend for every protected operation.

Tenant IDs and user IDs in the frontend are context identifiers, not security credentials.

### 9.2 Authentication State

Do not create competing authentication sources of truth across:
- React Context.
- Zustand.
- TanStack Query.
- Browser storage.
- Component-local state.

Use one approved authentication architecture. Expose only the minimal authentication context needed by the UI, and keep credential handling within the approved secure design.

Authentication state must not be reconstructed from untrusted client-side flags.

---

## 10. Real-Time and Event-Driven State

KAMPYN may use WebSockets, server-sent events, or other real-time mechanisms for:
- Community messages.
- Presence and typing indicators.
- Order status updates.
- Booking updates.
- Notifications.
- Shuttle updates.
- Administrative activity.

Real-time data must have explicit ownership and lifecycle rules.

### 10.1 Event Handling

Real-time event handlers must:
- Validate event shape.
- Verify the event belongs to the active tenant and relevant user context.
- Handle duplicate or out-of-order events.
- Avoid unbounded client-side accumulation.
- Reconcile events with TanStack Query or a dedicated transient client-state store.
- Clean up subscriptions when no longer required.

Do not treat an event arriving in the browser as proof of authorization or as the only durable record of a business operation.

### 10.2 Cache Reconciliation

Choose the appropriate update strategy:
- Update a known query cache entry when the event is authoritative, correctly scoped, and safely mergeable.
- Invalidate and refetch when the event cannot be reconciled reliably.
- Keep ephemeral state such as typing indicators outside durable server caches where practical.

Do not blindly append every event to every matching query. Avoid stale overwrites, duplicates, and race conditions between HTTP responses and real-time updates.

### 10.3 Reconnection

On connection loss and recovery:
- Restore subscriptions.
- Revalidate critical data.
- Handle missed events through backend-supported cursors or refetching.
- Prevent duplicate handlers.
- Expose connection status only where it helps the user.

For financial, booking, or order status, reconcile with the backend rather than assuming the client received every event.

---

## 11. State Persistence and Hydration

Persistence must have a documented reason and lifecycle.

### 11.1 Persistence Options

Use the narrowest suitable persistence mechanism:
- URL for navigable state.
- Server storage for account-level preferences or durable drafts.
- Browser storage for approved non-sensitive preferences.
- In-memory React or Zustand state for transient interactions.

Do not persist everything simply because a persistence library is available.

### 11.2 Versioning and Validation

Persisted client state is untrusted input.

- Version persisted schemas.
- Validate values before use.
- Handle corrupted data.
- Migrate only when migration is safe and deterministic.
- Drop incompatible or unsafe records.
- Avoid unbounded storage growth.
- Define cleanup behavior.

### 11.3 Hydration Safety

In Next.js:
- Ensure server and client initial render behavior is compatible.
- Avoid reading browser storage during server rendering.
- Avoid rendering sensitive or user-specific data before identity is established.
- Prevent hydration mismatch from persisted UI preferences.
- Use explicit loading or hydration boundaries when needed.

Persistence must never override a more authoritative server response.

---

## 12. State Performance

State management must support responsive interfaces without unnecessary complexity.

### 12.1 Rendering

- Subscribe components only to the state they consume.
- Keep local state close to the UI that uses it.
- Avoid broad context updates for frequently changing values.
- Avoid subscribing large component trees to high-frequency state.
- Split stores by responsibility.
- Use stable callbacks and memoization only where they provide measurable benefit.
- Avoid unnecessary deep cloning.
- Avoid storing large duplicated collections in client stores.

### 12.2 Large Collections

For large menus, orders, inventory, community feeds, and search results:
- Use backend pagination, filtering, and sorting where appropriate.
- Use TanStack Query for server-owned pages.
- Avoid loading entire datasets into Zustand.
- Use virtualization for genuinely large rendered lists.
- Keep only necessary client-side transient state.
- Avoid repeated O(n²) transformations in render paths.
- Prefer keyed maps or sets for repeated membership checks when appropriate.

Client-side optimization must not replace efficient backend query design.

### 12.3 High-Frequency State

For typing indicators, presence, scroll position, pointer movement, and other high-frequency updates:
- Keep subscriptions narrow.
- Avoid global store updates on every event.
- Throttle or debounce only when semantically appropriate.
- Avoid excessive persistence or logging.
- Clean up listeners and timers.

### 12.4 Measurement

Optimize based on evidence:
- React Profiler.
- Browser performance tooling.
- Query cache inspection.
- Network request analysis.
- Render count analysis.
- Real-user performance telemetry where available.

Do not add architectural complexity solely for hypothetical performance improvements.

---

## 13. State Security and Privacy

Frontend state may contain personal, financial, operational, or tenant-specific information.

All state handling must follow KAMPYN security and privacy policies.

- Treat all client state as untrusted.
- Never use frontend state as an authorization boundary.
- Do not store secrets in Zustand or ordinary browser storage.
- Do not expose sensitive values in query keys, URLs, logs, analytics, or error messages.
- Clear user-specific transient state at identity boundaries.
- Avoid retaining private data longer than required.
- Prevent cross-tenant cache and draft reuse.
- Avoid leaking data through shared component instances or server-side singletons.
- Validate externally sourced data before it enters trusted application logic.
- Restrict persisted state to approved data.

For community messages, HR records, reports, payment details, and other sensitive workflows, apply data minimization and access controls defined by the relevant security architecture.

---

## 14. Error Handling

State transitions must account for errors without leaving the UI inconsistent.

Handle:
- Network failures.
- Backend validation failures.
- Authorization failures.
- Tenant access changes.
- Session expiration.
- Request cancellation.
- Conflicts and stale updates.
- Rate limiting.
- Partial failures.
- Real-time disconnections.
- Persistence corruption.

Use consistent error mapping and user-safe messages.

Do not:
- Swallow errors silently.
- Treat every error as retryable.
- Keep the UI permanently in a pending state.
- Show raw backend stack traces.
- Mark a mutation successful before confirmation.
- Hide conflicts that require user action.

For critical operations, clearly distinguish:
- Not started.
- Pending.
- Confirmed.
- Failed.
- Unknown or awaiting verification.

An unknown payment or reservation result must be verified with the backend before presenting a definitive outcome.

---

## 15. Testing Requirements

State logic must be independently testable wherever practical.

### 15.1 Unit Tests

Test:
- Zustand actions and selectors.
- Reducer transitions.
- Derived state.
- Query key factories.
- Input normalization.
- Persistence validation and migration.
- Cache update and rollback helpers.
- State reset behavior.

### 15.2 Component Tests

Test:
- Loading, success, empty, and error states.
- User interactions.
- Form validation and submission.
- Mutation pending and failure behavior.
- Correct rendering after state changes.
- Modal and workflow lifecycle.
- Tenant and user context changes.

### 15.3 Integration Tests

Test:
- Query and mutation interactions.
- Cache invalidation and reconciliation.
- Authentication lifecycle.
- Tenant switching.
- Real-time event handling.
- Offline and reconnect behavior where supported.
- Persistence and hydration.
- Race conditions between requests and events.

### 15.4 Critical Workflow Tests

KAMPYN workflows requiring specific coverage include:
- Cart draft to order submission.
- Payment initiation and verification.
- Booking availability and confirmation.
- Inventory updates.
- Cancellation and refund status presentation.
- Community message lifecycle.
- Notification read state.
- Admin and HR approval workflows.

Tests must verify that client-side state never overrides backend authority.

### 15.5 Test Isolation

- Reset stores between tests.
- Use isolated QueryClient instances.
- Avoid shared mutable state across test cases.
- Mock network boundaries rather than internal implementation details.
- Test observable behavior.
- Include tenant and user isolation cases.

---

## 16. Observability and Debugging

State management should be diagnosable without exposing sensitive information.

- Use development-only query and store inspection tools where appropriate.
- Avoid logging private data or full payloads.
- Track relevant query failures and mutation outcomes through approved observability.
- Measure excessive refetching, cache churn, and rendering issues.
- Record workflow transitions only where operationally useful and privacy-approved.
- Ensure diagnostics can be disabled or controlled in self-hosted deployments.

Do not add sensitive state snapshots to production logs.

---

## 17. Code Organization

Suggested structure:

```text
src/
├── app/
│   ├── providers/
│   │   ├── query-provider.tsx
│   │   └── app-providers.tsx
│   └── ...
├── features/
│   ├── orders/
│   │   ├── api/
│   │   ├── queries/
│   │   ├── mutations/
│   │   ├── stores/
│   │   ├── hooks/
│   │   ├── components/
│   │   └── types/
│   ├── bookings/
│   └── community/
├── stores/
│   ├── ui/
│   ├── preferences/
│   └── cart/
├── shared/
│   ├── query/
│   ├── state/
│   └── validation/
└── ...
```

The structure is illustrative. Follow the established repository architecture and avoid introducing empty or unnecessary directories.

### 17.1 File Size

- Keep production source files at or below 200 lines where practical.
- Split large stores by domain and responsibility.
- Separate query keys, API functions, hooks, selectors, and complex transitions when doing so improves cohesion.
- Do not split files mechanically into tiny modules that obscure the design.

Any file exceeding the standard must have a clear architectural justification.

### 17.2 Naming

Use descriptive names:
- `use-orders.ts`
- `order-keys.ts`
- `cart-store.ts`
- `cart-selectors.ts`
- `use-cancel-order.ts`
- `booking-workflow-reducer.ts`

Avoid generic names such as:
- `global-store.ts`
- `data-store.ts`
- `state.ts`
- `common-hooks.ts`

Names must reflect the state owner and domain responsibility.

---

## 18. Anti-Patterns

The following are prohibited unless an explicit, reviewed exception exists:

- Duplicating TanStack Query data in Zustand.
- Putting every frontend state variable into a global store.
- Using React Context as a universal state-management solution.
- Storing derived values without a concrete reason.
- Using multiple competing sources of truth.
- Using query keys that omit relevant tenant, user, or filter context.
- Treating client-side state as authorization.
- Persisting sensitive state in ordinary browser storage.
- Using `useEffect` to synchronize redundant state unnecessarily.
- Performing network requests inside arbitrary store actions without an approved design.
- Using optimistic updates for irreversible operations without safe reconciliation.
- Treating client-side cart totals as authoritative.
- Assuming real-time events are complete, ordered, or delivered exactly once.
- Triggering broad cache invalidation after every mutation.
- Retrying unsafe writes without idempotency.
- Creating one oversized global store.
- Building complex abstractions for simple local state.
- Using global state to avoid reasonable component composition.
- Keeping stale user or tenant data after identity changes.
- Rendering unvalidated persisted or externally sourced state as trusted.
- Creating excessive state transitions that do not reflect meaningful domain behavior.

---

## 19. State Selection Decision Guide

Before implementing state, use this decision sequence:

1. **Can the value be derived?**  
   Derive it rather than storing it.

2. **Does it come from the backend?**  
   Use TanStack Query.

3. **Does it belong only to one component or a small subtree?**  
   Use React local state or a reducer.

4. **Must it be shareable, bookmarkable, or navigable?**  
   Use URL state.

5. **Is it form input or validation state?**  
   Keep it in the form lifecycle.

6. **Is it shared client-side state with a meaningful global lifecycle?**  
   Consider Zustand.

7. **Does it involve a complex workflow with restricted transitions?**  
   Consider a reducer or state machine.

8. **Must it survive reloads?**  
   Define a deliberate persistence strategy and validate the stored data.

9. **Is it critical, sensitive, tenant-scoped, or business-owned?**  
   Verify the authoritative backend design and define strict client lifecycle boundaries.

If multiple mechanisms appear necessary, document why and define synchronization and reset behavior explicitly.

---

## 20. Code Review Checklist

Before approving state-management changes, verify:

- [ ] Every state value has a clear owner.
- [ ] The selected mechanism matches the state lifecycle.
- [ ] Server state is managed through TanStack Query.
- [ ] Server data is not redundantly copied into Zustand.
- [ ] Local state remains local where practical.
- [ ] URL state is used for navigable page state.
- [ ] Form state is isolated from unrelated application state.
- [ ] Derived values are not unnecessarily stored.
- [ ] Query keys include relevant tenant, identity, and request parameters.
- [ ] Cache invalidation is deliberate and scoped.
- [ ] Mutations handle pending, success, error, and reconciliation.
- [ ] Optimistic updates have rollback and concurrency considerations.
- [ ] Non-idempotent operations are not retried unsafely.
- [ ] Tenant and authentication lifecycle transitions are safe.
- [ ] Persistence is justified, scoped, versioned, and validated.
- [ ] Sensitive data is not exposed or casually persisted.
- [ ] Real-time updates handle duplicates, ordering, and reconnection.
- [ ] Next.js server/client boundaries are respected.
- [ ] State updates avoid unnecessary renders and expensive transformations.
- [ ] Critical workflows are tested.
- [ ] Errors are surfaced safely and consistently.
- [ ] Store and query logic remain cohesive and maintainable.
- [ ] Files follow the project file-size and documentation standards.

---

## 21. Definition of Done

A state-management implementation is complete when:

- State ownership and source of truth are explicit.
- The chosen mechanism is appropriate to the lifecycle.
- No unnecessary duplicate state exists.
- Server-owned data remains authoritative on the backend.
- Query keys and cache behavior are correctly scoped.
- Mutations and state transitions handle success, failure, and recovery.
- Authentication and tenant boundaries are respected.
- Persistence and hydration are safe where applicable.
- Performance has been considered without premature complexity.
- Sensitive data is handled according to security policies.
- Unit, component, and integration tests cover relevant behavior.
- Critical workflows have appropriate failure and reconciliation handling.
- Documentation and code organization meet KAMPYN standards.

**Final principle:** Use the least complex state-management mechanism that preserves a single source of truth, protects tenant and user boundaries, and provides predictable behavior throughout the application's lifecycle.
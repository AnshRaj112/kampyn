# Server State Management Standards

## 1. Purpose

This document defines the standards for managing server state in KAMPYN's frontend applications.

KAMPYN is a multi-tenant university platform that handles frequently changing data across food ordering, bookings, inventory, scheduling, notifications, community features, and administrative workflows. Server state must remain consistent, secure, predictable, and performant across these features.

KAMPYN uses **TanStack Query as the primary client-side server-state management solution**, with the backend remaining the authoritative source of business data and rules.

Server-state management must be:

- **Consistent:** Cached data must reflect backend-confirmed state and mutations.
- **Predictable:** Queries, mutations, invalidation, and retries must have explicit behavior.
- **Tenant-safe:** Data from one university or tenant must never leak into another tenant's cache or UI.
- **Efficient:** Avoid redundant requests, excessive refetching, and unnecessary data transfer.
- **Resilient:** Handle network failures, stale data, retries, and partial failures gracefully.
- **Type-safe:** Query keys, API contracts, mutation inputs, and results must be strongly typed.
- **Maintainable:** Query logic must be reusable, feature-oriented, and independent of presentation components.
- **Observable:** Important failures, latency, and cache behavior must be diagnosable without exposing sensitive information.

This document applies to server-state fetching, caching, synchronization, mutations, optimistic updates, pagination, background updates, offline behavior, and cache lifecycle management.

Related policies:
- `frontend/react.md`
- `frontend/nextjs.md`
- `frontend/hooks.md`
- `frontend/forms.md`
- `frontend/components.md`
- `frontend/accessibility.md`
- `architecture/api.md`
- `architecture/authentication.md`
- `architecture/authorization.md`
- `architecture/multi-tenancy.md`
- `architecture/caching.md`
- `architecture/events.md`

---

## 2. Core Principles

All server-state implementations must follow these principles:

1. The backend is the single source of truth for persisted business data.
2. TanStack Query is the default owner of remote data cached in the client.
3. Zustand must not become a duplicate server-state cache.
4. Queries must be declarative, typed, and organized by feature.
5. Query keys must be deterministic, hierarchical, and tenant-aware.
6. Cache invalidation must be explicit and predictable.
7. Mutations must reconcile client state with backend-confirmed results.
8. Optimistic updates must only be used when correctness and rollback can be maintained.
9. Authorization must always be enforced by the backend.
10. Client-side cache isolation must be maintained across user and tenant changes.
11. Retry, refetch, and stale-data behavior must be configured according to the operation.
12. Server state must not be treated as permanently fresh.
13. Network failures must not silently become successful empty results.
14. Cache behavior must be testable independently of presentation.
15. Performance decisions must be based on actual usage and measurement.
16. All asynchronous operations must have clear loading, error, and success behavior.
17. Server-state logic must not be coupled unnecessarily to individual UI components.

---

## 3. What Is Server State?

Server state is data owned and managed by backend services, where the frontend retrieves a representation of that data over a network.

Examples in KAMPYN include:

- Food court menus and item availability.
- Shopping carts persisted by the backend.
- Order details and order status.
- Hostel and guest house availability.
- Booking records and booking history.
- Shuttle schedules and seat availability.
- Washing machine scheduling slots.
- Library vacancy information.
- User profiles and institutional memberships.
- Vendor information and operating hours.
- Inventory quantities.
- Complaints and complaint statuses.
- Community posts and replies.
- Notifications and notification preferences.
- Administrative reports and analytics.
- Payment status and transaction records.

Server state may change independently of the current browser session because of:
- Other users.
- Administrative operations.
- Background jobs.
- Scheduled tasks.
- Webhooks.
- Third-party integrations.
- Other frontend clients.
- Backend reconciliation processes.

Therefore, a previously fetched response must be treated as a cached snapshot, not as guaranteed current truth.

### 3.1 Server State vs. Client State

| State type | Description | Preferred owner |
|---|---|---|
| Server state | Data owned by backend services | TanStack Query |
| Local UI state | Temporary component interaction state | React `useState` or `useReducer` |
| Shared client state | Cross-component client-only state | Zustand |
| URL state | Navigable or shareable interface state | URL/search parameters |
| Form state | User input, dirty state, and validation | Approved form solution |
| Persistent business state | Records and business operations | Backend services |
| Transient shared state | Ephemeral presence, coordination, or cache data | Backend and appropriate infrastructure |

Do not store the same remote entity independently in TanStack Query, Zustand, and component state without a clear synchronization requirement.

### 3.2 Server State Characteristics

Server state is generally:
- Asynchronous.
- Remotely owned.
- Potentially stale.
- Subject to network failures.
- Shared between multiple users or clients.
- Governed by backend authorization.
- Potentially paginated or filtered.
- Affected by concurrent mutations.

Server-state design must account for these characteristics.

---

## 4. TanStack Query as the Standard

### 4.1 Primary Responsibility

TanStack Query must be used for client-side remote data operations that require:
- Fetching.
- Caching.
- Deduplication.
- Background refetching.
- Mutation lifecycle management.
- Cache invalidation.
- Pagination and infinite queries.
- Request cancellation.
- Synchronization between UI and backend responses.

Use the installed, project-approved TanStack Query version and its matching APIs. Avoid introducing another server-state library for overlapping responsibilities.

### 4.2 QueryClient

Create and configure a shared `QueryClient` according to the application's execution environment.

The configuration must define:
- Query defaults.
- Mutation defaults where appropriate.
- Retry behavior.
- Refetch behavior.
- Cache garbage collection.
- Error handling conventions.
- Development tooling policy.

In Next.js, avoid sharing a user-specific `QueryClient` across server requests. Server-side clients must be scoped to the relevant request or rendering operation. Browser-side clients may be long-lived within the authenticated application session.

### 4.3 Query Provider

The application must have a clear provider boundary for TanStack Query.

Provider requirements:
- Instantiate the browser `QueryClient` stably.
- Avoid creating a new client on every render.
- Avoid sharing user-specific cache state between server requests.
- Coordinate cache lifecycle with authentication and tenant changes.
- Keep provider initialization separate from feature query definitions.
- Integrate development tools only in approved development environments.

### 4.4 Query Defaults

Set global defaults for common behavior, but override them for features with distinct consistency or latency requirements.

Do not assume every endpoint has identical:
- Freshness requirements.
- Retry safety.
- Update frequency.
- Sensitivity.
- Data volume.
- Cache lifetime.

Avoid excessively aggressive defaults that cause unnecessary traffic or overly permissive defaults that display stale operational data.

---

## 5. Query Architecture

### 5.1 Feature-Oriented Organization

Query definitions must be organized by feature.

A representative structure:

```text
features/
  orders/
    api/
      orders.api.ts
    queries/
      order.keys.ts
      order.queries.ts
      order.mutations.ts
    hooks/
      useOrders.ts
      useOrderDetails.ts
      useCancelOrder.ts
    schemas/
      order.schema.ts
    types/
      order.types.ts
    components/
      OrderList.tsx
      OrderDetails.tsx
```

This is a conceptual structure. Use the repository's established organization and avoid introducing folders that do not add clarity.

### 5.2 API Layer

The API layer must encapsulate HTTP communication.

It is responsible for:
- Endpoint paths and HTTP methods.
- Request serialization.
- Response parsing.
- Authentication integration.
- Cancellation signal propagation.
- Error normalization.
- Runtime validation where required.
- Typed inputs and outputs.

The API layer must not:
- Contain presentation logic.
- Access React component state directly.
- Perform UI notifications.
- Silently convert errors into successful empty values.
- Duplicate backend business rules.

Example:

```ts
export async function getOrder(
  orderId: string,
  signal?: AbortSignal,
): Promise<Order> {
  return apiClient.get<Order>(`/orders/${orderId}`, {
    signal,
  });
}
```

Use the project's approved API client and response-validation conventions.

### 5.3 Query Definitions

Query definitions must centralize:
- Query keys.
- Query functions.
- Query-specific configuration.
- Relevant filters and identifiers.
- Cancellation behavior.
- Freshness requirements.

Use query options or a consistent feature-specific pattern to prevent duplicated query configuration.

Example:

```ts
export const orderQueries = {
  detail: (orderId: string) => ({
    queryKey: orderKeys.detail(orderId),
    queryFn: ({ signal }: { signal: AbortSignal }) =>
      getOrder(orderId, signal),
    enabled: orderId.length > 0,
  }),
};
```

The exact implementation must match the installed TanStack Query version and approved project conventions.

### 5.4 Query Hooks

Custom hooks should provide a feature-level interface to components.

Example:

```ts
export function useOrderDetails(orderId: string) {
  return useQuery(orderQueries.detail(orderId));
}
```

Hooks must:
- Have clear names.
- Expose only the data and operations needed by their consumers.
- Avoid unnecessary transformation of canonical server data.
- Avoid duplicating query definitions.
- Remain independently testable.
- Avoid coupling query logic to one component's visual behavior.

### 5.5 Component Usage

Components should consume query hooks rather than implementing raw fetching logic inline.

Example:

```tsx
function OrderDetails({ orderId }: { orderId: string }) {
  const query = useOrderDetails(orderId);

  if (query.isPending) {
    return <OrderDetailsSkeleton />;
  }

  if (query.isError) {
    return <OrderError onRetry={() => query.refetch()} />;
  }

  if (!query.data) {
    return <OrderNotFound />;
  }

  return <OrderDetailsView order={query.data} />;
}
```

The component is responsible for rendering appropriate UI states, while the query layer owns remote data lifecycle behavior.

---

## 6. Query Keys

### 6.1 Requirements

Every query must use a stable, deterministic query key.

Query keys must:
- Be serializable.
- Be structured and predictable.
- Include all variables that affect the result.
- Distinguish list, detail, and filtered views.
- Include relevant tenant or identity scope.
- Support safe targeted invalidation.
- Avoid unnecessary duplication.

Never use randomly generated values, current timestamps, or unstable object references as query keys.

### 6.2 Hierarchical Keys

Use hierarchical query keys that support broad and narrow invalidation.

Example:

```ts
export const orderKeys = {
  all: ["orders"] as const,

  lists: () => [...orderKeys.all, "list"] as const,

  list: (filters: OrderFilters) =>
    [...orderKeys.lists(), filters] as const,

  details: () => [...orderKeys.all, "detail"] as const,

  detail: (orderId: string) =>
    [...orderKeys.details(), orderId] as const,
};
```

This allows invalidating:
- All order-related data.
- All order lists.
- One order detail.
- A specific filtered list.

### 6.3 Tenant-Aware Keys

KAMPYN is a multi-tenant platform. Queries must be scoped to the active tenant whenever the result is tenant-dependent.

Example:

```ts
export const tenantOrderKeys = {
  all: (tenantId: string) =>
    ["tenants", tenantId, "orders"] as const,

  list: (tenantId: string, filters: OrderFilters) =>
    [...tenantOrderKeys.all(tenantId), "list", filters] as const,

  detail: (tenantId: string, orderId: string) =>
    [...tenantOrderKeys.all(tenantId), "detail", orderId] as const,
};
```

Tenant-aware keys are a cache isolation mechanism only. They do not provide authorization or prove that a user may access the requested tenant.

The backend must independently validate the user's tenant membership and access rights.

### 6.4 Identity-Aware Keys

Where a query's result depends on the authenticated identity, include the relevant stable identity scope in the query key or isolate the cache lifecycle on identity changes.

Examples:
- Personal notifications.
- User-specific preferences.
- Personal booking history.
- User-specific dashboard summaries.

Do not include access tokens, session secrets, or other credentials in query keys.

### 6.5 Query Key Factories

Every feature with multiple related queries should define a query-key factory or an equivalent centralized convention.

Do not independently author string keys in multiple components.

### 6.6 Filter Serialization

Query-key filter objects must be deterministic.

- Use normalized filter values.
- Avoid including irrelevant fields.
- Represent missing values consistently.
- Avoid mutable objects.
- Keep filter semantics stable.
- Ensure the query function uses the same effective filters represented by the key.

---

## 7. Cache Freshness and Lifecycle

### 7.1 Freshness

Cached data is not automatically current backend state.

Configure freshness based on the feature's consistency requirements.

Examples:
- Frequently changing order status may need short freshness intervals or event-driven updates.
- Relatively stable category metadata may tolerate longer freshness.
- Administrative analytics may use explicit refresh controls or scheduled refreshes.
- Critical availability must be revalidated before the backend accepts a booking or reservation.

### 7.2 `staleTime`

Use `staleTime` to define how long data may be treated as fresh by TanStack Query.

Guidelines:
- Use feature-specific freshness where required.
- Avoid setting every query to immediate staleness without a reason.
- Avoid treating long-lived cache data as proof of current availability.
- Revisit freshness settings when data update frequency changes.

### 7.3 `gcTime`

Use `gcTime` to control how long inactive query data remains in memory.

Guidelines:
- Set sensible defaults.
- Avoid retaining large or sensitive datasets unnecessarily.
- Consider the user's device memory and session behavior.
- Clear relevant data when identity or tenant scope changes.
- Avoid excessive cache retention for rapidly changing, high-cardinality queries.

Use the option name supported by the installed TanStack Query version.

### 7.4 Refetch Triggers

Refetch behavior may be triggered by:
- Component mount.
- Window focus.
- Network reconnection.
- Explicit user refresh.
- Query invalidation.
- Polling.
- Server events or subscriptions.

Configure these triggers according to the feature's consistency and resource requirements.

### 7.5 Stale Data Presentation

When stale data is displayed during a background refetch:
- Keep the UI understandable.
- Indicate refreshing state where it matters.
- Avoid implying that stale operational information is guaranteed current.
- Prevent stale values from authorizing or confirming critical actions.
- Reconcile the UI with the latest backend response.

### 7.6 Cache Retention

Cache retention must account for:
- Data sensitivity.
- User identity.
- Tenant boundaries.
- Data volume.
- Application lifecycle.
- Offline behavior.
- Device memory.

Do not retain sensitive server data indefinitely without a documented requirement.

---

## 8. Loading, Error, Empty, and Success States

### 8.1 Explicit UI States

Every query-driven feature must account for its relevant lifecycle states:

- Initial loading.
- Initial error.
- Successful response with data.
- Successful response with no results.
- Background fetching.
- Background refetch failure.
- Disabled or not-yet-enabled query.
- Partial data, if supported by the API contract.

### 8.2 Loading

Use appropriate loading UI:
- Skeletons for structured content.
- Spinners for short, localized actions.
- Progress indicators for long-running operations where progress is known.
- Stable layout placeholders to avoid layout shifts.

Avoid showing a full-page loading indicator for a small background refresh unless the workflow requires it.

### 8.3 Errors

Errors must:
- Be normalized through the API layer.
- Be represented clearly to the user.
- Avoid exposing stack traces or internal service details.
- Distinguish recoverable from non-recoverable failures.
- Provide retry or recovery actions when appropriate.

Do not return an empty array or default object when a request fails unless that behavior is explicitly part of the API contract.

### 8.4 Empty States

An empty result is not an error.

Empty states should:
- Explain the absence of data.
- Preserve relevant filters or context.
- Provide a useful next step where appropriate.
- Distinguish “no records exist” from “no records match these filters.”

### 8.5 Background Updates

For background refetches:
- Avoid unnecessarily replacing stable content with loading skeletons.
- Show a subtle refresh indicator where useful.
- Keep the current data visible when safe.
- Clearly handle refresh failures.
- Prevent duplicate user actions when a mutation is already pending.

### 8.6 Error Recovery

Recovery behavior must be appropriate to the operation.

Examples:
- Retry fetching a menu.
- Refresh order status.
- Revalidate booking availability.
- Preserve a complaint draft after a network error.
- Require renewed authentication when the session has expired.

---

## 9. Mutations

### 9.1 Mutation Responsibilities

Mutations represent operations that change backend-owned state.

Examples:
- Create or cancel an order.
- Update a profile.
- Submit a complaint.
- Book a hostel room.
- Reserve a resource.
- Update inventory.
- Mark a notification as read.
- Change a community post.

Mutations must be implemented through the approved API and TanStack Query mutation patterns.

### 9.2 Mutation Definitions

Mutation definitions must centralize:
- Input types.
- API function.
- Result types.
- Pending and error behavior.
- Cache reconciliation.
- Invalidation strategy.
- Retry safety.
- Optimistic update behavior, if applicable.

Example:

```ts
export function useCancelOrder() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: cancelOrder,
    onSuccess: (_result, input) => {
      void queryClient.invalidateQueries({
        queryKey: orderKeys.detail(input.orderId),
      });

      void queryClient.invalidateQueries({
        queryKey: orderKeys.lists(),
      });
    },
  });
}
```

The example is illustrative; actual input types and invalidation scope must match the feature's contracts.

### 9.3 Backend Confirmation

A successful mutation must be interpreted according to the backend's API contract.

- Use the returned canonical entity when available.
- Reconcile affected queries.
- Do not assume a client-side state change means the backend operation committed.
- Do not display irreversible success before the backend confirms it.
- Treat asynchronous workflows as pending until the backend reports their defined completion state.

### 9.4 Mutation State

Expose relevant mutation states to the UI:
- Idle.
- Pending.
- Success.
- Error.

Avoid maintaining separate component state that duplicates mutation lifecycle status unless the workflow requires additional independent state.

### 9.5 Duplicate Submissions

Prevent accidental duplicate submissions when the operation requires it.

Use:
- Disabled controls during pending operations.
- Request or operation identifiers where supported.
- Backend idempotency keys for appropriate high-impact operations.
- Explicit state transitions in the UI.

Frontend button disabling alone is not a sufficient duplicate-operation safeguard.

### 9.6 Mutation Retries

Do not retry mutations indiscriminately.

A mutation retry must account for:
- Whether the operation is idempotent.
- Whether the backend supports idempotency keys.
- Whether the request may have succeeded despite a client timeout.
- Whether retrying could cause duplicate orders, payments, or bookings.

Never automatically retry a non-idempotent high-impact operation without a safe idempotency strategy.

---

## 10. Cache Invalidation and Reconciliation

### 10.1 Invalidation Strategy

Every mutation must define how related cached data is reconciled.

Possible approaches:
- Update a query directly using the canonical mutation response.
- Invalidate the affected query.
- Invalidate a related group of queries.
- Reset a cache scope when its identity or tenant context is no longer valid.

Choose the narrowest strategy that maintains correctness.

### 10.2 Targeted Invalidation

Prefer targeted invalidation over clearing the entire query cache after ordinary mutations.

Examples:
- Updating one order should reconcile its detail and affected order lists.
- Changing a menu item may affect the item detail, menu listing, and relevant availability summaries.
- Marking one notification as read should reconcile that notification and relevant unread counts.
- Updating a user's profile should reconcile the affected profile and dependent summaries.

### 10.3 Direct Cache Updates

Use `setQueryData` or related cache-update APIs when:
- The mutation response contains canonical data.
- The affected query shape is known.
- The update can be applied immutably.
- The cache can be kept consistent without guessing.

Avoid manually patching many query variants if invalidation is simpler and safer.

### 10.4 Invalidation After Partial Success

For workflows involving multiple backend operations:
- Track each operation's outcome.
- Reconcile successfully committed changes.
- Invalidate queries when the resulting state is uncertain.
- Avoid presenting the entire workflow as successful if one required step failed.
- Provide a recovery path.

### 10.5 Invalidation Loops

Avoid:
- Invalidating queries repeatedly from render effects.
- Invalidating the same key from multiple redundant lifecycle handlers.
- Mutations that trigger uncontrolled refetch loops.
- Broad invalidation after every small UI event.

Invalidation must be tied to meaningful data changes.

---

## 11. Optimistic Updates

### 11.1 Usage Policy

Optimistic updates may be used when:
- The action is reversible or recoverable.
- The expected result is sufficiently predictable.
- The user benefits from immediate feedback.
- Rollback or reconciliation is reliable.
- Concurrent operations are handled safely.

Optimistic updates must not compromise business correctness.

### 11.2 Suitable Use Cases

Potentially suitable operations include:
- Toggling a simple user preference.
- Marking a notification as read.
- Updating low-risk presentation state persisted by the backend.
- Adding a reversible reaction to a community post.

Each use case still requires a risk assessment.

### 11.3 High-Risk Operations

Do not optimistically confirm:
- Payments.
- Final order acceptance.
- Inventory commitments.
- Seat or room availability.
- Resource reservations.
- Irreversible cancellations.
- Permission or role changes.
- Other operations whose failure could cause financial, operational, or access-control inconsistencies.

These operations must be confirmed by the backend before the UI represents them as committed.

### 11.4 Optimistic Update Lifecycle

When using an optimistic update:
1. Cancel conflicting in-flight queries where appropriate.
2. Snapshot the current cache state.
3. Apply a minimal optimistic change.
4. Execute the backend mutation.
5. Roll back on failure using the snapshot when safe.
6. Reconcile with the canonical backend response.
7. Invalidate affected queries when necessary.

### 11.5 Concurrency

Optimistic updates must consider:
- Multiple mutations against the same entity.
- Out-of-order responses.
- Other users changing the same data.
- Stale rollback snapshots.
- Partial updates and conflicts.

Avoid rollback strategies that overwrite newer, successfully committed changes.

When concurrency cannot be handled safely, prefer a pending state followed by backend-confirmed reconciliation.

---

## 12. Pagination and Infinite Queries

### 12.1 Pagination

Use server-side pagination for large datasets.

Pagination must:
- Use explicit, typed parameters.
- Include page or cursor state in the query key.
- Preserve filters and sort order.
- Handle empty and out-of-range results.
- Avoid fetching unnecessary pages.
- Respect backend limits.

### 12.2 Cursor-Based Pagination

Prefer cursor-based pagination for large or frequently changing datasets when supported by the backend.

Cursor pagination can reduce inconsistencies caused by records being inserted or deleted between page requests.

The cursor must be treated as an opaque backend contract unless the API explicitly defines its structure.

### 12.3 Infinite Queries

Use TanStack Query infinite queries when users naturally consume sequential pages, such as:
- Community feeds.
- Order history.
- Notification lists.
- Search results.
- Long activity streams.

Infinite queries must:
- Define `getNextPageParam` correctly.
- Prevent unnecessary duplicate page requests.
- Expose loading and error states for subsequent pages.
- Avoid unbounded cache growth.
- Support refreshing and invalidating loaded pages.
- Preserve filter and tenant scope.

### 12.4 Pagination State

Keep pagination in URL state when it should be shareable, bookmarkable, or navigable through browser history.

Use local state for temporary pagination interactions that do not need URL persistence.

Do not maintain conflicting pagination state in both the URL and a global store without a defined synchronization policy.

---

## 13. Search, Filtering, and Sorting

### 13.1 Query-Driven Search

Search requests must be represented in query keys using normalized search parameters.

Search state should be:
- Typed.
- Deterministic.
- Tenant-aware where applicable.
- Consistent with backend search contracts.

### 13.2 Debouncing

Debounce user input when sending requests on every keystroke would cause unnecessary traffic.

Debouncing must:
- Preserve immediate input responsiveness.
- Avoid stale result confusion.
- Cancel or disregard obsolete requests where possible.
- Keep the visible search term and result context clear.

Do not debounce actions where immediate submission or explicit user confirmation is the intended behavior.

### 13.3 Filtering

- Normalize filter values before they enter query keys.
- Avoid duplicate requests caused by semantically equivalent filter states.
- Keep filters consistent with the backend's supported query contract.
- Preserve filters across pagination when appropriate.
- Avoid applying large dataset filters entirely in the browser when server-side filtering is required.

### 13.4 Sorting

For large datasets, sorting should generally be performed by the backend.

Sorting parameters must be part of the query key. Ensure the UI's displayed sort direction accurately represents the request.

---

## 14. Polling and Real-Time Updates

### 14.1 Polling

Polling may be used when the application needs periodic updates and no suitable event-driven mechanism is available.

Potential use cases:
- Order status.
- Booking processing status.
- Long-running administrative operations.
- Resource availability snapshots.

Polling must:
- Use feature-specific intervals.
- Stop when data is no longer relevant.
- Avoid excessive request frequency.
- Respect visibility and connectivity where appropriate.
- Avoid polling while a conflicting mutation is in progress.
- Handle request failures and backoff intentionally.

### 14.2 Conditional Polling

Polling intervals may depend on query state.

For example:
- Poll more frequently while an operation is actively processing.
- Reduce polling after a terminal status.
- Stop polling when the user leaves the relevant workflow.
- Pause or reduce polling when the browser tab is hidden, where suitable.

### 14.3 Real-Time Events

Use approved event mechanisms, such as WebSockets or server-sent events, when real-time updates are required.

Event handlers must:
- Validate event structure.
- Confirm tenant and entity scope.
- Avoid trusting event payloads as authorization.
- Reconcile the appropriate cache entries.
- Handle disconnects and reconnects.
- Avoid creating duplicate subscriptions.

### 14.4 Event-Driven Reconciliation

An incoming event may:
- Update a known cache entry if the event contains a reliable canonical update.
- Invalidate affected queries when the event is incomplete or ordering is uncertain.
- Trigger a targeted refetch when authoritative reconciliation is needed.

Do not assume events are delivered exactly once or in order unless the event infrastructure explicitly guarantees those properties.

### 14.5 Polling and Events Together

Avoid overlapping polling and event subscriptions that perform redundant work without a clear consistency requirement.

If both are needed:
- Define which mechanism is primary.
- Use the secondary mechanism for recovery or reconciliation.
- Prevent duplicate UI notifications.
- Document the expected freshness behavior.

---

## 15. Authentication and Tenant Lifecycle

### 15.1 Authentication-Aware Cache

Server-state caches may contain user-specific or tenant-specific information.

The application must define cache behavior for:
- Login.
- Logout.
- Session expiration.
- Account switching.
- Tenant switching.
- Membership changes.
- Permission changes.

### 15.2 Logout

On logout:
- Clear or remove user-specific query data.
- Clear identity-scoped client state where appropriate.
- Cancel sensitive in-flight queries where possible.
- Remove persisted sensitive cache data if persistence is configured.
- Prevent previous user data from remaining visible during the next session.

### 15.3 Tenant Switching

When the active tenant changes:
- Prevent previous tenant data from appearing in the new tenant's UI.
- Use tenant-aware query keys.
- Reset tenant-scoped client state where necessary.
- Cancel or disregard obsolete requests.
- Revalidate tenant membership and permissions through the backend.
- Ensure persisted cache cannot cross tenant boundaries.

### 15.4 Session Expiration

When the backend indicates an expired or invalid session:
- Follow the approved authentication flow.
- Avoid uncontrolled retry loops.
- Prevent sensitive cached information from being exposed after session invalidation.
- Preserve recoverable user input where safe.
- Reconcile authentication and query state consistently.

### 15.5 Authorization Changes

When roles or permissions change:
- Revalidate the affected access context.
- Remove or invalidate data that is no longer authorized.
- Do not rely on hiding UI elements alone.
- Ensure backend access checks continue to protect all requests.

---

## 16. Cancellation and Race Conditions

### 16.1 Request Cancellation

Query functions should consume TanStack Query's `AbortSignal` where the underlying API client supports cancellation.

Cancellation is especially important for:
- Search-as-you-type.
- Route transitions.
- Changing filters.
- Obsolete detail requests.
- Tenant switching.
- Logout.

### 16.2 Race Conditions

Handle cases where:
- A newer request finishes before an older request.
- A mutation completes while a query is in flight.
- A user changes filters during loading.
- An entity is deleted while its details are being fetched.
- A tenant changes while old requests remain active.

Prefer TanStack Query's built-in query lifecycle mechanisms and stable query keys over manually managing request order.

### 16.3 Stale Responses

Do not apply an obsolete response to the wrong entity, filter, user, or tenant context.

Ensure request identity is represented correctly in query keys and request parameters.

### 16.4 Cancellation Limitations

Client-side cancellation does not guarantee the backend operation was cancelled.

For mutations, treat client disconnection or abort as an uncertain outcome unless the backend contract provides a reliable cancellation mechanism.

---

## 17. Retries and Network Resilience

### 17.1 Retry Policy

Retry behavior must be operation-specific.

For queries:
- Retry transient failures where recovery is plausible.
- Avoid retrying permanent client errors unnecessarily.
- Use bounded retries.
- Use suitable backoff.
- Avoid retry storms.

For mutations:
- Retry only when idempotency and operation semantics make it safe.
- Use backend-supported idempotency mechanisms for high-impact actions where appropriate.

### 17.2 Error Classification

Where practical, distinguish:
- Network failures.
- Timeouts.
- Rate limits.
- Authentication failures.
- Authorization failures.
- Validation errors.
- Conflicts.
- Not-found responses.
- Server failures.

Retry policy must account for the error category.

### 17.3 Offline and Reconnection

When offline support is required:
- Show connection state clearly.
- Distinguish cached data from confirmed current data.
- Avoid implying that an unsent mutation has been committed.
- Resume safe queries on reconnection.
- Reconcile uncertain operations with the backend.
- Avoid replaying unsafe mutations automatically.

### 17.4 Retry Visibility

Repeated retrying must not make the interface appear frozen.

Where appropriate:
- Display a connection or retry state.
- Provide a manual retry action.
- Preserve current data when safe.
- Communicate when the information may be stale.

---

## 18. Prefetching and Initial Data

### 18.1 Prefetching

Prefetch data when there is a clear likelihood it will be needed soon and the expected benefit exceeds the cost.

Suitable cases may include:
- Likely next route data.
- Frequently opened detail panels.
- Common workflow transitions.
- Predictable navigation paths.

Avoid prefetching large or sensitive datasets without a clear requirement.

### 18.2 Prefetch Conditions

Prefetching must:
- Respect tenant and identity scope.
- Use the same canonical query keys as regular queries.
- Avoid duplicate or competing requests.
- Respect freshness requirements.
- Avoid excessive background network use.

### 18.3 Initial Data

Use `initialData` only when the data is genuinely available and compatible with the target query's contract.

Do not use placeholder or incomplete values as if they were canonical backend data.

Use placeholder mechanisms only for temporary presentation and ensure the UI can distinguish placeholder content from confirmed data.

### 18.4 Next.js Hydration

When prefetching or hydrating TanStack Query in Next.js:
- Use request-scoped server clients.
- Dehydrate only the data required by the client.
- Avoid leaking data across server requests.
- Preserve matching query keys and serialization behavior.
- Avoid hydrating sensitive data into an unauthorized client boundary.
- Follow `frontend/nextjs.md` for Server Component and Client Component responsibilities.

---

## 19. Persistence and Offline Cache

### 19.1 Persistence Policy

Persisting TanStack Query cache is not the default.

Enable persistence only when there is a defined user benefit and an explicit review of:
- Data sensitivity.
- Tenant isolation.
- Identity lifecycle.
- Cache expiration.
- Schema compatibility.
- Storage limitations.
- Logout behavior.
- Offline reconciliation.

### 19.2 Sensitive Data

Do not persist sensitive server data in browser storage without explicit security review and approval.

Avoid persisting:
- Authentication credentials.
- Payment secrets.
- Sensitive personal information without a clear requirement.
- Data that the user is no longer authorized to access.

### 19.3 Cache Versioning

If persisted query data is used:
- Define a persistence version.
- Handle schema incompatibility.
- Expire obsolete data.
- Clear incompatible cache entries safely.
- Test upgrade and logout behavior.

### 19.4 Offline Mutations

Offline mutation queues require a dedicated design.

They must define:
- Which operations are safe to queue.
- Idempotency behavior.
- Ordering requirements.
- Conflict handling.
- Expiration.
- Retry limits.
- User-visible pending state.
- Reconciliation after reconnection.

Do not automatically queue high-impact operations such as payments, inventory commitments, or bookings without a reviewed backend-supported workflow.

---

## 20. Data Transformation and Validation

### 20.1 Canonical Data

Keep the canonical server response available in a predictable form.

Transform data when necessary for:
- UI-specific formatting.
- View models.
- Sorting or grouping for a local presentation.
- Deriving display-only values.

Do not mutate cached server data.

### 20.2 Runtime Validation

Validate external data at appropriate trust boundaries, especially when:
- API responses are not strongly guaranteed by the service contract.
- Data comes from third-party integrations.
- Payloads are dynamic or versioned.
- Persisted cache data is restored.
- Events are received through real-time channels.

Use the approved Zod schemas or equivalent runtime validation approach.

### 20.3 Transformation Location

Place transformations according to their responsibility:
- API normalization in the API layer.
- Shared domain-independent mapping in approved utilities.
- Query-specific selection in query configuration when suitable.
- Presentation-only formatting in UI utilities or components.

Avoid repeating the same transformation in multiple components.

### 20.4 Data Consistency

Do not silently normalize away meaningful backend distinctions, such as:
- Pending versus confirmed status.
- Missing versus null values.
- Zero versus unavailable quantities.
- Expired versus cancelled bookings.
- Estimated versus final totals.

Preserve distinctions required by the business contract.

---

## 21. Tenant and User Data Isolation

### 21.1 Isolation Requirements

Tenant isolation is a critical KAMPYN requirement.

The frontend must prevent accidental cache reuse across:
- Universities.
- Campus organizations.
- User accounts.
- Roles.
- Membership scopes.
- Other access-controlled contexts.

### 21.2 Cache Scope

Tenant-sensitive queries must use an explicit tenant scope in the query key or be protected through a reliable cache lifecycle boundary.

User-specific queries must similarly account for identity changes.

### 21.3 No Authorization by Cache

Cache keys are not security controls.

The backend must validate:
- Authentication.
- Tenant membership.
- Role and permission.
- Resource ownership.
- Requested operation.

### 21.4 Cross-Tenant Testing

Tests must verify that:
- Switching tenants does not display previous tenant data.
- Cached results are not reused incorrectly.
- In-flight responses from the previous tenant cannot contaminate the new context.
- Logout and account switching clear or isolate private data.
- Tenant-sensitive persisted cache is handled safely.

---

## 22. Feature-Specific Consistency Requirements

### 22.1 Food Ordering

Food ordering data may include:
- Menu items.
- Prices.
- Availability.
- Cart contents.
- Order details.
- Order status.
- Preparation and fulfillment updates.

Requirements:
- Treat backend-confirmed prices and totals as authoritative.
- Revalidate item availability during checkout.
- Do not use cached availability as proof that an item can be ordered.
- Reconcile cart and order data after mutations.
- Use safe idempotency for order creation where supported.
- Avoid optimistic confirmation of order acceptance or payment.

### 22.2 Payments

Payment state is high impact.

Requirements:
- Backend payment status is authoritative.
- Do not treat frontend redirects as proof of payment.
- Do not optimistically confirm payment success.
- Handle uncertain outcomes by querying the authoritative payment or order status.
- Avoid unsafe automatic mutation retries.
- Reconcile payment and order state through backend-confirmed workflows.

### 22.3 Bookings and Reservations

Requirements:
- Cached availability is informational, not a reservation guarantee.
- The backend must validate availability at booking time.
- Confirmed booking status must come from the backend.
- Handle conflicts and unavailable-slot responses explicitly.
- Reconcile related availability and booking queries after successful mutations.
- Do not optimistically represent an unconfirmed reservation as final.

### 22.4 Inventory

Requirements:
- Backend inventory quantities are authoritative.
- Cached stock values must not authorize commitments.
- Inventory-changing mutations must respect backend concurrency rules.
- Reconcile relevant product, inventory, and reporting queries.
- Avoid optimistic updates that imply a final stock commitment.

### 22.5 Community

Requirements:
- Use pagination or infinite queries for large feeds.
- Reconcile post and reply mutations predictably.
- Handle deleted or moderated content.
- Validate real-time events.
- Prevent cached private or tenant-scoped content from crossing boundaries.
- Avoid unbounded retention of feed data.

### 22.6 Notifications

Requirements:
- Keep unread counts and notification lists consistent.
- Reconcile read-state mutations.
- Avoid duplicate event-driven notifications.
- Scope data to the correct identity and tenant.
- Handle push or real-time events as update signals, not authorization evidence.

### 22.7 Administrative Analytics

Requirements:
- Make data freshness clear where relevant.
- Avoid frequent refetching of expensive reports.
- Use server-side aggregation for large datasets.
- Apply pagination or filters where appropriate.
- Ensure tenant and role authorization on the backend.
- Avoid exposing sensitive aggregate data through shared client caches.

---

## 23. Performance and Resource Management

### 23.1 Request Efficiency

- Deduplicate identical concurrent queries.
- Avoid repeated requests caused by unstable query keys.
- Avoid unnecessary refetching on every render or navigation.
- Batch requests where the backend provides an appropriate contract.
- Use pagination for large responses.
- Prefer targeted invalidation.

### 23.2 Cache Size

Monitor and control:
- Number of active queries.
- Number of inactive cached queries.
- Infinite query page accumulation.
- Large response payloads.
- Persisted cache volume.
- High-cardinality search keys.

Do not cache unbounded datasets without an explicit retention strategy.

### 23.3 Request Frequency

Use sensible refresh behavior based on the data's volatility and business impact.

Avoid:
- Very short polling intervals without justification.
- Refetching all queries after every mutation.
- Simultaneous polling and event-driven refetching without coordination.
- Retry storms after service outages.

### 23.4 Rendering Efficiency

Server-state updates should not cause unnecessary rerendering across unrelated features.

Use:
- Focused query hooks.
- Narrow subscriptions.
- Appropriate selectors.
- Stable component boundaries.
- Query-specific transformations only when useful.

Avoid premature memoization and unnecessary cache manipulation.

### 23.5 Measurement

Use approved monitoring and profiling tools to evaluate:
- Request count.
- Request latency.
- Cache hit behavior where measurable.
- Refetch frequency.
- Payload sizes.
- Query error rates.
- UI responsiveness after query updates.

Optimize measured bottlenecks rather than assumed ones.

---

## 24. Security and Privacy

### 24.1 Frontend Cache as Sensitive Data

Cached query data may contain private information even though it is held in the browser.

Apply data minimization and appropriate lifecycle handling.

### 24.2 Error Sanitization

Never expose:
- Stack traces.
- Internal database errors.
- Service credentials.
- Access tokens.
- Sensitive infrastructure details.
- Raw private request payloads.

Normalize errors through the API layer and display only safe information.

### 24.3 Cache Clearing

Clear or invalidate sensitive cache data when:
- The user logs out.
- The active identity changes.
- The active tenant changes.
- The session becomes invalid.
- Authorization context changes materially.

### 24.4 Browser Devtools

Development query devtools must not be enabled in production unless explicitly approved and safely configured.

Avoid exposing sensitive cached data through diagnostic interfaces, logs, or error reports.

### 24.5 Persisted State

Persisted server state requires explicit privacy and security review. Do not assume browser storage is private, encrypted, or inaccessible to other scripts running in the same origin.

---

## 25. Testing Standards

### 25.1 Query Tests

Query tests must verify:
- Correct query keys.
- Correct request parameters.
- Successful data retrieval.
- Error behavior.
- Retry policy where relevant.
- Cancellation behavior where relevant.
- Cache freshness and invalidation.
- Tenant and identity scoping.

### 25.2 Mutation Tests

Mutation tests must verify:
- Correct request payload.
- Pending, success, and error states.
- Duplicate-submission safeguards.
- Cache reconciliation.
- Rollback behavior for optimistic updates.
- Idempotency expectations.
- Error handling.
- Tenant-aware behavior.

### 25.3 Cache Tests

Test:
- Targeted invalidation.
- Direct cache updates.
- Removal of sensitive query data.
- Account switching.
- Tenant switching.
- Stale response handling.
- Concurrent query and mutation behavior.
- Pagination and infinite-query behavior.

### 25.4 Integration Tests

Integration tests should cover realistic feature workflows such as:
- Browse menu → add item → checkout → view order.
- Select resource → check availability → book → view confirmation.
- Submit complaint → view status → receive update.
- Receive notification → mark as read → verify unread count.

Tests must mock external network boundaries rather than rely on live production services.

### 25.5 Testing Tools

Use the project's approved testing stack, such as:
- Vitest or the configured unit-test runner.
- React Testing Library.
- TanStack Query testing utilities.
- MSW or the approved network mocking approach.
- Playwright or the configured end-to-end framework.

Do not introduce duplicate testing frameworks without architectural approval.

### 25.6 Test Isolation

Each test must have isolated query state.

- Use a fresh `QueryClient` where appropriate.
- Disable retries in tests unless retry behavior is explicitly under test.
- Clear cache and timers when required.
- Avoid tests that depend on execution order.
- Avoid leaking state between test cases.

---

## 26. Observability

Server-state operations must be diagnosable without exposing sensitive data.

Monitor relevant indicators such as:
- Query failure rate.
- Mutation failure rate.
- API latency.
- Retry frequency.
- Refetch frequency.
- Cache lifecycle issues.
- Real-time update failures.
- Authentication and tenant-context transitions.

Observability must:
- Use approved telemetry.
- Avoid logging secrets or sensitive personal data.
- Distinguish expected errors from defects.
- Prevent duplicate error reporting.
- Use safe correlation identifiers where available.
- Avoid telemetry failures interrupting the application.

Follow `architecture/observability.md` when that policy is present in the repository.

---

## 27. Self-Hosting and Environment Configuration

KAMPYN may be deployed as a hosted SaaS product or self-hosted by universities.

Server-state behavior must not depend on hardcoded infrastructure assumptions.

- Use environment-specific API configuration through approved application configuration.
- Avoid embedding secrets in client-side bundles.
- Ensure API base URLs and authentication behavior are configurable.
- Respect self-hosted deployment network constraints.
- Keep tenant scope explicit.
- Avoid assumptions that all deployments use the same domain, identity provider, or infrastructure.
- Ensure query behavior remains compatible with supported deployment configurations.

Client-side environment variables must be treated as public unless the framework explicitly guarantees server-only access.

---

## 28. Common Anti-Patterns

The following patterns are prohibited unless an explicit exception is documented:

- Using Zustand as a duplicate cache for server data.
- Fetching the same server resource through multiple unrelated mechanisms.
- Defining query keys independently throughout components.
- Omitting relevant variables from query keys.
- Using access tokens or secrets in query keys.
- Reusing tenant-sensitive cache entries across tenants.
- Treating cache keys as authorization.
- Treating stale availability as a confirmed booking.
- Optimistically confirming payments or other high-impact operations.
- Automatically retrying unsafe mutations.
- Returning empty results when requests fail.
- Invalidating the entire cache after every mutation without justification.
- Updating cached server data through direct mutation.
- Ignoring query cancellation and race conditions.
- Using unbounded polling.
- Creating uncontrolled refetch loops.
- Persisting sensitive server state without review.
- Hydrating user-specific data across server requests.
- Duplicating API response types throughout features.
- Embedding API requests directly in presentation components.
- Silently swallowing query or mutation errors.
- Displaying raw backend error details.
- Testing only happy paths.
- Allowing obsolete responses to overwrite current tenant or identity state.
- Assuming a client-side request cancellation means the backend operation did not commit.

---

## 29. Code Review Checklist

### Architecture
- [ ] Is TanStack Query used for appropriate server-state operations?
- [ ] Is server state separated from local UI and shared client state?
- [ ] Are API functions separated from components?
- [ ] Are query definitions organized by feature?
- [ ] Are query keys centralized and deterministic?

### Cache
- [ ] Are all query variables represented in the key?
- [ ] Are tenant and identity scopes handled?
- [ ] Are freshness and garbage-collection policies appropriate?
- [ ] Is invalidation targeted?
- [ ] Are direct cache updates safe and immutable?
- [ ] Is sensitive cache data cleared at the correct lifecycle boundaries?

### Queries
- [ ] Are query functions typed and cancellable where appropriate?
- [ ] Are loading, error, empty, and success states handled?
- [ ] Is retry behavior appropriate?
- [ ] Are stale-data implications considered?
- [ ] Are pagination and filtering handled efficiently?

### Mutations
- [ ] Is backend confirmation authoritative?
- [ ] Is duplicate submission handled where necessary?
- [ ] Is retry behavior safe?
- [ ] Are related queries reconciled?
- [ ] Are optimistic updates justified and reversible?
- [ ] Are high-impact operations protected from premature success presentation?

### Security
- [ ] Does the backend remain authoritative for authorization?
- [ ] Is tenant isolation maintained?
- [ ] Are secrets excluded from query keys and client state?
- [ ] Are sensitive caches cleared on identity transitions?
- [ ] Are errors sanitized?
- [ ] Is persisted data reviewed for privacy implications?

### Performance
- [ ] Are requests deduplicated?
- [ ] Are unnecessary refetches avoided?
- [ ] Is cache growth bounded?
- [ ] Are polling and event subscriptions coordinated?
- [ ] Are large responses paginated or appropriately bounded?
- [ ] Are performance optimizations evidence-based?

### Testing
- [ ] Are query and mutation behaviors tested?
- [ ] Are cache invalidation and reconciliation tested?
- [ ] Are tenant and identity transitions tested?
- [ ] Are race conditions and failure scenarios considered?
- [ ] Are important user workflows covered?
- [ ] Are tests isolated from each other?

---

## 30. Definition of Done

A server-state implementation is complete only when:

- [ ] TanStack Query is used according to the approved architecture.
- [ ] Query keys are centralized, deterministic, and correctly scoped.
- [ ] API requests use typed contracts and the approved API client.
- [ ] Loading, error, empty, and success states are handled.
- [ ] Freshness, retention, retry, and refetch policies are explicit.
- [ ] Mutations reconcile data with backend-confirmed results.
- [ ] Optimistic updates are justified, safe, and recoverable.
- [ ] Tenant and identity isolation are enforced in the client lifecycle.
- [ ] Authentication and authorization remain backend-authoritative.
- [ ] Cancellation, stale responses, and concurrency are handled appropriately.
- [ ] Sensitive data is not unnecessarily persisted or exposed.
- [ ] Performance and cache growth have been considered.
- [ ] Relevant unit, integration, and workflow tests pass.
- [ ] Logging and telemetry follow approved privacy requirements.
- [ ] Documentation and related API contracts are updated.
- [ ] No redundant server-state stores, duplicate fetching logic, or unjustified abstractions have been introduced.
- [ ] Any deviation from these standards is documented and approved.

**The objective is to keep KAMPYN's frontend synchronized with backend-owned data while preserving correctness, tenant isolation, performance, and a predictable user experience—even when data changes concurrently or network conditions are unreliable.**
# Frontend Hooks

## Purpose

This document defines the engineering standards for designing, implementing, organizing, testing, and maintaining React hooks across KAMPYN.

Hooks are responsible for encapsulating reusable stateful logic, managing component lifecycle behavior, composing application capabilities, and separating business logic from UI presentation.

KAMPYN uses hooks across authentication, food ordering, bookings, inventory, university administration, community features, search, notifications, and other workflows. Poorly designed hooks can introduce hidden dependencies, unnecessary re-renders, memory leaks, race conditions, and tightly coupled components.

Every hook must be:

- **Cohesive:** Own one clearly defined responsibility.
- **Reusable:** Provide value across multiple consumers when reuse is justified.
- **Predictable:** Have explicit inputs, outputs, state transitions, and side effects.
- **Composable:** Work naturally with other hooks without creating hidden dependencies.
- **Performant:** Avoid unnecessary computation, subscriptions, and re-renders.
- **Testable:** Allow meaningful behavior to be verified independently.
- **Safe:** Handle asynchronous work, cleanup, errors, and lifecycle transitions correctly.
- **Typed:** Expose explicit, strongly typed interfaces.
- **Maintainable:** Remain small, focused, and consistent with the frontend architecture.

This document is the source of truth for frontend hook engineering. It complements the component, forms, state-management, data-fetching, accessibility, testing, and architecture policies.

---

## 1. Core Principles

All hooks must follow these principles:

1. Follow the Rules of Hooks without exception.
2. Use hooks to encapsulate reusable stateful logic, not merely to move code out of components.
3. Keep hooks focused on one primary responsibility.
4. Prefer composition of small hooks over large, monolithic hooks.
5. Keep UI rendering and presentation outside domain-specific hooks.
6. Use TanStack Query for server-state fetching, caching, mutations, and synchronization.
7. Use Zustand for genuinely shared client-side state, not as a default replacement for local state.
8. Use React state for component-local state.
9. Use `useEffect` only to synchronize with external systems, not to derive values that can be calculated during rendering.
10. Clean up subscriptions, event listeners, timers, and other external resources.
11. Avoid unnecessary state duplication and synchronization.
12. Make asynchronous behavior resilient to stale responses, cancellation, and component lifecycle changes.
13. Do not introduce custom hooks for one-off logic unless they materially improve clarity, separation, or testability.
14. Keep dependencies explicit and avoid hidden global coupling.
15. Follow the repository's file-size, security, performance, accessibility, and testing standards.

Hooks must simplify the system. If a hook adds indirection without reducing complexity or improving reuse, keep the logic local.

---

## 2. What Should Be a Hook?

A custom hook is appropriate when logic involves React state, lifecycle, subscriptions, or composition of other hooks and has a clear reason to be extracted.

### 2.1 Appropriate Use Cases

Use custom hooks for:

- Reusable client-side stateful behavior
- Complex component interaction logic
- Subscriptions to external stores or browser events
- Lifecycle synchronization with external systems
- Feature-specific form behavior
- Reusable event-handling logic
- Coordinating TanStack Query operations
- Composing shared state and derived behavior
- Encapsulating reusable UI behavior, such as keyboard navigation
- Managing complex state transitions
- Integrating approved third-party React libraries

Examples:

```text
useAuth()
useCurrentUser()
useBookingAvailability()
useFoodCart()
useDebouncedValue()
useOnlineStatus()
useKeyboardNavigation()
useInventoryFilters()
useComplaintForm()
useNotifications()
```

These names are illustrative. Introduce each hook only when its responsibility and consumers justify it.

### 2.2 When Not to Create a Hook

Do not create a custom hook when:

- The logic is a simple pure function.
- The behavior is used only once and is already clear inside its component.
- The extraction only renames a few lines without improving readability.
- The logic does not need React state, lifecycle, or hook composition.
- A standard React or library hook already solves the problem.
- The hook would simply forward every argument and return value from another hook without adding value.

Use ordinary utility functions for pure transformations, formatting, calculations, and validation.

### 2.3 Hook Versus Utility Function

| Requirement | Preferred approach |
|---|---|
| Pure calculation | Utility function |
| String or date formatting | Utility function |
| Zod schema validation | Schema or utility |
| Component-local state | `useState` or `useReducer` |
| Shared client-side state | Zustand where justified |
| Server-state fetching | TanStack Query |
| Reusable stateful behavior | Custom hook |
| External subscription | Custom hook with proper cleanup |
| Static constants | Constants module |
| Complex state transitions | `useReducer` or a dedicated state abstraction |

Do not wrap pure functions in hooks solely to make them accessible from components.

---

## 3. Hook Naming and Organization

### 3.1 Naming Convention

Every custom hook must begin with `use`, followed by a descriptive name in camelCase.

Preferred:

```ts
useCurrentUser()
useBookingAvailability()
useCartActions()
useDebouncedValue()
```

Avoid:

```ts
currentUser()
bookingHook()
getCart()
handleData()
```

Names must communicate the behavior or capability provided by the hook.

### 3.2 Recommended Structure

```text
src/
├── hooks/
│   ├── use-debounced-value.ts
│   ├── use-online-status.ts
│   └── use-media-query.ts
│
├── features/
│   ├── auth/
│   │   └── hooks/
│   │       ├── use-auth.ts
│   │       └── use-current-user.ts
│   │
│   ├── food/
│   │   └── hooks/
│   │       ├── use-food-cart.ts
│   │       └── use-menu-filters.ts
│   │
│   ├── bookings/
│   │   └── hooks/
│   │       ├── use-booking-availability.ts
│   │       └── use-booking-form.ts
│   │
│   └── inventory/
│       └── hooks/
│           └── use-inventory-filters.ts
│
└── lib/
    └── hooks/
        └── ...
```

This is an illustrative organization. Follow the existing repository structure and place hooks close to the features that own them.

### 3.3 Global Versus Feature Hooks

Place a hook in a shared location only when it is genuinely generic or used across multiple independent features.

Examples of potentially shared hooks:

- `useDebouncedValue`
- `useOnlineStatus`
- `useMediaQuery`
- `usePrevious` where its behavior is clearly needed
- `useEventListener` where it meaningfully standardizes event subscriptions

Feature-specific hooks belong to their feature.

Do not place every hook in a global `hooks/` directory simply because it is a hook.

### 3.4 File Size and Cohesion

Follow the repository's general rule that production source files should normally remain at or below 200 lines.

Split hooks when responsibilities naturally separate, such as:

- Query configuration and feature behavior
- Independent state transitions
- Reusable event subscriptions
- Pure calculations and transformations
- Complex asynchronous workflows

Do not split small hooks into multiple files without a meaningful architectural benefit.

---

## 4. Rules of Hooks

All custom hooks and React components must follow the Rules of Hooks.

### 4.1 Top-Level Invocation

Hooks must be called at the top level of a React component or another custom hook.

Correct:

```tsx
function useProfile() {
  const [name, setName] = useState("");

  return { name, setName };
}
```

Incorrect:

```tsx
function useProfile(enabled: boolean) {
  if (enabled) {
    const [name, setName] = useState("");
    return { name, setName };
  }

  return null;
}
```

### 4.2 No Conditional Hook Calls

Never call hooks inside:

- Conditions
- Loops
- Nested functions
- Event handlers
- Callbacks passed to non-hook APIs
- `try/catch` blocks that change whether hooks execute
- Functions that are not React components or custom hooks

Hook execution order must remain consistent across renders.

### 4.3 Conditional Behavior

When behavior needs to be conditional, call the hook unconditionally and pass a condition through its supported arguments or configuration.

For example, a TanStack Query hook can use an `enabled` option to control whether a query executes.

Do not conditionally call the query hook itself.

### 4.4 React Compiler and Linting

Follow the React version, compiler configuration, and linting rules established by the repository.

Do not disable hook-related lint rules to bypass an architectural or lifecycle issue.

Any exception must be documented and reviewed.

---

## 5. Hook API Design

### 5.1 Explicit Inputs

Hooks must accept clear, strongly typed arguments.

Prefer:

```ts
type UseBookingAvailabilityOptions = {
  locationId: string;
  startDate: string;
  endDate: string;
  enabled?: boolean;
};

function useBookingAvailability({
  locationId,
  startDate,
  endDate,
  enabled = true,
}: UseBookingAvailabilityOptions) {
  // Hook implementation
}
```

Avoid long positional argument lists, ambiguous boolean parameters, and overly broad configuration objects.

### 5.2 Explicit Outputs

Return values must be predictable and documented through TypeScript types.

For example:

```ts
type UseFeatureResult = {
  value: string;
  isLoading: boolean;
  error: Error | null;
  refresh: () => void;
};
```

Use descriptive names rather than generic fields such as `data1`, `result`, or `flag`.

For hooks returning multiple values, prefer a named object when it improves readability, extensibility, and call-site clarity.

Use tuples only when their ordering is intuitive and part of an established convention.

### 5.3 Stable Public Interfaces

Treat exported hook return values and option types as public interfaces within the frontend architecture.

Avoid changing their shape unnecessarily.

If multiple features depend on a hook, changes must account for all consumers and preserve compatible behavior where practical.

### 5.4 Avoid Unnecessary Configuration

Do not expose options that have no meaningful consumer or supported behavior.

Avoid building a highly configurable hook to anticipate hypothetical future use cases.

Add configuration only when it represents a real requirement.

### 5.5 Avoid Hidden Dependencies

Hooks must not silently depend on arbitrary component state, mutable module-level variables, or unrelated feature internals.

Use explicit parameters, approved context providers, or documented shared stores.

When a hook requires context, fail clearly if used outside its required provider unless the context is intentionally optional.

---

## 6. State Management Within Hooks

### 6.1 Choosing State

Select the simplest appropriate state mechanism.

| State requirement | Preferred solution |
|---|---|
| Simple local value | `useState` |
| Complex related transitions | `useReducer` |
| Shared client state | Zustand where justified |
| Server-owned data | TanStack Query |
| Derived values | Calculate during render |
| External mutable store | `useSyncExternalStore` or approved library integration |
| Form values | React Hook Form or simple local state |
| Static configuration | Constant |

Do not duplicate server data in React state or Zustand when TanStack Query already owns it.

### 6.2 Avoid Derived State Duplication

Do not store a value that can be derived from existing state unless there is a clear reason.

Incorrect:

```tsx
const [items, setItems] = useState<Item[]>([]);
const [itemCount, setItemCount] = useState(0);
```

Preferred:

```tsx
const [items, setItems] = useState<Item[]>([]);
const itemCount = items.length;
```

Duplicated state introduces synchronization risks and additional update paths.

### 6.3 useReducer

Use `useReducer` when a hook manages complex related state transitions that are easier to understand as explicit actions.

Examples include:

- Multi-step workflows
- Complex filter transitions
- Local state machines
- Multiple related loading and error states
- Multi-stage interaction logic

Reducers must be pure, deterministic, and free from side effects.

Do not use `useReducer` for trivial state that `useState` handles clearly.

### 6.4 State Ownership

Every state value must have one clear owner.

Do not maintain competing copies of the same state in:

- Component state
- A custom hook
- Zustand
- TanStack Query
- Context

If a value is shared or synchronized, document which layer is authoritative and how updates propagate.

---

## 7. Effects and Lifecycle

### 7.1 Purpose of useEffect

Use `useEffect` to synchronize React with external systems.

Appropriate use cases include:

- Subscribing to browser or external events
- Connecting to a WebSocket
- Managing an external imperative API
- Synchronizing with browser APIs
- Starting and cleaning up external resources
- Integrating non-React libraries

Do not use `useEffect` to derive state that can be calculated during render or to orchestrate ordinary data fetching that TanStack Query already handles.

### 7.2 Dependency Management

All reactive values used by an effect must be represented correctly in its dependency list.

Do not suppress dependency warnings to force an effect to execute at a preferred frequency.

Instead, restructure the effect, stabilize inputs when appropriate, or move pure calculations outside the effect.

### 7.3 Cleanup

Every effect that creates an external resource must clean it up.

Examples:

- Remove event listeners
- Unsubscribe from stores
- Close WebSocket connections when no longer needed
- Clear timers
- Abort cancellable requests
- Dispose of external library resources

Cleanup must be safe when executed more than once.

### 7.4 Strict Mode

Hooks must behave correctly under React Strict Mode, including development-time effect setup and cleanup cycles.

Do not use flags or global variables to conceal lifecycle bugs.

### 7.5 Avoid Effect Chains

Avoid effects that update state only to trigger other effects that update more state.

Prefer direct event handling, reducers, derived values, or explicit state transitions.

Effect chains make behavior difficult to reason about and can cause repeated renders or inconsistent intermediate states.

---

## 8. Asynchronous Hooks

Asynchronous operations must be resilient to component lifecycle changes, request ordering, and failures.

### 8.1 Prefer TanStack Query for Server Data

Use TanStack Query for server-owned data and remote operations.

Custom hooks may encapsulate query configuration and feature-specific behavior, but must not recreate caching, retry, invalidation, and synchronization systems that TanStack Query already provides.

Example:

```ts
function useFoodCourt(foodCourtId: string) {
  return useQuery({
    queryKey: ["food-court", foodCourtId],
    queryFn: () => getFoodCourt(foodCourtId),
    enabled: Boolean(foodCourtId),
  });
}
```

Query keys must include all relevant parameters that affect the result, including tenant or user scope where required by the application's cache-isolation strategy.

### 8.2 Avoid Unmanaged Async Effects

Do not start asynchronous work inside an effect without a clear cancellation, stale-response, or lifecycle strategy.

When an asynchronous effect is necessary:

- Define how obsolete results are handled.
- Use cancellation where supported.
- Prevent stale responses from overwriting newer state.
- Handle expected errors.
- Clean up associated resources.
- Ensure state updates are safe for the current lifecycle.

### 8.3 Race Conditions

For operations such as search, autocomplete, availability lookup, and dependent field loading, ensure older requests cannot overwrite newer results.

Use query keys, cancellation, request identifiers, or another explicit strategy suitable for the operation.

Do not assume that network responses arrive in request order.

### 8.4 Cancellation

Pass `AbortSignal` through supported APIs when cancellation is appropriate.

Cancellation should not be used to conceal incorrect ownership or synchronization.

For operations that may have reached the backend, cancellation does not prove that the operation was rolled back.

### 8.5 Retry Behavior

Retries must reflect the operation's semantics.

- Retry safe, transient reads where appropriate.
- Use bounded retry policies.
- Respect server rate limits.
- Avoid retry storms.
- Do not automatically retry non-idempotent mutations without appropriate safeguards.
- Distinguish network errors from validation, authorization, and business-rule errors.

### 8.6 Unknown Outcomes

For consequential operations, such as order creation, payments, or bookings, a timeout may leave the outcome unknown.

Hooks must rely on the feature's idempotency and reconciliation strategy rather than assuming that a failed client request means the server performed no action.

---

## 9. TanStack Query Hooks

TanStack Query is the standard solution for server-state management.

### 9.1 Query Hooks

Encapsulate repeated query configuration in feature-specific hooks where this improves consistency and maintainability.

A query hook should clearly define:

- Query key
- Query function
- Relevant parameters
- Enabled conditions
- Stale-time and cache policy where required
- Retry behavior where non-default behavior is justified
- Error handling expectations
- Data selection where appropriate

### 9.2 Query Key Design

Query keys must be:

- Deterministic
- Serializable
- Consistent
- Complete for the requested data
- Scoped according to tenant and user isolation requirements

Use shared query-key factories for larger features where they prevent duplication and inconsistent invalidation.

Do not generate query keys from unstable objects or omit parameters that affect the result.

### 9.3 Mutation Hooks

Use mutation hooks for server-side state changes.

Mutation hooks may encapsulate:

- Mutation function
- Input and output types
- Success handling
- Query invalidation
- Optimistic update logic where safe
- Error mapping
- Idempotency requirements

Example:

```ts
function useCreateComplaint() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: createComplaint,
    onSuccess: () => {
      return queryClient.invalidateQueries({
        queryKey: ["complaints"],
      });
    },
  });
}
```

Mutation hooks must not assume that invalidating a query guarantees the business operation was successful.

### 9.4 Optimistic Updates

Use optimistic updates only when the behavior is well understood and safe to reconcile.

They must define:

- Which cached data is modified
- How the previous state is captured
- How rollback works
- How concurrent mutations are handled
- How the server response reconciles the optimistic result

Avoid optimistic updates for high-impact operations when the expected server result cannot be safely predicted.

### 9.5 Cache Invalidation

Invalidate or update only the affected query keys.

Avoid broad cache invalidation when targeted invalidation is possible.

Document complex invalidation dependencies where multiple features share related server data.

### 9.6 Avoid Query Abstraction Overload

Do not create wrappers around every TanStack Query call if they only forward arguments.

Introduce custom hooks when they add meaningful feature ownership, stable keys, reusable policies, type safety, or a clear consumer-facing API.

---

## 10. Zustand and Shared-State Hooks

Zustand is intended for shared client-side state where multiple components or features need access to the same state.

### 10.1 Appropriate Use Cases

Examples include:

- Shopping cart state where shared across multiple screens
- UI preferences
- Client-side workflow state
- Shared navigation or interaction state
- Temporary cross-component selections

Use the feature's established store architecture and keep stores cohesive.

### 10.2 Hook Selectors

Subscribe only to the state needed by the component.

Prefer focused selectors rather than subscribing to an entire store when only a small part is required.

Avoid selectors that construct unstable values unnecessarily or trigger avoidable renders.

### 10.3 Store Boundaries

Do not use Zustand as a replacement for:

- Backend authorization
- Durable persistence
- TanStack Query server-state caching
- React Hook Form
- Local component state

A store must not become a catch-all for unrelated application state.

### 10.4 Store Side Effects

Keep business operations and external side effects organized in accordance with the project's store and service architecture.

Avoid tightly coupling stores to UI components, browser-specific APIs, or unrelated feature internals.

---

## 11. Memoization and Referential Stability

Memoization must be intentional and driven by correctness or measurable performance needs.

### 11.1 useMemo

Use `useMemo` when a computation is sufficiently expensive or when referential stability is required by a specific consumer.

Do not memoize trivial expressions by default.

### 11.2 useCallback

Use `useCallback` when stable function identity provides a concrete benefit, such as interacting with memoized children or APIs that rely on stable callback references.

Do not wrap every function in `useCallback` automatically.

### 11.3 React.memo

Memoizing a component does not guarantee performance improvement if its props are unstable or its rendering cost is negligible.

Use profiling and evidence to justify memoization.

### 11.4 Correctness First

Memoization must never be used to make incorrect dependency handling appear correct.

Do not rely on a memoized value as durable storage or as a substitute for proper state ownership.

### 11.5 React Compiler

Where the repository uses React Compiler, follow its supported optimization model and project configuration.

Do not introduce redundant manual memoization that conflicts with the established compiler strategy.

---

## 12. External Subscriptions and Browser APIs

Hooks that integrate with external systems must define ownership, lifecycle, and cleanup.

### 12.1 Event Listeners

When listening to browser events:

- Register listeners in an appropriate lifecycle boundary.
- Remove listeners during cleanup.
- Use stable references where needed.
- Avoid registering duplicate listeners.
- Handle unavailable browser APIs in server-rendered environments.

### 12.2 Browser Environment

KAMPYN uses Next.js and may render components on the server.

Hooks must not access browser-only APIs during server rendering or module initialization.

Browser APIs such as `window`, `document`, `navigator`, and local storage must be accessed only in an appropriate client-side context.

Do not assume every component using a hook is a Client Component. Respect Next.js Server and Client Component boundaries.

### 12.3 WebSocket Hooks

Hooks that manage WebSocket connections must define:

- Connection ownership
- Authentication handling
- Tenant and user scoping
- Reconnection strategy
- Backoff behavior
- Subscription lifecycle
- Message validation
- Cleanup and disconnection
- Duplicate event handling
- Offline and reconnect states

Do not create a new socket connection for every component render.

Prefer a centralized connection manager or feature-owned provider where multiple consumers share the same connection.

### 12.4 Visibility and Connectivity

Hooks that expose online status, visibility, or other browser signals must account for the limitations of these signals.

For example, online status does not guarantee that the KAMPYN backend is reachable.

Do not treat browser connectivity as proof of successful API access.

### 12.5 External Stores

Use `useSyncExternalStore` or a supported library integration when subscribing to external mutable stores that need to participate in React rendering.

Do not manually mirror external mutable state into React state without a clear synchronization strategy.

---

## 13. Forms and Input Hooks

Hooks may encapsulate reusable form behavior when the logic is sufficiently complex or shared.

### 13.1 Form Ownership

Use React Hook Form for complex forms and local React state for simple inputs, according to the frontend forms policy.

A form hook may own:

- Form configuration
- Zod resolver integration
- Default values
- Form-specific derived behavior
- Field dependency coordination
- Submission orchestration

It must not duplicate the form library's existing state and validation mechanisms.

### 13.2 Input Behavior

Hooks for reusable input behavior may support:

- Debounced values
- Keyboard navigation
- Focus management
- Input formatting where safe
- Character counting
- Accessibility interaction patterns

Keep input behavior independent of unrelated business logic.

### 13.3 Remote Validation

Remote validation hooks must:

- Use appropriate request cancellation or stale-response handling.
- Avoid unnecessary requests.
- Respect rate limits.
- Distinguish validation failure from network failure.
- Avoid exposing sensitive account-existence information.
- Treat backend validation as authoritative.

Do not use remote validation as a substitute for validation during final submission.

---

## 14. Security and Privacy

Hooks must preserve the security boundaries of the frontend architecture.

### 14.1 Authentication and Authorization

Hooks may expose authentication state and coordinate UI behavior, but must not be treated as an authorization boundary.

Every protected operation must be authorized by the backend.

Do not assume that hiding a component or disabling a button prevents unauthorized requests.

### 14.2 Sensitive State

Do not expose sensitive information unnecessarily through hook return values.

Avoid placing secrets, passwords, authentication codes, or confidential user data in:

- Global stores without a justified design
- Browser storage without approval
- Logs
- Analytics events
- Error messages
- Debugging output

### 14.3 Tenant Isolation

Hooks that access tenant-scoped data must respect the application's tenant isolation and cache-scoping strategy.

Do not accept a client-provided tenant identifier as proof of authorization.

Ensure query keys, mutation parameters, and local caches cannot accidentally expose data from another tenant or user.

### 14.4 Untrusted Data

Validate and safely handle data received from external APIs, browser events, storage, WebSockets, and third-party libraries.

Do not assume that a TypeScript type guarantees runtime validity.

### 14.5 Permissions

Hooks may consume permission information for conditional UI behavior, but the backend must independently enforce the relevant permissions.

Do not allow client-side permission state to authorize sensitive operations.

---

## 15. Performance and Scalability

Hooks must be designed to avoid unnecessary work across frequently rendered or highly interactive components.

### 15.1 Render Efficiency

- Keep state close to its consumers.
- Subscribe only to the required store or query state.
- Avoid unnecessary state updates.
- Avoid unstable return objects where referential stability is important.
- Avoid triggering effects from newly created dependencies on every render.
- Avoid excessive context subscriptions.
- Use derived values instead of synchronized duplicate state.

### 15.2 Large Collections

Hooks managing large lists must consider:

- Server-side pagination
- Cursor-based pagination where appropriate
- Filtering and sorting efficiency
- Stable item identity
- Virtualization where rendering volume justifies it
- Memory use
- Query cache size

Do not load entire large datasets into local state without a justified requirement.

### 15.3 High-Frequency Events

Hooks processing frequent events, such as scrolling, typing, resizing, pointer movement, or live updates, must avoid excessive state updates.

Use appropriate throttling, debouncing, batching, or event filtering where it improves performance without changing expected behavior.

### 15.4 Resource Management

Avoid unbounded:

- Timers
- Subscriptions
- Event listeners
- Cached values
- In-flight requests
- WebSocket connections
- Reconnection loops

Every resource must have a defined lifecycle.

### 15.5 Measurement

Use React DevTools, profiling, and relevant application telemetry to identify real bottlenecks.

Do not add complex optimization machinery without evidence.

---

## 16. Error Handling

Hooks must handle expected failures in a consistent and explicit manner.

### 16.1 Error Ownership

Determine whether errors belong to:

- The hook
- The consuming component
- TanStack Query
- A shared error boundary
- A feature-level workflow

Avoid swallowing errors or handling the same failure redundantly across multiple layers.

### 16.2 Error Representation

Use typed or well-defined error representations where appropriate.

Do not expose internal exceptions directly to end users.

Preserve enough diagnostic context for approved observability systems while avoiding sensitive information.

### 16.3 Error Boundaries

Use React error boundaries for rendering failures that should be handled at a component-tree boundary.

Do not expect error boundaries to catch every asynchronous or event-handler failure.

Handle asynchronous errors through the appropriate promise, query, mutation, or subscription lifecycle.

### 16.4 Recovery

Hooks should expose enough information or actions for the consuming UI to provide appropriate recovery behavior, such as:

- Retry
- Refresh
- Reconnect
- Reset
- Cancel
- Reauthenticate

Do not expose meaningless recovery actions that cannot resolve the underlying failure.

---

## 17. Testing

Custom hooks must be tested according to their complexity, risk, and consumer impact.

### 17.1 Unit Tests

Test pure logic independently of React whenever possible.

Examples include:

- Value transformations
- Filtering and sorting
- Derived calculations
- State transition functions
- Validation helpers
- Query-key factories

### 17.2 Hook Tests

Test stateful hooks using the repository's approved React testing tools.

Cover:

- Initial state
- State transitions
- Returned values
- Callback behavior
- Effect setup and cleanup
- Dependency changes
- Conditional behavior
- Error handling
- Async completion
- Cancellation
- Stale-response handling

### 17.3 Integration Tests

Test hooks integrated with their relevant systems where appropriate.

Examples include:

- TanStack Query hooks with a test query client
- Zustand hooks with isolated store state
- Form hooks with schema validation
- WebSocket hooks with a controlled mock connection
- Browser-event hooks with simulated events

Tests must isolate state and avoid leaking subscriptions or caches between test cases.

### 17.4 Lifecycle Tests

Verify that hooks behave correctly when:

- Components mount
- Components unmount
- Inputs change
- Requests overlap
- Requests are cancelled
- Effects are re-established
- Strict Mode triggers development lifecycle checks

### 17.5 Security Tests

For hooks involving authentication, tenant-scoped data, or permissions, verify that UI state does not bypass backend authorization and that client caches are appropriately scoped.

### 17.6 Test Quality

Avoid tests that only assert implementation details or internal hook structure.

Prioritize observable behavior and externally meaningful contracts.

Do not mock away the behavior that the test is intended to verify.

---

## 18. Documentation

Complex, shared, or business-critical hooks must document:

- Purpose and owning feature
- Expected inputs
- Return values
- State ownership
- Side effects
- External dependencies
- Query and mutation behavior
- Error representation
- Cleanup behavior
- Cancellation and retry semantics
- Tenant or permission considerations
- Important performance constraints
- Usage examples where necessary

Keep documentation close to the hook or in the relevant feature documentation.

Avoid duplicating comments that simply restate the TypeScript signature.

---

## 19. Anti-Patterns

The following are prohibited unless an explicit architectural exception is approved:

- Calling hooks conditionally or outside valid React contexts.
- Creating custom hooks solely to rename existing hooks.
- Building monolithic hooks that own unrelated responsibilities.
- Mixing UI rendering with domain and persistence logic.
- Using `useEffect` for values that can be derived during render.
- Suppressing dependency warnings to hide lifecycle bugs.
- Duplicating server state in React state or Zustand without justification.
- Using Zustand as a default replacement for local component state.
- Reimplementing TanStack Query's caching and retry behavior.
- Performing unmanaged asynchronous work in effects.
- Ignoring stale responses and race conditions.
- Leaving subscriptions, timers, or listeners active after unmount.
- Accessing browser-only APIs during server rendering.
- Returning unstable values that cause avoidable renders without a reason.
- Applying memoization indiscriminately.
- Using global mutable variables to coordinate component state.
- Swallowing errors without a recovery or reporting strategy.
- Exposing sensitive information through hook outputs or logs.
- Treating frontend authentication or permission hooks as security boundaries.
- Creating shared hooks that depend on unrelated feature internals.
- Building abstractions for hypothetical future requirements.
- Writing hooks that are difficult to test because state, effects, and external calls are tightly coupled.

---

## 20. Hook Review Checklist

### Architecture
- [ ] The hook has a clear responsibility.
- [ ] The extraction materially improves reuse, clarity, or testability.
- [ ] The hook is located in the correct feature or shared directory.
- [ ] UI, domain, and data responsibilities are separated.
- [ ] The public API is explicit and strongly typed.
- [ ] The implementation remains cohesive and normally within the 200-line limit.

### React Rules
- [ ] The hook follows the Rules of Hooks.
- [ ] Hook calls are unconditional and top-level.
- [ ] Dependency handling is correct.
- [ ] The hook behaves correctly under Strict Mode.
- [ ] No lint rules have been suppressed without justification.

### State
- [ ] State has one clear owner.
- [ ] Derived values are not unnecessarily duplicated.
- [ ] React state, Zustand, and TanStack Query are used appropriately.
- [ ] State transitions are predictable.
- [ ] The hook does not create hidden synchronization dependencies.

### Lifecycle and Async
- [ ] Effects synchronize with external systems only.
- [ ] External resources are cleaned up.
- [ ] Async work handles cancellation where appropriate.
- [ ] Race conditions and stale responses are addressed.
- [ ] Retry behavior is appropriate for the operation.
- [ ] Unknown outcomes are handled safely for consequential actions.

### Security
- [ ] Sensitive data is handled appropriately.
- [ ] Tenant-scoped data follows isolation rules.
- [ ] Backend authorization remains authoritative.
- [ ] Untrusted external data is validated.
- [ ] No sensitive information is exposed through logs or errors.

### Performance
- [ ] Subscriptions are scoped to required state.
- [ ] Unnecessary re-renders are avoided.
- [ ] Large collections are handled efficiently.
- [ ] High-frequency events are managed appropriately.
- [ ] Resources and caches have bounded lifecycles.
- [ ] Optimization complexity is justified.

### Quality
- [ ] Core logic has appropriate tests.
- [ ] Stateful behavior and lifecycle are tested.
- [ ] Error and recovery paths are covered.
- [ ] External dependencies are tested at suitable boundaries.
- [ ] Documentation is sufficient for future maintenance.

---

## 21. Definition of Done

A custom hook is complete only when:

- Its purpose and ownership are clear.
- Its abstraction provides meaningful value.
- Its API is explicit, typed, and predictable.
- It follows the Rules of Hooks.
- Its state has a clear owner.
- Its effects and external resources have correct lifecycles.
- Its asynchronous behavior handles failures and stale results safely.
- Its use of React state, Zustand, and TanStack Query is appropriate.
- It respects security, privacy, and tenant-isolation requirements.
- It does not introduce unnecessary re-renders or resource usage.
- Its behavior is covered by appropriate tests.
- Its documentation is sufficient for its complexity and impact.
- It follows the project's code-quality and architectural standards.

**A hook is not complete merely because it makes a component shorter. It is complete when it encapsulates a clear responsibility while making behavior easier to reuse, understand, test, and maintain.**
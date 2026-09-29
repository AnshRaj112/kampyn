# React Engineering Standards

## 1. Purpose

This document defines the engineering standards for building, maintaining, and scaling React applications within KAMPYN.

React code must be:
- **Composable:** Built from small, reusable, cohesive components.
- **Predictable:** State changes and rendering behavior must be easy to understand.
- **Performant:** Avoid unnecessary renders, expensive computations, and inefficient data handling.
- **Accessible:** All interactive interfaces must support accessible usage.
- **Type-safe:** Use TypeScript to enforce clear contracts and prevent invalid states.
- **Maintainable:** Keep business logic, UI, state management, and data access appropriately separated.
- **Testable:** Components, hooks, and state transitions must be independently verifiable.
- **Secure:** Never treat frontend validation or UI restrictions as a replacement for backend enforcement.

These standards apply to React components, custom hooks, client-side state, UI composition, and React-specific application logic. Next.js-specific conventions are defined in `frontend/nextjs.md`.

---

## 2. Core Principles

All React development must follow these principles:

1. Prefer composition over inheritance.
2. Prefer simple, explicit implementations over clever abstractions.
3. Keep components small, cohesive, and focused on a single responsibility.
4. Keep business logic out of presentation components.
5. Keep state as close as possible to where it is used.
6. Derive values instead of storing duplicate state.
7. Prefer controlled, predictable data flow.
8. Use React's built-in capabilities before introducing additional dependencies.
9. Avoid unnecessary effects and imperative DOM manipulation.
10. Optimize based on measured performance, not assumptions.
11. Make invalid states difficult to represent through TypeScript.
12. Reuse established project components, hooks, utilities, and patterns.
13. Keep React components independent of specific backend implementations.
14. Treat server data and client state as separate concerns.
15. Do not introduce abstractions that make straightforward behavior harder to understand.

---

## 3. Component Architecture

### 3.1 Component Responsibilities

Every component must have a clearly defined responsibility.

A component may:
- Render a UI element.
- Compose other components.
- Manage local interaction state.
- Coordinate feature-specific behavior.
- Connect UI to application-level state through hooks.

A component should not become responsible for unrelated concerns such as authentication, data persistence, business policy, analytics, and rendering.

### 3.2 Component Categories

Use the following conceptual categories:

| Category | Responsibility | Example |
|---|---|---|
| Primitive | Basic, reusable UI building block | `Button`, `Input` |
| UI | Styled interface element | `StatusBadge`, `Avatar` |
| Layout | Page or section arrangement | `PageContainer`, `Stack` |
| Feature | Domain-specific interface | `OrderSummary`, `BookingCard` |
| Composite | Combines multiple UI and feature components | `OrderHistoryPanel` |
| Page | Composes a route-level screen | `OrdersPage` |
| Provider | Exposes shared context | `ThemeProvider` |
| Hook | Encapsulates reusable React behavior | `useDebounce` |

These are responsibility categories, not mandatory folders or separate component hierarchies.

### 3.3 Single Responsibility

Each component must have one primary reason to change.

Avoid components that:
- Render several unrelated features.
- Manage unrelated state domains.
- Contain large conditional rendering trees for multiple workflows.
- Mix complex business calculations with markup.
- Directly implement API protocols, persistence, and UI behavior in one place.

Extract a component when doing so improves readability, reuse, testability, or responsibility boundaries. Do not split every JSX element into its own component.

### 3.4 Composition

Prefer composing focused components over passing numerous flags to one component.

Avoid:

```tsx
<DashboardCard
  isOrder
  isAdmin
  isCompact
  showActions
  showInventory
  isBooking
/>
```

Prefer explicit composition:

```tsx
<OrderCard
  order={order}
  actions={<OrderActions orderId={order.id} />}
/>
```

Use `children` and well-defined slots when they make component responsibilities clearer.

### 3.5 Component Size

- Target fewer than **200 lines per production source file**.
- Treat the limit as an architectural guideline, not a reason to split code arbitrarily.
- When a file exceeds the limit, review its responsibilities and extract cohesive modules.
- Keep components short enough that their behavior can be understood without navigating excessive dependencies.
- Document architectural justification when a file must exceed the limit.

### 3.6 Reuse

Before creating a new component:
1. Search the existing codebase for an equivalent.
2. Check whether an existing component can be extended through composition.
3. Confirm the new component has a distinct responsibility.
4. Avoid duplicating styling, interaction, validation, or accessibility behavior.

Do not create a generic abstraction for a single use case unless it meaningfully improves structure or consistency.

---

## 4. TypeScript Standards

### 4.1 Strict Typing

React code must use TypeScript with strict compiler settings.

- Avoid `any`.
- Prefer `unknown` when the type is not known.
- Narrow unknown values before using them.
- Use explicit types for exported component props, hooks, utilities, and public contracts.
- Prefer inference for straightforward local variables.
- Avoid unsafe type assertions unless they are justified and documented.

### 4.2 Component Props

Use clear, explicit prop interfaces or types.

```tsx
interface OrderCardProps {
  order: Order;
  onViewDetails: (orderId: string) => void;
  isLoading?: boolean;
}

export function OrderCard({
  order,
  onViewDetails,
  isLoading = false,
}: OrderCardProps) {
  return (
    <article>
      <h2>{order.reference}</h2>
      <button
        type="button"
        disabled={isLoading}
        onClick={() => onViewDetails(order.id)}
      >
        View details
      </button>
    </article>
  );
}
```

Guidelines:
- Use descriptive prop names.
- Avoid overly broad configuration objects.
- Prefer required props unless optionality is intentional.
- Define meaningful defaults for optional props.
- Avoid passing entire application stores or service containers into presentational components.
- Avoid using `React.FC` as a default requirement; use ordinary function declarations with typed props.

### 4.3 Discriminated Unions

Use discriminated unions for mutually exclusive states.

```tsx
type RequestState<T> =
  | { status: "idle" }
  | { status: "loading" }
  | { status: "success"; data: T }
  | { status: "error"; message: string };
```

Avoid unrelated boolean flags that allow impossible combinations, such as `isLoading`, `isSuccess`, and `hasError` all being true simultaneously.

### 4.4 Event Types

Use React event types when explicitly declaring event handlers.

```tsx
const handleChange = (
  event: React.ChangeEvent<HTMLInputElement>,
) => {
  setQuery(event.target.value);
};
```

Prefer inferred event types when the handler is declared directly within JSX and the resulting type remains clear.

### 4.5 Avoid Type Duplication

- Reuse canonical domain types from the appropriate application layer.
- Do not independently redefine API response structures across multiple components.
- Keep UI-only types separate from backend domain contracts.
- Use schema-derived types where supported by the project's validation strategy.
- Do not assume frontend types guarantee backend validity.

---

## 5. State Management

### 5.1 State Ownership

State must live at the lowest level that can correctly own it.

Use:
- Component state for isolated UI interactions.
- Lifted state when multiple nearby components must coordinate.
- Context for stable, cross-tree dependencies.
- Zustand for shared client-side application state.
- TanStack Query for remote server state.
- URL state for shareable, navigable, or bookmarkable interface state.

Do not move local component state into a global store without a concrete requirement.

### 5.2 Local State

Use `useState` for simple local state.

```tsx
const [isOpen, setIsOpen] = useState(false);
```

Use `useReducer` when:
- State transitions are related and complex.
- Several actions update the same state.
- Explicit transition logic improves readability.
- The state structure has multiple dependent fields.

Do not use reducers merely to wrap simple state updates.

### 5.3 Derived State

Do not store values that can be calculated from existing state or props.

Avoid:

```tsx
const [items, setItems] = useState<Item[]>([]);
const [itemCount, setItemCount] = useState(0);
```

Prefer:

```tsx
const [items, setItems] = useState<Item[]>([]);
const itemCount = items.length;
```

Derived state reduces synchronization bugs and unnecessary updates.

### 5.4 Avoid Prop Drilling

Do not introduce global state solely to eliminate every prop chain.

Prefer, in order:
1. Direct props for simple and local composition.
2. Component composition and slots for complex trees.
3. Context for genuinely shared dependencies.
4. Zustand for shared client-side state with meaningful cross-feature ownership.

Avoid passing large objects through many layers when only a small, explicit value is needed.

### 5.5 State Separation

Maintain a clear distinction between:

- **Server state:** Data owned by backend services.
- **Client state:** UI preferences, temporary selections, open panels, and local interaction state.
- **URL state:** Filters, pagination, sorting, and shareable navigation state.
- **Form state:** User input, validation feedback, dirty state, and submission state.

Do not copy server responses into Zustand or local state by default. TanStack Query should remain the primary owner of cached server data.

### 5.6 State Mutation

Never mutate React state directly.

Avoid:

```tsx
items.push(newItem);
setItems(items);
```

Prefer:

```tsx
setItems((current) => [...current, newItem]);
```

Use immutable updates for objects, arrays, maps, and nested state. Use established immutable update utilities only when they provide a clear benefit.

### 5.7 Functional Updates

Use functional updates when the next state depends on the previous state.

```tsx
setCount((current) => current + 1);
```

This avoids relying on potentially stale values captured by closures.

---

## 6. React Hooks

### 6.1 Rules of Hooks

All hooks must follow React's Rules of Hooks:

- Call hooks only at the top level of React components or custom hooks.
- Never call hooks conditionally, inside loops, or inside nested functions.
- Keep hook call order stable across renders.
- Use custom hooks to encapsulate reusable React behavior.

Lint rules for hooks must be enabled and enforced.

### 6.2 Custom Hooks

Create a custom hook when behavior:
- Is reused across multiple components.
- Has a distinct lifecycle or state responsibility.
- Is complex enough to obscure the component.
- Benefits from independent testing.

Custom hooks must:
- Start with `use`.
- Have one clear responsibility.
- Expose a minimal and typed API.
- Avoid leaking internal implementation details.
- Handle cleanup for resources they create.
- Document important behavioral constraints.

Avoid creating hooks that simply rename one built-in hook without adding meaningful behavior.

### 6.3 `useEffect`

`useEffect` is intended for synchronizing React with external systems, such as:
- Browser APIs.
- Event subscriptions.
- Timers.
- Third-party imperative libraries.
- External connections or resources.

Do not use effects for:
- Calculating derived values.
- Transforming data for rendering.
- Responding to a user event that can be handled in the event handler.
- Copying props into state without a justified synchronization requirement.
- Chaining state updates that can be calculated directly.

Avoid unnecessary effects because they increase lifecycle complexity and can cause redundant rendering.

### 6.4 Effect Dependencies

- Include every reactive value used by an effect.
- Do not suppress dependency lint rules to silence warnings without a documented reason.
- Keep dependencies stable only when stability is required.
- Avoid recreating objects and functions in ways that cause unintended effect execution.
- Do not use effects to compensate for unclear state ownership.

### 6.5 Effect Cleanup

Every effect that creates a persistent resource must clean it up.

Examples include:
- Event listeners.
- Timers.
- WebSocket subscriptions.
- Observers.
- Third-party library instances.
- In-flight requests where cancellation is supported.

```tsx
useEffect(() => {
  const controller = new AbortController();

  void loadData({ signal: controller.signal });

  return () => {
    controller.abort();
  };
}, []);
```

Ensure cleanup is safe when invoked more than once and compatible with React Strict Mode.

### 6.6 `useMemo` and `useCallback`

Do not use memoization by default.

Use `useMemo` when:
- A calculation is measurably expensive.
- Stable references are required for a specific dependency or memoized child.
- Profiling shows a meaningful rendering benefit.

Use `useCallback` when:
- A stable callback reference is necessary.
- A memoized child benefits from stable props.
- The callback is a dependency of another hook and its identity matters.

Do not wrap every function or expression in memoization. Memoization has its own complexity and maintenance cost.

### 6.7 `useRef`

Use refs for mutable values that do not need to trigger rendering, including:
- DOM references.
- Imperative handles.
- Previous values when genuinely required.
- Timer or subscription handles.

Do not use refs as an alternative state store for values that must be reflected in the UI.

### 6.8 `useLayoutEffect`

Use `useLayoutEffect` only when DOM measurement or synchronous layout synchronization must occur before the browser paints.

Prefer `useEffect` for ordinary synchronization to avoid blocking rendering.

---

## 7. Rendering and Reconciliation

### 7.1 Pure Rendering

Components must remain pure during rendering.

Rendering must not:
- Mutate external state.
- Make API calls.
- Write to browser storage.
- Trigger navigation.
- Register persistent event listeners.
- Perform non-idempotent side effects.

Use event handlers, effects, or framework-specific mechanisms for side effects.

### 7.2 Conditional Rendering

Use clear conditional expressions and early returns where they improve readability.

Avoid deeply nested ternaries and large inline conditional trees.

Extract meaningful conditional sections into focused components when doing so improves comprehension.

### 7.3 List Rendering

- Always use stable, unique keys for rendered lists.
- Prefer persistent entity identifiers.
- Never use array indexes as keys for lists that can reorder, insert, or delete items.
- Do not generate random keys during rendering.
- Avoid unnecessary key changes that remount components and discard state.

```tsx
{orders.map((order) => (
  <OrderCard key={order.id} order={order} />
))}
```

### 7.4 Fragments

Use fragments when grouping elements without adding an unnecessary DOM node.

Avoid extra wrapper elements that interfere with semantics, layout, or accessibility.

### 7.5 Component Identity

Do not define stateful components inside another component unless remounting behavior is intentional.

Defining components within render scope can cause React to treat them as new component types, potentially resetting their state.

### 7.6 Strict Mode

Develop and test with React Strict Mode enabled where supported.

Code must tolerate development-time repeated rendering and effect setup/cleanup. Do not rely on an effect running exactly once as a correctness guarantee.

---

## 8. Data Fetching and Server State

### 8.1 Ownership

The backend is authoritative for:
- Business rules.
- Authorization.
- Persistence.
- Transactional operations.
- Tenant access enforcement.
- Data integrity.

React components must not independently reimplement backend domain rules as if they were authoritative.

### 8.2 TanStack Query

Use TanStack Query for remote server state when the application requires client-side fetching, caching, synchronization, mutations, or background refetching.

Guidelines:
- Use consistent, typed query keys.
- Include relevant tenant and identity scope in cache identity where applicable.
- Centralize query and mutation definitions by feature.
- Configure retry behavior intentionally.
- Invalidate or update caches after successful mutations.
- Handle loading, error, empty, and success states explicitly.
- Avoid duplicate fetching mechanisms for the same data.
- Cancel or ignore obsolete requests where appropriate.

### 8.3 Query Keys

Query keys must be:
- Deterministic.
- Structured.
- Consistent.
- Specific enough to prevent cache collisions.

Example:

```tsx
export const orderKeys = {
  all: ["orders"] as const,
  lists: () => [...orderKeys.all, "list"] as const,
  list: (filters: OrderFilters) =>
    [...orderKeys.lists(), filters] as const,
  detail: (orderId: string) =>
    [...orderKeys.all, "detail", orderId] as const,
};
```

Tenant-sensitive data must not be shared across tenant boundaries through a common cache identity. Cache identity does not replace backend authorization.

### 8.4 Mutations

Mutations must:
- Use explicit input and result types.
- Provide user-visible feedback where appropriate.
- Prevent accidental duplicate submissions when required.
- Handle errors without exposing internal server details.
- Invalidate or update relevant queries.
- Preserve consistency with backend-confirmed results.

Use optimistic updates only when the behavior is well understood and rollback or reconciliation is implemented.

### 8.5 Request Lifecycle

Avoid unmanaged API requests scattered throughout components.

Prefer feature-specific API functions and query hooks that centralize:
- Endpoint contracts.
- Serialization.
- Error normalization.
- Cancellation.
- Authentication integration.
- Query caching policy.

Never assume a successful HTTP response means a business operation was completed unless the API contract guarantees it.

### 8.6 Avoid Waterfalls

- Fetch independent data concurrently where practical.
- Avoid sequential requests when there is no dependency.
- Use backend aggregation endpoints when they meaningfully reduce network round trips.
- Do not introduce client-side orchestration that duplicates backend workflows.
- Measure network waterfalls before optimizing.

---

## 9. Forms and User Input

### 9.1 Form Ownership

Forms must have a clear owner for:
- Input values.
- Validation.
- Submission state.
- Error messages.
- Reset behavior.
- Dirty and touched states.

Use the project's approved form library and validation conventions rather than introducing competing form abstractions.

### 9.2 Validation

- Use Zod or the established schema-validation layer for reusable validation contracts.
- Validate at the appropriate client boundary for user experience.
- Treat backend validation as authoritative.
- Keep validation rules aligned with backend contracts where practical.
- Provide actionable field-level feedback.
- Avoid validating only through visual constraints or HTML attributes.

### 9.3 Submission

- Prevent duplicate submissions where necessary.
- Display a meaningful pending state.
- Preserve user input when a recoverable submission error occurs.
- Avoid clearing the form before a successful response unless the workflow explicitly requires it.
- Handle server-side field errors distinctly from general failures.

### 9.4 Controlled and Uncontrolled Inputs

Choose controlled or uncontrolled inputs intentionally.

- Use controlled inputs when the interface needs immediate state synchronization.
- Use uncontrolled inputs when direct state tracking is unnecessary and the form strategy supports it.
- Avoid mixing control modes during a component's lifecycle.

### 9.5 Accessibility

Every input must have:
- A programmatic label.
- Clear instructions where needed.
- Associated validation feedback.
- Keyboard accessibility.
- An appropriate input type and autocomplete behavior.

Refer to `frontend/forms.md` and `frontend/accessibility.md` for the detailed project conventions.

---

## 10. Context and Providers

### 10.1 Context Usage

Use React Context for stable, broadly shared dependencies such as:
- Theme configuration.
- Localization.
- Authentication presentation context.
- Feature configuration.
- Dependency access where justified.

Do not use Context as a default replacement for all global state.

### 10.2 Provider Design

Providers must:
- Have a clear responsibility.
- Be placed at the narrowest practical scope.
- Expose typed values.
- Avoid unnecessary state and rerenders.
- Avoid deeply nested provider structures without architectural justification.

### 10.3 Context Performance

Context updates can rerender consumers.

Where needed:
- Split unrelated context values.
- Keep provider values stable when appropriate.
- Separate state and dispatch contexts for suitable reducer-based designs.
- Use Zustand for shared state that benefits from selective subscriptions.

Do not add memoization without evidence that it improves behavior.

### 10.4 Provider Safety

Providers must not expose secrets, privileged credentials, or server-only objects to client-side React code.

Authentication or tenant context displayed by the frontend is not a substitute for backend authorization.

---

## 11. Zustand Integration

Zustand is intended for shared client-side state, not as a general replacement for TanStack Query.

Use it for state such as:
- Cross-feature UI preferences.
- Client-side workflow state.
- Shared selections.
- Persistent presentation settings where appropriate.
- Complex client-only interaction state.

Standards:
- Create small, domain-focused stores.
- Define explicit state and action types.
- Use selectors to subscribe to only the required state.
- Avoid storing duplicate server data.
- Avoid exposing unrestricted mutation patterns.
- Keep business invariants enforced by the backend.
- Handle persistence and reset behavior explicitly.
- Clear identity- or tenant-scoped state when the active context changes.

Avoid one giant application store that combines unrelated domains.

---

## 12. Performance Engineering

### 12.1 Performance Principles

Performance work must be evidence-driven.

Prioritize:
1. Correct state ownership.
2. Efficient data access and payload sizes.
3. Avoiding unnecessary rendering.
4. Appropriate component boundaries.
5. Efficient list rendering.
6. Lazy loading and code splitting where useful.
7. Measured optimization of expensive operations.

### 12.2 Unnecessary Renders

Avoid:
- Updating state when the value has not meaningfully changed.
- Storing derived values as separate state.
- Passing unstable object or function references where they cause measured problems.
- Subscribing components to broad global state unnecessarily.
- Creating oversized contexts with frequently changing values.

Use React DevTools Profiler and browser performance tools to identify real bottlenecks.

### 12.3 Large Lists

For large or frequently updated lists:
- Use pagination or cursor-based loading.
- Prefer server-side filtering and sorting for large datasets.
- Use virtualization when rendering volume makes it beneficial.
- Keep row components focused.
- Use stable keys.
- Avoid repeated expensive transformations during render.

Do not load unlimited records into the browser without a clear requirement.

### 12.4 Expensive Computation

- Avoid expensive synchronous work during render.
- Move suitable computations to backend services or workers where appropriate.
- Use memoization only when profiling supports it.
- Prefer efficient algorithms and appropriate data structures.
- Avoid nested iteration with avoidable quadratic complexity.

### 12.5 Bundle and Dependency Cost

- Avoid unnecessary runtime dependencies.
- Prefer tree-shakeable libraries.
- Use dynamic imports or framework-supported lazy loading for suitable heavy features.
- Avoid importing large modules for small functionality.
- Review bundle impact for major dependencies.

### 12.6 Image and Media Rendering

Use the application's approved image and media components.

- Provide appropriate dimensions and responsive behavior.
- Avoid layout shifts.
- Use lazy loading where appropriate.
- Do not render large media assets unnecessarily.
- Respect accessibility and reduced-motion preferences.

---

## 13. Error Handling

### 13.1 Error Boundaries

Use error boundaries to isolate failures in appropriate UI regions.

Error boundaries should:
- Prevent one feature failure from unnecessarily crashing unrelated UI.
- Present a useful fallback.
- Support recovery or retry where practical.
- Log diagnostic information through approved observability mechanisms.
- Avoid exposing internal stack traces or sensitive information to users.

### 13.2 Expected Errors

Handle expected outcomes explicitly, including:
- Network failures.
- Authorization failures.
- Validation errors.
- Conflict responses.
- Missing resources.
- Rate limiting.
- Expired sessions.
- Offline conditions.

Do not treat expected API failures as unhandled rendering exceptions.

### 13.3 Error Presentation

User-facing errors must:
- Explain what happened in understandable terms.
- Avoid leaking implementation details.
- Provide a next step where practical.
- Preserve recoverable input or workflow state.
- Avoid displaying raw backend exceptions.

### 13.4 Logging

- Use centralized logging or telemetry conventions.
- Avoid `console.log` and debug statements in production code.
- Never log access tokens, credentials, private user data, or sensitive request payloads.
- Include safe contextual identifiers where appropriate.
- Prevent duplicate or excessively noisy error reporting.

---

## 14. Security

### 14.1 Trust Boundaries

Everything received from the browser, URL, local storage, client state, or API response must be treated according to its trust boundary.

The frontend is not a security enforcement boundary.

### 14.2 Authorization

- UI visibility checks improve usability only.
- The backend must authorize every protected operation.
- Never rely on hidden buttons, disabled controls, route guards, or client-side roles as the only authorization mechanism.
- Do not assume a tenant ID or user ID supplied by the browser is trustworthy.

### 14.3 Sensitive Data

- Do not store secrets in React state, client bundles, or browser storage.
- Avoid persisting sensitive personal or institutional data unless explicitly required and approved.
- Do not expose internal API keys or service credentials.
- Clear sensitive UI state when a user logs out or changes identity.

### 14.4 XSS

- Avoid injecting untrusted HTML.
- Do not use `dangerouslySetInnerHTML` without a reviewed and appropriate sanitization strategy.
- Escape or sanitize user-generated content according to the rendering context.
- Treat Markdown, rich text, and embedded content as untrusted unless safely processed.

### 14.5 Browser Storage

Use local storage, session storage, IndexedDB, and cookies only for appropriate data.

- Do not store privileged secrets in browser-accessible storage.
- Validate and safely parse persisted values.
- Version persisted state where schema changes are possible.
- Handle unavailable or corrupted storage gracefully.
- Clear relevant persisted state on logout or tenant switch.

### 14.6 External Links

- Validate destinations where URLs are user-controlled.
- Use safe link attributes when opening new tabs.
- Avoid unsafe URL schemes.
- Do not render untrusted destinations without appropriate checks.

---

## 15. Accessibility

Accessibility is a functional requirement, not a finishing step.

React interfaces must:
- Use semantic HTML.
- Support complete keyboard interaction.
- Preserve visible focus.
- Associate labels and descriptions with controls.
- Announce relevant dynamic status updates.
- Manage focus for dialogs, menus, and other modal interactions.
- Avoid inaccessible custom controls when native elements suffice.
- Respect reduced-motion preferences.
- Support zoom and responsive layouts.

Use native elements whenever they provide the required interaction semantics.

Follow the detailed requirements in `frontend/accessibility.md`.

---

## 16. Styling and Design System

### 16.1 Styling Consistency

Use KAMPYN's approved styling system and design tokens.

- Reuse shared design tokens.
- Avoid duplicated colors, spacing, typography, and elevation values.
- Keep styles scoped to their intended component or feature.
- Avoid global styles that unintentionally affect unrelated features.
- Support responsive layouts and theming where required.

### 16.2 Component Styling

- Keep styling concerns separate from business logic.
- Avoid deeply coupled selectors.
- Avoid excessive one-off overrides.
- Use consistent naming and predictable variants.
- Prefer composition over complex prop-driven styling logic.

### 16.3 Design Tokens

Use shared tokens for:
- Color.
- Typography.
- Spacing.
- Border radius.
- Elevation.
- Breakpoints.
- Motion.
- Layering and z-index.

Do not hardcode repeated design values throughout the application.

### 16.4 Responsive UI

- Design for small and large viewports.
- Avoid fixed dimensions unless required.
- Prevent horizontal overflow.
- Ensure touch targets are appropriately sized.
- Validate layouts with long text and translated content.
- Support campus workflows across mobile and desktop devices.

---

## 17. React and Next.js Boundaries

KAMPYN uses Next.js, so React components must respect the framework's execution model.

- Prefer Server Components for server-rendered and data-oriented UI where appropriate.
- Use Client Components only when client-side interactivity, browser APIs, or client hooks are needed.
- Keep client boundaries narrow.
- Do not import server-only modules into client components.
- Do not pass non-serializable values across server/client boundaries.
- Avoid duplicating backend domain logic in client components, Server Actions, or Route Handlers.
- Follow `frontend/nextjs.md` for routing, rendering, server data access, caching, and deployment requirements.

React-specific abstractions must not obscure Next.js execution boundaries.

---

## 18. External Integrations

React components must not directly own complex integration protocols.

Examples include:
- Payment providers.
- Maps.
- Push notifications.
- WebSockets.
- Analytics.
- File uploads.
- Third-party identity providers.

Use feature-specific hooks, adapters, or services to encapsulate integration behavior.

Standards:
- Keep external dependencies behind explicit interfaces.
- Handle loading, failure, timeout, and cleanup.
- Validate returned data at trust boundaries.
- Avoid exposing credentials.
- Ensure integration failures do not corrupt unrelated UI state.
- Keep provider-specific logic out of reusable presentation components.

---

## 19. Testing Standards

### 19.1 Testing Philosophy

Test observable behavior rather than implementation details.

Tests must verify:
- Correct rendering.
- User interactions.
- State transitions.
- Accessibility-relevant behavior.
- Loading, error, empty, and success states.
- Integration with query and state layers.
- Regression-prone edge cases.

### 19.2 Component Tests

Use the project's approved React testing stack.

Component tests should:
- Render components with realistic props.
- Query elements through accessible roles and labels.
- Simulate user interactions.
- Assert visible outcomes.
- Mock external boundaries rather than internal implementation.
- Avoid brittle snapshots as the primary correctness measure.

### 19.3 Hook Tests

Test custom hooks when they contain meaningful behavior.

Cover:
- Initial state.
- State transitions.
- Dependency changes.
- Cleanup.
- Error handling.
- Edge cases.
- Async behavior where applicable.

### 19.4 Integration Tests

Integration tests should cover interactions between:
- Components and custom hooks.
- Forms and validation.
- TanStack Query and feature UI.
- Zustand stores and consumers.
- Authentication or tenant context and protected UI states.
- Error boundaries and recovery behavior.

### 19.5 Accessibility Tests

Include automated accessibility checks where practical, alongside manual keyboard and assistive-technology verification for important workflows.

### 19.6 Avoid Test Anti-Patterns

Do not:
- Test internal hook calls as the main measure of correctness.
- Assert implementation-specific class names without a clear reason.
- Mock every child component.
- Depend on arbitrary timeouts.
- Use snapshots as a substitute for behavioral assertions.
- Write tests that pass while the user-visible workflow is broken.

### 19.7 Coverage

Coverage is a signal, not the sole measure of quality.

Prioritize critical workflows and risk-based coverage over arbitrary percentage targets.

---

## 20. Code Organization

React code must follow the repository's approved feature-oriented structure.

A representative feature layout:

```text
features/
  orders/
    components/
      OrderCard.tsx
      OrderList.tsx
    hooks/
      useOrders.ts
      useOrderDetails.ts
    api/
      orders.api.ts
    schemas/
      order.schema.ts
    types/
      order.types.ts
    utils/
      order.utils.ts
    index.ts
```

Guidelines:
- Group code by domain or feature where appropriate.
- Keep shared components in approved shared locations.
- Keep feature-specific components inside their feature.
- Avoid circular dependencies.
- Avoid importing another feature's internal implementation directly.
- Expose only intentional public APIs from feature entry points.
- Do not create folders solely to satisfy a rigid template when a simpler structure is clearer.

### 20.1 Import Boundaries

- Prefer stable public exports over deep imports across feature boundaries.
- Avoid circular imports.
- Keep dependency direction explicit.
- Do not let shared UI depend on feature-specific business logic.
- Do not let presentation components depend directly on database or infrastructure modules.

### 20.2 Naming

Use consistent naming:
- Components: `PascalCase`.
- Hooks: `useCamelCase`.
- Utilities: `camelCase`.
- Types and interfaces: descriptive `PascalCase`.
- Constants: `UPPER_SNAKE_CASE` where appropriate.
- Component files: match the exported component name when practical.

Names must express intent rather than implementation trivia.

---

## 21. Dependency Management

Before adding a React dependency:
1. Confirm the functionality is needed.
2. Check whether React, Next.js, or existing project utilities already support it.
3. Evaluate maintenance, security, bundle size, accessibility, and licensing.
4. Confirm compatibility with the current framework and toolchain.
5. Prefer a small, focused dependency over a large utility package for one minor feature.
6. Document meaningful architectural dependencies.

Do not introduce multiple libraries that solve the same problem without a documented reason.

---

## 22. Observability

React applications should support diagnosis of production issues without exposing sensitive information.

- Capture actionable client-side errors through approved telemetry.
- Track important user-facing failures where appropriate.
- Use correlation identifiers when safely available.
- Avoid high-volume or duplicate event reporting.
- Do not log sensitive input or authentication material.
- Distinguish expected operational errors from application defects.
- Ensure telemetry failures do not interrupt user workflows.

Observability conventions must align with `architecture/observability.md` when that policy is present in the repository.

---

## 23. Internationalization and Localization

Where localization is supported:
- Avoid hardcoded user-facing strings in reusable components.
- Use the approved translation mechanism.
- Support long and translated text without layout breakage.
- Respect locale-specific date, time, number, and currency formatting.
- Avoid assumptions about text direction.
- Ensure translated labels remain accessible.

Localization must not change business calculations or currency semantics.

---

## 24. Common Anti-Patterns

The following patterns are prohibited unless a documented exception exists:

- Components responsible for unrelated features.
- Giant components containing entire workflows.
- One global store for all application state.
- Duplicating server state in Zustand without a defined synchronization policy.
- Effects used for derived state.
- Effects used to respond to ordinary user actions.
- Direct mutation of React state.
- Array indexes used as keys for dynamic lists.
- Unnecessary `useMemo` and `useCallback` everywhere.
- Unbounded data fetching and rendering.
- API calls scattered throughout presentation components.
- Business rules implemented only in frontend code.
- Client-side authorization treated as security enforcement.
- Unsafe HTML injection.
- Broad Context providers that trigger excessive rerenders.
- Deeply nested conditional rendering that obscures behavior.
- Components defined inside other components when identity stability is required.
- Suppressed hook dependency warnings without justification.
- Unnecessary dependencies and duplicate abstractions.
- Tests coupled to internal implementation details.
- `any` used to bypass type errors.
- Unhandled promises and ignored asynchronous failures.
- Persistent resources created without cleanup.

---

## 25. Code Review Checklist

### Architecture
- [ ] Does each component have a clear responsibility?
- [ ] Is the implementation composed from reusable, cohesive pieces?
- [ ] Are feature and shared-code boundaries respected?
- [ ] Is business logic kept out of presentation components?
- [ ] Are server/client boundaries respected?

### Type Safety
- [ ] Are component props and public APIs explicitly typed?
- [ ] Is `any` avoided?
- [ ] Are state variants modeled safely?
- [ ] Are unsafe assertions justified?
- [ ] Are shared domain types reused appropriately?

### State and Hooks
- [ ] Is state owned at the correct level?
- [ ] Is server state separated from client state?
- [ ] Is derived state calculated rather than duplicated?
- [ ] Are hooks called according to React's Rules of Hooks?
- [ ] Are effects necessary and correctly cleaned up?
- [ ] Is memoization justified?

### Performance
- [ ] Are list keys stable?
- [ ] Is data loading bounded and efficient?
- [ ] Are unnecessary renders avoided?
- [ ] Are expensive calculations justified or optimized?
- [ ] Have performance claims been measured?

### Security
- [ ] Is backend authorization treated as authoritative?
- [ ] Is untrusted content safely rendered?
- [ ] Are secrets and sensitive data protected?
- [ ] Are browser storage and external URLs handled safely?
- [ ] Is tenant-scoped state isolated appropriately?

### Accessibility
- [ ] Is semantic HTML used?
- [ ] Are keyboard and focus behaviors correct?
- [ ] Are controls labeled?
- [ ] Are errors and dynamic updates accessible?
- [ ] Are responsive and reduced-motion behaviors considered?

### Quality
- [ ] Are loading, error, empty, and success states handled?
- [ ] Are tests focused on observable behavior?
- [ ] Are dependencies justified?
- [ ] Are naming and code organization consistent?
- [ ] Is the source file within the 200-line target or justified?
- [ ] Is documentation updated where necessary?

---

## 26. Definition of Done

A React change is complete only when:

- [ ] The implementation follows the component and state architecture.
- [ ] Components have clear responsibilities and typed contracts.
- [ ] State ownership is explicit and duplicate state is avoided.
- [ ] Hooks follow React's rules and effects clean up correctly.
- [ ] Server state is handled through the approved data-fetching conventions.
- [ ] Backend business logic and authorization remain authoritative.
- [ ] Responsive and accessible behavior has been considered.
- [ ] Loading, error, empty, and success states are handled where applicable.
- [ ] Security and tenant-isolation implications have been reviewed.
- [ ] Relevant unit, integration, and accessibility tests pass.
- [ ] Linting, type checking, and production builds pass.
- [ ] Performance optimizations are justified.
- [ ] Documentation and related contracts are updated.
- [ ] No redundant code, unnecessary dependencies, or unjustified abstractions have been introduced.
- [ ] Any deviation from these standards is explicitly documented and approved.

**The objective is not to maximize abstraction or minimize line count. The objective is to build React interfaces that remain predictable, accessible, secure, performant, and maintainable as KAMPYN grows.**
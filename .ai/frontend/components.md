# Frontend Component Architecture & Standards

## 1. Purpose

This document defines the architecture, design principles, implementation standards, composition patterns, lifecycle management, testing requirements, and quality expectations for frontend components across KAMPYN.

KAMPYN uses a modular frontend architecture built around reusable, accessible, strongly typed components. Components MUST remain cohesive, maintainable, testable, performant, and independent of unrelated business logic.

These standards apply to:

- Next.js and React applications.
- Student-facing interfaces.
- University administration dashboards.
- Vendor and food court dashboards.
- Marketing websites.
- Shared UI libraries.
- Feature-specific components.
- SDK-provided UI components.
- Self-hosted university deployments.

All components MUST follow the relevant policies in:

- `.ai/frontend/accessibility.md`
- `.ai/frontend/state-management.md`
- `.ai/frontend/data-fetching.md`
- `.ai/frontend/performance.md`
- `.ai/frontend/styling.md`
- `.ai/frontend/testing.md`
- `.ai/frontend/architecture.md`
- `.ai/architecture/api.md`
- `.ai/architecture/frontend.md`
- `.ai/constitution/03-architecture-principle.md`
- `.ai/constitution/06-definition-of-done.md`

If a referenced document does not yet exist, this document MUST NOT be interpreted as proof that it has been created.

---

## 2. Core Principles

Every frontend component MUST follow these principles.

### 2.1 Single Responsibility

- A component MUST have one clearly defined primary responsibility.
- Components MUST NOT combine unrelated interface concerns.
- Components SHOULD remain small, cohesive, and focused.
- Complex functionality MUST be decomposed into smaller components, hooks, utilities, or feature modules.
- Components MUST NOT become general-purpose containers for unrelated application logic.

### 2.2 Reusability

- Repeated interface patterns SHOULD be implemented as reusable components.
- Shared components MUST have clearly defined APIs and behavior.
- Reuse MUST reduce duplication without introducing unnecessary abstraction.
- Components MUST NOT be generalized for hypothetical use cases without a real requirement.
- Feature-specific components SHOULD remain within their feature until reuse is justified.

### 2.3 Composition Over Complexity

- Prefer composing small components over building large configurable components.
- Prefer explicit component composition over deeply nested conditional rendering.
- Use slots, children, and well-defined props where appropriate.
- Avoid excessive boolean props that create many unpredictable component variations.
- Keep component responsibilities visible at the call site.

### 2.4 Predictability

- Components MUST have clear input and output contracts.
- Side effects MUST be explicit and controlled.
- Components MUST NOT unexpectedly mutate external state.
- Components MUST render consistently for equivalent inputs and state.
- Component APIs MUST avoid surprising behavior and hidden dependencies.

### 2.5 Accessibility by Default

- Shared components MUST provide accessible semantics and interaction behavior by default.
- Accessibility MUST NOT depend on every consumer remembering to add essential attributes.
- Interactive components MUST support appropriate keyboard interactions and focus behavior.
- Components MUST follow `.ai/frontend/accessibility.md`.

### 2.6 Performance by Design

- Components MUST avoid unnecessary renders, effects, subscriptions, and expensive calculations.
- Large lists and complex visualizations MUST use appropriate rendering strategies.
- Components MUST not perform expensive work during every render without justification.
- Performance optimization MUST be based on measured or reasonably anticipated workload.

---

## 3. Component Classification

KAMPYN components MUST be classified according to their responsibility and scope.

| Component type | Responsibility | Examples |
|---|---|---|
| Primitive | Basic reusable interface element | Button, Input, Checkbox |
| UI | Composed visual interface pattern | Card, Modal, Dropdown |
| Layout | Page structure and positioning | Header, Sidebar, PageContainer |
| Feature | Domain-specific user interaction | CartItem, BookingSlot |
| Composite | Coordinates multiple related components | CheckoutForm, SearchPanel |
| Page | Route-level composition and data orchestration | FoodCourtPage |
| Provider | Supplies shared context or application services | ThemeProvider |
| Utility | Non-visual reusable behavior | Formatting, validation helpers |
| Hook | Encapsulates reusable React behavior | useCart, useDebounce |

### 3.1 Primitive Components

Primitive components MUST:

- Provide a small and focused API.
- Avoid domain-specific business logic.
- Support required accessibility behavior.
- Be reusable across appropriate application areas.
- Avoid unnecessary dependencies on application services.
- Support styling through approved design-system mechanisms.

Examples:

- `Button`
- `IconButton`
- `Input`
- `Textarea`
- `Checkbox`
- `Radio`
- `Switch`
- `Badge`
- `Spinner`

### 3.2 UI Components

UI components combine primitives to provide consistent patterns.

Examples:

- `Card`
- `Dialog`
- `DropdownMenu`
- `Tooltip`
- `Tabs`
- `Accordion`
- `Toast`
- `Pagination`

UI components MUST remain domain-independent unless their purpose explicitly requires domain semantics.

### 3.3 Feature Components

Feature components implement a specific business-domain interface.

Examples:

- `FoodItemCard`
- `CartSummary`
- `BookingCalendar`
- `InventoryTable`
- `ComplaintStatus`
- `ShuttleSeatSelector`

Feature components MAY use domain types, feature hooks, and feature services through approved boundaries.

They MUST NOT directly access unrelated features' internal implementation details.

### 3.4 Page Components

Page components compose feature components and coordinate route-specific requirements.

They SHOULD:

- Define the page's major structure.
- Coordinate route-level data requirements.
- Handle page-level loading, error, and empty states.
- Compose reusable feature components.
- Keep business operations within appropriate application or feature layers.

Page components MUST NOT become monolithic implementations of every UI detail.

### 3.5 Providers

Providers MUST be introduced only when shared context is genuinely required.

Examples include:

- Authentication context.
- Theme context.
- Application configuration.
- Localization.
- Query client.
- Tenant configuration.

Rules:

- Providers MUST have a clear ownership and lifecycle.
- Global providers MUST NOT contain unrelated state.
- Provider nesting MUST remain understandable.
- Provider values SHOULD be stable to avoid unnecessary rerenders.
- Providers MUST NOT be used to avoid designing explicit component APIs.

---

## 4. Component File Organization

Components MUST be organized by responsibility and feature ownership.

Recommended structure:

```text
src/
├── app/
│   ├── layout.tsx
│   ├── page.tsx
│   └── (dashboard)/
│       ├── layout.tsx
│       └── food/
│           └── page.tsx
│
├── components/
│   ├── ui/
│   │   ├── button/
│   │   │   ├── button.tsx
│   │   │   ├── button.types.ts
│   │   │   ├── button.module.scss
│   │   │   ├── button.test.tsx
│   │   │   └── index.ts
│   │   ├── input/
│   │   ├── dialog/
│   │   └── table/
│   │
│   ├── layout/
│   │   ├── header/
│   │   ├── sidebar/
│   │   └── page-container/
│   │
│   └── shared/
│       ├── empty-state/
│       ├── error-state/
│       └── loading-state/
│
├── features/
│   ├── food/
│   │   ├── components/
│   │   │   ├── food-item-card/
│   │   │   ├── food-item-list/
│   │   │   └── cart-summary/
│   │   ├── hooks/
│   │   ├── services/
│   │   ├── schemas/
│   │   ├── types/
│   │   └── index.ts
│   │
│   ├── booking/
│   ├── inventory/
│   └── complaints/
│
├── hooks/
├── lib/
├── providers/
├── styles/
└── types/
```

This is a reference structure, not a requirement to create every directory for every application.

### 4.1 Component Folder

A component folder SHOULD contain only files directly related to that component.

Example:

```text
button/
├── button.tsx
├── button.types.ts
├── button.module.scss
├── button.test.tsx
└── index.ts
```

Rules:

- Keep implementation, types, styles, and tests cohesive.
- Do not create empty files or directories for hypothetical future requirements.
- Avoid excessive file splitting for trivial components.
- Keep public exports explicit.
- Use a local index file when it improves import clarity.

### 4.2 Naming Conventions

- React components MUST use PascalCase.
- Hooks MUST use the `use` prefix and camelCase.
- Component files SHOULD use consistent kebab-case or the repository's established naming convention.
- Type names MUST use PascalCase.
- Boolean props SHOULD use descriptive names such as `isLoading`, `isDisabled`, or `hasError`.
- Event callbacks SHOULD use `on` prefixes, such as `onChange` and `onSubmit`.
- Internal handlers SHOULD use action-oriented names such as `handleSubmit` or `handleSelection`.

Naming conventions MUST be consistent within each application or package.

---

## 5. Component Size and Complexity

### 5.1 File Size

- Production source files SHOULD NOT exceed 200 lines without explicit architectural justification.
- Components MUST be decomposed when their size or responsibilities become difficult to understand.
- Tests, generated code, and declarative data files MAY exceed this limit where justified.
- Splitting a component purely to satisfy a line count without improving cohesion is discouraged.

### 5.2 Complexity

- Components MUST avoid deeply nested conditionals.
- Complex rendering branches SHOULD be extracted into named subcomponents or render helpers where appropriate.
- Repeated JSX structures SHOULD be composed into reusable components when doing so improves maintainability.
- Avoid deeply nested ternary expressions.
- Avoid excessive inline callbacks containing business logic.
- Complex calculations MUST be moved into focused utilities or hooks.

### 5.3 Cognitive Load

A component SHOULD be understandable without navigating through numerous unrelated files.

Review component complexity when it has:

- Many unrelated props.
- Multiple independent state variables.
- Complex conditional rendering.
- Several effects with unrelated responsibilities.
- Multiple business workflows.
- Large nested JSX structures.
- Extensive data transformations.
- Tight coupling to several external services.

The goal is clarity and cohesion, not an arbitrary minimum component size.

---

## 6. Component API Design

Component APIs MUST be explicit, predictable, and strongly typed.

### 6.1 Props

- Props MUST be typed using TypeScript.
- Props MUST describe the component's actual responsibilities.
- Avoid passing entire application state objects when only a few fields are needed.
- Avoid overly broad types such as `any`.
- Optional props MUST have clear default behavior.
- Required information MUST be represented as required props.
- Props MUST NOT be silently ignored.
- Components MUST NOT mutate received props.

Example:

```tsx
export interface FoodItemCardProps {
  id: string;
  name: string;
  description?: string;
  price: number;
  imageUrl?: string;
  isAvailable: boolean;
  onAddToCart: (itemId: string) => void;
}

export function FoodItemCard({
  id,
  name,
  description,
  price,
  imageUrl,
  isAvailable,
  onAddToCart
}: FoodItemCardProps) {
  return (
    <article>
      <h3>{name}</h3>
      {description && <p>{description}</p>}
      <p>{price}</p>
      <button
        type="button"
        disabled={!isAvailable}
        onClick={() => onAddToCart(id)}
      >
        Add to cart
      </button>
    </article>
  );
}
```

The example is illustrative. Production components MUST use KAMPYN's approved currency formatting, image handling, design tokens, and accessibility patterns.

### 6.2 Boolean Props

- Boolean props MUST represent clear, independent states.
- Avoid creating components with many unrelated boolean props.
- Do not use boolean props to encode complex modes.
- When multiple mutually exclusive variants exist, prefer a typed variant or discriminated union.
- If combinations of props create invalid states, redesign the API to prevent those combinations.

Avoid:

```tsx
<Dialog
  isModal
  isDrawer
  isPopover
  isFullscreen
/>
```

Prefer an explicit variant when the behavior is mutually exclusive:

```tsx
type DialogVariant =
  | "modal"
  | "drawer"
  | "fullscreen";
```

### 6.3 Event Callbacks

- Event callbacks MUST have explicit types.
- Callback names MUST communicate the action or event.
- Components SHOULD report user intent rather than performing unrelated business operations.
- Callbacks MUST receive only the information the consumer needs.
- Components MUST NOT invoke callbacks unexpectedly during render.
- Async callback behavior MUST be handled explicitly when it affects component state or user feedback.

### 6.4 Children and Composition

Use `children` when consumers need to compose content naturally.

```tsx
interface CardProps {
  children: React.ReactNode;
  title?: string;
}

export function Card({ children, title }: CardProps) {
  return (
    <section className="card">
      {title && <h2>{title}</h2>}
      {children}
    </section>
  );
}
```

Rules:

- Composition MUST remain semantically valid.
- Components MUST document required child structure when necessary.
- Avoid ambiguous children APIs.
- Use named slots or explicit subcomponents for complex composition.
- Do not use `React.cloneElement` as a default composition strategy.

### 6.5 Ref Forwarding

- Components MUST expose refs only when consumers have a valid need.
- Refs MUST be typed correctly.
- Refs MUST NOT be used to bypass component contracts or application state management.
- Focusable components SHOULD support focus control where required for accessibility.
- Ref forwarding MUST preserve native element behavior where appropriate.

---

## 7. Component State Management

Component state MUST be kept at the narrowest appropriate scope.

### 7.1 Local State

Use React state for transient state owned by a single component or a small component subtree.

Examples:

- Dropdown open state.
- Password visibility.
- Temporary input values.
- Active tab.
- Local accordion state.
- Temporary form interaction state.

Rules:

- State MUST have a clear owner.
- State MUST NOT be duplicated unnecessarily.
- Derived values SHOULD be calculated from existing state instead of stored independently.
- State MUST remain consistent with the component's props and lifecycle.

### 7.2 Shared State

Use an approved shared state solution only when state genuinely needs to be shared across unrelated component branches or routes.

KAMPYN may use:

- Zustand for appropriate client-side shared state.
- TanStack Query for server-state caching and synchronization.
- React context for limited, stable shared configuration or scoped context.

The choice MUST follow `.ai/frontend/state-management.md` when available.

### 7.3 Server State

- Server data MUST NOT be copied into local component state without a clear reason.
- TanStack Query SHOULD manage server-state fetching, caching, invalidation, and synchronization.
- Components SHOULD consume feature-level query hooks rather than implementing repeated data-fetching logic.
- Mutations MUST invalidate or update affected query data through explicit strategies.
- Loading, error, stale, and empty states MUST be handled.

### 7.4 Derived State

- Derived values SHOULD be computed from their source state.
- Do not use effects to synchronize values that can be calculated during render.
- Use memoization only when there is a demonstrated or meaningful performance benefit.
- Derived state MUST NOT become a competing source of truth.

### 7.5 Controlled and Uncontrolled Components

- Components MUST document whether they are controlled, uncontrolled, or support both modes.
- Controlled components MUST reflect their value from props.
- Uncontrolled components MAY manage their own internal state.
- A component MUST NOT switch unexpectedly between controlled and uncontrolled modes.
- Controlled behavior MUST remain predictable during rerenders.

---

## 8. Component Lifecycle and Side Effects

Side effects MUST be explicit and carefully scoped.

### 8.1 Effects

Use React effects only to synchronize with external systems.

Appropriate use cases include:

- Browser event subscriptions.
- External widget integration.
- WebSocket lifecycle management.
- Imperative browser APIs.
- Third-party library synchronization.

Avoid effects for:

- Deriving values from props or state.
- Triggering ordinary event-driven actions.
- Duplicating server-state synchronization.
- Performing calculations that can occur during rendering.
- Updating state that can be calculated directly.

### 8.2 Cleanup

- Every effect that creates a subscription or resource MUST clean it up.
- Event listeners MUST be removed when no longer required.
- Timers MUST be cleared when appropriate.
- WebSocket connections MUST be managed according to the owning lifecycle.
- Aborted requests MUST not update stale component state.
- Cleanup MUST be safe when invoked more than once.

### 8.3 Strict Mode

- Components MUST behave correctly under React Strict Mode.
- Effects MUST not rely on being executed exactly once.
- Development-only repeated rendering MUST NOT create duplicate business operations.
- User-triggered mutations MUST NOT be initiated as an incidental side effect of rendering.

### 8.4 Async Behavior

- Asynchronous operations MUST handle success, failure, and cancellation where relevant.
- Components MUST avoid updating state based on stale asynchronous results.
- Loading and submission state MUST be consistent.
- Unhandled promise rejections MUST be prevented.
- Duplicate operations MUST be guarded when necessary.

---

## 9. Component Composition

Composition SHOULD be the primary mechanism for building complex interfaces.

### 9.1 Small Components

Complex interfaces SHOULD be decomposed into cohesive parts.

For example, a checkout page may consist of:

```text
CheckoutPage
├── CheckoutHeader
├── OrderSummary
│   ├── OrderItem
│   └── PriceBreakdown
├── DeliveryDetails
├── PaymentMethodSelector
├── CheckoutActions
└── CheckoutStatus
```

Each component MUST have a defined responsibility and an appropriate ownership boundary.

### 9.2 Avoid Prop Drilling

- Avoid passing unrelated state through multiple layers solely to reach a deeply nested component.
- Prefer composition, scoped context, or a suitable shared state solution when it meaningfully improves architecture.
- Do not introduce global state simply to avoid a small amount of prop passing.
- Shared state MUST have an explicit owner and lifecycle.

### 9.3 Compound Components

Compound components MAY be used when a group of components shares meaningful state or behavior.

Example:

```tsx
<Tabs.Root defaultValue="overview">
  <Tabs.List>
    <Tabs.Trigger value="overview">
      Overview
    </Tabs.Trigger>
    <Tabs.Trigger value="orders">
      Orders
    </Tabs.Trigger>
  </Tabs.List>

  <Tabs.Content value="overview">
    <Overview />
  </Tabs.Content>

  <Tabs.Content value="orders">
    <Orders />
  </Tabs.Content>
</Tabs.Root>
```

Rules:

- Compound components MUST have documented relationships.
- Child components MUST be used within the intended root.
- Context MUST remain scoped to the compound component.
- Keyboard and accessibility behavior MUST be implemented consistently.
- Invalid composition SHOULD be detected early where practical.

### 9.4 Render Props

Render props MAY be used when a consumer genuinely needs control over rendering while sharing behavior.

- Use render props sparingly.
- Prefer normal composition or custom hooks when they provide a simpler API.
- Render props MUST have clear TypeScript types.
- Avoid render-prop chains that make component ownership difficult to understand.

---

## 10. Server and Client Components

KAMPYN Next.js applications MUST use server and client components intentionally.

### 10.1 Server Components

Server components SHOULD be the default when client-side interactivity is unnecessary.

Use them for:

- Static and server-rendered content.
- Server-side data access through approved application boundaries.
- Layout composition.
- Non-interactive content.
- SEO-relevant marketing pages.

Rules:

- Server components MUST NOT use client-only React hooks.
- Server components MUST NOT expose server secrets to client code.
- Data passed to client components MUST be safe for client exposure.
- Server components SHOULD minimize unnecessary client JavaScript.

### 10.2 Client Components

Client components MUST be used when browser-side interactivity is required.

Examples:

- Local interactive state.
- Browser event handling.
- Client-side form interactions.
- Client-only libraries.
- Zustand stores.
- TanStack Query consumers where client-side query behavior is required.
- Browser-dependent APIs.

Rules:

- The `"use client"` directive MUST be placed only where required.
- Client boundaries SHOULD remain as narrow as practical.
- Avoid marking an entire page or layout as a client component for a small interactive element.
- Client components MUST NOT import server-only modules.
- Client component props MUST be serializable when crossing server-to-client boundaries, as required by the framework.

### 10.3 Boundary Design

- Server components SHOULD compose smaller client components for interactive sections.
- Client components SHOULD receive only the data required for their functionality.
- Shared modules MUST clearly separate server-only and client-safe code.
- Server/client boundaries MUST be documented when they affect architectural decisions.
- Hydration behavior MUST be tested for interactive components.

---

## 11. Styling and Visual Consistency

Component styling MUST follow KAMPYN's centralized design-system rules.

### 11.1 Styling

- Components MUST use approved styling conventions, such as the repository's established Tailwind CSS and SCSS Modules approach.
- Styling MUST remain scoped to the component or intentionally shared.
- Avoid global styles for component-specific presentation.
- Use design tokens for colors, spacing, typography, borders, and focus states.
- Avoid hardcoding repeated visual values.
- Do not introduce a second styling methodology without architectural approval.

### 11.2 Variants

- Component variants MUST be explicit and typed.
- Variants SHOULD cover actual product requirements.
- Avoid variants that duplicate the entire component implementation.
- Avoid uncontrolled combinations of size, color, state, and layout props.
- Visual variants MUST preserve accessibility and interaction behavior.

Example:

```tsx
type ButtonVariant =
  | "primary"
  | "secondary"
  | "outline"
  | "ghost"
  | "destructive";

type ButtonSize =
  | "sm"
  | "md"
  | "lg";
```

### 11.3 Responsive Behavior

- Components MUST support applicable responsive layouts.
- Fixed dimensions MUST be avoided unless required by the design.
- Content MUST remain usable when text expands.
- Interactive elements MUST remain accessible at narrow widths.
- Responsive behavior MUST not alter semantic relationships or create confusing focus order.

### 11.4 Themes

- Components MUST support approved themes.
- Theme-specific values MUST use design tokens.
- Components MUST NOT assume a fixed background color.
- Focus, disabled, error, and selected states MUST remain distinguishable in every supported theme.

---

## 12. Component Data Boundaries

Components MUST not become direct interfaces to the entire application infrastructure.

### 12.1 Data Access

- Components SHOULD consume data through feature hooks, server components, or explicitly defined props.
- Components MUST NOT contain raw database access.
- Components MUST NOT embed direct database queries.
- Components MUST NOT duplicate API client configuration.
- Components SHOULD NOT construct arbitrary network requests within rendering logic.
- Data-fetching behavior MUST remain within approved service and query boundaries.

### 12.2 Data Transformation

- Domain-level transformations MUST reside in appropriate feature utilities or services.
- Presentation-only transformations MAY occur in components.
- Complex filtering, sorting, and aggregation SHOULD be extracted.
- Currency, date, time, and localized formatting MUST use shared utilities.
- Components MUST NOT silently alter authoritative business values.

### 12.3 Validation

- Input validation MUST follow the application's approved schema and validation strategy.
- Zod MAY be used to validate runtime data at appropriate boundaries.
- TypeScript types MUST NOT be treated as runtime validation.
- Validation schemas SHOULD be shared where appropriate without coupling unrelated layers.
- Components MUST present validation feedback but MUST NOT replace authoritative backend validation.

---

## 13. Error, Loading, and Empty States

Every data-driven component MUST define relevant asynchronous and failure states.

### 13.1 Loading

- Loading states MUST communicate progress where appropriate.
- Avoid unnecessary layout shifts.
- Skeletons SHOULD represent the expected content structure.
- Loading indicators MUST not imply false progress.
- Loading controls MUST remain accessible.

### 13.2 Error

- Errors MUST be presented clearly and respectfully.
- User-facing messages MUST not expose stack traces, internal identifiers, or sensitive implementation details.
- Retry actions SHOULD be provided when the failure is recoverable.
- Error components MUST expose meaningful accessible names and statuses.
- Error boundaries SHOULD isolate failures to appropriate UI regions.

### 13.3 Empty

- Empty states MUST distinguish between no data, no search results, and missing permissions where appropriate.
- Empty states SHOULD explain the situation.
- A relevant next action SHOULD be provided when possible.
- Empty states MUST not be confused with loading or failure states.

### 13.4 Success

- Important successful actions MUST provide appropriate confirmation.
- Confirmation MUST reflect the actual operation result.
- Components MUST not display success before an authoritative mutation has succeeded.
- Critical confirmation information SHOULD remain available for reference when appropriate.

---

## 14. Performance Standards

Components MUST be designed to avoid unnecessary resource usage and rendering overhead.

### 14.1 Rendering

- Components MUST avoid unnecessary state updates.
- Component trees SHOULD remain appropriately scoped.
- Expensive calculations MUST be measured and optimized where needed.
- Large lists SHOULD use pagination, virtualization, or incremental loading where appropriate.
- Rendering MUST not depend on unstable or unnecessary object and function creation when it causes measurable issues.

### 14.2 Memoization

- `React.memo`, `useMemo`, and `useCallback` MUST NOT be applied indiscriminately.
- Memoization SHOULD be introduced when it prevents meaningful repeated work or rerenders.
- Dependencies MUST be complete and correct.
- Memoization MUST NOT be used to conceal incorrect state ownership or unstable architecture.
- Performance improvements SHOULD be validated through profiling or relevant measurements.

### 14.3 Lazy Loading

- Large or rarely used components SHOULD be lazy-loaded when doing so improves performance.
- Heavy client-side libraries SHOULD be imported only where needed.
- Suspense and loading boundaries MUST provide appropriate user feedback.
- Lazy loading MUST not create inaccessible or confusing focus behavior.

### 14.4 Lists

- Lists MUST use stable keys derived from persistent item identity where available.
- Array indexes MUST NOT be used as keys for reorderable, insertable, or deletable lists.
- Large lists MUST have a deliberate rendering strategy.
- Item components SHOULD avoid receiving large, unrelated objects.
- Pagination and virtualization MUST preserve keyboard and assistive-technology usability.

---

## 15. Security and Privacy

Components MUST not weaken KAMPYN's security or privacy boundaries.

- Sensitive data MUST NOT be stored in component state longer than necessary.
- Authentication tokens MUST NOT be exposed through UI props or rendered markup.
- Components MUST NOT render untrusted HTML without an approved sanitization strategy.
- User-generated content MUST be safely rendered.
- Sensitive form inputs MUST not be unnecessarily logged.
- Client-side authorization checks MUST NOT replace backend authorization.
- Tenant-specific information MUST be scoped through trusted application context.
- Error messages MUST not reveal internal infrastructure details.
- Destructive actions SHOULD provide clear confirmation where the impact warrants it.

Components MUST follow `.ai/constitution/05-security-bar.md` and the applicable security policies.

---

## 16. Component Testing

Every reusable component MUST have tests appropriate to its complexity and risk.

### 16.1 Unit and Component Tests

Test:

- Correct rendering.
- Required and optional props.
- Conditional states.
- User interactions.
- Callback behavior.
- Loading and error states.
- Validation feedback.
- Accessibility semantics.
- Responsive variants where testable.
- State transitions.

### 16.2 Interaction Tests

- Test through user-visible behavior rather than relying exclusively on internal implementation details.
- Prefer accessible queries such as role, label, and visible text.
- Verify keyboard interaction for custom widgets.
- Verify focus movement and restoration for dialogs and menus.
- Verify duplicate-action protection for critical controls.

### 16.3 Integration Tests

Feature components SHOULD be tested within their intended feature context.

Verify:

- Query and mutation integration.
- State synchronization.
- Error recovery.
- Navigation and routing behavior.
- Permission-dependent rendering.
- Tenant-aware data presentation.
- Interaction with sibling components.

### 16.4 Visual Regression

Visual regression testing MAY be used for:

- Shared UI primitives.
- Design-system components.
- Complex dashboards.
- Responsive layouts.
- Theme variants.
- High-impact workflows.

Visual regression MUST supplement functional and accessibility testing, not replace them.

### 16.5 Test Independence

- Tests MUST be deterministic where practical.
- Tests MUST not depend on production services.
- External integrations MUST be mocked or isolated at appropriate boundaries.
- Tests MUST clean up resources and state.
- Shared test utilities SHOULD reduce duplication without hiding test intent.

---

## 17. Component Documentation

Reusable components MUST have sufficient documentation for other developers to use them correctly.

Documentation SHOULD include:

- Component purpose.
- Intended usage.
- Props and types.
- Required and optional configuration.
- Variants and states.
- Accessibility behavior.
- Keyboard interactions.
- Styling and theming behavior.
- Examples.
- Known limitations.

Complex or high-impact components MUST document relevant edge cases and interaction contracts.

Documentation MUST remain consistent with actual component behavior.

---

## 18. Component Reuse and Abstraction

Reuse MUST be driven by genuine shared behavior and design requirements.

### 18.1 When to Extract

Extract a component when:

- A meaningful interface pattern is repeated.
- A feature has a clearly separable responsibility.
- A complex section can be made easier to understand through composition.
- A shared accessibility or interaction contract needs centralized implementation.
- Independent testing or maintenance would benefit from separation.

### 18.2 When Not to Extract

Avoid extraction when:

- A component is used once and is already clear.
- The abstraction introduces more complexity than it removes.
- The component requires excessive configuration to support unrelated cases.
- The abstraction hides important domain behavior.
- A small JSX expression would be clearer than a separate component.

### 18.3 Shared Component Governance

- Shared components MUST have an identifiable owner.
- Changes to shared APIs MUST consider all consumers.
- Breaking changes MUST be deliberate and documented.
- Deprecated props MUST have a migration path.
- Shared component changes MUST include appropriate regression tests.
- Application-specific behavior MUST NOT be added to shared components without a valid cross-feature requirement.

---

## 19. Prohibited Anti-Patterns

The following practices are prohibited unless an explicit architectural exception is approved.

- Components responsible for multiple unrelated business workflows.
- Monolithic components containing excessive rendering, state, and side-effect logic.
- Repeated copies of the same meaningful UI behavior across features.
- Premature abstraction for hypothetical future use.
- Excessive boolean props that create unpredictable variants.
- `any` types for public component contracts without justification.
- Mutating props or externally owned state.
- Triggering business mutations during render.
- Using effects to derive ordinary state.
- Performing raw infrastructure access directly inside UI components.
- Duplicating server state in local state without a clear requirement.
- Unnecessary global state for local interactions.
- Arbitrary DOM manipulation to replace React state management.
- Unstable list keys for mutable lists.
- Uncontrolled use of memoization.
- Broad client-component boundaries without justification.
- Custom interactive controls without accessibility behavior.
- Untrusted HTML rendering without an approved sanitization strategy.
- Swallowing errors without appropriate handling.
- Components that silently ignore required props or unsupported states.
- Deeply coupled components that depend on unrelated feature internals.
- Unnecessary dependencies introduced for trivial functionality.

---

## 20. Component Review Checklist

### Architecture

- [ ] The component has a clear single responsibility.
- [ ] Its classification and ownership are clear.
- [ ] The component is located in the appropriate directory.
- [ ] Its file size and complexity are reasonable.
- [ ] Reuse is justified by real requirements.
- [ ] Dependencies follow approved architecture boundaries.

### API and Types

- [ ] Props are strongly typed.
- [ ] Public APIs are explicit and predictable.
- [ ] Boolean props do not create invalid combinations.
- [ ] Callback contracts are clear.
- [ ] Props are not mutated.
- [ ] Required states and defaults are documented.

### State and Effects

- [ ] State is owned at the correct scope.
- [ ] Derived values are not duplicated unnecessarily.
- [ ] Server state uses the approved data-fetching approach.
- [ ] Effects synchronize only with external systems.
- [ ] Side effects have appropriate cleanup.
- [ ] Async behavior handles failures and stale results.

### Accessibility

- [ ] Semantic HTML is used appropriately.
- [ ] Interactive elements have accessible names.
- [ ] Keyboard behavior is implemented.
- [ ] Focus behavior is correct.
- [ ] Accessible states and descriptions are exposed.
- [ ] Reduced-motion and responsive requirements are considered.

### Styling and Performance

- [ ] Approved design tokens and styling conventions are used.
- [ ] Responsive layouts are supported.
- [ ] Themes preserve accessibility.
- [ ] Rendering work is reasonable.
- [ ] Lists use stable keys.
- [ ] Memoization and lazy loading are justified.

### Security and Reliability

- [ ] Untrusted content is safely handled.
- [ ] Sensitive data is protected.
- [ ] Client-side checks do not replace backend authorization.
- [ ] Errors are handled safely.
- [ ] Critical actions prevent accidental duplicate execution where necessary.

### Testing and Documentation

- [ ] Component tests cover relevant behavior.
- [ ] Interaction and accessibility tests are included.
- [ ] Feature integration has been considered.
- [ ] Public component APIs are documented.
- [ ] Known limitations are recorded.
- [ ] Related architecture documentation is updated.

---

## 21. Definition of Done

A component is considered complete only when:

- It has a clear purpose and responsibility.
- Its API is strongly typed and predictable.
- It follows KAMPYN's component architecture.
- It uses appropriate composition and state ownership.
- It respects server/client boundaries.
- It follows accessibility standards.
- It uses approved design-system and styling conventions.
- It handles relevant loading, error, empty, and success states.
- It meets reasonable performance expectations.
- It respects security and privacy boundaries.
- It has appropriate unit, component, integration, or end-to-end test coverage.
- It has sufficient documentation for its intended consumers.
- It introduces no unnecessary duplication or abstraction.
- It complies with the relevant engineering policies.

**Final rule:** Every KAMPYN component MUST be cohesive, reusable where justified, accessible by default, strongly typed, testable, and predictable. Components are the foundation of the frontend architecture and MUST make the overall system easier to maintain rather than shifting complexity into hidden dependencies or oversized abstractions.
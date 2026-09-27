# KAMPYN Frontend Architecture

## 1. Purpose

This document defines the frontend architecture for KAMPYN.

The frontend is responsible for:

- User interaction.
- Presentation.
- Client-side navigation.
- Form interaction.
- API consumption.
- Server-state synchronization.
- Client-side UI state.
- Runtime validation.
- Authentication-aware presentation.
- Accessibility.
- Responsive behavior.
- Search and discovery interfaces.
- Real-time user experiences where required.

The frontend must not become the owner of business rules that belong to the backend.

The backend remains authoritative for:

- Business rules.
- Authorization.
- Data integrity.
- Resource ownership.
- Financial operations.
- Inventory allocation.
- Booking decisions.
- Security-sensitive state transitions.

---

# 2. Frontend Technology

The primary frontend stack is:

```text
Next.js
React
TypeScript
Tailwind CSS
SCSS / CSS Modules
TanStack Query
Zustand
Zod
```

Additional libraries may be introduced when they solve a clearly identified requirement.

Dependencies must not be introduced merely to avoid writing small amounts of well-understood code.

---

# 3. Frontend Architecture

The frontend should follow a layered structure:

```text
┌──────────────────────────────┐
│         UI / Pages           │
├──────────────────────────────┤
│       Feature Modules        │
├──────────────────────────────┤
│     Client State / Queries   │
├──────────────────────────────┤
│       API / Contracts        │
├──────────────────────────────┤
│      Shared UI / Utils       │
└──────────────┬───────────────┘
               ↓
          Backend API
```

Conceptually:

```text
Presentation
    ↓
Feature
    ↓
State / Data Access
    ↓
API Contract
    ↓
Backend
```

The exact directory structure may evolve, but dependency direction must remain clear.

---

# 4. Server vs Client Responsibility

Next.js provides both server and client execution environments.

Use server-side capabilities when they provide a meaningful benefit.

Use client components when the UI requires:

- Browser APIs.
- Local interaction.
- Event handlers.
- Client-side state.
- Real-time interaction.
- Interactive forms.
- Client-side animations.

Do not mark entire page trees as client components unnecessarily.

Prefer the smallest possible client boundary.

---

# 5. Server Components

Server components should be preferred for UI that does not require client-side interaction.

Good candidates include:

- Static content.
- Marketing pages.
- SEO-sensitive pages.
- Server-fetched public data where appropriate.
- Layout structures.
- Read-only content.

Server components should avoid unnecessary client-side JavaScript.

---

# 6. Client Components

Client components are appropriate when the component requires:

- `useState`.
- `useEffect`.
- Browser APIs.
- Event handlers.
- Interactive forms.
- Client-side state.
- TanStack Query.
- Zustand.
- Real-time subscriptions.

Keep client components focused.

Do not turn an entire application route into a client component merely because one child component needs browser interaction.

---

# 7. Feature-Based Organization

Frontend code should be organized around meaningful features rather than arbitrary technical categories.

Prefer:

```text
features/
├── auth/
├── ordering/
├── inventory/
├── bookings/
├── food-courts/
├── hostel/
├── library/
├── shuttle/
├── complaints/
├── community/
├── search/
└── notifications/
```

over a structure where all components, hooks, and API functions from unrelated domains are mixed together.

A feature should own its UI-specific behavior and data access where practical.

---

# 8. Shared Components

Shared components should contain genuinely reusable behavior.

Examples:

```text
Button
Modal
Dialog
Input
Select
Table
Pagination
LoadingState
EmptyState
ErrorState
```

Do not place feature-specific components in a generic shared directory simply to avoid moving them later.

Reuse should be based on demonstrated commonality.

---

# 9. Design System

KAMPYN should maintain a consistent design system.

Centralize:

- Typography.
- Spacing.
- Colors.
- Borders.
- Shadows.
- Radius.
- Interactive states.
- Form controls.
- Layout primitives.

Components should consume design tokens rather than repeatedly inventing visual values.

---

# 10. Styling

Tailwind should be used for utility-based styling where appropriate.

SCSS/CSS Modules may be used for:

- Complex component styling.
- Stateful visual systems.
- Large style compositions.
- Cases where local stylesheet organization improves maintainability.

Avoid mixing multiple styling approaches arbitrarily within the same component.

---

# 11. Component Responsibility

A component should primarily handle:

- Presentation.
- User interaction.
- UI composition.

Avoid placing substantial business logic directly inside JSX.

Prefer:

```text
Component
   ↓
Hook / Feature Logic
   ↓
API / State Layer
```

rather than:

```text
Component
   ↓
HTTP request
   ↓
business calculation
   ↓
authorization decision
   ↓
state mutation
```

---

# 12. Component Size

Production source files should normally remain below 200 lines.

If a component exceeds the threshold:

1. Identify separate responsibilities.
2. Extract meaningful subcomponents.
3. Extract reusable hooks where appropriate.
4. Move domain-independent logic into suitable modules.

Do not split a component into meaningless fragments solely to satisfy a line count.

---

# 13. Props

Props should be:

- Explicit.
- Typed.
- Minimal.
- Meaningful.

Avoid passing large objects when only a few fields are required.

Prefer:

```text
UserCard
  userId
  name
  avatarUrl
```

over:

```text
UserCard
  entireUserObject
```

when the component does not need the complete object.

---

# 14. TypeScript

TypeScript is the default language for frontend application code.

Avoid:

```text
any
```

unless there is a documented and unavoidable boundary.

Prefer:

- Explicit interfaces.
- Type aliases.
- Discriminated unions.
- Generics.
- Narrowing.
- Branded/domain types where useful.

Do not use TypeScript types as a substitute for runtime validation.

---

# 15. Runtime Validation

Zod should be used at untrusted runtime boundaries where validation is required.

Examples:

```text
API responses
URL parameters
Form submissions
External integration data
Local persisted state
Environment configuration
```

Conceptually:

```text
Unknown Data
    ↓
Zod Validation
    ↓
Typed Application Data
```

TypeScript provides compile-time guarantees.

Zod provides runtime guarantees.

They solve different problems.

---

# 16. API Contracts

Frontend API clients must follow the backend API contracts.

Do not manually duplicate API semantics across components.

Prefer:

```text
Component
    ↓
Feature Hook
    ↓
API Client
    ↓
Backend Contract
```

The API layer should centralize:

- Request construction.
- Authentication handling.
- Serialization.
- Error normalization.
- Response validation where required.

---

# 17. API Client

Create a reusable API client rather than scattering raw `fetch` calls throughout the application.

Avoid:

```text
Component A → fetch()
Component B → fetch()
Component C → fetch()
Component D → fetch()
```

Prefer:

```text
Components
    ↓
Feature Hooks
    ↓
API Client
    ↓
HTTP Layer
```

The client should remain thin and predictable.

---

# 18. TanStack Query

TanStack Query should be the primary mechanism for server state.

Use it for:

- API data.
- Fetching.
- Caching.
- Background refetching.
- Mutation state.
- Query invalidation.
- Request deduplication.
- Pagination.
- Infinite queries.

Do not copy server responses into Zustand merely to make them globally accessible.

---

# 19. Query Ownership

Each feature should define its query behavior.

Example:

```text
ordering/
├── api/
├── queries/
├── mutations/
└── components/
```

Query definitions should be reusable across pages that consume the same server resource.

---

# 20. Query Keys

TanStack Query keys must be deterministic and include every parameter that changes the result.

Example:

```text
[
  "food-items",
  tenantId,
  foodCourtId,
  filters,
  sort
]
```

Do not create multiple incompatible query-key conventions for the same resource.

Centralize query-key factories where they improve consistency.

---

# 21. Query Invalidation

Mutations must invalidate or update affected queries intentionally.

Example:

```text
Update Food Item
      ↓
Mutation succeeds
      ↓
Invalidate food-item queries
```

Do not invalidate the entire application cache after every mutation.

Invalidate the smallest relevant scope.

---

# 22. Optimistic Updates

Optimistic updates may be used when:

- The expected result is predictable.
- The operation is reversible.
- Failure can be rolled back safely.
- User experience materially benefits.

The client must preserve a reliable rollback path.

Do not optimistically represent security-sensitive or irreversible state transitions without careful design.

---

# 23. Zustand

Zustand should manage client-owned state.

Appropriate examples:

- UI preferences.
- Sidebar state.
- Modal state.
- Temporary workflow state.
- Local filters where URL state is not more appropriate.
- Client-only interaction state.

Do not use Zustand as the primary server-state cache.

---

# 24. State Ownership

Every piece of state should have one clear owner.

Use:

```text
Server state
→ TanStack Query

Client/UI state
→ Zustand / local React state

URL state
→ Next.js routing/search params

Form state
→ Form-specific state management
```

Avoid maintaining the same state simultaneously in multiple systems.

---

# 25. URL State

State that should be:

- Shareable.
- Bookmarkable.
- Navigable.
- Searchable.

should generally live in the URL.

Examples:

```text
/search?q=burger
/orders?status=pending
/food-courts?id=123
```

Do not store navigation-critical state only inside Zustand.

---

# 26. Forms

Forms should have:

- Explicit validation.
- Clear field errors.
- Loading states.
- Submission feedback.
- Accessible labels.
- Keyboard support.
- Server error handling.

Zod schemas may be shared between form validation and API-boundary validation where appropriate.

Do not assume client validation replaces server validation.

---

# 27. Authentication

Authentication state must be derived from the backend authentication architecture.

The frontend may display:

- Login state.
- User information.
- Session state.
- Loading state.

It must not independently decide that a user is authorized to perform a protected operation.

Authorization must be enforced by the backend.

---

# 28. Authorization-Aware UI

The frontend may hide or disable UI based on known permissions to improve user experience.

However:

```text
Hidden button
≠
Authorization
```

Every protected API operation must be authorized server-side.

Frontend permission state must never be trusted as a security boundary.

---

# 29. Tenant Context

The active tenant/institution must be explicit in the frontend architecture.

Tenant context may influence:

- API requests.
- Query keys.
- Navigation.
- UI configuration.
- Branding.
- Feature availability.

Tenant context must never be inferred solely from untrusted client input.

The backend must independently validate tenant access.

---

# 30. Multi-Tenant Query Isolation

Tenant context must be included wherever cached client state can otherwise collide.

For example:

```text
[
  "menu",
  tenantId,
  foodCourtId
]
```

rather than:

```text
[
  "menu",
  foodCourtId
]
```

when the same resource identifier can exist across tenants.

---

# 31. Loading States

Every asynchronous UI flow should define its loading behavior.

Possible states:

```text
Initial Loading
Background Refresh
Submitting
Loading More
Refreshing
```

Avoid replacing the entire interface with a loading spinner when only a small section is changing.

---

# 32. Error States

Errors must be represented intentionally.

Distinguish between:

```text
Validation Error
Authentication Error
Authorization Error
Not Found
Conflict
Rate Limited
Network Failure
Server Failure
```

The UI should provide an appropriate response for each.

Do not expose raw backend error messages indiscriminately.

---

# 33. Empty States

Empty data is not necessarily an error.

Examples:

```text
No orders yet
No available rooms
No search results
No complaints
No notifications
```

Empty states should communicate what happened and, where appropriate, what the user can do next.

---

# 34. Suspense and Streaming

Next.js streaming and Suspense may be used where they improve perceived performance.

Use them intentionally.

Avoid creating excessive nested loading boundaries that make the interface visually unstable or difficult to understand.

---

# 35. Navigation

Navigation should use the framework's routing mechanisms.

Avoid manually manipulating browser history unless required.

Routes should represent meaningful application resources.

Examples:

```text
/food-courts
/food-courts/:id
/orders
/orders/:id
/bookings
/bookings/:id
```

Avoid deeply nested routes without a clear information architecture reason.

---

# 36. Search

Search interfaces should separate:

```text
Input State
     ↓
Debounced Query
     ↓
TanStack Query
     ↓
Search API
     ↓
Results
```

Do not issue a network request for every keystroke without debouncing or another appropriate strategy.

Search state that should be shareable should generally be represented in URL parameters.

---

# 37. Real-Time Features

Real-time features may use:

- WebSockets.
- Server-Sent Events.
- Polling.
- Push mechanisms.

Choose based on actual freshness requirements.

Examples:

```text
Order status
Chat
Notifications
Live availability
```

Real-time updates should synchronize with TanStack Query or feature state rather than creating a second uncontrolled source of server state.

---

# 38. Real-Time Reconnection

Real-time connections must handle:

- Disconnects.
- Reconnection.
- Backoff.
- Authentication expiry.
- Duplicate messages.
- Missed events.

The client must not assume that a persistent connection remains healthy indefinitely.

---

# 39. Event Synchronization

When real-time events update server state:

```text
Event
  ↓
Validate
  ↓
Update / Invalidate Query
  ↓
UI Re-renders
```

Do not independently mutate large amounts of duplicated client state when query invalidation is safer and simpler.

---

# 40. Accessibility

All user-facing components should support accessibility.

Consider:

- Semantic HTML.
- Keyboard navigation.
- Focus management.
- Labels.
- Accessible names.
- Screen readers.
- Contrast.
- Error announcements.
- Reduced motion.
- Touch targets.

Accessibility must be part of component design, not an afterthought.

---

# 41. Responsive Design

KAMPYN should support the required device classes through responsive layouts.

Avoid building separate implementations for every screen size unless the interaction model genuinely differs.

Prefer:

```text
Shared component
      ↓
Responsive layout
```

over duplicated desktop/mobile component trees.

---

# 42. Mobile Experience

Campus users may primarily access KAMPYN through mobile devices.

Mobile interfaces should consider:

- Touch interaction.
- Network variability.
- Smaller screens.
- Reduced bandwidth.
- Battery usage.
- Loading performance.
- Offline or degraded states where appropriate.

---

# 43. Performance

Frontend performance must be treated as an architectural concern.

Consider:

- JavaScript bundle size.
- Server rendering.
- Client component boundaries.
- Network requests.
- Image size.
- Font loading.
- React rendering.
- Data fetching.
- Large lists.
- Hydration cost.

Do not optimize through arbitrary memoization.

Measure before introducing complex optimizations.

---

# 44. React Rendering

Avoid unnecessary rerenders caused by:

- Unstable props.
- Large global state subscriptions.
- Incorrect Zustand selectors.
- Unnecessary context updates.
- Recreating expensive values.

Use:

```text
useMemo
useCallback
React.memo
```

only when they address a demonstrated rendering problem or a clear architectural requirement.

---

# 45. Large Lists

Large datasets should not be rendered entirely into the DOM.

Use:

- Pagination.
- Infinite scrolling.
- Virtualization.
- Incremental rendering.

The appropriate technique depends on the interaction model.

Examples include:

```text
Orders
Search results
Community posts
Notifications
Inventory
```

---

# 46. Images

Images should be optimized appropriately.

Consider:

- Responsive sizes.
- Modern formats.
- Lazy loading.
- Dimensions.
- Compression.
- CDN/object storage delivery.

Do not ship original high-resolution images when a smaller representation is sufficient.

---

# 47. Frontend Caching

Frontend caching should follow:

```text
Server State
    ↓
TanStack Query
```

and:

```text
Client State
    ↓
Zustand / React State
```

Do not create multiple independent caches for the same API resource without an explicit reason.

See:

```text id="b6q2n8"
architecture/caching.md
```

for platform-level caching architecture.

---

# 48. Error Boundaries

Use error boundaries around meaningful UI boundaries where a component failure should not crash the entire application.

Examples:

```text
Application
 ├── Navigation
 ├── Food Ordering
 ├── Community
 └── Profile
```

A failure in one independently recoverable section should not necessarily destroy unrelated application state.

---

# 49. Browser APIs

Browser-specific APIs must remain inside appropriate client boundaries.

Examples:

- `localStorage`.
- `sessionStorage`.
- Geolocation.
- Notifications.
- Clipboard.
- Media APIs.
- WebSocket APIs.

Do not access browser globals from server-rendered code.

---

# 50. Local Storage

Never store sensitive authentication secrets in local storage unless the authentication architecture explicitly requires and secures that model.

Local storage is accessible to JavaScript running in the origin.

Prefer secure authentication mechanisms defined by the authentication architecture.

---

# 51. Side Effects

Side effects should be isolated.

Examples:

```text
API calls
Analytics
Subscriptions
Browser storage
WebSocket connections
Timers
```

Avoid performing uncontrolled side effects during rendering.

Use appropriate lifecycle boundaries.

---

# 52. Data Fetching

Data fetching should be predictable.

Avoid fetching the same resource independently from multiple components when a shared query can provide the data.

Prefer:

```text
Feature Query
     ↓
TanStack Query
     ↓
Multiple Consumers
```

This improves deduplication and consistency.

---

# 53. Request Cancellation

Long-running or obsolete requests should be cancellable where the underlying client supports it.

Examples:

```text
Search
Route changes
Autocomplete
Rapid filter changes
```

Do not allow abandoned requests to overwrite newer UI state.

---

# 54. API Error Normalization

The API client should normalize backend errors into predictable frontend structures.

For example:

```text
ApiError
├── code
├── message
├── status
├── details
└── requestId
```

Components should not need to understand raw HTTP implementation details.

---

# 55. Environment Configuration

Frontend configuration must distinguish between:

```text
Public configuration
Private server configuration
```

Only intentionally public values may be exposed to browser bundles.

Never expose:

- Private API keys.
- Database credentials.
- Internal service credentials.
- Signing secrets.

---

# 56. Feature Flags

Feature flags should have:

- Clear ownership.
- Defined scope.
- Predictable defaults.
- Expiration or cleanup expectations.

Avoid permanent feature-flag branches that become an alternate architecture.

---

# 57. Internationalization

If KAMPYN supports multiple institutions or regions, user-facing text should not be scattered as inaccessible hardcoded strings where localization is expected.

Consider:

- Locale.
- Date formatting.
- Number formatting.
- Currency.
- Time zones.
- Text expansion.
- Right-to-left support if required.

Do not introduce internationalization infrastructure before requirements justify it.

---

# 58. Time and Dates

The frontend must distinguish between:

- Instant in time.
- Local date.
- Local time.
- Date-time in a specific timezone.

Never assume browser local time represents the institution's business timezone.

Format dates according to the relevant tenant or user context.

---

# 59. Authentication Redirects

Authentication redirects must prevent open redirect vulnerabilities.

Only redirect to trusted application destinations.

Do not blindly redirect to a user-provided URL after login.

---

# 60. Frontend Security

Frontend security must account for:

- XSS.
- CSRF where applicable.
- Token exposure.
- Unsafe HTML.
- Untrusted URLs.
- File uploads.
- Third-party scripts.
- Dependency vulnerabilities.

Never use unsafe HTML rendering for untrusted content without appropriate sanitization.

---

# 61. User-Generated Content

Community, chat, reviews, complaints, and other user-generated content must be treated as untrusted.

The frontend should:

- Escape/render safely.
- Avoid unsafe HTML.
- Handle malicious URLs.
- Display moderation states.
- Avoid executing embedded content.

The backend remains responsible for authoritative validation and moderation enforcement.

---

# 62. File Uploads

Frontend file uploads should validate:

- File size.
- File type.
- File count.
- User intent.

Client-side validation improves UX but does not replace server-side validation.

Uploads should use the backend/object-storage architecture defined by the platform.

---

# 63. Analytics

Analytics instrumentation should be centralized where possible.

Events should use stable names and schemas.

Avoid scattering arbitrary analytics calls throughout components.

Analytics must not leak sensitive information.

---

# 64. Testing

Frontend architecture should support:

- Component tests.
- Hook tests.
- API client tests.
- Query behavior tests.
- Form validation tests.
- Accessibility tests.
- Integration tests.
- End-to-end tests.

Test behavior rather than implementation details.

---

# 65. Generated Code

Generated API clients, types, or schemas must be clearly separated from handwritten source.

Do not manually modify generated files unless the generation workflow explicitly supports it.

The source schema remains authoritative.

---

# 66. Dependency Management

Frontend dependencies must have a clear purpose.

Before adding a package, determine:

- Whether existing code already solves the problem.
- Bundle impact.
- Maintenance status.
- Security implications.
- License.
- Compatibility with Next.js and React.
- Whether the abstraction is actually necessary.

Prefer the existing platform and standard APIs when they are sufficient.

---

# 67. Frontend Architecture Boundaries

The frontend must not:

- Directly access databases.
- Directly access Redis.
- Directly access OpenSearch.
- Contain backend secrets.
- Implement authoritative authorization.
- Implement financial invariants.
- Implement authoritative inventory allocation.
- Depend on backend internals.

The frontend communicates through defined application contracts.

---

# 68. Self-Hosted Frontend

KAMPYN may be self-hosted by universities.

The frontend must therefore support:

- Configurable API endpoints.
- Institution-specific branding where supported.
- Environment-specific configuration.
- Authentication provider configuration.
- Feature configuration.
- Version compatibility.

Self-hosted configuration must not require modifying application source code.

---

# 69. Frontend Deployment

Deployment should produce deterministic artifacts.

Consider:

- Build-time configuration.
- Runtime configuration where supported.
- Asset caching.
- CDN behavior.
- Versioning.
- Source maps.
- Error reporting.
- Environment separation.

Production builds must not accidentally contain development credentials or debugging configuration.

---

# 70. Frontend Observability

Frontend observability may include:

- JavaScript errors.
- API failures.
- Route performance.
- Core Web Vitals.
- Client-side latency.
- Failed mutations.
- Real-time connection failures.

Observability data must avoid exposing sensitive user information.

---

# 71. Frontend Change Checklist

Before completing a frontend change:

- [ ] Correct feature/module is identified.
- [ ] Server/client boundary is intentional.
- [ ] Component responsibilities are clear.
- [ ] Reusable components are reused where appropriate.
- [ ] File size remains maintainable.
- [ ] TypeScript types are explicit.
- [ ] Runtime validation exists at required boundaries.
- [ ] TanStack Query owns server state.
- [ ] Zustand owns only appropriate client state.
- [ ] URL state is used where appropriate.
- [ ] API calls use the shared API layer.
- [ ] Loading states are handled.
- [ ] Error states are handled.
- [ ] Empty states are handled.
- [ ] Authorization is not trusted from the frontend.
- [ ] Tenant isolation is preserved.
- [ ] Accessibility is considered.
- [ ] Responsive behavior is considered.
- [ ] Performance implications are understood.
- [ ] Large lists are bounded or virtualized where necessary.
- [ ] User-generated content is rendered safely.
- [ ] Sensitive configuration is not exposed.
- [ ] Tests are updated.
- [ ] Documentation is updated where architecture changed.

---

# 72. Final Frontend Principle

The KAMPYN frontend should be a fast, accessible, maintainable client of the platform's APIs.

The architectural relationship is:

```text
                     KAMPYN Frontend
                            │
        ┌───────────────────┼───────────────────┐
        ↓                   ↓                   ↓
     Next.js             React UI          Client State
        │                   │              Zustand
        │                   │
        └───────────┬───────┘
                    ↓
              Feature Layer
                    ↓
            TanStack Query
                    ↓
              API Client
                    ↓
              Zod Boundary
                    ↓
               KAMPYN API
                    ↓
          Authoritative Backend
```

The frontend should follow:

```text
Presentation
    ↓
Feature Logic
    ↓
Server / Client State
    ↓
API Contract
    ↓
Backend
```

while keeping authoritative business behavior on the server.

The core principles are:

```text
Clear boundaries
Strong typing
Runtime validation
Single state ownership
Accessible UI
Predictable data fetching
Minimal client JavaScript
Secure rendering
Responsive design
Measured performance
```

A good KAMPYN frontend should make the platform feel simple to users while keeping the underlying architecture disciplined and predictable.
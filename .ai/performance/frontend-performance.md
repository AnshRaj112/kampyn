# Frontend Performance Engineering

## 1. Purpose

This document defines the performance engineering standards for the KAMPYN frontend. It establishes measurable performance targets, architectural requirements, optimization practices, and testing procedures to ensure a fast, responsive, accessible, and resource-efficient user experience.

Frontend performance must be considered throughout the development lifecycle, from architecture and component design to implementation, testing, deployment, and monitoring.

These standards apply to:

- Next.js applications and App Router architecture.
- React components, hooks, and client-side interactions.
- TypeScript application code.
- TanStack Query and Zustand state management.
- API communication and data fetching.
- Rendering, hydration, and navigation.
- Asset delivery, caching, and code splitting.
- Real-time updates, notifications, and community features.
- Responsive web applications across mobile, tablet, and desktop.
- Multi-tenant deployments and university self-hosted installations.

The objective is not to optimize every operation prematurely. The objective is to prevent avoidable performance problems through sound architectural decisions and to measure optimizations against real workloads.

---

## 2. Core Principles

### 2.1 Performance Is a Requirement

- Performance is a functional quality requirement, not a final polishing task.
- Every significant feature must consider rendering cost, network overhead, memory consumption, and interaction latency.
- Performance-sensitive decisions must be supported by profiling, benchmarking, or documented workload assumptions.
- Performance improvements must not compromise correctness, security, accessibility, or maintainability.

### 2.2 Optimize Real Bottlenecks

- Measure before optimizing.
- Identify the actual bottleneck before modifying implementation.
- Prefer improvements that reduce work rather than merely moving it between components.
- Avoid speculative memoization, unnecessary caching, and premature architectural complexity.
- Re-measure after every significant optimization to verify its impact.

### 2.3 Server-First Architecture

- Use Next.js Server Components by default.
- Introduce Client Components only when browser interactivity, local state, or browser APIs require them.
- Keep client-side JavaScript as small as reasonably possible.
- Move data retrieval, authorization, and suitable computation to the server.
- Avoid transferring data to the browser when the UI does not need it.

### 2.4 Minimize Unnecessary Work

- Avoid redundant renders, requests, computations, and state updates.
- Reuse existing data and shared abstractions where appropriate.
- Batch operations when doing so reduces overhead without harming responsiveness.
- Use appropriate data structures and algorithms for the expected workload.
- Prefer bounded operations and resource consumption.

### 2.5 Optimize for Real Users

Performance must be evaluated across:

- Low-end and mid-range mobile devices.
- Slow or unstable network connections.
- High-latency API environments.
- Large datasets and long-running user sessions.
- Different browser engines and viewport sizes.
- University environments with varying network infrastructure.

Desktop performance alone is not sufficient evidence of a performant application.

---

## 3. Performance Budgets and Targets

All performance targets must be measured using repeatable methods and representative workloads.

### 3.1 Core Web Vitals

KAMPYN should target the following Core Web Vitals at the 75th percentile of real-user measurements.

| Metric                          |        Target | Description                                            |
| ------------------------------- | ------------: | ------------------------------------------------------ |
| Largest Contentful Paint (LCP)  | ≤ 2.5 seconds | Time until the largest visible content element renders |
| Interaction to Next Paint (INP) |      ≤ 200 ms | Responsiveness to user interactions                    |
| Cumulative Layout Shift (CLS)   |         ≤ 0.1 | Visual layout stability                                |

These are field-performance targets. Lab measurements should be used to diagnose issues but must not be treated as a substitute for real-user monitoring.

### 3.2 Additional Performance Metrics

| Metric                       | Engineering target                          |
| ---------------------------- | ------------------------------------------- |
| First Contentful Paint (FCP) | ≤ 1.8 seconds                               |
| Time to First Byte (TTFB)    | ≤ 800 ms                                    |
| Initial JavaScript           | Minimize; enforce route-specific budgets    |
| Hydration                    | Limit to necessary interactive regions      |
| API response latency         | Defined per endpoint and workload           |
| Long tasks                   | Minimize tasks that block the main thread   |
| Layout shifts                | Prevent avoidable movement during loading   |
| Memory usage                 | Remain stable during extended user sessions |

These are engineering targets and diagnostic indicators. Actual budgets must account for hosting, device capability, network conditions, and route complexity.

### 3.3 JavaScript Budgets

- Establish initial JavaScript budgets for each major route.
- Track compressed and uncompressed JavaScript sizes.
- Avoid allowing shared bundles to grow without justification.
- Separate large, infrequently used features from critical route code.
- Review bundle changes during pull requests.
- Treat significant budget increases as architectural decisions requiring justification.

Suggested initial budgets:

| Resource                               | Suggested initial budget |
| -------------------------------------- | -----------------------: |
| Initial route JavaScript, compressed   |                 ≤ 200 KB |
| Shared JavaScript, compressed          |                 ≤ 150 KB |
| Initial CSS, compressed                |                 ≤ 100 KB |
| Non-critical route-specific JavaScript |           Load on demand |
| Large third-party libraries            | Explicit review required |

These are starting budgets, not universal limits. Adjust them using measured performance on representative devices and networks.

### 3.4 Route-Level Budgets

Each major route must have a defined performance budget appropriate to its complexity.

Examples:

- Marketing pages should prioritize fast rendering, minimal JavaScript, and SEO.
- Food discovery pages should prioritize fast initial content and efficient search interactions.
- Checkout pages should prioritize responsiveness, correctness, and predictable state transitions.
- Admin dashboards should prioritize efficient data loading, rendering, and filtering.
- Community and chat interfaces should prioritize interaction responsiveness and stable memory usage.
- Analytics pages should prioritize bounded datasets, efficient visualization, and controlled computation.

A route must not inherit an excessive budget merely because another route requires more JavaScript.

---

## 4. Rendering Architecture

### 4.1 Server Components by Default

- Use Server Components for static, data-driven, and non-interactive UI.
- Fetch authorized data on the server where appropriate.
- Keep database and internal service access on the server.
- Avoid adding `"use client"` to high-level layouts or route trees without a clear requirement.
- Keep Client Components close to the interactive feature that needs them.
- Pass only the data required by the Client Component.

### 4.2 Client Component Boundaries

Client Components are appropriate for:

- Interactive forms and validation feedback.
- Search inputs with client-side interaction.
- Dialogs, dropdowns, and interactive menus.
- Zustand stores and local interactive state.
- TanStack Query features requiring client-side synchronization.
- Real-time subscriptions and browser notifications.
- Browser APIs such as geolocation, media access, and local storage.

Client Components must:

- Have a clearly defined interaction requirement.
- Avoid importing server-only modules.
- Receive serializable props when crossing the Server-to-Client boundary.
- Avoid pulling large dependency trees into shared client bundles.
- Keep state and effects scoped to the smallest practical subtree.

### 4.3 Rendering Strategy Selection

Choose rendering strategies based on data freshness, personalization, interaction, and caching requirements.

| Strategy                        | Appropriate use                                                |
| ------------------------------- | -------------------------------------------------------------- |
| Static rendering                | Public, relatively stable content                              |
| Incremental Static Regeneration | Public content that can be refreshed periodically              |
| Server-side rendering           | Personalized or request-specific content                       |
| Client-side rendering           | Highly interactive, browser-dependent interfaces               |
| Streaming                       | Pages with independently loadable sections                     |
| Hybrid rendering                | Routes combining server-rendered data with client interactions |

Do not use client-side rendering by default when the content can be rendered on the server.

### 4.4 Streaming and Suspense

- Use streaming when independent sections can render without blocking the entire page.
- Use Suspense boundaries around genuinely asynchronous regions.
- Provide meaningful skeletons or loading placeholders.
- Keep fallback layouts dimensionally stable to avoid CLS.
- Avoid creating excessive Suspense boundaries that complicate rendering without measurable benefit.
- Ensure failures in one independently rendered section do not unnecessarily prevent unrelated sections from appearing.

### 4.5 Hydration Efficiency

- Minimize the amount of HTML requiring hydration.
- Keep server-rendered content outside interactive client boundaries.
- Avoid unnecessary effects that run immediately after hydration.
- Do not duplicate server-rendered work in client effects.
- Prevent hydration mismatches caused by browser-only values, timestamps, random values, or inconsistent locale formatting.
- Use client-only rendering only when the feature genuinely depends on browser state.

---

## 5. React Component Performance

### 5.1 Component Design

- Keep components small, cohesive, and focused on one responsibility.
- Separate data orchestration from presentation where it improves reuse and testability.
- Avoid large components that combine data fetching, complex state, rendering, and unrelated interactions.
- Prefer composition over deeply nested conditional rendering.
- Avoid creating unnecessary wrapper components that add complexity without clear value.
- Follow the project's general guideline of keeping production source files at or below 200 lines unless a documented architectural justification exists.

### 5.2 Avoid Unnecessary Re-Renders

- Keep state close to the components that consume it.
- Avoid placing frequently changing state in high-level providers.
- Avoid broad context updates that re-render large component trees.
- Split components when unrelated state changes cause unnecessary rendering.
- Use stable props where practical.
- Avoid updating state when the new value is semantically unchanged.
- Do not use memoization as a replacement for sound state ownership.

### 5.3 Memoization

Use `React.memo`, `useMemo`, and `useCallback` only when they provide a measurable or well-understood benefit.

Appropriate use cases include:

- Expensive derived calculations.
- Large lists with stable item props.
- Components that re-render frequently due to unrelated parent updates.
- Stable callback references required by memoized children or subscription APIs.

Avoid:

- Wrapping every component in `React.memo`.
- Memoizing trivial calculations.
- Creating `useCallback` for every event handler.
- Using `useMemo` to hide incorrect state design.
- Relying on memoization for correctness.
- Maintaining complex dependency arrays without a clear need.

Memoization must not introduce stale values or incorrect behavior.

### 5.4 Effects

- Use effects to synchronize with external systems, not to derive ordinary render state.
- Prefer direct computation for inexpensive derived values.
- Avoid effects that update state based on other state when the value can be derived during rendering.
- Define effect dependencies correctly.
- Clean up event listeners, subscriptions, observers, and timers.
- Prevent duplicate network requests caused by unnecessary effects.
- Handle cancellation or stale results for asynchronous operations.
- Avoid effect chains that trigger multiple avoidable renders.

### 5.5 Lists and Reconciliation

- Use stable, unique keys derived from persistent identity.
- Never use array indexes as keys for reorderable, insertable, or removable lists.
- Avoid recreating large collections unnecessarily.
- Keep list item components isolated when their updates are independent.
- Use pagination or virtualization for large datasets.
- Avoid rendering hidden items when they can be loaded on demand.

---

## 6. Data Fetching and API Efficiency

### 6.1 General Requirements

- Fetch only the data needed by the current view.
- Avoid redundant requests for data already available in the current request or query cache.
- Avoid sequential requests when independent requests can safely run in parallel.
- Prefer server-side data retrieval for server-rendered content.
- Use explicit loading, error, and empty states.
- Apply request timeouts and cancellation where supported.
- Avoid fetching large payloads when only a small projection is needed.

### 6.2 Request Waterfalls

Avoid unnecessary request waterfalls, particularly during initial page rendering.

Prefer:

- Parallel retrieval of independent data.
- Aggregated backend endpoints when they meaningfully reduce round trips.
- Server-side composition of data for server-rendered pages.
- Prefetching when navigation intent is clear and the cost is justified.

Do not create overly broad API endpoints solely to reduce the number of requests. API boundaries must remain cohesive and authorization must be enforced for every resource.

### 6.3 Payload Optimization

- Request only required fields.
- Paginate large collections.
- Avoid deeply nested response objects that the UI does not need.
- Use appropriate compression at the transport layer.
- Avoid sending duplicate representations of the same data.
- Avoid embedding large images or binary data in JSON responses.
- Use explicit response schemas and stable contracts.

### 6.4 Request Deduplication

- Reuse in-flight requests when the same resource is requested concurrently.
- Use TanStack Query deduplication for client-side server state where applicable.
- Avoid implementing a separate request cache if the existing query layer already provides the needed behavior.
- Ensure deduplication respects tenant and identity boundaries.

### 6.5 Cancellation and Stale Requests

- Cancel requests when a user navigates away or changes the active query, where supported.
- Prevent stale search responses from overwriting newer results.
- Avoid updating unmounted components with obsolete asynchronous results.
- Use `AbortController` or the relevant library cancellation mechanism.
- Ensure cancellation does not accidentally interrupt required transactional operations.

---

## 7. TanStack Query Performance

TanStack Query is the primary client-side server-state management solution.

### 7.1 Query Design

- Use deterministic query keys.
- Include all variables that affect the returned data.
- Include tenant and relevant identity scope where data differs by tenant or user.
- Keep query functions focused on one resource or cohesive data requirement.
- Use shared query-key factories to prevent inconsistent cache identities.
- Avoid duplicating the same server state in Zustand or component state.

### 7.2 Freshness and Cache Lifecycle

- Define `staleTime` according to the data's expected freshness.
- Define `gcTime` according to memory constraints and navigation patterns.
- Avoid defaulting all data to permanently fresh or immediately stale.
- Use invalidation after successful mutations when necessary.
- Use direct cache updates only when the updated result can be reconstructed reliably.
- Clear or isolate cached data when users log out or switch tenants.

### 7.3 Query Optimization

- Use `select` to derive only the data required by a component when appropriate.
- Avoid broad query subscriptions when only a small part of the result is needed.
- Use `enabled` to prevent requests until required inputs are available.
- Prefetch likely next-page or next-route data only when the expected benefit exceeds the cost.
- Use infinite queries for suitable continuously paginated interfaces.
- Avoid aggressive polling when event-driven updates are available.

### 7.4 Mutations

- Use mutation callbacks for targeted cache updates and invalidation.
- Use optimistic updates only when rollback and reconciliation are well-defined.
- Prevent duplicate submissions for sensitive actions.
- Use idempotency keys for operations where duplicate requests could cause side effects, supported by the backend contract.
- Avoid invalidating unrelated queries.
- Keep payment, booking, and ordering outcomes authoritative on the backend.

### 7.5 Realtime Synchronization

- Use WebSockets or another approved event mechanism for features that require live updates.
- Update or invalidate only relevant queries after receiving an event.
- Batch frequent events when appropriate.
- Avoid refetching every query for every incoming event.
- Reconcile events with current server state and handle reconnects safely.
- Stop subscriptions when their owning feature is no longer active.

---

## 8. Zustand and Local State Performance

Zustand is intended for client-side UI and interaction state, not as a second server-state cache.

### 8.1 Store Design

- Create focused stores with clear ownership.
- Avoid a single global store for unrelated features.
- Keep transient component state local when it does not need to be shared.
- Do not mirror complete API responses into Zustand when TanStack Query already owns them.
- Keep store updates immutable and scoped.

### 8.2 Selectors

- Subscribe components to only the state they require.
- Use stable selectors where necessary.
- Avoid selectors that allocate large new objects on every render without an equality strategy.
- Avoid subscribing an entire page to a store when only a small component needs the value.

### 8.3 Persistence

- Persist only state that must survive reloads.
- Avoid persisting sensitive information or authorization decisions.
- Version persisted data where schema changes are possible.
- Validate restored state before use.
- Clear user- or tenant-scoped persisted state on logout or tenant change.
- Keep storage operations small and avoid synchronous heavy processing during startup.

---

## 9. Bundle Size and Code Splitting

### 9.1 Bundle Management

- Import only the required functionality from dependencies.
- Avoid unnecessarily large utility libraries for simple operations.
- Review dependency size before introducing major packages.
- Remove unused dependencies and exports.
- Avoid importing server-only modules into client bundles.
- Use bundle analysis to identify unexpected dependency growth.

### 9.2 Dynamic Imports

Use dynamic imports for:

- Large charts and data visualizations.
- Rich text editors.
- Complex maps.
- Infrequently used admin tools.
- Heavy file-processing interfaces.
- Large modal workflows that are not needed on initial render.

Dynamic imports must:

- Provide a useful loading state.
- Handle loading failures.
- Avoid delaying critical content.
- Be applied where splitting actually reduces initial work.

### 9.3 Route-Based Splitting

- Keep route-specific dependencies isolated.
- Avoid importing all admin features into public routes.
- Avoid placing large feature registries in a shared client bundle.
- Split rarely used features from frequently used entry points.
- Inspect shared chunks to prevent unrelated routes from inheriting unnecessary code.

### 9.4 Third-Party Dependencies

- Review the performance and maintenance impact of every significant dependency.
- Prefer native browser capabilities when they provide a simpler and sufficient solution.
- Load non-essential third-party scripts asynchronously or after the critical rendering path.
- Avoid unnecessary analytics, tracking, and UI libraries.
- Ensure third-party failures do not block core KAMPYN workflows.

---

## 10. Asset Optimization

### 10.1 Images

- Use Next.js image optimization where supported by the deployment environment.
- Specify image dimensions or aspect ratios to prevent layout shifts.
- Use responsive image sizes appropriate to the rendered layout.
- Prefer modern image formats when supported.
- Use lazy loading for below-the-fold images.
- Prioritize images that are genuinely part of the initial viewport.
- Avoid loading full-resolution originals for small thumbnails.
- Use appropriately sized placeholders where beneficial.
- Do not lazy-load critical above-the-fold imagery without a reason.

### 10.2 Fonts

- Prefer optimized local font delivery when practical.
- Use only the required font families, weights, and subsets.
- Avoid loading multiple redundant font variants.
- Use suitable font-display behavior to avoid invisible text.
- Preload only fonts required for initial rendering.
- Ensure fallback fonts do not cause significant layout shifts.

### 10.3 Icons and SVG

- Prefer lightweight SVG or icon components over large icon bundles.
- Import only the required icons.
- Avoid embedding large, redundant SVG definitions.
- Keep decorative graphics from adding unnecessary rendering complexity.
- Provide accessible names for meaningful icons and hide purely decorative icons from assistive technologies.

### 10.4 Video and Other Media

- Avoid autoplaying large media by default.
- Use appropriate poster images and metadata.
- Load media only when it is relevant to the user.
- Prefer streaming or range-based delivery for large media.
- Avoid placing large binary assets directly in application bundles.

---

## 11. CSS and Layout Performance

### 11.1 Styling

- Prefer simple, maintainable CSS.
- Avoid unnecessarily complex selectors.
- Remove unused styles and redundant declarations.
- Avoid frequent runtime style recalculations caused by excessive DOM changes.
- Use CSS variables for shared design tokens where appropriate.
- Avoid generating large amounts of dynamic CSS for simple visual variations.

### 11.2 Layout Stability

- Reserve space for images, embeds, and asynchronous content.
- Avoid inserting content above existing content without reserving layout space.
- Ensure banners, notifications, and loading placeholders do not cause unexpected layout shifts.
- Use stable skeleton dimensions.
- Avoid changing element dimensions unnecessarily during interaction.

### 11.3 Animation

- Prefer `transform` and `opacity` for animations where appropriate.
- Avoid animating layout-heavy properties such as width, height, top, and left when alternatives are suitable.
- Avoid excessive simultaneous animations.
- Respect `prefers-reduced-motion`.
- Keep transitions short and purposeful.
- Ensure animations do not delay critical interactions or obscure important feedback.

### 11.4 DOM Complexity

- Avoid unnecessarily deep DOM trees.
- Do not render large hidden component trees when they are not needed.
- Use virtualization for large lists where appropriate.
- Keep repeated structures simple.
- Avoid excessive wrappers that add no semantic or layout value.

---

## 12. Large Lists, Tables, and Data-Heavy UI

KAMPYN includes menus, orders, inventory, bookings, complaints, users, and analytics. These interfaces may contain large datasets.

### 12.1 Pagination

- Prefer server-side pagination for large datasets.
- Use cursor-based pagination when it suits the ordering and consistency requirements.
- Keep page sizes bounded.
- Preserve filters and sorting across navigation.
- Avoid loading the entire dataset into the browser for convenience.

### 12.2 Virtualization

Use virtualization when rendering a large collection would create significant DOM, layout, or memory overhead.

Suitable use cases include:

- Large order histories.
- Inventory tables.
- User management lists.
- Community message histories.
- Long activity feeds.
- Large administrative datasets.

Virtualization must preserve:

- Keyboard navigation.
- Screen-reader usability.
- Stable item identity.
- Correct scrolling behavior.
- Suitable loading and empty states.

Do not virtualize small lists where the added complexity provides no meaningful benefit.

### 12.3 Filtering and Sorting

- Perform large or authorization-sensitive filtering on the backend.
- Avoid repeated sorting or filtering during every render.
- Use memoization for expensive client-side derived data when profiling justifies it.
- Debounce text input where it reduces unnecessary requests or computation.
- Keep filter state explicit and deterministic.
- Ensure pagination is reset or reconciled when filters change.

### 12.4 Tables

- Request only visible or necessary columns.
- Avoid rendering complex cell components unnecessarily.
- Keep row keys stable.
- Use server-side sorting for large datasets.
- Avoid recalculating all cell values when only one row changes.
- Use column virtualization only when table width or rendering cost justifies it.

---

## 13. Search and Discovery Performance

Search is a core KAMPYN capability and must remain responsive as vendors, items, and campus content grow.

### 13.1 Search Interaction

- Debounce user input when requests are triggered by typing.
- Cancel obsolete in-flight requests.
- Prevent stale responses from replacing newer results.
- Avoid searching on every keystroke when the query has not meaningfully changed.
- Use clear minimum query requirements where appropriate.
- Provide immediate interaction feedback without blocking the input.

### 13.2 Search Architecture

- Use OpenSearch for search workloads where its capabilities are justified.
- Keep authoritative domain data in the appropriate source-of-truth datastore.
- Avoid retrieving an entire catalog to perform browser-side search.
- Use server-side filtering, ranking, and pagination for large search collections.
- Treat search indexes as derived data that may lag behind authoritative data.
- Ensure search results are scoped by tenant and access permissions.

### 13.3 Search Result Rendering

- Render a bounded number of results per page or batch.
- Use appropriately sized thumbnails.
- Avoid expensive client-side ranking on large result sets.
- Cache search results only with correct query, filter, tenant, and user scoping.
- Provide loading, no-results, error, and retry states.

---

## 14. Forms and User Interactions

### 14.1 Form Performance

- Keep form state scoped to the form or relevant fields.
- Avoid rerendering unrelated application sections when a field changes.
- Validate locally when possible, and validate authoritatively on the backend.
- Use debounced asynchronous validation only when needed.
- Avoid repeated validation of unchanged fields.
- Keep large forms divided into cohesive sections.

### 14.2 Interaction Responsiveness

- Provide immediate visual feedback for user actions.
- Avoid blocking the main thread with expensive calculations.
- Use asynchronous processing or a Web Worker for appropriate CPU-intensive browser tasks.
- Disable or guard duplicate submissions where appropriate.
- Ensure loading indicators do not prevent necessary cancellation or navigation.
- Avoid artificial delays in interaction flows.

### 14.3 Optimistic UI

Optimistic UI may be used for low-risk operations when rollback behavior is reliable.

Suitable examples may include:

- Toggling a preference.
- Updating a local filter.
- Marking a non-critical notification as read.

Use authoritative confirmation for:

- Payments.
- Orders.
- Bookings.
- Inventory adjustments.
- Financial records.
- Permission changes.

The backend remains the source of truth for all domain-critical operations.

---

## 15. Real-Time Features

Community, chat, notifications, and operational dashboards may require real-time updates.

### 15.1 Connection Management

- Avoid unnecessary persistent connections.
- Reuse approved connection infrastructure where suitable.
- Establish clear connection ownership and cleanup behavior.
- Reconnect with bounded exponential backoff and jitter.
- Avoid reconnect storms after network or service outages.
- Respect visibility and lifecycle states where appropriate.

### 15.2 Event Processing

- Process only events relevant to the active tenant, user, and feature.
- Avoid triggering full-page or global-state updates for individual events.
- Batch high-frequency updates when appropriate.
- Avoid expensive synchronous work in message handlers.
- Handle duplicate, delayed, or out-of-order events according to the event contract.

### 15.3 Chat and Community

- Load message history in bounded pages.
- Virtualize long conversations when necessary.
- Avoid repeatedly rendering the entire message history when one message changes.
- Keep typing indicators and presence updates lightweight.
- Throttle or batch high-frequency ephemeral updates.
- Avoid unnecessary serialization and deserialization.
- Clean up subscriptions and listeners when leaving a conversation.

---

## 16. Caching and Client-Side Storage

### 16.1 Cache Ownership

Use each caching layer for its intended responsibility.

| Layer              | Responsibility                                           |
| ------------------ | -------------------------------------------------------- |
| Browser HTTP cache | Cacheable static and HTTP resources                      |
| CDN                | Public assets and eligible cacheable content             |
| Next.js cache      | Eligible server-rendered and fetched data                |
| TanStack Query     | Client-side server-state caching                         |
| Zustand            | Client-side UI and interaction state                     |
| Redis              | Backend cache, coordination, and selected ephemeral data |
| OpenSearch         | Search indexes and discovery workloads                   |

Do not introduce duplicate caches without a documented consistency and invalidation strategy.

### 16.2 Cache Correctness

- Define freshness requirements for each resource.
- Use explicit cache keys and invalidation policies.
- Include tenant and identity scope where required.
- Prevent private or tenant-specific data from entering shared caches.
- Clear relevant client caches on logout or tenant switching.
- Ensure stale data cannot authorize sensitive actions.
- Treat cached values as data, never as proof of permission.

### 16.3 Browser Storage

- Store only data that must persist in browser storage.
- Avoid storing secrets, credentials, or sensitive personal data in local storage.
- Validate and version persisted data.
- Handle storage unavailability and malformed values.
- Keep startup hydration from storage lightweight.
- Clear scoped persisted state when identity or tenant context changes.

---

## 17. Main-Thread and Memory Efficiency

### 17.1 Main-Thread Work

- Keep synchronous browser tasks short.
- Avoid long-running loops during rendering or event handling.
- Move suitable CPU-intensive tasks to Web Workers.
- Split large operations into bounded chunks when appropriate.
- Avoid unnecessary JSON parsing and serialization of large objects.
- Avoid repeatedly transforming large datasets in component render functions.

### 17.2 Web Workers

Consider Web Workers for:

- Large client-side data transformations.
- Expensive file parsing.
- Complex calculations that must run in the browser.
- Operations that would otherwise block user interaction.

Workers must:

- Have bounded input and output sizes.
- Avoid duplicating large data unnecessarily.
- Have clear cancellation and lifecycle behavior.
- Be used only when moving work off the main thread provides a measurable benefit.

### 17.3 Memory Management

- Clean up subscriptions, observers, listeners, and retained references.
- Avoid retaining large datasets longer than necessary.
- Avoid unbounded client-side caches.
- Release temporary data after processing.
- Avoid storing duplicate representations of large datasets.
- Test long-lived pages for memory growth.
- Inspect detached DOM nodes and retained closures when investigating leaks.

### 17.4 Resource Limits

- Bound pagination and result sizes.
- Bound concurrent requests where appropriate.
- Limit the number of active subscriptions.
- Avoid unbounded queues and event buffers.
- Use backpressure or controlled batching for high-frequency data.
- Ensure the UI remains responsive when the backend is slow.

---

## 18. Responsive and Mobile Performance

KAMPYN must remain usable on devices with different capabilities and network conditions.

- Design mobile experiences around constrained screen sizes and touch interaction.
- Avoid loading desktop-only dependencies on mobile routes when they are not needed.
- Use responsive image sizes and layouts.
- Minimize blocking resources.
- Avoid unnecessarily large initial payloads.
- Keep touch interactions responsive and visually stable.
- Avoid expensive animations on lower-powered devices.
- Support reduced-motion preferences.
- Ensure loading and error states remain usable on narrow viewports.
- Test real-device performance where feasible.

Responsive design is not only about layout. It must also account for CPU, memory, bandwidth, and input capabilities.

---

## 19. Accessibility and Performance

Performance optimizations must preserve accessibility.

- Use semantic HTML and accessible component patterns.
- Preserve keyboard navigation when virtualizing or dynamically loading content.
- Announce meaningful asynchronous updates to assistive technologies.
- Avoid removing content from the accessibility tree solely to reduce rendering cost.
- Ensure loading states are understandable without relying only on animation.
- Respect reduced-motion preferences.
- Avoid focus loss during list updates, navigation, and optimistic mutations.
- Ensure responsive interfaces remain usable with zoom and text scaling.

Accessibility and performance should be designed together rather than traded off unnecessarily.

---

## 20. Multi-Tenant Performance

KAMPYN serves multiple universities and may support dedicated or self-hosted deployments.

### 20.1 Tenant Isolation

- Include tenant scope in client-side cache keys wherever data is tenant-specific.
- Clear or isolate tenant-scoped state during tenant changes.
- Avoid reusing data across tenants.
- Do not rely on frontend filtering for authorization or tenant isolation.
- Keep backend authorization authoritative.

### 20.2 Tenant-Specific Configuration

- Avoid repeated configuration requests when configuration can be safely cached.
- Cache tenant configuration with explicit versioning and invalidation.
- Keep feature flags and configuration access efficient.
- Avoid unnecessarily loading configuration for unrelated features.
- Ensure tenant-specific branding does not create excessive runtime style or asset overhead.

### 20.3 Noisy-Neighbor Considerations

- Avoid unbounded client requests from large tenants or high-traffic pages.
- Use pagination, batching, and request cancellation.
- Avoid expensive background refreshes that continue when features are inactive.
- Coordinate client polling and realtime subscriptions with backend capacity limits.
- Measure performance across tenants with different dataset sizes.

### 20.4 Self-Hosted Deployments

- Avoid hard dependencies on one hosting provider's performance features.
- Document requirements for image optimization, caching, and asset delivery.
- Provide sensible defaults for installations without a CDN.
- Ensure runtime configuration does not force unnecessary client-side rendering.
- Test deployment-specific behavior, including asset paths and caching headers.

---

## 21. Security and Performance

Performance optimizations must not weaken security.

- Never bypass authorization to reduce request latency.
- Do not expose secrets or privileged data in client bundles.
- Do not treat cached data as proof of authorization.
- Avoid overly broad responses that expose unnecessary user or tenant data.
- Validate untrusted data before expensive processing.
- Bound input lengths, result counts, and computational workloads.
- Avoid client-side caching of sensitive data unless the security model explicitly permits it.
- Apply rate limits and request constraints at the backend.
- Avoid speculative prefetching of sensitive resources without a valid authorization context.

Security-sensitive operations must remain correct even when caches are stale, requests are retried, or users navigate rapidly.

---

## 22. Error Handling and Degraded Performance

Frontend applications must remain usable when services are slow, unavailable, or return partial data.

- Provide loading, error, empty, and retry states.
- Use timeouts and cancellation where appropriate.
- Avoid infinite retry loops.
- Use bounded retry policies with backoff for transient failures.
- Avoid blocking unrelated interface sections when one request fails.
- Preserve user input where safe during recoverable failures.
- Avoid retrying non-idempotent operations without an appropriate idempotency contract.
- Use stale data only when its use is safe and clearly understood.
- Ensure critical workflows provide clear feedback when an operation's outcome is uncertain.

The UI must not falsely indicate that an order, payment, booking, or inventory change succeeded when the backend has not confirmed it.

---

## 23. Monitoring and Observability

### 23.1 Real User Monitoring

Track real-user performance for important routes and workflows.

Measure:

- LCP.
- INP.
- CLS.
- FCP and TTFB where useful.
- Route navigation timing.
- JavaScript errors.
- API latency.
- Failed requests.
- Hydration and rendering issues.
- Device class, browser, and network context where privacy-compliant.

Use aggregated measurements and avoid collecting unnecessary personal data.

### 23.2 Route-Level Monitoring

Track critical workflows separately, including:

- Authentication.
- Food discovery.
- Cart and checkout.
- Order tracking.
- Hostel and guest-house bookings.
- Facility scheduling.
- Search and discovery.
- Complaints and support.
- Community and chat.
- Administrative dashboards.

A site-wide average must not hide a poorly performing critical route.

### 23.3 Performance Correlation

Where feasible, correlate frontend measurements with:

- API endpoint latency.
- Backend errors.
- Cache hit rates.
- Database query latency.
- Network failures.
- Deployment versions.
- Tenant-specific workload patterns.

Use request or trace identifiers where available, without exposing sensitive data.

### 23.4 Alerting

Create alerts for:

- Sustained deterioration in Core Web Vitals.
- Large increases in JavaScript errors.
- Significant API latency regressions.
- Critical workflow failures.
- Unexpected bundle-size growth.
- Abnormal memory growth in long-lived interfaces.

Alerts must have actionable thresholds and avoid excessive noise.

---

## 24. Performance Testing

### 24.1 Testing Strategy

Performance testing must be part of the development and release lifecycle.

Use:

- Lighthouse for lab diagnostics.
- Chrome DevTools for runtime profiling.
- React Profiler for component render analysis.
- Browser performance traces for main-thread and rendering analysis.
- Bundle analyzers for dependency and chunk inspection.
- Real User Monitoring for production behavior.
- Load and integration testing for backend-dependent frontend workflows.

Choose tools according to the bottleneck being investigated.

### 24.2 Representative Workloads

Test using realistic:

- Dataset sizes.
- Component counts.
- API response payloads.
- Network latency.
- Device capabilities.
- User interaction patterns.
- Concurrent requests.
- Realtime event rates.
- Session duration.

Do not validate performance only with tiny mock datasets.

### 24.3 Automated Performance Checks

Where practical, include automated checks for:

- Route bundle size.
- Unexpected dependency growth.
- Core Web Vitals regressions in controlled test environments.
- Rendering performance of critical components.
- Large-list behavior.
- Repeated API requests.
- Memory leaks in long-lived interfaces.

Automated lab results should be interpreted carefully because test environments can vary.

### 24.4 Regression Testing

- Record a baseline before significant performance changes.
- Compare before-and-after measurements under similar conditions.
- Investigate material regressions before merging.
- Document intentional budget increases.
- Re-test after dependency upgrades and major framework changes.
- Verify that performance improvements do not introduce accessibility or correctness regressions.

---

## 25. Optimization Workflow

All significant frontend performance work should follow a consistent workflow.

1. **Identify:** Define the affected route, interaction, or resource.
2. **Measure:** Capture a baseline using a suitable tool and representative workload.
3. **Diagnose:** Determine whether the bottleneck is network, server, JavaScript, rendering, memory, or layout.
4. **Prioritize:** Estimate user impact, frequency, severity, and implementation risk.
5. **Design:** Choose the simplest solution that addresses the identified bottleneck.
6. **Implement:** Make a focused change without unnecessary architectural expansion.
7. **Validate:** Re-measure using the same conditions and verify functional behavior.
8. **Review:** Check accessibility, security, tenant isolation, and maintainability.
9. **Document:** Record meaningful trade-offs, updated budgets, or architectural decisions.
10. **Monitor:** Confirm that the improvement is sustained in real-world usage.

Do not accept an optimization solely because it appears more sophisticated or reduces a theoretical operation count.

---

## 26. Common Anti-Patterns

The following practices are prohibited unless an explicit, documented justification exists:

- Using Client Components for entire routes without a requirement.
- Fetching all data on the client when server rendering is appropriate.
- Loading entire datasets into browser memory.
- Rendering thousands of DOM elements without considering pagination or virtualization.
- Duplicating TanStack Query data in Zustand.
- Adding memoization to every component by default.
- Triggering network requests from effects without proper lifecycle handling.
- Creating request waterfalls for independent resources.
- Refetching all server state after every mutation.
- Polling aggressively when event-driven updates are suitable.
- Importing large dependencies into the initial route unnecessarily.
- Loading large visualizations before they are needed.
- Using unbounded caches or event buffers.
- Performing heavy computation in render functions.
- Recomputing large derived datasets on every render.
- Using array indexes as keys for dynamic lists.
- Ignoring stale requests during search and navigation.
- Persisting sensitive information in browser storage.
- Allowing tenant-specific state to leak across tenant transitions.
- Using optimistic UI for sensitive operations without reliable rollback.
- Ignoring image dimensions and causing layout shifts.
- Applying performance optimizations without measuring their impact.
- Sacrificing accessibility to reduce rendering work.
- Treating lab metrics as proof of production performance.
- Ignoring performance regressions after dependency upgrades.

---

## 27. Code Review Requirements

Reviewers must evaluate frontend performance changes for:

- Appropriate use of Server and Client Components.
- Correct state ownership and subscription scope.
- Avoidance of redundant renders and state updates.
- Efficient data fetching and request lifecycle management.
- Correct TanStack Query keying, freshness, and invalidation.
- Appropriate Zustand selectors and store boundaries.
- Bundle size and dependency impact.
- Image, font, and asset optimization.
- Efficient list, table, and search rendering.
- Bounded memory, request concurrency, and event processing.
- Responsive behavior on constrained devices.
- Tenant-aware client-side caching.
- Accessibility and keyboard usability.
- Performance tests or profiling evidence where warranted.
- Documented justification for material performance trade-offs.

Performance-related changes must not introduce duplicated abstractions or unnecessary complexity.

---

## 28. Implementation Checklist

### Architecture

- [ ] Server Components are used by default.
- [ ] Client Components are scoped to actual interactive requirements.
- [ ] Rendering strategy matches data freshness and personalization needs.
- [ ] Heavy or infrequently used features are isolated from critical bundles.
- [ ] Server-only modules cannot leak into client bundles.

### Rendering

- [ ] State is owned by the smallest appropriate component or store.
- [ ] Unnecessary re-renders have been avoided.
- [ ] Memoization is justified by workload or measurement.
- [ ] Effects synchronize with external systems and clean up correctly.
- [ ] Large lists use bounded rendering strategies.
- [ ] Hydration is limited to required interactive regions.

### Data Fetching

- [ ] Only required data is requested.
- [ ] Independent requests are parallelized where appropriate.
- [ ] Request waterfalls have been avoided where possible.
- [ ] Stale requests are cancelled or ignored safely.
- [ ] Payloads and pagination are bounded.
- [ ] Tenant and identity scopes are included in relevant cache keys.

### State and Caching

- [ ] TanStack Query owns client-side server state.
- [ ] Zustand is limited to client-side interaction and UI state.
- [ ] Cache freshness and invalidation are explicitly defined.
- [ ] Duplicate server-state caches have been avoided.
- [ ] Sensitive and tenant-scoped data is handled safely.
- [ ] Logout and tenant switching clear or isolate relevant state.

### Assets and Bundles

- [ ] Images have appropriate sizes and dimensions.
- [ ] Fonts and icons are optimized.
- [ ] Large dependencies are justified.
- [ ] Route-specific code is split appropriately.
- [ ] Initial bundle budgets are respected or justified.
- [ ] Non-critical third-party scripts do not block core rendering.

### Responsiveness and Reliability

- [ ] The main thread is not blocked by unnecessary work.
- [ ] Long-running tasks are bounded or moved off-thread where appropriate.
- [ ] Memory and subscriptions are cleaned up.
- [ ] Loading, error, and empty states are implemented.
- [ ] Retry and cancellation behavior is appropriate.
- [ ] Critical operations are not falsely reported as successful.

### Validation

- [ ] Critical routes have defined performance expectations.
- [ ] Representative workloads have been tested.
- [ ] Performance regressions have been evaluated.
- [ ] Accessibility and responsive behavior remain correct.
- [ ] Relevant monitoring is available for production workflows.

---

## 29. Definition of Done

A frontend feature is considered performance-ready when:

- Its rendering architecture is appropriate for its interactivity and data requirements.
- It avoids unnecessary client-side JavaScript, requests, renders, and computations.
- It uses bounded data retrieval and rendering strategies.
- Its state and caching behavior is correct, including tenant and identity isolation.
- Its assets and dependencies are reasonably optimized.
- It remains responsive on representative devices and network conditions.
- It handles slow services, failures, and repeated interactions safely.
- It meets the applicable route-level performance budget or has a documented, approved exception.
- Relevant performance testing or profiling has been completed.
- No known critical performance regression remains unresolved.
- Accessibility, security, and functional correctness are preserved.
- Monitoring is in place for critical production workflows where appropriate.

---

## 30. Final Engineering Standard

Frontend performance in KAMPYN must be achieved through deliberate architecture, efficient rendering, controlled data flow, bounded resource consumption, and continuous measurement.

Every optimization must serve a measurable user or system need. Simplicity, correctness, security, accessibility, and maintainability remain mandatory.

**The standard is not to write the least code or achieve the lowest theoretical complexity. The standard is to deliver the required experience with the least unnecessary work, predictable resource usage, and measurable performance under realistic conditions.**

# UI Performance Standards

## 1. Purpose

This document defines the performance standards for KAMPYN's frontend user interface, ensuring fast, responsive, efficient, and accessible experiences across devices, network conditions, and university deployments.

KAMPYN serves students, faculty, university administrators, vendors, and staff across workflows such as food ordering, hostel services, bookings, inventory, community interactions, and notifications. UI performance must remain predictable as the platform scales in users, data volume, and functionality.

Performance is a product requirement, not a post-release optimization.

### Core objectives

- Minimize initial page load and time to meaningful interaction.
- Keep navigation, scrolling, animations, and interactions responsive.
- Reduce unnecessary rendering, network requests, and JavaScript execution.
- Optimize images, fonts, and other frontend assets.
- Prevent memory leaks, excessive resource consumption, and performance degradation.
- Preserve usability on low-end devices and constrained networks.
- Establish measurable performance budgets and enforce them through testing.

---

## 2. Non-Negotiable Principles

1. **Measure before optimizing:** Performance changes must be guided by profiling, benchmarks, and reproducible evidence.
2. **Server-first rendering:** Use Next.js Server Components by default and introduce client-side JavaScript only when necessary.
3. **Avoid unnecessary work:** Prevent redundant renders, calculations, requests, and data transformations.
4. **Optimize for real devices:** Consider low-end mobile devices, limited memory, and slower networks.
5. **Keep interactions responsive:** User actions must not be blocked by expensive rendering or synchronous processing.
6. **Load progressively:** Deliver critical content first and defer nonessential resources.
7. **Optimize without compromising correctness:** Performance improvements must preserve functionality, data consistency, accessibility, and security.
8. **Avoid premature complexity:** Do not introduce caching, memoization, virtualization, or custom rendering systems without a demonstrated need.
9. **Use platform capabilities:** Prefer built-in browser, React, and Next.js optimizations over custom implementations.
10. **Prevent regressions:** Performance budgets and automated checks must be part of the development workflow.

---

## 3. Performance Budgets

Performance budgets are measurable limits that help prevent frontend regressions. They should be evaluated against production builds and realistic deployment conditions.

### 3.1 Core Web Vitals

KAMPYN should target the following Core Web Vitals at the 75th percentile of real-user page visits, measured separately for mobile and desktop wherever sufficient data is available.

| Metric | Target | Description |
|---|---:|---|
| LCP | ≤ 2.5 seconds | Largest Contentful Paint |
| INP | ≤ 200 ms | Interaction to Next Paint |
| CLS | ≤ 0.1 | Cumulative Layout Shift |

These are field-performance targets, not guarantees for every device or visit.

### 3.2 Additional performance targets

| Metric | Target |
|---|---:|
| Initial JavaScript (compressed) | ≤ 200 KB per primary route |
| Initial CSS (compressed) | ≤ 100 KB per primary route |
| Initial critical request count | Minimize; avoid redundant requests |
| Long tasks on the main thread | Avoid tasks exceeding 50 ms |
| Unnecessary client-side hydration | Zero by design |
| Avoidable layout shifts | Zero by design |
| Duplicate API requests | Zero where deduplication is supported |
| Unbounded client-side lists | Prohibited for large datasets |

These are engineering budgets and should be adjusted only through documented justification and measurement. Third-party scripts and framework overhead must be included when assessing actual user experience.

### 3.3 Budget enforcement

- Measure production builds, not only development mode.
- Track route-level JavaScript and CSS sizes.
- Identify and review significant bundle-size increases.
- Establish baseline measurements for critical user workflows.
- Record exceptions with the reason, impact, mitigation, and review date.
- Do not silently increase performance budgets to make a failing check pass.

Critical workflows should include:

- Student home and search.
- Food court listings and menus.
- Cart and checkout.
- Order tracking.
- Hostel and guest-house booking.
- Library availability.
- Shuttle booking.
- Vendor dashboards.
- University administration dashboards.
- Community and chat interfaces.

---

## 4. Rendering Strategy

### 4.1 Server Components by default

Next.js Server Components should be the default for route segments and components that do not require browser-only interactivity.

Use Server Components for:

- Static and mostly static page structure.
- Server-side data retrieval.
- Initial page content.
- SEO metadata and public marketing pages.
- Read-only information and summaries.
- Secure server-side operations that belong in the frontend's server boundary.

Benefits include:

- Reduced client-side JavaScript.
- Less hydration work.
- Earlier delivery of meaningful content.
- Smaller interactive bundles.
- Reduced dependence on client-side data fetching for initial rendering.

Do not convert a component to a Client Component solely because it displays dynamic data. Server Components can render data retrieved on the server.

### 4.2 Client Components

Use Client Components only when browser-side capabilities or interactivity are required, such as:

- Local interactive state.
- Event handlers.
- Browser APIs.
- Interactive forms and controls.
- Realtime client connections.
- Client-side state subscriptions.

Keep client boundaries narrow. Place interactive controls in small Client Components rather than marking an entire page or layout as client-rendered.

Avoid:

- Adding `"use client"` to shared layouts without a clear requirement.
- Moving server data fetching into the browser without a reason.
- Passing large, unnecessary objects across the Server-to-Client boundary.
- Hydrating static content that does not need client interaction.
- Duplicating server-rendered content with client-side rendering.

### 4.3 Static and dynamic rendering

Use static rendering for content that does not require request-specific data.

Use dynamic rendering when the response depends on request-time information, such as the authenticated user, tenant context, or other request-specific values.

Apply caching and revalidation deliberately. Never use shared caching for user-specific or tenant-specific data unless the cache is correctly isolated by identity and tenant scope.

### 4.4 Streaming and Suspense

Use streaming and React Suspense when a page has independent regions that can render progressively.

Examples include:

- Displaying the page shell before food court results are ready.
- Rendering a dashboard layout while individual summary cards load.
- Showing booking details while noncritical recommendations load.

Requirements:

- Keep primary content outside unnecessary loading boundaries.
- Use meaningful skeletons that resemble the final content dimensions.
- Avoid excessive nested Suspense boundaries.
- Ensure fallback states do not cause layout shifts.
- Handle errors independently where sections can fail separately.

---

## 5. React Rendering Efficiency

### 5.1 Minimize unnecessary renders

React components should render when their output needs to change, not because unrelated state has changed.

Standards:

- Keep state as close as possible to the components that use it.
- Separate unrelated interactive sections.
- Avoid placing frequently changing state in high-level providers.
- Avoid passing newly created objects and functions through large component trees when their identity affects rendering.
- Use stable keys for lists.
- Keep components focused on a single responsibility.

Do not add `React.memo`, `useMemo`, or `useCallback` everywhere. Apply memoization when profiling shows meaningful avoidable work or when stable references are required for a specific integration.

### 5.2 State isolation

State should be owned by the smallest suitable scope.

Use:

- Component state for local UI behavior.
- Zustand for shared client-side UI state that genuinely needs cross-component access.
- TanStack Query for remote server state.
- URL state for shareable or navigable state, such as filters, sorting, and pagination.
- Form state for temporary form interactions.

Avoid:

- Duplicating server state in Zustand without a defined synchronization requirement.
- Storing derived values that can be calculated cheaply.
- Putting every interaction into a global store.
- Updating global state for changes that affect only one component.

### 5.3 Derived state and expensive computations

- Prefer deriving values during rendering when calculations are inexpensive.
- Avoid redundant state and synchronization effects.
- Memoize expensive calculations only when profiling demonstrates a benefit.
- Keep expensive transformations outside frequently rendered components when appropriate.
- Avoid repeated filtering, sorting, and mapping of large collections on every render.
- Use server-side aggregation or pagination when datasets are large.

Do not optimize code by obscuring its logic or creating unnecessary abstraction layers.

### 5.4 Context performance

React Context should not become a high-frequency global state mechanism.

- Keep contexts narrowly scoped.
- Split contexts by responsibility and update frequency.
- Avoid storing rapidly changing values in broad providers.
- Prefer a suitable state store when independent subscriptions are needed.
- Prevent unrelated components from rerendering due to broad context updates.

Authentication and tenant context must be designed with special care to prevent stale or cross-tenant UI state.

---

## 6. JavaScript Bundle Optimization

### 6.1 Dependency management

Every dependency contributes to bundle size, execution time, maintenance, or supply-chain risk.

Before adding a dependency:

- Verify that native browser or framework functionality is insufficient.
- Evaluate package size and dependency tree.
- Check whether only a small part of the package is required.
- Confirm compatibility with the project's Next.js and TypeScript setup.
- Consider maintenance status and security implications.
- Avoid multiple libraries that solve the same problem.

Do not introduce a large utility package for a small, easily maintained operation.

### 6.2 Code splitting

Split code along meaningful route and feature boundaries.

- Let Next.js handle route-level code splitting.
- Lazy-load expensive, noncritical interactive features when appropriate.
- Load rarely used administrative tools only when needed.
- Defer large editors, charts, maps, and complex visualizations until they are relevant.
- Avoid splitting small components into many chunks without a measured benefit.

Use `next/dynamic` when a feature benefits from deferred loading or must be loaded on the client.

### 6.3 Tree shaking

- Use package entry points that support tree shaking.
- Prefer named imports where the package supports them.
- Avoid importing entire libraries for isolated functionality.
- Review package exports and build output.
- Remove unused dependencies and dead code.

Do not assume that a named import guarantees effective tree shaking. Verify the resulting production bundle.

### 6.4 Third-party scripts

Third-party scripts can affect page responsiveness, security, and privacy.

- Add scripts only when they provide clear product value.
- Use Next.js Script strategies appropriately.
- Defer noncritical scripts.
- Avoid blocking initial rendering with analytics, chat widgets, or tracking scripts.
- Review script execution cost and network behavior.
- Do not load duplicate analytics or monitoring scripts.
- Respect user consent and applicable privacy requirements.

Third-party scripts must never be allowed to bypass KAMPYN's security or tenant-isolation requirements.

---

## 7. Data Fetching and Network Performance

### 7.1 Data-fetching strategy

Use the appropriate fetching mechanism for the rendering context.

- Use Server Component fetching for server-rendered initial content where suitable.
- Use TanStack Query for client-side server state, background refetching, mutations, and interactive data.
- Use Route Handlers or Server Actions only where appropriate to the frontend architecture; the backend remains authoritative for business logic, persistence, and authorization.
- Avoid fetching the same data independently from multiple components when requests can be shared or deduplicated.

### 7.2 Request efficiency

- Avoid sequential requests when independent requests can run concurrently.
- Fetch only the fields required by the view.
- Prefer server-side aggregation for dashboards with many small dependent requests.
- Debounce high-frequency search inputs where appropriate.
- Cancel obsolete requests when users change routes, filters, or search terms.
- Avoid polling when events or a suitable refresh strategy can meet the requirement.
- Apply request timeouts and error handling consistently.

Do not combine unrelated API calls into a single endpoint solely to reduce request count. API boundaries should continue to reflect clear domain responsibilities.

### 7.3 TanStack Query

TanStack Query is the primary client-side server-state management system.

- Define stable, typed query keys.
- Include tenant and relevant identity scope in keys for tenant-specific or user-specific data.
- Configure `staleTime` and garbage collection according to the data's expected freshness and usage.
- Reuse queries for shared data rather than duplicating requests.
- Invalidate or update relevant queries after mutations.
- Use pagination or infinite queries for large collections.
- Avoid aggressive refetching that creates unnecessary server load.
- Handle loading, error, empty, and stale states explicitly.

Never treat cached client data as authorization evidence. Backend authorization must be enforced for every protected operation.

### 7.4 Search and filtering

Search is a core KAMPYN experience and must remain responsive as the catalog grows.

- Debounce requests for free-text search where appropriate.
- Avoid sending a request for every keystroke.
- Keep search, filters, and pagination consistent with URL state when useful.
- Use server-side filtering for large datasets.
- Return bounded result sets.
- Avoid downloading entire catalogs to filter them in the browser.
- Cancel or ignore stale responses to prevent outdated results from replacing newer ones.
- Use OpenSearch through the backend when appropriate; never expose direct privileged search infrastructure access to the browser.

### 7.5 Realtime interfaces

Realtime interfaces such as order tracking, chat, and availability updates should be designed to minimize unnecessary rendering and network activity.

- Subscribe only to the events relevant to the active view and authorized user.
- Unsubscribe when components unmount or scope changes.
- Batch or coalesce frequent UI updates where safe.
- Avoid updating large global state trees for every event.
- Reconcile incoming events with TanStack Query caches or the relevant state owner.
- Use reconnect and backoff strategies.
- Ensure stale events cannot overwrite newer authoritative data.
- Respect tenant boundaries in subscriptions and cache updates.

---

## 8. Images, Fonts, and Static Assets

### 8.1 Images

Use `next/image` for images where its optimization features are applicable.

- Specify image dimensions or aspect ratios to reserve layout space.
- Use responsive sizing appropriate to the rendered layout.
- Prefer modern formats when supported by the delivery pipeline.
- Serve appropriately sized images instead of large originals.
- Lazy-load below-the-fold images.
- Prioritize only the image that is genuinely critical to initial rendering.
- Provide meaningful alternative text where the image conveys information.
- Use decorative images without unnecessary accessibility noise.

For food listings, vendor profiles, and product catalogs, generate or serve multiple image sizes instead of transferring full-resolution uploads to every device.

Do not lazy-load the primary LCP image when it needs to appear immediately.

### 8.2 Fonts

- Prefer a limited font family and weight set.
- Use `next/font` for supported self-hosted or Google font workflows.
- Avoid loading unused font weights and styles.
- Prefer modern, compressed font formats where supported.
- Use `font-display` behavior that avoids invisible text for extended periods.
- Reserve fallback typography that minimizes layout shifts.

### 8.3 Icons and illustrations

- Prefer a consistent icon library with tree-shakeable imports.
- Avoid loading large icon collections when only a few icons are used.
- Prefer optimized SVGs for static illustrations.
- Avoid embedding oversized base64 assets in JavaScript bundles.
- Ensure icons do not cause unnecessary layout or rendering work.

### 8.4 Asset delivery

- Use content hashing and long-lived caching for immutable assets.
- Ensure correct cache-control headers.
- Compress text-based assets.
- Avoid shipping unused media.
- Use a CDN or suitable asset delivery strategy when deployment requirements justify it.
- Keep asset hosting configurable for university self-hosted deployments.

---

## 9. Lists, Tables, and Large Datasets

KAMPYN may display large menus, order histories, inventory tables, member lists, and administrative reports. Rendering unbounded collections in the browser is prohibited.

### 9.1 Pagination

Prefer server-side pagination when:

- Datasets are large.
- Users need direct page navigation.
- The backend can efficiently apply filtering and sorting.
- The interface is table-oriented or administrative.

Pagination requirements:

- Keep page size bounded.
- Use stable ordering.
- Avoid fetching records that are not displayed.
- Preserve relevant filters and sort state.
- Use cursor pagination where offset pagination becomes inefficient or unstable.

### 9.2 Virtualization

Use list or table virtualization when the number of rendered elements materially affects layout, memory, or interaction performance.

Virtualization should be considered for:

- Long order histories.
- Large inventory lists.
- Extensive user directories.
- Large message histories.
- High-volume administrative tables.

Requirements:

- Preserve keyboard navigation and screen-reader usability.
- Ensure focus remains stable as items mount and unmount.
- Maintain predictable scroll behavior.
- Support variable item heights only when necessary.
- Avoid virtualizing small lists where it adds complexity without measurable benefit.

### 9.3 Rendering large tables

- Render only the columns required by the current view.
- Avoid expensive per-cell formatting on every render.
- Use server-side sorting and filtering for large datasets.
- Keep row identity stable.
- Avoid deeply nested component trees for simple tabular data.
- Use memoization only where it reduces measured rendering cost.
- Ensure responsive alternatives exist for smaller screens.

### 9.4 Large client-side transformations

Do not perform expensive operations on large datasets during rendering.

Where appropriate:

- Aggregate or transform data on the backend.
- Use paginated or chunked data retrieval.
- Move CPU-intensive, independent tasks to Web Workers when browser-side computation is justified.
- Provide progress feedback for long-running operations.
- Keep the main thread available for user interaction.

Do not introduce Web Workers for trivial computations.

---

## 10. CSS, Layout, and Animation Performance

### 10.1 Efficient styling

KAMPYN uses a consistent styling system. Styling decisions should minimize unnecessary layout and paint work.

- Prefer CSS for presentation and layout.
- Avoid excessive runtime style calculations.
- Reuse design tokens and shared component styles.
- Avoid broad selectors that cause unintended styling dependencies.
- Avoid repeated DOM measurement and style mutation.
- Use CSS layout primitives such as Flexbox and Grid appropriately.
- Keep selectors and component styles maintainable.

### 10.2 Layout stability

Prevent unexpected movement during page rendering.

- Set image and media dimensions.
- Reserve space for asynchronous content.
- Avoid inserting banners or content above visible elements without reserved space.
- Use stable skeleton dimensions.
- Avoid late-loading fonts that cause major text reflow.
- Keep interactive controls from changing size unexpectedly.

### 10.3 Animation

Animations must be purposeful, short, and non-blocking.

- Prefer `transform` and `opacity` for animations.
- Avoid repeatedly animating layout-affecting properties such as `width`, `height`, `top`, or `left` where a transform can achieve the same effect.
- Avoid expensive blur, shadow, and filter effects on large moving surfaces.
- Avoid continuous animations that provide no meaningful information.
- Respect `prefers-reduced-motion`.
- Ensure animations do not delay critical interactions.

### 10.4 Scroll performance

- Avoid expensive work on every scroll event.
- Use CSS features such as `position: sticky` when suitable.
- Use passive event listeners for relevant native scroll and touch listeners.
- Avoid repeatedly reading layout properties and then writing styles in the same loop.
- Use `requestAnimationFrame` only when custom animation or visual synchronization genuinely requires it.
- Keep sticky headers, drawers, and overlays responsive on low-end devices.

---

## 11. Forms and Interactive Workflows

Forms are central to food ordering, booking, complaints, and administration. Their responsiveness and reliability must be maintained under both normal and slow network conditions.

### 11.1 Form performance

- Validate fields locally for immediate feedback where possible.
- Use Zod for runtime validation at appropriate boundaries.
- Avoid triggering expensive validation on every keystroke unless the interaction requires it.
- Debounce remote validation.
- Keep form state scoped to the form.
- Avoid rerendering unrelated fields when one field changes.
- Prevent duplicate submissions.
- Disable or otherwise guard repeated submission while an operation is pending.

Client-side validation improves usability but does not replace backend validation.

### 11.2 Mutations

- Show immediate and clear feedback for user actions.
- Use optimistic updates only when rollback and reconciliation behavior are defined.
- Keep pending states accessible.
- Prevent accidental duplicate orders, bookings, or payments through appropriate backend idempotency and client-side submission handling.
- Avoid blocking the whole interface when only one section is being updated.
- Reconcile UI state with authoritative server responses.

### 11.3 Checkout and booking

Checkout and booking interfaces must prioritize correctness over animation or visual effects.

- Keep cart and booking summaries responsive.
- Avoid unnecessary refetching during multi-step workflows.
- Clearly communicate pending, success, failure, and retry states.
- Preserve entered information safely when a recoverable failure occurs.
- Do not represent an order or booking as confirmed before backend confirmation.
- Never let an optimistic UI update be mistaken for a completed financial transaction.

---

## 12. Zustand and Client-Side State Performance

Zustand may be used for shared client-side UI state where it improves state ownership and avoids unnecessary prop drilling.

Standards:

- Keep stores focused and domain-specific.
- Subscribe components to the smallest state slice they need.
- Use selectors to avoid unrelated rerenders.
- Avoid storing large server datasets in Zustand.
- Avoid copying TanStack Query data into global stores without a clearly defined reason.
- Keep derived state out of the store when it can be computed cheaply.
- Avoid broad subscriptions to entire store objects.
- Clean up transient state when its lifecycle ends.

Examples of suitable Zustand state include:

- Open navigation drawer.
- Active modal.
- Temporary cart interface state, where not duplicated from authoritative cart data.
- User-selected interface preferences.
- Local multi-step interaction state shared across components.

Authentication and tenant transitions must reset or correctly scope relevant client-side state to avoid stale data appearing across accounts or universities.

---

## 13. Responsive Performance

KAMPYN must remain usable on mobile phones, tablets, laptops, and desktop displays.

- Use responsive layouts instead of maintaining separate duplicated page implementations.
- Avoid shipping desktop-only heavy components to mobile when they are not needed.
- Optimize image dimensions for the viewport.
- Keep touch interactions responsive.
- Avoid oversized navigation and dashboard bundles on mobile.
- Test at narrow widths and with device throttling.
- Ensure that responsive layout changes do not create excessive layout recalculation.
- Keep tables and complex dashboards usable on small screens through appropriate presentation patterns.

Responsive performance must be measured on realistic mobile hardware, not inferred from desktop performance.

---

## 14. Loading, Empty, and Error States

Performance includes how quickly the interface communicates that work is progressing and how well it recovers from failure.

Every asynchronous view must have appropriate states:

- Initial loading.
- Background refresh.
- Empty result.
- Partial data.
- Error.
- Retry.
- Pending mutation.
- Success.

Standards:

- Use skeletons for structured content where they improve perceived continuity.
- Use lightweight indicators for small actions.
- Avoid displaying a full-page loading screen for localized updates.
- Prevent layout shifts when loading states are replaced.
- Keep stale data visible during safe background refreshes where appropriate.
- Clearly distinguish stale or incomplete data when it matters to the user.
- Provide retry actions when recovery is possible.

Loading indicators must not conceal a failed or indefinitely stalled operation.

---

## 15. Caching and Offline Considerations

### 15.1 Browser and framework caching

Use caching only when the data's ownership, freshness, and invalidation requirements are understood.

- Cache immutable static assets aggressively.
- Configure Next.js caching and revalidation deliberately.
- Use TanStack Query caching for appropriate client-side server state.
- Avoid caching user-specific or tenant-specific data in shared scopes.
- Invalidate or reconcile affected data after mutations.
- Ensure logout, account switching, and tenant switching clear or isolate sensitive cached state.

### 15.2 Offline and poor-network behavior

For workflows where resilience is valuable:

- Show connection and request status clearly.
- Preserve recoverable form input where safe.
- Distinguish queued actions from confirmed actions.
- Avoid silently discarding user changes.
- Retry transient failures carefully with bounded backoff.
- Avoid automatically retrying non-idempotent financial or booking operations without backend-supported idempotency.

Offline capabilities must be explicitly designed and tested. Do not imply that an action succeeded when the server has not confirmed it.

---

## 16. Memory and Resource Management

Long-lived pages such as dashboards, community interfaces, and chat views must not accumulate unbounded resources.

- Clean up event listeners, timers, subscriptions, and observers.
- Abort obsolete network requests where supported.
- Avoid retaining large objects unnecessarily.
- Release resources associated with closed modals and unmounted features.
- Keep realtime subscriptions limited to active and authorized scopes.
- Avoid retaining obsolete page data indefinitely.
- Avoid repeated creation of large in-memory collections.
- Monitor memory behavior during long sessions.

Use `useEffect` cleanup for subscriptions and browser resources created by effects. Prefer framework-managed lifecycle behavior wherever possible.

Performance testing should include navigating between routes, switching tenants, opening and closing overlays, and leaving realtime views active for extended periods.

---

## 17. Accessibility and Performance

Performance optimizations must not degrade accessibility.

- Preserve semantic HTML and accessible names.
- Maintain keyboard navigation and visible focus.
- Ensure loading and status changes are announced appropriately.
- Avoid removing meaningful content to reduce rendering cost.
- Respect reduced-motion preferences.
- Keep virtualized content accessible.
- Ensure touch targets remain usable.
- Prevent loading transitions from unexpectedly moving keyboard focus.
- Provide accessible alternatives for complex visualizations.

Accessibility requirements remain mandatory even when an alternative implementation appears faster.

---

## 18. Security and Tenant Isolation

Frontend performance optimizations must never weaken security.

- Do not expose secrets or privileged API credentials in client bundles.
- Do not treat hidden UI elements as authorization.
- Ensure protected data is fetched through authorized backend interfaces.
- Scope client-side caches by tenant and relevant identity.
- Clear or isolate cached data during logout and tenant switching.
- Avoid placing sensitive data in persistent browser storage without explicit justification.
- Prevent stale responses from a previous tenant or user from updating the active interface.
- Ensure realtime subscriptions are authorized and cleaned up correctly.

Backend authorization and tenant isolation remain authoritative regardless of how the frontend caches or renders data.

---

## 19. Performance Measurement and Profiling

Optimization must be based on reproducible measurements.

### 19.1 Local profiling

Use appropriate tools such as:

- React DevTools Profiler.
- Chrome DevTools Performance panel.
- Chrome DevTools Network panel.
- Lighthouse.
- Next.js bundle analysis tools.
- Browser memory profiling.

Investigate:

- Unnecessary React renders.
- Long tasks and main-thread blocking.
- Large JavaScript chunks.
- Expensive layout and paint operations.
- Slow API requests.
- Repeated network requests.
- Excessive hydration.
- Memory growth and retained objects.

### 19.2 Field performance

Where privacy-compliant telemetry is available, monitor real-user metrics.

Track:

- LCP.
- INP.
- CLS.
- Route-level loading behavior.
- API latency.
- Client-side errors.
- Device and connection characteristics at an appropriately aggregated level.

Do not collect unnecessary personal data to measure performance. Telemetry must follow KAMPYN's privacy and security policies.

### 19.3 Reproducibility

Every significant optimization should record:

- The affected route or component.
- The observed performance issue.
- The device and test conditions.
- The baseline measurement.
- The change made.
- The result after the change.
- Any trade-offs or regressions.

Do not claim an optimization improved performance based only on subjective perception.

---

## 20. Automated Performance Testing

Performance should be part of continuous integration for critical routes and workflows.

### 20.1 Build checks

- Run production builds in CI.
- Detect unexpected bundle-size increases.
- Identify unused dependencies where practical.
- Validate static asset output.
- Check for client/server boundary mistakes and avoidable client bundles.

### 20.2 Browser tests

Use browser automation to validate:

- Critical route rendering.
- Navigation responsiveness.
- Form interaction behavior.
- Loading and error states.
- Search and filtering.
- Pagination and virtualized lists.
- Responsive layouts.
- Tenant switching and logout cleanup.

### 20.3 Performance regression checks

- Run Lighthouse or equivalent checks on representative production-like routes.
- Use consistent test conditions for comparisons.
- Track significant regressions against agreed budgets.
- Avoid relying on a single Lighthouse score as the sole performance measure.
- Use field metrics when sufficient real-user data is available.

Performance tests should be stable, actionable, and tied to user-facing outcomes. Flaky tests should be corrected rather than routinely ignored.

---

## 21. Anti-Patterns

The following practices are prohibited unless a documented, measured exception is approved.

- Marking entire route trees as Client Components without need.
- Fetching large datasets to filter or sort them entirely in the browser.
- Storing duplicated server state in multiple state-management systems.
- Applying memoization indiscriminately.
- Rendering unbounded lists or tables.
- Loading large libraries for small isolated functions.
- Loading noncritical third-party scripts synchronously.
- Triggering duplicate requests from multiple components.
- Performing expensive transformations during every render.
- Repeatedly measuring and mutating DOM layout in the same execution cycle.
- Creating global state for isolated local interactions.
- Using optimistic UI as a substitute for backend confirmation.
- Adding complex caching without a defined invalidation strategy.
- Retaining subscriptions after a component or tenant scope is no longer active.
- Using large animations that block interaction or cause layout instability.
- Treating development-mode performance as representative of production.
- Removing accessibility behavior to improve a benchmark.
- Increasing performance budgets without evidence or documented review.

---

## 22. Performance Review Checklist

### Rendering
- [ ] Server Components are used by default.
- [ ] Client Components are limited to necessary interactive boundaries.
- [ ] Static content is not unnecessarily hydrated.
- [ ] Suspense and streaming are used where they provide meaningful value.
- [ ] Component state is scoped appropriately.
- [ ] Unnecessary renders have been investigated.

### Bundles and assets
- [ ] Route-level bundle sizes are within agreed budgets.
- [ ] Dependencies are justified and reviewed.
- [ ] Noncritical code is deferred where appropriate.
- [ ] Images are responsive and appropriately sized.
- [ ] Fonts and icons are optimized.
- [ ] Third-party scripts are limited and appropriately loaded.

### Data and network
- [ ] Data fetching avoids unnecessary duplication.
- [ ] Requests are concurrent where dependencies allow.
- [ ] Search and filtering are appropriately debounced and bounded.
- [ ] TanStack Query keys and cache scopes are correct.
- [ ] Large datasets use pagination or virtualization where necessary.
- [ ] Realtime subscriptions are scoped and cleaned up.
- [ ] Retries and mutations preserve correctness.

### UI responsiveness
- [ ] Layout shifts are minimized.
- [ ] Animations avoid expensive layout work.
- [ ] Forms remain responsive during validation and submission.
- [ ] Loading, empty, and error states are implemented.
- [ ] Mobile and constrained-network behavior has been tested.
- [ ] Accessibility has not been compromised.

### Security and reliability
- [ ] Tenant and identity cache isolation is correct.
- [ ] Sensitive data is not exposed in client bundles.
- [ ] Stale responses cannot overwrite the active tenant's data.
- [ ] Event listeners, observers, and subscriptions are cleaned up.
- [ ] Long-session memory behavior has been considered.

### Validation
- [ ] Production build has been tested.
- [ ] Critical routes have been profiled.
- [ ] Performance budgets have been checked.
- [ ] Significant optimizations have baseline and post-change measurements.
- [ ] CI checks cover relevant performance regressions.

---

## 23. Definition of Done

A frontend feature is not complete until:

1. It follows the agreed rendering strategy and avoids unnecessary client-side JavaScript.
2. It uses the appropriate state-management and data-fetching systems.
3. It avoids redundant requests, expensive rendering, and unbounded client-side data.
4. Images, fonts, and assets are appropriately optimized.
5. Loading, empty, error, pending, and success states are handled.
6. It remains responsive on supported mobile and desktop viewports.
7. It preserves accessibility and tenant isolation.
8. Event listeners, subscriptions, and other resources are cleaned up correctly.
9. Relevant automated tests pass.
10. Production performance has been measured for critical or performance-sensitive workflows.
11. Any performance-budget exception is documented with evidence and an explicit rationale.
12. The implementation does not introduce an avoidable performance regression.

---

## 24. Guiding Principle

KAMPYN's frontend must remain fast as the product grows in users, tenants, features, and data volume.

Performance is achieved through deliberate rendering boundaries, efficient state ownership, bounded data retrieval, optimized assets, responsive interactions, and continuous measurement—not indiscriminate memoization or unnecessary architectural complexity.

**Build for responsiveness, measure real behavior, and optimize the bottleneck without compromising correctness, security, or accessibility.**
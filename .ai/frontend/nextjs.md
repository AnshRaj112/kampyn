# Next.js Engineering Standards

## Purpose

This document defines the engineering standards for building, structuring, optimizing, securing, and deploying KAMPYN's frontend applications using Next.js and TypeScript.

KAMPYN is a multi-tenant university hospitality platform that supports food ordering, bookings, inventory management, university administration, community features, notifications, search, and other campus services. Its frontend must support high traffic, multiple user roles, tenant-specific experiences, responsive interfaces, and both SaaS and self-hosted deployments.

Next.js is the primary frontend framework and provides routing, server rendering, static generation, server-side capabilities, asset optimization, and application-level performance features.

Every Next.js application must be:

- **Architecturally sound:** Follow clear boundaries between routing, presentation, business logic, and data access.
- **Secure:** Protect sensitive data, authentication flows, server operations, and tenant isolation.
- **Performant:** Minimize unnecessary JavaScript, network requests, rendering work, and resource usage.
- **Scalable:** Support increasing users, tenants, features, and deployment environments.
- **Accessible:** Meet the project's accessibility standards.
- **Maintainable:** Use reusable, cohesive modules and predictable conventions.
- **Type-safe:** Use TypeScript and runtime validation at external data boundaries.
- **Deployable:** Support consistent builds and configurations across SaaS and self-hosted environments.

This document is the source of truth for Next.js-specific engineering practices. It complements the frontend architecture, components, hooks, forms, state-management, data-fetching, performance, security, testing, and accessibility policies.

---

## 1. Core Principles

All Next.js development must follow these principles:

1. Use the **Next.js App Router** as the standard routing architecture.
2. Use **TypeScript** with strict type checking.
3. Prefer React Server Components (RSC) by default.
4. Use Client Components only when browser-side interactivity or client-only capabilities require them.
5. Keep client boundaries as narrow as practical.
6. Use Server Components for server-side data loading and rendering when appropriate.
7. Keep business logic in feature or domain modules rather than inside page components.
8. Use Route Handlers for HTTP endpoints that genuinely belong to the Next.js application.
9. Keep the backend as the authoritative owner of business rules, persistence, and authorization.
10. Use TanStack Query for client-side server-state synchronization where needed.
11. Use Zustand for justified shared client state, not as a default global state container.
12. Use Zod for runtime validation of external and user-provided data.
13. Avoid duplicating backend capabilities in Next.js without an explicit architectural reason.
14. Use environment variables and deployment configuration safely.
15. Optimize rendering and data fetching based on actual application requirements.
16. Build tenant-aware experiences without trusting client-provided tenant identifiers.
17. Ensure all routes have explicit loading, error, empty, and authorization behavior where applicable.
18. Design applications to support both managed SaaS and university self-hosted environments.
19. Keep source files cohesive and normally at or below 200 lines.
20. Prefer simple, well-supported Next.js patterns over unnecessary framework abstractions.

---

## 2. Framework and Project Configuration

### 2.1 App Router

Use the Next.js App Router for all new application routing.

The App Router provides:

- File-system routing
- Nested layouts
- React Server Components
- Streaming and Suspense
- Route Handlers
- Loading and error boundaries
- Server-side rendering and static rendering strategies

Do not introduce the Pages Router into new features.

Existing Pages Router code must be migrated only through a deliberate migration plan that accounts for routing, rendering, data fetching, and deployment behavior.

### 2.2 TypeScript

All production application code must use TypeScript.

Enable strict compiler settings and avoid weakening type safety to bypass implementation issues.

Requirements:

- Avoid `any` unless a reviewed exception is necessary.
- Prefer `unknown` for values of uncertain shape.
- Validate untrusted data at runtime.
- Use explicit types for exported functions, hooks, services, and API contracts.
- Infer types from Zod schemas where appropriate.
- Avoid duplicated or conflicting definitions of shared contracts.
- Use discriminated unions for complex state and response types.

TypeScript types do not provide runtime validation and must not be treated as a security boundary.

### 2.3 Configuration

Keep framework configuration centralized and intentional.

The configuration must be reviewed when modifying:

- Rendering behavior
- Image optimization
- Redirects and rewrites
- Security headers
- Build output
- Experimental features
- Caching
- Server actions
- Internationalization
- Deployment output

Do not enable experimental or unstable Next.js features in production without a documented reason, compatibility review, and rollback plan.

### 2.4 Dependency Management

- Use approved and compatible package versions.
- Avoid introducing overlapping libraries for the same responsibility.
- Keep framework and React versions compatible.
- Remove unused dependencies.
- Review security advisories and breaking changes before upgrades.
- Lock dependencies consistently through the project's package manager.

Do not upgrade major framework dependencies as an unrelated part of a feature implementation.

---

## 3. Recommended Application Structure

The following structure is illustrative and should be adapted to the repository's existing organization.

```text
src/
├── app/
│   ├── (marketing)/
│   │   ├── page.tsx
│   │   ├── about/
│   │   │   └── page.tsx
│   │   └── layout.tsx
│   │
│   ├── (auth)/
│   │   ├── login/
│   │   │   └── page.tsx
│   │   ├── register/
│   │   │   └── page.tsx
│   │   └── layout.tsx
│   │
│   ├── (platform)/
│   │   ├── dashboard/
│   │   │   └── page.tsx
│   │   ├── food/
│   │   │   └── page.tsx
│   │   ├── bookings/
│   │   │   └── page.tsx
│   │   └── layout.tsx
│   │
│   ├── api/
│   │   └── health/
│   │       └── route.ts
│   │
│   ├── error.tsx
│   ├── global-error.tsx
│   ├── not-found.tsx
│   ├── loading.tsx
│   ├── layout.tsx
│   └── globals.css
│
├── components/
│   ├── ui/
│   ├── layout/
│   └── shared/
│
├── features/
│   ├── auth/
│   ├── food/
│   ├── bookings/
│   ├── inventory/
│   ├── community/
│   └── notifications/
│
├── hooks/
├── lib/
│   ├── api/
│   ├── auth/
│   ├── config/
│   ├── validation/
│   └── utils/
│
├── providers/
├── stores/
├── styles/
└── types/
```

### 3.1 Route Ownership

The `app/` directory must primarily define:

- Route hierarchy
- Layout hierarchy
- Page entry points
- Route-level loading and error boundaries
- Metadata
- Route Handlers where appropriate

Keep page files small and delegate substantial behavior to feature modules.

### 3.2 Feature Ownership

The `features/` directory should own domain-specific frontend behavior, including:

- Components
- Hooks
- Schemas
- Client-side state
- API integration
- Feature-specific types
- UI transformations

Avoid importing internal implementation details across features without an explicit shared contract.

### 3.3 Shared Modules

Shared components and utilities must have clear ownership and stable interfaces.

Do not move a module into a global shared directory merely because two components use it. Ensure the shared abstraction genuinely represents common behavior and does not create inappropriate coupling.

### 3.4 Route Groups

Use route groups to organize routes without changing their public URL structure.

Examples:

- `(marketing)`
- `(auth)`
- `(platform)`
- `(admin)`

Route groups should represent meaningful layout, access, or organizational boundaries.

Avoid excessive nesting and redundant layouts.

### 3.5 Dynamic Routes

Use dynamic route segments only when the URL represents a meaningful resource or parameter.

Examples:

```text
food-courts/[foodCourtId]
bookings/[bookingId]
universities/[universitySlug]
```

Validate route parameters at the boundary and confirm that the requested resource belongs to the authenticated user's authorized tenant and access scope.

Never treat the presence of a valid-looking route identifier as proof of authorization.

---

## 4. React Server Components

### 4.1 Server Components by Default

Components in the App Router are Server Components by default.

Prefer Server Components for:

- Initial page rendering
- Server-side data retrieval
- Metadata generation
- Static or request-specific content
- Rendering content that does not require browser APIs
- Reducing client-side JavaScript

Do not add `"use client"` to a component simply because it uses props, renders JSX, or needs to display server-provided data.

### 4.2 Server Component Responsibilities

Server Components may:

- Read approved server-side data sources
- Call backend APIs through approved server-side clients
- Render static or request-specific content
- Compose Client Components
- Pass safe, serializable data to Client Components
- Perform server-side input validation for relevant operations

They must not expose secrets, internal credentials, or unauthorized server data to the client.

### 4.3 Server Component Limitations

Server Components must not use browser-only APIs or client-only React hooks.

They cannot directly manage browser interaction state such as:

- Click-driven local state
- Browser event listeners
- Client-side effects
- Client-side store subscriptions

Move only the interactive portion into a Client Component rather than converting an entire page or layout.

### 4.4 Server-to-Client Data

When passing data to Client Components:

- Pass only the data required by the client.
- Ensure values are serializable under the relevant React and Next.js constraints.
- Avoid exposing private server-side fields.
- Avoid passing large objects unnecessarily.
- Prefer explicit view models rather than raw database records.

Do not pass database clients, secrets, server-only services, or privileged backend objects across the server-client boundary.

### 4.5 Server-Only Modules

Mark sensitive server-only modules using the project's approved server-only boundary mechanism.

Server-only code must not be imported by Client Components or modules that may enter the client bundle.

Review shared modules carefully to ensure they do not accidentally pull server dependencies into browser code.

---

## 5. Client Components

### 5.1 When to Use Client Components

Use Client Components when browser-side capabilities are required, including:

- Interactive controls
- Local component state
- Client-side effects
- Browser event handling
- Zustand subscriptions
- Client-side TanStack Query behavior
- Interactive forms
- WebSocket subscriptions
- Browser storage integrations where approved
- Rich interactive visualizations

### 5.2 Narrow Client Boundaries

Keep Client Components as small and localized as possible.

For example, a page containing a static menu and an interactive cart should render the menu on the server where appropriate and isolate cart interactions within a Client Component.

Avoid marking entire route trees as client-rendered without a strong reason.

### 5.3 Client Component Responsibilities

Client Components may own presentation and interaction logic, but must not become authoritative for:

- Authorization
- Payment calculations
- Inventory correctness
- Booking availability
- Tenant access decisions
- Privileged data access
- Durable business rules

These responsibilities must be enforced by trusted backend services.

### 5.4 Client Component Boundaries

A Client Component can import other client-compatible modules, but should not import server-only code.

Keep the client dependency graph free from:

- Database clients
- Secret configuration
- Server-only services
- Backend credentials
- Filesystem-dependent server utilities

Review shared imports when introducing new dependencies.

### 5.5 Hydration

Client Components must render consistently between server-generated HTML and the initial client render.

Avoid using unstable values during initial rendering, including:

- Random values
- Browser-only state unavailable on the server
- Time-dependent output without a stable strategy
- Inconsistent locale assumptions
- Uncontrolled external data

Do not suppress hydration warnings as a routine solution. Identify and fix the source of the mismatch.

---

## 6. Routing and Navigation

### 6.1 Navigation APIs

Use Next.js navigation APIs for application routing.

Prefer:

- `Link` for standard navigation
- `useRouter` for imperative navigation when required
- `usePathname` for pathname-dependent UI
- `useSearchParams` for client-side query parameter access
- Server-side route parameter handling where supported by the installed Next.js version

Avoid direct `window.location` navigation for ordinary internal routing unless a full document navigation is specifically required.

### 6.2 URL Design

URLs should be:

- Predictable
- Readable
- Stable
- Meaningful
- Free from sensitive information

Do not place passwords, authentication codes, payment credentials, or private form values in query strings or path segments.

### 6.3 Query Parameters

Treat URL query parameters as untrusted input.

Validate and normalize them before use.

Use query parameters for suitable shareable state such as:

- Search queries
- Sort order
- Pagination
- Non-sensitive filters
- Tab selection where appropriate

Do not store large, confidential, or highly transient UI state in URLs.

### 6.4 Navigation State

Avoid unnecessary synchronization between URL state, local component state, Zustand, and TanStack Query.

Define a clear source of truth for each value.

When a filter is intentionally shareable or bookmarkable, the URL may own that state. Keep the other state layers synchronized through explicit, predictable behavior.

### 6.5 Redirects

Redirect destinations must be validated to prevent open redirect vulnerabilities.

Do not blindly trust client-provided `returnTo`, `redirect`, or equivalent parameters.

Prefer allowlisted internal destinations or strict same-origin validation.

### 6.6 Route Protection

Protected routes must enforce authorization on the backend or trusted server-side boundary.

Client-side guards may improve user experience but must never be the sole security mechanism.

Route protection must account for:

- Authentication
- Tenant membership
- Role permissions
- Resource ownership
- Resource state
- Feature access

---

## 7. Layouts and Templates

### 7.1 Root Layout

The root layout must define the minimum shared application structure, including:

- HTML document structure
- Global providers where genuinely required
- Global styles
- Shared metadata defaults
- Application-wide UI infrastructure

Avoid placing feature-specific data fetching or complex business logic in the root layout.

### 7.2 Nested Layouts

Use nested layouts for meaningful shared structures such as:

- Marketing website layout
- Authentication layout
- Student dashboard layout
- Vendor dashboard layout
- University administration layout

Keep layouts focused on shared presentation and route-level composition.

### 7.3 Persistent UI

Layouts can preserve shared UI between navigations.

Use this behavior intentionally for navigation, dashboards, and other persistent interface elements.

Do not assume that navigating between routes resets all nested client state.

Define state reset and persistence behavior explicitly for feature transitions.

### 7.4 Templates

Use templates when a route transition must create a new instance of the contained subtree rather than preserving layout state.

Do not introduce templates without a clear lifecycle or interaction requirement.

---

## 8. Data Fetching

### 8.1 General Strategy

Choose the data-fetching mechanism according to where the data is consumed and how it changes.

| Requirement | Preferred approach |
|---|---|
| Initial server-rendered content | Server Component |
| Request-specific server data | Server Component or approved server-side service |
| Client-side server-state synchronization | TanStack Query |
| User-triggered server mutation | Server Action or API mutation according to the backend boundary |
| External webhook or HTTP endpoint | Route Handler where appropriate |
| Shared client interaction state | Zustand where justified |
| Static public content | Static generation or approved content source |

Avoid fetching the same data independently on the server and client without a defined hydration or synchronization strategy.

### 8.2 Backend Integration

KAMPYN's backend is the authoritative owner of business logic, persistence, and access control.

The frontend must use approved API clients and service modules rather than directly connecting to databases or internal infrastructure.

Keep API base URLs and request behavior centralized.

### 8.3 Server-Side Fetching

Server-side data fetching should:

- Use approved backend service clients.
- Apply authentication and authorization context appropriately.
- Avoid leaking private data into rendered output.
- Use deliberate cache behavior.
- Handle errors and missing resources explicitly.
- Avoid unnecessary sequential requests.
- Use parallel requests when independent operations can safely run concurrently.

### 8.4 Client-Side Fetching

Use TanStack Query for client-side server-state requirements such as:

- Refreshable dashboard data
- Search suggestions
- Dynamic availability
- Infinite scrolling
- Live or periodically refreshed views
- User-triggered data refresh
- Mutation-driven cache updates

Define query keys, stale times, retry behavior, and invalidation policies according to the data's consistency requirements.

### 8.5 Avoid Waterfalls

Identify unnecessary sequential fetches.

When data dependencies are independent, fetch concurrently where practical.

When dependencies are real, model them explicitly rather than using nested effects or unnecessary client-side orchestration.

### 8.6 Runtime Validation

Validate data from external services and APIs at appropriate boundaries.

Use Zod or approved runtime validation for data that cannot be trusted solely through compile-time types.

Do not assume that a TypeScript interface proves the response is structurally valid.

---

## 9. Caching and Rendering Strategy

Caching behavior must be deliberate and aligned with data sensitivity, freshness, tenant isolation, and the installed Next.js version.

### 9.1 Rendering Modes

Select rendering strategies according to the route and its data.

| Strategy | Appropriate use |
|---|---|
| Static rendering | Public content that can safely be shared |
| Dynamic server rendering | Request-specific or user-specific content |
| Incremental regeneration | Public or shared content with controlled freshness |
| Client-side fetching | Interactive or frequently changing data |
| Streaming | Pages with independently loadable sections |

Do not use static rendering for private, tenant-specific, or user-specific content unless the caching and isolation behavior is proven safe.

### 9.2 Cache Safety

Before enabling caching, determine:

- Whether the data is public or private
- Whether it is tenant-specific
- Whether it is user-specific
- How frequently it changes
- Whether stale data is acceptable
- How invalidation works
- Whether the cache can be shared across requests or tenants

Never allow one user's private data to be served to another user through shared caching.

### 9.3 Next.js Cache APIs

Use only cache APIs and configuration supported by the project's installed Next.js version.

Next.js caching semantics can change between versions and rendering models. Do not assume that fetch caching, route caching, or component caching behaves identically across upgrades.

Document non-obvious cache behavior and verify it through integration tests.

### 9.4 TanStack Query Cache

TanStack Query's client cache is distinct from Next.js server-side caching.

Define how these layers interact when data is prefetched, hydrated, invalidated, or refreshed.

Avoid maintaining conflicting freshness assumptions across server and client caches.

### 9.5 Dynamic and Sensitive Data

Authentication-sensitive and tenant-specific data must use explicit cache boundaries.

Prefer no shared caching for sensitive request-specific content unless a carefully reviewed design proves that data isolation and invalidation are correct.

### 9.6 Cache Invalidation

Cache invalidation must follow successful authoritative mutations and the appropriate consistency model.

Do not invalidate every query or route indiscriminately.

For workflows that update multiple resources, document which server and client caches require refresh.

---

## 10. Server Actions

### 10.1 Appropriate Use

Server Actions may be used for appropriate server-side mutations that are tightly integrated with the Next.js application.

Examples may include:

- Small application-owned form submissions
- Internal UI mutations
- Simple operations that benefit from server-side execution

They must fit the established backend ownership and API architecture.

### 10.2 Backend Boundary

Server Actions must not become a parallel business backend or bypass the established KAMPYN backend services.

If a business operation belongs to the Go backend, the Server Action should call the approved backend API or service boundary rather than reimplementing its business rules.

### 10.3 Security

Treat Server Actions as externally invocable server endpoints.

They must:

- Authenticate the caller.
- Authorize the requested operation.
- Validate all input at runtime.
- Enforce tenant isolation.
- Avoid trusting client-supplied role or ownership fields.
- Return safe, minimal responses.
- Avoid exposing secrets or internal exceptions.
- Protect consequential mutations against duplicate execution.

Do not assume that Server Actions are safe merely because they are not directly exposed as traditional REST endpoints.

### 10.4 Input and Output

Use explicit input and output types, with runtime validation at the action boundary.

Return structured success and error states where appropriate.

Avoid returning database models, internal service objects, or excessive backend details.

### 10.5 Server Actions and Forms

Use Server Actions for form workflows only when they align with the project's API ownership, error handling, validation, and observability conventions.

For forms that rely on TanStack Query, dynamic client state, or backend API contracts, use the established client mutation approach where it provides clearer ownership.

### 10.6 Side Effects

Do not perform irreversible side effects without defining transaction, idempotency, and recovery behavior.

Server Actions do not provide automatic distributed transactions across the backend, database, payment providers, or other external systems.

---

## 11. Route Handlers

### 11.1 Appropriate Use

Use Route Handlers for HTTP endpoints that genuinely belong to the Next.js application.

Possible use cases include:

- Application-owned webhooks
- Authentication callbacks where architecturally appropriate
- Health endpoints
- Lightweight backend-for-frontend endpoints
- File-upload coordination where explicitly approved
- Proxying approved backend requests where required

Do not create Route Handlers for every backend endpoint merely to add another layer of forwarding.

### 11.2 Responsibilities

Route Handlers must:

- Validate request input.
- Authenticate and authorize as required.
- Apply tenant isolation.
- Return explicit status codes and response structures.
- Handle expected errors.
- Avoid exposing internal exceptions.
- Use safe HTTP methods and caching behavior.
- Apply suitable rate limiting or abuse protections where required.
- Emit appropriate operational telemetry without sensitive data.

### 11.3 Backend Proxying

When acting as a backend-for-frontend layer, a Route Handler must have a clear reason to exist.

Examples include:

- Hiding a private upstream service behind a controlled server boundary
- Adapting a response for a specific frontend requirement
- Coordinating multiple backend requests
- Applying application-specific request policy

Do not create redundant proxy layers that add latency, duplicate validation, and complicate debugging without a clear benefit.

### 11.4 Webhooks

Webhook handlers must verify authenticity according to the provider's supported mechanism.

They must handle duplicate delivery, retries, malformed input, and replay considerations.

A webhook handler should acknowledge only according to the provider contract and the durability guarantees of the receiving workflow.

---

## 12. Authentication and Authorization

### 12.1 Authentication

Authentication must follow the approved KAMPYN identity architecture.

Next.js may consume and coordinate authentication state, but it must not independently redefine identity or credential verification rules.

Use secure server-side mechanisms for session and credential handling as specified by the authentication architecture.

### 12.2 Session Handling

- Use secure cookie configuration where cookie-based sessions are adopted.
- Set suitable `HttpOnly`, `Secure`, and `SameSite` attributes according to the authentication design.
- Avoid exposing session secrets to Client Components.
- Define session renewal and expiration behavior.
- Handle session invalidation and logout consistently.
- Avoid persisting sensitive credentials in browser storage.

### 12.3 Authorization

Authorization must be enforced by trusted backend services or explicitly approved server-side authorization boundaries.

Frontend checks may control visibility and navigation but cannot grant access.

Validate access for:

- Tenant membership
- User role
- Resource ownership
- Administrative actions
- Sensitive data
- Feature permissions
- Resource state transitions

### 12.4 Protected Rendering

Server-side checks can prevent protected content from being rendered for unauthorized requests.

However, protected data must also be secured at the API boundary.

Never rely solely on layout-level redirects, middleware checks, or client-side guards.

### 12.5 Middleware and Proxy Boundaries

Use middleware or the version-appropriate request interception mechanism for appropriate cross-cutting concerns, such as lightweight routing decisions or request preprocessing.

Do not place complex business authorization, database workflows, or heavy application logic in middleware.

Middleware must not be considered the only security layer.

Use only mechanisms supported by the project's installed Next.js version and document their execution and deployment constraints.

---

## 13. Multi-Tenancy

KAMPYN must support multiple universities with isolated data, configuration, branding, and permissions.

### 13.1 Tenant Context

Tenant resolution must use the approved server-side tenant identification strategy.

Possible inputs may include:

- Verified hostname or subdomain
- Authenticated membership
- Trusted server-side routing context
- Approved university-specific deployment configuration

A tenant identifier provided by the browser must never be treated as proof of membership or authorization.

### 13.2 Tenant-Aware Rendering

Tenant-specific UI may include:

- University branding
- Feature availability
- Campus locations
- Food court listings
- Booking policies
- Navigation
- Localization
- Institutional configuration

Tenant configuration must be loaded through approved, validated boundaries.

### 13.3 Cache Isolation

Tenant-scoped content must be isolated across:

- Server-side data caches
- TanStack Query caches
- Zustand stores where applicable
- Client-side persisted state
- CDN or reverse-proxy caching
- Generated static content

Never assume that a request-specific tenant value is automatically part of every cache key.

### 13.4 Tenant Switching

When a user switches universities or active tenant context:

- Revalidate access to the selected tenant.
- Reset or reconcile tenant-scoped client state.
- Invalidate or clear affected client caches.
- Prevent stale content from remaining visible.
- Revalidate permissions and resource ownership.
- Avoid submitting forms created under a previous tenant context.

### 13.5 Self-Hosted Universities

Support self-hosted deployments through explicit configuration rather than university-specific code forks.

Tenant and deployment configuration must not expose secrets or permit unauthorized configuration changes.

---

## 14. Metadata and SEO

### 14.1 Metadata API

Use the Next.js Metadata API for route-level metadata.

Metadata must be:

- Accurate
- Descriptive
- Consistent with page content
- Appropriate to the route's visibility
- Safe from untrusted input

### 14.2 Public Versus Private Routes

Public marketing pages, public listings, and suitable discovery content may be indexed when intended.

Private dashboard, authentication, personal, and administrative routes must not unintentionally expose sensitive content through metadata or indexing.

Use appropriate indexing directives and ensure protected content is not rendered publicly.

### 14.3 Dynamic Metadata

Generate dynamic metadata only from validated and authorized data.

Avoid exposing confidential tenant, user, booking, complaint, or administrative information in titles, descriptions, and Open Graph data.

### 14.4 Canonical URLs

Use canonical URLs for public content where relevant.

Ensure canonical values use trusted, normalized origins and do not allow arbitrary host or redirect injection.

---

## 15. Performance Optimization

### 15.1 Core Strategy

Optimize the application around real user experience, not isolated framework metrics.

Focus on:

- Initial load performance
- Interaction responsiveness
- Navigation performance
- Rendering efficiency
- Network efficiency
- Image and font delivery
- JavaScript bundle size
- Runtime memory usage

### 15.2 JavaScript Bundle

- Prefer Server Components for non-interactive content.
- Keep client boundaries narrow.
- Avoid importing large libraries into shared client entry points unnecessarily.
- Lazy-load heavy components when appropriate.
- Analyze bundle output periodically.
- Remove unused dependencies and code.
- Avoid duplicating packages or implementations.

### 15.3 Dynamic Imports

Use dynamic imports for components that are expensive and not required for initial rendering, such as complex visualizations, editors, or non-critical interactive tools.

Do not lazy-load essential content in ways that harm accessibility, discoverability, or perceived performance.

### 15.4 Streaming and Suspense

Use Suspense and streaming when independent page sections can render progressively.

Ensure fallback content is meaningful and does not cause major layout shifts.

Do not introduce Suspense boundaries without understanding the rendering and error behavior of the relevant components.

### 15.5 Images

Use the Next.js image optimization system where appropriate.

- Provide dimensions or an aspect ratio to reduce layout shifts.
- Use appropriate responsive sizing.
- Prioritize only genuinely critical above-the-fold images.
- Use suitable image formats and quality.
- Avoid unnecessarily large source images.
- Configure remote image hosts explicitly.
- Do not allow arbitrary remote image hosts.

For university branding, menu images, and profile images, follow the platform's approved image storage and delivery architecture.

### 15.6 Fonts

Use the project's approved font-loading approach.

Avoid unnecessary font weights, styles, and external network dependencies.

Ensure fonts do not create avoidable layout shifts or block rendering.

### 15.7 Navigation

Use prefetching appropriately for routes where it improves navigation.

Avoid aggressive prefetching of large, rarely visited, or expensive routes when it creates unnecessary traffic or resource consumption.

### 15.8 Rendering Performance

Avoid:

- Large Client Components for static content
- Repeated fetching of the same data
- Unnecessary state synchronization
- Expensive calculations on every render
- Unbounded client-side lists
- Excessive context updates
- Unnecessary hydration

Use profiling and production measurements to validate optimizations.

---

## 16. Styling and Design System

### 16.1 Styling Strategy

Follow the existing KAMPYN design-system standards.

Use the approved combination of:

- Tailwind CSS
- SCSS Modules
- Shared design tokens
- Reusable UI components

Avoid introducing additional styling frameworks without an explicit architectural decision.

### 16.2 Component Styling

- Keep component-specific styles close to their components.
- Use shared tokens for colors, spacing, typography, and radius.
- Avoid duplicating global styles.
- Use responsive design conventions.
- Support light and dark themes if enabled by the application.
- Keep interactive states visually consistent.

### 16.3 Server and Client Styling

Ensure styling works consistently across Server and Client Components.

Avoid styling approaches that depend on unavailable browser state during server rendering.

### 16.4 Responsive Design

All routes must support the target mobile, tablet, and desktop experiences.

Campus users may rely on mobile devices and variable network conditions. Layouts must remain usable at small widths and with touch input.

### 16.5 Accessibility

Use semantic HTML and the approved accessible component primitives.

Do not sacrifice keyboard support, focus visibility, contrast, or screen-reader compatibility for visual consistency.

---

## 17. Providers and Context

### 17.1 Provider Placement

Place providers at the narrowest level that satisfies their consumers.

Examples include:

- Query client provider
- Theme provider
- Authentication context
- Feature-specific context
- Shared UI infrastructure

Do not place every provider at the root layout by default.

### 17.2 Client Provider Boundaries

Providers that depend on client-side state or browser APIs must be implemented as Client Components.

Keep server layouts as Server Components where possible and compose the required client providers through narrow boundaries.

### 17.3 Context Design

Use Context for suitable dependency injection and shared state where its update characteristics are appropriate.

Do not use one large global context for unrelated application data.

Use Zustand for shared state when its selector-based subscription model better fits the requirement.

### 17.4 Provider Lifecycle

Providers must have clear initialization, update, and cleanup behavior.

Avoid recreating expensive provider instances on every render or resetting global state unexpectedly during navigation.

---

## 18. Error Handling and Loading States

### 18.1 Loading UI

Use route-level and component-level loading states where appropriate.

Provide meaningful feedback for:

- Page transitions
- Server data loading
- Client-side queries
- Form submissions
- Uploads
- Background refreshes

Avoid indefinite spinners without explanatory context.

### 18.2 Error Boundaries

Use the appropriate App Router error boundaries for route-level rendering failures.

Implement `error.tsx` and `global-error.tsx` according to the application's routing and layout requirements.

Error UIs must provide safe, understandable feedback and appropriate recovery actions.

### 18.3 Not Found

Use `not-found.tsx` or `notFound()` for missing resources where appropriate.

Do not reveal whether a sensitive resource exists to unauthorized users when the security model requires indistinguishable responses.

### 18.4 Error Classification

Distinguish between:

- Validation errors
- Authentication failures
- Authorization failures
- Missing resources
- Business conflicts
- Network failures
- Server failures
- Unexpected rendering errors

Avoid showing raw internal exceptions to users.

### 18.5 Recovery

Where practical, allow users to retry, navigate safely, refresh data, or return to a known working route.

Do not automatically retry consequential mutations without an idempotency strategy.

---

## 19. Environment Variables and Configuration

### 19.1 Server-Only Secrets

Secrets must remain on the server.

Examples include:

- Backend service credentials
- Database credentials
- Signing secrets
- Internal API tokens
- Payment provider secrets
- Private integration credentials

Never prefix a secret with `NEXT_PUBLIC_`.

### 19.2 Public Environment Variables

Variables prefixed with `NEXT_PUBLIC_` must be treated as public, since their values can be embedded in client-side bundles.

Only non-sensitive configuration may use public environment variables.

### 19.3 Configuration Validation

Validate required environment variables during startup or build according to the deployment model.

Use Zod or an equivalent runtime validation approach for environment configuration.

Fail clearly when required configuration is missing or malformed.

### 19.4 Environment Separation

Keep development, testing, staging, production, and self-hosted deployment configurations clearly separated.

Avoid hardcoded URLs, credentials, or environment-specific behavior in application source code.

### 19.5 Runtime Versus Build-Time Configuration

Distinguish between build-time and runtime configuration.

For self-hosted deployments, ensure that required configuration can be supplied through the documented deployment process without rebuilding the application unnecessarily, where the chosen Next.js deployment mode supports runtime configuration.

Document configuration constraints for each supported deployment target.

---

## 20. Security

### 20.1 General Security

Follow the KAMPYN security architecture and frontend security policies.

Next.js-specific requirements include:

- Keep secrets server-side.
- Validate untrusted route and request input.
- Enforce authorization at trusted boundaries.
- Prevent cross-tenant data leakage.
- Secure Server Actions and Route Handlers.
- Use safe redirect validation.
- Avoid unsafe HTML injection.
- Configure security headers through the approved deployment architecture.
- Keep dependencies patched.
- Avoid exposing internal implementation details.

### 20.2 Content Security Policy

Use an approved Content Security Policy strategy appropriate to the application's rendering, asset, and third-party integration requirements.

Avoid unnecessarily broad script, style, image, and connection sources.

Test CSP behavior against:

- Next.js runtime requirements
- Image delivery
- Authentication
- Analytics
- Payment integrations
- Approved embedded services

Do not disable CSP protections merely to resolve integration errors without a security review.

### 20.3 Cross-Site Scripting

Avoid rendering user-provided HTML.

If rendering rich content is a genuine requirement, use an approved sanitization strategy and enforce safe content policies at the appropriate boundary.

Do not treat TypeScript types or client-side validation as XSS protection.

### 20.4 Request Security

Use appropriate protections for cookies, state-changing requests, redirects, webhooks, and sensitive operations.

Review the security model whenever introducing a new server-side endpoint or changing authentication behavior.

### 20.5 Dependency Security

Review new dependencies for:

- Maintenance status
- Known vulnerabilities
- Bundle impact
- Transitive dependencies
- License compatibility
- Runtime requirements

Do not add a large dependency for functionality that can be safely and clearly implemented using existing platform capabilities.

---

## 21. Search and URL-Driven Features

KAMPYN may use OpenSearch or other approved search services through the backend.

### 21.1 Search Integration

Search pages must use approved backend search APIs rather than connecting directly to OpenSearch from the browser.

The frontend should handle:

- Search input
- Filters
- Sorting
- Pagination
- Loading and empty states
- Error recovery
- Search result rendering

### 21.2 Search State

Use URL query parameters for shareable, non-sensitive search state where appropriate.

Use debouncing and request cancellation or stale-response handling for interactive search.

### 21.3 Search Security

The backend must enforce tenant scope, permissions, and data visibility.

Do not assume that filtering results in the frontend provides access control.

### 21.4 Search Performance

Avoid excessive requests, large unbounded result sets, and unnecessary rerenders.

Use backend pagination and suitable caching strategies for the workload.

---

## 22. Real-Time Features

KAMPYN may provide real-time updates for orders, bookings, notifications, inventory, community messaging, and other campus operations.

### 22.1 Connection Ownership

Use an approved real-time connection architecture.

Avoid creating separate duplicate WebSocket connections for every component.

### 22.2 Message Validation

Treat all incoming real-time messages as untrusted.

Validate their structure and handle unsupported or malformed events safely.

### 22.3 Tenant and User Scope

Real-time subscriptions must be authorized and scoped according to the backend's identity and tenant model.

Do not trust client-supplied room names or channel identifiers as authorization.

### 22.4 Reconnection

Define reconnection behavior, backoff, stale-state reconciliation, and duplicate-event handling.

A successful WebSocket connection does not guarantee that all events have been received.

### 22.5 Server-State Reconciliation

Use TanStack Query or the approved feature state architecture to reconcile real-time events with server-owned data.

For critical workflows, fetch authoritative state after reconnecting or detecting a gap.

---

## 23. Internationalization and Localization

Next.js applications must support localization according to the product's approved internationalization architecture.

Requirements:

- Use the approved localization library and routing conventions.
- Avoid hardcoded user-facing strings in reusable components.
- Support locale-aware date, time, and number formatting.
- Preserve Unicode input.
- Handle right-to-left layouts if required by supported locales.
- Avoid assuming a single country-specific address or name format.
- Ensure metadata and navigation reflect the active locale where applicable.
- Validate localized forms against stable business rules.

Localization must not change the underlying API contract or weaken input validation.

---

## 24. Testing

Next.js applications require testing at multiple levels.

### 24.1 Unit Tests

Test:

- Pure utility functions
- Data transformations
- Validation schemas
- Query-key factories
- Permission-dependent presentation logic
- Feature state transitions
- Formatting and parsing

### 24.2 Component Tests

Test:

- Component rendering
- Client-side interactions
- Form validation
- Loading and error states
- Accessibility behavior
- Conditional rendering
- Keyboard interaction
- State transitions

### 24.3 Server Component Tests

Use the repository's approved testing approach for server-rendered behavior.

Verify important server-side data mapping, route behavior, and output contracts through appropriate integration or end-to-end tests.

Avoid brittle tests that depend heavily on internal Next.js implementation details.

### 24.4 Route Handler and Server Action Tests

Test:

- Input validation
- Authentication and authorization
- Tenant isolation
- Response structure and status codes
- Error handling
- Idempotency for consequential operations
- Safe handling of malformed requests
- Rate limiting and abuse controls where applicable

### 24.5 End-to-End Tests

Critical user journeys must have end-to-end coverage.

Examples include:

- Registration and login
- Food discovery and ordering
- Booking creation and modification
- Payment-related flows
- Complaint submission
- Inventory management
- Administrative operations
- Tenant switching
- Search and navigation

### 24.6 Rendering and Cache Tests

Where caching or rendering behavior is consequential, test the deployed or production-like behavior.

Verify:

- Public versus private rendering
- Tenant isolation
- Cache invalidation
- Revalidation
- Authentication boundaries
- Navigation state behavior
- Server and client synchronization

### 24.7 Accessibility Tests

Use automated checks alongside manual keyboard and assistive-technology testing.

Verify focus management, semantic structure, accessible names, contrast, error announcements, and interactive control behavior.

### 24.8 Build Validation

CI must verify relevant quality gates, including:

- TypeScript checks
- Linting
- Unit tests
- Integration tests
- Production build
- Applicable end-to-end tests
- Dependency and security checks

Do not treat a successful local development server as evidence that the production build is correct.

---

## 25. Deployment and Self-Hosting

KAMPYN must support controlled deployment across managed infrastructure and university-owned environments.

### 25.1 Deployment Compatibility

The frontend must be designed around the supported Next.js deployment target.

Document:

- Build requirements
- Runtime requirements
- Environment variables
- Server execution model
- Static asset handling
- Image optimization requirements
- Authentication configuration
- Backend connectivity
- Health-check behavior
- Scaling expectations

### 25.2 Docker

When containerized:

- Use approved base images.
- Prefer multi-stage builds where appropriate.
- Run with least privilege.
- Keep secrets out of images and build logs.
- Use explicit runtime configuration.
- Keep images reproducible and reasonably small.
- Provide suitable health checks.
- Handle termination and shutdown correctly.
- Follow the repository's Docker and infrastructure standards.

Use the deployment output mode and Docker strategy that match the project's supported Next.js runtime. Do not assume every Next.js feature works identically in a standalone or static deployment.

### 25.3 Self-Hosted Configuration

Universities must be able to configure their own deployment through documented, supported configuration.

Avoid university-specific source-code forks for:

- Branding
- API endpoints
- Tenant configuration
- Authentication integrations
- Feature availability
- Approved campus-specific settings

University-specific customization must remain within explicit extension points and security boundaries.

### 25.4 Scaling

Design the frontend deployment to support horizontal scaling where required.

Avoid depending on local process memory or local disk for state that must survive restarts or be shared across instances.

Use approved shared infrastructure for sessions, cache, uploaded content, and other distributed state.

### 25.5 Health and Readiness

Expose health or readiness endpoints only where they are needed by the deployment architecture.

Do not disclose sensitive operational details in public health responses.

### 25.6 Deployment Verification

Before release, verify:

- Production build succeeds.
- Required environment variables are configured.
- Backend connectivity works.
- Authentication and tenant routing behave correctly.
- Cache boundaries are safe.
- Critical routes render.
- Assets load correctly.
- Error and recovery flows behave as expected.
- Logs and telemetry contain no sensitive information.

---

## 26. Observability

Next.js applications must support operational diagnosis without exposing private user information.

### 26.1 Logging

Use the approved logging and telemetry infrastructure.

Capture appropriate details such as:

- Route and operation category
- Error classification
- Request or trace correlation identifier
- Relevant timing
- Deployment version
- Environment
- Non-sensitive tenant reference where approved

Never log credentials, session secrets, authentication codes, payment credentials, private messages, or sensitive form content.

### 26.2 Metrics

Monitor relevant signals such as:

- Route response times
- Server-side rendering latency
- API request failures
- Client-side runtime errors
- Navigation performance
- Hydration errors
- Cache hit and miss behavior where observable
- WebSocket connection failures
- Build and deployment health

### 26.3 Tracing

Use distributed tracing or correlation identifiers where supported to connect frontend server operations with backend requests.

Do not introduce unnecessary tracing overhead or leak sensitive identifiers into telemetry.

### 26.4 Error Reporting

Use approved error-reporting tools and ensure sensitive data is redacted before transmission.

Separate expected business errors from unexpected application failures.

### 26.5 Alerting

Define alerts for operationally meaningful failures, especially those affecting authentication, ordering, payments, bookings, and administrative workflows.

Avoid noisy alerts that do not lead to actionable investigation.

---

## 27. Code Quality and Maintainability

### 27.1 Separation of Concerns

Keep these responsibilities separate:

- Route definition
- Page composition
- Presentation
- Stateful interaction
- API communication
- Validation
- Business logic
- Configuration
- Infrastructure

Avoid implementing a complete feature in a single page file.

### 27.2 Reusability

Reuse shared components and feature modules where the abstractions are meaningful.

Do not overgeneralize feature-specific behavior into generic helpers that obscure business rules.

### 27.3 Complexity

Prefer clear control flow and explicit data transformations.

Avoid unnecessary nested conditionals, deeply coupled components, and large effect-driven workflows.

Use established data structures and algorithms appropriate to the expected workload.

### 27.4 Types

Use strong types at module boundaries.

Avoid:

- Unnecessary `any`
- Unsafe type assertions
- Duplicated API interfaces
- Overly broad generic types
- Types that claim external data is valid without runtime checks

### 27.5 Comments

Comments should explain non-obvious decisions, framework constraints, security considerations, or important tradeoffs.

Do not add comments that merely restate code.

### 27.6 Code Generation

Generated code must follow the same quality, security, testing, and architecture requirements as manually written code.

Do not modify generated files directly unless the generation workflow explicitly requires it.

---

## 28. Anti-Patterns

The following are prohibited unless an explicit architectural exception is approved:

- Using the Pages Router for new features.
- Marking large route trees as Client Components without justification.
- Fetching all server data from the browser by default.
- Using Server Components as a replacement for backend business services.
- Reimplementing Go backend business rules in Next.js.
- Connecting directly to databases from frontend modules.
- Exposing secrets through client-side bundles or public environment variables.
- Trusting client-side route guards as authorization.
- Trusting client-provided tenant identifiers.
- Creating redundant proxy Route Handlers.
- Using Server Actions as an unreviewed replacement for the backend API.
- Assuming Server Actions automatically provide authentication or authorization.
- Enabling unsafe shared caching for private or tenant-specific content.
- Duplicating server state across TanStack Query, Zustand, and React state without clear ownership.
- Fetching the same data repeatedly through unrelated layers.
- Building complex data-fetching logic inside page components.
- Suppressing hydration warnings instead of fixing mismatches.
- Accessing browser APIs during server rendering.
- Using unstable or experimental framework features in production without review.
- Placing secrets in URLs, metadata, client logs, or browser storage.
- Using middleware as the sole authorization boundary.
- Creating university-specific source forks for configuration-only differences.
- Using broad cache invalidation when targeted invalidation is possible.
- Implementing unbounded retries or unsafe mutation retries.
- Adding heavy dependencies without evaluating their alternatives and impact.
- Ignoring production build, deployment, and cache behavior during testing.

---

## 29. Next.js Review Checklist

### Architecture
- [ ] The App Router is used.
- [ ] Route files remain focused on routing and composition.
- [ ] Feature logic belongs to the appropriate feature module.
- [ ] Server and Client Component boundaries are deliberate.
- [ ] Shared abstractions have clear ownership.
- [ ] Files remain cohesive and normally within the 200-line limit.

### Rendering
- [ ] Server Components are used by default.
- [ ] Client Components are limited to necessary interactive boundaries.
- [ ] Server-to-client data is minimal and safe.
- [ ] Hydration behavior is deterministic.
- [ ] Rendering strategy matches data sensitivity and freshness requirements.
- [ ] Loading, error, and not-found states are defined.

### Data
- [ ] Data is fetched through approved service boundaries.
- [ ] TanStack Query is used appropriately for client-side server state.
- [ ] Query keys include relevant scope and parameters.
- [ ] Runtime validation is applied at external boundaries.
- [ ] Cache behavior is explicit.
- [ ] Cache invalidation is appropriately scoped.
- [ ] Backend business logic remains authoritative.

### Security
- [ ] Secrets remain server-side.
- [ ] Route parameters and query strings are validated.
- [ ] Authentication and authorization are enforced at trusted boundaries.
- [ ] Tenant isolation is maintained.
- [ ] Server Actions and Route Handlers validate and authorize requests.
- [ ] Redirect destinations are validated.
- [ ] Sensitive data is not exposed through rendered output, metadata, logs, or URLs.
- [ ] Security headers and CSP follow the approved architecture.

### Performance
- [ ] Client bundles are appropriately sized.
- [ ] Heavy components are loaded intentionally.
- [ ] Images and fonts are optimized.
- [ ] Unnecessary waterfalls and duplicate requests are avoided.
- [ ] Rendering and cache behavior are appropriate.
- [ ] Performance has been measured for critical routes.

### User Experience
- [ ] Navigation is predictable.
- [ ] Responsive layouts work across target devices.
- [ ] Forms and controls are accessible.
- [ ] Error and recovery paths are understandable.
- [ ] Tenant switching clears or reconciles scoped state.
- [ ] Internationalization requirements are respected.

### Quality and Operations
- [ ] Type checks and linting pass.
- [ ] Relevant unit and integration tests exist.
- [ ] Critical workflows have end-to-end tests.
- [ ] Production builds succeed.
- [ ] Environment configuration is validated.
- [ ] Deployment and self-hosting requirements are documented.
- [ ] Observability is useful and privacy-safe.

---

## 30. Definition of Done

A Next.js feature is complete only when:

- It follows the App Router architecture and repository conventions.
- Its Server and Client Component boundaries are intentional.
- Its business logic and API integration are separated appropriately.
- Its rendering and caching strategies are safe for its data.
- Its route parameters and external inputs are validated.
- Its authentication, authorization, and tenant boundaries are enforced.
- Its loading, error, empty, and recovery states are implemented.
- Its components are responsive and accessible.
- Its JavaScript and rendering performance are appropriate.
- Its sensitive data remains protected.
- Its tests cover expected behavior, failures, and critical workflows.
- Its production build and supported deployment model are verified.
- Its observability and configuration meet operational requirements.
- Its documentation is sufficient for future development and self-hosted deployment.

**A Next.js feature is not complete merely because it renders correctly in development. It is complete when its rendering, data boundaries, security, performance, accessibility, and deployment behavior are reliable in the environments KAMPYN supports.**
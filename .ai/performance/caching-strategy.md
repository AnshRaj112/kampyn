# Cache Strategy

## 1. Purpose

This document defines the caching standards for KAMPYN across the frontend, backend, databases, APIs, distributed infrastructure, and self-hosted deployments.

Caching must improve performance, reduce unnecessary computation and database load, and provide responsive experiences without compromising data correctness, tenant isolation, authorization, or consistency.

Caching is an optimization, not a source of truth. Every cache must have a defined purpose, ownership, expiration policy, invalidation strategy, failure behavior, and monitoring mechanism.

## 2. Core Principles

All caching implementations MUST follow these principles:

- **Correctness first:** Cached data must never override authoritative data or cause incorrect business decisions.
- **Explicit ownership:** Every cache must have a clearly defined owner and lifecycle.
- **Tenant isolation:** Cached data must never leak across universities, tenants, users, or authorization boundaries.
- **Bounded storage:** Every cache must have a capacity limit, eviction policy, or retention strategy.
- **Controlled expiration:** Every cached entry must have an explicit TTL or a documented reason for not expiring.
- **Targeted invalidation:** Cache invalidation must be predictable, scoped, and testable.
- **Failure tolerance:** Cache unavailability must not cause unnecessary system-wide failures.
- **Measurable value:** Caching must be justified by performance measurements, not assumptions.
- **No hidden dependencies:** Business logic must remain correct when all caches are empty or unavailable.
- **Minimal duplication:** Do not introduce multiple caching layers for the same data without a documented reason.

## 3. Cache Architecture

KAMPYN may use multiple caching layers, each serving a specific purpose.

| Layer | Technology | Primary responsibility |
|---|---|---|
| Browser cache | HTTP cache, Cache Storage | Static assets and explicitly cacheable responses |
| Next.js cache | Next.js server-side caching | Reusable server-rendered and fetched data |
| Client server-state cache | TanStack Query | API response caching and synchronization |
| Client state | Zustand | Temporary UI state, not general-purpose server caching |
| Application cache | Redis | Shared, distributed application caching |
| Process-local cache | In-memory bounded cache | Small, short-lived, non-critical local data |
| Database cache | Database-managed mechanisms | Internal query and page-level optimization |
| Search index | OpenSearch | Derived searchable data and search-oriented retrieval |
| CDN cache | CDN or reverse proxy | Public static assets and explicitly cacheable public content |

Each layer must have a clearly differentiated role. Data should not be cached at every layer by default.

### 3.1 Source of Truth

KAMPYN's authoritative data stores are responsible for persistent business state.

- PostgreSQL is the primary source of truth for relational and transactional data.
- MongoDB is used for justified document-oriented workloads.
- Redis is used for transient, cache, and coordination workloads.
- OpenSearch is a derived search index and must not be treated as the authoritative record.
- Object storage is responsible for persistent binary objects.

Cached records and derived indexes must be reconstructible from their authoritative sources or have a documented recovery mechanism.

### 3.2 Cache Selection

Before introducing a cache, answer the following:

1. Is the data expensive to compute, retrieve, or serialize?
2. Is it requested frequently enough to justify caching?
3. How stale can the data safely become?
4. What event or action makes the cached value invalid?
5. What is the expected cache size and memory cost?
6. What happens when the cache is unavailable?
7. Is the data tenant-specific, user-specific, or authorization-sensitive?
8. Can the data be cached at a more appropriate layer?

If caching does not provide a measurable benefit or introduces unacceptable consistency risks, optimize the underlying operation instead.

## 4. Cache Classification

Classify each cache entry before implementation.

| Cache type | Purpose | Typical examples |
|---|---|---|
| Reference data | Avoid repeated retrieval of stable data | Categories, supported service types |
| Read-through cache | Accelerate frequent reads | Vendor profiles, food court details |
| Computed data | Avoid expensive recomputation | Aggregated dashboard metrics |
| Session cache | Store temporary session-related data | Session metadata |
| Search cache | Avoid repeated search work | Popular public search queries |
| Configuration cache | Reduce configuration lookups | Tenant-level feature settings |
| Coordination cache | Coordinate distributed processes | Locks, rate limits, idempotency records |
| Client server-state cache | Reduce repeated API requests | Menus, booking history |
| Static asset cache | Reuse immutable resources | Versioned JavaScript and images |

Coordination data, authentication state, and business-critical records require stricter correctness guarantees than ordinary performance caches.

## 5. Cache-Aside Strategy

Cache-aside is the default strategy for backend application caching when appropriate.

### 5.1 Read Flow

1. Receive a read request.
2. Authenticate and establish the tenant and authorization context.
3. Construct a cache key containing the required isolation dimensions.
4. Check the cache.
5. If the entry exists and is valid, return the cached value.
6. If the entry is missing, retrieve data from the authoritative source.
7. Apply the required authorization and business rules.
8. Store the eligible result in the cache with an explicit TTL.
9. Return the result.

Cache lookup must never replace authorization checks. Authorization must be performed before serving data, or the cached value must be proven to be valid for the current authorization context.

### 5.2 Write Flow

For business-critical writes, the default strategy is:

1. Validate the request.
2. Authenticate and authorize the operation.
3. Execute the business operation against the authoritative datastore.
4. Commit the transaction.
5. Invalidate or update affected cache entries through a reliable mechanism.
6. Return the result according to the API contract.

Do not write to a cache first and assume that the database write will succeed.

For operations where cache invalidation must reliably follow a database commit, use a transactional outbox or another durable event-delivery mechanism.

### 5.3 Cache-Aside Example

```ts
async function getVendor(
  tenantId: string,
  vendorId: string
): Promise<Vendor> {
  const key = `kampyn:v1:tenant:${tenantId}:vendor:${vendorId}`;

  const cached = await redis.get(key);

  if (cached) {
    return vendorSchema.parse(JSON.parse(cached));
  }

  const vendor = await vendorRepository.findById({
    tenantId,
    vendorId,
  });

  if (!vendor) {
    throw new VendorNotFoundError();
  }

  await redis.set(
    key,
    JSON.stringify(vendor),
    "EX",
    300
  );

  return vendor;
}
```

This is illustrative pseudocode. Production code must also account for authorization, cache errors, serialization failures, tenant-aware repository access, and invalidation.

## 6. TTL and Expiration Policies

Every cache entry must have an explicit TTL unless its lifecycle is managed by a documented invalidation-only policy.

TTL values must reflect the rate of change, business impact of stale data, and cache capacity.

### 6.1 Suggested Initial TTLs

These values are starting points for load testing and must be adjusted according to observed usage and correctness requirements.

| Data category | Suggested TTL | Notes |
|---|---:|---|
| Public reference data | 1–24 hours | Invalidate on administrative changes |
| Vendor public profiles | 1–5 minutes | Invalidate on relevant profile updates |
| Food court menus | 30–120 seconds | Invalidate when menu or availability changes |
| Item search results | 15–60 seconds | Revalidate after catalog updates |
| Public category listings | 5–30 minutes | Version or invalidate on changes |
| Tenant configuration | 1–5 minutes | Invalidate on configuration changes |
| User profile summaries | 30–120 seconds | Scope by tenant and user |
| Dashboard aggregates | 30–180 seconds | Prefer precomputation for expensive queries |
| Booking availability | 5–15 seconds | Always confirm against authoritative state before booking |
| Inventory availability | 1–10 seconds | Never rely on cache for final stock validation |
| Order details | 5–30 seconds | Authoritative reads required for critical transitions |
| Payment status | Avoid long-lived cache | Confirm using authoritative payment records |
| Authorization decisions | Avoid general-purpose caching | If cached, use strict scope and rapid invalidation |
| Static versioned assets | 1 year | Only when content-addressed or versioned |
| Short-lived coordination keys | Operation-specific | Must have safe expiration and recovery |

### 6.2 TTL Rules

- TTL must be chosen according to the maximum acceptable staleness.
- Randomized TTL jitter SHOULD be used for high-volume keys to reduce synchronized expiration.
- Critical business data MUST NOT depend on TTL alone for correctness.
- Expiration must not be used as a substitute for event-based invalidation where timely updates are required.
- Cache entries with unknown or unbounded lifetimes MUST NOT be introduced.
- TTL changes must be reviewed for their effects on memory, database load, and stale reads.

### 6.3 TTL Jitter

When many keys are created together, they may expire at approximately the same time and create a cache stampede.

Use bounded randomized jitter for suitable entries.

```ts
function getCacheTtl(
  baseSeconds: number,
  jitterSeconds: number
): number {
  return (
    baseSeconds +
    Math.floor(Math.random() * (jitterSeconds + 1))
  );
}
```

Jitter must remain within the data's acceptable staleness window.

## 7. Cache Key Design

Cache keys must be deterministic, unambiguous, versioned where appropriate, and designed for efficient invalidation.

### 7.1 Naming Convention

Use the following general format:

```text
kampyn:{environment}:{version}:{tenant}:{domain}:{resource}:{identifier}
```

Example:

```text
kampyn:prod:v1:tenant_123:vendor:profile:vendor_456
kampyn:prod:v1:tenant_123:menu:foodcourt:fc_789
kampyn:prod:v1:tenant_123:user:profile:user_234
```

### 7.2 Key Requirements

- Include tenant identity for tenant-scoped data.
- Include user identity when data is user-specific.
- Include relevant authorization or visibility scope where it changes the result.
- Include query parameters that affect the returned value.
- Normalize identifiers and parameters consistently.
- Use key versioning when the cached data structure or serialization contract changes.
- Keep keys within the chosen Redis deployment's practical length and memory limits.
- Avoid sensitive personal information, credentials, access tokens, or payment details in keys.
- Do not include raw user-supplied strings without normalization and length limits.

### 7.3 Query-Based Keys

Search and list caches must account for all meaningful query parameters.

Example:

```text
kampyn:prod:v1:tenant_123:search:items:
  query_hash:
  filters_hash:
  sort:
  page_cursor
```

In production, construct a canonical representation of the query and hash it using a suitable stable hash function.

Do not depend on arbitrary object serialization order or omit filters that affect the response.

### 7.4 Tenant Isolation

Tenant-scoped data MUST include tenant identity in its cache key.

A tenant identifier supplied by a client must not be blindly trusted. Resolve tenant context through the authenticated request and validate the user's membership and permissions before accessing tenant data.

Never serve tenant-specific cache data based solely on a client-provided tenant identifier.

Cross-tenant access must be tested explicitly, including cache-hit scenarios.

## 8. Cache Invalidation

Cache invalidation is a core part of cache correctness and must be designed alongside the data model and write operations.

### 8.1 Invalidation Methods

| Method | Description | Suitable use |
|---|---|---|
| Key invalidation | Delete a specific key | Individual entity updates |
| Pattern or namespace invalidation | Remove related keys | Controlled bulk invalidation |
| Tag-based invalidation | Invalidate keys associated with a tag | Related query and aggregate caches |
| Versioned keys | Advance a version namespace | Large or broad invalidation |
| TTL expiration | Allow entries to expire naturally | Bounded staleness |
| Event-driven invalidation | Invalidate on domain events | Frequently updated domain data |
| Write-through update | Update cache after a successful write | Carefully controlled predictable records |

Prefer targeted invalidation over broad deletion.

### 8.2 Domain Events

Relevant domain events should drive invalidation when timely consistency matters.

Examples:

| Event | Potential cache invalidation |
|---|---|
| `VendorUpdated` | Vendor profile, vendor listings |
| `MenuUpdated` | Menu, item listings, related search results |
| `ItemPriceChanged` | Item detail, menu summaries, search results |
| `OrderCreated` | Relevant order summaries and aggregates |
| `OrderStatusChanged` | Order detail, user order list, dashboard metrics |
| `InventoryUpdated` | Inventory summaries and related availability views |
| `BookingCreated` | Relevant booking summaries and availability views |
| `BookingCancelled` | Booking summaries and availability views |
| `TenantConfigurationUpdated` | Tenant configuration and feature settings |
| `UserRoleChanged` | Relevant permission-related cached views |
| `UserDeactivated` | User-specific caches and active sessions as required |

Events must carry the identifiers necessary for scoped invalidation without exposing unnecessary sensitive data.

### 8.3 Transactional Invalidation

When a write and its invalidation event must be reliable:

1. Update the authoritative record in a database transaction.
2. Write an outbox event in the same transaction.
3. Commit the transaction.
4. Publish the outbox event asynchronously.
5. Process the event idempotently.
6. Invalidate or refresh the affected cache entries.
7. Record failures and retry safely.

Do not assume that publishing a message immediately after a database commit is reliable if the process can crash between those operations.

### 8.4 Invalidation Failure

If invalidation fails:

- Record the failure with relevant correlation and entity identifiers.
- Retry using bounded exponential backoff with jitter.
- Make the event handler idempotent.
- Alert when failures exceed defined thresholds.
- Use short TTLs or authoritative reads for critical data while invalidation is degraded.
- Provide a controlled recovery process for stale namespaces or affected cache regions.

Invalidation failures must not silently result in indefinite stale data.

## 9. Cache Stampede and Thundering Herd Protection

A cache stampede occurs when many requests attempt to recompute the same missing or expired entry simultaneously.

KAMPYN must use appropriate protection for expensive, high-traffic cache entries.

### 9.1 Single-Flight

Within a single process, coalesce concurrent requests for the same key so that only one request performs the expensive computation.

Single-flight state must be bounded and cleaned up after completion.

### 9.2 Distributed Locks

For expensive work shared across multiple backend instances, a short-lived distributed lock may be used.

Distributed locks MUST:

- Have a bounded TTL.
- Use unique ownership tokens.
- Be released only by their owner.
- Have bounded acquisition timeouts.
- Handle process crashes and lock expiration.
- Avoid being the only protection for business-critical correctness.
- Use fencing tokens or an equivalent safeguard when stale lock holders could perform unsafe writes.

A Redis lock is not a replacement for database constraints, transactions, or atomic business operations.

### 9.3 Stale-While-Revalidate

For data where bounded staleness is acceptable:

1. Serve a still-usable cached value.
2. Trigger a background refresh when the entry is stale.
3. Ensure only a bounded number of refreshes run concurrently.
4. Replace the cache entry after a successful refresh.
5. Preserve an explicit maximum stale age.

Do not use stale-while-revalidate for final inventory validation, payment confirmation, booking confirmation, or authorization decisions.

### 9.4 Negative Caching

Negative caching may be used for repeated requests for non-existent resources when it reduces unnecessary load.

Rules:

- Use short TTLs.
- Scope by tenant and resource identity.
- Invalidate when the resource is created.
- Avoid caching authorization failures as resource-not-found results unless the security contract explicitly requires it.
- Do not allow negative cache entries to hide newly created resources beyond the documented TTL.

## 10. Redis Caching Standards

Redis is KAMPYN's shared distributed cache and coordination layer where justified.

### 10.1 Redis Responsibilities

Redis may be used for:

- Shared application cache entries.
- Short-lived session metadata.
- Rate limiting and abuse protection.
- Distributed coordination.
- Temporary presence information.
- Carefully designed idempotency records.
- Queueing or event-stream workloads where the selected Redis mechanism is appropriate.

Redis MUST NOT become an undocumented replacement for authoritative persistent storage.

### 10.2 Data Structures

Choose the simplest Redis data structure that meets the workload.

| Structure | Appropriate use |
|---|---|
| String | Serialized cache values, counters, simple tokens |
| Hash | Small structured records with independently accessed fields |
| Set | Unique membership or relationship collections |
| Sorted set | Ranked items, timed schedules, ordered score collections |
| Stream | Durable-ish ordered event processing with explicit retention and recovery |
| List | Simple queue patterns when their delivery semantics are sufficient |

Use atomic Redis operations or Lua scripts for multi-step operations that require atomicity.

Avoid storing large, deeply nested objects in Redis without measuring serialization, transfer, and memory costs.

### 10.3 Memory Management

- Configure explicit memory limits.
- Choose an eviction policy suitable for the workload.
- Monitor memory fragmentation and key growth.
- Apply TTLs to transient keys.
- Avoid unbounded sets, lists, hashes, and sorted sets.
- Set maximum payload sizes for cached values.
- Monitor large keys and high-cardinality key patterns.
- Review memory implications before adding new cached domains.

Do not rely on Redis eviction behavior to preserve critical coordination or persistence data.

Where cache and coordination workloads have conflicting durability or eviction needs, isolate them into separate Redis instances or deployments.

### 10.4 Serialization

- Use stable, documented serialization formats.
- Validate data after deserialization at trust boundaries.
- Version cached schemas when formats change.
- Handle malformed or obsolete values as cache misses.
- Avoid serializing executable content.
- Avoid storing secrets or unnecessary personal information.
- Use compression only when it provides a measured net benefit.

A serialization failure must not result in an unhandled application crash.

### 10.5 Redis Availability

Redis must be treated as an external dependency.

- Set connection and command timeouts.
- Use bounded retries.
- Prevent retry storms.
- Apply circuit breaking where suitable.
- Expose cache availability and latency metrics.
- Define behavior for Redis failover and restart.
- Ensure backend services can safely degrade when cache reads fail.

For ordinary cache reads, a Redis failure should generally fall back to the authoritative data source, subject to rate protection and capacity limits.

For rate limiting, locks, and other coordination features, define an explicit safe failure policy for each operation. Never silently treat an unavailable coordination mechanism as a successful authorization or correctness check.

## 11. Backend Cache Strategy

Backend caching should be implemented through shared, reusable infrastructure rather than scattered Redis calls.

### 11.1 Cache Abstraction

Provide a small, typed cache interface with operations appropriate to the system.

Illustrative contract:

```ts
interface CacheService {
  get<T>(
    key: string,
    schema: ZodSchema<T>
  ): Promise<T | null>;

  set<T>(
    key: string,
    value: T,
    ttlSeconds: number
  ): Promise<void>;

  delete(key: string): Promise<void>;

  deleteMany(keys: string[]): Promise<void>;
}
```

The exact implementation must match KAMPYN's backend language and package boundaries.

Requirements:

- Centralize key construction where possible.
- Standardize serialization and error handling.
- Require explicit TTLs.
- Keep domain-specific cache policy outside the low-level Redis adapter.
- Avoid leaking Redis-specific implementation details into unrelated domain services.
- Support metrics and tracing.
- Make cache behavior replaceable and testable.

Do not build a generic caching framework with features that no current workload requires.

### 11.2 Repository Integration

Repositories should own authoritative data access and query optimization.

Services should coordinate cache policy and business operations.

Avoid placing hidden cache behavior in repository methods unless that behavior is explicitly part of the repository contract and is consistently applied.

Do not allow a cache to bypass tenant filters, repository constraints, or domain validation.

### 11.3 Response Caching

HTTP response caching may be used for explicitly cacheable responses.

Responses involving user-specific information, sensitive information, or permission-dependent data must not be publicly cached.

Set HTTP cache directives intentionally, such as:

```http
Cache-Control: private, no-store
```

for sensitive responses where storage is not permitted, or an appropriate private caching policy for eligible user-specific responses.

Public cache directives may only be used after verifying that the response contains no tenant-private, user-private, or authorization-sensitive information.

Do not rely on `Vary` headers alone to provide tenant or authorization isolation.

## 12. Next.js and Frontend Caching

Frontend caching must distinguish server-rendered data, client server-state, local UI state, and browser-managed assets.

### 12.1 Next.js

- Prefer Server Components for server-side data retrieval when appropriate.
- Use Next.js caching only where cache lifetime and invalidation are understood.
- Set explicit cache and revalidation behavior for data fetching.
- Avoid caching personalized responses in shared server caches.
- Ensure tenant and user context are reflected in cache boundaries.
- Revalidate affected content after mutations where supported.
- Avoid duplicating the backend's authoritative business logic in frontend cache handlers.
- Review the impact of static generation and revalidation on tenant-specific pages.

Next.js server caching must never cause data from one authenticated user or tenant to be rendered for another.

### 12.2 TanStack Query

TanStack Query is the primary client-side server-state cache.

It should manage:

- API response caching.
- Query freshness and garbage collection.
- Request deduplication.
- Mutation state.
- Cache invalidation.
- Optimistic updates where safe.
- Background refetching.
- Pagination and infinite queries.
- Realtime-driven cache synchronization.

Every query key must include the tenant and identity scope when those dimensions affect the response.

Example:

```ts
const vendorKeys = {
  all: (tenantId: string) =>
    ["tenant", tenantId, "vendors"] as const,

  detail: (
    tenantId: string,
    vendorId: string
  ) =>
    [
      "tenant",
      tenantId,
      "vendors",
      "detail",
      vendorId,
    ] as const,
};
```

Set `staleTime` and `gcTime` according to the data's volatility and user experience requirements. Do not apply one global duration to every query.

### 12.3 Zustand

Zustand is intended for client-side UI and application state, not as a replacement for TanStack Query.

Appropriate examples include:

- Active modal state.
- Temporary filters.
- Navigation preferences.
- Local multi-step form state.
- Short-lived interaction state.

Avoid duplicating server-owned records in Zustand when they are already managed by TanStack Query.

Persisted Zustand state must be treated as untrusted, versioned where necessary, and cleared or scoped when authentication or tenant context changes.

### 12.4 Authentication and Tenant Changes

When a user signs out, changes account, or switches tenants:

- Clear or isolate relevant TanStack Query entries.
- Reset tenant-scoped UI state.
- Remove sensitive persisted browser state.
- Cancel obsolete in-flight requests where possible.
- Prevent previous identity data from being rendered during transitions.
- Revalidate data under the new authenticated context.

A query-key change alone may not be sufficient if old sensitive data remains accessible through other client state or persistence mechanisms.

## 13. Domain-Specific Cache Policies

KAMPYN handles workflows with different consistency requirements. Cache policy must be domain-aware.

### 13.1 Food Ordering

Potentially cacheable:

- Public vendor details.
- Menu listings.
- Item descriptions.
- Category data.
- Non-critical display aggregates.

Must be validated against authoritative data before finalizing an order:

- Item availability.
- Current item price.
- Applicable taxes and charges.
- Order eligibility.
- Inventory or stock constraints.
- Payment state.

A cached menu may be displayed for responsiveness, but the backend must revalidate all relevant conditions when creating or confirming an order.

### 13.2 Inventory

Inventory availability is time-sensitive and may change due to concurrent orders or administrative actions.

- Cache inventory summaries only when bounded staleness is acceptable.
- Keep cache TTLs short.
- Revalidate stock against the authoritative inventory state during reservation or order confirmation.
- Use transactional updates, atomic operations, or other appropriate concurrency controls.
- Invalidate or refresh affected entries after inventory events.

A cache hit must never authorize overselling.

### 13.3 Bookings

Booking availability may be cached briefly to improve browsing performance.

However:

- The final booking operation must verify availability authoritatively.
- Prevent duplicate bookings using database constraints and appropriate concurrency control.
- Invalidate relevant availability entries after booking creation, modification, or cancellation.
- Do not guarantee availability based on a cached result.
- Clearly distinguish indicative availability from confirmed booking status.

### 13.4 Orders and Payments

- Cache order summaries only when suitable for the expected staleness.
- Revalidate critical status transitions against authoritative records.
- Do not use cache entries as proof of successful payment.
- Do not treat a cached payment callback or client response as authoritative confirmation.
- Use idempotency and transaction-safe processing for payment operations.
- Invalidate relevant order views after state transitions.

### 13.5 Search

OpenSearch is the search-oriented derived index, while Redis may cache frequently repeated search results when beneficial.

- Cache only normalized, repeatable queries.
- Include tenant scope, filters, sorting, and pagination context in keys.
- Use short TTLs for dynamic catalog results.
- Invalidate or version relevant entries when indexed data changes.
- Account for OpenSearch indexing delay.
- Do not treat search results as authoritative for business transactions.

Avoid caching high-cardinality, low-reuse searches without measured benefit.

### 13.6 Library, Shuttle, and Shared Facilities

Availability for library spaces, shuttle schedules, washing machines, and shared facilities can change rapidly.

- Cache static descriptions and configuration separately from live availability.
- Use short TTLs for availability snapshots.
- Revalidate at the time of reservation or allocation.
- Invalidate affected views after state-changing operations.
- Ensure occupancy and reservation constraints are enforced by authoritative services.

### 13.7 Dashboards and Analytics

Dashboards often involve expensive aggregations.

- Prefer precomputed aggregates or materialized views when appropriate.
- Cache aggregates with a documented freshness target.
- Separate tenant-specific and role-specific aggregates.
- Invalidate or refresh aggregates after relevant events when feasible.
- Clearly define whether metrics are real-time, near-real-time, or delayed.
- Avoid recomputing expensive analytics on every dashboard request.

Analytics caches must not expose records outside the requesting user's permitted scope.

### 13.8 Community and Chat

- Cache stable community metadata and suitable public configuration.
- Keep live message delivery on the designated realtime mechanism.
- Use bounded caches for recent-message history where justified.
- Apply membership and access checks before returning cached community content.
- Invalidate or scope caches when membership, moderation, or visibility changes.
- Avoid using a cache as the sole persistent store for messages or moderation records.

## 14. Cache Consistency Models

Each cached dataset must document its expected consistency model.

| Model | Meaning | Suitable examples |
|---|---|---|
| Strong consistency | Reads reflect committed authoritative state | Final booking or inventory validation |
| Read-after-write | A user sees their successful mutation on subsequent reads | User profile or order updates |
| Eventual consistency | Cached or derived data converges after asynchronous updates | Search indexes and selected aggregates |
| Bounded staleness | Data may be stale within a defined time window | Public menus and non-critical summaries |
| Best-effort | Cache may be missing or stale without affecting correctness | Non-critical display enhancements |

### 14.1 Consistency Requirements

- Document the consistency model for every important cached domain.
- Identify which operations require authoritative reads.
- Define acceptable stale windows.
- Ensure UI language does not imply confirmation when only cached information is available.
- Make state transitions authoritative at the backend.
- Treat cache invalidation as asynchronous where it is asynchronous.

A cache policy must not imply stronger consistency than the underlying implementation provides.

## 15. Multi-Tenant and Self-Hosted Caching

KAMPYN must support multi-tenant SaaS deployments and universities operating self-hosted installations.

### 15.1 SaaS Deployments

- Tenant-specific keys must include validated tenant scope.
- Shared cache infrastructure must enforce strict namespace isolation.
- Avoid tenant-derived unbounded key cardinality.
- Monitor memory and traffic distribution by tenant.
- Consider noisy-neighbor effects when large tenants create high cache load.
- Define tenant-aware invalidation and operational recovery procedures.

Where required by contractual, security, or operational constraints, tenants may use isolated cache deployments or namespaces with independent resource limits.

### 15.2 Self-Hosted Deployments

Self-hosted installations must be able to configure:

- Redis connection settings.
- Cache capacity and eviction policy.
- TTL policies where exposed as supported configuration.
- Cache feature toggles.
- Monitoring and health checks.
- Cache warm-up and recovery procedures.

Avoid hardcoding SaaS-only infrastructure assumptions into application cache logic.

The self-hosted installation must remain functionally correct when the optional cache layer is disabled, except for explicitly documented coordination features that require their own supported alternative.

## 16. Cache Warm-Up

Cache warm-up may be used when predictable high-traffic periods or expensive initialization justify it.

Potential candidates include:

- Tenant configuration.
- Public service categories.
- Frequently accessed vendor metadata.
- Common static reference data.

Requirements:

- Warm only high-value entries.
- Bound concurrency and request rates.
- Avoid loading entire datasets unnecessarily.
- Use the authoritative datastore as the source.
- Ensure warm-up is idempotent.
- Avoid making application startup depend on optional cache warm-up.
- Monitor warm-up failures and duration.

Do not prepopulate caches with large amounts of data based solely on speculative usage.

## 17. Cache Failure and Degradation

Cache failure must be handled according to the role of the cache.

### 17.1 Ordinary Read Cache Failure

For non-critical read caching:

1. Record the cache failure.
2. Apply bounded retry or circuit-breaker behavior.
3. Fall back to the authoritative datastore if safe and capacity allows.
4. Return the authoritative result.
5. Skip or defer cache population if Redis remains unavailable.

### 17.2 Database Load Protection

If Redis is unavailable, uncontrolled fallback traffic may overwhelm the database.

Use appropriate protection mechanisms:

- Request concurrency limits.
- Rate limiting.
- Single-flight request coalescing.
- Bounded retries.
- Query timeouts.
- Circuit breakers.
- Graceful degradation for expensive non-critical features.

Fallback behavior must be load-tested, not assumed to be safe.

### 17.3 Coordination Failure

Coordination features require explicit fail-open or fail-closed decisions based on risk.

Examples:

- A non-critical cache refresh lock may safely be bypassed with bounded duplicate work.
- A security-sensitive rate limit may require a conservative fallback.
- A lock protecting a critical operation must not be treated as proof of exclusive ownership when the coordination system is unavailable.

Where safe coordination cannot be guaranteed, reject or defer the affected operation rather than silently bypassing its correctness requirements.

## 18. Cache Security

Caching must not weaken KAMPYN's security boundaries.

- Never cache passwords, raw credentials, private keys, or payment authentication secrets.
- Avoid storing sensitive personal data unless strictly required and protected by an approved design.
- Enforce tenant isolation in keys and access paths.
- Apply authorization before serving sensitive cached data.
- Prevent cache poisoning through strict input validation and canonical key construction.
- Restrict Redis network access and authenticate service connections.
- Use encryption in transit where supported and required by the deployment.
- Limit access to cache administration and diagnostic interfaces.
- Redact sensitive values from logs and traces.
- Define retention and deletion policies for cached personal data.
- Ensure account deactivation and membership changes invalidate or block access to affected cached data.

Cache invalidation is not a substitute for authorization. Sensitive data must be protected even if an invalidation event is delayed or lost.

## 19. Performance Optimization

Caching should be introduced after identifying the actual bottleneck.

### 19.1 Optimize Before Caching

Consider the following before adding a cache:

- Database query optimization.
- Appropriate database indexes.
- Efficient algorithms and data structures.
- Pagination and bounded result sizes.
- Avoiding N+1 queries.
- Reducing unnecessary serialization.
- Eliminating redundant API requests.
- Batch retrieval.
- Streaming large datasets.
- Precomputing expensive aggregates.

Caching a poorly designed query may hide the problem while creating additional invalidation and operational complexity.

### 19.2 Payload Size

- Cache only fields required by the consuming workload.
- Avoid caching complete database records when only a summary is needed.
- Set maximum cache value sizes.
- Avoid large collections as individual cache values.
- Consider compression only after benchmarking CPU, memory, and latency trade-offs.
- Use pagination or partitioned keys for large result sets.

### 19.3 Cache Hit Ratio

Track cache hit ratio by cache domain and key category.

A low hit ratio may indicate:

- High-cardinality queries.
- Incorrect TTL values.
- Poor key design.
- Excessive invalidation.
- Low reuse.
- A cache that is not appropriate for the workload.

Do not optimize solely for a high hit ratio. Correctness, latency, database load, memory cost, and operational complexity must be considered together.

## 20. Observability

Every production cache must be observable.

### 20.1 Required Metrics

Track, where applicable:

- Cache hits.
- Cache misses.
- Cache hit ratio.
- Cache read and write latency.
- Cache errors and timeouts.
- Cache connection failures.
- Key eviction count.
- Key expiration count.
- Memory usage and fragmentation.
- Cache item count.
- Large-key occurrences.
- Invalidation event volume.
- Invalidation delay.
- Invalidation failures.
- Lock acquisition latency and failures.
- Stampede prevention activity.
- Cache fallback requests.
- Authoritative datastore load during cache degradation.

### 20.2 Logging

Logs should capture:

- Cache operation category.
- Cache namespace or domain.
- Operation outcome.
- Latency.
- Error category.
- Invalidation event type.
- Relevant correlation identifiers.

Do not log full cache values, secrets, raw personal data, or unredacted sensitive keys.

### 20.3 Tracing

Distributed traces should identify cache operations and their relationship to database calls.

Traces should help distinguish:

- Cache hit latency.
- Cache miss latency.
- Authoritative datastore latency.
- Serialization overhead.
- Invalidation delay.
- Fallback behavior.

Use low-cardinality labels for metrics and tracing. Do not use user IDs, tenant IDs, or cache keys as unbounded metric labels.

### 20.4 Alerting

Establish alerts for conditions such as:

- Sustained Redis unavailability.
- Significant cache latency increase.
- Unexpected memory pressure.
- High eviction rates affecting critical workloads.
- Repeated invalidation failures.
- Excessive fallback database traffic.
- Unusual key growth.
- Persistent cache stampedes.

Alert thresholds must be derived from production baselines and service-level objectives.

## 21. Testing Requirements

Caching behavior must be tested as part of the feature, not treated as infrastructure-only behavior.

### 21.1 Unit Tests

Test:

- Key generation and normalization.
- Tenant and user scoping.
- TTL configuration.
- Serialization and deserialization.
- Cache hit and miss paths.
- Malformed cache values.
- Invalidation key selection.
- Cache error handling.
- Negative caching behavior.
- Versioned cache compatibility.

### 21.2 Integration Tests

Test:

- Redis connectivity and reconnection.
- Cache-aside behavior.
- Expiration.
- Concurrent cache misses.
- Invalidation after successful writes.
- Outbox-driven invalidation.
- Idempotent event processing.
- Redis unavailability and recovery.
- Tenant isolation across cache hits and misses.
- Authorization changes and stale cache prevention.

### 21.3 Performance Tests

Test:

- Cold-cache performance.
- Warm-cache performance.
- High cache-hit workloads.
- High cache-miss workloads.
- Concurrent expiration.
- Cache stampedes.
- Large values.
- High-cardinality keys.
- Redis memory pressure.
- Database load during cache failure.
- Tenant traffic imbalance.
- Cache warm-up under load.

### 21.4 Correctness Tests

For critical domains, prove that stale cache entries cannot cause invalid business operations.

Test concurrent scenarios involving:

- Order placement.
- Inventory reservation.
- Booking creation and cancellation.
- Payment status changes.
- Authorization and membership changes.

Database constraints, transactions, and domain invariants must remain effective regardless of cache state.

## 22. Anti-Patterns

The following practices are prohibited unless an explicit architectural exception is approved:

- Treating cached data as the source of truth.
- Using cache hits to bypass authorization.
- Sharing tenant-specific entries across tenants.
- Caching all API responses by default.
- Assigning a universal TTL to unrelated data.
- Relying only on TTL for critical invalidation.
- Performing cache-first writes for transactional business operations without a justified consistency design.
- Using Redis locks as the only protection against duplicate business operations.
- Storing unlimited collections or unbounded keys.
- Caching large datasets without measuring memory impact.
- Repeating cache logic across multiple services.
- Using wildcard key deletion as a routine invalidation mechanism on high-volume shared Redis deployments.
- Retrying cache operations without bounded limits.
- Allowing cache failure to trigger uncontrolled database traffic.
- Duplicating TanStack Query server-state in Zustand without a documented requirement.
- Caching sensitive responses in shared CDN or server caches.
- Assuming OpenSearch results are transactionally current.
- Ignoring cache invalidation failures.
- Introducing caching without metrics, tests, and a defined recovery strategy.

## 23. Implementation Checklist

### Architecture
- [ ] Cache use is justified by a measured workload.
- [ ] Cache owner and source of truth are documented.
- [ ] The selected caching layer is appropriate.
- [ ] Consistency requirements are defined.
- [ ] Failure and recovery behavior are specified.

### Key Design
- [ ] Keys are deterministic and versioned where needed.
- [ ] Tenant scope is included and validated.
- [ ] User and authorization scope is handled where relevant.
- [ ] Query filters and sorting parameters are represented.
- [ ] Sensitive data is excluded from keys.
- [ ] Key cardinality and size are bounded.

### Lifecycle
- [ ] Every entry has a defined TTL or documented invalidation-only policy.
- [ ] Expiration is compatible with the stale-data requirement.
- [ ] Invalidation triggers are identified.
- [ ] Write and invalidation reliability is addressed.
- [ ] Negative caching and stampede controls are considered.

### Security
- [ ] Cache access does not bypass authentication or authorization.
- [ ] Tenant isolation is tested.
- [ ] Sensitive information is protected.
- [ ] Logging and monitoring redact sensitive values.
- [ ] User and tenant lifecycle changes are handled.

### Performance
- [ ] Cache hit and miss paths are measured.
- [ ] Database fallback load is considered.
- [ ] Payload sizes and memory use are bounded.
- [ ] Concurrency and stampede risks are addressed.
- [ ] Cache value is compared with non-cache optimizations.

### Reliability
- [ ] Redis timeouts and bounded retries are configured.
- [ ] Degraded operation is defined.
- [ ] Invalidation is observable and recoverable.
- [ ] Redis memory and eviction behavior are monitored.
- [ ] Critical operations remain correct during cache failures.

### Testing and Operations
- [ ] Unit and integration tests cover cache behavior.
- [ ] Critical consistency scenarios are tested.
- [ ] Performance tests cover cold and warm caches.
- [ ] Metrics, logs, and traces are available.
- [ ] Operational recovery and cache flush procedures are documented.

## 24. Definition of Done

A caching implementation is complete when:

- Its purpose, owner, source of truth, and consistency model are documented.
- Cache keys are deterministic and respect tenant and authorization boundaries.
- TTL, invalidation, and eviction behavior are explicitly defined.
- Cache failure and recovery paths are implemented.
- Critical business operations remain correct without cached data.
- Concurrency and stampede risks are appropriately addressed.
- Security, privacy, and data-retention requirements are satisfied.
- Unit, integration, and relevant performance tests pass.
- Observability is available for hits, misses, latency, errors, invalidation, and resource use.
- Documentation reflects the actual implementation and operational requirements.

**Final principle:** Cache to reduce unnecessary work, not to conceal inefficient architecture or weaken correctness. KAMPYN must remain secure and functionally correct with a cold, stale, or unavailable cache.
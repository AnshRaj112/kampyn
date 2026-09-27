# KAMPYN Caching Architecture

## 1. Purpose

This document defines the caching architecture for KAMPYN.

Caching is an optimization layer, not a substitute for authoritative data storage.

A cache must improve:

- Latency.
- Throughput.
- Database load.
- External service load.
- Search performance.
- User experience.

Caching must never silently compromise:

- Data correctness.
- Authorization.
- Tenant isolation.
- Consistency requirements.
- Availability guarantees.
- Resource ownership.

Every cache must have a clearly defined owner, lifecycle, consistency model, and failure behavior.

---

# 2. Core Principle

The fundamental caching model is:

```text id="7v3nq8"
Authoritative Source
        ↓
      Cache
        ↓
     Consumer
```

The authoritative source remains responsible for correctness.

Examples:

```text id="q8m2x5"
PostgreSQL / MongoDB
        ↓
      Redis
```

and:

```text id="h4k7p1"
Primary Database
        ↓
   Search Projection
        ↓
    OpenSearch
```

OpenSearch is a search projection, not a traditional cache, but it follows the same principle of deriving data from an authoritative source.

---

# 3. When to Cache

Caching should be introduced when there is a measurable or well-understood benefit.

Good candidates include:

- Frequently read data.
- Expensive computations.
- Expensive database queries.
- External API responses.
- Session-related state where appropriate.
- Rate-limiting state.
- Short-lived lookup data.
- Search suggestions.
- Configuration that changes infrequently.

Do not cache simply because Redis is available.

The first question should be:

```text id="m8c4r1"
What expensive work does this cache eliminate?
```

---

# 4. Cacheability Assessment

Before introducing a cache, determine:

1. What is being cached?
2. What is the authoritative source?
3. How frequently is it read?
4. How frequently does it change?
5. How stale can it safely be?
6. Who owns the cache entry?
7. What invalidates it?
8. What happens when the cache is unavailable?
9. Can the cached value contain tenant-specific data?
10. Can the cached value contain sensitive data?
11. What is the memory/storage cost?
12. How will the cache be monitored?

If these questions cannot be answered, the cache design is incomplete.

---

# 5. Cache Ownership

Every cache entry must have an identifiable owner.

The owner defines:

- Key format.
- Value schema.
- TTL.
- Invalidation behavior.
- Serialization.
- Failure handling.

Avoid multiple unrelated modules writing to the same cache key.

Prefer:

```text id="c2x7n4"
orders module
    ↓
orders:* keys

inventory module
    ↓
inventory:* keys
```

over:

```text id="x8q1m5"
global:* keys
```

with unclear ownership.

---

# 6. Cache Keys

Cache keys must be:

- Deterministic.
- Collision-resistant.
- Scoped.
- Versionable where necessary.
- Easy to inspect operationally.

Prefer:

```text id="p6r4k2"
kampyn:{tenantId}:foodcourt:{foodCourtId}:menu:{version}
```

over ambiguous keys such as:

```text id="v9m3q1"
menu:123
```

when tenant isolation is required.

---

# 7. Tenant Isolation

Tenant isolation is mandatory for tenant-specific caches.

A cache key must contain enough scope to prevent cross-tenant collisions.

Conceptually:

```text id="a7f2m9"
Tenant A
    ↓
tenant:A:resource:123

Tenant B
    ↓
tenant:B:resource:123
```

Never assume the database's tenant filtering will protect a cached response.

Cache isolation must be independently correct.

---

# 8. Authorization and Cache

Caching must never bypass authorization.

A cached value must only be served when the requester is authorized to receive it.

If authorization differs between users, the cache key or cache architecture must account for the relevant authorization scope.

Do not use:

```text id="h5q9w3"
resource:123
```

for data where two users with different permissions may receive different representations.

Possible approaches include:

- Authorization-independent cache plus filtering.
- User/role/scoped cache keys.
- Caching only public/shared data.
- Performing authorization before cache retrieval.

Choose based on correctness and cache efficiency.

---

# 9. Cache Value Scope

Cache entries should contain the minimum information required.

Avoid caching:

- Large unrelated objects.
- Sensitive information unnecessarily.
- Internal database metadata.
- Credentials.
- Authentication secrets.

If a cached object contains sensitive information, review:

- Encryption.
- Access control.
- TTL.
- Logging.
- Eviction.
- Backup behavior.

---

# 10. Cache-Aside Pattern

Cache-aside is the default pattern for many KAMPYN read workloads.

Flow:

```text id="e8m3q7"
Request
   ↓
Check Cache
   ↓
 ┌───────────────┐
 │ Cache Hit?    │
 └───────┬───────┘
         │
     Yes │ No
         ↓
      Return
         │
         └──────────────┐
                        ↓
                 Read Source
                        ↓
                  Update Cache
                        ↓
                     Return
```

Typical implementation:

```text id="k4n8p2"
1. Read cache
2. On hit → return
3. On miss → read authoritative source
4. Store result in cache
5. Return result
```

Cache misses must remain correct.

---

# 11. Write-Through Caching

Write-through caching may be used when both source and cache must be updated as part of the write path.

Conceptually:

```text id="w6r2m9"
Write Request
      ↓
Cache
      ↓
Authoritative Store
```

Use only when the consistency and failure semantics are clearly understood.

Do not assume writing the cache and database automatically creates atomic consistency.

---

# 12. Write-Behind Caching

Write-behind caching introduces additional durability and consistency complexity.

Use only when there is a strong architectural reason.

If used, define:

- Durable queueing.
- Failure handling.
- Ordering.
- Retry.
- Data loss behavior.
- Recovery.
- Reconciliation.

Do not use write-behind simply to make writes appear faster.

---

# 13. Read-Through Caching

Read-through caching may be used when a dedicated caching abstraction owns source retrieval.

The abstraction must still clearly define:

- Source of truth.
- Failure behavior.
- TTL.
- Invalidation.
- Authorization.
- Tenant scope.

Do not hide important consistency behavior behind an opaque generic cache abstraction.

---

# 14. TTL

Every temporary cache entry should have an intentional TTL unless there is a documented reason for indefinite retention.

TTL should reflect:

- Data volatility.
- Acceptable staleness.
- Cost of recomputation.
- Memory pressure.
- Business requirements.

Do not choose TTLs arbitrarily.

For example:

```text id="q4m8c2"
Highly volatile data
→ short TTL

Slow-changing configuration
→ longer TTL
```

The exact value must be determined by the workload.

---

# 15. Cache Invalidation

Cache invalidation must be explicit.

Possible strategies:

- TTL expiration.
- Delete on mutation.
- Update on mutation.
- Event-driven invalidation.
- Versioned keys.
- Full namespace invalidation where justified.

Do not assume TTL alone is sufficient for data that must become consistent immediately after a mutation.

---

# 16. Mutation Invalidation

For mutable data:

```text id="m3p7x9"
Write
  ↓
Authoritative Store
  ↓
Invalidate / Update Cache
```

The ordering must be chosen carefully.

A common safe pattern is:

```text id="r8c4n1"
Update Source
    ↓
Commit
    ↓
Invalidate Cache
```

This prevents a failed database write from unnecessarily invalidating valid cached state.

---

# 17. Cache Invalidation Race Conditions

Cache invalidation can race with concurrent reads.

Example:

```text id="v7m2q5"
Request A reads old value
Request B updates database
Request B invalidates cache
Request A writes old value into cache
```

This can reintroduce stale data.

Mitigations may include:

- Versioned values.
- Cache timestamps/version numbers.
- Delete-after-write strategies.
- Write-through patterns.
- Event sequencing.
- Short TTLs.
- Single-writer ownership.

The appropriate mechanism depends on consistency requirements.

---

# 18. Cache Stampede

A cache stampede occurs when many requests simultaneously miss the same expensive key.

Example:

```text id="x2n8k4"
Cache expires
    ↓
10,000 requests miss
    ↓
10,000 database queries
```

For expensive/high-traffic keys, consider:

- Request coalescing.
- Single-flight.
- Distributed locking where justified.
- Early refresh.
- Stale-while-revalidate.
- Randomized TTL jitter.

Do not add distributed locks automatically.

Use the simplest mechanism that protects the underlying dependency.

---

# 19. TTL Jitter

Large numbers of keys with identical TTLs can expire simultaneously.

For large workloads, consider adding controlled jitter to TTLs.

Conceptually:

```text id="f6q2m8"
Base TTL
    +
Small random variation
    ↓
Distributed expiration
```

Use jitter only where synchronized expiration is an actual concern.

---

# 20. Negative Caching

Negative results may be cached where repeated misses are expensive.

Examples:

```text id="p5m9c3"
User not found
Product unavailable
Search returned no results
```

Negative caching must use short and intentional TTLs.

Do not allow temporary absence to become a long-lived false state.

---

# 21. Stale Data

Every cached resource must have a defined staleness tolerance.

Ask:

```text id="m7q2v4"
How wrong can this data safely be?
```

Examples:

- Public metadata may tolerate moderate staleness.
- Menu information may tolerate short staleness depending on business requirements.
- Inventory availability may require near-real-time correctness.
- Payment state should generally rely on authoritative state.

Never cache business-critical state without explicitly defining its consistency requirements.

---

# 22. Business-Critical Data

Do not use stale cache data as the authoritative basis for:

- Payment authorization.
- Inventory deduction.
- Reservation allocation.
- Booking confirmation.
- Permission decisions.
- Security-sensitive account state.

A cache may accelerate reads, but the authoritative system must enforce the final invariant.

---

# 23. Redis Architecture

Redis may serve as the primary shared cache for KAMPYN.

Redis usage must define:

- Connection configuration.
- Authentication.
- TLS where required.
- Namespace.
- Key format.
- TTL.
- Serialization.
- Failure behavior.
- Memory policy.

Redis should remain an infrastructure dependency rather than being directly accessed by arbitrary application modules.

---

# 24. Redis Client Ownership

Initialize Redis clients through the application's infrastructure layer.

Avoid creating Redis clients inside:

- HTTP handlers.
- Domain objects.
- Individual repository methods.

Prefer:

```text id="u8k3p6"
Application Startup
    ↓
Redis Client
    ↓
Cache Adapter
    ↓
Application / Infrastructure
```

Connection lifecycle must be explicit.

---

# 25. Redis Serialization

Cache serialization must be stable and versionable where necessary.

Consider:

- Schema changes.
- Backward compatibility during deployments.
- Null values.
- Optional fields.
- Numeric precision.
- Time formats.

Do not serialize arbitrary application objects without understanding their evolution behavior.

---

# 26. Cache Versioning

When cached value structures change incompatibly, use versioned keys.

Example:

```text id="n4p7c2"
menu:v1:{tenant}:{id}
menu:v2:{tenant}:{id}
```

This can prevent new application versions from attempting to deserialize old cache values.

Invalidate obsolete versions when safe.

---

# 27. Cache Failures

Cache failure must have an explicit behavior.

For ordinary read caching:

```text id="f2m8q4"
Redis unavailable
      ↓
Read authoritative source
```

where performance allows it.

For infrastructure uses where Redis is required for correctness, such as certain distributed coordination mechanisms, the system must explicitly define failure behavior.

Do not silently treat every Redis failure as a cache miss when Redis is being used for something more than caching.

---

# 28. Cache Availability

Do not make ordinary cache availability a hard requirement for serving data when the authoritative source remains available.

Prefer graceful degradation:

```text id="k7r3m9"
Cache available
    ↓
Fast path

Cache unavailable
    ↓
Authoritative path
```

However, protect the authoritative datastore from overload during widespread cache failure.

Use:

- Rate limiting.
- Request coalescing.
- Backpressure.
- Bounded concurrency.
- Circuit breakers where appropriate.

---

# 29. Cache Failure Cascades

A cache outage can cause a sudden increase in database traffic.

Review:

```text id="c5v8n2"
Cache Failure
    ↓
Cache Misses
    ↓
Database Load
    ↓
Database Saturation
    ↓
API Latency
    ↓
Retries
    ↓
Further Load
```

Retry behavior must not amplify cache failures.

---

# 30. Cache Warmup

Cache warmup may be appropriate for predictable high-value data.

Examples:

- Frequently accessed configuration.
- Popular campus metadata.
- Frequently searched public information.

Warmup must be:

- Bounded.
- Repeatable.
- Failure-tolerant.

Do not preload enormous datasets into cache without evidence that they are valuable.

---

# 31. Lazy Loading

Prefer lazy population for most caches.

Do not populate the entire database into Redis at startup unless the dataset is intentionally small and the architecture requires it.

Lazy loading keeps memory usage proportional to actual access patterns.

---

# 32. Cache Eviction

Cache eviction policies must match workload characteristics.

Consider:

- Memory limits.
- Entry size.
- Access frequency.
- TTL.
- Working set.

Do not assume an eviction policy can compensate for an incorrectly sized cache.

Monitor eviction behavior.

---

# 33. Memory Management

Cache memory is finite.

Avoid storing:

- Massive objects.
- Redundant representations.
- Unbounded lists.
- Entire datasets unnecessarily.

For large results, consider:

- Pagination.
- Compression where appropriate.
- Smaller representations.
- Object storage.
- Database queries.

Do not use Redis as a dumping ground for application state.

---

# 34. Cache Entry Size

Cache entries should remain reasonably bounded.

Large cache values can cause:

- Memory pressure.
- Network overhead.
- Serialization cost.
- Slow Redis operations.
- Increased eviction.

If a value becomes very large, reconsider whether it belongs in Redis.

---

# 35. Cache Compression

Compression may be useful for large values when network and memory costs justify it.

However, consider:

- CPU cost.
- Latency.
- Serialization overhead.
- Compression ratio.

Do not compress tiny values unnecessarily.

Measure before introducing compression.

---

# 36. Local vs Distributed Cache

A local in-process cache may be appropriate for:

- Extremely hot immutable data.
- Small configuration.
- Expensive computations.

Redis or another distributed cache is appropriate when:

- Multiple application instances must share state.
- Cache consistency across instances matters.
- Horizontal scaling is required.

Do not introduce both local and distributed caching without clearly defining their interaction.

---

# 37. Multi-Level Caching

If multiple cache layers exist:

```text id="j5n8q2"
Browser
   ↓
CDN
   ↓
Application Memory
   ↓
Redis
   ↓
Database
```

each layer must define:

- TTL.
- Invalidation.
- Ownership.
- Consistency.
- Security.

Multiple caching layers increase invalidation complexity.

Do not add another cache layer without a measurable reason.

---

# 38. HTTP Caching

For appropriate public or semi-public responses, HTTP caching may reduce application load.

Use:

- `Cache-Control`.
- `ETag`.
- `Last-Modified`.
- Conditional requests.

Never cache private responses publicly.

Sensitive or user-specific responses must have appropriate cache directives.

---

# 39. CDN Caching

CDN caching may be appropriate for:

- Public assets.
- Images.
- Static files.
- Public marketing content.

Do not cache authenticated or tenant-sensitive responses through a shared CDN unless the cache policy explicitly guarantees isolation.

Cache invalidation must be defined for mutable assets.

---

# 40. Frontend Caching

Frontend caching should follow the frontend architecture.

Use TanStack Query for server-state caching.

Use Zustand for client-owned state, not as a replacement for server-state caching.

Frontend caches must account for:

- Tenant context.
- Authentication.
- Query parameters.
- Resource identity.
- Staleness.
- Mutation invalidation.

Do not maintain independent caches of the same server resource without a clear reason.

---

# 41. Search Caching

Search results may be cached when:

- Queries repeat frequently.
- Results can tolerate short staleness.
- Query complexity is high.

Search cache keys must include all result-affecting parameters:

```text id="n7q2m4"
tenant
query
filters
sort
page/cursor
locale
permission scope where applicable
```

Never serve one user's restricted search results to another user.

---

# 42. Cache Key Cardinality

Avoid cache keys with uncontrolled cardinality.

Examples of risky dimensions:

- Arbitrary user input.
- Full query strings.
- High-cardinality timestamps.
- Random identifiers.

High-cardinality keys can consume cache memory without producing meaningful reuse.

Bound or normalize cache dimensions where appropriate.

---

# 43. Cache Invalidation Through Events

For distributed systems, event-driven invalidation may be useful.

Example:

```text id="x6m4p8"
Order Updated
    ↓
Domain Event
    ↓
Cache Invalidation Consumer
    ↓
Invalidate order-related keys
```

Consumers must handle:

- Duplicate events.
- Delayed events.
- Failed processing.
- Out-of-order events.

Do not make correctness depend on an event being delivered exactly once unless the infrastructure explicitly guarantees it.

---

# 44. Cache Consistency Models

Every important cache should have an explicit consistency model.

Possible models include:

```text id="q3v7m2"
Strong / effectively immediate
Eventual
Best effort
Stale-while-revalidate
```

The chosen model must match the business requirement.

Do not describe a cache as "real-time" without defining what that means operationally.

---

# 45. Cache Observability

Monitor cache behavior.

Useful metrics include:

- Hit rate.
- Miss rate.
- Eviction rate.
- Entry count.
- Memory usage.
- Operation latency.
- Error rate.
- Connection count.
- Hot keys.
- Stampede events.

A cache that cannot be observed is difficult to operate safely.

---

# 46. Cache Hit Rate

Hit rate is useful but not sufficient.

A high hit rate does not automatically mean the cache is beneficial.

Also consider:

- Latency reduction.
- Database load reduction.
- Memory cost.
- Cache maintenance complexity.
- Staleness.
- Failure behavior.

Measure actual system-level impact.

---

# 47. Hot Keys

A single extremely popular cache key can become a bottleneck.

Examples:

```text id="m8q2v5"
popular menu
popular food court
popular configuration
```

If hot keys become a problem, consider:

- Replication.
- Local caching.
- Request coalescing.
- Read distribution.
- Data partitioning.

Do not prematurely optimize for theoretical hot-key problems.

---

# 48. Cache Stampede Protection

For particularly expensive keys, single-flight/request coalescing may be preferable to distributed locking.

Conceptually:

```text id="c4n7x2"
100 requests
     ↓
same missing key
     ↓
1 source request
     ↓
99 requests share result
```

This reduces pressure on the authoritative source without introducing unnecessary distributed coordination.

---

# 49. Distributed Locks

Distributed locks must not be treated as ordinary caching.

Use them only when there is a clearly defined coordination requirement.

If using distributed locks, define:

- Lock key.
- Ownership.
- TTL.
- Renewal.
- Failure behavior.
- Deadlock prevention.
- Fencing where required.

Never assume a Redis lock automatically guarantees correctness under all failure modes.

Prefer database-level atomicity when the invariant belongs to the database.

---

# 50. Rate Limiting State

Redis may store rate-limiting counters.

Rate limiting state should define:

- Key scope.
- Window.
- Expiration.
- Atomicity.
- Failure behavior.

Use atomic Redis operations or scripts where necessary to prevent race conditions.

Do not implement rate limiting with non-atomic:

```text id="r6m3q8"
GET
↓
increment in application
↓
SET
```

when concurrent requests can race.

---

# 51. Session State

If Redis stores session state, session architecture must explicitly define:

- Session ownership.
- TTL.
- Revocation.
- Serialization.
- Availability requirements.
- Failover behavior.

Session storage is security-sensitive and must not be treated as ordinary application caching.

---

# 52. Authorization Data

Authorization-related cache entries require stronger consistency guarantees than ordinary performance caches.

Examples:

- Permission assignments.
- Role mappings.
- Tenant membership.
- Administrative privileges.

When permissions change, cached authorization state must not remain valid indefinitely.

Prefer authoritative checks for highly sensitive operations.

---

# 53. Cache Warming After Deployment

Do not automatically warm every cache during deployment.

If warmup is necessary:

- Define the exact keys.
- Bound the workload.
- Avoid overwhelming databases.
- Run asynchronously where appropriate.
- Monitor completion.

Deployment should not fail merely because optional cache warmup fails.

---

# 54. Cache Migration

When changing cache formats:

Prefer:

```text id="w5n2q8"
Versioned keys
        ↓
Gradual adoption
        ↓
Old key expiration
```

rather than requiring a dangerous coordinated cache flush during deployment.

A full cache flush should be considered an operational event, not a routine deployment step.

---

# 55. Cache Invalidation During Deployment

Application versions may temporarily coexist.

Ensure that:

- Old and new versions can handle relevant cache values.
- Cache keys are versioned when necessary.
- Serialization remains compatible during rollout.
- Invalidation remains valid across versions.

Do not assume all application instances update simultaneously.

---

# 56. Cache Security

Protect Redis and other cache infrastructure.

Use:

- Authentication.
- Network isolation.
- TLS where appropriate.
- Least-privilege access.
- Secure configuration.
- Monitoring.

Do not expose Redis directly to the public internet.

Never assume cache data is harmless merely because it is temporary.

---

# 57. Cache Failure Testing

Test cache failure scenarios.

At minimum consider:

- Redis unavailable.
- Connection timeout.
- Slow Redis.
- Cache miss storm.
- Corrupted value.
- Serialization mismatch.
- Expired entry.
- Eviction.
- Network partition.

Verify that ordinary cache failure does not unnecessarily cause system-wide failure.

---

# 58. Cache Testing

Cache tests should cover:

- Hit behavior.
- Miss behavior.
- Population.
- TTL.
- Invalidation.
- Serialization.
- Tenant isolation.
- Authorization scope.
- Concurrent access.
- Failure fallback.
- Version changes.

Do not test only that Redis was called.

Test the resulting application behavior.

---

# 59. Cache Review Checklist

Before completing a caching-related change:

- [ ] Authoritative source is identified.
- [ ] Cache ownership is clear.
- [ ] Cache purpose is explicit.
- [ ] Key structure is deterministic.
- [ ] Tenant isolation is preserved.
- [ ] Authorization is preserved.
- [ ] TTL is intentional.
- [ ] Invalidation strategy is defined.
- [ ] Staleness tolerance is documented.
- [ ] Cache failures have defined behavior.
- [ ] Cache stampede behavior is considered.
- [ ] Memory usage is bounded.
- [ ] Entry size is reasonable.
- [ ] Serialization is stable.
- [ ] Versioning is considered.
- [ ] Database overload during cache failure is considered.
- [ ] Distributed locking is not used unnecessarily.
- [ ] Search caching is authorization-safe.
- [ ] Session/permission caching receives appropriate security treatment.
- [ ] Metrics exist where operationally important.
- [ ] Tests cover hit, miss, invalidation, and failure.
- [ ] Documentation reflects the caching model.

---

# 60. Final Caching Principle

Caching should make the system faster without changing what the system means.

The architecture should remain:

```text id="g8m4q2"
Authoritative Source
        ↓
   Cache / Projection
        ↓
     Consumer
```

with:

```text id="x5r7n1"
Explicit ownership
Explicit TTL
Explicit invalidation
Explicit consistency
Explicit tenant scope
Explicit failure behavior
```

Never introduce a cache without defining what happens when:

```text id="q9m2c6"
The cache is empty.
The cache is stale.
The cache is unavailable.
The cache contains old data.
The cache is corrupted.
The cache is overloaded.
```

The authoritative system must remain capable of restoring correct state.

Optimize for:

```text id="v4k8p3"
Correctness
    ↓
Consistency
    ↓
Isolation
    ↓
Performance
    ↓
Operational simplicity
```

A cache is successful when it reduces expensive work while remaining invisible to the correctness of the system.
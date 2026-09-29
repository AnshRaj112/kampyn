# Redis Engineering Standards

## 1. Purpose

Redis is KAMPYN's in-memory data platform for high-speed caching, ephemeral state, distributed coordination, rate limiting, temporary data, and selected real-time workloads.

Redis is designed to reduce latency, offload frequently accessed data from authoritative databases, and support coordination between distributed application instances.

Redis is **not a primary source of truth for durable business data**. PostgreSQL and MongoDB remain authoritative for their respective domains.

These standards define how Redis data structures, caching, expiration, distributed coordination, tenant isolation, performance, reliability, security, and operational infrastructure must be designed and maintained.

### Core Principles

- Redis is primarily a transient data and coordination system.
- PostgreSQL and MongoDB remain the authoritative sources of business data.
- Every Redis key must have a documented purpose, owner, and lifecycle.
- Every cache must have an explicit expiration and invalidation strategy.
- Redis operations must be atomic where correctness depends on concurrency.
- Redis must not be treated as a transactional substitute for authoritative databases.
- Memory usage must be bounded and monitored.
- Tenant isolation must be enforced for tenant-scoped keys and operations.
- Distributed locks must be used sparingly and safely.
- Redis failure must not cause unrecoverable loss of authoritative business data.
- Redis access must be encapsulated behind domain-oriented services.
- Redis configuration must support both KAMPYN-managed and university self-hosted deployments.

---

## 2. Architectural Position

Redis is a supporting infrastructure component within KAMPYN's polyglot persistence architecture.

### Data Ownership

| System | Responsibility |
|---|---|
| PostgreSQL | Relational and transactional source of truth |
| MongoDB | Justified document-oriented source of truth |
| Redis | Cache, ephemeral state, rate limiting, and distributed coordination |
| OpenSearch | Derived search, filtering, facets, and discovery |
| Object Storage | Images, documents, exports, and binary assets |
| Backend Services | Business logic, authorization, cache policies, and coordination |

### Redis Responsibilities

Redis MAY be used for:

- Frequently accessed data caching.
- Session storage where the approved authentication architecture permits it.
- API rate limiting.
- Temporary verification challenges and short-lived tokens.
- Distributed coordination and carefully designed locks.
- Idempotency coordination for transient workflows.
- Short-lived request deduplication.
- Temporary counters and throttling.
- Online presence and ephemeral user activity.
- Real-time application state where loss is acceptable or recoverable.
- Background-job coordination when a dedicated queue or broker is not required.
- Short-lived search-result caching.
- Temporary OTP or verification attempt tracking.
- WebSocket connection metadata and transient routing information.

### Redis Must Not Be the Sole Authority For

- Orders and order history.
- Payment status or financial balances.
- Inventory ownership or final stock quantities.
- Booking confirmations and resource allocation.
- Durable user account records.
- Authoritative role and permission definitions.
- Complaints and other records requiring durable retention.
- Permanent audit logs.
- Durable business events.
- Any record whose loss would violate a business, legal, or compliance requirement.

If Redis is used to accelerate a critical workflow, the authoritative state and recovery strategy must remain in the appropriate durable datastore.

---

## 3. Redis Deployment Strategy

Redis deployment must be selected according to workload, availability requirements, data-loss tolerance, and operational complexity.

### Deployment Options

| Deployment | Suitable Use |
|---|---|
| Standalone Redis | Local development, testing, and non-critical workloads |
| Redis with replication | Improved read availability and failover support |
| Redis Sentinel | Monitoring and automated failover for supported non-clustered deployments |
| Redis Cluster | Horizontal data distribution and high-volume workloads |
| Managed Redis | Production deployments where managed operations reduce infrastructure overhead |

### Selection Rules

- Use the simplest deployment that satisfies documented reliability and capacity requirements.
- Do not introduce Redis Cluster before there is a justified need for horizontal scaling or sharding.
- Do not assume replication provides strong consistency or protects against all data loss.
- Document the failover behavior and data-loss characteristics of the selected topology.
- Ensure application clients support the selected deployment mode.
- Validate compatibility of Redis commands and data structures with the chosen topology.
- Ensure local development and automated tests reproduce relevant production semantics.

### KAMPYN Deployment Considerations

KAMPYN may be deployed as a multi-tenant SaaS platform or self-hosted by individual universities.

Redis configuration must support:

- Tenant-aware key design.
- Independent environment configuration.
- Per-deployment capacity planning.
- Authentication and network restrictions.
- Configurable persistence and eviction policies.
- Monitoring and operational health checks.
- Optional separation of workloads into dedicated Redis instances or deployments.

Do not assume all universities require the same Redis topology or capacity.

---

## 4. Appropriate Use Cases

Redis should be selected when low latency, transient state, or coordination materially benefits the system.

### Caching

Redis is suitable for frequently accessed data that can be recomputed or reloaded from an authoritative source.

Examples:

- Food item details.
- Vendor and food court metadata.
- Public service configuration.
- Frequently accessed tenant settings.
- Menu summaries.
- Search result pages with stable filters.
- Expensive read-only aggregation results.
- Short-lived API responses.

### Ephemeral State

Redis may hold data that is expected to expire or can be recreated.

Examples:

- Login verification challenges.
- Temporary password-reset tokens.
- Short-lived rate-limit counters.
- Online user presence.
- Typing indicators.
- Temporary workflow state.
- WebSocket connection mappings.
- Short-lived deduplication markers.

### Coordination

Redis may be used for carefully bounded coordination tasks.

Examples:

- Distributed rate limiting.
- Short-lived work coordination.
- Best-effort distributed locks.
- Request coalescing.
- Temporary leader-election-like coordination only where the guarantees are sufficient.

Critical coordination must use a mechanism whose guarantees match the business invariant. Redis locks must not be treated as a substitute for database constraints or transactional correctness.

### Inappropriate Use Cases

Avoid Redis for:

- Large-scale relational joins.
- Complex transactional workflows.
- Permanent storage of business records.
- Unbounded event history.
- Long-term analytical datasets.
- Durable message delivery without appropriate persistence and recovery guarantees.
- Data requiring strong consistency across multiple independent keys unless the design explicitly supports it.

---

## 5. Key Design and Naming

Every Redis key must be understandable, unique within its namespace, and associated with a defined lifecycle.

### Naming Convention

Use:

```text
<platform>:<environment>:<domain>:<entity>:<identifier>:<purpose>
```

Examples:

```text
kampyn:prod:catalog:item:item_123
kampyn:prod:tenant:university_001:settings
kampyn:prod:ratelimit:user:user_123
kampyn:prod:presence:tenant_001:user_123
kampyn:prod:session:session_456
```

The examples are illustrative; the actual convention must be consistent across services.

### Key Rules

- Use lowercase names separated by colons.
- Include a clear domain or service namespace.
- Include the environment when Redis is shared across environments.
- Include tenant context for tenant-scoped keys.
- Use stable identifiers rather than user-controlled free-form strings.
- Keep key lengths reasonable.
- Avoid storing secrets or sensitive values in key names.
- Avoid ambiguous abbreviations.
- Define key ownership and purpose.
- Ensure related keys can be identified and cleaned up safely.
- Avoid key patterns that require expensive full-database scans during normal application requests.

### Tenant-Aware Keys

Tenant-scoped keys must include a trusted tenant identifier.

Example:

```text
kampyn:prod:tenant:university_001:catalog:item:item_123
```

Rules:

- Resolve tenant context from trusted authentication and authorization logic.
- Do not rely exclusively on client-supplied tenant IDs.
- Avoid cross-tenant key collisions.
- Validate tenant ownership before caching or retrieving tenant-specific data.
- Use tenant-aware invalidation.
- Ensure administrative cross-tenant operations are explicitly authorized.

### Cluster Hash Tags

When using Redis Cluster, hash tags MAY be used when multiple keys must reside in the same hash slot for supported multi-key operations.

Example:

```text
kampyn:prod:tenant:{university_001}:cart:user_123
kampyn:prod:tenant:{university_001}:cart:user_123:items
```

Rules:

- Use hash tags only when multi-key slot affinity is required.
- Avoid placing all tenants or all application keys in one shared hash slot.
- Consider the risk of uneven traffic distribution and hot slots.
- Document cross-key atomicity assumptions.
- Ensure the hash-tag strategy is compatible with the chosen client and deployment topology.

---

## 6. Data Structure Selection

Choose Redis data structures based on access patterns, atomicity requirements, memory usage, and lifecycle.

| Data Structure | Typical Use |
|---|---|
| String | Cache entries, counters, tokens, simple values |
| Hash | Structured records with independently accessed fields |
| List | Ordered transient queues and bounded recent activity |
| Set | Unique membership, tags, and membership checks |
| Sorted Set | Ranking, time-ordered activity, and scheduled expiry queues |
| Stream | Event-like processing where Redis Streams are explicitly approved |
| Bitmap | Compact boolean or presence tracking |
| HyperLogLog | Approximate cardinality estimation |
| Geo | Geographic lookup where supported use cases justify it |

### Selection Rules

- Prefer the simplest structure that supports the required operations.
- Document expected size and cardinality.
- Define expiration and cleanup behavior.
- Avoid large unbounded collections.
- Use atomic Redis commands or scripts for multi-step state transitions.
- Understand the complexity and memory overhead of the selected structure.
- Do not use a specialized structure solely because it exists.

### Strings

Use strings for:

- Simple cached JSON or serialized values.
- Short-lived tokens.
- Atomic counters.
- Small immutable or replaceable cache values.

Avoid placing extremely large serialized records into individual string values.

### Hashes

Use hashes for compact, related fields where individual field updates are useful.

Avoid treating Redis hashes as an alternative relational database or persisting large, unbounded object structures without a clear lifecycle.

### Lists

Use lists only for bounded sequences with known operational semantics.

- Define maximum length.
- Use trimming where appropriate.
- Define behavior when consumers fail.
- Do not treat a simple list as a durable queue unless persistence, acknowledgment, recovery, and delivery guarantees have been explicitly designed.

### Sets

Use sets for unique membership and membership checks.

- Define membership expiration or cleanup.
- Avoid unbounded growth.
- Ensure tenant ownership is represented in the key.
- Use atomic set operations where concurrency requires them.

### Sorted Sets

Use sorted sets for ranked or time-ordered data.

Examples include:

- Top items over a bounded interval.
- Recently active users.
- Temporary scheduling queues.
- Time-based expiration indexes.

Define score semantics and cleanup rules clearly. Avoid using client-controlled scores without validation.

### Streams

Redis Streams MAY be used for suitable event-processing workflows.

If used:

- Define consumer groups and acknowledgment behavior.
- Configure persistence and retention appropriately.
- Define pending-entry recovery.
- Make consumers idempotent.
- Define dead-letter or failure handling.
- Monitor stream length and pending messages.
- Do not treat stream delivery as equivalent to a transactional outbox unless the architecture explicitly closes the reliability gap.

---

## 7. Caching Standards

Caching must improve performance without compromising correctness, tenant isolation, or data privacy.

### Cache-Aside Pattern

Cache-aside is the default caching strategy unless another pattern is explicitly justified.

Typical flow:

1. Read the requested key from Redis.
2. If a valid entry exists, return it.
3. If the key is missing, load the value from the authoritative datastore.
4. Cache the result with a bounded TTL.
5. Return the value.
6. Invalidate or update the cache when authoritative data changes according to the defined strategy.

### Cache Rules

- Every cache entry MUST have an explicit TTL or documented cleanup mechanism.
- Every cache MUST identify its authoritative source.
- Cache keys MUST include relevant tenant and authorization context.
- Cached data MUST be safe to serve to the requesting user.
- Cache invalidation behavior MUST be documented.
- Cached values MUST be treated as potentially stale.
- Cache failures MUST have defined fallback behavior.
- Cache payloads MUST be versioned when schema evolution requires it.
- Cache sizes and key cardinality MUST be bounded.
- Cache hit rates and latency MUST be observable.

### TTL Selection

TTL must reflect the data's expected change frequency, acceptable staleness, and cost of recomputation.

| Data Type | Example TTL Strategy |
|---|---|
| Stable public configuration | Longer bounded TTL |
| Frequently updated menu data | Shorter bounded TTL |
| Search results | Short TTL based on freshness requirements |
| Tenant configuration | Explicit invalidation plus fallback TTL |
| Temporary verification challenge | Short, fixed expiry |
| Online presence | Short expiry with periodic refresh |
| Rate-limit counters | Defined by the rate-limit window |

These are qualitative guidelines, not fixed defaults. Each cache must have an explicitly selected expiration policy.

### Cache Invalidation

Choose an invalidation strategy that matches the workload.

Possible strategies:

- TTL-based expiration.
- Explicit invalidation after a successful database transaction.
- Transactional outbox events consumed by cache invalidation workers.
- Versioned cache keys.
- Short-lived caches with authoritative validation for critical fields.

Rules:

- Do not invalidate before the authoritative write commits.
- Do not assume cache invalidation succeeds merely because the database write succeeded.
- Define recovery behavior for missed invalidation events.
- Use TTLs as a safety net where practical.
- Make repeated invalidations safe.
- Invalidate related derived keys when their source data changes.
- Ensure invalidation operations remain tenant-scoped.

### Cache Stampede Prevention

When many requests miss the same key simultaneously:

- Use request coalescing or controlled refresh coordination where justified.
- Apply bounded concurrency to cache fills.
- Consider TTL jitter to reduce synchronized expiration.
- Use stale-while-revalidate only where serving bounded staleness is acceptable.
- Avoid unbounded background refresh tasks.
- Ensure lock expiry and failure recovery are defined if locks are used.

### Negative Caching

Negative caching may be used for repeated lookups of missing records.

- Use a short, bounded TTL.
- Ensure newly created records invalidate the negative entry.
- Avoid caching authorization failures as if they were missing entities.
- Ensure negative entries do not disclose cross-tenant existence.
- Do not use negative caching for business decisions requiring authoritative confirmation.

---

## 8. Cache Consistency and Correctness

Redis caches are not guaranteed to reflect the latest committed database state.

### Rules

- Use PostgreSQL or the appropriate authoritative store for decisions that require current business state.
- Do not rely on cached inventory for final stock deduction.
- Do not rely on cached booking availability for reservation confirmation.
- Do not rely on cached payment status for financial settlement.
- Do not rely on cached permissions as the sole basis for sensitive authorization decisions.
- Define acceptable staleness for each cache.
- Document read-after-write behavior.
- Invalidate or refresh caches when authoritative data changes.
- Use versioning or authoritative validation where stale reads could cause incorrect actions.

### Read-After-Write

Where users expect immediate visibility after updating a record:

- Read from the authoritative database after the write, or
- Update the cache using a controlled, version-aware process, or
- Use another documented strategy that satisfies the required consistency.

Do not promise strong read-after-write consistency from an asynchronous invalidation process alone.

### Cache Payload Versioning

For structured cache values:

- Include a schema version when format evolution requires it.
- Handle missing or incompatible cached values safely.
- Prefer cache invalidation over complex compatibility logic when entries are inexpensive to regenerate.
- Use versioned key namespaces for breaking changes.
- Avoid deserializing untrusted or unsafe object formats.

---

## 9. Cache Eviction and Memory Management

Redis memory is finite. Memory planning and eviction behavior must be explicit.

### Memory Requirements

- Estimate memory usage for each workload.
- Set an appropriate `maxmemory` policy.
- Monitor memory consumption and fragmentation.
- Bound large values and collection cardinality.
- Avoid unbounded keys and collections.
- Define behavior when Redis reaches memory limits.
- Account for replication, persistence, client buffers, and allocator overhead.
- Leave sufficient capacity for operational safety.

### Eviction Policies

Choose the eviction policy based on workload semantics.

| Policy | Typical Consideration |
|---|---|
| `noeviction` | Rejects memory-growing writes when the configured limit is reached |
| `allkeys-lru` | Evicts least recently used keys across the dataset |
| `allkeys-lfu` | Evicts less frequently used keys across the dataset |
| `volatile-lru` | Evicts least recently used keys that have expiry |
| `volatile-lfu` | Evicts less frequently used keys that have expiry |
| `volatile-ttl` | Evicts keys with expiry, favoring shorter remaining TTL |

No policy is universally correct.

### Rules

- Select an eviction policy based on whether the dataset is disposable, transient, or persistence-sensitive.
- Do not mix critical coordination state and disposable caches without understanding the effect of eviction.
- Prefer separate Redis instances or deployments when workloads require incompatible eviction and persistence policies.
- Do not rely on eviction as the only cleanup strategy for important ephemeral state.
- Monitor evictions and rejected writes.
- Treat unexpected eviction of coordination keys as a correctness risk.

### Memory Efficiency

- Use compact key names without sacrificing clarity.
- Keep values appropriately sized.
- Avoid duplicating large data unnecessarily.
- Use hashes or other compact structures only where they materially reduce overhead and remain manageable.
- Bound lists, streams, and sorted sets.
- Remove expired and obsolete data through appropriate lifecycle policies.
- Measure actual memory usage instead of estimating solely from payload size.

---

## 10. Expiration and Lifecycle

Every transient Redis key must have a defined lifecycle.

### Expiration Rules

- Set TTL atomically with the creation or update of temporary keys wherever practical.
- Avoid separate write and expiry commands where a crash could leave a key without expiration.
- Refresh TTL only when the intended lifecycle requires it.
- Ensure retries do not accidentally extend a security-sensitive token's lifetime.
- Define behavior when keys expire during active workflows.
- Avoid relying on key expiration timing for exact scheduling or guaranteed execution.

### Key Lifecycle

Every Redis key category must document:

- Purpose.
- Owner.
- Creation trigger.
- Update behavior.
- TTL or cleanup strategy.
- Invalidation trigger.
- Tenant scope.
- Data sensitivity.
- Behavior on expiration.
- Behavior during Redis failure.
- Recovery or recreation strategy.

### Expiry Notifications

Redis keyspace notifications MAY be used for non-critical reactive workflows.

They MUST NOT be treated as a guaranteed durable scheduling or event-delivery mechanism.

For business-critical scheduled tasks, use a durable scheduler or background-job architecture with explicit persistence and recovery.

---

## 11. Atomic Operations and Lua Scripting

Redis commands are atomic individually, but multi-command workflows may require additional protection.

### Atomicity Rules

- Prefer built-in atomic commands when they satisfy the requirement.
- Use transactions or Lua scripts for suitable multi-command atomic operations.
- Understand Redis Cluster key-slot restrictions.
- Keep scripts small and bounded.
- Avoid long-running scripts that block the Redis event loop.
- Validate all script inputs.
- Make scripts deterministic where practical.
- Version and test non-trivial scripts.
- Do not use atomic Redis operations as a substitute for authoritative database transactions.

### Transactions

Redis `MULTI`/`EXEC` can group commands, but they do not provide rollback semantics equivalent to PostgreSQL transactions.

Rules:

- Understand command execution and error behavior.
- Do not assume a failed command rolls back the entire transaction.
- Use `WATCH` only where optimistic concurrency semantics are appropriate.
- Handle conflicts and retries explicitly.
- Prefer Lua scripts for small operations that need conditional logic and atomic execution.

### Lua Scripts

Lua MAY be used for operations such as:

- Atomic rate-limit checks and updates.
- Conditional counter updates.
- Compare-and-delete lock release.
- Small state transitions requiring atomicity.

Avoid using Lua for large data transformations, lengthy computation, or operations that should belong in an authoritative database.

---

## 12. Distributed Locks and Coordination

Distributed locks are a specialized coordination mechanism and must be used cautiously.

### Appropriate Uses

- Avoiding duplicate execution of non-critical cache refreshes.
- Preventing redundant work across multiple workers.
- Coordinating short-lived background tasks where occasional recovery is acceptable.
- Request coalescing for expensive, repeatable operations.

### Lock Requirements

- Use a unique, unpredictable lock token for each acquisition.
- Set a bounded expiry during lock acquisition.
- Use atomic acquisition, such as `SET key token NX PX <ttl>`.
- Release a lock only if the stored token matches the caller's token.
- Do not delete a lock unconditionally.
- Define behavior when a lock expires while work is still running.
- Use bounded retries with jitter when acquisition fails.
- Keep lock duration as short as practical.
- Avoid holding locks during slow external requests.
- Record contention and lock failure metrics.

### Lock Release

Use an atomic compare-and-delete operation, typically implemented with a small Lua script.

Conceptual behavior:

```lua
if redis.call("GET", KEYS[1]) == ARGV[1] then
    return redis.call("DEL", KEYS[1])
end

return 0
```

The caller must not release a lock acquired by another process after the original lock has expired.

### Fencing and Critical Workflows

A Redis lock alone may be insufficient for critical workflows because:

- A process can pause beyond the lock TTL.
- A lock can expire while the original holder continues running.
- Network partitions and failover can create ambiguous ownership.
- Redis coordination does not automatically enforce ownership in the authoritative database.

For critical operations:

- Prefer database constraints, atomic updates, or transactional locking.
- Consider fencing tokens or version checks where the design requires protection from stale lock holders.
- Ensure the authoritative resource validates the operation.
- Do not use Redis locks as the sole protection against double booking, overselling, or duplicate financial settlement.

### Lock Failure

If a lock cannot be acquired:

- Do not assume ownership.
- Retry only within a defined budget.
- Return a controlled failure, defer the job, or use another approved workflow.
- Do not continue the protected operation without the required coordination guarantee.

---

## 13. Rate Limiting

Redis may be used for distributed rate limiting across multiple application instances.

### Requirements

- Define rate-limit scope: user, tenant, IP address, API key, or endpoint.
- Use trusted identity information where available.
- Apply separate limits to expensive or abuse-prone operations.
- Use atomic operations for counter updates.
- Define window semantics and reset behavior.
- Set TTLs for rate-limit state.
- Return consistent rate-limit response metadata.
- Ensure rate-limit keys are tenant-aware where required.
- Monitor rejected requests and anomalous traffic.

### Algorithms

| Algorithm | Typical Use |
|---|---|
| Fixed window | Simple bounded request limits |
| Sliding window counter | Smoother enforcement with lower complexity than exact sliding logs |
| Sliding window log | More precise limits with higher memory and processing cost |
| Token bucket | Controlled bursts with a defined sustained rate |
| Leaky bucket | Smoothing traffic to a consistent processing rate |

Choose an algorithm based on accuracy, memory overhead, throughput, and expected traffic behavior.

### Atomicity

The rate-limit check and update must be atomic to prevent concurrent requests from bypassing limits.

Use a Lua script or another proven atomic design for multi-step decisions.

### Failure Policy

Define whether rate limiting is fail-open or fail-closed for each endpoint class.

- Security-sensitive endpoints may require stricter handling.
- Non-critical browsing endpoints may use a documented degraded mode.
- Redis failure must not silently disable essential abuse protection without an approved fallback.
- Any local fallback must account for its per-instance limitations.

---

## 14. Sessions, Tokens, and Verification State

Redis MAY store short-lived authentication-related state when approved by the authentication architecture.

### Session Storage

- Use opaque session identifiers.
- Store only the session fields required by the application.
- Set explicit expiration.
- Rotate or revoke sessions according to authentication policy.
- Ensure tenant and user context are validated.
- Do not treat the Redis session as a substitute for authoritative account status or permissions.
- Ensure Redis access is restricted and encrypted in transit.
- Define logout and global session-revocation behavior.

### Verification Challenges

Temporary OTPs, password-reset challenges, and verification state may be stored in Redis if the approved security design permits it.

Requirements:

- Use cryptographically secure challenge generation.
- Store only the minimum required data.
- Use short, explicit expiration.
- Enforce attempt limits atomically.
- Bind challenges to the intended account and purpose.
- Prevent challenge reuse.
- Avoid exposing challenge values through logs, metrics, or keys.
- Define resend throttling and invalidation.
- Ensure sensitive verification state cannot be retrieved cross-tenant.

Redis expiration is not a replacement for server-side validation of challenge purpose, ownership, and validity.

---

## 15. Real-Time and Ephemeral State

Redis can support transient real-time features where the loss of state is acceptable or recoverable.

### Presence

Presence records may represent online status or recent activity.

- Use bounded TTLs.
- Refresh presence on an approved heartbeat or activity signal.
- Treat presence as approximate rather than guaranteed.
- Clean up stale state through expiration.
- Scope presence by tenant and user.
- Avoid using presence as proof of identity, authorization, or availability for critical actions.

### Typing Indicators

- Use short-lived state.
- Apply tenant and conversation boundaries.
- Prevent unbounded per-user or per-conversation growth.
- Treat missed or delayed indicators as acceptable.
- Do not persist typing state as durable conversation content.

### WebSocket Metadata

Redis may store transient routing metadata for distributed WebSocket deployments.

- Define connection ownership and expiration.
- Remove disconnected connections safely.
- Handle worker restarts and missed cleanup.
- Avoid treating connection metadata as durable.
- Ensure message delivery guarantees are handled by the appropriate messaging architecture.

### Real-Time Notifications

If Redis Pub/Sub is used:

- Treat it as transient message delivery.
- Do not assume disconnected subscribers will receive missed messages.
- Use durable event infrastructure where guaranteed delivery is required.
- Define reconnect and resynchronization behavior.
- Avoid publishing sensitive payloads unnecessarily.

---

## 16. Queues, Streams, and Background Processing

Redis may support selected transient work queues or stream-processing workflows, but durability and delivery guarantees must be explicitly evaluated.

### General Rules

- Prefer a dedicated durable queue or event platform for critical business events where required.
- Define acknowledgment and retry behavior.
- Make job processing idempotent.
- Bound queue length and payload size.
- Track pending and failed work.
- Provide dead-letter or recovery mechanisms where required.
- Avoid using Redis Pub/Sub for jobs that must not be lost.
- Define behavior during failover and restart.
- Monitor queue depth and processing lag.

### Redis Streams

When using Redis Streams:

- Define consumer groups.
- Acknowledge only after processing has reached the required completion state.
- Recover abandoned pending entries.
- Make processing safe for duplicates.
- Configure retention and trimming.
- Monitor pending entries, stream size, and consumer health.
- Define how failed messages are retried or moved to a dead-letter mechanism.
- Ensure the persistence configuration matches the expected delivery guarantees.

### Background Job Separation

Keep queue semantics separate from ordinary cache access.

Where queues have different durability, memory, or eviction requirements, use separate Redis deployments or a dedicated messaging system rather than sharing a single undifferentiated instance.

---

## 17. Redis and Multi-Tenancy

Tenant isolation must apply to every tenant-scoped Redis workload.

### Isolation Requirements

- Include tenant identifiers in tenant-scoped keys.
- Resolve tenant context from trusted sources.
- Validate tenant ownership before writing and reading cached data.
- Prevent cross-tenant collisions.
- Apply tenant-aware cache invalidation.
- Ensure rate-limit scopes are correctly defined.
- Scope presence, sessions, and temporary workflow state according to the security model.
- Restrict cross-tenant administrative access.
- Ensure background workers preserve tenant context.
- Test tenant isolation across all Redis-backed services.

### Shared Versus Dedicated Instances

Shared Redis is acceptable where workloads have compatible security, persistence, memory, and eviction requirements.

Dedicated Redis instances or deployments SHOULD be considered when:

- A tenant requires stronger infrastructure isolation.
- A workload has substantially different persistence requirements.
- A high-volume tenant creates noisy-neighbor risks.
- Workloads need different eviction policies.
- Operational or regulatory requirements demand separation.

Do not create an instance per tenant by default without a documented provisioning and maintenance strategy.

### Noisy-Neighbor Protection

- Monitor memory and request volume by workload or tenant where practical.
- Apply per-tenant rate limits where required.
- Bound tenant-controlled collection sizes.
- Use workload separation when shared resource contention becomes material.
- Avoid allowing a single tenant to create unbounded key cardinality or large values.

---

## 18. Redis Security

Redis must be treated as an internal infrastructure service.

### Network Security

- Do not expose Redis directly to public networks.
- Restrict access to approved application and operational networks.
- Use private networking wherever available.
- Enable TLS for production traffic.
- Restrict administrative endpoints and commands.
- Apply network policies and firewall rules.
- Ensure self-hosted installations follow documented secure network defaults.

### Authentication and Authorization

- Require authentication in production.
- Use service-specific credentials where supported.
- Apply least-privilege access through Redis ACLs.
- Restrict dangerous commands for application identities.
- Separate administrative and application access.
- Rotate credentials according to the platform security policy.
- Store secrets in an approved secret-management system.
- Never embed credentials in source code, images, logs, or frontend bundles.

### Data Protection

- Do not store passwords, long-lived secrets, or unnecessary sensitive data in Redis.
- Encrypt sensitive values at the application layer when required by the security architecture.
- Use TLS in transit.
- Protect snapshots and persistence files.
- Restrict access to Redis backups.
- Define data retention and deletion requirements.
- Avoid placing sensitive information in keys or metrics labels.

### Command Restrictions

Application identities SHOULD be restricted from dangerous administrative commands such as:

- `FLUSHALL`
- `FLUSHDB`
- `CONFIG`
- `SHUTDOWN`
- Unrestricted `DEBUG`
- Other administrative or destructive commands not required by the service

Exact ACL configuration must match the supported Redis version and the commands used by the application.

### Input Validation

- Validate all user-controlled values before using them in Redis operations.
- Do not allow arbitrary key names or command construction from client input.
- Bound payload size, collection size, and command complexity.
- Use safe serialization formats and validate decoded data.

---

## 19. Persistence and Data Durability

Redis persistence must be selected according to the durability needs of each workload.

### Persistence Options

| Option | Consideration |
|---|---|
| No persistence | Appropriate only for fully disposable, reconstructable data |
| RDB snapshots | Periodic snapshots with potential loss of recent writes |
| AOF | Append-only persistence with configurable durability and write overhead |
| RDB + AOF | Combined recovery options with additional operational considerations |
| Managed persistence | Provider-specific durability and recovery behavior that must be validated |

### Rules

- Define the data-loss tolerance for each Redis workload.
- Configure persistence explicitly rather than relying on deployment defaults.
- Do not assume replication alone provides durability.
- Document persistence behavior during restart, failover, and infrastructure replacement.
- Ensure backups and snapshots are access-controlled.
- Test restore procedures for any Redis state that requires recovery.
- Avoid persisting disposable cache data when the cost outweighs the recovery value.
- Do not claim zero data loss without a verified architecture that supports it.

### Workload Separation

Redis may host workloads with different durability requirements.

Where persistence, eviction, security, or availability needs conflict, separate them into distinct instances or deployments.

For example, disposable cache entries should not share an undifferentiated lifecycle with session or coordination state whose loss has a larger operational impact.

---

## 20. High Availability and Failover

Redis availability requirements must be documented for each workload.

### Requirements

- Choose an appropriate replication and failover model.
- Configure health checks and monitoring.
- Ensure clients support reconnect and topology changes.
- Define behavior for in-flight operations during failover.
- Handle ambiguous command outcomes safely.
- Apply bounded retries with backoff and jitter.
- Test failover under realistic application traffic.
- Monitor replication lag and failover events.
- Document expected availability and data-loss limitations.

### Failover Behavior

During failover:

- In-flight requests may fail or return ambiguous outcomes.
- Recent writes may be lost depending on topology and persistence configuration.
- Distributed locks may have uncertain ownership.
- Consumers may reconnect and repeat work.
- Cache entries may disappear or become stale.

Application services must be designed to recover safely from these conditions.

### Critical Workflows

For workflows involving orders, payments, bookings, or inventory:

- Redis failure must not create duplicate authoritative transactions.
- Use PostgreSQL or the relevant source of truth to confirm final state.
- Apply idempotency keys and transaction constraints.
- Reconcile ambiguous outcomes.
- Avoid treating Redis acknowledgments as proof of durable business completion.

---

## 21. Redis Client and Connection Management

Redis clients must be initialized centrally and reused.

### Requirements

- Use a maintained, approved Redis client library.
- Reuse connection pools or multiplexed connections according to the client's design.
- Do not create a new connection for every operation.
- Configure connection, command, and pool timeouts.
- Handle reconnection and topology changes.
- Configure TLS and authentication.
- Bound concurrent operations.
- Monitor connection counts, errors, and latency.
- Ensure shutdown closes clients gracefully.
- Propagate request context and cancellation where supported.

### Client Boundaries

- Keep Redis client initialization in a shared infrastructure module.
- Expose domain-oriented interfaces to application services.
- Avoid leaking raw Redis commands into unrelated business logic.
- Centralize serialization, key construction, and TTL policy where appropriate.
- Avoid generic abstractions that hide important atomicity or expiration behavior.
- Keep scripts and multi-key operations explicitly documented.

---

## 22. Serialization and Schema Evolution

Redis values must use safe, explicit, and versionable serialization.

### Requirements

- Prefer JSON or another approved interoperable serialization format for structured values.
- Use binary serialization only where there is a documented need and compatibility strategy.
- Validate decoded values.
- Include schema versions when formats may evolve.
- Define behavior for missing, malformed, or incompatible entries.
- Avoid unsafe object deserialization.
- Bound serialized payload size.
- Keep serialization deterministic where it supports hashing, comparison, or idempotency.

### Cache Versioning

For breaking cache changes, use versioned key namespaces.

Example:

```text
kampyn:prod:catalog:v2:item:item_123
```

During rollout:

- Define whether old and new versions can coexist.
- Ensure readers handle cache misses safely.
- Avoid indefinite support for obsolete cache schemas.
- Use invalidation or expiry to remove older values.
- Coordinate cache changes across services sharing the same keys.

---

## 23. Redis and Other Data Systems

Redis must integrate with the wider KAMPYN architecture through clear ownership and consistency boundaries.

### Redis and PostgreSQL

- PostgreSQL remains authoritative for durable relational state.
- Redis may cache PostgreSQL-backed reads.
- Cache invalidation must occur after successful authoritative commits.
- Use an outbox or other reliable event mechanism where missed invalidations matter.
- Do not use Redis as the only record of a successful business transaction.
- Use database constraints and transactions for critical concurrency invariants.

### Redis and MongoDB

- MongoDB remains authoritative for approved document-oriented workloads.
- Redis may cache selected MongoDB-backed reads.
- Define invalidation and refresh behavior.
- Do not duplicate document ownership without a clear synchronization contract.
- Ensure tenant scope is preserved across both systems.

### Redis and OpenSearch

- OpenSearch remains the derived search index.
- Redis may cache search responses or suggestions where justified.
- Cache keys must account for tenant, filters, sorting, and relevant visibility context.
- Define invalidation or TTL behavior when indexed data changes.
- Do not treat cached search results as current business truth.
- Validate critical actions against authoritative services.

### Redis and Event Infrastructure

- Redis Pub/Sub and Streams have different delivery and durability characteristics from dedicated event systems.
- Use transactional outbox patterns where database changes must reliably generate events.
- Define message acknowledgment, retry, replay, and failure handling.
- Ensure duplicate processing is safe.
- Avoid creating circular dependencies where Redis is both the only durable state and the only recovery mechanism.

### Redis and Object Storage

Redis may cache object metadata or temporary references, but binary assets should remain in object storage.

- Avoid storing large files in Redis.
- Use short-lived cache entries for temporary object references where appropriate.
- Ensure cached object metadata is invalidated when authoritative metadata changes.
- Do not treat Redis as the durable owner of uploaded files.

---

## 24. Performance and Scalability

Redis performance depends on memory pressure, command complexity, network latency, key distribution, and concurrency.

### Command Efficiency

- Prefer commands with predictable, bounded complexity.
- Avoid blocking operations on shared instances unless explicitly designed for them.
- Avoid large `KEYS` scans in production request paths.
- Use `SCAN` for controlled administrative iteration where appropriate.
- Avoid retrieving entire collections when only a subset is required.
- Use pipelining for independent operations where it reduces network round trips.
- Do not pipeline commands when doing so changes required error or atomicity semantics.
- Use Lua scripts sparingly and keep execution bounded.
- Avoid oversized multi-command operations that block other clients.

### Latency

- Measure Redis round-trip latency.
- Reduce unnecessary sequential calls.
- Use pipelining or batching where appropriate.
- Avoid excessive serialization and deserialization.
- Keep network placement close to application services where possible.
- Monitor slow commands and command latency distributions.
- Set timeouts based on workload requirements.

### Hot Keys

Hot keys can cause uneven load and latency.

- Identify keys with disproportionately high request volume.
- Avoid unnecessary global counters and shared mutable keys.
- Partition or shard workload where justified.
- Use local caching for safe immutable values when appropriate.
- Use bounded batching to reduce repeated access.
- Avoid introducing a single coordination key that serializes unrelated tenants or requests.

### Large Values and Collections

- Bound value size and collection cardinality.
- Avoid unbounded lists, hashes, sets, sorted sets, and streams.
- Paginate or fetch subsets where supported.
- Use appropriate trimming and cleanup.
- Monitor memory usage per workload where practical.
- Investigate latency spikes caused by large values or expensive commands.

### Capacity Planning

Capacity planning must account for:

- Number of keys.
- Average and maximum value sizes.
- Expiration patterns.
- Cache hit and miss rates.
- Peak request throughput.
- Command complexity.
- Connection counts.
- Replication overhead.
- Persistence and snapshot requirements.
- Memory fragmentation.
- Failover capacity.
- Tenant distribution.
- Workload growth.

Do not treat configured `maxmemory` as the total infrastructure memory requirement. Account for Redis overhead and host-level operational needs.

---

## 25. Eviction, Persistence, and Workload Isolation

Caching and coordination workloads can have incompatible failure semantics.

### Requirements

- Classify Redis keys by workload and durability.
- Define eviction behavior for each deployment.
- Avoid sharing one instance across workloads with conflicting memory and persistence needs without explicit review.
- Ensure critical ephemeral state has a recovery or failure strategy.
- Monitor eviction rates and memory pressure.
- Avoid silent loss of coordination data that affects correctness.

### Recommended Separation

Where operationally justified, separate:

- General disposable caches.
- Session and verification state.
- Rate limiting.
- Background queues or streams.
- Coordination and locks.
- Real-time presence and routing metadata.

Separation may be achieved through distinct Redis instances, clusters, or managed deployments, depending on security and infrastructure capabilities.

Logical database numbers alone must not be treated as strong security, resource, or failure isolation.

---

## 26. Observability and Monitoring

Redis-backed services must be observable at both infrastructure and application levels.

### Infrastructure Metrics

Monitor:

- Memory usage and fragmentation.
- Connected clients.
- Commands per second.
- Command latency.
- Cache hit and miss ratios.
- Evicted keys.
- Expired keys.
- Rejected writes.
- CPU utilization.
- Network throughput.
- Replication lag.
- Persistence status.
- Snapshot and AOF rewrite duration where applicable.
- Keyspace size and growth.
- Slow commands.
- Failover events.
- Cluster slot health where applicable.

### Application Metrics

Monitor:

- Cache hit and miss rates by workload.
- Cache-fill latency.
- Cache invalidation failures.
- Redis request errors.
- Command timeout rates.
- Rate-limit rejections.
- Lock acquisition failures and contention.
- Queue or stream depth.
- Consumer processing lag.
- Session lookup failures.
- Presence refresh failures.
- Serialization errors.
- Fallback-to-database rates.

### Logging

- Use structured logs.
- Include correlation IDs and safe tenant context where permitted.
- Avoid logging sensitive values or full cache payloads.
- Do not log verification codes, session tokens, or credentials.
- Record failure categories and relevant operation context.
- Apply sampling for high-volume repetitive events.

### Alerting

Alerts SHOULD cover:

- Redis unavailability.
- Sustained high memory usage.
- Unexpected evictions.
- Rejected writes.
- High command latency.
- Connection exhaustion.
- Replication or failover issues.
- Persistence failures.
- Cache invalidation backlog.
- Excessive cache misses.
- Growing queue or stream lag.
- Lock contention anomalies.
- Abnormal keyspace growth.

Thresholds must be based on deployment-specific service objectives.

---

## 27. Testing Standards

Every production Redis-backed feature must have appropriate automated tests.

### Unit Tests

Test:

- Key construction.
- Tenant scoping.
- TTL selection.
- Serialization and deserialization.
- Cache policy decisions.
- Invalidation logic.
- Rate-limit calculations.
- Lock ownership checks.
- Retry and error classification.
- Input validation.

### Integration Tests

Test against a compatible Redis environment:

- Key creation and expiration.
- Cache hits and misses.
- Cache invalidation.
- Atomic counters.
- Hash, set, list, and sorted-set behavior where used.
- Lua scripts.
- Transaction behavior.
- Cluster-compatible multi-key operations where relevant.
- Session lifecycle.
- Rate limiting.
- Lock acquisition and release.
- Pub/Sub reconnect behavior.
- Stream acknowledgment and pending-entry recovery where used.
- Serialization compatibility.
- Connection interruption and recovery.

Mocks must not be the only test of atomic operations, Lua behavior, expiration, locking, or data-structure semantics.

### Security Tests

Verify:

- Tenant-scoped keys cannot collide or leak across tenants.
- Unauthorized users cannot access another tenant's cached data.
- Client-controlled input cannot construct arbitrary Redis keys or commands.
- Sensitive values are not exposed in logs.
- Application identities cannot execute prohibited administrative commands.
- Session and verification state expires and is revoked correctly.

### Concurrency Tests

Test:

- Simultaneous cache fills.
- Concurrent rate-limit updates.
- Lock acquisition contention.
- Lock expiry during work.
- Duplicate event processing.
- Retry behavior after timeouts.
- Consumer restarts.
- Concurrent updates to transient state.

### Failure Tests

Test:

- Redis unavailability.
- Network timeouts.
- Connection resets.
- Failover.
- Cache misses during outages.
- Memory pressure and eviction.
- Persistence recovery where applicable.
- Partial queue or stream processing failure.

### Performance Tests

Benchmark:

- Read and write latency.
- Throughput under concurrent load.
- Cache hit ratios.
- Serialization overhead.
- Memory growth.
- Key-expiration patterns.
- Hot-key behavior.
- Bulk operations.
- Failover recovery.
- Tenant distribution under representative workloads.

---

## 28. Configuration and Environment Management

Redis configuration must be centralized, environment-aware, and securely managed.

### Requirements

- Keep endpoints and credentials outside source code.
- Use environment-specific configuration.
- Validate required settings during startup.
- Configure timeouts, retry budgets, TLS, and connection limits explicitly.
- Define key namespaces and TTL policies centrally.
- Separate credentials by service where supported.
- Document persistence and eviction configuration.
- Keep deployment manifests version-controlled.
- Avoid exposing Redis endpoints publicly.
- Support self-hosted configuration without hardcoded provider-specific assumptions.

### Example Environment Variables

```env
REDIS_URL=
REDIS_USERNAME=
REDIS_PASSWORD=
REDIS_TLS_ENABLED=true
REDIS_CONNECT_TIMEOUT_MS=3000
REDIS_COMMAND_TIMEOUT_MS=2000
REDIS_MAX_CONNECTIONS=50
REDIS_KEY_PREFIX=kampyn:prod
```

These are illustrative names. The actual configuration contract must be consistent across services, packages, and deployment environments.

Never commit production credentials or embed them in frontend bundles.

---

## 29. Development Standards

### Code Organization

Redis access must be encapsulated behind domain-oriented interfaces.

Suggested structure:

```text
internal/
  cache/
    client/
      client.go
      config.go
    catalog/
      cache.go
      keys.go
      serializer.go
    sessions/
      store.go
      keys.go
    ratelimit/
      limiter.go
      scripts.go
    coordination/
      lock.go
      scripts.go
    presence/
      store.go
      keys.go
    streams/
      consumer.go
      producer.go
    observability/
      metrics.go
      tracing.go
```

This is illustrative and must be adapted to the approved KAMPYN repository architecture.

### Coding Rules

- Centralize Redis client initialization.
- Reuse clients and connection pools.
- Do not create clients per request.
- Keep key construction deterministic and testable.
- Centralize TTL policy where it is shared.
- Use typed interfaces for domain-specific Redis operations.
- Avoid exposing raw Redis commands throughout unrelated business modules.
- Keep Lua scripts version-controlled and tested.
- Propagate context and cancellation where supported.
- Handle errors explicitly.
- Avoid silent fallback behavior that hides persistent Redis failures.
- Keep modules cohesive and production source files normally below 200 lines unless there is documented architectural justification.

### Error Handling

Redis errors must be classified appropriately.

Typical categories include:

- Connection errors.
- Command timeouts.
- Authentication and authorization failures.
- Serialization errors.
- Memory-limit errors.
- Cluster redirection and topology errors.
- Script errors.
- Missing or expired keys.
- Data-structure type mismatches.

Each domain service must define which errors can be safely treated as cache misses and which require a controlled failure.

A connection error must not automatically be interpreted as a cache miss if doing so would cause uncontrolled database load.

---

## 30. Self-Hosted University Deployments

KAMPYN must support universities operating their own Redis infrastructure.

### Documentation Requirements

Provide:

- Supported Redis versions and compatible alternatives, if approved.
- Minimum and recommended resource requirements.
- Network and firewall configuration.
- TLS and authentication setup.
- ACL and service-identity configuration.
- Memory and eviction configuration.
- Persistence and backup options.
- Replication and failover options.
- Monitoring and health checks.
- Connection and timeout configuration.
- Upgrade and compatibility procedures.
- Recovery and troubleshooting procedures.
- Integration requirements for dependent KAMPYN services.

### Deployment Flexibility

- Avoid hardcoding provider-specific endpoints.
- Keep cache behavior independent of a particular cloud provider.
- Document any features that require Redis-specific commands or modules.
- Validate the supported feature set against the self-hosted deployment.
- Ensure required scripts and client capabilities are compatible with the documented Redis versions.
- Provide secure defaults while allowing institution-approved infrastructure configuration.

Self-hosting support must be validated through installation, recovery, and upgrade testing.

---

## 31. Redis and Privacy

Redis often holds temporary information that may still be sensitive.

### Requirements

- Classify Redis data according to its sensitivity.
- Store only the minimum required information.
- Apply explicit TTLs to sensitive ephemeral state.
- Restrict access to session, verification, and user-specific keys.
- Avoid storing unnecessary personal information.
- Ensure data is removed when its retention purpose ends.
- Avoid including sensitive information in keys, logs, or metrics labels.
- Protect snapshots and persistence files.
- Define access and retention policies for operational tooling.
- Apply tenant-specific controls where relevant.

Temporary storage does not remove privacy or security obligations.

---

## 32. Incident Response and Troubleshooting

Redis operations must have documented procedures for common incidents.

### Common Incidents

| Incident | Required Response |
|---|---|
| High memory usage | Identify key growth, workload, and eviction pressure |
| High cache misses | Review TTLs, invalidation, hot paths, and backend load |
| High command latency | Inspect slow commands, large values, CPU, and network |
| Connection exhaustion | Review pool configuration, client lifecycle, and concurrency |
| Unexpected evictions | Inspect memory policy and workload separation |
| Redis unavailable | Apply defined fallback or degraded-mode behavior |
| Stale cache data | Review invalidation delivery, TTL, and authoritative state |
| Lock contention | Inspect lock scope, duration, and unnecessary serialization |
| Stream backlog | Inspect consumer health, processing failures, and capacity |
| Replication issues | Review topology, network health, lag, and failover state |
| Persistence failure | Protect remaining state and follow recovery procedures |

### Incident Requirements

- Identify the affected workload and tenants where permitted.
- Determine whether authoritative data is affected.
- Avoid destructive cleanup commands without explicit approval.
- Preserve relevant diagnostic evidence.
- Follow the documented recovery procedure.
- Reconcile any state that may have been lost or left inconsistent.
- Document root cause and preventive actions for significant incidents.

Do not use `FLUSHALL` or `FLUSHDB` as routine troubleshooting commands.

---

## 33. Anti-Patterns

The following practices are prohibited unless a documented exception is approved:

- Treating Redis as the authoritative database for durable business state.
- Storing orders, payments, bookings, or final inventory solely in Redis.
- Creating a new Redis client for every request.
- Exposing Redis directly to public networks.
- Embedding credentials in source code or frontend applications.
- Using unbounded keys or collections.
- Creating keys without an ownership and lifecycle definition.
- Omitting TTLs for temporary data.
- Assuming every cache entry is current.
- Relying on cache invalidation alone for critical business correctness.
- Invalidating caches before authoritative database commits.
- Trusting client-provided tenant IDs without authorization.
- Using Redis locks as the sole protection for critical financial or booking operations.
- Releasing locks without verifying lock ownership.
- Using unbounded lock retries.
- Using Pub/Sub for guaranteed durable delivery.
- Treating Redis Streams as automatically equivalent to a transactional outbox.
- Running expensive blocking commands on shared production instances.
- Using `KEYS` for routine production request processing.
- Executing arbitrary Lua scripts from untrusted input.
- Ignoring memory pressure and eviction behavior.
- Mixing incompatible durability and eviction workloads without review.
- Using Redis expiration as an exact scheduling guarantee.
- Assuming replication eliminates data loss.
- Retrying ambiguous writes without idempotency controls.
- Logging session tokens, OTPs, or sensitive cache payloads.
- Skipping concurrency, failure, and tenant-isolation tests.
- Introducing Redis Cluster without a documented scaling requirement.

---

## 34. Required Documentation

Every production Redis-backed workload MUST document:

- Workload owner and business purpose.
- Authoritative source of truth.
- Key naming convention.
- Tenant-isolation model.
- Data structures used and rationale.
- Value schema and serialization format.
- TTL and lifecycle strategy.
- Cache consistency and invalidation policy.
- Memory and cardinality limits.
- Eviction and persistence requirements.
- Atomicity and concurrency guarantees.
- Failure and fallback behavior.
- Connection and timeout configuration.
- Security and access-control requirements.
- Metrics, logging, tracing, and alerts.
- Backup and recovery expectations where relevant.
- Testing strategy.
- Self-hosted deployment requirements where applicable.

Significant changes to Redis topology, persistence, eviction policy, or coordination guarantees should be documented in an architecture decision record (ADR).

---

## 35. Review Checklist

Before approving a Redis-backed feature, verify:

- [ ] Redis is appropriate for the workload.
- [ ] The authoritative source of truth is documented.
- [ ] Key ownership and naming are explicit.
- [ ] Tenant context is enforced in tenant-scoped keys and operations.
- [ ] Data structures match the access pattern.
- [ ] TTL and cleanup behavior are defined.
- [ ] Cache invalidation and staleness are understood.
- [ ] Critical business decisions do not rely solely on cached values.
- [ ] Memory usage and collection cardinality are bounded.
- [ ] Eviction and persistence policies are appropriate.
- [ ] Atomic operations protect required concurrency invariants.
- [ ] Distributed locks have bounded expiry and safe release behavior.
- [ ] Rate limiting is atomic and has a documented failure policy.
- [ ] Sessions and verification state have explicit security controls.
- [ ] Client libraries and connection pools are centrally managed.
- [ ] Timeouts, retries, and fallback behavior are configured.
- [ ] Redis access is authenticated, encrypted, and network-restricted.
- [ ] Sensitive values are minimized and protected.
- [ ] Observability and alerting are implemented.
- [ ] Failover and recovery behavior are documented.
- [ ] Unit, integration, concurrency, and failure tests pass.
- [ ] Performance has been evaluated using representative workloads.
- [ ] Self-hosted deployment requirements are documented where applicable.

---

## 36. Definition of Done

A Redis-backed feature is complete only when:

1. Its purpose and workload owner are documented.
2. Its authoritative source of truth is explicit.
3. Key naming, tenant scope, and lifecycle are defined.
4. Data structures and serialization are appropriate and bounded.
5. TTL, invalidation, and staleness behavior are documented.
6. Memory, eviction, and persistence policies are suitable for the workload.
7. Atomicity and concurrency requirements are implemented.
8. Failure, retry, and fallback behavior are defined.
9. Connection management and configuration follow platform standards.
10. Authentication, authorization, and privacy requirements are satisfied.
11. Metrics, logs, traces, and alerts are available.
12. Recovery and operational procedures are documented.
13. Unit, integration, security, concurrency, and relevant performance tests pass.
14. Cross-system consistency and reconciliation behavior are defined.
15. Self-hosted deployment requirements are documented where applicable.

**Final principle:** Redis exists to make KAMPYN faster and to support transient coordination—not to replace durable business storage. Every key must have a clear purpose, bounded lifecycle, and failure strategy, while critical business correctness remains protected by authoritative systems.
# OpenSearch Engineering Standards

## 1. Purpose

OpenSearch is KAMPYN's dedicated search and discovery engine. It provides fast, flexible, full-text, faceted, and relevance-based search across platform entities such as food items, vendors, food courts, services, announcements, and other searchable resources.

OpenSearch is a **derived search system, not a source of truth**. It must never replace PostgreSQL, MongoDB, or other authoritative data stores.

These standards define how OpenSearch indices, mappings, indexing pipelines, queries, synchronization, tenant isolation, performance, security, and operational reliability must be designed and implemented.

### Core Principles

- PostgreSQL and MongoDB remain the authoritative sources of business data.
- OpenSearch contains denormalized, search-optimized representations of authoritative records.
- Search results must respect tenant boundaries and authorization requirements.
- Indexing must be resilient to retries, duplicate events, delayed delivery, and partial failures.
- Search must remain performant as the number of tenants, documents, and queries grows.
- Every index must have an explicit owner, mapping, lifecycle, and synchronization strategy.
- Search relevance must be measurable, testable, and configurable.
- OpenSearch must not be used as a transactional database or general-purpose cache.

---

## 2. Architectural Position

OpenSearch is a specialized infrastructure component within the KAMPYN ecosystem.

### Data Ownership

| System | Responsibility |
|---|---|
| PostgreSQL | Primary relational and transactional source of truth |
| MongoDB | Justified document-oriented source of truth |
| Redis | Caching, ephemeral state, rate limiting, and coordination |
| OpenSearch | Full-text search, filtering, facets, ranking, and discovery |
| Object Storage | Images, documents, and other binary assets |
| Backend Services | Business logic, authorization, indexing orchestration, and search APIs |

### Data Flow

```text
                 ┌─────────────────────┐
                 │  Backend Services   │
                 └──────────┬──────────┘
                            │
                  Create / Update / Delete
                            │
                            ▼
                 ┌─────────────────────┐
                 │ Authoritative Store │
                 │ PostgreSQL / MongoDB│
                 └──────────┬──────────┘
                            │
                   Transactional Outbox
                   or Reliable CDC/Event
                            │
                            ▼
                 ┌─────────────────────┐
                 │ Event Infrastructure│
                 └──────────┬──────────┘
                            │
                            ▼
                 ┌─────────────────────┐
                 │ Indexing Workers    │
                 │ Validate / Transform│
                 └──────────┬──────────┘
                            │
                            ▼
                 ┌─────────────────────┐
                 │    OpenSearch       │
                 │ Search Indices      │
                 └──────────┬──────────┘
                            │
                            ▼
                 ┌─────────────────────┐
                 │ Search API          │
                 │ Filter / Authorize  │
                 └──────────┬──────────┘
                            │
                            ▼
                 ┌─────────────────────┐
                 │ Web / Mobile / SDK  │
                 └─────────────────────┘
```

### Mandatory Rules

- Business writes must commit to the authoritative store before they are treated as successful.
- Search indexing must not be the only persistence mechanism for business data.
- Search APIs must not expose direct OpenSearch credentials or unrestricted cluster access to clients.
- Indexing failures must not silently discard authoritative changes.
- Search results must be treated as potentially stale.
- Any operation requiring guaranteed current business state must validate against the authoritative store.

---

## 3. Appropriate Use Cases

OpenSearch SHOULD be used for workloads that benefit from text search, flexible filtering, faceted discovery, or relevance scoring.

### Approved Use Cases

- Food item and menu search.
- Vendor and food court discovery.
- Search across campus services and facilities.
- Announcements and public content discovery.
- Filtering by category, availability, price, location, rating, and tags.
- Autocomplete and search suggestions.
- Typo-tolerant search where appropriate.
- Multi-field relevance ranking.
- Aggregations and faceted navigation.
- Search analytics and query insights, subject to privacy policies.

### Inappropriate Use Cases

OpenSearch MUST NOT be used as the primary mechanism for:

- Financial transactions or payment records.
- Order creation, fulfillment, or authoritative order status.
- Inventory reservation or stock deduction.
- Booking confirmation or conflict prevention.
- Authentication credentials, password storage, or session authority.
- Access-control policy evaluation.
- Distributed locking or transactional coordination.
- General-purpose relational joins.
- Durable event storage or the sole source of audit history.
- High-integrity counters or balances.

For operations that require strong consistency, use the appropriate authoritative datastore and business service.

---

## 4. Index Ownership and Boundaries

Every index MUST have a clear domain owner and documented purpose.

### Index Design Rules

- Create indices around search use cases, not simply one index per database table or collection.
- Avoid one enormous index containing unrelated domain entities with incompatible mappings and access patterns.
- Avoid excessive index fragmentation that creates operational overhead.
- Keep document structures focused on the fields required for search and result presentation.
- Denormalize only the fields required to support approved search experiences.
- Maintain explicit ownership for index creation, updates, mappings, and deletion.
- Document all source entities contributing to an index.

### Example Index Boundaries

| Index | Purpose |
|---|---|
| `kampyn-items-v1` | Searchable food items and products |
| `kampyn-vendors-v1` | Vendor and food court discovery |
| `kampyn-services-v1` | Campus service discovery |
| `kampyn-announcements-v1` | Searchable announcements |
| `kampyn-suggestions-v1` | Autocomplete and query suggestions, if independently justified |

These are illustrative names. Actual index boundaries must be driven by the approved domain model and query requirements.

### Index Naming Convention

Use:

```text
<platform>-<domain>-<purpose>-v<version>
```

Examples:

```text
kampyn-items-search-v1
kampyn-vendors-search-v1
kampyn-announcements-search-v1
```

Rules:

- Use lowercase letters, numbers, and hyphens.
- Include a schema version for versioned physical indices.
- Avoid ambiguous or temporary production names.
- Do not reuse an index name for an incompatible schema.
- Use aliases to abstract versioned physical indices from applications.

---

## 5. Document Modeling

OpenSearch documents must be optimized for search, not for transactional normalization.

### Modeling Requirements

Every indexed document MUST:

- Have a deterministic document ID.
- Include the tenant identifier when tenant-scoped.
- Include the authoritative entity identifier.
- Include fields required for filtering and authorization.
- Include a schema version or an equivalent documented mapping version.
- Include synchronization metadata where required.
- Exclude unnecessary sensitive or confidential information.
- Have explicit field types in the index mapping.

### Example Document

```json
{
  "id": "item_123",
  "tenant_id": "university_001",
  "entity_type": "food_item",
  "name": "Paneer Butter Masala",
  "description": "Creamy paneer curry",
  "category_id": "main_course",
  "category_name": "Main Course",
  "vendor_id": "vendor_456",
  "vendor_name": "Campus Kitchen",
  "food_court_id": "court_001",
  "price_minor": 18000,
  "currency": "INR",
  "is_available": true,
  "is_active": true,
  "tags": ["paneer", "vegetarian"],
  "updated_at": "2026-09-28T10:00:00Z",
  "schema_version": 1
}
```

The example is illustrative. Fields must match the canonical KAMPYN domain model.

### Denormalization

Denormalization is permitted when it improves search efficiency.

For example, a food item document may contain its vendor name and food court name to avoid runtime joins during search.

Rules:

- Store only fields needed by approved search experiences.
- Define how changes to referenced entities update denormalized documents.
- Do not assume denormalized fields are always current.
- Use authoritative stores to resolve critical business information.
- Avoid duplicating large nested objects into every search document.
- Avoid storing complete records simply for convenience.

### Document IDs

- Document IDs MUST be deterministic and stable.
- Use authoritative entity IDs wherever possible.
- If IDs are not globally unique, include a tenant-safe deterministic namespace.
- Never generate a new random document ID for each indexing attempt.
- Repeated indexing of the same logical entity must update the same document.

---

## 6. Mapping Standards

Explicit mappings MUST be defined for all production indices.

Dynamic mapping MUST NOT be relied upon for critical production fields because it can create unintended types, inconsistent schemas, and mapping explosion.

### Common Field Types

| Field | Recommended Type |
|---|---|
| `id` | `keyword` |
| `tenant_id` | `keyword` |
| `entity_type` | `keyword` |
| `name` | `text` with suitable keyword subfield |
| `description` | `text` |
| `category_id` | `keyword` |
| `vendor_id` | `keyword` |
| `price_minor` | `long` |
| `currency` | `keyword` |
| `is_available` | `boolean` |
| `is_active` | `boolean` |
| `tags` | `keyword` |
| `rating` | `scaled_float` where appropriate |
| `created_at` | `date` |
| `updated_at` | `date` |
| `location` | `geo_point` where geographic search is required |

### Example Mapping

```json
{
  "mappings": {
    "dynamic": "strict",
    "properties": {
      "id": {
        "type": "keyword"
      },
      "tenant_id": {
        "type": "keyword"
      },
      "entity_type": {
        "type": "keyword"
      },
      "name": {
        "type": "text",
        "fields": {
          "keyword": {
            "type": "keyword",
            "ignore_above": 256
          }
        }
      },
      "description": {
        "type": "text"
      },
      "category_id": {
        "type": "keyword"
      },
      "vendor_id": {
        "type": "keyword"
      },
      "price_minor": {
        "type": "long"
      },
      "currency": {
        "type": "keyword"
      },
      "is_available": {
        "type": "boolean"
      },
      "is_active": {
        "type": "boolean"
      },
      "tags": {
        "type": "keyword"
      },
      "updated_at": {
        "type": "date"
      },
      "schema_version": {
        "type": "integer"
      }
    }
  }
}
```

### Mapping Rules

- Use `keyword` for exact matching, filtering, sorting, and aggregations.
- Use `text` for analyzed full-text search.
- Use multi-fields when both analyzed text and exact matching are needed.
- Use numeric types for numeric comparisons and sorting.
- Use `date` for temporal fields.
- Use `geo_point` or `geo_shape` only where location-based search requires them.
- Avoid unrestricted `object` fields and unbounded dynamic structures.
- Define nested mappings only when nested-object query semantics are necessary.
- Avoid mapping the same logical field with inconsistent types across indices.
- Use explicit analyzers for language-specific requirements.
- Set sensible limits for keyword field lengths and nested structures.

### Mapping Changes

Existing field types generally cannot be changed in place safely.

For incompatible mapping changes:

1. Create a new versioned index.
2. Apply the new mapping and settings.
3. Backfill documents.
4. Validate data and query behavior.
5. Switch the read/write aliases according to the migration plan.
6. Retain the previous index for a defined rollback period.
7. Remove obsolete indices only after approval.

Refer to `data/migrations.md` for migration governance.

---

## 7. Analyzers and Text Processing

Analyzers must be selected according to the language, data characteristics, and search behavior of the domain.

### Standards

- Use the standard analyzer as a baseline where suitable.
- Use language-specific analyzers when required by the target user population.
- Normalize text consistently across indexing and query construction.
- Support case-insensitive search through analyzers or normalized keyword fields as appropriate.
- Use synonyms only when there is a clear product requirement and a governed synonym source.
- Avoid aggressive stemming or tokenization that damages names, identifiers, or product terminology.
- Preserve exact-match capabilities for vendor names, IDs, categories, and other structured fields.
- Test analyzers against realistic campus search terms, spelling variants, abbreviations, and multilingual content.

### Multilingual Search

KAMPYN may serve universities with multilingual user populations.

Where multilingual search is required:

- Identify the supported languages explicitly.
- Define language-aware analyzers and query behavior.
- Avoid assuming a single analyzer provides equivalent quality across languages.
- Preserve original display text.
- Test mixed-language, transliterated, and code-switched queries where relevant.
- Document limitations for unsupported languages.

Do not add complex language pipelines without evidence that they improve the intended search experience.

---

## 8. Search API Design

Applications MUST access OpenSearch through backend search services.

### Search Service Responsibilities

The search service is responsible for:

- Validating and normalizing incoming search requests.
- Resolving tenant context from trusted authentication and routing context.
- Applying mandatory tenant and visibility filters.
- Enforcing query complexity and pagination limits.
- Constructing approved OpenSearch queries.
- Applying ranking and sorting rules.
- Transforming OpenSearch responses into stable API contracts.
- Handling timeouts, partial failures, and unavailable indices.
- Recording safe search telemetry.

### API Rules

- Use explicit request and response schemas.
- Validate query length, filters, sort fields, and pagination parameters.
- Use allowlisted sort fields and filterable fields.
- Apply maximum page sizes.
- Prefer cursor or `search_after` pagination for deep result traversal.
- Do not expose raw OpenSearch query DSL to public clients.
- Do not expose index names, internal cluster details, or sensitive scoring metadata.
- Do not allow arbitrary query scripts or user-supplied aggregations.
- Return stable identifiers that allow authoritative records to be retrieved where needed.

### Example Request

```json
{
  "query": "paneer",
  "filters": {
    "category_id": ["main_course"],
    "is_available": true
  },
  "sort": "relevance",
  "page_size": 20
}
```

### Example Response

```json
{
  "items": [
    {
      "id": "item_123",
      "name": "Paneer Butter Masala",
      "vendor_id": "vendor_456",
      "price_minor": 18000,
      "currency": "INR"
    }
  ],
  "next_cursor": null
}
```

These examples are illustrative. Final contracts must be defined in the API specifications and validated with Zod or the relevant backend validation framework.

---

## 9. Query Design and Relevance

Search queries must be intentional, bounded, and aligned with the user experience.

### Query Design

- Use `match` or `multi_match` for analyzed full-text search.
- Use `term` and `terms` for exact filters.
- Use `range` for numeric and temporal ranges.
- Use `bool` queries to combine mandatory filters and relevance clauses.
- Use `filter` context for structured constraints that do not require scoring.
- Use aggregations for supported facets and discovery filters.
- Use autocomplete-specific fields or indices when their behavior differs materially from full-text search.
- Avoid expensive wildcard, regexp, script, and leading-wildcard queries unless explicitly justified and controlled.
- Bound the number and complexity of filters and aggregations.

### Relevance

Relevance must reflect documented product requirements rather than arbitrary boosts.

Potential signals include:

- Text match quality.
- Exact name matches.
- Category match.
- Vendor or food court context.
- Availability.
- Explicitly defined popularity signals.
- User-selected sorting and filters.

Rules:

- Keep ranking logic in a testable search module.
- Do not use sensitive personal attributes for search ranking.
- Do not make ranking dependent on unbounded or unvalidated client parameters.
- Ensure boosts cannot override mandatory visibility or tenant filters.
- Document ranking changes and evaluate their impact.
- Keep relevance scoring separate from business authorization.

### Typo Tolerance and Fuzzy Search

Fuzzy matching may improve discovery for misspelled queries but can increase query cost and produce irrelevant results.

- Enable fuzzy behavior only for appropriate fields and query lengths.
- Avoid fuzzy matching for IDs, codes, and exact identifiers.
- Set explicit edit-distance and expansion limits.
- Test common misspellings and near-match false positives.
- Prefer autocomplete or curated suggestions when they provide a more predictable experience.

### Search Suggestions

Autocomplete should be designed as a distinct use case where necessary.

- Keep suggestion payloads small.
- Define minimum query length and result limits.
- Apply the same tenant and visibility constraints as regular search.
- Prevent suggestions from exposing hidden or unauthorized entities.
- Avoid returning sensitive search terms or user-specific data without a defined privacy basis.

---

## 10. Multi-Tenancy and Data Isolation

Tenant isolation is a mandatory security requirement for KAMPYN.

### Tenant Identification

Every tenant-scoped document MUST include a `tenant_id` field mapped as `keyword`.

The tenant context MUST be obtained from trusted backend authentication, verified service context, or a trusted routing layer. It MUST NOT be accepted solely from an untrusted client-supplied filter.

### Query Isolation

Every tenant-scoped search MUST include a mandatory tenant filter.

Example:

```json
{
  "query": {
    "bool": {
      "filter": [
        {
          "term": {
            "tenant_id": "university_001"
          }
        },
        {
          "term": {
            "is_active": true
          }
        }
      ],
      "must": [
        {
          "match": {
            "name": "paneer"
          }
        }
      ]
    }
  }
}
```

The tenant ID in this example must be injected by trusted backend logic, not copied from a request without authorization.

### Isolation Requirements

- Tenant filters MUST be applied to every tenant-scoped query, aggregation, suggestion, and document lookup.
- Bulk indexing operations MUST validate tenant ownership for every document.
- Tenant identifiers MUST be included in document identity or otherwise enforced through a proven uniqueness strategy.
- Tenant context MUST be preserved in indexing events and worker processing.
- Search responses MUST not leak cross-tenant data through hits, counts, facets, suggestions, errors, or debug information.
- Administrative cross-tenant access MUST be explicitly authorized, audited, and restricted.
- Tenant deletion and offboarding workflows MUST include index cleanup or equivalent data isolation actions.

### Isolation Models

Possible approaches include:

- Shared indices with mandatory tenant filters.
- Separate indices for selected high-isolation or high-scale tenants.
- Separate clusters for deployments requiring independent infrastructure.

The chosen model must consider operational cost, scale, isolation requirements, index count, shard overhead, and self-hosting needs.

Do not create an index per tenant by default without a documented capacity and lifecycle plan.

---

## 11. Authorization and Visibility

OpenSearch is not an authorization authority.

Search filtering can reduce exposure, but final access to protected business resources must be enforced by the relevant backend services.

### Rules

- Apply visibility filters before returning results.
- Use trusted user and tenant context to construct access constraints.
- Distinguish public, tenant-visible, role-restricted, and user-specific content.
- Do not rely on document presence or obscurity as access control.
- Revalidate permissions against authoritative services for sensitive actions.
- Avoid indexing secrets, authentication tokens, private credentials, or unnecessary personal data.
- Ensure aggregations and result counts cannot reveal protected records.
- Ensure search logs do not capture confidential query content unnecessarily.

If visibility changes, the indexing pipeline must propagate those changes reliably, with a defined strategy for preventing stale documents from remaining discoverable.

---

## 12. Indexing and Synchronization

Index synchronization must be reliable, repeatable, and observable.

### Source of Truth

Authoritative business data remains in PostgreSQL or MongoDB. OpenSearch documents are derived representations.

### Preferred Synchronization Pattern

Use a transactional outbox or another reliable event-delivery mechanism where appropriate.

The event should include enough information to identify the affected entity and its tenant, while avoiding unnecessary replication of sensitive data.

Example event:

```json
{
  "event_id": "evt_123",
  "event_type": "item.updated",
  "tenant_id": "university_001",
  "entity_id": "item_123",
  "entity_version": 12,
  "occurred_at": "2026-09-28T10:00:00Z"
}
```

### Indexing Worker Responsibilities

- Consume events reliably.
- Validate event schemas and tenant context.
- Load authoritative data when required.
- Transform source data into the canonical search document.
- Apply deterministic document IDs.
- Use bulk indexing for suitable workloads.
- Retry transient failures with bounded exponential backoff and jitter.
- Route unrecoverable events to a dead-letter mechanism.
- Record indexing outcomes and processing latency.
- Support replay and reconciliation.
- Avoid acknowledging an event before the required indexing operation has been handled according to the delivery contract.

### Idempotency

Indexing MUST be idempotent.

- Reprocessing the same event must not create duplicate documents.
- Use deterministic IDs.
- Track entity versions or equivalent ordering metadata where necessary.
- Reject or safely handle stale updates.
- Ensure delete events cannot be unintentionally undone by delayed older updates.
- Make retry behavior safe after timeouts where the write outcome is unknown.

### Event Ordering

Events may be delayed, duplicated, or processed out of order.

- Do not assume delivery order unless the event infrastructure guarantees it for the relevant key.
- Use entity versions, sequence numbers, or authoritative-state reloads to protect against stale writes.
- Define conflict-resolution behavior for concurrent updates.
- Treat deletes and visibility revocations as high-priority synchronization events.

### Bulk Indexing

Bulk indexing SHOULD be used for backfills and suitable event batches.

- Bound batch size by both document count and payload size.
- Handle per-document failures in bulk responses.
- Retry only retryable failures.
- Avoid unbounded queues or large in-memory batches.
- Apply backpressure when OpenSearch is overloaded.
- Monitor bulk latency, rejection rates, and failure counts.

---

## 13. Deletes and Stale Documents

Deletes and visibility changes must propagate to OpenSearch reliably.

### Delete Requirements

- Use explicit deletion events or reconciliation workflows.
- Make repeated deletes safe.
- Protect against delayed updates recreating deleted documents.
- Define tombstone or version-tracking behavior where needed.
- Ensure tenant offboarding removes or isolates the tenant's indexed data.
- Define retention and cleanup rules for soft-deleted entities.

### Stale Search Results

OpenSearch may return stale data because indexing is asynchronous.

- Document expected indexing latency for each search use case.
- Do not use search results as proof of current availability, inventory, price, booking status, or authorization.
- Validate critical data against authoritative services before executing business actions.
- Define cache invalidation behavior where search results are cached separately.
- Ensure updates that revoke visibility are propagated with suitable urgency.

---

## 14. Reindexing and Backfills

Every index must support rebuilding from authoritative data or another explicitly approved durable source.

### Reindexing Requirements

- Provide a documented full reindex procedure.
- Support incremental backfills where appropriate.
- Track backfill progress and failures.
- Ensure reindexing is restartable.
- Avoid blocking normal search traffic unnecessarily.
- Define capacity limits and rate controls.
- Validate document counts and representative field correctness.
- Ensure tenant isolation throughout the process.

### Versioned Reindexing

Use versioned indices and aliases for mapping or transformation changes.

Example:

```text
kampyn-items-search-v1
kampyn-items-search-v2

kampyn-items-search-read
kampyn-items-search-write
```

A safe reindex workflow:

1. Create the new versioned index.
2. Apply approved mappings and index settings.
3. Start a snapshot or consistent backfill strategy appropriate to the source.
4. Populate documents.
5. Replay or reconcile changes that occurred during the backfill.
6. Validate counts, representative documents, and query behavior.
7. Switch aliases.
8. Monitor errors, latency, and relevance.
9. Retain the previous index for the defined rollback period.
10. Remove the previous index only after approval.

The exact backfill and event-replay strategy must prevent lost updates between the initial data scan and alias cutover.

### Reconciliation

Reconciliation SHOULD detect:

- Missing documents.
- Unexpected documents.
- Stale entity versions.
- Incorrect tenant ownership.
- Mapping or transformation errors.
- Failed or skipped events.

Reconciliation must be rate-limited and designed to avoid overwhelming source databases or OpenSearch.

---

## 15. Pagination, Sorting, and Facets

### Pagination

- Use `from` and `size` only for shallow pagination within documented limits.
- Use `search_after` with a stable sort for deep pagination.
- Use a point-in-time context where consistent traversal is required and supported.
- Define cursor expiration and invalidation behavior.
- Avoid exposing raw OpenSearch sort state directly as an unvalidated client-controlled parameter.

### Sorting

- Allowlist sort fields.
- Use keyword or numeric fields with explicit mappings for sorting.
- Define stable tie-breakers to prevent duplicates or skipped results across pages.
- Ensure sorting cannot bypass visibility or tenant filters.
- Do not sort on analyzed text fields.

### Faceted Search

- Use aggregations for approved facets such as category, vendor, availability, or price range.
- Apply the same authorization and tenant filters to aggregations as to search hits.
- Limit aggregation size and complexity.
- Avoid exposing sensitive category counts or hidden entities.
- Monitor aggregation latency and memory consumption.

---

## 16. Performance and Scalability

OpenSearch performance depends on document shape, mappings, query complexity, shard strategy, indexing volume, and cluster capacity.

### Query Performance

- Profile expensive queries before optimizing.
- Use filters for structured constraints.
- Limit result size and aggregation cardinality.
- Avoid unnecessary wildcard, script, and deep-pagination operations.
- Retrieve only the fields required by the response.
- Use request timeouts and cancellation where supported.
- Avoid broad queries across unrelated indices.
- Cache only queries with stable semantics and an explicit invalidation strategy.
- Measure latency by query class and tenant scale.

### Index Performance

- Keep documents compact.
- Avoid unbounded arrays and deeply nested structures.
- Avoid unnecessary stored fields and doc values.
- Choose analyzers and mappings based on actual search requirements.
- Tune refresh intervals according to indexing and freshness requirements.
- Use bulk operations for high-volume ingestion.
- Monitor merge activity, segment counts, and indexing pressure.

### Shard Strategy

- Choose shard count based on expected data volume, query load, and deployment topology.
- Avoid excessive small shards.
- Avoid oversized shards that impair recovery, relocation, and query performance.
- Review shard allocation as tenant count and index volume grow.
- Document shard sizing assumptions and operational thresholds.
- Revisit shard configuration through planned index-version changes where necessary.

Do not prescribe a universal shard size or replica count without considering the actual workload and infrastructure.

### Capacity Planning

Capacity planning must account for:

- Active document count and growth.
- Average and peak document size.
- Indexing throughput.
- Query concurrency.
- Aggregation workload.
- Replica overhead.
- Reindexing and recovery requirements.
- Heap, disk, CPU, and network constraints.
- Tenant growth and deployment topology.

Load tests must represent realistic document distributions and query patterns.

---

## 17. Caching

OpenSearch may benefit from built-in caches, while application-level caching may be implemented with Redis when justified.

### Rules

- Define cache ownership and invalidation behavior.
- Do not assume cached search results are current.
- Include tenant context and all relevant filters in cache identity.
- Do not share cached results across tenants unless the data is explicitly public and safe to share.
- Avoid caching highly personalized or permission-sensitive results without a robust authorization-aware key and invalidation strategy.
- Set bounded TTLs and memory limits.
- Avoid cache stampedes using approved coordination patterns.
- Measure cache hit rates and verify that caching materially improves performance.

Caching must not weaken tenant isolation or authorization.

---

## 18. Error Handling and Resilience

Search is a dependent capability. Temporary OpenSearch failures must not corrupt authoritative business data.

### Failure Categories

| Failure | Required Handling |
|---|---|
| Invalid request | Return a clear client error |
| Unauthorized request | Reject before querying protected data |
| OpenSearch timeout | Apply bounded timeout handling and safe failure response |
| Temporary cluster failure | Retry only where safe and appropriate |
| Bulk indexing partial failure | Inspect per-document results and retry eligible items |
| Mapping conflict | Stop affected writes, alert, and resolve schema incompatibility |
| Event processing failure | Retry with limits and dead-letter unrecoverable events |
| Index unavailable | Return a controlled service error or approved degraded response |
| Stale result | Validate authoritative state for critical operations |

### Resilience Requirements

- Use connection pooling and bounded concurrency.
- Configure request timeouts.
- Apply circuit breaking or equivalent dependency protection where appropriate.
- Avoid unbounded retry loops.
- Use exponential backoff with jitter for retryable failures.
- Apply backpressure to indexing pipelines.
- Define graceful degradation for non-critical search use cases.
- Do not return fabricated or misleading search results when the index is unavailable.

Search degradation must never cause unauthorized access or incorrect business transactions.

---

## 19. Security Standards

OpenSearch must be protected as an internal infrastructure service.

### Authentication and Access

- Require authentication for all cluster and index access.
- Use service-specific identities and least-privilege permissions.
- Restrict administrative APIs to authorized operational identities.
- Do not embed credentials in source code, container images, or frontend bundles.
- Store secrets in an approved secret-management system.
- Use TLS for traffic in transit.
- Restrict network access to approved services and administrative paths.
- Rotate credentials according to the platform's security policy.

### Data Protection

- Index only fields required for approved search use cases.
- Exclude passwords, tokens, secrets, and unnecessary personal information.
- Apply appropriate encryption at rest where supported by the deployment.
- Restrict snapshots and backups to authorized identities.
- Define retention and deletion requirements for indexed data.
- Ensure test and development indices do not contain unapproved production personal data.

### Query Security

- Never accept unrestricted OpenSearch DSL from untrusted clients.
- Validate user input and bound query complexity.
- Disallow arbitrary scripts from public search interfaces.
- Enforce tenant and visibility filters in backend-owned query builders.
- Prevent injection into query structures by using typed request models and safe query construction.
- Rate-limit abusive or expensive search patterns.

### Auditability

Record security-relevant administrative operations, permission changes, and access events according to KAMPYN's audit policy.

Avoid logging credentials, private documents, or sensitive user search content.

---

## 20. Observability and Monitoring

OpenSearch and its indexing pipelines must be observable in production.

### Cluster Metrics

Monitor:

- Cluster health.
- Node availability.
- CPU and memory utilization.
- JVM heap pressure where applicable.
- Disk utilization and disk watermarks.
- Shard allocation and relocation.
- Query and indexing latency.
- Search and indexing rejections.
- Thread-pool queues.
- Segment and merge activity.
- Recovery duration.
- Snapshot success and failure.

### Application Metrics

Monitor:

- Search request count.
- Search latency percentiles.
- Search error rate.
- Timeout rate.
- Empty-result rate.
- Query class and filter distribution.
- Indexing throughput.
- Indexing lag.
- Event processing failures.
- Dead-letter queue depth.
- Reindex progress and duration.
- Reconciliation discrepancies.

### Logging and Tracing

- Use structured logs.
- Include correlation IDs and trace context.
- Record tenant identifiers only where permitted by the data-classification policy.
- Avoid logging raw sensitive queries and document payloads.
- Trace requests across API, event processing, indexing, and search components.
- Make indexing lag and query latency measurable against defined service objectives.

### Alerting

Alerts SHOULD cover:

- Cluster health degradation.
- Disk watermarks or capacity exhaustion.
- Sustained high heap pressure.
- High query latency or error rate.
- Indexing lag exceeding the defined threshold.
- Repeated bulk failures.
- Dead-letter queue growth.
- Failed snapshots.
- Missing or unexpectedly stale indices.
- Reconciliation mismatches.

Thresholds must be set according to deployment-specific service objectives rather than arbitrary universal values.

---

## 21. Backup and Disaster Recovery

OpenSearch indices are derived data, but rebuilding them may be expensive and time-consuming.

### Requirements

- Define snapshot policies for production indices.
- Store snapshots in approved, access-controlled storage.
- Document recovery time and recovery point objectives.
- Test snapshot restoration periodically.
- Validate mappings, aliases, and index settings during recovery.
- Maintain a documented rebuild procedure from authoritative sources.
- Ensure event retention or reconciliation can recover changes that occurred during restoration.
- Define whether each index is restored from snapshots, rebuilt, or handled through a combination of both.

A successful snapshot operation does not, by itself, prove recoverability. Restoration must be tested.

---

## 22. Configuration and Environment Management

OpenSearch configuration must be environment-aware and centrally managed.

### Requirements

- Keep cluster endpoints and credentials outside source code.
- Use environment-specific configuration.
- Validate required settings during service startup.
- Separate local, development, staging, and production configurations.
- Use TLS and authentication in production.
- Avoid exposing administrative endpoints publicly.
- Configure connection limits, timeouts, and retry policies explicitly.
- Keep index mappings, templates, analyzers, and lifecycle policies version-controlled.
- Ensure self-hosted deployments can configure equivalent settings without depending on KAMPYN-managed infrastructure.

### Example Environment Variables

```env
OPENSEARCH_URL=
OPENSEARCH_USERNAME=
OPENSEARCH_PASSWORD=
OPENSEARCH_TLS_ENABLED=true
OPENSEARCH_REQUEST_TIMEOUT_MS=3000
OPENSEARCH_MAX_CONNECTIONS=50
OPENSEARCH_MAX_RETRIES=2
```

These names are examples; the final configuration contract must be consistent across services, deployment manifests, and documentation.

Never commit real credentials or production endpoints containing secrets.

---

## 23. Development Standards

### Code Organization

OpenSearch access must be encapsulated behind domain-oriented search modules.

A suggested structure:

```text
internal/
  search/
    client/
      client.go
      config.go
    items/
      document.go
      mapping.go
      query.go
      repository.go
      service.go
    vendors/
      document.go
      mapping.go
      query.go
      repository.go
      service.go
    indexing/
      consumer.go
      transformer.go
      bulk.go
      retry.go
    migrations/
      registry.go
      aliases.go
    observability/
      metrics.go
      tracing.go
```

This is illustrative and should be adapted to the approved KAMPYN repository architecture.

### Coding Rules

- Keep OpenSearch client initialization centralized.
- Do not create a new client per request.
- Keep query construction separate from HTTP handlers.
- Keep domain transformations explicit and testable.
- Reuse shared indexing and retry infrastructure.
- Avoid duplicated mappings and query definitions.
- Keep modules cohesive and source files normally below 200 lines unless there is documented architectural justification.
- Use context propagation and cancellation for backend operations.
- Handle errors explicitly; do not silently ignore failures.
- Keep index and alias names in centrally managed configuration.
- Avoid exposing OpenSearch-specific types throughout unrelated domain modules.

### Dependency Boundaries

- Domain services must not depend on raw OpenSearch client details.
- Search repositories must encapsulate index and query implementation.
- API handlers must call search services rather than constructing raw DSL.
- Indexing workers must use canonical domain transformations.
- Business services must use authoritative repositories for transactional operations.

---

## 24. Testing Standards

Every production search feature must include appropriate automated tests.

### Unit Tests

Test:

- Query construction.
- Mapping definitions.
- Document transformations.
- Tenant filter injection.
- Visibility filter injection.
- Sort-field allowlists.
- Pagination cursor validation.
- Input validation.
- Error classification.
- Retry and idempotency behavior.

### Integration Tests

Test against a compatible OpenSearch environment:

- Index creation and mapping application.
- Indexing, updating, and deleting documents.
- Full-text matching.
- Structured filtering.
- Sorting and pagination.
- Aggregations and facets.
- Analyzer behavior.
- Alias switching.
- Bulk partial failures.
- Retry behavior.
- Versioned reindexing.
- Search service timeouts and unavailable indices.

### Security Tests

Verify:

- Cross-tenant queries cannot expose data.
- Unauthorized roles cannot discover restricted content.
- Tenant filters apply to aggregations and suggestions.
- Untrusted clients cannot inject raw DSL.
- Hidden or deleted documents do not remain discoverable beyond the defined consistency window.
- Administrative operations require appropriate privileges.

### Performance Tests

Load tests SHOULD measure:

- Median and tail search latency.
- Indexing throughput.
- Concurrent query capacity.
- Bulk indexing performance.
- Aggregation cost.
- Reindex duration.
- Resource consumption under realistic data volumes.
- Degradation under partial cluster failure.

Test datasets should reflect expected production document counts, field cardinality, tenant distribution, and query complexity.

### Relevance Tests

Maintain a representative query set with expected result characteristics.

Evaluate:

- Exact-match behavior.
- Ranking consistency.
- Typo tolerance.
- Synonym behavior.
- Language-specific search.
- Filter correctness.
- No-result and low-result queries.
- Relevance regressions after analyzer, mapping, or ranking changes.

---

## 25. Deployment and Operations

### Deployment Requirements

- Use supported and compatible OpenSearch versions.
- Pin versions for production deployments.
- Define cluster sizing and topology per environment.
- Configure storage, backups, network access, and security before production use.
- Apply index templates and mappings through version-controlled deployment procedures.
- Separate application identities from administrative identities.
- Validate cluster health and index readiness during deployment.
- Define upgrade, rollback, and compatibility procedures.
- Avoid making unreviewed production mapping or cluster-setting changes manually.

### Self-Hosted Universities

KAMPYN must support deployments where universities operate their own infrastructure.

Self-hosting documentation must define:

- Supported OpenSearch version range.
- Minimum and recommended resource requirements.
- Network and TLS requirements.
- Authentication and role configuration.
- Index templates and mappings.
- Snapshot storage configuration.
- Indexing worker connectivity.
- Health checks and operational metrics.
- Upgrade and reindexing procedures.
- Backup, restore, and disaster-recovery procedures.

University-specific infrastructure values must remain configurable. Do not hardcode KAMPYN-managed cluster endpoints into shared packages or SDKs.

---

## 26. Search Analytics and Privacy

Search analytics can help improve discovery but must respect privacy and data minimization.

### Requirements

- Collect only search telemetry that has an approved product or operational purpose.
- Prefer aggregated metrics over storing raw user search histories.
- Define retention periods for search events.
- Restrict access to analytics data.
- Avoid collecting sensitive query content unnecessarily.
- Do not use search telemetry to infer sensitive personal attributes.
- Apply tenant boundaries to tenant-level analytics.
- Document consent or other applicable privacy requirements where relevant.

Search analytics must not become an ungoverned source of user profiling.

---

## 27. Anti-Patterns

The following practices are prohibited unless a documented exception is approved:

- Treating OpenSearch as the source of truth.
- Using OpenSearch to enforce transactions or prevent booking and inventory conflicts.
- Exposing cluster credentials to frontend applications or SDKs.
- Accepting raw query DSL from public clients.
- Omitting tenant filters from tenant-scoped search.
- Relying on client-provided tenant IDs as the sole authority.
- Using dynamic mappings without schema governance.
- Changing production mappings incompatibly in place.
- Creating a new random document ID on every indexing attempt.
- Ignoring out-of-order events and stale writes.
- Dropping failed indexing events without recovery.
- Assuming successful event delivery means successful indexing.
- Using unbounded retries or queues.
- Returning stale search data as guaranteed current business state.
- Allowing arbitrary wildcard or script queries.
- Using deep `from`/`size` pagination without limits.
- Creating excessive shards or indices without capacity planning.
- Indexing complete source records without a search-specific justification.
- Storing secrets or unnecessary sensitive information in documents.
- Sharing cached search results across tenants without explicit safe semantics.
- Performing unvalidated alias switches or destructive index deletion.
- Skipping restore, reindex, and tenant-isolation tests.

---

## 28. Required Documentation

Every production index MUST have a documented specification containing:

- Index name and owner.
- Business purpose and supported search experiences.
- Authoritative source entities.
- Document schema and example.
- Explicit mapping and analyzer configuration.
- Tenant and visibility model.
- Indexing and synchronization mechanism.
- Update and delete behavior.
- Event ordering and idempotency strategy.
- Query types, filters, facets, and sorting.
- Pagination strategy.
- Freshness expectations.
- Reindexing and rollback procedure.
- Retention and deletion requirements.
- Capacity assumptions.
- Monitoring and alerting.
- Backup and recovery expectations.
- Test coverage and relevance evaluation approach.

---

## 29. Review Checklist

Before approving an OpenSearch feature or index, verify:

- [ ] The search use case genuinely requires OpenSearch.
- [ ] The authoritative source of truth is documented.
- [ ] Index ownership and lifecycle are defined.
- [ ] Document IDs are deterministic.
- [ ] Mappings are explicit and versioned.
- [ ] Analyzers are appropriate for the supported languages.
- [ ] Tenant isolation is enforced in every applicable query.
- [ ] Authorization and visibility constraints are applied.
- [ ] Search APIs validate and bound user input.
- [ ] Query complexity, pagination, and aggregations are controlled.
- [ ] Indexing is idempotent and resilient to duplicate events.
- [ ] Out-of-order events and stale writes are handled.
- [ ] Deletes and visibility changes propagate reliably.
- [ ] Backfill, reindexing, and rollback procedures are defined.
- [ ] Search freshness limitations are documented.
- [ ] Performance has been evaluated using representative workloads.
- [ ] Metrics, logs, traces, and alerts are implemented.
- [ ] Credentials, network access, and sensitive data are protected.
- [ ] Backup and disaster-recovery procedures are documented.
- [ ] Unit, integration, security, and relevant performance tests pass.
- [ ] Self-hosted deployment requirements are documented where applicable.

---

## 30. Definition of Done

An OpenSearch feature is complete only when:

1. Its business purpose and index ownership are documented.
2. The source of truth and document transformation are explicit.
3. Mappings and analyzers are version-controlled.
4. Tenant isolation and visibility rules are enforced and tested.
5. Search APIs have validated contracts and bounded query behavior.
6. Indexing is idempotent, retryable, and observable.
7. Update, delete, replay, and reconciliation behavior are defined.
8. Reindexing and rollback procedures are available.
9. Performance and relevance are evaluated against representative workloads.
10. Security, privacy, and operational requirements are satisfied.
11. Automated tests cover expected behavior and important failure modes.
12. Required deployment and self-hosting documentation is complete.

**Final principle:** OpenSearch exists to make KAMPYN data discoverable, not authoritative. Every search document must be traceable to its source, every query must respect tenant and authorization boundaries, and every indexing failure must have a defined recovery path.
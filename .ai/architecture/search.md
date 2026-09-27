# KAMPYN Search Architecture

## 1. Purpose

Search provides fast, relevant discovery across KAMPYN resources without coupling user-facing search directly to transactional database queries.

KAMPYN may need to search:

- food items
- food courts
- vendors
- restaurants
- hostels
- rooms
- amenities
- complaints where authorized
- university services
- announcements
- community content where permitted
- users where authorized
- administrative resources
- other tenant-specific entities

The search architecture must provide:

- low-latency search
- relevance ranking
- typo tolerance
- filtering
- sorting
- faceting
- autocomplete
- pagination
- tenant isolation
- permission-aware results
- scalable indexing
- reliable synchronization
- index rebuilding
- versioned mappings
- observability
- graceful degradation

The core principle is:

> **Search is a derived read model, not the authoritative source of business data.**

---

# 2. Search Position in the Architecture

The high-level architecture is:

```text
                    User
                     │
                     ▼
              Next.js Frontend
                     │
                     ▼
                Search API
                     │
                     ▼
             Search Application
                     │
          ┌──────────┴──────────┐
          │                     │
          ▼                     ▼
   Search Repository       Authorization
          │                     │
          └──────────┬──────────┘
                     ▼
                 OpenSearch
                     ▲
                     │
              Indexing Pipeline
                     ▲
                     │
              Domain Events
                     ▲
                     │
              Source Database
```

The transactional path and search path must remain conceptually separate.

---

# 3. Source of Truth

The authoritative data remains in the appropriate primary datastore.

Example:

```text
Food Item
   │
   ├── Source of Truth → PostgreSQL
   │
   └── Search Projection → OpenSearch
```

Another example:

```text
Vendor
   │
   ├── Source of Truth → PostgreSQL
   │
   └── Search Projection → OpenSearch
```

OpenSearch must not become the authoritative source for transactional state unless a specific architectural decision explicitly establishes it as such.

---

# 4. Search Read Model

Search operates on a denormalized read model.

Example:

```text
PostgreSQL
    │
    │ domain event
    ▼
Indexing Pipeline
    │
    ▼
Search Document
    │
    ▼
OpenSearch
```

A search document may combine information from multiple source entities.

Example:

```json
{
  "id": "item_123",
  "tenant_id": "university_1",
  "type": "food_item",
  "name": "Paneer Biryani",
  "description": "Spiced rice with paneer",
  "vendor_id": "vendor_10",
  "vendor_name": "Campus Kitchen",
  "food_court_id": "court_2",
  "category": "Biryani",
  "price": 180,
  "availability": true,
  "tags": ["paneer", "biryani", "vegetarian"],
  "rating": 4.5
}
```

The document is optimized for search rather than normalized relational storage.

---

# 5. Search Responsibilities

The search subsystem is responsible for:

- query interpretation
- text matching
- relevance ranking
- filtering
- sorting
- pagination
- autocomplete
- faceting
- search result shaping
- index management
- document indexing
- document updates
- document deletion
- index versioning
- synchronization
- search observability

It must not become responsible for:

- authoritative business state
- payment processing
- authorization decisions
- transactional workflows
- inventory mutation
- booking allocation
- order state transitions

---

# 6. Search Architecture Layers

The backend search implementation should follow:

```text
Transport
    │
    ▼
Search Application
    │
    ▼
Search Domain / Query Model
    │
    ▼
Search Repository
    │
    ▼
OpenSearch Infrastructure
```

Indexing follows a separate path:

```text
Database
    │
    ▼
Domain Event / Outbox
    │
    ▼
Indexing Worker
    │
    ▼
Document Builder
    │
    ▼
Search Repository
    │
    ▼
OpenSearch
```

---

# 7. Search API

The API should expose a stable search contract.

Example:

```text
GET /api/v1/search
```

Possible parameters:

```text
q
type
category
vendor
food_court
price_min
price_max
availability
sort
cursor
limit
```

The API should not expose raw OpenSearch queries.

Avoid:

```text
POST /search
{
  "query": {
    "bool": {
      ...
    }
  }
}
```

unless the API is explicitly designed as a low-level administrative search interface.

Normal application APIs should expose controlled domain-level search semantics.

---

# 8. Search Query Model

Search requests should use typed application-level structures.

Example:

```go
type SearchQuery struct {
    TenantID   TenantID
    Text       string
    Types      []SearchType
    Filters    SearchFilters
    Sort       SearchSort
    Cursor     *string
    Limit      int
}
```

The application layer should validate and normalize the request before it reaches OpenSearch.

---

# 9. Search Validation

Validate:

- query length
- query syntax
- allowed entity types
- allowed filters
- allowed sort fields
- pagination limits
- numeric ranges
- enum values
- tenant scope

For example:

```text
limit = 20
```

may be valid while:

```text
limit = 100000
```

must be rejected or bounded.

---

# 10. Search Types

KAMPYN should define explicit searchable entity types.

Example:

```text
food_item
vendor
food_court
hostel
room
service
announcement
community_post
user
```

Not every entity should automatically become searchable.

Searchability should be an explicit domain decision.

---

# 11. Searchable Fields

Each searchable entity must define which fields participate in search.

Example:

```text
Food Item
├── name
├── description
├── category
├── tags
└── vendor name
```

Some fields should be:

```text
searchable
```

while others should only be:

```text
filterable
sortable
aggregatable
display-only
```

Avoid making every field searchable by default.

---

# 12. Field Classification

Each search document field should have a deliberate purpose.

Example:

```text
name
    → searchable
    → sortable

description
    → searchable

category
    → searchable
    → filterable
    → aggregatable

price
    → filterable
    → sortable

tenant_id
    → filterable
    → never user-controlled

created_at
    → sortable
    → filterable

id
    → exact lookup
```

This improves performance and prevents accidental query behavior.

---

# 13. Text Analysis

Search quality depends on appropriate text analysis.

Depending on the entity, the search system may use:

- lowercase normalization
- stemming
- tokenization
- stop-word handling
- synonyms
- language-specific analyzers
- ASCII normalization where appropriate
- edge n-grams for autocomplete

Text analysis should be selected according to actual search requirements.

Do not add analyzers merely because they are available.

---

# 14. Multilingual Search

KAMPYN may operate in environments where users search using multiple languages.

The architecture should allow language-aware analysis.

For example:

```text
English
Hindi
Regional Languages
Transliterated Text
```

Language-specific fields may be used when needed.

Example:

```text
name
name_en
name_hi
name_local
```

The indexing strategy must preserve the original display value while maintaining appropriate searchable representations.

---

# 15. Transliteration

Where required, search may support transliterated input.

For example:

```text
"paneer"
"पनीर"
"paneer"
```

may need to resolve to related content.

This should be implemented deliberately through:

- normalized fields
- transliteration fields
- aliases
- synonyms

Do not rely on uncontrolled fuzzy matching to solve language differences.

---

# 16. Typo Tolerance

Search should tolerate reasonable user typos.

Example:

```text
"panner biryani"
```

may match:

```text
"paneer biryani"
```

Typo tolerance must remain bounded.

Overly aggressive fuzzy matching can produce:

- irrelevant results
- slow queries
- unexpected ranking

Use fuzziness selectively.

---

# 17. Relevance Ranking

Search results should be ranked according to explicit relevance rules.

A typical conceptual ranking may consider:

```text
Exact match
    ↓
Prefix match
    ↓
Strong field match
    ↓
Multi-field match
    ↓
Fuzzy match
```

Additional business-neutral ranking signals may include:

```text
text relevance
availability
category match
location relationship
quality signals
```

Business-specific ranking rules must be documented and tested.

---

# 18. Search Ranking Separation

Search relevance should not silently encode authorization or business policy.

For example:

```text
Search relevance
        ≠
Authorization
```

Authorization determines whether a result may be returned.

Ranking determines how eligible results are ordered.

---

# 19. Authorization-Aware Search

Search results must respect authorization.

Example:

```text
User
 │
 ▼
Authorization Context
 │
 ▼
Search Query
 │
 ▼
OpenSearch
```

Where practical, tenant and visibility filters should be applied during the search query itself.

Never:

```text
Search everything
     ↓
Fetch thousands of results
     ↓
Filter unauthorized results in application memory
```

This is inefficient and can create security risks.

---

# 20. Tenant Filtering

Every tenant-scoped search query must include tenant scope.

Example:

```json
{
  "term": {
    "tenant_id": "tenant_123"
  }
}
```

Tenant scope should be derived from trusted server-side context.

Never accept:

```text
tenant_id=tenant_123
```

from an untrusted client and assume it is valid.

---

# 21. Tenant Index Strategy

Several index strategies may be possible:

### Shared Index

```text
kampyn-items
    ├── tenant A
    ├── tenant B
    └── tenant C
```

Requires strict tenant filtering.

### Tenant-Specific Indices

```text
tenant-a-items
tenant-b-items
tenant-c-items
```

Provides stronger physical separation but increases operational complexity.

### Dedicated Deployment

A self-hosted institution may have its own OpenSearch cluster.

The search abstraction should support these deployment strategies without changing application semantics.

---

# 22. Recommended Default

For SaaS deployments, a shared index with strict tenant filtering can be appropriate when operational scale and isolation requirements permit it.

The architecture must make tenant filtering mandatory rather than relying on individual developers to remember it.

For higher-isolation deployments, dedicated indices or clusters may be used.

The choice should be driven by:

- scale
- isolation requirements
- operational complexity
- cost
- self-hosting requirements

---

# 23. Index Naming

Index names must be predictable and versioned.

Example:

```text
kampyn-items-v1
kampyn-vendors-v1
kampyn-food-courts-v1
```

For tenant-specific strategies:

```text
kampyn-tenant-123-items-v1
```

Index naming must not expose sensitive information.

---

# 24. Index Versioning

Search mappings should be versioned.

Example:

```text
items-v1
items-v2
items-v3
```

Do not mutate production mappings blindly when a change requires incompatible schema behavior.

Prefer:

```text
v1
 │
 ▼
create v2
 │
 ▼
reindex
 │
 ▼
validate
 │
 ▼
switch alias
 │
 ▼
retire v1
```

---

# 25. Index Aliases

Application code should preferably use logical aliases rather than hardcoded physical index versions.

Example:

```text
items-current
      │
      ▼
items-v3
```

During migration:

```text
items-current
      │
      ├── old → items-v2
      │
      └── new → items-v3
```

The application continues using:

```text
items-current
```

while the physical index changes.

---

# 26. Reindexing

Reindexing must be a first-class operational capability.

Possible process:

```text
Source Database
      │
      ▼
Bulk Extraction
      │
      ▼
Document Builder
      │
      ▼
New Index
      │
      ▼
Validation
      │
      ▼
Alias Switch
```

Reindexing should be:

- resumable where practical
- observable
- bounded
- tenant-aware
- version-aware
- safe to retry

---

# 27. Initial Index Creation

When introducing a new searchable entity:

```text
1. Define document
2. Define mapping
3. Create index
4. Build indexing pipeline
5. Backfill existing data
6. Validate documents
7. Enable search
```

Search should not be enabled before the index contains the required data and mappings.

---

# 28. Index Synchronization

Search synchronization should preferably be event-driven.

```text
Primary Database
      │
      ▼
Transactional Outbox
      │
      ▼
Domain Event
      │
      ▼
Indexing Consumer
      │
      ▼
OpenSearch
```

For example:

```text
ItemUpdated
    ↓
ItemSearchIndexer
    ↓
Update item document
```

---

# 29. Why Not Dual Writes?

Avoid:

```text
Application
   ├── write PostgreSQL
   └── write OpenSearch
```

because:

```text
PostgreSQL succeeds
OpenSearch fails
```

can leave the systems inconsistent.

Prefer:

```text
Database Transaction
   ├── business mutation
   └── outbox event
          ↓
      async indexing
```

---

# 30. Eventual Consistency

Search is normally eventually consistent.

Therefore:

```text
Database Updated
      ↓
Event Published
      ↓
Indexer Processes Event
      ↓
Search Updated
```

may take some time.

The application must not assume:

```text
database update
=
immediate search update
```

unless the specific workflow guarantees otherwise.

---

# 31. Search Consistency

For critical workflows, the authoritative database must always be consulted when search data cannot safely determine current state.

For example:

```text
Search
  → item found
  ↓
Order creation
  → validate current item/price/availability
  ↓
Database
```

Never trust search results as authoritative transactional state.

---

# 32. Availability

Search may index fields such as:

```text
available
open
active
```

but these values may become stale.

For a transaction:

```text
Search says available
       ↓
Database says unavailable
```

the database wins.

Search is for discovery.

Transactional systems are for decisions that require current state.

---

# 33. Search Filters

Filters should be typed and controlled.

Examples:

```text
category
vendor
food_court
price
rating
availability
location
type
```

Filters should use exact or appropriate field types rather than full-text matching where possible.

---

# 34. Facets and Aggregations

Search may provide facets such as:

```text
Category
Vendor
Food Court
Price Range
Rating
```

Example:

```text
Search Results
├── 42 items
├── Categories
│   ├── Biryani: 12
│   ├── Snacks: 18
│   └── Drinks: 12
└── Vendors
    ├── Vendor A: 20
    └── Vendor B: 22
```

Aggregations should be bounded and designed for actual UI requirements.

Avoid expensive unrestricted aggregations.

---

# 35. Sorting

Supported sort options should be explicit.

Examples:

```text
relevance
price_asc
price_desc
rating_desc
newest
```

Do not accept arbitrary field names from clients.

Sort fields must come from a server-defined allowlist.

---

# 36. Pagination

Search should use efficient pagination.

For normal result browsing:

```text
cursor
limit
```

is preferred for deep pagination.

Avoid large offsets such as:

```text
from=1000000
```

for high-volume indices.

Use cursor/search-after style mechanisms where appropriate.

---

# 37. Autocomplete

Autocomplete should be optimized separately from full search.

Example:

```text
User types:
"pan"

        ↓

Autocomplete

        ↓

Paneer Biryani
Paneer Roll
Paneer Tikka
```

Autocomplete should prioritize:

- low latency
- prefix matching
- relevant suggestions
- bounded result count

It should not execute the full search pipeline unnecessarily.

---

# 38. Suggestion Sources

Autocomplete may use:

```text
entity names
categories
vendors
food courts
popular search terms
```

Search suggestions should not expose unauthorized entities.

---

# 39. Search History

If KAMPYN stores search history, it must be treated as user-related data.

Requirements include:

- authorization
- retention
- deletion
- privacy
- tenant isolation
- minimal collection

Search history should not automatically be used as a ranking signal without an explicit product decision.

---

# 40. Search Analytics

Search analytics may record aggregated events such as:

```text
query
result_count
latency
zero_result
selected_result
entity_type
```

Sensitive user input must be handled carefully.

Search analytics should avoid storing unnecessary private content.

---

# 41. Zero-Result Searches

Zero-result searches should be measurable.

Track:

```text
zero_result_count
zero_result_rate
```

This can identify:

- missing content
- synonym gaps
- spelling issues
- indexing failures
- poor query interpretation

Zero-result analytics should be aggregated and privacy-aware.

---

# 42. Search Query Normalization

Before querying OpenSearch, normalize where appropriate:

```text
trim whitespace
normalize casing
normalize supported Unicode forms
remove meaningless repeated whitespace
validate length
```

Do not aggressively rewrite user queries in ways that change meaning.

---

# 43. Search Query Security

Never construct raw OpenSearch queries directly from untrusted client input.

Bad:

```text
client query
    ↓
raw OpenSearch DSL
```

Prefer:

```text
client query
    ↓
validation
    ↓
typed SearchQuery
    ↓
query builder
    ↓
OpenSearch DSL
```

This prevents query injection and uncontrolled resource usage.

---

# 44. Query Complexity Limits

Search queries must have resource boundaries.

Limit:

```text
query length
filter count
facet count
result count
sort options
wildcard usage
fuzzy distance
regex usage
```

Avoid exposing unrestricted wildcard or regular-expression queries to normal users.

---

# 45. Search Timeouts

Search requests should have bounded timeouts.

If OpenSearch becomes slow:

```text
API
  │
  ▼
Search
  │
  └── timeout
       ↓
controlled error
```

Do not allow indefinitely running searches to consume connections and resources.

---

# 46. Search Failure Behavior

Search is generally not the source of truth.

If OpenSearch becomes unavailable:

```text
OpenSearch unavailable
       │
       ▼
Search request fails gracefully
```

Critical transactional workflows should continue where possible.

Depending on product requirements, the application may provide:

```text
fallback database search
```

for limited use cases.

A database fallback must not become an accidental permanent architecture.

---

# 47. Database Fallback

A fallback may be appropriate for small datasets or critical administrative functionality.

Example:

```text
Search
 │
 ├── OpenSearch available
 │      ↓
 │   OpenSearch
 │
 └── OpenSearch unavailable
        ↓
    Controlled fallback
```

The fallback must have:

- strict limits
- explicit query scope
- timeout
- observability
- no unbounded table scans

---

# 48. Search Caching

Frequently repeated search queries may be cached.

However, caching must account for:

```text
tenant
authorization
query
filters
sort
pagination
index version
```

Cache keys must not allow cross-tenant or cross-user data leakage.

For highly personalized searches, caching should be used carefully.

---

# 49. Search Cache Invalidation

Search cache invalidation should account for index updates.

A cache key may include:

```text
search:v3:
tenant:
query:
filters:
sort:
cursor:
```

Index versioning can naturally help invalidate older cached results.

---

# 50. Search Repository

The search repository should abstract OpenSearch.

Example:

```go
type ItemSearchRepository interface {
    Search(ctx context.Context, query SearchQuery) (SearchResult, error)
    Index(ctx context.Context, document ItemDocument) error
    Delete(ctx context.Context, id ItemID) error
}
```

Application code should not construct OpenSearch DSL directly.

---

# 51. Search Document Builder

Document construction should be separated from the OpenSearch client.

```text
Domain / Source Data
       │
       ▼
Document Builder
       │
       ▼
Search Document
       │
       ▼
OpenSearch Repository
```

This makes indexing logic testable independently from the search infrastructure.

---

# 52. Document Denormalization

Search documents may denormalize data for performance.

Example:

```text
Item
 ├── vendor_id
 ├── vendor_name
 ├── food_court_id
 ├── food_court_name
 └── category_name
```

This avoids expensive joins during search.

However, denormalized fields create synchronization responsibilities.

---

# 53. Denormalization Updates

If a vendor name changes:

```text
VendorUpdated
      │
      ▼
Search Indexer
      │
      ▼
Update affected documents
```

The indexing architecture must understand these dependencies.

Avoid leaving stale denormalized data indefinitely.

---

# 54. Partial Updates

When possible, update only affected fields rather than rebuilding large documents unnecessarily.

However, correctness takes priority over micro-optimization.

If a document is inexpensive to rebuild, a full replacement may be simpler and safer.

---

# 55. Delete Semantics

When an entity is deleted or becomes non-searchable:

```text
EntityDeleted
     │
     ▼
Search Indexer
     │
     ▼
Remove / hide document
```

Soft-deleted entities must not remain discoverable unintentionally.

Visibility changes should also trigger indexing updates.

---

# 56. Reconciliation

The search system should periodically be reconcilable with the source of truth.

Possible process:

```text
Source Database
      │
      ▼
Expected Documents
      │
      ├── compare
      │
      ▼
OpenSearch
```

Reconciliation can detect:

- missing documents
- stale documents
- duplicate documents
- incorrect tenant IDs
- incorrect visibility
- indexing failures

---

# 57. Dead-Letter Handling

Failed indexing events should not disappear.

Example:

```text
Event
  │
  ▼
Indexer
  │
  ├── success
  │
  └── failure
        │
        ▼
      Retry
        │
        ▼
   Dead Letter
```

Dead-letter events should contain enough metadata to diagnose the failure without exposing sensitive data.

---

# 58. Retry Strategy

Indexing retries should use bounded exponential backoff.

Consider:

```text
attempt 1
   ↓
attempt 2
   ↓
attempt 3
   ↓
dead letter
```

Do not retry indefinitely.

Permanent failures should be distinguished from transient failures.

---

# 59. Idempotent Indexing

Indexing operations should be idempotent.

For example:

```text
Index Item 123
```

executed multiple times should result in the same effective search document.

Use stable document IDs.

Avoid generating a new random document ID on every indexing attempt.

---

# 60. Ordering

Search indexing may receive events out of order.

Example:

```text
ItemUpdated version 5
ItemUpdated version 6
ItemUpdated version 5
```

The system must avoid allowing an older event to overwrite newer state.

Possible approaches include:

```text
version numbers
sequence numbers
timestamps where appropriate
source-of-truth refresh
```

Version-based approaches are preferable where ordering matters.

---

# 61. Event Versioning

Search consumers must handle event schema versions.

Example:

```text
ItemUpdated.v1
ItemUpdated.v2
```

The indexer should support compatible event evolution.

Breaking event changes must follow the event architecture and compatibility strategy.

---

# 62. Search Observability

Search operations must emit telemetry.

Measure:

```text
search requests
search latency
search errors
zero-result rate
result count
indexing latency
indexing failures
retry count
dead-letter count
```

Useful dimensions include:

```text
entity type
operation
environment
index
```

Avoid high-cardinality metric labels such as raw query strings.

---

# 63. Search Logs

Useful structured search logs include:

```text
operation
entity_type
index
duration
result_count
error_code
tenant_id
trace_id
```

Do not automatically log complete search queries when they may contain sensitive user input.

---

# 64. Search Tracing

A search request should be traceable:

```text
HTTP Request
     │
     ▼
Search Application
     │
     ▼
Search Repository
     │
     ▼
OpenSearch
```

Indexing should also be traceable:

```text
Domain Event
     │
     ▼
Indexing Worker
     │
     ▼
Document Builder
     │
     ▼
OpenSearch
```

---

# 65. Search Metrics

Important metrics include:

```text
search_request_total
search_error_total
search_duration
search_zero_result_total
search_result_count
search_indexing_total
search_indexing_error_total
search_indexing_duration
search_retry_total
search_dead_letter_total
```

Metrics must use controlled labels.

---

# 66. Search Performance

Search performance should be measured using realistic workloads.

Monitor:

```text
p50
p95
p99
```

for:

- autocomplete
- normal search
- filtered search
- faceted search
- indexing
- bulk indexing

Performance testing should include realistic tenant and dataset sizes.

---

# 67. OpenSearch Capacity

Monitor:

```text
CPU
memory
heap
disk
shard count
index size
search queue
indexing queue
rejected requests
cluster health
```

Search capacity planning must consider:

```text
documents
document size
query rate
indexing rate
replicas
retention
tenant growth
```

---

# 68. Sharding

Sharding must be based on actual scale.

Do not create excessive shards simply because the system may grow.

Poor shard design can increase:

- memory consumption
- cluster overhead
- query latency
- recovery time

Shard strategy should be documented and reviewed as data volume grows.

---

# 69. Replicas

Replicas provide availability and search capacity.

The appropriate number depends on:

```text
availability requirements
query volume
cluster size
failure tolerance
cost
```

Replica configuration must be compatible with self-hosted deployments.

---

# 70. Index Lifecycle

Search indices may require lifecycle policies.

Consider:

```text
active
read-only
archived
deleted
```

for data with retention requirements.

Transactional entities that remain searchable may require different lifecycle policies from logs or analytics data.

---

# 71. Search and Analytics Separation

Search indexes should not automatically become the analytics warehouse.

If analytics requires:

```text
large historical datasets
complex aggregations
financial reporting
business intelligence
```

use the appropriate analytics architecture.

Search should remain optimized for interactive discovery.

---

# 72. Search and Recommendations

Search and recommendations are different systems.

```text
Search
    → user query driven

Recommendation
    → system-generated suggestions
```

Recommendation logic should not be hidden inside generic search ranking unless explicitly designed that way.

---

# 73. Search and Personalization

If search becomes personalized:

```text
User
  │
  ▼
Search Query
  │
  ├── relevance
  ├── permissions
  └── personalization
```

Personalization must not override authorization.

The system should also carefully consider caching because personalized results are more difficult to cache safely.

---

# 74. Search and Location

For location-aware searches, use explicit geographic fields.

Example:

```text
location
 ├── latitude
 └── longitude
```

Possible queries:

```text
nearby vendors
nearest food court
services within radius
```

Location filters must be tenant-aware and privacy-conscious.

---

# 75. Search and Availability

Availability may be indexed for discovery:

```text
available = true
```

but real-time availability must be confirmed against the authoritative system before a transaction.

Example:

```text
Search
  → Item available

Order
  → Check current inventory
  → Reserve inventory
```

---

# 76. Search and Pricing

Search may display indexed prices.

However, transactional workflows must validate the current price from the authoritative datastore.

Do not allow:

```text
OpenSearch price
      ↓
payment amount
```

without authoritative verification.

---

# 77. Search and Permissions

Some content may have visibility rules such as:

```text
public
tenant
role
department
private
moderated
```

Search documents should contain enough visibility metadata to filter appropriately.

Sensitive content should not simply be indexed and filtered later.

---

# 78. Community Search

If community content becomes searchable:

```text
Community Post
     │
     ├── visibility
     ├── tenant
     ├── author
     ├── moderation state
     └── content
```

must be considered.

Private or deleted content must not remain searchable.

Search indexing must respond to:

```text
post deletion
moderation
visibility change
user deletion
tenant suspension
```

---

# 79. Search and User Deletion

When a user is deleted or anonymized, search indexes must be updated accordingly.

Possible flow:

```text
User Deletion
      │
      ├── primary data
      ├── cached data
      ├── search documents
      └── derived analytics where applicable
```

Search is part of the derived-data deletion strategy.

---

# 80. Search and Tenant Deletion

When a tenant is decommissioned:

```text
Tenant Decommission
       │
       ├── disable search
       ├── remove / isolate indices
       ├── stop indexing jobs
       ├── clear tenant caches
       └── verify deletion
```

No tenant data should remain discoverable after the required deletion process completes.

---

# 81. Self-Hosted Search

Self-hosted universities should be able to deploy the search subsystem independently.

A deployment may look like:

```text
KAMPYN
├── Frontend
├── API
├── Workers
├── PostgreSQL
├── MongoDB
├── Redis
└── OpenSearch
```

The application should configure OpenSearch through environment/configuration rather than hardcoded infrastructure assumptions.

---

# 82. Search Configuration

Configuration may include:

```text
OPENSEARCH_URL
OPENSEARCH_USERNAME
OPENSEARCH_PASSWORD
OPENSEARCH_INDEX_PREFIX
OPENSEARCH_TLS_ENABLED
SEARCH_TIMEOUT
SEARCH_MAX_RESULTS
SEARCH_DEFAULT_PAGE_SIZE
```

Secrets must be supplied through secure secret management.

---

# 83. Search Availability During Deployment

Index migrations must not require unnecessary application downtime.

Prefer:

```text
Create new index
       ↓
Backfill
       ↓
Validate
       ↓
Switch alias
       ↓
Continue serving
       ↓
Retire old index
```

This supports zero-downtime search schema migrations.

---

# 84. Search Testing

Search requires multiple levels of testing.

### Unit Tests

Test:

```text
query normalization
filter construction
ranking configuration
document mapping
error mapping
```

### Integration Tests

Test:

```text
actual OpenSearch queries
mappings
filters
sorting
pagination
aggregations
indexing
deletion
```

### Contract Tests

Test:

```text
Search API contract
search result schema
indexing event schema
```

### Performance Tests

Test:

```text
realistic dataset
concurrent queries
large indexes
autocomplete load
bulk indexing
```

---

# 85. Search Security Testing

Test:

```text
tenant isolation
authorization filtering
query injection
wildcard abuse
regex abuse
large query limits
large result limits
private content exposure
deleted content exposure
```

A critical security invariant is:

```text
Unauthorized data must never appear in search results.
```

---

# 86. Search Failure Testing

Test:

```text
OpenSearch unavailable
OpenSearch timeout
index missing
index corrupted
mapping mismatch
indexing event duplicated
indexing event delayed
event delivered out of order
dead-letter accumulation
```

The application should fail predictably.

---

# 87. Search Rebuild Testing

Index rebuilds should be tested before production.

Verify:

```text
source document count
indexed document count
tenant distribution
missing documents
duplicate documents
visibility
mapping correctness
search relevance
```

Alias switching should be tested as an operational procedure.

---

# 88. Search Data Integrity

The following relationship should remain true:

```text
Primary Data
      │
      ▼
Expected Search Projection
      │
      ▼
Actual Search Projection
```

Differences must be detectable and recoverable.

Search consistency should therefore be treated as an operational invariant rather than assumed.

---

# 89. Definition of Done

A searchable entity is complete when:

- [ ] Searchable fields are explicitly defined.
- [ ] Search document is documented.
- [ ] Source of truth is identified.
- [ ] Tenant isolation is enforced.
- [ ] Authorization rules are defined.
- [ ] Index mapping exists.
- [ ] Index versioning exists.
- [ ] Indexing events exist.
- [ ] Indexing is idempotent.
- [ ] Deletes are handled.
- [ ] Reindexing is supported.
- [ ] Search API is validated.
- [ ] Pagination is bounded.
- [ ] Sorting is allowlisted.
- [ ] Query complexity is bounded.
- [ ] Observability exists.
- [ ] Security tests exist.
- [ ] Integration tests exist.
- [ ] Performance has been evaluated.
- [ ] Reconciliation is possible.

---

# 90. Final Principle

KAMPYN search should follow:

```text
                    SOURCE OF TRUTH
                           │
                ┌──────────┴──────────┐
                │                     │
           PostgreSQL              MongoDB
                │                     │
                └──────────┬──────────┘
                           │
                    Domain Events
                           │
                    Transactional
                       Outbox
                           │
                           ▼
                  Indexing Workers
                           │
                           ▼
                   Document Builder
                           │
                           ▼
                      OpenSearch
                           │
                  ┌────────┴────────┐
                  │                 │
                  ▼                 ▼
              Search API       Autocomplete
                  │
                  ▼
              Frontend
```

The core rules are:

> **OpenSearch is a derived search system, not the source of truth.**

> **Every tenant-scoped search must enforce tenant isolation.**

> **Authorization must be enforced before results are exposed, not after retrieving unrestricted data.**

> **Search must be optimized for discovery; transactional operations must use authoritative data.**

> **Indexing should be asynchronous, idempotent, observable, and recoverable.**

> **Search schemas must be versioned and rebuildable.**

> **Search queries must be typed, bounded, validated, and protected against resource abuse.**

> **Denormalization is acceptable for search performance, but synchronization becomes an explicit architectural responsibility.**

> **A search failure must not compromise transactional correctness.**

> **The search layer should remain replaceable without changing the business logic of KAMPYN.**
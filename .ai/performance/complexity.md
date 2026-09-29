# Complexity Analysis and Optimization

## 1. Purpose

This document defines KAMPYN's standards for analyzing, designing, reviewing, and optimizing the computational complexity of software systems.

Every algorithm, data structure, database operation, API endpoint, background job, and frontend interaction must be designed with its computational cost, memory consumption, scalability, and expected workload in mind.

The objective is to ensure KAMPYN remains responsive, resource-efficient, maintainable, and scalable as the number of universities, tenants, students, staff, vendors, orders, bookings, and concurrent requests grows.

Complexity optimization must be driven by measurable requirements and realistic workloads. An algorithm with a theoretically better complexity is not automatically preferable if it increases memory usage, operational overhead, or implementation risk without a meaningful practical benefit.

## 2. Core Principles

All production code MUST follow these principles:

- **Analyze before optimizing:** Understand the workload, bottleneck, input size, and performance requirements before changing an implementation.
- **Choose appropriate data structures:** Use data structures that match access patterns, update frequency, ordering requirements, and memory constraints.
- **Avoid unnecessary quadratic operations:** Do not introduce avoidable \(O(n^2)\) or worse behavior for workloads expected to grow.
- **Bound resource usage:** Every operation processing potentially large input must have appropriate limits, pagination, streaming, or chunking.
- **Optimize end-to-end behavior:** Include network calls, database queries, serialization, memory allocation, and concurrency in performance analysis.
- **Prefer predictable performance:** Avoid algorithms whose worst-case behavior can cause unacceptable latency or resource exhaustion.
- **Measure real workloads:** Use profiling, benchmarks, query plans, and production metrics to validate performance.
- **Maintain correctness:** Performance improvements must preserve business invariants, authorization, tenant isolation, and data integrity.
- **Avoid premature complexity:** Do not introduce advanced algorithms, caching, parallelism, or distributed coordination without a justified need.
- **Document meaningful trade-offs:** Record complexity, resource costs, and limitations for performance-critical components.

## 3. Complexity Fundamentals

Complexity describes how the resource requirements of an operation grow as its input size increases.

### 3.1 Time Complexity

Time complexity describes how an algorithm's execution cost scales with input size.

| Complexity | Name | Typical behavior | Example |
|---|---|---|---|
| \(O(1)\) | Constant | Independent of input size | Hash-map lookup, on average |
| \(O(\log n)\) | Logarithmic | Grows slowly as input increases | Binary search |
| \(O(n)\) | Linear | Grows proportionally to input size | Single-pass traversal |
| \(O(n \log n)\) | Linearithmic | Common efficient sorting growth | Merge sort |
| \(O(n^2)\) | Quadratic | Pairwise growth | Comparing every pair |
| \(O(n^3)\) | Cubic | Triple nested growth | Naive matrix multiplication |
| \(O(2^n)\) | Exponential | Doubles with each added input | Exhaustive subset enumeration |
| \(O(n!)\) | Factorial | Rapid combinatorial growth | Exhaustive permutation enumeration |

These are asymptotic growth categories. Actual execution time depends on constants, implementation details, hardware, data distribution, and runtime overhead.

### 3.2 Space Complexity

Space complexity describes how memory requirements grow with input size.

Examples:

- \(O(1)\): Fixed auxiliary storage.
- \(O(n)\): An additional array or hash map proportional to input size.
- \(O(n^2)\): A matrix or pairwise relationship table.
- \(O(\log n)\): A recursive call stack for a balanced divide-and-conquer algorithm.

Distinguish between:

- **Input space:** Memory occupied by the input itself.
- **Auxiliary space:** Additional memory used by the algorithm.
- **Total space:** Input, auxiliary data, runtime overhead, and retained objects.

For KAMPYN, memory consumption is particularly important for large menu catalogs, order histories, analytics, search results, and bulk data processing.

### 3.3 Best, Average, and Worst Cases

Complexity analysis should consider the relevant input distribution.

- **Best case:** The least work required for a valid input of a given size.
- **Average case:** Expected work under an explicitly stated distribution.
- **Worst case:** The maximum work required for a valid input of a given size.

Use worst-case analysis for operations exposed to untrusted input, security-sensitive paths, and workloads where latency spikes are unacceptable.

Average-case analysis may be useful for common application workloads, but its assumptions must be documented.

### 3.4 Amortized Complexity

Amortized analysis measures the average cost per operation over a sequence of operations, even when individual operations can be expensive.

For example, dynamically growing an array may occasionally require copying its elements, but appending is generally amortized \(O(1)\).

Do not mistake amortized guarantees for worst-case guarantees when designing latency-sensitive operations.

## 4. Complexity Budgets

Performance-critical code must have complexity expectations appropriate to its workload.

| Workload | Preferred target | Notes |
|---|---|---|
| Identifier lookup | \(O(1)\) average or \(O(\log n)\) | Hash maps or indexed lookup |
| Single collection traversal | \(O(n)\) | Prefer one-pass processing |
| Sorting records | \(O(n \log n)\) | Use an appropriate stable or unstable sort |
| Deduplication | \(O(n)\) average | Use a set or hash map |
| Grouping records | \(O(n)\) average | Use a map keyed by group |
| Pairwise comparisons | Avoid when possible | Consider indexing, hashing, or sorting |
| Paginated API retrieval | Bounded per page | Avoid loading entire collections |
| Large file processing | Streaming or bounded chunks | Avoid full-file memory loading |
| Graph traversal | \(O(V + E)\) | Use BFS or DFS as appropriate |
| Database filtering | Index-supported where justified | Validate with query plans |
| Search | Index-backed where appropriate | Avoid repeated full scans |
| Repeated aggregation | Precompute or cache if justified | Define freshness requirements |

These targets are design guidelines, not unconditional guarantees. Data structures, access patterns, consistency requirements, and actual input sizes determine the appropriate solution.

### 4.1 Complexity Review Thresholds

The following rules apply to production code:

- Avoid avoidable \(O(n^2)\) and higher complexity when input size can grow significantly.
- Any \(O(n^2)\) operation on user-controlled or potentially large input requires explicit justification.
- Any exponential or factorial algorithm must have a strict, enforced input bound and a documented use case.
- Unbounded recursion is prohibited.
- Unbounded collection growth is prohibited for long-lived processes.
- Expensive operations must have timeouts, workload limits, or other suitable resource controls.

A higher-complexity solution may be acceptable for demonstrably small, fixed-size input, provided the input bound is enforced and documented.

## 5. Data Structure Selection

Choosing the right data structure is often more effective than optimizing the algorithm around an unsuitable one.

### 5.1 Common Data Structures

| Data structure | Typical lookup | Insert | Delete | Appropriate use |
|---|---:|---:|---:|---|
| Array / Slice | \(O(n)\) | \(O(1)\) amortized append | \(O(n)\) arbitrary removal | Sequential data and iteration |
| Linked list | \(O(n)\) | \(O(1)\) with node | \(O(1)\) with node | Frequent local insertions with retained node references |
| Hash map | \(O(1)\) average | \(O(1)\) average | \(O(1)\) average | Key-based retrieval and grouping |
| Hash set | \(O(1)\) average | \(O(1)\) average | \(O(1)\) average | Membership checks and deduplication |
| Balanced tree | \(O(\log n)\) | \(O(\log n)\) | \(O(\log n)\) | Ordered lookup and range queries |
| Heap | \(O(1)\) peek | \(O(\log n)\) | \(O(\log n)\) extract | Priority queues |
| Queue | \(O(1)\) enqueue/dequeue | \(O(1)\) | \(O(1)\) | FIFO processing |
| Stack | \(O(1)\) top | \(O(1)\) push | \(O(1)\) pop | LIFO processing |
| Trie | \(O(k)\) | \(O(k)\) | Depends on implementation | Prefix lookup for bounded key length |
| Graph adjacency list | \(O(V+E)\) traversal | Workload-dependent | Workload-dependent | Relationship and dependency traversal |

These are typical complexity bounds. Actual performance depends on implementation, collisions, balancing, allocation, and the operations being performed.

### 5.2 Selection Rules

- Use arrays or slices for sequential traversal and compact data storage.
- Use maps for repeated lookups by stable identifiers.
- Use sets for membership checks and deduplication.
- Use balanced trees when ordered access and range queries are required.
- Use heaps for prioritized work queues.
- Use queues for FIFO workloads.
- Use graphs when relationships and traversal are first-class domain requirements.
- Avoid custom data structures unless standard structures cannot meet documented requirements.
- Consider memory overhead and cache locality, not just asymptotic complexity.

### 5.3 Avoid Linear Membership Checks in Loops

Inefficient:

```ts
const existingIds = ["a", "b", "c", "d"];

const missing = incomingItems.filter(
  (item) => !existingIds.includes(item.id)
);
```

If both collections grow, repeatedly scanning the existing list can result in \(O(n \times m)\) work.

Preferred:

```ts
const existingIdSet = new Set(existingIds);

const missing = incomingItems.filter(
  (item) => !existingIdSet.has(item.id)
);
```

Building the set requires \(O(m)\) average time, and the subsequent membership checks require \(O(n)\) average time, resulting in \(O(n + m)\) average total time.

## 6. Loop and Collection Optimization

### 6.1 Avoid Unnecessary Nested Loops

Nested loops are not inherently inefficient, but their total complexity must be understood.

Inefficient:

```ts
for (const order of orders) {
  for (const item of inventory) {
    if (order.itemId === item.id) {
      order.inventory = item;
    }
  }
}
```

This performs \(O(n \times m)\) comparisons.

Preferred:

```ts
const inventoryById = new Map(
  inventory.map((item) => [item.id, item])
);

for (const order of orders) {
  order.inventory = inventoryById.get(order.itemId);
}
```

This reduces average lookup work to \(O(n + m)\), at the cost of additional \(O(m)\) memory.

### 6.2 Avoid Repeated Array Copies

Avoid copying an entire collection repeatedly inside a loop when a single mutable local accumulator or a more suitable data structure would suffice.

Inefficient:

```ts
const result = items.reduce(
  (acc, item) => [...acc, transform(item)],
  []
);
```

Repeated copying can cause quadratic total work.

Preferred:

```ts
const result = [];

for (const item of items) {
  result.push(transform(item));
}
```

Use immutable transformations where they improve correctness and maintainability, but avoid repeated large allocations without a reason.

### 6.3 Combine Compatible Passes

When operations can safely be combined, consider processing the collection once instead of traversing it repeatedly.

However, do not combine unrelated responsibilities into a single difficult-to-maintain function merely to save a pass.

Clarity and cohesive module design remain mandatory.

### 6.4 Avoid Redundant Computation

- Calculate reusable values once when their inputs remain unchanged.
- Avoid repeated parsing and serialization of the same value.
- Avoid recalculating derived properties within inner loops.
- Reuse validated lookup maps within a bounded operation.
- Avoid retaining derived data beyond its useful lifetime.

Memoization should be introduced only when repeated computation is significant and cache invalidation or memory retention is manageable.

## 7. Searching and Sorting

### 7.1 Searching

Select the search strategy based on data organization and query requirements.

| Strategy | Typical complexity | Use case |
|---|---|---|
| Linear search | \(O(n)\) | Small or unsorted collections |
| Binary search | \(O(\log n)\) | Sorted random-access collections |
| Hash lookup | \(O(1)\) average | Exact identifier or key lookup |
| Balanced-tree lookup | \(O(\log n)\) | Ordered and range-based lookup |
| Trie lookup | \(O(k)\) | Prefix matching |
| Inverted index | Query-dependent | Full-text search |

Binary search requires sorted data and a compatible ordering. Do not apply it to unsorted collections without first sorting or maintaining an ordered representation.

### 7.2 Sorting

- Prefer standard-library sorting implementations.
- Choose stable sorting when equal-key ordering must be preserved.
- Avoid sorting when only the minimum, maximum, or top \(k\) records are needed.
- Use bounded heaps or selection algorithms for top-\(k\) workloads when appropriate.
- Avoid repeatedly sorting a collection after each individual insertion when batch processing or an ordered structure is more suitable.

### 7.3 Top-K Retrieval

If a request requires only the highest-ranked \(k\) elements from \(n\) records, sorting the entire collection may perform unnecessary work.

Depending on the workload, use:

- A bounded heap with approximately \(O(n \log k)\) time and \(O(k)\) auxiliary space.
- A selection algorithm with expected linear time where its behavior and implementation are appropriate.
- Database-side ordering and limits when the data already resides in a database and the query plan is efficient.

Choose based on input size, ordering guarantees, memory budget, and whether the data is already loaded.

## 8. Database Complexity

Database operations are part of application complexity and must be reviewed with the same rigor as in-memory algorithms.

### 8.1 Query Design

- Retrieve only the required columns.
- Filter as close to the data source as practical.
- Use suitable indexes for frequent filters, joins, and ordering.
- Avoid unnecessary full-table scans on large datasets.
- Use bounded pagination for large result sets.
- Batch related queries when doing so reduces round trips without creating excessive payloads.
- Avoid loading complete records when only existence or aggregate information is required.
- Review query plans for performance-sensitive queries.
- Ensure query constraints include tenant scope.

### 8.2 N+1 Queries

N+1 query patterns can cause \(O(n)\) database round trips for \(n\) records, creating significant network and database overhead.

Instead of issuing a separate query for every record:

- Use a join or eager-loading strategy where appropriate.
- Batch related identifiers into a bounded query.
- Use a DataLoader-style batching pattern for repeated resolver access.
- Select only necessary fields.
- Avoid over-fetching large relationships.

Batch size must be bounded to prevent oversized SQL statements, payloads, and memory usage.

### 8.3 Pagination

Use pagination for potentially large datasets.

- Prefer cursor-based pagination for large, frequently changing datasets when suitable.
- Use deterministic ordering with a unique tie-breaker.
- Avoid large offset scans when query cost grows with the offset.
- Enforce maximum page sizes.
- Include tenant and authorization filters in every paginated query.
- Ensure cursors cannot be manipulated to bypass data-access restrictions.

Pagination reduces per-request workload but does not automatically make an expensive query efficient. Indexing and query plans must still be evaluated.

### 8.4 Aggregation

For expensive aggregations:

- Use database-side aggregation when appropriate.
- Use indexes and partitioning only when justified by measured access patterns.
- Consider materialized views for repeated stable computations.
- Precompute frequently requested metrics when freshness requirements permit.
- Avoid transferring large datasets to the application solely to aggregate them.
- Apply tenant and authorization constraints before returning aggregate results.

### 8.5 Query Complexity Is Not Just Big-O

Database query cost depends on:

- Table and index size.
- Selectivity.
- Join strategy.
- Index coverage.
- Sorting and grouping.
- Disk and memory access.
- Lock contention.
- Network round trips.
- Concurrent workload.
- Data distribution and statistics.

Use database execution plans and workload measurements rather than estimating query performance from SQL syntax alone.

## 9. API Complexity and Resource Limits

API endpoints must have predictable resource consumption.

### 9.1 Request Processing

Each endpoint must account for:

- Request parsing.
- Validation.
- Authentication.
- Authorization.
- Database access.
- Serialization.
- External service calls.
- Memory allocation.
- Response size.

Avoid doing repeated validation, parsing, or database lookups when a validated result can safely be reused within the request.

### 9.2 Input Limits

Enforce limits for:

- Request body size.
- Array and batch lengths.
- Page sizes.
- Search query length.
- Filter count and complexity.
- Upload size.
- Concurrent operations.
- Export and report workloads.
- Nested request depth where relevant.

Input limits protect the system from accidental overload and algorithmic denial-of-service attacks.

### 9.3 Bounded Work

Expensive API operations must use one or more of the following:

- Pagination.
- Chunking.
- Streaming.
- Asynchronous background processing.
- Rate limiting.
- Concurrency limits.
- Query timeouts.
- Maximum input sizes.
- Explicit workload quotas.

A request must not be allowed to trigger unbounded CPU, memory, database, or external-service consumption.

### 9.4 External Services

Complexity analysis must include external calls.

- Avoid sequential calls when independent calls can be safely batched or run concurrently.
- Apply concurrency limits.
- Use request timeouts.
- Bound retries and exponential backoff.
- Avoid retrying non-idempotent operations without an idempotency mechanism.
- Prefer provider-side batch APIs when they meet correctness requirements.
- Prevent one slow dependency from blocking unrelated work.

Parallel execution can reduce wall-clock latency but may increase resource usage and downstream load.

## 10. Streaming and Large Data Processing

Large datasets must not be loaded entirely into memory unless the maximum size is explicitly bounded and the memory budget supports it.

### 10.1 Streaming

Use streaming for:

- Large file imports.
- Data exports.
- Bulk reconciliation.
- Event processing.
- Large API responses.
- Log processing.
- Batch transformations.

Streaming should process records incrementally while maintaining bounded working memory.

### 10.2 Chunking

Chunking may be used when streaming is not directly supported or when operations require bounded batches.

Requirements:

- Define a maximum chunk size.
- Account for record size as well as record count.
- Release temporary resources after each chunk.
- Support safe retries and idempotent processing.
- Preserve ordering where required.
- Avoid accumulating all chunk results in memory.
- Track progress and failures for long-running jobs.

### 10.3 Memory-Aware Algorithms

When processing large inputs:

- Prefer incremental aggregation where possible.
- Use bounded queues and channels.
- Avoid unnecessary copies of large buffers.
- Reuse buffers when safe and beneficial.
- Avoid retaining references to processed records.
- Apply backpressure to downstream consumers.
- Separate processing concurrency from input size.
- Estimate peak memory usage under realistic concurrency.

### 10.4 External Sorting and Partitioning

For workloads exceeding available memory:

- Use external sorting where ordering is required.
- Partition input by stable keys where appropriate.
- Use bounded merge operations.
- Store intermediate results in suitable temporary storage.
- Define cleanup and recovery procedures.
- Avoid partition schemes that produce highly skewed workloads.

The choice of partitioning and merge strategy must be validated against real data distribution.

## 11. Concurrency and Parallelism

Parallelism may improve throughput or latency, but it also introduces scheduling, synchronization, memory, and consistency costs.

### 11.1 Appropriate Use

Consider concurrency for:

- Independent network requests.
- Independent data partitions.
- Background job processing.
- CPU-intensive tasks where parallel execution is supported.
- Independent read operations.
- Large batch workloads with natural partition boundaries.

### 11.2 Requirements

- Bound worker count and concurrency.
- Apply backpressure.
- Avoid unnecessary synchronization.
- Prevent concurrent mutation of shared state.
- Define cancellation and timeout behavior.
- Handle partial failures.
- Ensure retries are safe.
- Preserve required ordering.
- Monitor queue depth and worker utilization.

### 11.3 Avoid Over-Parallelization

Excessive concurrency can increase:

- Memory usage.
- Context switching.
- Database connection pressure.
- Network contention.
- Lock contention.
- Downstream rate-limit failures.
- Tail latency.

Concurrency limits must be based on available resources and dependency capacity, not simply the number of available processors.

### 11.4 Race Conditions

Concurrent code must preserve domain invariants.

Use database transactions, atomic operations, optimistic concurrency, or other appropriate synchronization mechanisms.

Do not rely on in-process mutexes to protect data shared across multiple application instances.

## 12. Frontend Complexity

Frontend algorithms directly affect responsiveness, rendering time, battery consumption, and user experience.

### 12.1 Rendering

- Avoid unnecessary component re-renders.
- Keep component state as local as practical.
- Avoid expensive calculations during every render.
- Use memoization when profiling demonstrates value.
- Split large components into cohesive units.
- Avoid deeply nested conditional rendering that obscures performance behavior.
- Use virtualization for large lists when appropriate.
- Avoid rendering large hidden collections.
- Prefer server-side computation for suitable workloads.

### 12.2 React and Next.js

- Prefer Server Components for work that does not require browser interactivity.
- Keep Client Components small and focused.
- Avoid unnecessary client-side data transformations.
- Prevent repeated fetching of identical data.
- Use TanStack Query for client server-state management.
- Use Zustand for suitable local UI state rather than duplicating server state.
- Keep query keys stable and complete.
- Avoid broad state subscriptions that trigger unrelated rendering.
- Use dynamic imports for expensive components when beneficial.

### 12.3 Lists and Tables

For large lists and tables:

- Paginate or virtualize.
- Use stable, unique React keys.
- Avoid repeated filtering and sorting on every render.
- Precompute lookup maps for repeated membership checks.
- Debounce expensive search interactions where appropriate.
- Avoid rendering unused columns or data.
- Keep row components lightweight.
- Preserve keyboard navigation and accessibility when virtualizing.

### 12.4 Search and Filtering

For client-side filtering:

- Use linear scans for small bounded collections.
- Build indexes or lookup maps when repeated access justifies their cost.
- Debounce expensive user input.
- Avoid recomputing identical filters.
- Move large-scale filtering to the backend or search service.
- Enforce request cancellation or ignore stale responses when users rapidly change queries.

Do not download an entire tenant's catalog to perform a large search in the browser.

### 12.5 Frontend State

- Avoid duplicating derived values in state.
- Derive simple values from their source.
- Avoid deep-cloning large objects for minor updates.
- Use normalized state for large related collections where it simplifies updates.
- Keep persisted state small.
- Clear sensitive or tenant-scoped state on identity transitions.

## 13. Memory and Allocation Efficiency

Memory complexity affects garbage collection, latency, throughput, and deployment cost.

### 13.1 General Rules

- Avoid unnecessary object creation in hot paths.
- Avoid repeated cloning of large collections.
- Release references to temporary objects when no longer needed.
- Bound queues, maps, caches, and buffers.
- Avoid memory leaks caused by unremoved listeners, subscriptions, timers, or retained closures.
- Reuse allocations only when it improves measured performance and does not compromise safety or clarity.
- Account for concurrent requests when calculating peak memory usage.

### 13.2 Object Lifetime

Long-lived services must distinguish between:

- Request-scoped data.
- Job-scoped data.
- Process-scoped state.
- Shared cache data.
- Persistent records.

Do not retain request-scoped objects in global or process-wide structures without a clear lifecycle and size bound.

### 13.3 Garbage Collection

Do not optimize for fewer allocations at the expense of correctness or maintainability without evidence.

Profile allocation rate, heap growth, garbage-collection frequency, and pause time under realistic workloads.

### 13.4 Memory Budgeting

For every high-volume operation, estimate:

- Input memory.
- Intermediate data memory.
- Output memory.
- Buffer and queue memory.
- Per-worker memory.
- Serialization overhead.
- Cache retention.
- Concurrent request impact.

The estimated peak must fit within the service's allocated memory with sufficient operational headroom.

## 14. Algorithmic Security

Complexity is also a security concern.

Untrusted inputs must not be able to trigger uncontrolled resource consumption.

### 14.1 Required Controls

- Bound input sizes and nesting depth.
- Bound regular-expression complexity.
- Avoid catastrophic backtracking patterns.
- Limit sorting, filtering, and aggregation workloads.
- Enforce query timeouts.
- Apply rate limits.
- Limit concurrency and queue depth.
- Use bounded retries.
- Reject malformed or excessively complex inputs early.
- Avoid unbounded recursive traversal.
- Protect graph and relationship traversal from unexpectedly large reachable sets.

### 14.2 Regular Expressions

Regular expressions must be reviewed for worst-case behavior when applied to user-controlled input.

Prefer predictable parsing strategies for complex or structured inputs.

Use input length limits and suitable regex engines or patterns when regular expressions are required.

### 14.3 Graph and Relationship Traversal

For community relationships, access graphs, organizational structures, or dependency traversal:

- Define maximum traversal depth where required.
- Track visited nodes to prevent repeated traversal and cycles.
- Avoid revisiting nodes unnecessarily.
- Bound returned result counts.
- Enforce tenant and authorization boundaries during traversal.
- Use indexed relationships for high-volume queries.

## 15. Complexity in KAMPYN Domains

### 15.1 Food Ordering

- Resolve items by identifier through indexed database queries or in-memory maps.
- Avoid scanning an entire catalog for every item in an order.
- Batch item retrieval when validating multiple order lines.
- Calculate totals using a single bounded pass where appropriate.
- Revalidate prices and availability against authoritative data.
- Bound cart size and request payloads.
- Use transaction-safe operations for order creation.

### 15.2 Inventory

- Use maps or indexes for repeated item lookups.
- Avoid pairwise comparison of entire inventory datasets when keys can be used.
- Batch stock updates appropriately.
- Use atomic or transactional stock adjustments.
- Bound reconciliation workloads.
- Stream large inventory imports and exports.

### 15.3 Bookings

- Query availability through indexed time and resource dimensions.
- Avoid scanning all historical bookings for each availability request.
- Use suitable interval or overlap checks.
- Enforce booking constraints transactionally.
- Bound time-window sizes and result counts.
- Avoid assuming cached availability remains valid at confirmation time.

### 15.4 Search

- Use OpenSearch for suitable full-text and filtered search workloads.
- Apply result limits and pagination.
- Avoid unbounded wildcard or expensive query patterns.
- Limit filter and aggregation complexity.
- Normalize and validate user queries.
- Monitor query latency and shard-level resource consumption.
- Use caching only when repeated-query measurements justify it.

### 15.5 Community and Chat

- Use indexed queries for conversation and membership retrieval.
- Paginate message history.
- Avoid repeatedly scanning all community members to resolve access.
- Use bounded fan-out strategies for message delivery.
- Avoid broadcasting to unrelated tenants or communities.
- Keep realtime queues bounded and apply backpressure.
- Enforce authorization at retrieval and delivery boundaries.

### 15.6 Complaints and HR

- Use tenant-scoped indexes for assignment, status, and ownership queries.
- Paginate large complaint and employee records.
- Avoid per-record permission queries where safe batching is possible.
- Apply access filters before aggregations and exports.
- Bound bulk updates and report generation.

### 15.7 Analytics

- Avoid recomputing expensive metrics for every dashboard request.
- Use appropriate aggregations, materialized views, or precomputation.
- Process large datasets in bounded chunks.
- Partition large workloads where appropriate.
- Define metric freshness and tenant scope.
- Avoid unbounded exports and synchronous report generation.

## 16. Complexity Documentation

Performance-sensitive components must document their complexity where it is not obvious.

A useful complexity note should include:

- Operation and workload.
- Expected input size.
- Time complexity.
- Auxiliary space complexity.
- Important assumptions.
- Worst-case behavior.
- External dependencies.
- Concurrency and resource limits.
- Benchmark or profiling evidence, where available.

Example:

```md
### Vendor Item Lookup

- Input: n order lines and m vendor items.
- Strategy: Build a map from item ID to item.
- Time: O(n + m) average.
- Auxiliary space: O(m).
- Assumption: Item IDs are unique within the validated tenant scope.
- Limits: Maximum order-line count enforced by API validation.
- Correctness: Final price and availability are revalidated by the backend.
```

Do not add complexity comments to every trivial loop. Documentation should focus on operations where scaling behavior affects architectural or operational decisions.

## 17. Benchmarking and Profiling

Optimization must be supported by evidence.

### 17.1 Benchmarking

Benchmarks should:

- Use representative input sizes and distributions.
- Include small, typical, and large workloads.
- Measure warm and cold paths where relevant.
- Include concurrency where applicable.
- Track memory use alongside execution time.
- Avoid relying on a single run.
- Compare equivalent implementations under the same conditions.
- Record runtime, hardware, configuration, and dataset characteristics.

### 17.2 Profiling

Use suitable profiling tools for the relevant runtime and infrastructure.

Profile:

- CPU hotspots.
- Allocation rate.
- Heap growth.
- Garbage collection.
- Blocking operations.
- Database query execution.
- Network latency.
- Lock contention.
- Queue wait time.
- Serialization and deserialization.
- Frontend rendering and bundle cost.

### 17.3 Optimization Process

1. Define the performance problem and expected workload.
2. Establish a baseline.
3. Identify the bottleneck through profiling or query analysis.
4. Form a specific optimization hypothesis.
5. Implement the smallest appropriate change.
6. Verify functional correctness.
7. Benchmark against the baseline.
8. Test memory use and worst-case behavior.
9. Document the measured result and trade-offs.
10. Monitor the change under realistic deployment conditions.

Do not claim an optimization improved performance without measurements that support the claim.

## 18. Code Review Requirements

Reviewers must assess the following when evaluating performance-sensitive changes:

- Does the implementation scale with expected input size?
- Are nested loops necessary and bounded?
- Are repeated membership checks using suitable structures?
- Are large collections loaded or copied unnecessarily?
- Are database queries indexed and tenant-scoped?
- Are N+1 query patterns avoided?
- Are pagination and maximum input limits enforced?
- Are external requests batched or bounded appropriately?
- Is concurrency limited and safe?
- Is memory growth predictable?
- Can untrusted input trigger pathological behavior?
- Are caches and derived data used appropriately?
- Are performance claims supported by benchmarks or profiling?
- Does the optimization preserve correctness, security, and maintainability?

Any significant complexity regression must be justified and documented before merging.

## 19. Anti-Patterns

The following practices are prohibited unless a documented exception is approved:

- Using avoidable quadratic algorithms on large or unbounded input.
- Performing linear membership checks repeatedly inside large loops.
- Repeatedly copying growing arrays or collections.
- Sorting entire collections when only a small subset is required.
- Loading entire datasets when streaming or pagination is appropriate.
- Executing one database query per record in a large collection.
- Using unbounded recursion.
- Running unbounded concurrent tasks.
- Performing large aggregations synchronously in user-facing requests without resource limits.
- Using expensive regular expressions on unrestricted input.
- Introducing custom algorithms without explaining why standard-library options are insufficient.
- Optimizing based on intuition without profiling or measurement.
- Increasing memory consumption substantially to improve time without evaluating deployment constraints.
- Adding complex concurrency or caching mechanisms without a measured requirement.
- Ignoring worst-case input behavior for public APIs.
- Removing validation or authorization to improve performance.
- Sacrificing transactional correctness for faster execution.

## 20. Implementation Checklist

### Algorithm Design
- [ ] Time complexity is understood.
- [ ] Auxiliary space complexity is understood.
- [ ] Worst-case behavior is acceptable.
- [ ] Data structures match access patterns.
- [ ] Avoidable quadratic or worse operations are eliminated.
- [ ] Input size assumptions are explicit and enforced.

### Database and API
- [ ] Queries are tenant-scoped and appropriately indexed.
- [ ] N+1 queries are avoided.
- [ ] Pagination or streaming is used for large results.
- [ ] Request and response sizes are bounded.
- [ ] External calls use appropriate timeouts and concurrency limits.
- [ ] Expensive operations have appropriate resource controls.

### Memory and Concurrency
- [ ] Temporary allocations are reasonable.
- [ ] Collections and queues are bounded.
- [ ] Peak memory under concurrency is considered.
- [ ] Shared state is safe.
- [ ] Backpressure and cancellation are implemented where required.
- [ ] Race conditions and partial failures are handled.

### Frontend
- [ ] Rendering work is minimized.
- [ ] Large lists are paginated or virtualized where appropriate.
- [ ] Repeated transformations are avoided.
- [ ] Server-state and local-state responsibilities are separated.
- [ ] Network requests are bounded and deduplicated where suitable.
- [ ] Accessibility and user experience remain intact.

### Security
- [ ] Untrusted input cannot trigger unbounded work.
- [ ] Algorithmic denial-of-service risks are considered.
- [ ] Query and traversal complexity is bounded.
- [ ] Rate limits and timeouts are appropriate.
- [ ] Tenant isolation and authorization are preserved.

### Validation
- [ ] Correctness tests pass.
- [ ] Representative performance benchmarks are available where justified.
- [ ] Memory behavior is evaluated.
- [ ] Significant changes are profiled.
- [ ] Performance trade-offs are documented.
- [ ] No unacceptable regression is introduced.

## 21. Definition of Done

A performance-sensitive implementation is complete when:

- Its expected workload and scaling requirements are understood.
- Its time and space complexity are appropriate for expected input sizes.
- Data structures and algorithms are selected for the actual access patterns.
- Potential bottlenecks in database, network, serialization, memory, and concurrency are considered.
- Resource usage is bounded where inputs or workloads can grow.
- Security, tenant isolation, and business correctness are preserved.
- Tests validate functional behavior and relevant edge cases.
- Benchmarks or profiling support any significant performance claims.
- Meaningful complexity trade-offs and limitations are documented.
- Code review confirms there are no unjustified complexity regressions.

**Final principle:** KAMPYN must scale through deliberate algorithmic and architectural choices. Prefer simple, predictable, bounded operations that meet measured performance requirements while preserving correctness, security, and maintainability.
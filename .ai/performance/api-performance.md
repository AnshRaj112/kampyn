# Algorithm Design and Performance Standards

## 1. Purpose

This document defines KAMPYN's standards for algorithm design, computational complexity, data structures, and runtime efficiency across frontend, backend, APIs, and data-processing services.

KAMPYN operates across multiple university services, including food ordering, inventory management, bookings, search, community interactions, notifications, and administrative workflows. As the platform scales across tenants, users, transactions, and datasets, inefficient algorithms can lead to increased latency, memory consumption, infrastructure costs, and degraded user experience.

Every algorithm must be selected based on its correctness, complexity, scalability, memory usage, and suitability for the workload.

### Core objectives

- Prefer efficient algorithms and data structures for the expected workload.
- Avoid unnecessary quadratic and higher-complexity operations.
- Minimize redundant computation and repeated data traversal.
- Optimize time and space complexity without sacrificing correctness.
- Design algorithms that scale predictably with data volume.
- Avoid unbounded memory consumption and unnecessary data copying.
- Use concurrency and parallelism only where they provide measurable benefits.
- Establish complexity expectations for performance-critical operations.
- Measure actual performance rather than relying solely on theoretical complexity.
- Prevent algorithmic regressions through code review, testing, and profiling.

---

## 2. Non-Negotiable Principles

1. **Correctness first:** An optimized algorithm that produces incorrect results is unacceptable.
2. **Analyze before implementing:** Understand input size, operation frequency, constraints, and expected growth before selecting an algorithm.
3. **Avoid unnecessary O(n²) operations:** Quadratic complexity must not be introduced where an efficient alternative is reasonably available.
4. **Choose data structures deliberately:** Data structures must match the access patterns and update requirements of the workload.
5. **Avoid repeated work:** Cache, precompute, index, or restructure computations when repeated work is a demonstrated bottleneck.
6. **Bound resource usage:** Algorithms must have predictable time and memory behavior under expected and worst-case inputs.
7. **Optimize the bottleneck:** Focus on measured hot paths instead of prematurely optimizing insignificant operations.
8. **Preserve scalability:** Design for realistic growth in tenants, users, records, and concurrent operations.
9. **Avoid unnecessary complexity:** Prefer a simpler efficient algorithm over a complicated optimization with marginal benefit.
10. **Use explicit complexity reasoning:** Performance-critical algorithms must document their expected time and space complexity.
11. **Account for I/O:** Network, database, filesystem, and serialization costs must be considered alongside computational complexity.
12. **Prevent algorithmic abuse:** Validate and bound user-controlled inputs to avoid resource-exhaustion vulnerabilities.

---

## 3. Algorithm Selection

Algorithm selection must be based on the problem's constraints rather than familiarity or convenience alone.

Before implementing a performance-sensitive algorithm, identify:

- Input size and expected growth.
- Frequency of execution.
- Required time complexity.
- Acceptable memory consumption.
- Ordering and stability requirements.
- Read/write frequency.
- Concurrency requirements.
- Data distribution and potential worst cases.
- Whether processing can be streamed or performed incrementally.
- Whether the operation is CPU-bound, memory-bound, or I/O-bound.

### 3.1 Complexity preference

The following table provides general guidance, not a universal ranking. An algorithm's actual suitability depends on input size, constant factors, memory behavior, and the workload.

| Complexity | General suitability | Typical examples |
|---|---|---|
| O(1) | Ideal for frequent direct operations | Hash-map lookup, array index access |
| O(log n) | Efficient for large ordered datasets | Binary search, balanced tree lookup |
| O(n) | Usually suitable for a single pass | Traversal, aggregation, linear search |
| O(n log n) | Common for large sorting workloads | Merge sort, heap sort |
| O(n²) | Avoid for large inputs unless justified | Naive pairwise comparison |
| O(n³) | Generally unsuitable for large inputs | Naive all-pairs matrix operations |
| O(2ⁿ) | Restricted to small inputs or specialized pruning | Exhaustive subset enumeration |
| O(n!) | Restricted to very small inputs | Exhaustive permutation search |

An O(n) solution is not automatically better than O(log n) for every workload. Likewise, an O(n log n) solution may be preferable to an O(n) approach if it provides stronger correctness guarantees or substantially lower constant costs in the relevant context.

### 3.2 Input-size awareness

An algorithm suitable for a list of 50 items may be unsuitable for 500,000 records.

Developers must consider expected production-scale inputs and realistic worst-case conditions rather than testing only small examples.

### 3.3 Complexity trade-offs

When selecting between alternatives, consider:

- Runtime complexity.
- Space complexity.
- Data structure overhead.
- Cache locality.
- Memory allocation and garbage collection.
- I/O and serialization costs.
- Concurrency and synchronization.
- Implementation complexity.
- Maintainability and correctness guarantees.

A trade-off must be justified by the actual requirements of the feature.

---

## 4. Time and Space Complexity

Performance-sensitive algorithms must be analyzed for both time and space complexity.

### 4.1 Time complexity

Time complexity describes how the number of operations grows with input size.

For example:

```ts
function findItem(
  items: readonly Item[],
  targetId: string,
): Item | undefined {
  return items.find((item) => item.id === targetId);
}
```

This linear lookup has O(n) time complexity.

When repeated lookups are required, building an appropriate index may reduce the overall cost.

```ts
const itemsById = new Map(
  items.map((item) => [item.id, item]),
);

const item = itemsById.get(targetId);
```

Building the map takes O(n) expected time and O(n) additional space. Subsequent lookups take O(1) expected time.

The indexed approach is useful when many lookups justify the upfront construction and memory cost.

### 4.2 Space complexity

Space complexity describes additional memory requirements as input size grows.

Developers must account for:

- Temporary collections.
- Recursion stacks.
- Cloned objects.
- Cached values.
- Intermediate transformation results.
- Buffers.
- Queues and concurrent tasks.
- Retained references and closures.

Avoid creating multiple full-size copies of large datasets when an in-place, streaming, or incremental approach is suitable.

### 4.3 Auxiliary space

Document auxiliary space separately when it materially affects the algorithm's behavior.

For example:

- Iterative linear traversal: O(1) auxiliary space.
- Building a lookup map: O(n) auxiliary space.
- Recursive traversal of a balanced tree: O(log n) stack space.
- Recursive traversal of a degenerate tree: O(n) stack space.

### 4.4 Amortized complexity

Some operations have occasional expensive steps but remain efficient over a sequence of operations.

Dynamic array insertion, for example, is typically amortized O(1) at the end, although an individual resize can take O(n).

Use amortized analysis where it more accurately describes real workloads.

---

## 5. Data Structure Selection

Data structures must be chosen based on operation patterns, not merely convenience.

| Data structure | Typical use | Common performance characteristic |
|---|---|---|
| Array | Ordered collections and indexed access | O(1) indexed access, O(n) search |
| Object / Map | Key-based lookup | O(1) expected lookup for hash-based maps |
| Set | Uniqueness and membership checks | O(1) expected membership check |
| Queue | FIFO processing | O(1) enqueue/dequeue with a suitable implementation |
| Stack | LIFO processing | O(1) push/pop |
| Heap | Priority-based retrieval | O(log n) insertion and removal |
| Balanced tree | Ordered lookup and traversal | O(log n) search and updates |
| Trie | Prefix-based lookup | Depends on key length |
| Graph | Relationships and connectivity | Depends on vertices and edges |
| Ring buffer | Bounded streaming or event processing | O(1) insertion/removal |
| Bloom filter | Probabilistic membership checks | O(k) operations for k hash functions |

These are typical characteristics. Actual performance depends on implementation, runtime, data distribution, and workload.

### 5.1 Arrays

Use arrays when:

- Ordering is important.
- Sequential traversal is common.
- Indexed access is required.
- Collection size is reasonably bounded or growth is managed.

Avoid repeated array searches inside loops when a lookup structure can be built once.

### 5.2 Maps and sets

Use `Map` for key-value lookup and `Set` for uniqueness or membership checks.

Example:

```ts
const activeUserIds = new Set<string>(activeUsers);

const isActive = activeUserIds.has(userId);
```

Prefer these structures over repeated linear membership checks for sufficiently large or frequently queried collections.

Do not assume hash-based operations have guaranteed worst-case O(1) complexity. Treat them as expected-time operations.

### 5.3 Queues and stacks

Use queues for FIFO work and stacks for LIFO processing.

Avoid implementing a queue using repeated `Array.shift()` for large collections if the operation causes repeated element movement.

Use a head index, deque, ring buffer, or suitable queue implementation when processing volume justifies it.

### 5.4 Heaps

Use a heap when repeatedly retrieving the highest- or lowest-priority element.

Examples include:

- Priority-based job scheduling.
- Selecting the next scheduled task.
- Maintaining a bounded set of top results.

Avoid sorting an entire collection repeatedly when a heap can satisfy the required operations more efficiently.

### 5.5 Graphs

Use graph structures for workflows involving explicit relationships and traversal.

Examples include:

- Dependency relationships.
- Route connectivity.
- Organizational structures.
- Resource relationships.

Choose adjacency lists or other representations based on graph density and required operations. Avoid dense matrix representations for sparse graphs without a specific reason.

---

## 6. Avoiding Quadratic Complexity

Unnecessary O(n²) complexity is a major performance risk and must be avoided for large or unbounded inputs.

### 6.1 Nested-loop detection

Review nested loops that traverse the same or related collections.

Example of a potentially inefficient implementation:

```ts
const unmatched = sourceItems.filter(
  (source) =>
    !targetItems.some(
      (target) => target.id === source.id,
    ),
);
```

For source and target arrays of size n, this may require O(n²) comparisons.

A lookup set can reduce the expected time complexity:

```ts
const targetIds = new Set(
  targetItems.map((item) => item.id),
);

const unmatched = sourceItems.filter(
  (source) => !targetIds.has(source.id),
);
```

This approach requires O(n) expected time and O(n) additional space.

### 6.2 Nested loops that are acceptable

Nested loops are not automatically inefficient.

They may be justified when:

- Input sizes are strictly small and bounded.
- The operation genuinely requires pairwise comparisons.
- A more efficient algorithm is unavailable or introduces unacceptable trade-offs.
- Profiling shows that the operation is not a meaningful bottleneck.

Any such exception must include a documented rationale when the algorithm is performance-sensitive.

### 6.3 Alternatives to quadratic operations

Consider:

- Hash-based indexing.
- Sorting followed by a linear merge.
- Binary search over sorted data.
- Grouping records by key.
- Precomputed lookup tables.
- Database joins and indexes.
- Streaming comparisons.
- Domain-specific algorithms.

Select the alternative that meets correctness, memory, and ordering requirements.

---

## 7. Searching and Lookup Algorithms

Search operations must match the structure and size of the dataset.

### 7.1 Linear search

Linear search is suitable for:

- Small collections.
- One-off lookups.
- Unordered collections where building an index is not worthwhile.

Complexity: O(n) time, O(1) auxiliary space.

### 7.2 Binary search

Binary search is appropriate for sorted collections where repeated lookups are required.

Complexity: O(log n) time and O(1) auxiliary space for an iterative implementation.

Requirements:

- The collection must be sorted according to the same comparison rule.
- Boundary conditions must be tested.
- Duplicate-value behavior must be defined.
- The cost of maintaining sorted order must be considered.

Do not apply binary search to unsorted data.

### 7.3 Hash-based lookup

Use hash-based structures when frequent key-based lookups are required.

Complexity: O(1) expected lookup time, with O(n) space for an index over n records.

Consider:

- Index construction cost.
- Memory consumption.
- Key normalization.
- Collision behavior.
- Mutation and invalidation requirements.

### 7.4 Search at scale

For large datasets, avoid loading all records into the frontend solely to perform searches.

Prefer backend filtering, database indexes, or dedicated search infrastructure such as OpenSearch through KAMPYN's backend APIs.

Search requests must be bounded, tenant-scoped, and protected by backend authorization.

---

## 8. Sorting and Ordering

Sorting algorithms must be selected according to dataset size, required stability, ordering semantics, and available memory.

### 8.1 General standards

- Prefer well-tested built-in sorting implementations for ordinary application needs.
- Define explicit comparators for domain-specific ordering.
- Ensure sorting is deterministic when the user expects stable ordering.
- Avoid repeatedly sorting unchanged data.
- Avoid sorting large datasets in the browser when server-side sorting is more appropriate.
- Validate sort fields and directions before constructing requests.

### 8.2 Common complexity

| Algorithm | Average time | Worst-case time | Auxiliary space |
|---|---:|---:|---:|
| Insertion sort | O(n²) | O(n²) | O(1) |
| Merge sort | O(n log n) | O(n log n) | O(n) |
| Heap sort | O(n log n) | O(n log n) | O(1) |
| Quicksort | O(n log n) | O(n²) | Depends on implementation |

Use these as algorithmic reference characteristics. Built-in runtime sorting implementations may use hybrid algorithms and implementation-specific guarantees.

### 8.3 Sorting large datasets

For large administrative tables, order histories, and inventory lists:

- Prefer backend sorting.
- Use stable pagination ordering.
- Include a deterministic tie-breaker where necessary.
- Avoid repeatedly sorting every page independently if global ordering is required.
- Use external sorting or database-supported ordering for datasets that exceed practical memory limits.

---

## 9. Grouping, Aggregation, and Deduplication

Grouping and aggregation should generally use a single pass when possible.

### 9.1 Grouping

Prefer keyed accumulators over repeated filtering for each group.

```ts
const ordersByVendor = new Map<string, Order[]>();

for (const order of orders) {
  const existing = ordersByVendor.get(order.vendorId);

  if (existing) {
    existing.push(order);
  } else {
    ordersByVendor.set(order.vendorId, [order]);
  }
}
```

Expected time complexity: O(n), where n is the number of orders.

Auxiliary space is O(n) for the grouped output.

### 9.2 Aggregation

Use a single-pass accumulator for basic totals, counts, and summaries.

```ts
const totals = orders.reduce(
  (result, order) => ({
    count: result.count + 1,
    amount: result.amount + order.total,
  }),
  { count: 0, amount: 0 },
);
```

For high-volume processing, avoid creating unnecessary intermediate objects on every iteration. Use a mutable local accumulator when appropriate and safe.

### 9.3 Deduplication

Use a `Set` or keyed `Map` for expected linear-time deduplication when the identity rule is well-defined.

Define explicitly:

- The deduplication key.
- Which duplicate record is retained.
- Whether ordering is preserved.
- How conflicting records are handled.

Do not deduplicate records using an incomplete key that can merge distinct domain entities.

---

## 10. Pagination and Large-Scale Data Processing

Pagination and bounded processing are mandatory for unbounded or potentially large datasets.

### 10.1 Pagination

Choose pagination according to the data access pattern.

**Offset pagination** is suitable for smaller datasets and direct page navigation when the underlying query remains efficient.

**Cursor pagination** is preferred for large, frequently changing datasets or sequential traversal where offset costs or consistency issues become significant.

Requirements:

- Bound page size.
- Use deterministic ordering.
- Validate pagination inputs.
- Avoid retrieving unused fields or records.
- Keep pagination behavior consistent with filtering and sorting.
- Ensure tenant scoping is enforced by the backend.

### 10.2 Streaming

Use streaming when the dataset is too large to process safely in memory or when incremental results are valuable.

Examples:

- Large exports.
- Report generation.
- Data imports.
- Bulk validation.
- Event processing.

Streaming must define:

- Bounded buffer sizes.
- Backpressure behavior.
- Error propagation.
- Cancellation handling.
- Partial-result semantics.
- Resource cleanup.

### 10.3 Chunking

Chunking can limit memory consumption and allow incremental processing.

Select chunk sizes based on:

- Record size.
- Memory constraints.
- I/O behavior.
- Downstream processing cost.
- Concurrency level.

Avoid arbitrary chunk sizes without measurement.

### 10.4 Incremental processing

Where appropriate, process only changed records rather than repeatedly processing an entire dataset.

Incremental processing must define how updates, deletions, retries, and out-of-order events are handled.

Never assume an incremental result is correct without accounting for missed or duplicated changes.

---

## 11. Recursion and Iteration

Recursion can improve clarity for naturally recursive problems but must be evaluated for stack depth and memory use.

### 11.1 Recursion

Use recursion when:

- The problem is naturally hierarchical.
- Maximum depth is known and safely bounded.
- The implementation remains clear and correct.
- Stack usage is acceptable.

Avoid deep recursion over user-controlled or unbounded input where stack overflow is possible.

### 11.2 Iteration

Prefer iterative traversal when:

- Input depth is unbounded.
- Stack limits may be exceeded.
- Explicit memory management is required.
- Iterative processing is clearer for the problem.

Use explicit stacks or queues to implement depth-first or breadth-first traversal safely.

### 11.3 Tail recursion

Do not assume that the runtime optimizes tail-recursive functions. Verify the target language and runtime guarantees before depending on tail-call optimization.

---

## 12. Graph and Relationship Algorithms

Use established graph algorithms for relationship-based workflows instead of ad hoc repeated traversal.

Common algorithms include:

| Problem | Suitable algorithm |
|---|---|
| Shortest path with nonnegative weights | Dijkstra's algorithm |
| Shortest path with negative edges | Bellman-Ford, where appropriate |
| Unweighted shortest path | Breadth-first search |
| Dependency ordering | Topological sort |
| Connected components | DFS, BFS, or disjoint-set union |
| Minimum spanning tree | Kruskal's or Prim's algorithm |
| Cycle detection | DFS with visitation states or other suitable methods |

### 12.1 Graph representation

- Use adjacency lists for sparse graphs.
- Use adjacency matrices only when density and access patterns justify them.
- Avoid rebuilding graph structures unnecessarily.
- Validate node and edge limits for user-controlled graph input.
- Prevent unbounded traversal and excessive memory allocation.

### 12.2 Workflow dependencies

For task dependencies or other directed workflows:

- Detect cycles where acyclic structure is required.
- Validate references.
- Define behavior for disconnected nodes.
- Ensure deterministic output when required.
- Test large and malformed graphs.

---

## 13. Concurrency and Parallelism

Concurrency and parallelism should be introduced only when they improve throughput, latency, or resource utilization without compromising correctness.

### 13.1 Concurrency

Concurrency is useful for overlapping independent I/O operations.

Examples include:

- Fetching independent dashboard data.
- Loading separate page sections.
- Processing independent external service calls.

Use bounded concurrency to avoid overwhelming downstream systems.

### 13.2 Parallelism

Parallelism may help CPU-intensive work where the runtime and deployment environment support it.

Examples include:

- Large independent transformations.
- Image processing.
- Batch computation.
- Complex analytics.

Consider the cost of:

- Scheduling.
- Serialization.
- Data copying.
- Synchronization.
- Memory overhead.
- Contention.
- Result aggregation.

Parallelism is not automatically faster than a well-optimized sequential algorithm.

### 13.3 Bounded concurrency

Never start unbounded numbers of tasks from user-controlled input.

Use concurrency limits, backpressure, cancellation, and resource budgets appropriate to the workload.

### 13.4 Race conditions

Concurrent operations must define:

- Ownership of mutable state.
- Ordering guarantees.
- Conflict resolution.
- Cancellation behavior.
- Retry semantics.
- Idempotency requirements.

Critical operations such as payments, inventory adjustments, and bookings require authoritative backend coordination.

---

## 14. Caching and Precomputation

Caching can reduce repeated computation, but it introduces memory costs, invalidation complexity, and consistency risks.

### 14.1 Appropriate caching

Consider caching when:

- Computation is expensive.
- Results are reused frequently.
- Cache keys can be defined correctly.
- Data freshness requirements are understood.
- Cache invalidation is manageable.

### 14.2 Cache design

Every cache must define:

- Key structure.
- Scope and ownership.
- TTL or invalidation strategy.
- Maximum size or eviction policy.
- Behavior on cache miss.
- Behavior on stale data.
- Tenant and identity isolation where applicable.

### 14.3 Avoiding ineffective caching

Do not cache:

- Cheap computations without a demonstrated benefit.
- Highly volatile data without a suitable invalidation strategy.
- Sensitive data in inappropriate scopes.
- Large objects that create disproportionate memory pressure.

### 14.4 Precomputation

Precompute values when the cost can be amortized over repeated use and the underlying data does not change too frequently.

Ensure precomputed data is refreshed or invalidated correctly when its source changes.

---

## 15. Database-Aware Algorithm Design

Application-level algorithms must account for database query behavior.

### 15.1 Avoid application-side joins at scale

Do not fetch multiple large collections and perform expensive joins in application memory when a suitable database join, aggregation, or indexed query can perform the operation more efficiently.

Choose the execution layer according to data volume, database capability, consistency requirements, and network cost.

### 15.2 N+1 query prevention

Avoid issuing one database request for every record in a collection.

Prefer:

- Batch retrieval.
- Appropriate joins.
- Data loaders.
- Aggregation pipelines.
- Carefully scoped prefetching.

Avoid fetching excessive related data simply to eliminate query count.

### 15.3 Index-aware algorithms

Design application queries around appropriate indexes.

- Match indexes to real filtering and ordering patterns.
- Inspect query plans for critical operations.
- Avoid unnecessary full collection scans.
- Avoid creating excessive indexes that degrade write performance.
- Keep query result sizes bounded.

### 15.4 Query complexity

The algorithm's effective cost includes database execution and network transfer.

A single API request can still be expensive if it triggers unbounded scans, large joins, or excessive serialization.

Measure end-to-end latency and database execution separately where possible.

---

## 16. Frontend Algorithm Standards

Frontend algorithms must preserve interaction responsiveness and avoid unnecessary main-thread work.

### 16.1 Rendering and transformations

- Avoid repeated filtering, sorting, or mapping of large arrays during rendering.
- Use stable identifiers for list keys.
- Keep calculations local to the relevant component or data layer.
- Memoize only measured expensive calculations.
- Prefer server-side processing for large datasets.
- Use virtualization when rendering cost is demonstrated to be significant.

### 16.2 Search

- Debounce high-frequency user input where appropriate.
- Cancel obsolete requests.
- Use local search only for suitably bounded collections.
- Use backend search for large catalogs and records.
- Prevent stale responses from replacing current results.

### 16.3 UI scheduling

- Keep event handlers short.
- Avoid synchronous expensive computation during user interactions.
- Use browser scheduling features or Web Workers only where justified.
- Keep long-running tasks cancellable when possible.
- Provide progress feedback for lengthy operations.

### 16.4 State updates

- Update only the relevant state slice.
- Avoid rebuilding large state objects unnecessarily.
- Avoid repeatedly copying large collections for isolated updates.
- Use immutable updates according to the project's state-management standards.
- Keep server state in TanStack Query rather than duplicating it in UI stores.

---

## 17. Memory Efficiency

Memory efficiency is a first-class algorithmic requirement.

### 17.1 Allocation

Avoid:

- Repeated large array and object cloning.
- Unnecessary intermediate collections.
- Unbounded caches.
- Excessively deep object structures.
- Repeated serialization and deserialization.
- Retaining obsolete data after operations complete.

### 17.2 Streaming and bounded memory

For large datasets, prefer:

- Iterators.
- Streams.
- Bounded buffers.
- Incremental aggregation.
- Chunked processing.
- Database-side computation.

Avoid loading an entire large file or result set into memory unless the size is bounded and justified.

### 17.3 Memory leaks

Ensure that:

- Event listeners are removed.
- Timers are cleared.
- Subscriptions are closed.
- Large references are released.
- Completed tasks do not retain unnecessary state.
- Cache eviction works as intended.

### 17.4 Memory complexity review

For performance-critical features, document expected additional memory consumption as input size grows.

When a feature processes potentially large datasets, specify maximum supported input sizes or resource limits.

---

## 18. Algorithmic Security

Algorithmic inefficiency can be exploited through malicious or unusually large inputs.

### 18.1 Input bounds

Validate and limit:

- Collection sizes.
- String lengths.
- Nesting depth.
- Pagination limits.
- Batch sizes.
- Search query length.
- File sizes.
- Graph vertices and edges.
- Concurrent task counts.
- Regular expression complexity where relevant.

Limits must be enforced by the backend for protected operations.

### 18.2 Denial-of-service prevention

Avoid algorithms with catastrophic or uncontrolled resource consumption when operating on user-controlled input.

Examples include:

- Exponential exhaustive search without strict bounds.
- Pathological regular expressions.
- Unbounded recursion.
- Unbounded task creation.
- Quadratic comparisons over large inputs.
- Unbounded graph traversal.
- Excessive parsing of deeply nested payloads.

### 18.3 Rate limiting

Use backend rate limiting and abuse controls for expensive operations such as search, reporting, uploads, and bulk processing.

Frontend throttling and debouncing improve usability but are not security controls.

### 18.4 Resource exhaustion

Define safe limits for CPU time, memory, request duration, and downstream work for computationally expensive endpoints.

Ensure cancellation and timeouts do not leave background tasks or partial state unmanaged.

---

## 19. Testing Algorithm Correctness and Complexity

Algorithms must be tested for correctness, edge cases, and realistic scale.

### 19.1 Unit tests

Cover:

- Typical inputs.
- Empty inputs.
- Single-element inputs.
- Duplicate values.
- Already sorted and reverse-sorted data.
- Boundary values.
- Invalid input.
- Large inputs.
- Worst-case distributions.
- Overflow and precision edge cases where applicable.

### 19.2 Property-based testing

Use property-based testing for algorithms where general invariants are more valuable than a small number of examples.

Examples:

- Sorting output is ordered.
- Sorting preserves the multiset of input values.
- Deduplication preserves the defined representative for each key.
- Pagination returns no duplicate records under its stated consistency model.
- Graph traversal visits only reachable nodes.
- Aggregation matches a trusted reference implementation.

### 19.3 Differential testing

For complex algorithms, compare results with a simple trusted implementation on bounded test inputs.

This can help identify correctness errors in optimized implementations.

### 19.4 Performance testing

Benchmark critical algorithms using:

- Representative input sizes.
- Realistic data distributions.
- Worst-case inputs.
- Repeated execution.
- Memory measurements.
- Relevant runtime and deployment configurations.

Record the environment and avoid comparing benchmarks collected under substantially different conditions.

### 19.5 Regression testing

When an algorithm is optimized:

1. Preserve correctness tests.
2. Add a reproducible benchmark where warranted.
3. Compare runtime and memory against a baseline.
4. Test edge cases and adverse input distributions.
5. Confirm that the optimization does not introduce unacceptable complexity or operational cost.

Do not make strict timing assertions in ordinary unit tests when execution variability would make them unreliable. Use dedicated benchmarks or performance-test infrastructure instead.

---

## 20. Benchmarking and Profiling

Performance claims must be supported by measurements.

### 20.1 Profiling tools

Use language- and runtime-appropriate tools, such as:

- Chrome DevTools Performance and Memory panels.
- React DevTools Profiler.
- Node.js profiling and diagnostic tools.
- Go CPU and memory profiling.
- Rust benchmarking and profiling tools.
- Database query plans and execution statistics.
- Distributed tracing and application performance monitoring.

### 20.2 Benchmark design

A benchmark should specify:

- The algorithm and implementation.
- Input size and distribution.
- Runtime and hardware.
- Warm-up behavior where relevant.
- Number of iterations.
- Timing methodology.
- Memory measurements.
- Baseline and comparison results.

### 20.3 Avoid misleading benchmarks

Do not:

- Benchmark only trivial inputs.
- Compare different hardware without qualification.
- Ignore startup and allocation costs when they matter.
- Use a single measurement as definitive evidence.
- Optimize only for synthetic data that does not reflect production.
- Infer scalability from one input size.

### 20.4 Profiling outcomes

A performance investigation should identify the actual bottleneck, such as:

- CPU-intensive computation.
- Memory allocation or garbage collection.
- Database execution.
- Network latency.
- Serialization.
- Lock contention.
- Rendering or layout work.
- Excessive I/O.

Apply the optimization to the identified bottleneck rather than rewriting unrelated code.

---

## 21. Documentation Requirements

Performance-sensitive algorithms must include documentation sufficient for reviewers and maintainers to understand their behavior.

Document, where relevant:

- Problem being solved.
- Expected input size.
- Algorithm selected.
- Time complexity.
- Auxiliary space complexity.
- Important assumptions.
- Worst-case behavior.
- Resource bounds.
- Correctness invariants.
- Concurrency and consistency assumptions.
- Benchmark evidence for significant optimizations.

Do not add complexity annotations to every trivial function. Focus on algorithms whose behavior could materially affect system performance or scalability.

---

## 22. Code Review Standards

Every performance-sensitive algorithm must be reviewed for:

- Correctness and edge cases.
- Appropriate algorithm selection.
- Time and space complexity.
- Avoidable nested loops.
- Repeated computation.
- Data structure suitability.
- Memory allocation and copying.
- Database and network interaction.
- Input bounds and algorithmic abuse.
- Concurrency safety.
- Readability and maintainability.
- Test and benchmark coverage where appropriate.

Reviewers must distinguish between theoretical concerns and demonstrated performance risks. Unnecessary optimization should not make the code harder to understand without measurable benefit.

### 22.1 Complexity exceptions

An algorithm with O(n²) or higher complexity may be accepted when:

- Input size is explicitly small and bounded.
- The problem inherently requires such work.
- No suitable alternative meets correctness or product constraints.
- Measured costs are acceptable for the intended workload.

The exception must document the input bounds, rationale, expected resource usage, and conditions that would trigger reconsideration.

---

## 23. Algorithmic Anti-Patterns

The following practices are prohibited unless explicitly justified.

- Using nested linear searches for large collections when an index is suitable.
- Repeatedly sorting unchanged data.
- Recomputing expensive values in frequently executed paths.
- Performing unbounded work on user-controlled inputs.
- Loading unbounded datasets into memory.
- Using inappropriate data structures for high-frequency operations.
- Creating unbounded concurrent tasks.
- Using recursive algorithms with uncontrolled depth.
- Repeatedly copying large collections for small updates without justification.
- Performing application-side joins over large datasets when a suitable database operation exists.
- Triggering N+1 queries.
- Rebuilding indexes for every lookup.
- Introducing caches without invalidation and memory policies.
- Using parallelism without measuring its overhead and benefit.
- Optimizing based on intuition alone.
- Replacing clear code with complex micro-optimizations that have no measurable benefit.
- Ignoring worst-case behavior because average-case benchmarks look acceptable.
- Treating frontend throttling as a substitute for backend resource controls.
- Claiming an algorithm is O(1) without accounting for the relevant assumptions.

---

## 24. Performance Review Checklist

### Algorithm design
- [ ] The problem and constraints are clearly understood.
- [ ] The selected algorithm matches the workload.
- [ ] Expected input sizes and growth are considered.
- [ ] Time complexity has been assessed.
- [ ] Space complexity has been assessed.
- [ ] Worst-case behavior is understood.
- [ ] Avoidable O(n²) or higher-complexity operations have been removed or justified.

### Data structures
- [ ] Data structures match access and update patterns.
- [ ] Repeated lookups use suitable indexing where beneficial.
- [ ] Collection ordering and duplicate semantics are explicit.
- [ ] Memory overhead is acceptable.
- [ ] Cache invalidation and index lifecycle are considered.

### Runtime and resources
- [ ] Memory allocation and data copying are reasonable.
- [ ] Large inputs are bounded or streamed.
- [ ] Concurrency is bounded.
- [ ] CPU-intensive work does not unnecessarily block critical interactions.
- [ ] Network and database costs are included in the design.
- [ ] User-controlled inputs cannot cause unbounded computation.

### Correctness and reliability
- [ ] Edge cases are covered.
- [ ] Invariants are documented where needed.
- [ ] Concurrent operations are safe.
- [ ] Cancellation and error handling are correct.
- [ ] Tests cover relevant input distributions.
- [ ] Optimizations preserve the expected output and ordering semantics.

### Measurement
- [ ] Critical algorithms have been profiled or benchmarked where appropriate.
- [ ] Benchmark conditions are documented.
- [ ] Performance claims are supported by evidence.
- [ ] Regressions are checked against an appropriate baseline.
- [ ] Any complexity exception has a documented rationale.

---

## 25. Definition of Done

A performance-sensitive algorithm is not complete until:

1. Its correctness requirements and input constraints are defined.
2. The chosen algorithm and data structures are appropriate for the workload.
3. Time and space complexity are understood.
4. Avoidable quadratic or higher-complexity operations have been removed or justified.
5. Worst-case and resource-exhaustion behavior have been considered.
6. Large inputs are handled through suitable bounds, pagination, streaming, or incremental processing.
7. Database, network, and serialization costs have been considered where relevant.
8. Concurrency and memory usage are controlled.
9. Unit and edge-case tests pass.
10. Performance-sensitive paths have appropriate profiling or benchmark evidence.
11. Significant trade-offs and exceptions are documented.
12. The implementation remains readable, maintainable, and consistent with KAMPYN's engineering standards.

---

## 26. Guiding Principle

KAMPYN's algorithms must be designed for predictable growth, bounded resource consumption, and reliable correctness across all supported workflows.

The objective is not to achieve the lowest theoretical complexity at any cost. It is to select an algorithm that performs efficiently for realistic workloads, behaves safely under adverse inputs, and remains understandable and maintainable as the platform evolves.

**Choose the right algorithm, bound its cost, prove its correctness, and measure its performance.**
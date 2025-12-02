# genogrove — quick conceptual briefing

A short, practical reference for the core structure and common operations in the genogrove library.

## Library overview
- Purpose: in-memory B-tree-like store of keys (commonly genomic intervals) across multiple named indices (e.g., chromosomes). Supports fast insert and efficient interval intersection queries.
- Core types:
  - `grove<key_type, data_type>` — top-level container for one or more root trees (one per index name).
  - `node<key_type, data_type>` — internal B-tree node holding keys and child pointers.
  - `key<...>` — typed key object holding the user data (e.g., interval + payload).

## Graph building (the grove data structure)
- The grove stores a map of root nodes keyed by index name. Each root is the root of a B-tree-like structure.
- Sorted inserts use a cached `rightmost_node` for fast append-like insertions.
- Unsorted inserts traverse the tree from root to leaf (binary-like navigation inside each node) and insert into the correct leaf, splitting nodes when full.

## Common operations and complexity (time / memory)
- insert (unsorted) — time: O(log_k N + k) where k = order, N = #keys. The log_k N search finds the leaf; insertion requires shifting up to O(k) items in the node. Memory: O(1) extra per key (but the implementation performs one heap allocation per key). Splits are O(k) and happen amortized less frequently.
- insert (sorted/rightmost) — time: amortized O(1) per key if growing at end and few splits; worst-case similar to unsorted if splits propagate. Memory: same per-key allocation.
- bulk load — not implemented as a dedicated path; building from sorted data via repeated `insert_sorted` is OK but a specialized bulk-loader would be O(N) and much faster for large sorted batches.
- search / intersect (interval query) — time: O(log_k N + M) where M is number of matching keys returned; nodes scanned are typically local and the tree traversal is efficient. Memory: search uses O(M) to store results.
- split_node — time O(k) (moves keys/children); memory O(k) temporary for copies/moves.
- get_root / get_rightmost_node — O(1) map lookups.

## Practical notes
- Small `order` (k) values are frequent: linear scans inside nodes are cheap for small k. For large k, switching to binary search (lower_bound) for the insertion point would be more efficient.
- Current implementation allocates a `key` on the heap per insert. This is the main run-time/memory overhead for bulk inserts (malloc cost + fragmentation + pointer indirection). Replacing per-key heap allocation with an arena/pool or keeping keys inline inside nodes will offer the biggest win.

---

## Edits made (insertion-focused)
This section documents the small, safe improvements I implemented to reduce copy/heap overhead during insertions. All edits were low-risk header-only changes and validated with unit tests & benchmarks.

1. Fixed build error in `include/genogrove/data_type/any_type.hpp` (missing `#include <istream>`). This allowed the benchmarks to compile.

2. Added move-aware insertion to reduce copies and improve insertion throughput for temporaries:
   - File: `include/genogrove/structure/grove/node.hpp`
     - Added an rvalue overload: `insert_key(gdt::key<...>&&)` which constructs the heap key with `std::move` to avoid an extra copy.
   - File: `include/genogrove/structure/grove/grove.hpp`
     - Changed sorted-insert path and when allocating parent key on splits to use `std::move()` for the temporary `key` objects so the new rvalue overload and move-constructors are used.

3. Rationale & effect
   - Why: the default code constructed temporary keys and then copied them into heap-allocated `key` objects (1 copy + 1 allocation). Using move semantics for temporaries reduces copies.
   - Result: unit tests stayed green (48/48 passed). Benchmarks show a measurable improvement for some workloads (example: unsorted 5000/k=10 case improved by ~9% in my runs). Greatest future gains come from removing per-key heap allocations (a pool/arena or inline storage).

If you'd like, I can implement a per-grove arena allocator next (largest single performance improvement), or a bulk-loader path for sorted input. Tell me which direction to implement next.

---

File created: `docs/insertion_briefing.md`

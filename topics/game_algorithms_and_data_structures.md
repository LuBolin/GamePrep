# Algorithms and Data Structures for Game Interviews

See also: [[game_math_for_interviews]], [[ue_ai_navigation]], [[ue_networking_and_replication]], [[ue_hands_on_projects]], [[ue_interview_question_bank]].

Algorithm interviews test whether you can turn an ambiguous requirement into a correct, bounded and explainable solution. For a three-year game engineer, the strongest signal is not reciting every exotic structure: it is choosing a simple representation, stating invariants, deriving time/space cost, handling edge cases and connecting the result to game workloads.

## 1. The interview-solving loop

Before coding:

1. Restate the input, output and examples.
2. Ask for size bounds, update/query ratio, ordering, duplicates, mutability and memory constraints.
3. Identify whether exact or approximate output is required.
4. Give a correct baseline—even brute force—and its cost.
5. Name the repeated work or bottleneck that justifies a better structure.
6. State the invariant before implementation.
7. Walk one normal and one adversarial example.
8. Test empty, one-element, duplicate, disconnected, overflow and maximum-shape cases.
9. Discuss production concerns: allocations, cache locality, determinism, stale handles, threading and profiling.

This sequence prevents a common failure: coding a remembered algorithm that solves a nearby problem.

## 2. Complexity is a growth model, not a stopwatch

Big-O describes an asymptotic upper growth class after dropping constant factors and lower-order terms. Also state:

- what `n`, `V`, `E`, grid width/height or bucket occupancy mean;
- worst, expected/average or amortised case;
- time and auxiliary space;
- preprocessing/build versus update versus query;
- output-sensitive cost such as `O(k)` results;
- assumptions such as a good hash or non-negative edge weights.

Examples:

- scan unsorted enemies for nearest: `O(n)` time, `O(1)` extra space;
- sort then binary-search once: `O(n log n)` build plus `O(log n)` query—not a win for one query;
- hash lookup: expected `O(1)`, worst `O(n)` under collisions;
- BFS/DFS adjacency-list traversal: `O(V + E)`;
- binary-heap Dijkstra/A*: commonly `O((V + E) log V)` in the relevant implementation model;
- spatial query: structure-dependent and distribution-dependent; report candidates/output, update and rebuild cost instead of promising “logarithmic”.

Constants and data layout matter in games. A linear scan over a compact array can beat pointer-heavy `O(log n)` trees at realistic sizes. Complexity filters impossible approaches; measurement chooses among viable ones. [SRC-ALG-001] [SRC-ALG-002]

### Amortised analysis

A dynamic array append is usually constant amortised, even though occasional growth allocates and moves/copies many elements. A latency-sensitive frame can still feel that spike. Reserve capacity when a credible bound exists, avoid retaining invalidated pointers/iterators, and measure peak as well as average.

## 3. Data-shape questions that drive selection

Ask these before naming a container:

- Is order meaningful? Sorted, insertion, stable or irrelevant?
- Is lookup by integer index, key, priority, spatial region or relationship?
- Are inserts/removes rare, batched or per frame?
- Must references/indices remain stable?
- Is iteration hot and predictable?
- How many elements and how skewed are keys/positions?
- Is deterministic iteration/replay required?
- Is the data authoritative, replicated or rebuilt locally?
- Can tombstones, duplicate queue entries or deferred compaction simplify updates?

The best structure often combines two views—for example a dense array for iteration plus hash map from stable ID to dense index.

## 4. Core sequence structures

### Dynamic array (`std::vector`, `TArray`)

Strengths: contiguous storage, `O(1)` index, excellent iteration/locality, amortised append and easy sort/binary search. Costs: middle insert/erase shifts elements; growth and erasure can invalidate references/iterators; large objects make relocation expensive.

Default to a dynamic array for small-to-medium collections unless another access pattern is proven dominant. Use swap-remove when order is irrelevant. Use stable IDs/handles instead of persistent element addresses when relocation is possible. [SRC-ALG-002]

### Fixed array

Use a compile-time/fixed-capacity array when maximum size is intrinsic: board directions, fixed player slots, small lookup tables. It has no growth allocation and communicates the bound. Do not fake an unbounded collection with a silently truncating fixed buffer.

### Linked list

Linked lists offer constant-time splice/erase **given an iterator/node**, but finding that node remains linear. Nodes add allocation and pointer-chasing cost and lose random access/locality. They are awareness-level for most gameplay interviews; choose them only when stable nodes and frequent known-position splicing outweigh iteration cost.

### Deque, stack and queue

A deque supports efficient insertion/removal at both ends without contiguous whole-buffer relocation. A stack is LIFO: DFS, parser/evaluation state, undo or iterative recursion. A queue is FIFO: BFS, flood fill, event/work scheduling.

Use the abstract operation in the explanation. “BFS needs a queue because nodes must expand in non-decreasing edge count” is stronger than “I used a deque”. [SRC-ALG-002]

### Ring buffer

A bounded circular array supports predictable `O(1)` push/pop and allocation-free steady state. Define full policy: overwrite oldest, reject newest, grow or back-pressure. Uses include recent input/history for rollback, telemetry, audio/network buffers and fixed event queues.

```cpp
template <typename T, std::size_t N>
class RingBuffer
{
public:
    bool Push(const T& Value)
    {
        if (Count == N) return false;
        Data[(Head + Count) % N] = Value;
        ++Count;
        return true;
    }

    bool Pop(T& Out)
    {
        if (Count == 0) return false;
        Out = std::move(Data[Head]);
        Head = (Head + 1) % N;
        --Count;
        return true;
    }

private:
    std::array<T, N> Data{};
    std::size_t Head = 0;
    std::size_t Count = 0;
};
```

Invariant: live values occupy logical offsets `[0, Count)` from `Head`, modulo capacity.

## 5. Associative structures

### Hash table

Hash maps/sets give expected constant-time lookup/insertion/removal with a suitable hash and controlled load, but worst-case can be linear. Costs include buckets, slack, hash computation, collision handling, rehash invalidation and usually non-semantic iteration order. [SRC-ALG-003]

Good uses: ID→object/index, visited set, memoisation/cache, sparse grid cells and deduplication.

Correct key requirements:

- equality and hash agree: equal keys must hash equally;
- keys do not mutate in a way that changes hash/equality while stored;
- combine all identity fields with adequate distribution;
- do not depend on iteration order for deterministic gameplay;
- reserve if approximate count is known and rehash spikes matter.

For a dense bounded integer domain, direct-index array or bitset can be faster and smaller than hashing.

### Ordered tree map/set

An ordered associative tree commonly offers `O(log n)` lookup/insert/erase, sorted iteration and range queries. Choose it when ordering/predecessor/successor/range behaviour matters, not merely because worst-case complexity looks safer. Node allocation and pointer traversal can dominate small hot sets.

### Trie awareness

A trie indexes keys by prefix elements/characters. Lookup depends on key length rather than number of keys, but memory can be high. It fits autocomplete, command/tag prefix exploration or dictionary matching. A sorted array plus binary range search is often simpler for static data.

## 6. Heap and priority queue

A binary heap maintains a partial-order invariant: each parent outranks its children. A priority queue exposes the top in `O(1)` and inserts/removes top in `O(log n)` for the standard adaptor. [SRC-ALG-004]

Uses:

- A*/Dijkstra frontier;
- schedule next timer/event;
- top-K items;
- best-first search;
- streaming/merge of sorted inputs.

Common errors:

- comparator direction reversed (standard `priority_queue` is max-first by default);
- expecting efficient arbitrary deletion/decrease-key from an adaptor that does not expose it;
- mutating an item's priority while stored;
- forgetting deterministic tie-breaks;
- accepting stale duplicate entries without checking current best cost.

For A*, pushing a new entry when cost improves and skipping stale popped entries can be simpler than decrease-key. State the extra memory/work trade-off.

## 7. Sorting and searching

### Sorting

Comparison sorting has a typical `O(n log n)` target; `std::sort` requires random-access iterators and is not stable, while stable sorting preserves equivalent-element order at additional cost/requirements. [SRC-ALG-005]

Ask:

- Is a full sort needed, or only min/max/top-K/partition?
- Must equal elements retain previous order?
- Can integer/ranged keys use counting/radix/buckets?
- Is sorting once enabling many queries?
- Does comparator satisfy strict weak ordering?

Never write comparators such as `return A.Score <= B.Score`; equality must not claim both precede each other.

### Binary search and `lower_bound`

Binary search needs a sorted/partitioned range. `lower_bound` finds the first element not ordered before the target with logarithmic comparisons, but non-random-access iterators can still require linear increments; ordered containers' member lookup is preferable there. [SRC-ALG-006]

Boundary-safe half-open interval model:

```text
lo = 0, hi = n                 // answer lies in [lo, hi)
while lo < hi:
    mid = lo + (hi - lo) / 2
    if value[mid] < target: lo = mid + 1
    else: hi = mid
return lo                      // insertion point / first >= target
```

Half-open ranges make empty input and insertion at `n` natural. Test duplicates, target below/above range and overflow-safe midpoint.

### Selection

For one minimum/maximum, scan `O(n)`. For top K, a size-K heap is often `O(n log k)`. For kth order statistic, quickselect/`nth_element` gives average/typical linear-style selection without fully ordering everything (exact standard guarantee/version details should be checked). Choose the output actually required.

## 8. Recursion and iteration

Recursion mirrors trees/graphs but consumes call stack and can overflow on deep/adversarial input. Iterative traversal makes frontier storage and budget/resumption explicit. In a frame-budgeted game, an explicit queue/stack can process N nodes per frame, cancel safely and expose progress.

Every recursive solution needs:

- base case;
- progress toward base case;
- maximum depth reasoning;
- ownership/lifetime of shared mutable state;
- cycle handling if the structure is a graph, not a tree.

Memoisation caches overlapping subproblems; dynamic programming chooses an evaluation order and stores necessary prior state. Do not call plain recursion “DP”.

## 9. Trees and graph representation

### Tree concepts

A tree is a connected acyclic graph. Know root/parent/child/depth/height/leaf/subtree. Traversals:

- preorder: process node before children—serialisation/copy/hierarchy display;
- postorder: children before node—size aggregation/destruction;
- inorder: meaningful for binary search tree sorted order;
- level order: BFS by depth.

Balanced search trees preserve logarithmic height; an ordinary BST can degrade to a chain.

### Graph representation

- adjacency list: `O(V + E)` storage, efficient sparse-neighbour iteration;
- adjacency matrix: `O(V²)` storage, constant edge membership, useful for small dense graphs;
- implicit graph: generate neighbours from a grid/state rather than store all edges.

Clarify directed/undirected, weighted/unweighted, negative weights, parallel edges and connectivity before choosing an algorithm.

## 10. BFS and DFS

Both visit each reachable vertex/edge once with proper visited tracking: `O(V + E)` adjacency-list time and `O(V)` auxiliary state.

### BFS

FIFO frontier. Finds shortest path by number of edges in an unweighted/equal-cost graph. Useful for grid range, flood fill, nearest matching state and distance field.

Mark visited when enqueuing, not when dequeuing, to avoid repeated queue entries. Store `came_from` only if reconstructing a path.

### DFS

Stack/recursion frontier. Useful for reachability, components, cycle detection, topological reasoning, maze generation and exhaustive backtracking. It does not generally find shortest paths.

### Disconnected graphs

One traversal from a start visits one reachable component. To process all components, iterate all vertices and start a traversal at each unvisited vertex.

## 11. Dijkstra and A*

### Dijkstra

Dijkstra finds shortest paths with non-negative edge weights. It expands the lowest known path cost `g(n)` from a min-priority frontier. Relax edge `(u,v)` when `g(u)+w(u,v)` improves `g(v)`. Negative edges invalidate the settled-cost reasoning.

### A*

A* orders by:

```text
f(n) = g(n) + h(n)
```

where `g` is known start cost and `h` estimates remaining cost. With a suitable admissible heuristic, A* retains optimality; consistency simplifies graph-search behaviour. `h=0` reduces to Dijkstra. An overestimating/weighted heuristic can trade optimality for fewer expansions if the product accepts that trade. [SRC-ALG-007] [SRC-ALG-008]

Match heuristic to movement/cost:

- Manhattan for four-direction unit grid;
- Chebyshev/octile-style reasoning for allowed diagonal rules;
- Euclidean when movement/cost permits straight-line lower bound;
- nav graph distance lower bound according to edge semantics.

Multiplying a heuristic arbitrarily can make it inadmissible. Also distinguish **pathfinding** (global route) from **path following/steering/avoidance** (executing route around dynamic neighbours).

### Production pathfinding questions

- graph representation/navmesh/grid/waypoints;
- path quality versus search time;
- dynamic obstacle invalidation and replanning;
- requests per frame and scheduling/budget;
- shared paths, hierarchical search or flow fields for crowds;
- deterministic tie-break/order;
- partial path and cancellation policy;
- path smoothing/string pulling;
- instrumentation: expanded nodes, frontier peak, cost, time and cache hit.

Red Blob's implementation demonstrates BFS, Dijkstra and A* as the same graph framework with different frontier/cost rules, and its grid optimisation guide highlights reducing graph size and queue work. [SRC-ALG-007] [SRC-ALG-009]

## 12. Topological sort

A topological order exists only for a directed acyclic graph (DAG): every edge `u→v` places `u` before `v`.

Kahn's algorithm:

1. compute indegree of each vertex;
2. enqueue all zero-indegree vertices;
3. pop one, append it, decrement neighbours;
4. enqueue newly zero-indegree neighbours;
5. if output count is less than `V`, a cycle exists.

Cost is `O(V + E)`. Uses: build dependencies, quest prerequisites, processor/system schedule, asset pipeline. Deterministic builds need deterministic selection among multiple zero-indegree nodes.

DFS finishing-order is another method; colour states detect a back edge/cycle. Report a useful cycle path rather than only “sort failed”. [SRC-ALG-010]

## 13. Union–find / disjoint-set union

DSU maintains partitioned sets with:

- `Find(x)`: representative of x's component;
- `Union(a,b)`: merge components.

Path compression plus union by rank/size yields effectively near-constant amortised operations (`O(α(n))`). It is excellent for incremental connectivity and Kruskal-style minimum spanning trees, procedural room/region connectivity or offline grouping. It does **not** answer shortest paths, and ordinary DSU does not handle edge deletion well. [SRC-ALG-011]

Invariant: every element follows parent links to one root; root metadata such as size/rank is updated only on union.

## 14. Weighted random and sampling

For non-negative weights `w[i]`:

1. compute total `W`;
2. sample `r` uniformly in `[0,W)`;
3. return first prefix sum greater than `r`.

Linear scan is `O(n)` per draw. Precomputed prefix sums plus binary search give `O(n)` build, `O(log n)` draw when weights are static. Alias tables can give `O(1)` draws after preprocessing at greater complexity. Dynamic weights change the choice; a Fenwick tree can support prefix updates/search in `O(log n)`.

Define zero-total policy, reject negative/NaN weights, use sufficient precision and specify deterministic RNG seed/stream. “Random” does not mean globally uniform when weights differ.

### Shuffle bag

For anti-streak design, put weighted/repeated outcomes in a bag, shuffle and consume; this changes statistical independence deliberately. State that product choice rather than mislabelling it weighted random.

## 15. Grid and spatial hash

A uniform grid/spatial hash maps a position/AABB to discrete cells. Query checks relevant cells and then performs exact filtering.

```text
cell.x = floor(position.x / cellSize)
cell.y = floor(position.y / cellSize)
```

Use mathematical floor for negative coordinates; truncation toward zero aliases cells around the origin. Hash signed integer cell coordinates as integers, not a collision-prone string.

### Cell-size trade-off

- too small: an object overlaps many cells; update/query overhead rises;
- too large: each cell contains many false candidates;
- mixed object sizes: multi-cell insertion, hierarchical grids or separate large-object path may help.

Policies to define:

- insert by centre or all overlapped cells;
- deduplicate candidates found in multiple cells;
- update only when cell coverage changes;
- handle huge/out-of-bounds objects;
- rebuild versus incremental mutation;
- thread/read snapshot strategy;
- worst-case clustering (all objects in one cell remains `O(n²)` pair testing).

Spatial partitioning reduces candidate pairs; it does not perform exact collision or guarantee good distribution.

## 16. Quadtrees, octrees, k-d trees and BVHs

### Quadtree / octree

Recursively subdivide 2D/3D axis-aligned space into fixed children. Useful when spatial extent is bounded and distribution benefits from adaptive resolution. Challenges: objects spanning boundaries, loose bounds, update churn, empty-node overhead and depth limits.

### k-d tree

Binary split by dimension/hyperplane, effective for point nearest-neighbour/range queries, especially relatively static data. Frequent movement can require costly maintenance/rebuild; high dimension degrades usefulness.

### BVH

A bounding volume hierarchy groups object bounds in a tree. Traversal rejects whole subtrees whose bounds miss the query. Common in ray tracing and collision broadphase; build quality, refit/rebuild policy and coherent traversal determine performance. Real-Time Collision Detection organises broadphase, BVH, spatial partitioning and primitive tests as separate layers. [SRC-ALG-012]

### R-tree awareness

R-trees are balanced spatial indexes for boxes/geometry and support intersection/nearest queries. Boost's documented R-tree illustrates storing box+ID and querying intersecting or nearest values. [SRC-ALG-013]

### Selection matrix

| Workload | First candidate | Main caution |
|---|---|---|
| uniformly distributed moving 2D agents | uniform grid/spatial hash | tune cell size; clustering |
| bounded adaptive 2D/3D region | quadtree/octree | straddlers and update churn |
| static point nearest-neighbour | k-d tree | dynamic rebuild/high dimensions |
| rays/collision objects with bounds | BVH | build/refit quality |
| dynamic geometry boxes/range query | R-tree | node/update/query constants |
| small N | flat array scan | often simplest and fastest |

Benchmark the actual distribution, motion and query types. Average random data can hide the exact clustered formation a game creates.

## 17. Broadphase and narrowphase

Broadphase cheaply generates plausible collision/interest pairs using bounds/partitioning. Narrowphase runs accurate shape/geometry tests only for candidates. The same architecture appears in:

- physics collision;
- AI neighbour queries;
- network interest management;
- rendering culling;
- interaction targeting.

False positives are acceptable in broadphase; false negatives are not. Track candidate count, true-hit ratio, duplicate pairs, update/build time and maximum bucket/node occupancy.

AABB overlap in each axis is cheap and conservative. OBB/sphere/convex/mesh tests belong later according to the exact shape and accuracy needed. The maths chapter derives the primitive tests.

## 18. Noise awareness

Noise functions generate spatially coherent pseudo-random values useful for terrain, texture, animation variation and procedural parameters. Distinguish:

- white/unrelated random samples;
- value/gradient/simplex-style coherent noise families;
- fractal combinations (octaves, frequency/lacunarity, amplitude/persistence);
- domain warping and thresholding.

Noise is not automatically terrain, deterministic across every implementation/platform, or free of visible axis/repetition artefacts. Record seed, coordinate scale, output range and implementation; profile octave/sample count.

## 19. Network-algorithm awareness

### Interest management

Use spatial/team/visibility/relevance structures to avoid evaluating/sending every entity to every connection. The structure returns candidates; authoritative relevance rules filter them. Updates, hysteresis and per-connection budgets are as important as query cost.

### Snapshot interpolation

Store timestamped authoritative snapshots in a bounded buffer and render behind latest server time, interpolating between surrounding snapshots. Define packet loss, extrapolation cap, teleport/discontinuity and buffer-delay policy.

### Lag compensation

Retain bounded historical collision state, rewind to a validated client fire time, perform server-authoritative query and restore. This is history/query architecture plus security policy—not simply “trust client timestamp”.

### Rollback

Store/reconstruct prior deterministic-enough state and input, restore on late authoritative input, then resimulate. Cost depends on state size, simulation duration, side-effect suppression and determinism. Use ring buffers, stable frame IDs and idempotent presentation boundaries.

These are architecture awareness topics; networking implementation details remain in the networking chapter.

## 20. Common coding patterns

### Two pointers

Maintain two indices into sorted/partitionable data: pair sum, compaction, partition, interval merge. State what region each pointer has already proven.

### Sliding window

Maintain an aggregate over a contiguous interval, expanding right and shrinking left while an invariant holds. Works when removing left contribution is possible and the predicate has suitable monotonic behaviour.

### Prefix sums / difference arrays

Prefix sums turn static range-sum queries into `O(1)` after `O(n)` preprocessing. Difference arrays batch range additions then materialise once. Extend to 2D grids carefully with inclusion–exclusion and bounds.

### Monotonic stack/queue awareness

Maintain increasing/decreasing candidates while removing dominated elements: next greater element, sliding-window extrema, visibility spans. The amortised linear argument is that each element is pushed and popped at most once.

### Bitsets

Compact bounded Boolean sets and enable word-wide intersection/union. Useful for visibility masks, fixed ability/category sets and graph reachability at bounded scale. Document bit-index schema and avoid opaque magic masks.

## 21. Debugging algorithm failures

### Wrong answer

1. Restate invariant and preconditions.
2. Reduce to the smallest counterexample.
3. Log state transitions, not just final output.
4. Check boundary/index/range convention.
5. Check duplicate/cycle/visited timing.
6. Check comparator/hash/equality contract.
7. Check integer overflow, float NaN and sentinel collision.
8. Compare against a slow reference implementation with generated small inputs.

Property/differential tests are powerful: compare spatial-hash neighbours with all-pairs; A* cost with Dijkstra; binary-search insertion with sorted result; DSU connectivity with small BFS.

### Performance miss

Measure input distribution and phases:

- build/update/query time;
- allocations and bytes;
- visited/expanded/candidate counts;
- frontier/stack/bucket maximum;
- cache misses/data layout where tooling permits;
- worst/P95/P99, not only average;
- frame amortisation and cancellation latency.

Then decide whether the issue is asymptotic work, constant/layout, rebuild frequency, poor distribution, excessive result size or downstream narrowphase.

## 22. Common misconceptions

| Misconception | Better model |
|---|---|
| “Hash lookup is O(1).” | Expected constant under assumptions; collisions/load can degrade and rehash costs exist. |
| “Binary search is always O(log n).” | Comparisons are logarithmic; iterator movement may not be, and sorting/build cost matters. |
| “Linked list inserts are O(1), so it is faster.” | Only once position is known; allocation and locality often dominate. |
| “A* is faster Dijkstra.” | It is Dijkstra-style best-first search informed by a heuristic; benefit/correctness depend on heuristic/graph. |
| “DFS finds a path, so it finds the shortest.” | DFS finds some reachable path; BFS is shortest by edge count in unweighted graphs. |
| “Spatial hash makes collision O(n).” | It reduces candidates under favourable distribution; worst-case clustering remains quadratic. |
| “Quadtree is better than a grid.” | Choice depends on extent, distribution, motion and query/update ratio. |
| “Big-O decides fastest implementation.” | It excludes poor scaling; realistic constants/layout and distributions require measurement. |
| “Random seed makes all platforms deterministic.” | Algorithm, floating-point, iteration order and state consumption must also match. |
| “Optimised solution is the interview goal.” | Correct constraints, invariant, edge cases and communication come first. |

## 23. Strong interview answer patterns

### Choosing a container

> I ask which operations dominate, whether order/stable references/determinism matter and the maximum count. I default to a contiguous array for iteration and simple ownership. If key lookup dominates, I may add a hash index while retaining dense storage. I state expected versus worst-case hash behaviour, invalidation and memory cost, then benchmark realistic counts before choosing a node-based tree.

### Explaining A*

> A* keeps the cheapest frontier by `f=g+h`, where `g` is cost already paid and `h` estimates remaining cost. With non-negative costs and an admissible/consistent heuristic it can preserve optimality while expanding fewer nodes than Dijkstra; `h=0` is Dijkstra. I match the heuristic to movement rules, store best cost/came-from, tolerate or update stale frontier entries deliberately, and measure expansions/frontier/time under representative maps.

### Designing neighbour queries

> I start with all-pairs as a correctness oracle. For many moving similarly sized agents, I try a uniform grid/spatial hash: insert into overlapped cells, query relevant cells, deduplicate and exact-filter candidates. Cell size balances cell coverage against bucket occupancy. I test negative coordinates, large objects and clustered worst cases, and compare build/update/query plus candidate-to-hit ratio against the baseline.

## 24. Hands-on verification suite

1. Build a small algorithm harness with deterministic seed, timers and table-driven/property tests.
2. Implement lower-bound binary search and compare every insertion point with the standard algorithm.
3. Implement BFS, Dijkstra and A* over one graph interface; verify A* cost against Dijkstra and visualise expanded nodes.
4. Implement topological sort with deterministic tie-break and useful cycle report.
5. Implement DSU with path compression/union-by-size and compare random connectivity against BFS.
6. Implement weighted random linear and prefix/binary variants; run distribution/zero-weight tests.
7. Implement all-pairs and spatial-hash neighbour queries for moving agents; compare exact result sets and profile uniform/clustered/mixed-size cases.
8. Add a rollback-style ring buffer with wrap, overwrite/reject and frame-ID tests.
9. For each solution, present baseline, invariant, complexity, adversarial input, memory/invalidation and game use in five minutes.

## 25. Source map

- complexity and standard container/algorithm contracts: [SRC-ALG-001] [SRC-ALG-002] [SRC-ALG-003] [SRC-ALG-004] [SRC-ALG-005] [SRC-ALG-006]
- BFS/Dijkstra/A* and grid optimisation: [SRC-ALG-007] [SRC-ALG-008] [SRC-ALG-009]
- topological sort and union-find: [SRC-ALG-010] [SRC-ALG-011]
- collision/spatial indexes: [SRC-ALG-012] [SRC-ALG-013] [SRC-ALG-014]

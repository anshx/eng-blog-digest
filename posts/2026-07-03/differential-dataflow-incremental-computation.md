---
title: "Building Differential Dataflow from Scratch"
source: https://materialize.com/blog/differential-from-scratch/
author: Materialize Engineering
company: Materialize
date_posted: 2023-02-08
date_digested: 2026-07-03
---

# Building Differential Dataflow from Scratch

## What's new to learn

1. **Signed multisets as the core abstraction** — Differential dataflow represents collections not as sets of rows but as functions from `(element, timestamp) → integer`. Positive integers mean insertions, negative mean deletions. This algebraic trick is what makes composition of incremental operators correct.

2. **The differential join rule** — Every relational operator has an incremental (differentiable) form. The join version is `Δ(A ⋈ B) = (ΔA ⋈ B_old) + (A_old ⋈ ΔB) + (ΔA ⋈ ΔB)` — the Leibniz product rule applied to database tables instead of real numbers.

3. **Multi-dimensional logical timestamps for iterative algorithms** — When computation contains loops (PageRank, shortest paths, strongly connected components), timestamps become 2D vectors `(epoch, iteration)`, forming a partial order. A monotonically advancing *progress frontier* lets the system determine when iteration `i` is complete and compact historical data below it.

## Prerequisites

- Dataflow programming: operators (nodes) connected by edges, data flows through
- Relational algebra: map, filter, join, group-by and what each one does
- Rough understanding of why batch-vs-streaming is a tradeoff (the archive's streaming/watermarks post is useful context)
- The calculus product rule `d(fg) = f'g + fg'` — the intuition, not the formalism

## The core idea

Every time you run `SELECT * FROM orders JOIN users ON orders.user_id = users.id WHERE total > 100`, your database re-scans both tables from scratch. If you have 10 million orders and 1 arrived in the last second, you still paid O(N) to find it.

Differential dataflow asks: *what is the change in the output for a given change in the input?* If only 1 order was inserted, the join output changes only by the rows that new order participates in — O(matching rows in users), not O(all orders).

The key abstraction is a **signed multiset** (called a *collection* in the codebase). Instead of representing a table as a set of rows `{row1, row2, ...}`, you represent it as a function:

```
(element, timestamp) → integer
```

- `+1` means "this element was inserted at this timestamp"
- `-1` means "this element was deleted at this timestamp"
- The *current view* at time `t` is any element whose multiplicities sum to `> 0` across all `t' ≤ t`

Example: a users table over three time steps.

```
(Alice, 30)    t=0  → +1    # Alice inserted
(Bob,   25)    t=0  → +1    # Bob inserted
(Bob,   25)    t=1  → -1    # Bob deleted
(Charlie, 28)  t=1  → +1    # Charlie inserted
```

At `t=0`: `{Alice, Bob}`. At `t=1`: `{Alice, Charlie}`. The table at each time is recoverable from the signed history.

This representation is an abelian group: you can add and subtract collections just like integers. That algebraic property is what lets you compose incremental operators correctly.

## Mechanics

### Differentiating map and filter

For **map** with function `f`: if the input changes by `Δ`, the output changes by applying `f` to every element in `Δ`. No historical state needed.

For **filter** with predicate `p`: if the input changes by `Δ`, the output changes by filtering `Δ`. Again, no historical state.

Both are O(|Δ|) — proportional to the change, not the full collection.

### Differentiating join — the product rule

For `A ⋈ B` (join on a key), if both A and B change simultaneously:

```
Δ(A ⋈ B) = (ΔA ⋈ B_old) + (A_old ⋈ ΔB) + (ΔA ⋈ ΔB)
```

This is the Leibniz product rule: `d(fg) = f'g + fg' + f'g'`. Each term handles one direction of change:
- New rows in A joined against the existing B
- Existing A joined against new rows in B
- New rows in A joined against new rows in B (often zero if only one side changes per batch)

The cost is O(|ΔA| × key-fanout in B + |A| × key-fanout in ΔB). When changes are small, this is far cheaper than re-running the full join.

### The arrangement: indexed versioned state

To compute `ΔA ⋈ B_old`, you need to quickly look up all B rows matching a given key *as of the previous time*. This requires B to be indexed.

The **arrangement** is the core data structure: a sorted, versioned index of `(key, value, timestamp, diff)` tuples, similar to a B-tree but multi-versioned. It answers "give me all values for key `k` as of time `t`" in O(log n + results) time.

Arrangements are **immutable and shareable**. If two operators both need to look up the same collection by key, they share one arrangement — like a secondary index in a relational database, but one that persists across all past times. When B is arranged, `ΔA ⋈ B_old` becomes a fast indexed lookup for each row in ΔA.

Concretely, an arrangement is built by merging batches of `(key, value, timestamp, diff)` tuples into increasingly large sorted arrays — the same LSM tree structure as RocksDB, but in memory and with a richer key structure.

### Group-by and aggregation

For `GROUP BY key, COUNT(*)`: when a row is inserted with key `k`, the count for `k` increases by 1. When deleted, it decreases.

The tricky part: the arrangement for the aggregate must track the *previous* count to emit a retraction `(k, old_count, -1)` alongside the assertion `(k, new_count, +1)`. This way, downstream operators always see a net change, never stale state.

### Iterative computation with multi-dimensional timestamps

Many useful algorithms are iterative: PageRank converges over rounds, BFS discovers nodes layer by layer, strongly connected components require multiple passes.

In differential dataflow, a loop is a *feedback edge* in the dataflow graph. Output feeds back as input for the next iteration, with timestamps `(epoch, iteration)`:

```
epoch=0, iteration=0: initial edges injected
epoch=0, iteration=1: new paths discovered via one-hop joins
epoch=0, iteration=2: paths discovered via two-hop joins
...
epoch=0, iteration=k: fixed point (no new paths)
epoch=1, iteration=0: new edges injected (a graph update arrived)
```

A **progress frontier** tracks the lowest timestamp that is not yet fully computed. When the frontier advances past `(e, i)`, the system knows all inputs at that timestamp have been processed. It can then:
1. Deliver all output changes at `(e, i)` to downstream operators
2. Compact (garbage-collect) historical state below the frontier

This is equivalent to the watermark concept in streaming systems: the frontier is a monotonically advancing bound on "I have processed everything before this point."

The invariant for correctness: never deliver changes at timestamp `t` until the frontier has advanced past `t`. This ensures that each iteration sees a consistent view of the previous iteration's output.

### Compaction

Over time, an arrangement accumulates many `(element, t1, +1)` and `(element, t2, -1)` pairs. Compaction merges these below the frontier: if the frontier is at `t`, any pair where both timestamps are `< t` can be summed and, if the result is 0, deleted entirely. This keeps memory bounded as long as data changes less often than compaction runs.

### A worked example: transitive closure

Suppose you want to maintain all pairs `(a, b)` such that there is a directed path from `a` to `b` in a graph, and update this as edges are added or removed.

```
edges = {(A→B), (B→C)}
reach(0) = edges                            # direct connections
reach(k+1) = reach(k) ∪ (reach(k) ⋈ edges) # extend by one hop
```

In differential dataflow:
1. Inject the initial edges at `(epoch=0, iteration=0)`.
2. The join `reach ⋈ edges` produces new paths at `(epoch=0, iteration=1)`.
3. Those new paths feed back, producing more paths at `iteration=2`, etc.
4. When no new paths are produced (empty delta), the fixed point is reached.
5. If a new edge `(C→D)` arrives at `epoch=1`, only the paths that use that edge are recomputed — differential propagation through the loop, not a full restart.

Without differential dataflow, step 5 requires re-running all iterations from scratch. With it, only the affected paths propagate forward.

## Where it breaks

**Distinct and deduplication are expensive.** The `DISTINCT` operator must track the full count of each element (not just presence/absence), so it can correctly determine when the last copy of an element is deleted. This requires an arrangement keyed by element, adding overhead proportional to unique elements.

**Hot keys destroy parallelism.** Arrangements are sharded by key. If one key has disproportionately many values (a celebrity user with millions of followers), the worker handling that shard is a bottleneck. Standard mitigation: re-key with `(key, hash_of_value)`, which splits the shard but requires knowing the skew in advance.

**Compaction can lag under sustained load.** If updates arrive faster than compaction merges them, the arrangement grows without bound. This requires careful tuning of merge policies (how often to compact, at what size thresholds).

**Not every computation has a cheap derivative.** Median, top-k with arbitrary tie-breaking, and some graph algorithms have no efficient incremental form — the only correct approach is to recompute from scratch (or use approximate structures).

**Debugging negative multiplicities is hard.** An element with multiplicity `-2` in an arrangement means two deletes without matching inserts — a bug upstream. Tracing it requires understanding the full causal history of timestamps, which is non-obvious.

## Why it works

The fundamental insight: **every database operator is a mathematical function from collections to collections, and mathematical functions have derivatives.**

In calculus, `d(fg)/dx = f'g + fg'`. In differential dataflow, `Δ(A ⋈ B) = (ΔA ⋈ B) + (A ⋈ ΔB)`. The structural identity is exact: join is multiplication of characteristic functions, and the product rule follows.

The signed multiset formalism makes this rigorous. Natural-number multiplicities (bags) form a commutative *monoid* — you can add elements but not subtract them. Lifting to integers (allowing negatives) gives a commutative *group*: now subtraction (deletion) is well-defined, and operator composition is linear (i.e., `F(A + B) = F(A) + F(B)`). Linearity is precisely what lets you write `Δ(F(A)) = F(ΔA)` for map and filter.

Join is not linear (it's bilinear), which is why the product rule has cross terms. But the algebraic structure still holds, and the cross term `ΔA ⋈ ΔB` is usually cheap (both sides rarely change simultaneously).

**The deeper principle: push work proportional to change, not to state size.** This same principle appears elsewhere:

- **Event sourcing**: store a log of changes (inserts/updates/deletes) and replay to get current state. Differential dataflow does this but also *propagates* changes forward through a computation graph without full replay.
- **Git's delta compression**: store only the difference between versions. A diff is the signed change to a file; applying it is the differential operator.
- **Functional reactive programming (FRP)**: values vary over time, changes propagate through a dependency graph. Differential dataflow is the database/analytics version of FRP — with the addition of joins, aggregations, and iterative loops.
- **The streaming watermark** (covered 2026-06-20): the progress frontier in differential dataflow is exactly a watermark — a monotonically non-decreasing bound on processed timestamps. The streaming post shows why such a bound is necessary for correctness; differential dataflow shows how to use it for historical compaction too.
- **MapReduce** (covered 2026-06-25): MapReduce is differential dataflow with one timestamp and no shared arrangements. Every job re-scans full datasets; differential dataflow makes every step incremental. The monoid requirement for MapReduce reducers is the same abelian group requirement here.

## Going deeper

1. **Naiad: A Timely Dataflow System** (Murray et al., SOSP 2013) — The execution engine underlying differential dataflow. Explains the structured loop model, the pointstamp (timestamp vector) protocol, and how progress tracking achieves coordination-free iteration. https://dl.acm.org/doi/10.1145/2517349.2522738

2. **DBSP: Automatic Incremental View Maintenance for Rich Query Languages** (Budiu, McSherry et al., VLDB 2023) — A formal algebraic treatment showing that *any* SQL query (including recursive CTEs) can be automatically incrementalized by compiling it to Z-set operators. More rigorous than the original differential dataflow paper and covers SQL-specific patterns. https://arxiv.org/abs/2203.16684

3. **Frank McSherry's blog** (https://github.com/frankmcsherry/blog) — McSherry regularly posts annotated Rust implementations of differential dataflow internals, working through specific algorithms (strongly connected components, graph connectivity, Datalog) with actual performance numbers. The most direct path from "I understand the idea" to "I understand the implementation."

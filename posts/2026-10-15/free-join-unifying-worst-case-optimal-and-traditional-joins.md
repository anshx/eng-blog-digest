---
title: "Free Join: Unifying Worst-Case Optimal and Traditional Joins"
source: https://arxiv.org/abs/2301.10841
author: Yisu Remy Wang, Max Willsey, Dan Suciu
company: University of Washington
date_posted: 2023-06-01
date_digested: 2026-10-15
---

# Free Join: Unifying Worst-Case Optimal and Traditional Joins

## What's new to learn

1. **Worst-case optimal join (WCOJ)**: A multiway join algorithm that runs in O(AGM) time — the information-theoretically minimum upper bound on output size — by iterating over attributes one at a time and intersecting relation indexes, rather than joining two relations at a time. For some cyclic queries (e.g., triangle counting), this is *exponentially* faster than any binary join plan.

2. **Tries as a unified join data structure**: Representing each relation as a nested hash map (a trie keyed on its attributes in order) lets you treat binary hash join and WCOJ as the *same* trie-descent algorithm, differing only in how many relations you intersect per descent step.

3. **Free Join plan**: A hybrid plan type that mixes binary-join-style steps (descend multiple levels of one trie per step) and WCOJ-style steps (descend one level across all tries simultaneously), so the optimizer can pick the best strategy for each subquery independent of the others.

## Prerequisites

- **Relational join**: inner join of two tables on matching key columns; result size can be up to |R|·|S|.
- **Hash join**: build a hash table on the smaller relation, probe it with each row of the larger; dominant in practice.
- **Join ordering**: for a multi-way join R₁ ⋈ R₂ ⋈ R₃, the order in which you execute the binary steps matters enormously for intermediate result size.
- **Acyclic vs. cyclic queries**: a query hypergraph is acyclic if it has a join tree (each variable appears in a contiguous subtree); cyclic queries — like triangle counting `R(a,b) ⋈ S(b,c) ⋈ T(a,c)` — have no such tree and are harder to optimize.

Non-obvious prerequisite: the **AGM (Atserias–Grohe–Marx) bound** — the tight worst-case upper bound on join output size in terms of fractional edge covers of the query hypergraph. For a triangle query over three relations each of size N, the AGM bound is N^{3/2}, not N^2.

## The core idea

The paper's insight is a change of representation: stop thinking of a relation as a flat table and start thinking of it as a **trie** — a nested hash map where each level corresponds to one attribute.

For relation R(a, b) with attribute order (a, b):
```
Trie(R) = { a₁ → { b₁, b₃ },
             a₂ → { b₂ },
             ... }
```

Now observe:

**Hash join is trie descent with full lookup on the inner side.** When you probe the build-side hash table with a key `a=v`, you retrieve the entire bucket `{b : (v,b) ∈ R}` in a single step. In trie terms, you descend one level for `a` and get the complete sub-trie.

**WCOJ (Generic Join) is synchronized single-level descent across all tries.** At each step, you have a partial assignment of variables (e.g., `a=v`) and you extend it by one variable. You intersect the sets of valid values across *all* relations that contain that variable, picking only values that appear in every relation simultaneously.

**Free Join chooses the descent granularity per plan node.** A plan node specifies:
- A set of relations ("atoms") participating in this node
- Which variables are "bound" (from ancestors) vs. "free" (being iterated here)
- How deeply to descend each atom's trie in one step

If a node descends a single trie multiple levels before intersecting: binary-join behavior (probe, emit, then recurse). If a node intersects multiple tries at the same level: WCOJ behavior. A single plan can use both modes in different subtrees.

The unification becomes: *"every join algorithm is a strategy for visiting leaves of the product-relation trie, differing only in the order and granularity of trie descent."*

## Mechanics

**Running example: triangle query**

```sql
SELECT COUNT(*) FROM R(a,b), S(b,c), T(a,c)
WHERE R.b = S.b AND S.c = T.c AND R.a = T.a
```

**Approach 1 — Binary join plan:**
```
Step 1: tmp₁ = R ⋈_b S      # build hash(S) on b, probe with R.b
Step 2: result = tmp₁ ⋈_{a,c} T
```
Worst case: if every vertex connects to every other, |tmp₁| = Θ(N²), and the final join also takes Θ(N²). For N=10,000 edges, that's 10⁸ intermediate tuples.

**Approach 2 — Generic Join (WCOJ):**
Attribute order: a, b, c.

```
for each value v_a in (values of a in R) ∩ (values of a in T):
    for each value v_b in (values of b in R where a=v_a) ∩ (values of b in S):
        for each value v_c in (values of c in S where b=v_b) ∩ (values of c in T where a=v_a):
            emit (v_a, v_b, v_c)
```

Each intersection is a set operation. With sorted lists or hash sets, the cost is bounded by O(N^{3/2}) — the AGM bound for triangles. No intermediate tuple is materialized unless it is part of the final output.

**Approach 3 — A Free Join plan:**

```
for each value v_a in Trie(R)[a] ∩ Trie(T)[a]:    # WCOJ: intersect R and T on 'a'
    for each v_b in Trie(R)[a=v_a][b]:            # binary: scan R's b-values for this a
        for each v_c in Trie(S)[b=v_b][c]:        # binary: scan S's c-values for this b
            probe Trie(T)[a=v_a][c=v_c]           # binary: point-probe T on (a,c)
            if hit: emit (v_a, v_b, v_c)
```

The first loop is WCOJ-style (intersect R and T to find shared `a` values), while the inner loops are binary-join-style (scan one trie, probe another). This achieves O(N^{3/2}) worst-case while keeping the constant factors close to a hash join.

**The trie implementation**

A trie over R(a,b) is built once as a two-level `HashMap<a, HashSet<b>>`. For a join node that "binds" attribute `a` from its parent and iterates over `b`, the lookup is: `trie[v_a]` → iterate over the set. For intersection of two tries on `a`, Free Join uses the smaller set to probe the larger: for `v` in `smaller.keys()`, yield `v` only if `larger.contains_key(v)`.

**Optimizer and cost model**

Free Join's optimizer must choose:
1. Which attributes to intersect (WCOJ-style) vs. which to probe (binary-style)
2. The depth of trie descent per plan node
3. The variable order within WCOJ-style nodes

The optimizer enumerates Free Join plans by extending the Cascades framework: it considers both "widen the node" (add more relations to an intersection) and "deepen a descent" (move a relation's lookup into an inner loop). The cost model estimates intersection size using relation statistics.

## Where it breaks

**Bad statistics ruin the plan.** The optimizer's choice between binary and WCOJ-style steps rests on estimated intersection sizes. If statistics are stale or skewed (one hub vertex has 10× average degree), the plan can pay O(N²) even though WCOJ would give O(N^{3/2}).

**Trie build cost.** Building two-level hash maps is slower than building a flat hash table. For two-relation joins, the trie overhead makes Free Join slower than a standard hash join — the optimizer should fall back to binary join for small joins.

**The plan space explodes.** Traditional join planning considers N! orderings for N relations. Free Join adds another axis: how to group relations into plan nodes. This makes optimal Free Join planning NP-hard in general, so the optimizer uses heuristics.

**Pointer chasing in deep tries.** Each trie level is a heap-allocated hash map, so deep tries cause O(depth) cache misses per lookup. Flat hash tables with composite keys avoid this. For queries with long attribute chains and small selectivity, flat hash tables with careful schema design beat tries.

**Distributed settings are unsolved.** Free Join assumes a shared-memory single-node execution model. Extending it to distributed joins — where you need to ship data across machines — is an open problem. Distributed WCOJ exists but requires specialized data partitioning that is not yet integrated with Free Join plans.

**Doesn't cover inequality joins or NL joins.** Free Join only handles equi-joins (equality predicates). Range joins or theta-joins require different data structures (interval trees, sort-merge) that don't fit the trie model naturally.

## Why it works

The deeper principle: **join evaluation is trie descent, and every join plan is a traversal order over the cross-product trie.**

Imagine the "virtual" relation R₁ × R₂ × ... × Rₙ — all possible tuple combinations. An equi-join filter selects a subset of this cross-product. Any join algorithm enumerates exactly that subset, differing only in *which parts of the cross-product it materializes as it goes*.

Binary hash join materializes full intermediate relations (some of which may be large) and then filters at the next join. WCOJ never materializes any intermediate tuple until it has verified that it satisfies *all* join predicates simultaneously. Free Join interpolates: it materializes intermediate results only when the plan node's selectivity is high enough that the materialized size is small.

The "oh X is just Y" insight: **WCOJ is binary join with a fully decomposed key.** Hash join looks up a composite key `(a=v₁, b=v₂)` in a hash map in one step; WCOJ applies the same lookup but splits it into two steps: look up `a=v₁` in a level-1 hash map, then look up `b=v₂` in the level-2 hash map — and intersects across all relations at each level. The split costs one extra indirection but gains the ability to prune at each level before descending further.

This connects to several patterns in the archive:
- **Adaptive Radix Tree** (ART): also a trie-descent structure where the node type adapts to density, same principle as Free Join adapting step size to data distribution.
- **Cascades query optimizer**: Free Join extends Cascades with a new "group relations into plan nodes" rewrite rule.
- **CALM theorem**: WCOJ is a coordination-free algorithm exactly because it is monotone — each step only adds output tuples — connecting join correctness to the monotonicity principle.
- **Scalability at COST**: Free Join is a reminder that algorithmic improvement (WCOJ vs binary join for cyclic queries) can dwarf scaling improvements, the same lesson as single-thread Hilbert-curve graphs beating 128-machine clusters.

## Going deeper

1. **The AGM bound and WCOJ origins**: Ngo, Porat, Ré, Rudra, *"Skew Strikes Back: New Developments in the Theory of Join Algorithms"* (PODS 2012, arXiv:1310.3314) — derives the AGM bound and proves WCOJ achieves it.
2. **Leapfrog Triejoin**: Veldhuizen, *"Leapfrog Triejoin: A Simple, Worst-Case Optimal Join Algorithm"* (ICDT 2014, arXiv:1210.0481) — a simpler WCOJ implementation using sorted arrays and a leap/seek API; the algorithm used in production at RelationalAI.
3. **DuckDB's Adaptive Factorization**: Groß, ten Wolde, Boncz, *"Adaptive Factorization Using Linear-Chained Hash Tables"* (CIDR 2025) — shows how to integrate WCOJ-style intersections into DuckDB's hash join executor adaptively at runtime, gaining 16× on cyclic graph queries with minimal regression on TPC-H.

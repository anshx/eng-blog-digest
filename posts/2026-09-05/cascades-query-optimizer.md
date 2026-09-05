---
title: "The Cascades Framework for Query Optimization"
source: http://sites.computer.org/debull/95SEP-CD.pdf
author: Goetz Graefe
company: Portland State University (implemented at Tandem NonStop SQL and Microsoft SQL Server)
date_posted: 1995-09-01
date_digested: 2026-09-05
---

# The Cascades Framework for Query Optimization

## What's new to learn

1. **The Memo data structure**: A compact directed hypergraph where each node is an *equivalence group* — all logically identical sub-expressions — so that two plans sharing a common sub-expression share the *same child group* rather than copying it. The Memo lets a database optimizer represent millions of plan alternatives using only thousands of nodes, the same trick that makes chess engine transposition tables work.

2. **Two-tier rule system**: Cascades separates *transformation rules* (logical-to-logical rewrites that preserve equivalence, e.g. join commutativity: `A⋈B ≡ B⋈A`) from *implementation rules* (logical-to-physical mappings that choose how to execute, e.g. "implement this logical Join as a HashJoin"). This separation means adding a new algebraic identity or a new execution algorithm is adding exactly one rule, with no changes to the optimizer's search machinery.

3. **Required physical properties with branch-and-bound pruning**: Parent operators *request* output characteristics (sorted by column X, hash-partitioned by column Y) from their child groups. This creates a constraint that propagates down the plan tree, and combined with an upper-bound cost carried through every recursive call, the optimizer prunes provably suboptimal branches before fully evaluating them — an exact parallel to alpha-beta pruning in game trees.

## Prerequisites

- What a relational query plan is: a tree of physical operators (Hash Join, Sort-Merge Join, Index Scan, Filter, Sort) connected by dataflow edges; a "plan" maps SQL to a specific execution strategy
- Dynamic programming as a technique: solve sub-problems once, memoize results, combine bottom-up
- Why "equivalent" plans can have wildly different costs: `SELECT * FROM A JOIN B JOIN C` can join in 6 different orders; the same logical result, but the sizes of intermediate results can differ by many orders of magnitude

## The core idea

Every SQL query has exponentially many equivalent physical execution plans. A query joining five tables already has 120 possible join orderings, each of which can use hash join or sort-merge join or nested-loop join, with indexes or without. The optimizer must find the cheapest plan without explicitly trying them all.

System R (IBM, 1979) showed the baseline: bottom-up dynamic programming over join subsets. For each subset of the joined tables, record the cheapest plan to compute that subset; compose them upward. This works great for join ordering — it's the algorithm most OLTP databases still use at core — but it's hard-coded to one class of transformation. Adding a new algebraic rewrite (like pushing a correlated subquery down, or matching a materialized view) requires structural surgery on the optimizer.

Graefe's insight was that the *optimizer infrastructure* — search order, memoization, cost propagation — should be entirely separate from the *mathematical content of the search* (which expressions are equivalent, which physical operators implement which logical ones). If you draw that boundary correctly, adding new optimizations is just registering new rules, and the search engine handles the rest.

The resulting architecture, Cascades, has three components: a **Memo** that stores explored plan alternatives compactly, a **rule set** that defines what transformations and implementations are legal, and a **task scheduler** that drives a top-down branch-and-bound search. Query optimizers in Tandem NonStop SQL, Microsoft SQL Server 7.0 onward, CockroachDB, Apache Spark Catalyst, Apache Calcite (used by Flink, Druid, Hive), and Greenplum Orca all descend directly from this paper.

## Mechanics

### The Memo

The Memo is a DAG. Every node is a **group** representing one logical expression — one "way to compute some sub-result." A group contains multiple **member expressions**, each of which can be:
- **Logical**: an abstract relational algebra node (`LogicalJoin(Group1, Group2)`)
- **Physical**: a concrete execution operator (`HashJoin(Group1, Group2, cost=450)`)

The key structural point is that a member expression's *children are groups, not expressions*. Two physical plans that share a common sub-expression point to the *same child group*. This is exactly how chess engine transposition tables work: rather than re-exploring a board position every time it's reached by a different sequence of moves, you store the position's evaluation once and look it up by hash.

For a query `SELECT * FROM A JOIN B JOIN C`:
```
Group 0 (root: full result)
  LogicalJoin(Group1, Group2)   ← A⋈B first, then ⋈C
  LogicalJoin(Group3, Group4)   ← A first, then ⋈(B⋈C)
  HashJoin(Group1, Group2, cost=1200)
  SortMergeJoin(Group1, Group2, cost=950)

Group 1 (A⋈B, sorted)
  LogicalJoin(GroupA, GroupB)
  SortMergeJoin(GroupA, GroupB, cost=300)
  HashJoin(GroupA, GroupB, cost=420)

Group 2 (C, filtered)
  LogicalFilter(GroupC, pred)
  IndexScan(C, pred, cost=80)
  SeqScan+Filter(C, pred, cost=200)
```

Groups 1 and 2 appear as inputs to multiple expressions in Group 0. Their costs are computed once.

### The five task types

The optimizer runs as a priority queue of tasks:

1. **OptimizeGroup(g, required\_props, upper\_bound)**: The entry point. "Find the cheapest physical plan in group g that produces output with properties `required_props`, with cost under `upper_bound`." Schedules OptimizeExpression for every logical expression currently in g.

2. **OptimizeExpression(e, required\_props, upper\_bound)**: "Try to realize logical expression e as a physical plan satisfying `required_props`." Applies all applicable *implementation rules* to e (generating physical expressions) and all applicable *transformation rules* to e (generating new logical expressions, added to the same group). Each new physical expression generates an OptimizeInputs task.

3. **ExploreGroup(g, rule\_set)**: "Ensure group g has been fully expanded under `rule_set`." Schedules ExploreExpression for each logical expression in g.

4. **ExploreExpression(e, rule)**: "Apply transformation rule `rule` to logical expression e." If the rule fires, the resulting logical expression is inserted into the same group as e (it's equivalent). A bit map on each expression tracks which rules have already been applied, preventing infinite re-application.

5. **OptimizeInputs(physical\_expr, required\_props, upper\_bound)**: "Cost this physical expression by recursively optimizing its child groups." For each child group, determine the *required physical properties* that child must produce (e.g. HashJoin requires no specific input order; SortMergeJoin requires both inputs sorted on the join key). Schedule OptimizeGroup for each child with those requirements and a tightened upper bound. If partial cost (operator cost + cheapest-so-far child costs) already exceeds `upper_bound`, **prune**: abandon this branch entirely.

### Required physical properties and enforcers

A physical property is a characteristic of an operator's output: sort order, distribution/partitioning, grouping. Parent operators *declare* what properties they need from each input; the optimizer propagates these down.

If the cheapest physical plan for a child group does not naturally produce the required property, the optimizer *inserts an enforcer*: an explicit operator that converts. A Sort enforcer produces a sorted output by calling an external sort. A Redistribute enforcer repartitions data across nodes for parallel execution.

The optimizer considers both alternatives — "find a plan that naturally produces the property" and "find the cheapest plan and enforce the property on top" — and picks whichever is cheaper. This means sort-merge join and hash join compete fairly: if sort-merge join's co-requirement for sorted inputs is cheaper to achieve (because an index already provides sorted order), it wins; if not, hash join wins.

### Transformation rule examples

**Logical-to-logical (transformation rules)**:
- Join commutativity: `Join(A, B) → Join(B, A)`
- Join associativity: `Join(Join(A, B), C) → Join(A, Join(B, C))`
- Predicate pushdown: `Filter(Join(A, B), pred_on_A) → Join(Filter(A, pred_on_A), B)`
- Subquery unnesting: `Select(exists(correlated_subquery)) → Semi-Join(...)`

**Logical-to-physical (implementation rules)**:
- `Join(A, B) → HashJoin(A, B)`
- `Join(A, B) → SortMergeJoin(A, B)`
- `Join(A, B, pred) → IndexNestedLoopJoin(A, B, pred)` — only applicable if B has an index on the join key
- `Scan(Table) → SeqScan(Table)`
- `Scan(Table, pred) → IndexScan(Table, pred)` — only applicable if an index exists

The implementor extends the system by *adding rules*. Adding a new join algorithm is one implementation rule. Adding support for a new algebraic identity is one transformation rule.

### The branch-and-bound loop in detail

Every OptimizeGroup call passes a cost upper bound initialized to the caller's current best (or infinity on the first call). As OptimizeInputs processes child groups left-to-right:

```
partial_cost = cost_of_operator_itself
for each child group in left-to-right order:
    child_cost = OptimizeGroup(child, child_required_props, upper_bound - partial_cost)
    partial_cost += child_cost
    if partial_cost >= upper_bound:
        prune: abort this physical expression
return partial_cost
```

On each return from OptimizeGroup, if the returned plan is cheaper than the current group winner, the bound tightens. This is exactly alpha-beta pruning: the current-best of the parent acts as the beta cutoff.

CockroachDB's optimizer (which is Cascades-based) reports that in practice, the optimizer explores only a small fraction of the mathematical search space because branch-and-bound prunes aggressively once a good plan is found early.

## Where it breaks

**Cardinality estimation errors invalidate the pruning.** The entire branch-and-bound is correct only if cost estimates are accurate. But cost depends on cardinality (how many rows does each operator produce?), and cardinality estimation — especially for multi-join queries with correlated predicates — is notoriously unreliable. When estimates are off by 100×, the upper-bound pruning confidently eliminates the optimal plan and keeps a catastrophic one. Cascades defines *how to search*; it says nothing about *how to estimate*. Bad statistics are the primary source of bad query plans in production.

**Rule ordering effects.** Because transformations are applied opportunistically, the order in which rules fire affects which expressions get explored first. If rule A opens up new expressions that rule B could transform, but the system evaluates rule B's applicability before rule A fires, rule B gets a bit set saying "already applied" and the combination A→B is never explored. Designers must reason about rule interaction to ensure completeness.

**Rule explosion.** Extensibility is a double-edged sword. CockroachDB uses 200+ rules; SQL Server uses hundreds more. Large rule sets are hard to debug and can create exponential combinatorial explosions in certain query shapes (e.g. star schemas with many dimension tables). Some production systems cap the number of transformation phases or time-limit optimization.

**Optimization overhead for short queries.** For a lookup-by-primary-key query, the "optimal" plan is obvious, but a full Cascades search still allocates Memo groups, dispatches tasks, and applies rules. Systems like MySQL and PostgreSQL often do a fast "trivial plan" check first and invoke the full optimizer only when there is real complexity to explore.

## Why it works

The deeper principle: **Cascades is dynamic programming on a DAG of algebraic equivalences, with branch-and-bound pruning over cost, and the Memo as a transposition table.**

Each of these three ingredients is a well-known algorithmic idea:

| Ingredient | General technique | Cascades analog |
|---|---|---|
| Compact equivalence representation | Union-Find / congruence closure | Memo groups |
| Avoiding redundant sub-problem computation | Dynamic programming (memoization) | Memo's shared child groups |
| Pruning provably suboptimal branches | Alpha-beta / branch-and-bound | Upper-bound propagation in OptimizeInputs |
| Avoiding revisiting equivalent states | Transposition table (chess) | Bit maps on expressions tracking applied rules |

The alpha-beta / branch-and-bound connection is the sharpest: in chess, once you know the best move from the current position scores α, you can prune any opponent move that leads to a position scoring ≥ β (their current best), because they'll never allow you to reach it. In Cascades, once you know the best plan for a group costs X, you can prune any alternative plan whose partial cost (with children unresolved) already exceeds X — because fully evaluating the children can only increase the cost.

The critical insight that makes this scale is *sharing*. If two query plans share a sub-expression — say, both `(A⋈B)⋈C` and `A⋈(B⋈C)` need to compute `B⋈C` (just in different contexts) — the Memo represents `B⋈C` as a single group, computed once, referenced multiple times. Without sharing, the search space is exponential in the number of tables. With sharing, it's polynomial for most practical queries.

This explains why every major analytic query engine uses a Cascades-descended optimizer: the rule-extensibility means new algebraic identities (lateral join rewrites, view-matching, common sub-expression elimination, partition pruning) can be added independently without changing the search infrastructure. The optimizer framework is a fixed cost; the return on investment comes from accumulating rules over years.

The connection to CS fundamentals runs deep: Cascades is a special case of **AND-OR graph search** (from AI planning) applied to relational algebra. Logical expressions are OR nodes (multiple equivalent implementations). Physical plans are AND nodes (must evaluate all children). Cost propagates bottom-up; pruning propagates top-down. The Memo is the closed list. This pattern appears everywhere: Boolean satisfiability solvers (DPLL with clause learning), constraint propagation (AC-3), probabilistic graphical model inference (sum-product on factor graphs). The domain changes; the structure is the same.

## Going deeper

1. **"The Cascades Framework for Query Optimization" – Goetz Graefe (IEEE Data Engineering Bulletin, September 1995)**: The original paper; 11 dense pages. Read sections 2 (Memo) and 3 (tasks) first; section 4 (properties) rewards a second read after you understand the search loop. The paper also describes Graefe's earlier Volcano framework (1994) and explains exactly what Cascades changed.

2. **"An Overview of Query Optimization in Relational Systems" – Surajit Chaudhuri (PODS 1998, accessible from Microsoft Research)**: A tutorial paper that situates Cascades alongside System R's bottom-up DP and explains cardinality estimation and statistics. The best single paper to read if you want to understand the full query optimization pipeline, not just the search framework.

3. **"Inside CockroachDB's SQL Layer: Query Optimization"** (engineering blog, CockroachDB): A production account of implementing a Cascades-style optimizer from scratch in Go, including rule representation, the Optgen DSL for authoring rules, and the decisions made when Cascades theory met real SQL complexity. CockroachDB's optimizer is open-source (github.com/cockroachdb/cockroach/pkg/sql/opt) and is one of the cleanest modern implementations to read alongside the original paper.

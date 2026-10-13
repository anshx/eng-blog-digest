---
title: "egg: Fast and Extensible Equality Saturation"
source: https://arxiv.org/abs/2004.03082
author: Max Willsey, Chandrakana Nandi, Yisu Remy Wang, Oliver Flatt, Zachary Tatlock, Pavel Panchekha
company: University of Washington
date_posted: 2021-01-01
date_digested: 2026-10-13
---

# egg: Fast and Extensible Equality Saturation

## What's new to learn

1. **E-graph**: a data structure that simultaneously represents a large (often exponential) set of mutually equivalent programs as a compact directed acyclic graph of *e-classes* (equivalence classes of terms), where adding an equivalence never deletes existing structure — it only grows the graph.

2. **Equality saturation**: an optimization strategy that applies every rewrite rule to every e-class simultaneously, iterating to a fixed point, so that by the time you extract the "best" program the entire reachable equivalence closure has been explored — no phase ordering, no irrevocable choices.

3. **E-class analysis**: a lattice-valued annotation on each e-class (e.g., "minimum cost", "constant value", "sign") that is maintained incrementally as new equivalences are merged, letting the optimizer reason about all programs in a class without ever enumerating them.

## Prerequisites

- **Term rewriting / ASTs**: the idea that programs are trees and a rewrite rule replaces one subtree with another (e.g., `(* x 2) → (<< x 1)`).
- **Union-Find (Disjoint Set Union)**: the classic data structure for tracking which elements belong to the same equivalence class in nearly O(1) amortized time.
- **Lattices**: a partial order where any two elements have a unique join (least upper bound). Used in dataflow analysis and abstract interpretation.
- The archive's [Cascades query optimizer post](../2026-09-05/cascades-query-optimizer.md) gives helpful context but is not required.

## The core idea

Every optimizing compiler or query planner faces the **phase ordering problem**: rewrite A before B and you may end up unable to apply C; rewrite B first and A becomes unreachable. The root cause is that rewrites are *destructive* — applying `x * 2 → x << 1` replaces the multiplication and permanently discards it, even if a later rule needed that multiplication as a sub-pattern.

E-graphs solve this by making rewrites *additive*. Instead of replacing `(* x 2)` with `(<< x 1)`, an e-graph says "these two terms are now equivalent" and records both. Every term lives in an *e-class* (a set of mutually equivalent terms). A rewrite rule doesn't replace; it *merges* the target e-class with a new e-class for the rewritten form.

After running all rewrite rules to saturation (until no new merges occur), the e-graph compactly encodes every program the rules can derive from the input. Extraction then picks the cheapest representative from each e-class to produce the optimized output.

The mental model at the whiteboard: imagine every program you could derive as a node in a graph, and edges as rewrite rule applications. Equality saturation is BFS over this graph, but with deduplication — nodes that are equivalent get merged, keeping the graph tractable. Extraction is Dijkstra's algorithm over the merged graph to find the cheapest path.

## Mechanics

### E-nodes and E-classes

An **e-node** represents one application of a function (operator) whose *children are e-classes, not terms*. For example, `(* e1 e2)` is an e-node where `e1` and `e2` are references to e-classes, not to concrete subterms. This indirection is what enables sharing: if two e-nodes happen to reference the same child e-class, they automatically benefit from any future merges of that class.

An **e-class** is just a set of e-nodes that are currently known to be equivalent, maintained via union-find.

### Adding a term

When you add a term `(* x 2)` to an empty e-graph:
1. Create an e-class for the leaf `x` containing the e-node `x`.
2. Create an e-class for the leaf `2` containing the e-node `2`.
3. Create an e-class containing the e-node `(* [class_x] [class_2])`.

The result is three e-classes, each with one e-node.

### Applying a rewrite rule

Rule: `(* ?x 2) → (<< ?x 1)`. Pattern match finds that e-class 3 contains an e-node matching the LHS. The rewriter:
1. Constructs the RHS `(<< [class_x] [class_1])` where `class_1` is a new e-class for the constant 1.
2. Calls `union(class_3, class_new_shift)` — merging the two e-classes.

Now e-class 3 contains **both** `(* x 2)` and `(<< x 1)`. Neither was deleted.

### Rebuilding: egg's key contribution

Naively, every `union()` call must immediately propagate upward through all parent e-nodes that reference the merged class — updating them to point to the canonical representative. This is called *congruence closure maintenance*, and doing it eagerly is expensive.

egg's innovation is **deferred rebuilding**: batch all pending unions and restore the congruence invariant once per iteration of the saturation loop, rather than after every individual union. This amortizes the overhead and is the primary reason egg is 10–100× faster than prior e-graph implementations on real-world workloads.

### E-class analysis

Each e-class carries a user-defined *analysis value* from a user-defined lattice. When two classes merge, their values are joined via the lattice's join operator. When a new e-node is added to a class, the class's value is updated via a user-supplied `make` function over the e-node's children's values.

Concrete example — constant folding: the analysis tracks "if all terms in this class have the same constant value, what is it?" The lattice is `Option<Constant>` with join `(Some(a), Some(b)) → Some(a) if a == b, else None`. When e-class 3 contains `(* 4 2)` and no variables, the analysis sets its value to `Some(8)`. Extraction can then replace the entire class with the constant `8` without ever searching for arithmetic simplifications at extraction time.

### Extraction

Once saturation terminates, the e-graph contains a forest of e-classes. The extractor walks bottom-up, choosing at each e-class the e-node that minimizes the cost function (e.g., AST node count, or estimated CPU cycle count). The e-node's children are themselves e-classes, so the extraction recurses. This is a greedy tree DP: optimal locally but may miss globally optimal solutions when cost is non-decomposable. (For decomposable cost functions it is globally optimal.)

### Saturation termination

The main loop:
```
loop:
  for each rewrite rule r:
    for each e-class c:
      if r's LHS matches some e-node in c:
        construct RHS, call union(c, rhs_class)
  rebuild()
  if no new merges occurred: break
```
The loop terminates when the e-graph reaches a fixed point — no new equivalences can be added. This is guaranteed when the rewrite rule set is finite and the e-graph itself is finite (bounded term depth). For rule sets with infinite derivations, saturation may not terminate; practitioners set an iteration limit or e-graph size bound.

## Where it breaks

**Exponential growth**: even with sharing, rule sets that interact aggressively can cause the e-graph to grow exponentially in the number of terms. Empirically, this rarely happens on practical programs, but it is a real risk for rule sets with many variable-capture rules.

**Non-termination**: some rule systems have cycles that never saturate (e.g., commutativity + associativity produce infinite equivalent forms). egg handles this with iteration limits and size budgets.

**Extraction complexity**: if the cost function is not locally decomposable (e.g., it depends on how many times a shared sub-expression is computed), greedy extraction is suboptimal. Extracting the globally optimal program is NP-hard in general; practitioners use ILP or heuristics.

**Side effects and sequential semantics**: equality saturation assumes rewrites are semantics-preserving regardless of order. In the presence of effectful operations (reads, writes, exceptions), this is harder to guarantee without adding guards to rules.

**Scalability to very large programs**: E-graph-based optimization is most effective on small to medium expressions (up to ~10^5 nodes). For whole-program optimization, the e-graph is typically applied one function or one basic block at a time.

## Why it works

### The Cascades Memo is an E-graph in disguise

The archive's [Cascades post](../2026-09-05/cascades-query-optimizer.md) describes how modern SQL query optimizers explore equivalent query plans by memoizing expressions in a *Memo structure*: each equivalence group holds all known alternative implementations of a logical sub-plan, and transformation rules add alternatives to groups. This is precisely an e-graph, with groups = e-classes and transformation rules = rewrite rules. Cascades applies rules lazily, top-down, with branch-and-bound pruning. Equality saturation applies rules eagerly, bottom-up, with deferred extraction. The correctness argument is the same; the trade-off is completeness vs. cost.

### Equality saturation ≅ abstract interpretation at the program level

Abstract interpretation (the foundation of static analysis tools like Facebook's Infer and Microsoft's CodeQL) computes a fixed point of a monotone function over a lattice of abstract values. Equality saturation computes a fixed point of a monotone function (adding equivalences) over the lattice of e-graphs (ordered by subsumption). Both frameworks use:
- A lattice to represent incomplete knowledge
- Monotone functions to propagate information
- Fixed-point iteration as the central algorithm

The difference: abstract interpretation compresses programs into abstract values (losing precision); equality saturation compresses programs into equivalence classes (lossless for the queries it encodes).

### The phase ordering problem is a consequence of greediness

Destructive rewriting is a greedy algorithm: it commits to a rewrite before seeing whether future rewrites would have benefited from the original form. Greedy algorithms fail when local optima diverge from global optima. E-graphs avoid greediness entirely by deferring commitment until the entire equivalence closure is known. This is the same insight behind A* (don't commit to a path until you have the full heuristic estimate) and MVCC (don't commit a transaction until the full write set is visible).

## Going deeper

1. **egglog (2023)**: An extension of egg that adds Datalog-style relational reasoning to e-graphs, unifying equality saturation with database query evaluation. Paper: https://arxiv.org/abs/2304.04332. Used in Cranelift's mid-end optimizer for WebAssembly.

2. **TASO (NeurIPS 2019)**: E-graph-based optimization of ML computation graphs (substituting one sequence of tensor operations for an equivalent, cheaper one). Demonstrates equality saturation on operator fusion and layout transformation for deep learning. Paper: https://cs.stanford.edu/~padon/taso-sosp19.pdf

3. **Cranelift's use of e-graphs in production** (2023): The WebAssembly compiler backend used in Wasmtime and SpiderMonkey replaced its peephole optimizer with an e-graph-based mid-end, reducing code size and improving compile-time optimization quality at the same time. Blog: https://cfallin.org/blog/2023/01/20/cranelift-egraphs/

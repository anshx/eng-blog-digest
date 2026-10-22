---
title: "An Extensible SAT-Solver"
source: https://minisat.se/downloads/MiniSat.pdf
author: Niklas Eén, Niklas Sörensson
company: Chalmers University of Technology
date_posted: 2004
date_digested: 2026-10-22
---

# An Extensible SAT-Solver

## What's new to learn

1. **Unit propagation is BFS over an implication graph** — when a clause has only one unset literal, that literal is forced; exhausting all such forces is forward chaining over an AND/OR constraint network, topologically equivalent to BFS.

2. **Conflict clause learning is lazy resolution proof derivation** — when search reaches a contradiction, the "learned clause" is a new clause derivable by resolution from the original problem, compactly encoding why that partial assignment was infeasible, forever pruning that region of the search space.

3. **Non-chronological backtracking via First UIPs** — instead of undoing decisions one at a time, the solver identifies the single node through which all causal paths to the conflict flow (the First Unique Implication Point) and jumps directly past all decisions that are causally irrelevant to the conflict.

## Prerequisites

- Boolean logic: literals (variables and their negations), clauses (disjunctions of literals), CNF (a conjunction of clauses)
- Backtracking DFS / recursion
- Basic graph theory: directed graphs, path, dominator
- Familiarity with DPLL is helpful but not required

## The core idea

The Boolean Satisfiability (SAT) problem asks: given a formula in conjunctive normal form (CNF) — a big AND of OR-clauses — is there a variable assignment that makes every clause true? SAT is NP-complete in theory, yet modern CDCL solvers handle industrial instances with millions of variables in seconds.

The 1962 DPLL algorithm is the baseline: make a decision (guess a variable's value), propagate forced consequences, and if you hit a contradiction, backtrack and try the opposite. It's correct but naive — it backtracks chronologically and learns *nothing* from conflicts, so it can repeat the same mistake across different branches.

**CDCL** (Conflict-Driven Clause Learning) adds two interlocking ideas:

1. **When you hit a contradiction, don't just undo — learn.** Trace the implication graph backward from the conflict to derive a new clause that certifies the infeasibility of that partial assignment. Add it to the clause database permanently.

2. **Use the learned clause to jump, not crawl.** The learned clause encodes exactly how far back to backtrack (the "asserting level") and immediately forces a propagation when you get there. A single conflict can cause the solver to jump back five decision levels, then immediately propagate a chain of 50 consequences — collapsing what would have been exponential re-exploration into a single leap.

The mental model: **CDCL is a backtracking search that turns every dead-end into a compact reusable lemma, converting exponential worst-case behavior into efficient proof search on structured instances.**

## Mechanics

### Data structures

CDCL maintains:
- **Trail**: an ordered list of (literal, decision-level, reason-clause) tuples — the current partial assignment in assignment order.
- **Implication graph**: a DAG where each implied literal points to the clause that forced it; decision literals have no parents.
- **Clause database**: the original clauses plus all learned clauses.
- **Two-watched-literal scheme (MiniSat's key implementation insight)**: each clause watches exactly two of its literals. When one watch becomes False, check if the clause is now a unit clause. This makes unit propagation O(1) per literal flip on average rather than O(clause-length), a ~5× speedup.

### The CDCL loop

```
level = 0
while true:
  conflict = unit_propagate()
  if conflict:
    if level == 0: return UNSAT  // no decisions to undo, problem is unsatisfiable
    learned_clause, asserting_level = analyze_conflict(conflict)
    add_to_database(learned_clause)
    backjump(asserting_level)
    propagate(forced_literal_from_learned_clause)
  else:
    if all_variables_assigned(): return SAT
    if should_restart(): restart()
    decide(pick_variable_by_VSIDS())
    level += 1
```

### Conflict analysis: deriving the learned clause

When a conflict is found (a clause with all literals False), the implication graph contains a subgraph showing how each decision led to the contradiction. The solver performs **resolution** — repeatedly combining pairs of clauses that contain complementary literals — tracing backward from the conflict node toward the current decision level.

The result is the **First UIP** (First Unique Implication Point): the single literal at the current decision level that lies on every path from the current decision to the conflict. It's the "dominator" of the conflict in graph terms.

The learned clause is: `(¬First_UIP) ∨ (all literals from previous decision levels that imply the conflict)`.

Critically, this clause is always an *asserting clause*: at the asserting level (the level of the second-highest literal in the learned clause), all but one literal are False, so `¬First_UIP` is immediately forced — triggering a chain of new propagations the moment the solver arrives.

### Non-chronological backtracking

With chronological backtracking, the solver undoes decisions one at a time until it reaches a decision it can flip. With CDCL, it undoes everything above the asserting level in a single jump — potentially many decision levels at once. The learned clause immediately becomes a unit clause at the asserting level, forcing a new assignment and possibly cascading new implications.

### VSIDS: adaptive decision ordering

**Variable State Independent Decaying Sum (VSIDS)** assigns each variable an activity score. Every variable that appears in a learned conflict clause gets its score bumped. Periodically, all scores are multiplied by a decay factor (e.g., 0.95). The solver always picks the highest-activity unassigned variable for its next decision.

The effect: the solver focuses on variables that have recently been involved in conflicts — the "hot" part of the constraint network. This is an online bandit-style adaptive heuristic that concentrates search effort where the structure is tightest.

### Restarts and clause deletion

Modern CDCL adds two more ingredients:

- **Restarts**: periodically clear all decisions and the trail (but keep all learned clauses). This prevents the solver from getting stuck in an unproductive region, while preserving all accumulated lemmas.
- **Clause deletion**: learned clauses consume memory. MiniSat deletes half the learned clauses when the database grows too large, keeping those with the lowest **Literal Block Distance (LBD)** — a measure of how many distinct decision levels the clause's literals span. Low LBD = clause's literals are tightly coupled = more likely to be useful again.

### Scale numbers

MiniSat 2.0 (2006) is ~600 lines of C++. The SAT 2003 competition featured instances with ~1 million variables and ~4 million clauses; CDCL solvers solved them in seconds. Modern industrial hardware verification problems (e.g., Intel chip equivalence checking) have hundreds of millions of variables; specialized CDCL solvers are the only known practical tools.

## Where it breaks

1. **Random k-SAT near the phase transition** (clause/variable ratio ≈ 4.27 for 3-SAT): these instances lack short resolution proofs, so learned clauses don't help and the solver degrades to near-exponential behavior. CDCL's practical power comes entirely from problem structure.

2. **Space blowup from learned clauses**: pathological instances cause the learned clause database to grow explosively. Clause deletion heuristics are empirical; no provably optimal strategy is known.

3. **Counting and optimization**: CDCL naturally solves *decision* SAT (satisfiable or not). For **#SAT** (count satisfying assignments), **MaxSAT** (find the best-weight assignment), or **QSAT** (quantified Boolean formulas), different or extended algorithms are needed, though CDCL is often a subroutine.

4. **Parallelism**: SAT is awkward to parallelize — the search tree has data-dependent structure, and shared learned clauses cause synchronization overhead. Portfolio approaches (run many independent solver configurations, share some clauses) outperform scaling a single CDCL instance, but the algorithm itself is fundamentally sequential.

## Why it works

The core insight is that **CDCL is an incremental proof search engine**. By Robinson's completeness theorem, any unsatisfiable CNF formula has a *resolution refutation* — a tree of resolution steps that derives the empty clause. CDCL discovers this refutation piecemeal by chasing implications: each conflict yields one resolution step, each learned clause is a lemma.

Industrial SAT instances are hard in theory but easy in practice because they're **highly structured**: chip designs, planning problems, and verification conditions have many implied constraints, so decisions propagate deeply and resolution proofs are short. The phase transition shows the other side: random instances near the critical ratio have *long* minimum resolution proofs, and CDCL is correctly hard on them.

**The deeper principle**: CDCL is the same "speculate-verify-learn" pattern that appears throughout systems:
- **Optimistic concurrency control** (Percolator, Postgres SSI): speculatively execute, detect conflicts at commit time, abort and retry — with each abort teaching the system what to avoid.
- **Branch prediction and speculative execution**: commit on prediction-correct, flush on misprediction — and update branch history. The CPU's branch predictor table is its "learned clause database."
- **Alpha-beta pruning in game trees**: a discovered lower bound prunes all branches that can't exceed it — a conflict clause that permanently removes a subtree.
- **BDD-based model checking**: BDDs represent *sets* of states symbolically; CDCL's learned clauses are a clause-based version of the same idea — symbolically eliminating large state-space regions.

The unifying insight: **whenever a search problem's infeasibility can be certified by a short proof, turning each dead end into a reusable certificate converts exponential backtracking into polynomial proof search.**

## Going deeper

1. **The GRASP paper** — Marques-Silva & Sakallah, "GRASP: A Search Algorithm for Propositional Satisfiability" (IEEE Trans. Computers, 1999). The first published formalization of conflict-based search and the ancestor of all modern CDCL solvers: https://dl.acm.org/doi/10.1109/12.769433

2. **Glucose and LBD** — Audemard & Simon, "Predicting Learnt Clauses Quality in Modern SAT Solvers" (IJCAI 2009). Explains why LBD (Literal Block Distance) reliably predicts clause reuse and why aggressive deletion with low-LBD retention dominates. Glucose extended CDCL to win multiple SAT competitions: https://www.labri.fr/perso/lsimon/research/glucose/

3. **DPLL(T) and SMT solving** — Ganzinger et al., "DPLL(T): Fast Decision Procedures" (CAV 2004). Shows how to lift CDCL to *theories* (linear arithmetic, arrays, bit-vectors) by tightly coupling the CDCL core with specialized theory solvers — the architecture of Z3, Yices, and CVC5: https://dl.acm.org/doi/10.1007/978-3-540-25863-9_6

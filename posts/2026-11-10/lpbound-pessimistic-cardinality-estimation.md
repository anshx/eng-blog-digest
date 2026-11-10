---
title: "LpBound: Pessimistic Cardinality Estimation Using ℓp-Norms of Degree Sequences"
source: https://arxiv.org/abs/2502.05912
author: Haozhe Zhang, Christoph Mayer, Mahmoud Abo Khamis, Dan Olteanu, Dan Suciu
company: University of Washington / University of Oxford
date_posted: 2025-02-10
date_digested: 2026-11-10
---

# LpBound: Pessimistic Cardinality Estimation Using ℓp-Norms of Degree Sequences

## What's new to learn

1. **Degree sequence ℓp-norms as statistics**: For attribute B in relation R, the degree sequence is the histogram deg_R(B=b) = "how many rows have B=b." The ℓp-norm of that histogram (for varying p from 1 to ∞) captures different aspects of skew — p=1 is total row count, p=2 is a second-moment measure, p=∞ is the maximum fanout. Using *all* of these p-norms as statistics, not just counts, is the new idea.

2. **Pessimistic (guaranteed upper-bound) estimation**: Instead of guessing the most likely join size (which query optimizers currently do, and get wrong on correlated or skewed data), LpBound guarantees the true join output is at most its estimate. An optimizer that trusts an upper bound picks a plan that is correct even in the worst case.

3. **Cardinality estimation as a linear program**: For any multijoin query, the optimal (tightest) ℓp-norm bound can be found by solving a small LP over entropy variables. The LP constraints are Shannon inequalities — the same information-theoretic constraints that govern all entropy functions — and each degree-sequence statistic adds one constraint.

## Prerequisites

- SQL joins: what a natural join is and why estimating its output size matters for the query optimizer
- Cardinality estimation basics: histograms, independence assumption, selectivity
- The Cascades query optimizer (see the 2026-09-05 digest entry) — it needs cardinality estimates to cost alternative plans
- The AGM bound / worst-case optimal join (see the 2026-10-15 Free Join entry) — LpBound generalizes it
- Hölder's inequality at a high school algebra level: for sequences x_b and y_b, Σ_b x_b·y_b ≤ (Σ_b x_b^p)^(1/p) · (Σ_b y_b^q)^(1/q) when 1/p + 1/q = 1

## The core idea

**The fundamental problem with traditional cardinality estimation** is the independence assumption. If you estimate |R(A,B) ⋈ S(B,C)| using only |R|, |S|, and the fraction of distinct B values in each relation, you implicitly assume that which values appear in R on attribute B is statistically independent of which values appear in S on attribute B. In practice this is false: correlated attributes and heavy-tailed distributions routinely cause 1,000× underestimates. The optimizer then picks a nested-loop join assuming the result is 100 rows, when it is actually 100,000 rows.

**LpBound's approach**: don't estimate the most likely size; estimate the maximum possible size consistent with the observed statistics. That bound is always correct. If the bound says "at most 50,000 rows," then the optimizer can choose a plan that handles 50,000 rows without being surprised.

**The key statistic is the ℓp-norm of the degree sequence.** For R(A, B), define:
- deg_R(b) = number of tuples in R where B = b (this is the degree of value b)
- The ℓp-norm: ||deg_R(B)||_p = ( Σ_b deg_R(b)^p )^(1/p)

This family of norms captures the "shape" of the B-fanout distribution:
- p=1: Σ deg_R(b) = |R|. Just the total count.
- p=2: √(Σ deg_R(b)²). Sensitive to variance; large if a few B values have many rows.
- p=∞: max_b deg_R(b). The worst-case fanout.

For a two-way join R(A,B) ⋈ S(B,C):

|R ⋈ S| = Σ_b deg_R(b) · deg_S(b)

Apply Hölder's inequality with any exponents p, q satisfying 1/p + 1/q = 1:

|R ⋈ S| ≤ ||deg_R(B)||_p · ||deg_S(B)||_q

This is a **guaranteed upper bound**. The tightest bound comes from choosing p, q to minimize the right-hand side — which is a one-dimensional search (since q = p/(p−1)).

For multi-way joins (R₁ ⋈ R₂ ⋈ … ⋈ Rk) with multiple shared attributes, the bound generalizes via a network of Hölder applications. The optimal exponents for the whole system are found by solving a linear program, where variables are (logs of) exponents and constraints come from Shannon inequalities.

## Mechanics

**Offline pre-computation (index time)**:
For each relation R and each subset of its attributes that might be a join attribute, compute ||deg_R(B)||_p for p ∈ {1, 2, 4, …, ∞} (a discretized set of about 10–20 values). This is a one-pass scan per attribute. These statistics are stored alongside the table and updated incrementally on writes (ℓp-norms of degree histograms update cheaply via the insertion/deletion of a single row's contribution).

**At query optimization time (online, < 2 ms)**:
1. For the given multijoin query Q(X₁,…,Xn) = R₁ ⋈ R₂ ⋈ … ⋈ Rk, extract the join graph: nodes are query variables, hyperedges are the relations.
2. Write the LP:
   - Variables: h(S) for each subset S of query variables (representing a generalized "entropy")
   - Constraints: Shannon inequalities — submodularity (h(A ∪ B) + h(A ∩ B) ≤ h(A) + h(B)) and monotonicity (h(A) ≤ h(B) for A ⊆ B)
   - For each degree-sequence statistic ||deg_Ri(B)||_p, add a constraint: (1/p)·h({B} given the remaining attributes of Ri) ≤ log(||deg_Ri(B)||_p)
   - Objective: minimize h(X₁,…,Xn) (the entropy of the full output, which bounds log|Q|)
3. Solve the LP (small, O(2^n) variables for n join variables — tractable for queries with up to ~20 variables).
4. The optimal LP value e* gives the bound: |Q| ≤ 2^(e*).

**Predicate propagation**: Selection predicates (WHERE clause filters) reduce degree sequences. A predicate "A = 5" on R(A,B) reduces the ℓp-norm of deg_R(B) to only the rows with A = 5. LpBound propagates predicates through the degree-sequence statistics before computing the bound, using prefix degree sequences (precomputed degree statistics conditioned on attribute-value ranges) to handle range predicates efficiently.

**Result**: When injected into Postgres's planner in place of the built-in estimator, LpBound picks query plans that are as good as those derived from the true cardinalities on 95% of benchmark queries. Traditional estimators achieve this on about 60%.

## Where it breaks

**Loose bounds for dense cyclic queries**: For a triangle query R(A,B) ⋈ S(B,C) ⋈ T(A,C) where all three relations have size n and uniform degree distribution, LpBound gives O(n^{3/2}), which is tight (matches the AGM bound). But for even denser data distributions, the ℓp-norm statistics may not carry enough information and the bound can be 2–5× above the true size.

**No correlation across relation boundaries**: LpBound uses per-relation statistics and assumes independence across different relations (beyond what the join structure encodes). If R(A,B) and S(B,C) are correlated in the values they share on B — e.g., both tend to avoid value b=42 — the bound won't reflect that and will be pessimistic.

**Storage overhead**: Storing ℓp-norms for many p values and many attribute subsets can multiply statistics storage by 10–50×. Practical systems will need to be selective about which statistics to maintain.

**Dynamic tables**: The offline ℓp-norm computation is cheap to maintain incrementally for simple inserts/deletes, but bulk reloads and schema changes require a full rescan.

**Not a replacement for sampling**: For highly selective predicates on correlated attributes, fine-grained sampling still gives tighter estimates than ℓp-norm bounds. LpBound is best used as a floor ("the optimizer will not be surprised by a result larger than this") combined with a sampling estimate as the primary guess.

## Why it works

The deeper principle is that **join sizes are inner products of degree sequences**, and inner products are exactly what Hölder's inequality is designed to bound.

For any join along a shared attribute B:
|R ⋈ S| = ⟨deg_R(B), deg_S(B)⟩ = Σ_b deg_R(b) · deg_S(b)

This is a dot product of two vectors. Hölder's inequality says: a dot product is bounded by a product of ℓp-norms. The choice of p, q trades off how much "weight" you put on the R side versus the S side. Choosing p = q = 2 gives the Cauchy-Schwarz bound. The optimal choice depends on which relation is more skewed.

The generalization to multi-way joins works because every Shannon inequality is ultimately a constraint on a sum of entropies, and log(ℓp-norm) is a Rényi entropy (a one-parameter family of entropy measures that coincides with Shannon entropy at p=1 and max-entropy at p=∞). The LP over Shannon inequalities is exactly the LP that finds the optimal "distribution" of Hölder exponents across all pairwise join steps in the network.

**The unifying principle**: the AGM bound (covered in the Free Join digest entry) is the special case where every p=1 — you use only the row counts (ℓ1-norms) as statistics. LpBound says: use the full ℓp-norm family (second moment, fourth moment, max), not just the count. Each additional statistic tightens the bound. The LP finds the optimal way to combine them.

This is the same "higher moments give tighter bounds" principle that underlies:
- **Chebyshev's inequality vs Markov's inequality**: Markov uses E[X] (the first moment, the ℓ1 analog); Chebyshev uses E[X²] (the second moment, the ℓ2 analog). The Chebyshev bound is strictly tighter.
- **Concentration inequalities**: Chernoff bounds use the moment-generating function (all moments) and are tighter than either.
- **Compressed sensing**: the RIP condition is exactly a bound on how much an operator can stretch an ℓ2-norm, and it uses the structure of the measurement matrix rather than just its size.

The connection to the query optimizer: Cascades (see 2026-09-05 digest) uses cardinality estimates to cost alternative plans and pick the cheapest one. A bad estimate causes Cascades to pick the wrong plan. LpBound doesn't eliminate estimation error — but it eliminates the asymmetric failure mode: an upper bound can never cause the optimizer to be surprised by a result that's larger than expected, which is the failure mode that leads to multi-hour queries.

## Going deeper

1. **The AGM bound paper**: Atserias, Grohe, Marx, "Size Bounds and Query Plans for Relational Joins" (FOCS 2008 / SIAM J. Computing 2013) — the theoretical foundation for worst-case join sizes that LpBound builds on.
2. **PODS 2024 predecessor**: Gottesman and Suciu, "Degree Constraints in Cardinality Estimation" — the direct precursor to LpBound that introduced degree sequences as statistics.
3. **Worst-case optimal join execution**: see the 2026-10-15 Free Join digest entry for how knowing the right upper bounds leads to execution strategies that actually achieve them.

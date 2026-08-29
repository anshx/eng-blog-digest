---
title: "Timsort: Python's Sorting Algorithm and the Entropy Bound"
source: https://github.com/python/cpython/blob/main/Objects/listsort.txt
author: Tim Peters (original); Stefan Pochmann, Sebastian Wild, J. Ian Munro (later additions)
company: CPython / Python Software Foundation
date_posted: 2002 (continuously updated)
date_digested: 2026-08-29
---

# Timsort: Python's Sorting Algorithm and the Entropy Bound

## What's new to learn

1. **Entropy-adaptive sorting**: A sort's comparison cost can be bounded by n·H, where H is the Shannon entropy of the input's run-length distribution — meaning the algorithm pays proportionally to *how unsorted* the input actually is, not just its size.

2. **Galloping mode**: An adaptive exponential-then-binary search technique that turns O(n) merge comparison cost into O(log n) when one run dominates, with a self-tuning threshold that eliminates the technique's overhead on random data.

3. **Powersort's merge tree**: Choosing which consecutive sorted runs to merge, in what order, is equivalent to building an *alphabetic binary tree* — the ordering-constrained variant of Huffman coding. Powersort computes a near-optimal solution online, one run at a time, in O(1) per decision.

## Prerequisites

- Merge sort: how merging two sorted arrays works, O(n log n) complexity.
- Binary search (for galloping mode).
- Familiarity with information entropy H = −Σ pᵢ log₂ pᵢ helps for the deeper principle section, but isn't required to understand the mechanics.

## The core idea

Real-world data isn't random. Arrays of database records, network packets sorted by timestamp, log lines, spreadsheet columns — all tend to arrive *almost sorted*, or to contain long runs of already-ordered elements. A general-purpose sorting algorithm that ignores this leaves performance on the table.

Timsort's central bet: **detect existing sorted subsequences ("runs"), extend short ones cheaply, then merge them in the order that minimises total comparisons.** On a fully sorted array it runs in O(n). On a random array it's O(n log n). Between those extremes it degrades smoothly, with cost tracking the information content of the run-length structure rather than n log n.

Powersort (integrated into CPython in 3.11) sharpened the merge-ordering strategy to something provably near-optimal: the total comparison cost is at most n·H + 2n, where H = −Σ(lᵢ/n) log₂(lᵢ/n) is the entropy of the run lengths. If all runs are equal length, H = log₂ k and cost approaches n log₂ k — exactly what you'd pay merging k equal-length arrays. If the input is sorted (one run), H = 0 and cost is O(n).

## Mechanics

### 1. Run detection — O(n)

Scan left to right finding maximal monotone (non-decreasing or strictly decreasing) subsequences. Reverse descending runs in-place using O(n) pointer swaps — they become ascending for free. Each run is recorded on a stack as (start, length).

### 2. Minimum run length (minrun)

Short runs are wasteful: if n = 2048 and every natural run is length 3, you'd start 682 merges. Instead, runs shorter than `minrun` are extended by binary insertion sort.

`minrun` targets making the number of final runs a power of 2, so all merge levels are balanced. For n elements:

```
k = 0
while n >= 64:
    k |= n & 1   # record if any bit was shifted off
    n >>= 1
minrun = n + k   # ≈ 32–64 for any input size
```

This ensures that if the true number of runs is k, after extension each is approximately n/k long, and all merge levels process roughly equal-sized inputs. Recent CPython switched to a per-run scheme (Stefan Pochmann, 2025) that produces perfectly balanced merge trees for any n, not just powers of 2.

Binary insertion sort is used (not linear insertion) because in CPython, one Python object comparison costs ~10–100 ns (pointer-chasing through the type dispatch, `__lt__`, etc.), so saving comparisons dominates even at the cost of quadratic data movement for small runs.

### 3. Galloping mode — the adaptive fast-path

During a merge of runs A and B, the naïve step compares A[i] and B[j] and advances one pointer. But when data has structure, one run often provides many consecutive winners. If A wins 7 times in a row (`min_gallop = 7`), switch into galloping mode:

**Galloping**: exponentially probe B at positions 0, 1, 3, 7, 15, ... until B[2ᵏ] > A[i], then binary search in B[2ᵏ⁻¹ .. 2ᵏ]. This locates the insertion point for A[i] in O(log m) comparisons instead of O(m), where m is the gap to the boundary.

**Adaptive threshold**: `min_gallop` is not fixed. Every time galloping succeeds (the probe reaches ≥ `min_gallop` consecutive elements from one side), decrement `min_gallop` by 1 — making galloping easier to re-enter. Every time it fails to help, increment by 1 — raising the bar. For random data, `min_gallop` drifts to ~50, effectively disabling galloping (no overhead). For nearly-sorted data, it falls to 1–2, maximizing its benefit.

The break-even point: at position i, linear search costs i+1 comparisons; galloping costs 2⌊log₂ i⌋ + 2. Galloping beats linear search at i ≥ 5, which is why the default threshold is 7 (slightly conservative).

### 4. Powersort merge strategy — the stack discipline

The most surprising part of timsort's history: the original merge strategy was **subtly wrong**. Tim Peters' original invariants for the run stack were:

```
len(A[-3]) > len(A[-2]) + len(A[-1])
len(A[-2]) > len(A[-1])
```

These are intended to keep the stack balanced (no run merging with a dramatically larger partner), but researchers trying to write a formal correctness proof in 2015 found that the invariants could be violated on certain inputs (short arrays with specific lengths). Python shipped this for ~13 years across billions of installations without visible bugs — but a proof was impossible.

Powersort replaces the heuristic invariant with a mathematically grounded one. For each consecutive pair of runs, compute their **power**: the depth in a conceptual merge tree at which they would naturally belong together.

**Power computation**: Two adjacent runs start at positions s₁ and s₂ in the array (length n), with lengths n₁ and n₂. Let a = (s₁ + n₁/2) / n and b = (s₂ + n₂/2) / n be their normalized midpoints. The power is the position of the first bit where the binary expansions of a and b diverge:

```
# CPython uses integer bit arithmetic to avoid floating point:
# "a" tracks 2*(s1 + n1//2), "b" tracks 2*(s2 + n2//2)
power = 0
while (a >> power) == (b >> power):  # same bit at position `power`
    power += 1
```

A low power means the two runs "belong to the same subtree near the leaves" — merge them now. A high power means they span a large portion of the array and belong to different subtrees — defer.

The stack invariant is then simple: powers on the stack are strictly decreasing from bottom to top. When a new run arrives, merge all stack entries with power ≥ the new run's power, then push. This greedy rule produces a near-optimal merge tree in a single left-to-right pass.

### 5. Preliminary binary searches

Before merging A and B, timsort uses gallop-like binary searches to trim them: find the first element in B that's less than A[0] (skip the head of B that's already in place), and find the last element in A that's ≤ B[-1] (skip the tail of A). On nearly-sorted data, this can eliminate most of the merge work. On random data it adds two binary searches — cheap at the merge sizes timsort uses.

## Where it breaks

**Truly random data**: no natural runs longer than 1–2 elements. timsort spends all its time building minrun-length runs via binary insertion sort and doing O(n log n) merges with no galloping benefit. Performance is on par with a well-tuned mergesort but not better.

**Many tiny runs**: if n = 10⁶ and every element is a natural run of length 1, the run stack holds thousands of entries. Memory is O(log n) (merge stack), but construction is O(n).

**Galloping threshold is empirically tuned**: `MIN_GALLOP = 7` was chosen by empirical benchmarking on random and nearly-sorted data. The optimal threshold depends on comparison cost relative to cache miss cost, which varies by object type and hardware generation.

**Stability requirement constrains implementation**: timsort must be stable (equal elements retain original order). This rules out in-place partition schemes; timsort always allocates O(min(n₁,n₂)) temp storage for each merge. On memory-constrained systems (embedded, large heap pressure), this matters.

**Powersort is near-optimal, not optimal**: computing the provably optimal merge sequence requires O(k²) time for k runs. Powersort is O(k) but may pay up to ≤ n·H + 2n comparisons, vs. a true optimal that achieves n·H + O(n). The constant factor is ≤ 2×.

## Why it works

**Galloping is binary search on the wrong axis.** In a standard binary search you search for a *value* in a sorted array. Galloping searches for the *transition point* in a run of consecutive winners. The exponential probe is the classic doubling trick: overshoot by doubling, then binary search in the last interval. This is the same as the "one-sided binary search" in online algorithms for finding a sorted boundary.

**The deeper principle — sorting as entropy coding**: The cost of powersort is n·H + 2n where H = −Σ(lᵢ/n) log₂(lᵢ/n) over all run lengths. This is exactly the same formula as the Shannon entropy of a source whose symbols have probabilities proportional to run lengths.

Why? Consider the merge tree powersort builds. Each leaf is a run of length lᵢ. The cost of the merge that creates the root is proportional to n (the total length). The cost of a merge at depth d is proportional to the sum of lengths of its subtree leaves. The total cost equals Σ lᵢ · depthᵢ — exactly the total weighted path length of the binary tree, which is minimised (over all binary trees with a fixed leaf ordering) when depthᵢ = log₂(n/lᵢ) + O(1), giving a total of n·H + O(n).

This is the **alphabetic tree problem**: find the binary tree minimising total weighted path length while preserving the left-to-right order of leaves (run order must be preserved — you can only merge adjacent runs). This is the ordering-constrained variant of Huffman coding. Unrestricted Huffman can pair any two leaves; the alphabetic tree problem restricts pairs to adjacent leaves.

The alphabetic tree problem was solved by Knuth (O(n²)), then Hu-Shing (O(n log n)), and finally Mehlhorn gave an O(n log n) near-optimal algorithm (1980). Powersort achieves the same near-optimal cost in O(n) — one pass over the run lengths — by using the "power" as an online proxy for depth in the optimal tree. The insight is that two runs' optimal merge depth depends only on their positions within the array (their midpoints), not on global knowledge of all other runs.

Put differently: **sorting with timsort/powersort is entropy-bounded.** The cost you pay is proportional to the information content of the run-length description of the input. A Kolmogorov-compressible input (one with structure) sorts in sub-linear comparisons; an incompressible (random) input pays the full n log n.

This same entropy-adaptive pattern appears in:
- **Adaptive data compression** (arithmetic coding, ANS): cost = n·H bits for entropy H
- **Adaptive Huffman coding**: builds an optimal code online, one symbol at a time
- **Splay trees**: amortized access cost on a sequence is bounded by the entropy of the access distribution

The unifying principle: **algorithms that adapt their strategy to the empirical distribution of their input can achieve information-theoretically optimal cost, even when the distribution is unknown in advance.**

## Going deeper

1. **"Nearly-Optimal Mergesorts: Fast, Practical Sorting Methods That Optimally Adapt to Existing Runs"** — Munro & Wild, ESA 2018: The original powersort paper, with the full entropy bound proof and connection to alphabetic trees. Available via [Sebastian Wild's publications page](https://www.wild-inter.net/publications/munro-wild-2018).

2. **CPython listsort.txt** (primary source): Tim Peters' original design document, updated to describe powersort. Every line is pedagogically rich — read it sequentially at [github.com/python/cpython/blob/main/Objects/listsort.txt](https://github.com/python/cpython/blob/main/Objects/listsort.txt).

3. **"Optimal Alphabetic Binary Trees"** — Hu & Shing (1982) and Mehlhorn (1980): The theoretical foundation. For a modern treatment see Knuth's *The Art of Computer Programming*, Vol. 3, §6.2.2, which covers the optimal merge problem and its relation to Huffman coding.

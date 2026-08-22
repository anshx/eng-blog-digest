---
title: "Cache-Oblivious Algorithms"
source: https://dl.acm.org/doi/10.1145/2071379.2071383
author: Matteo Frigo, Charles E. Leiserson, Harald Prokop, Sridhar Ramachandran
company: MIT CSAIL
date_posted: 1999-10-01
date_digested: 2026-08-22
---

# Cache-Oblivious Algorithms

## What's new to learn

1. **The ideal-cache model**: a two-level abstract machine — cache of size *M* split into blocks of *B* words, fully associative, optimal replacement — that lets you count I/O cost without specifying hardware. Algorithm analysis in this model transfers automatically to real multi-level hierarchies.

2. **Cache-oblivious algorithms**: programs that contain no references to *M* or *B* yet are provably optimal in the ideal-cache model — and therefore simultaneously optimal at every level of the real memory hierarchy.

3. **Van Emde Boas (vEB) tree layout**: a recursive memory layout for static binary trees that achieves O(log_B N) cache misses per search without knowing B, by self-similarly packing each subtree into a contiguous block at whichever recursion level makes it fit.

---

## Prerequisites

- Binary search trees, recursion, asymptotic notation
- A rough intuition for CPU caches: L1/L2/L3 each have a size and a block (cache-line) size; accesses that miss go to the next level
- Why B-trees exist: branching factor B keeps each node within one disk block, giving O(log_B N) seeks for a search — the cache-*aware* baseline this paper beats without the tuning
- The Aggarwal–Vitter I/O model (1988) — a disk/RAM two-level model; the ideal-cache model is the same with optimal replacement added

---

## The core idea

Every cache-tuned program you've ever seen hard-codes something. A B-tree picks branching factor 4096 to match disk pages. A matrix multiply chooses tile size 64×64 to fit in L1. A sort uses a 512-element insertion-sort base case. When the hardware changes, you retune.

The cache-oblivious insight: **you don't have to tune for any level if your algorithm recurses all the way to a single element.**

Here is why. Suppose you write a matrix multiply that recursively partitions A, B, and C each into four quadrants and recurses on eight sub-problems. No tile size appears anywhere in the code. But watch what happens as the problem shrinks:

- When the sub-matrix fits in L3 (at some depth *d₃*), the recursion from that point onward costs zero L3 misses.
- When it fits in L2 (depth *d₂ > d₃*), zero L2 misses from there down.
- When it fits in L1 (depth *d₁ > d₂*), zero L1 misses from there down.

The algorithm never knows where those cutoff depths are. It doesn't have to: the recursion structure *automatically* exploits each cache level at the right grain size. The same code is simultaneously optimal for L1, L2, L3, and disk — not by accident but because the ideal-cache theorem proves it.

---

## Mechanics

### 1. The ideal-cache model

Formally: a RAM with a two-level store. The fast level (cache) holds *M* words total, organized into *B*-word **blocks**. The slow level holds the full address space. Every memory access to a word not in cache brings in its entire block (costs 1 I/O transfer). Cache is **fully associative** — any block can occupy any cache slot. Replacement is **optimal** (MIN algorithm: evict the block whose next use is farthest in the future).

The model counts **cache misses** (block transfers), not clock cycles. An algorithm is *cache-oblivious* if its code contains no references to *M* or *B*.

The **tall-cache assumption**: M ≥ B². This holds in every real hardware configuration (a 64-byte cache line in a 64 KB L1 gives B=8 words, M=8192 words, M/B=1024 ≫ B). Without it, certain lower bounds break.

### 2. Baseline: why naive binary search isn't optimal

A sorted array of N elements, searched by binary search. The access pattern jumps all over memory. In the worst case: O(log₂ N) misses (one per comparison).

A cache-aware B-tree with branching factor B does better: O(log_B N) misses. But the code explicitly uses B as a parameter. If B changes (cache line doubles in size), you rebuild the tree.

**Lower bound (from Aggarwal–Vitter 1988)**: any comparison-based search of N elements costs Ω(log_B N) cache misses. Can we match this without knowing B?

### 3. Van Emde Boas (vEB) layout

Store a complete binary tree of height *h* in memory using this recursive rule:

1. Split the tree at height ⌈h/2⌉. This creates one **top subtree** of height ⌈h/2⌉ and 2^⌈h/2⌉ **bottom subtrees** each of height ⌊h/2⌋.
2. Write the top subtree to memory contiguously (in vEB order, recursively).
3. Write each bottom subtree contiguously (in vEB order, recursively), in any order.

The defining property: any subtree of height *k* occupies exactly 2^k − 1 **consecutive** memory words.

**Why this matters for cache misses**: when you search from the root, you traverse a path of length *h = log₂ N*. Consider what happens at the recursion level where the subtree height *k* satisfies 2^k ≤ B (i.e., the subtree fits in one block). From that point down, every remaining node on the search path costs **zero additional cache misses** — they're all in one block already loaded.

The crossover depth is k = log₂ B, so the last log₂ B / 2 ≈ ½ log₂ B levels cost O(1) misses total. The first part of the path costs O(log₂ N − ½ log₂ B) = O(log₂(N/B)) misses, each in a new subtree not yet loaded.

Total: **O(log_B N)** cache misses — optimal, achieved without knowing B.

### 4. Cache-oblivious matrix transpose

Transpose an *m × n* matrix. The naive "swap A[i][j] and A[j][i]" approach is cache-hostile when the matrix is stored row-major: column accesses stride through memory.

Cache-oblivious approach:
```
transpose(A, lo_r, hi_r, lo_c, hi_c):
    if (hi_r - lo_r) * (hi_c - lo_c) <= threshold:
        naive_transpose_block(...)
        return
    if (hi_r - lo_r) >= (hi_c - lo_c):
        mid = (lo_r + hi_r) / 2
        transpose(A, lo_r, mid, lo_c, hi_c)
        transpose(A, mid, hi_r, lo_c, hi_c)
    else:
        mid = (lo_c + hi_c) / 2
        transpose(A, lo_r, hi_r, lo_c, mid)
        transpose(A, lo_r, hi_r, mid, hi_c)
```

No mention of B or M anywhere. Yet at some recursion depth the sub-matrix fits entirely in cache, giving zero misses for all sub-work thereafter.

**Cache miss bound**: Θ(1 + mn/B) — asymptotically optimal (reading mn/B blocks is a lower bound since you must touch every element at least once).

### 5. Cache-oblivious FFT

The standard Cooley–Tukey FFT splits an *N*-point DFT into two *N/2*-point DFTs and a butterfly combination, repeatedly. Frigo et al. show that when this recursion is carried to its base case (N=1), the resulting access pattern is cache-oblivious.

**Cache miss bound**: Θ(1 + (N/B)(1 + log_M N))

This matches the information-theoretic lower bound for FFT in the I/O model.

**Practical impact**: FFTW (the Fastest Fourier Transform in the West) is built on this principle. Its code generator emits recursively structured FFT codelets, and at runtime it selects which base-case sizes to use — but the cache-oblivious property is already baked in by the recursive decomposition.

### 6. Cache-oblivious sorting

Optimal comparison-based sorting costs Θ((N/B) log_{M/B}(N/B)) cache misses in the I/O model. Merge sort is a good start (each merge is sequential), but standard iterative merge sort still has poor cache behavior at the level boundary where sorted runs are roughly half the cache size.

**Funnel sort** (introduced in the same paper) achieves the optimal bound cache-obliviously via a *k-merger* (a binary tree of FIFO queues that merges k sorted sequences into one). The recursion:

1. Split N elements into N^(1/2) groups, sort each recursively.
2. Merge with a √N-way funnel.
3. The funnel itself is recursive — each merger is a binary tree of smaller mergers.

The funnel's recursive structure means that at each level, the "active" portion of the funnel fits in cache at the right grain.

**Bound**: Θ(1 + (N/B) log_{M/B}(N/B)) — optimal, without knowing M or B.

### 7. The universality theorem

The killer result: **any algorithm optimal in the ideal-cache model (with tall-cache assumption) is also optimal on any multi-level cache hierarchy.**

Proof sketch: the ideal-cache model with MIN replacement dominates any real cache policy (LRU/FIFO are at most 2× worse than MIN for programs satisfying the tall-cache assumption, by a result of Sleator and Tarjan). And a multi-level hierarchy cannot be harder than the two-level model for a cache-oblivious algorithm, because the algorithm looks the same at every grain size.

This means you write one algorithm, analyze it in the simple two-level model, and get optimal guarantees across L1, L2, L3, and disk simultaneously — at no extra coding cost.

---

## Where it breaks

**Real replacement is not optimal**: real caches use LRU or set-associative pseudo-LRU, not MIN. For cache-oblivious algorithms satisfying the tall-cache assumption, LRU adds at most a constant factor — but "constant" can mean 2–3× in practice.

**Tall-cache assumption is not always satisfied**: the analysis requires M ≥ B² (or sometimes M ≥ B^{1+ε}). This holds on every real machine today, but edge cases (tiny scratchpads on embedded GPUs) can violate it.

**Hidden constants can be large**: funnel sort has excellent asymptotic cache behavior but is slower in practice than well-tuned in-place introsort on modern hardware because its constant factors and code complexity hurt real CPUs' branch predictors and instruction caches. Theory optimizes I/O transfers; real performance depends on many more factors.

**Model ignores prefetching**: hardware prefetchers detect sequential and strided access patterns and speculatively load blocks before they're needed, collapsing the cost of predictable sequential scans. Cache-oblivious analysis doesn't model this — an algorithm that's I/O-optimal but random-access can still be slower than a cache-aware sequential one that benefits from prefetching.

**Parallelism creates conflict**: in NUMA systems, "cache-oblivious" data structures like vEB trees can cause false sharing across NUMA nodes. The recursive structure that's great for cache locality is bad for partitioning work cleanly across sockets.

**Not good for heterogeneous memory**: the ideal-cache model assumes uniform fast and slow memory. GPUs have a complex hierarchy (registers, L1, L2, HBM, host DRAM) where the bottleneck shifts by workload; a single cache-oblivious analysis captures none of this.

---

## Why it works

The deep principle is **scale invariance via recursion**.

A cache-*aware* algorithm is a point: it works optimally for one specific (M, B) pair. A cache-*oblivious* algorithm is a *function* over all (M, B) pairs, because its recursive structure looks identical at every scale.

Put another way: the hierarchy has levels at multiple scales. A recursive algorithm that halves its input at each step passes through all those scales. At each scale, some portion of the recursion tree fits within the cache at that level. The algorithm automatically exploits each cache level at the grain that fits.

This is the same principle as:

- **Divide-and-conquer in algorithms** — any problem where the optimal solution recurses to base cases is naturally "self-similar" in the same way.
- **Fractal geometry** — a fractal looks the same at every zoom level; a cache-oblivious algorithm has the same cache-behavior at every recursion depth.
- **Wavelet transforms** — recursive multi-scale decomposition is cache-oblivious for the same structural reason.
- **B-trees as a cache-*aware* special case** — a B-tree is the cache-oblivious idea frozen at one level: "pack B keys per node" is the single-scale version of "recurse until you fit in B."

The vEB tree layout makes this explicit: it's a recursive *memory layout* rather than a recursive *algorithm*. The layout self-similarly packs any subtree into a contiguous block at the depth where it fits, without knowing in advance what "fitting" means for any particular B.

The universality theorem is the formal proof that scale-invariance is enough: if you're optimal at one (M, B), the ideal-cache model's structural properties guarantee you're optimal at all of them.

---

## Going deeper

1. **Demaine 2002 survey** — "Cache-Oblivious Algorithms and Data Structures" (in Proc. EEF Summer School on Algorithm Theory) — a more accessible 40-page treatment that covers cache-oblivious B-trees, priority queues, and sorting; a good next read after the original paper. Freely available from Demaine's MIT webpage.

2. **Bender, Demaine, Farach-Colton 2000** — "Cache-Oblivious B-Trees" (FOCS 2000) — how to build a *dynamic* (insert/delete capable) search tree with the vEB layout using a packed-memory array as the substrate; the key additional technique is that a small unsorted buffer per leaf amortizes the cost of maintaining the vEB order under insertions.

3. **FFTW source code** — Frigo implemented the cache-oblivious FFT ideas directly in FFTW; reading its codegen directory shows how the recursive decomposition gets implemented for real hardware. FFTW papers (available at fftw.org) describe how the planner chooses base-case sizes empirically while the recursive structure provides the cache-oblivious guarantee.

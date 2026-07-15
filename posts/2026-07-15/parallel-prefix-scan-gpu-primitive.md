---
title: "Parallel Prefix Sum (Scan) with CUDA"
source: https://developer.nvidia.com/gpugems/gpugems3/part-vi-gpu-computing/chapter-39-parallel-prefix-sum-scan-cuda
author: Mark Harris
company: NVIDIA
date_posted: 2007
date_digested: 2026-07-15
---

# Parallel Prefix Sum (Scan) with CUDA

## What's new to learn

1. **Parallel prefix scan**: Given an array and any associative binary operator, each output element is the cumulative combination of all preceding inputs. A prefix *sum* is the most common instance, but the operator can be anything — max, XOR, matrix multiply, logical AND.

2. **The Blelloch two-phase algorithm**: A work-efficient O(n) parallel scan using a balanced binary tree. An *upsweep* (reduce) phase aggregates sums up the tree; a *downsweep* (distribute) phase propagates partial sums back down. Together they achieve O(log n) depth with only 2n − 2 total operations — optimal.

3. **Work efficiency vs. parallelism depth**: Naive parallel scan (Hillis-Steele) achieves O(log n) depth but does O(n log n) total work. On a machine with fewer processors than n this wastes throughput. Blelloch's approach is both depth-optimal and work-optimal; understanding why requires distinguishing *span* (depth) from *work* in parallel complexity.

## Prerequisites

- Familiarity with GPU thread blocks and shared memory (not required to understand the algorithm, but necessary to follow the CUDA implementation)
- Basic parallel computing concepts: SIMD, thread divergence, warp synchronization
- Understanding of why sequential prefix sums are O(n): each step reads all previous outputs

## The core idea

A prefix sum looks inherently sequential: to compute `out[i] = a[0] + a[1] + ... + a[i]` you seem to need `out[i-1]` first. The insight that breaks this is **associativity**.

Because addition associates (`(a+b)+c == a+(b+c)`), you can parenthesize the computation any way you like. Instead of left-to-right evaluation, use a balanced binary tree: in O(log n) parallel rounds, combine pairs of neighbors, then pairs of those results, and so on. The tree has the same leaves and root as the sequential sum, but the work is laid out differently.

Blelloch's algorithm exploits this with two passes over a binary tree built on the input:

**Phase 1 — Upsweep (reduce):** Start at the leaves. In each round d, every other node at level d adds its left child's value into its own. After log n rounds, the root holds the total sum. Cost: n−1 additions, O(log n) depth.

**Phase 2 — Downsweep (distribute):** Set the root to 0 (identity). Sweep back down: for each node, give its left child the node's current value, and give its right child `node_value + left_child_original`. After log n rounds, each leaf holds the exclusive prefix sum up to its position. Cost: n−1 operations, O(log n) depth.

An *exclusive* scan at position i holds the sum of all elements *before* i (so `out[0] = 0`). An *inclusive* scan holds the sum *including* element i. Exclusive scan is more useful algorithmically — you need to know where my segment *starts*, not where it *ends*.

## Mechanics

**Hillis-Steele (naive, not work-efficient):**
```
for d = 1, 2, 4, 8, ..., n/2:
    parallel for all i >= d:
        a[i] += a[i - d]
```
Total work: O(n log n). On a 32-wide warp, this wastes most threads in early rounds.

**Blelloch scan on a single tile (in shared memory):**
```
// Upsweep
stride = 1
while stride <= n/2:
    parallel for i = stride-1 to n-1 step 2*stride:
        shared[i + stride] += shared[i]
    stride *= 2

shared[n-1] = 0  // set root to identity

// Downsweep
stride = n/2
while stride >= 1:
    parallel for i = stride-1 to n-1 step 2*stride:
        left  = shared[i]
        shared[i]          = shared[i + stride]
        shared[i + stride] += left
    stride /= 2
```
Total work: 2n − 2 additions, 2 log n parallel rounds.

**Multi-tile global scan:** For arrays larger than one tile (typically 1024 elements on CUDA), the algorithm applies hierarchically:
1. Scan each tile independently in shared memory; write each tile's total to a small `partial_sums` array.
2. Scan the `partial_sums` array (recursively, same algorithm).
3. Add each tile's corresponding scanned partial sum to every element in that tile.

This gives an overall three-kernel pipeline: `tile_scan → scan_partial_sums → add_offsets`.

**Avoiding shared memory bank conflicts:** CUDA shared memory is divided into 32 banks; simultaneous accesses to the same bank serialize. The sequential tree traversal naturally accesses strided memory. Adding a small padding word every 32 entries shifts conflicting accesses to different banks, recovering full bandwidth. The GPU Gems chapter shows this adds ~10% throughput improvement in practice.

**Performance numbers (from the chapter, 2007 hardware):**
- Naïve O(n log n) scan: ~0.5 GB/s effective throughput
- Work-efficient Blelloch scan: ~6.3 GB/s (2048-element tile, GeForce 8800 GTX)
- Optimal bandwidth for that GPU: ~86 GB/s — showing that even this implementation was still bandwidth-limited by global memory round-trips

Modern CUB implementations using decoupled look-back (Merrill & Garland 2016) achieve 95%+ of peak memory bandwidth on Ampere/Ada GPUs by eliminating the three-kernel pipeline and producing a single-pass scan.

**Key applications of scan:**
- **Compaction / stream filter**: compute flag array, scan it for output indices, scatter matching elements. Turns O(n) sequential filter into O(n) work, O(log n) depth parallel filter.
- **Radix sort**: use scan to compute histogram bucket start offsets, then scatter keys. The scan is why parallel radix sort beats comparison sort on GPUs.
- **Sparse format construction**: CSR row pointer array is just a scan of per-row nnz counts.
- **Variable-length encoding**: scan of element sizes gives each element's output byte offset, enabling parallel encoding.
- **Ring AllReduce scatter-reduce phase**: gradients are segmented; each worker computes partial sums of one segment. The segments are then rotated — this is a parallel prefix over the ring topology with gradient tensors as elements.
- **Mamba selective scan**: the SSM state update `h_t = A_t * h_{t-1} + B_t * x_t` is a first-order linear recurrence. Linearizing: `(h_t, 1) = (A_t, B_t*x_t) ⊗ (h_{t-1}, 1)` where `⊗` is a 2×2 matrix multiply. This makes it a parallel prefix scan with matrix multiplication as the monoid — exactly the Blelloch algorithm, executed on GPU SRAM to avoid HBM materialization.

## Where it breaks

**Operator must be associative** — not commutative, but associative. This rules out floating-point "exact summation" because floating-point addition is only approximately associative. In practice this rarely matters: the reordering introduces at most O(n · ε) error, the same order as other floating-point accumulation schemes.

**Only for reduction-like operations** — scan doesn't generalize to arbitrary dependencies. If element i's computation depends on the *result* (not just the input) of element i−1, you cannot parallelize it this way.

**Small-array overhead** — for n < a few hundred elements, the parallelism overhead (kernel launch, synchronization) outweighs the gain. Sequential scan on CPU L1 cache beats GPU scan for tiny inputs.

**Memory bandwidth is usually the real limit** — on modern GPUs, compute is far ahead of memory bandwidth. A scan that reads and writes the entire array twice is memory-bound. The work-efficiency of Blelloch vs. Hillis-Steele barely matters if both pipelines saturate memory bandwidth first.

## Why it works

The fundamental insight is that **associativity is equivalent to free re-parenthesization**, and any computation expressible as a binary tree can be evaluated in O(log n) parallel steps with O(n) work by choosing the tree structure of the computation.

This is a specific instance of a broader principle in parallel complexity: the **NC class** (problems solvable in polylogarithmic depth with polynomially many processors) is characterized precisely by problems reducible to associative operations over a monoid. The monoid structure — associativity and an identity element — is all you need to unlock parallel evaluation.

The Blelloch upsweep/downsweep scheme is provably optimal: any circuit computing all n prefix values requires at least n − 1 binary operations, so 2n − 2 operations is within a factor of 2 of optimal. The depth bound O(log n) is also optimal by the same circuit argument.

The unifying frame: every "aggregation" operation in systems across this archive is a parallel prefix scan under some monoid:

| System | Operator | Identity |
|--------|----------|----------|
| Ring AllReduce scatter-reduce | gradient addition | zero tensor |
| Mamba selective scan | 2×2 SSM state matrix | identity matrix |
| GPU global reduce | sum / max / AND | 0 / −∞ / true |
| Segment tree range query | associative range fn | neutral element |
| Radix sort histogram offsets | integer addition | 0 |
| Stream compaction | count (flag sum) | 0 |
| Kafka offset assignment | message count | 0 |

The GPU Gems chapter revealed this in the context of CUDA, but the algebraic insight predates it by decades — it goes back to Blelloch's 1990 work on NESL and to Ladner & Fischer's 1980 circuit theory for parallel prefix.

The practical lesson: whenever you see a computation whose output at position i depends on all inputs before i, ask whether the dependency can be satisfied by an *associative combination* rather than a chain of dependent reads. If yes, your O(n) sequential loop can become an O(log n) depth parallel primitive — for free.

## Going deeper

1. **Guy Blelloch, "Prefix Sums and Their Applications" (1990), CMU-CS-90-190** — the original technical report that framed scan as the building block of data-parallel programming. Describes 14 fundamental algorithms (sort, graph search, tree operations) that reduce to scan. Available at https://www.cs.cmu.edu/~blelloch/papers/Ble90.pdf.

2. **Duane Merrill & Michael Garland, "Single-pass parallel prefix scan with decoupled look-back" (2016)** — shows how to eliminate the three-kernel pipeline and achieve single-pass global scan by having each tile "look back" at predecessor tile results via a small flag word in global memory. The technique is in NVIDIA CUB and Thrust today. arxiv.org/abs/1611.07974.

3. **Mamba paper — Gu & Dao, "Mamba: Linear-Time Sequence Modeling with Selective State Spaces" (2023)** — Section 3.3 ("Hardware-aware Algorithm") describes exactly how Blelloch's parallel scan is applied to SSM state updates inside GPU SRAM to avoid materializing the expanded state in HBM, directly paralleling the FlashAttention IO-awareness insight. arxiv.org/abs/2312.00752.

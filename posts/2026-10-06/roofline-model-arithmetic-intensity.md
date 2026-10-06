---
title: "Roofline: An Insightful Visual Performance Model for Floating-Point Programs and Multicore Architectures"
source: https://dl.acm.org/doi/10.1145/1498765.1498785
author: Samuel Williams, Andrew Waterman, David Patterson
company: UC Berkeley / Lawrence Berkeley National Laboratory
date_posted: 2009-04-01
date_digested: 2026-10-06
---

# Roofline: An Insightful Visual Performance Model for Floating-Point Programs and Multicore Architectures

## What's new to learn

1. **Arithmetic Intensity (AI)**: the ratio of floating-point operations performed to bytes transferred to/from DRAM — the single number that predicts which hardware bottleneck a kernel will hit.
2. **The two roofs**: every processor has a *peak compute* ceiling (GFLOP/s) and a *peak memory bandwidth* ceiling (GB/s × AI), and a kernel's attainable performance is bounded by whichever it reaches first.
3. **The ridge point**: the critical AI value (= peak_compute / peak_bandwidth) at which a kernel transitions from memory-bandwidth-bound to compute-bound — the break-even where both resources are simultaneously saturated.

## Prerequisites

- Basic understanding of processor architecture: cores, SIMD, DRAM vs. cache hierarchy
- Familiarity with FLOP/s as a performance unit and memory bandwidth in GB/s
- A rough sense of how matrix operations work (multiply-add loops)

## The core idea

Every program has two potential hardware bottlenecks: it can run out of *compute capacity* (FLOPs/s the ALUs can issue), or it can run out of *memory bandwidth* (bytes/s the DRAM can deliver). For a given kernel, which limit you hit depends on one number: how many FLOPs you do per byte you move from DRAM, called **Arithmetic Intensity** (AI).

Plot this on a log-log chart. On the x-axis: AI in FLOP/byte. On the y-axis: achievable performance in GFLOP/s. Two lines form the "roof":

- A **horizontal line** at the processor's peak compute rate (compute-bound ceiling).
- A **diagonal line** with slope = 1 starting from the origin at height = peak_bandwidth × AI (memory-bandwidth-bound ceiling).

Any kernel must sit at or below both lines — i.e., its performance ≤ min(peak_compute, peak_bandwidth × AI). The lines intersect at the **ridge point**, where AI = peak_compute / peak_bandwidth. To the left: you're memory-bound. To the right: you're compute-bound.

The power of this model is that you can reason about *two separate questions* independently: (1) where does your kernel sit on the AI axis? (2) which roof does it hit? The gap between your measured performance and the relevant roof tells you how much optimization potential remains.

## Mechanics

### Measuring Arithmetic Intensity

For a dense matrix-matrix multiplication (C = A × B), where A, B, C are N×N matrices of doubles:

- **FLOPs**: 2N³ (N³ multiplies + N³ adds)
- **DRAM bytes**: 3 × N² × 8 = 24N² bytes (read A, B; write C; each element is 8 bytes)
- **AI** = 2N³ / 24N² = **N/12 FLOP/byte**

AI grows with N. At N=12: AI = 1 FLOP/byte. At N=1200: AI = 100 FLOP/byte. The same algorithm moves from firmly memory-bound to firmly compute-bound just by increasing problem size — and the roofline plot shows exactly where that transition happens.

For dense matrix-*vector* multiplication (y = Ax), N×N times N:

- **FLOPs**: 2N²
- **DRAM bytes**: N²×8 (read A) + N×8 (read x, negligible) ≈ 8N²
- **AI** = 2N²/8N² = **0.25 FLOP/byte**

This is *always* memory-bound, regardless of N, because FLOP growth is the same order as data growth. The roofline model makes this obvious immediately.

For **sparse matrix-vector multiply** (SpMV), with ~7 non-zeros per row:

- Each non-zero: 2 FLOPs (multiply + add), 12 bytes (8 for float value + 4 for index)
- **AI ≈ 2/12 ≈ 0.17 FLOP/byte** — even lower. Memory-bound forever.

### Reading the roofline plot

On a modern GPU (NVIDIA A100):

- Peak BF16 Tensor Core compute: **312 TFLOP/s**
- HBM2e memory bandwidth: **2000 GB/s**
- **Ridge point** = 312×10¹² / 2000×10⁹ = **156 FLOP/byte**

Any kernel with AI < 156 FLOP/byte is memory-bandwidth-bound on an A100. Standard attention (materializing the N×N attention matrix in HBM) has AI around 1–10 FLOP/byte — far below the ridge point. Dense matrix-matrix multiplication with large N easily exceeds it.

### Ceilings below the roofs

The paper goes further by identifying *ceilings* — achievable performance under various restrictions:

**Compute ceilings** (from top to bottom):
1. Peak SIMD + FMA (the actual roof)
2. Peak SIMD without FMA (half the roof, if FMA instructions unused)
3. No SIMD: scalar throughput only (1/4 to 1/8 of roof on typical hardware)

**Memory bandwidth ceilings**:
1. HW prefetch + unit-stride accesses (the actual memory roof)
2. HW prefetch + non-unit-stride accesses (~½ roof)
3. No prefetch (much lower)

By plotting where your kernel actually lands relative to these ceilings, you can identify which specific optimization to apply first. If you're below the "no SIMD" compute ceiling, vectorization is the high-leverage move. If you're below the "unit-stride" bandwidth ceiling, access pattern restructuring is.

### The optimization loop

1. **Compute your kernel's AI** (via hardware counters: PMU events or `perf stat`).
2. **Plot it on the roofline** for your target hardware.
3. **Identify the binding constraint**: memory-bound or compute-bound?
4. **Apply the right optimization**:
   - *Memory-bound*: reduce data movement — loop tiling/blocking to fit working set in L2/L3, change traversal order for unit-stride access, data layout transformations (AoS → SoA).
   - *Compute-bound*: increase throughput — SIMD vectorization, FMA fusion, register blocking, reduce branch divergence.
5. **Re-measure AI and performance**. Tiling *increases* AI (same FLOPs, less DRAM traffic) so you may cross the ridge point and become compute-bound; then switch optimization strategies.

### Case study numbers (from the 2009 paper)

The paper benchmarks seven HPC kernels on four multicore platforms of the era (Intel Clovertown 8-core at 41.6 GFLOP/s peak, AMD Barcelona at 38.4 GFLOP/s, Intel Nehalem at 83.2 GFLOP/s, etc.):

| Kernel | AI (FLOP/byte) | Bound |
|--------|----------------|-------|
| SpMV | 0.17 | Memory |
| Stencil | 0.5 | Memory |
| FFT | 1.1 | Memory |
| N-body | 2.0 | Memory/Transitional |
| Dense BLAS-3 (large N) | 100+ | Compute |

Most real-world kernels cluster at AI < 10 — firmly below the ridge point — which means the bottleneck is almost always memory bandwidth, not compute. This was the key insight of the paper: hardware vendors were racing to deliver more TFLOP/s while workloads were almost universally memory-bandwidth-limited.

## Where it breaks

**Cache hierarchy simplification**: the model counts only DRAM bytes. If a kernel's working set fits in L2/L3 cache, the relevant bandwidth is cache bandwidth (typically 5–20× higher than DRAM), and the effective AI against that bandwidth tier is much higher. Multi-level roofline models (one "roof" per memory level) exist but add complexity.

**Arithmetic intensity is kernel-specific, not program-wide**: a full application is a sequence of kernels with different AIs; end-to-end performance depends on kernel composition and overlap.

**GPU occupancy and latency hiding not captured**: on GPUs, if you don't have enough warps to hide memory latency (due to register pressure or shared memory limits), you hit an "occupancy cliff" that the basic roofline doesn't model. The *Empirical Roofline Toolkit* (ERT) and *roofline + latency-hiding* extensions address this.

**Integer and mixed-precision work**: the original model focuses on floating-point. Integer kernels (hash joins, sort) are often memory-bound at very different AIs, and mixed-precision (INT4 weight + FP16 activation) requires separate roofs for each type.

**Communication-computation overlap**: distributed training has a *communication roof* (AllReduce bandwidth) distinct from the memory roof. Ring AllReduce's bandwidth-optimality matters precisely because it keeps the communication roof as high as possible.

## Why it works

The roofline model is a direct application of **Amdahl's Law to hardware resource contention**: when two resources (compute and memory bandwidth) are both needed, the slower one is the bottleneck, and the faster one is wasted. The min() ceiling over two independent resource capacities is the exact same structure as Amdahl's parallel speedup formula.

The deeper principle is **balanced system design**. An "ideally balanced" hardware design has peak_compute / peak_bandwidth = the AI of the dominant workload. When designers add more FLOP/s without proportional memory bandwidth growth (as happened with early GPU TFLOP races), they push the ridge point right while typical workload AIs stay fixed — wasting compute budget. When Apple Silicon unified its CPU/GPU memory and dramatically widened the memory bus (~800 GB/s for M-series), it was lowering the ridge point to match the AI of graphics and ML workloads.

**Every architecture choice is implicitly a bet on the AI of dominant workloads:**

- **Tensor Cores** (on NVIDIA A100) raise the compute roof by ~8× for matrix ops while keeping the memory roof constant — they're only useful for kernels above the *new* (higher) ridge point.
- **FlashAttention** is "loop tiling to increase AI past the ridge point": by processing attention in tiles that fit in SRAM, it eliminates HBM round-trips for the NxN matrix, raising effective AI from ~2 to ~50+ FLOP/byte and shifting standard attention from memory-bound to compute-bound.
- **MonetDB/X100's vector model** keeps column vectors in L1/L2 cache, replacing DRAM bandwidth with cache bandwidth — a 10–50× higher roof.
- **BLAS Level 3 vs Level 2**: GEMMs have O(N) AI; GEMVs have O(1) AI. Hardware designers optimized for GEMM; BLAS libraries exploit this by recasting Level 2 work as Level 3 where possible.

The "oh, X is just an instance of Y" insight is: *every performance optimization technique is either "raise the relevant roof" (vectorize, use Tensor Cores) or "increase arithmetic intensity past the current roof" (tile, fuse, reorder), and the roofline plot tells you which to do first.*

## Going deeper

1. **"Empirical Roofline Toolkit" (ERT)** — [cs.berkeley.edu/~bbrock](https://crd.lbl.gov/divisions/amcr/computer-science-dept/par/research/roofline/software/ert/) — the practical tool for measuring machine roofline ceilings on your specific hardware.
2. **FlashAttention-2 paper** (Tri Dao, 2023) — the analysis section explicitly applies roofline thinking to show why fused tiled attention changes the regime from memory-bound to compute-bound; Section 3 is a direct worked example of roofline-guided optimization.
3. **"Performance Analysis and Tuning on Modern CPUs"** by Denis Bakhvalov (2024 book, free online at easyperf.net) — extends roofline to the full microarchitecture bottleneck taxonomy (front-end bound, back-end bound, memory bound, retiring) using Intel's Top-Down Microarchitecture Analysis (TMA) framework.

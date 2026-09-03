---
title: "Flash-Decoding for Long-Context Inference"
source: https://pytorch.org/blog/flash-decoding/
author: Tri Dao, Daniel Haziza, Francisco Massa, Grigory Sizov
company: TriDAO / Meta AI / Stanford
date_posted: 2023-10-12
date_digested: 2026-09-03
---

# Flash-Decoding for Long-Context Inference

## What's new to learn

- **Sequence-dimension parallelism during decode**: FlashAttention parallelizes attention across the batch and head dimensions, but during autoregressive generation there is only 1 query token — so those dimensions alone leave most GPU SMs idle. Flash-Decoding adds a third parallelism axis by splitting the KV sequence into independent chunks, one per SM.

- **Online softmax is a composable partial reduction**: The triple (max_score, sum_exp, weighted_value_sum) is a sufficient statistic for a partial softmax result. Two independent partial results can be merged exactly without re-reading either chunk — making attention an associative reduction amenable to parallel decomposition exactly like MapReduce combine or Ring AllReduce scatter-reduce.

- **Utilization collapse during decode**: Even a fully-loaded serving system can have near-zero SM utilization during decode steps if `batch_size × num_heads < num_SMs`. This is the hidden bottleneck behind slow long-context generation.

## Prerequisites

- How autoregressive generation works: one new token per forward pass, attending to all previous KV pairs.
- FlashAttention's tiled approach: tiles the N×N attention matrix to stay in SRAM, using online softmax per tile to avoid materializing the full matrix. (See the 2026-05-26 digest entry.)
- GPU architecture basics: SM (streaming multiprocessor) as the unit of parallelism; HBM as slow global memory vs SRAM as fast on-chip memory; memory bandwidth as the decode-phase bottleneck (not FLOP count).

## The core idea

During prefill, every query token has its own work item: N queries × h heads × b batch items = N·h·b independent pieces of work. An A100 has 108 SMs, so as long as this product is in the hundreds, all SMs stay busy.

During decode, there is only **one** new query token. The product becomes 1 × h × b = h·b. For LLaMA-2-70B with h=64 heads and batch size b=1, that's 64 work items — barely 59% of 108 SMs, and the other 41% sit idle. Shrink the batch (common in latency-sensitive deployments) and utilization craters further.

Flash-Decoding's solution: split K and V into C chunks of length N/C across the sequence dimension. Now there are C × h × b independent work items. For C=128 chunks, b=1, h=64, that's 8192 items — full utilization across 108 SMs and then some. Each SM computes a partial attention output over its chunk, then a cheap second kernel merges the partial results into the true answer.

## Mechanics

Given a single query vector **q** ∈ ℝ^d, key matrix **K** ∈ ℝ^{N×d}, value matrix **V** ∈ ℝ^{N×d}:

**Step 1 — Split**: Partition K and V into C contiguous chunks: **K**_i ∈ ℝ^{(N/C)×d}, **V**_i ∈ ℝ^{(N/C)×d}. Assign chunk i to an independent SM.

**Step 2 — Parallel partial attention** (one SM per chunk):

Each SM runs a mini FlashAttention over its (N/C)-length chunk and produces three values:

```
s_i  = q · K_iᵀ / √d          # raw attention logits for this chunk, shape (N/C,)
m_i  = max(s_i)                 # running max for numerical stability
l_i  = Σ_j exp(s_i[j] − m_i)   # normalisation denominator
o_i  = Σ_j exp(s_i[j] − m_i) · V_i[j]   # unnormalised output, shape (d,)
```

Write the triple (m_i, l_i, **o**_i) back to HBM. This is cheap: 2 scalars + 1 d-dimensional vector per SM.

**Step 3 — Reduce** (a single fast kernel):

Merge C partial results into the final output:

```
m    = max(m_1, ..., m_C)
l_i' = l_i · exp(m_i − m)        # rescale each partial denominator
l    = Σ_i l_i'                   # total denominator
o    = Σ_i (l_i' · o_i) / l       # final output
```

The rescaling factor `exp(m_i − m)` corrects for the fact that each SM used its own local max for stability. Because max is taken globally in the reduction and each partial result is re-weighted by `exp(local_max − global_max)`, the result is numerically identical to computing the full softmax in a single pass.

**Why this works**: The triple (m_i, l_i, **o**_i) is a *sufficient statistic* for the partial softmax. Two partial results can be combined without any information loss. This property makes attention parallelizable over the sequence dimension in the same way summation is parallelizable over an array.

**Performance**:

| Context length | Batch size | Speedup vs. standard attention |
|---|---|---|
| 64K tokens | 1 | ~8× |
| 16K tokens | 1 | ~2× |
| 4K tokens | 1 | ~1.2× |
| <1K tokens | any | negligible or negative |

The break-even point is around 4K tokens because the intermediate HBM write (step 2) and the reduction kernel (step 3) add overhead that only pays off once the sequence is long enough that the utilization gain outweighs it.

## Where it breaks

**Short sequences**: Below ~4K tokens, standard FlashAttention fills the GPU adequately via batch × head parallelism. Flash-Decoding adds HBM round-trips for the intermediate (m, l, o) values, making it strictly slower.

**Large batches**: If b × h ≫ 108 already, the GPU is fully utilised without sequence splitting. Flash-Decoding's extra overhead then hurts. It is a niche optimization for the small-batch / long-context regime.

**Variable-length batches**: Real serving systems often pack sequences of different lengths into one batch. Chunking a ragged batch evenly is messy; uneven chunks cause some SMs to finish early and idle. Production implementations (e.g., FlashInfer) handle this with page-table-style block addressing rather than contiguous splits.

**Extra HBM pressure**: The (m, l, **o**) intermediate values must be written and re-read for each chunk. For very narrow d (small head dimension), this round-trip dominates. Flash-Decoding assumes large d (e.g., 128) where the intermediate write is cheap relative to the compute savings.

## Why it works

Flash-Decoding is an instance of a much more general pattern: **parallel reduction over a monoid where the monoid element is a sufficient statistic**.

Softmax is not immediately associative — `softmax([1,2]) + softmax([3,4]) ≠ softmax([1,2,3,4])`. But if you carry (max, sum_exp) alongside each partial output, you can reconstruct the globally-normalised result from any set of partial results. The (m, l, **o**) triple is the "monoid element" for attention-weighted sums, and the merge step in Step 3 is just the monoid's combine operation.

This is the same structure as:

- **Ring AllReduce** (covered in the 2026-06-19 digest): each worker accumulates partial gradients; a scatter-reduce phase combines them element-wise. The monoid is ℝ^d under addition.
- **Parallel prefix scans** (covered 2026-07-15): the monoid is any associative operator; Blelloch's upsweep/downsweep executes it in O(log n) depth with O(n) work.
- **HyperLogLog** (covered 2026-06-05): each register holds the maximum observed leading-zero count. Merging two HLL sketches takes the element-wise max — another composable partial aggregate.
- **MapReduce combine** (covered 2026-06-25): commutative-monoid aggregation is the condition for correctness of the combine/reduce phase.

The key diagnostic question for any aggregation: *Can I express the partial result as a fixed-size sufficient statistic that supports an associative merge?* If yes, the computation parallelizes over any dimension of the input. Online softmax discovered that attention falls into this class; Flash-Decoding exploits that to reclaim GPU parallelism on the one axis (sequence length) that previous attention kernels left unparallelized.

Concretely, the technique generalises to any attention variant where the reduction dimension (sequence length) is independent across positions. Causal (masked) attention adds a minor complication — chunks must respect the causal mask — but the composable-statistics approach still works.

## Going deeper

1. **FlashAttention-3 paper** (Dao et al., 2024): Shows how asynchronous warp specialization (covered in the 2026-07-26 digest) and low-precision arithmetic combine with Flash-Decoding-style parallelism to push A100 attention kernels to 75% of peak FLOPs/s.

2. **FlashInfer** (Ye et al., 2024) — [github.com/flashinfer-ai/flashinfer](https://github.com/flashinfer-ai/flashinfer): Production implementation that handles paged KV caches (PagedAttention-style block tables), ragged batches, and speculative decoding. The source code is the best place to see Flash-Decoding mechanics in real kernels.

3. **"Online normalizer calculation for softmax"** (Milakov & Gimelshein, 2018): The paper that first described the (max, sum_exp) carry-forward trick at the per-tile level. Flash-Decoding applies that same trick at the inter-SM level; understanding the original tile-level formulation makes the cross-SM generalisation obvious.

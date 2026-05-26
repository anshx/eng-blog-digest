---
title: "FlashAttention: Fast and Memory-Efficient Exact Attention with IO-Awareness"
source: https://arxiv.org/abs/2205.14135
author: Tri Dao, Daniel Y. Fu, Stefano Ermon, Atri Rudra, Christopher Ré
company: Stanford University / Dao-AILab
date_posted: 2022-05-27
date_digested: 2026-05-26
---

# FlashAttention: Fast and Memory-Efficient Exact Attention with IO-Awareness

## What's new to learn

- **IO complexity vs. FLOP complexity**: The right way to analyze an algorithm on modern GPUs is to count bytes moved between HBM (main GPU memory) and on-chip SRAM, not just floating-point operations — because for attention, the memory bus is the bottleneck by a factor of ~1000×.
- **Online softmax**: The exact softmax over an arbitrary-length sequence can be computed in a single streaming pass using only O(1) running statistics (a running maximum and a running normalizer), without ever materializing the full attention score matrix.
- **Recomputation as a memory trade**: Discarding the N×N attention matrix and recomputing it tile by tile in the backward pass trades a ~33% increase in FLOP count for a quadratic-to-linear peak memory reduction — a trade that is almost always worth making.

## Prerequisites

- Self-attention: what Q, K, V matrices are, and what scaled dot-product attention computes: `softmax(QK^T / sqrt(d)) * V`
- GPU memory tiers: the difference between HBM (tens of GB, ~2 TB/s bandwidth) and on-chip SRAM (~20 MB on an A100, ~19 TB/s bandwidth)
- Numerical softmax stability: why we subtract the row maximum before exponentiating
- Backpropagation: why intermediate activations must be stored during the forward pass to compute gradients in the backward pass

## The core idea

Standard self-attention computes attention scores for every pair of query and key positions, producing an N×N matrix. For N=4,096 and 16-bit floats that is a 128 MB tensor. The GPU reads and writes it multiple times: once to store the raw scores, again to read them for the softmax, again to write the normalized probabilities, and again to read those probabilities for the weighted sum with V. For large N this matrix dwarfs the useful compute.

The insight in FlashAttention is that the N×N matrix never needs to exist in HBM. Attention can be computed in tiles that fit entirely in on-chip SRAM: load a block of K and V from HBM, loop over blocks of Q and accumulate partial outputs using an online softmax update, then write the final output once. The output is mathematically identical to standard attention — no approximation is involved. The speedup comes purely from eliminating unnecessary memory traffic.

## Mechanics

### GPU memory hierarchy and the roofline

An A100 GPU has ~80 GB of HBM at ~2 TB/s and ~20 MB of L2/SRAM at ~19 TB/s — a 10× bandwidth gap. A computation's **arithmetic intensity** (FLOPS per byte accessed) determines whether it is compute-bound (high intensity, GPU cores are the limit) or memory-bound (low intensity, the HBM bus is the limit).

For attention with sequence length N and head dimension d:
- FLOPS: ~4N²d (two matrix multiplications)
- HBM bytes (standard): ~2N²+ 4Nd — dominated by the N×N attention matrix
- Arithmetic intensity ≈ d/N

For N=4,096 and d=64, this is 0.016 FLOPS/byte, far below the A100's roofline ridge point (~900 FLOPS/byte). Standard attention is spending 99.9% of its time waiting for memory, not computing. Every optimization that reduces FLOP count while leaving memory traffic unchanged is wasted effort.

### The tiling algorithm (forward pass)

FlashAttention partitions K and V into column blocks of width B_c = ⌈M / (4d)⌉, where M is the SRAM size. It partitions Q into row blocks of height B_r = min(B_c, d). The outer loop iterates over K/V column blocks (loaded once to SRAM), and the inner loop iterates over Q row blocks.

For each pair (Q_i, K_j, V_j) that fit together in SRAM:
1. Compute the raw attention scores: `S_ij = Q_i @ K_j^T ∈ ℝ^{B_r × B_c}`
2. Compute the block's row-wise max: `m̃_ij = rowmax(S_ij)`
3. Compute block-local unnormalized attention: `P̃_ij = exp(S_ij - m̃_ij)`
4. Compute block-local partition sum: `l̃_ij = rowsum(P̃_ij)`
5. Update running statistics using the online softmax rule (see below)
6. Accumulate the output: `O_i += P̃_ij @ V_j` (with appropriate rescaling)

After all blocks of K/V have been visited, the accumulated O_i is divided by the final normalizer l_i to produce the correct output.

**HBM access count:** Each element of K and V is loaded into SRAM T_r = ⌈N/B_r⌉ times (once per Q block), giving total K/V reads of O(N²d/M). For M=20 MB and d=64, this is roughly 40× fewer bytes than the standard O(N²) reads for the attention matrix alone.

### The online softmax update rule

The core mathematical trick that makes tiling possible is that softmax can be computed incrementally. Given running statistics from all K/V blocks seen so far — a row-wise maximum m ∈ ℝ^{B_r} and a row-wise normalizer l ∈ ℝ^{B_r} — the update when processing a new block with scores S_new is:

```
m_new   = max(m_old, rowmax(S_new))

l_new   = exp(m_old - m_new) * l_old
        + rowsum(exp(S_new - m_new))

O_new   = diag(l_new)^{-1} * (
            diag(l_old) * diag(exp(m_old - m_new)) * O_old
            + exp(S_new - m_new) @ V_new
          )
```

The rescaling factor `exp(m_old - m_new)` corrects the previously accumulated output and normalizer for the updated maximum. After T_c iterations, O_new contains the exact softmax-weighted output for every query position.

This is the **streaming maximum / online normalizer** algorithm (Milakov & Gimelshein, 2018), applied here to allow one-pass, tile-by-tile accumulation of softmax output.

### Backward pass via recomputation

Standard autodiff saves all intermediate tensors during the forward pass for use in computing gradients. For attention this includes the full P = softmax(QK^T/√d) matrix — O(N²) storage.

FlashAttention discards P during the forward pass and instead saves only the per-row softmax statistics (m, l), which are O(N). In the backward pass, it recomputes P tile by tile (repeating the forward tiling pass) and immediately uses each tile to compute the corresponding gradient. Peak memory drops from O(N²) to O(N). FLOP count increases by ~33% (one extra forward pass), but this overhead is dominated by the memory savings on any sequence length where standard attention was feasible at all.

### Measured results (A100 80GB, PyTorch 1.13)

| Sequence length | Standard attention speedup (FA vs. baseline) | Memory savings |
|---|---|---|
| 512 | 1.7× | 3× |
| 1,024 | 2.4× | 5× |
| 2,048 | 3.3× | 10× |
| 4,096 | 4.0× | 20× |

End-to-end GPT-2 training on A100: 3.0× faster. BERT training: 3.3× faster. These speedups come with no change in model output — FlashAttention computes the mathematically identical result.

## Where it breaks

**Custom CUDA kernels are required.** FlashAttention cannot be expressed in standard PyTorch eager mode because it needs explicit control over when data is moved between HBM and SRAM. The implementation is a hand-written CUDA kernel (and later a Triton kernel). This means it is hardware-specific and must be updated for each new GPU generation (H100 needed FA3 to exploit asynchronous memory copy and FP8 pipelines).

**Short sequences see diminishing returns.** The tiling bookkeeping adds overhead; for N < 128 the benefit over standard attention is marginal or negative.

**Causal masking and other variants require extra logic.** Each new attention variant (sliding window, ALiBi, paged KV cache) needs its own kernel. FA1 only supported dense causal and non-causal attention; subsequent versions added more patterns.

**FA2's parallelism strategy doesn't generalize to all hardware.** FA2 splits Q across thread blocks (rather than K/V) to improve GPU occupancy; this was optimal on A100 but required further restructuring for H100's warp-specialized execution model.

**Approximate attention is a different solution.** Linformer, Performer, BigBird, and other sub-quadratic attention approaches reduce the O(N²) FLOP complexity; FlashAttention does not. For N > 100K those methods may still be necessary despite FA's memory efficiency.

## Why it works

FlashAttention is **loop tiling applied to GPU attention**. Loop tiling is a 50-year-old compiler transformation: restructure a doubly-nested loop so that the inner loop accesses a small tile of memory repeatedly from fast cache rather than pulling new data from slow main memory on every iteration. This is exactly what FlashAttention does — the outer loop over K/V blocks corresponds to the cache-resident tile; the inner loop over Q blocks is the fast inner kernel.

The deeper principle is the **roofline model** (Williams et al., 2009): every algorithm lives somewhere on a curve relating arithmetic intensity to achievable throughput. Below the ridge point, adding more compute units does nothing because the memory bus is full. The only way to improve performance in the memory-bound regime is to reduce bytes moved. FlashAttention moved attention from far below the roofline to near it.

The same "minimize memory traffic, not FLOP count" idea appears everywhere in high-performance computing:

- **BLAS/LAPACK**: optimized matrix-multiply (DGEMM) uses tiled loops to keep hot data in L1/L2 cache
- **Cache-oblivious algorithms** (Frigo et al., 1999): recursive divide-and-conquer naturally adapts tiling to any memory tier without knowing cache sizes at compile time
- **External-memory algorithms**: merge sort, B-trees, and LSM-tree compaction all minimize disk I/O by maximizing sequential access and keeping hot data in RAM
- **Streaming algorithms**: online softmax is the attention-specific instance of the general pattern of computing aggregates (max, sum, histogram) in one pass over data using O(1) accumulators

The surprising part of FlashAttention is not the technique — loop tiling is in every HPC textbook — but the application. It was not obvious that attention, framed for years as a quadratic-complexity algorithm, was actually a memory-bound problem whose practical complexity could be reduced by algorithm restructuring without any change in mathematical output.

## Going deeper

1. **FlashAttention-2** (Dao, 2023): https://arxiv.org/abs/2307.08691 — splits Q across thread blocks instead of K/V, halving non-matmul FLOP count and improving occupancy on A100; the version integrated into PyTorch 2.2+ as `torch.nn.functional.scaled_dot_product_attention`.

2. **Online normalizer calculation for softmax** (Milakov & Gimelshein, 2018): https://arxiv.org/abs/1805.02867 — the streaming algorithm that makes tile-by-tile softmax accumulation possible; only three pages and essential background for understanding why online softmax is exact.

3. **Roofline: an insightful visual performance model for multicore architectures** (Williams, Waterman, Patterson, 2009): https://dl.acm.org/doi/10.1145/1498765.1498785 — the framework for diagnosing compute-bound vs. memory-bound regimes; reading this changes how you look at every algorithm's performance profile.

---
title: "FlexAttention: The Flexibility of PyTorch with the Performance of FlashAttention"
source: https://pytorch.org/blog/flexattention/
author: Driss Guessous, Yanbo Liang, Joy Dong, Horace He (Team PyTorch)
company: Meta / PyTorch
date_posted: 2024-08-07
date_digested: 2026-10-28
---

# FlexAttention: The Flexibility of PyTorch with the Performance of FlashAttention

## What's new to learn

1. **`score_mod` as the universal attention API**: Any attention variant can be expressed as a single callable `(score, batch, head, query_idx, key_idx) → modified_score` that specifies what to do to the raw dot-product before the softmax — and this function can be compiled into a fused, IO-aware Triton kernel automatically.

2. **`block_mask` for block-sparse attention**: Structured sparsity patterns (causal, sliding window, document masking) can be expressed as a mask over *(query_tile, key_tile)* pairs computed offline; the kernel skips entire tiles at runtime, reducing FLOPs from O(N²) to O(N × pattern_size) without any custom CUDA.

3. **Algorithm / schedule separation applied to deep learning kernels**: FlexAttention demonstrates that the *what* of a computation (score modification, sparsity pattern) can be cleanly separated from the *how* (IO-aware tiling, SIMD vectorization, online softmax accumulation), with a compiler bridging the gap — the same principle as SQL, Halide, and BLAS, applied to attention.

---

## Prerequisites

- **Standard attention mechanics**: what Q, K, V are; how `softmax(QKᵀ/√dₖ) V` is computed; why the N×N score matrix is an issue at long sequence lengths.
- **FlashAttention's IO-aware insight** (covered 2026-05-26): why materializing the full N×N matrix in GPU HBM is the real bottleneck, and how tiling + online softmax eliminates it.
- **GPU memory hierarchy basics**: the difference between HBM (high-bandwidth memory, ~1–2 TB/s) and on-chip SRAM (~20 TB/s); why arithmetic is cheap but memory moves are expensive on modern GPUs.
- **`torch.compile` basics**: that PyTorch can trace Python functions and emit optimized GPU kernels (Triton or CUDA) from them.

---

## The core idea

Every attention variant ever proposed — causal masking, sliding window, ALiBi positional bias, RoPE, document-level masking, RLHF prefix attention, Longformer, BigBird — modifies the same quantity: the scalar dot-product score `qᵢ · kⱼ` before it enters the softmax. The shape of the output is always the same: a weighted sum of value vectors.

FlexAttention exposes this as a single hook:

```python
def score_mod(score, batch, head, query_idx, key_idx) -> Tensor:
    # score: the raw dot product qᵢ·kⱼ/√dₖ  (scalar)
    # return: the modified score to feed into softmax
    ...
```

Anything you can compute from those five arguments — index offsets, pre-computed bias tables, negated infinities for masking — goes here. You write plain PyTorch; `flex_attention` compiles it.

Combined with a `block_mask` that pre-identifies which *(query_tile, key_tile)* blocks are entirely masked out, the compiled kernel can skip those blocks at no cost.

The result: you write 10 lines of Python and get the same throughput as a hand-tuned FlashAttention2 kernel, for *any* attention variant.

---

## Mechanics

### 1. The `score_mod` function

`score_mod` is called conceptually at the innermost point of the attention kernel: after the dot product is formed for a single (query, key) pair, before the online softmax accumulates it.

```python
# Causal masking
def causal(score, b, h, q, kv):
    return torch.where(q >= kv, score, -float("inf"))

# ALiBi: linear position bias per attention head
def alibi(score, b, h, q, kv):
    bias = -alibi_slopes[h] * (q - kv).abs()
    return score + bias

# Sliding window (only attend to last 128 tokens)
def sliding_window(score, b, h, q, kv):
    return torch.where(q - kv <= 128, score, -float("inf"))

# Document masking: tokens in different docs never attend to each other
def doc_mask(score, b, h, q, kv):
    return torch.where(doc_ids[q] == doc_ids[kv], score, -float("inf"))
```

Any of these plugs directly into:

```python
output = flex_attention(Q, K, V, score_mod=causal)
```

Composition is just function composition — you can chain modifiers in a closure:

```python
def causal_alibi(score, b, h, q, kv):
    return alibi(causal(score, b, h, q, kv), b, h, q, kv)
```

### 2. Compilation path

`torch.compile` traces `score_mod` through AOT Autograd, extracts a functional computation graph, and generates a Triton kernel. The key architectural choice: the score_mod call is **inlined** into FlashAttention's outer tiling loop, not executed as a separate step.

Concretely, the generated kernel structure is:

```
for q_tile in query_tiles:
    online_max, online_sum, acc = -inf, 0, 0
    for kv_tile in key_value_tiles:
        if block_mask[q_tile, kv_tile] == MASKED_OUT:
            continue                        # ← block_mask skips whole tiles
        dots = Q[q_tile] @ K[kv_tile].T    # raw dot products
        for each (qi, kvi) in tile:
            dots[qi, kvi] = score_mod(dots[qi, kvi], b, h, qi, kvi)  # inlined
        online_max, online_sum, acc = update_online_softmax(...)
    output[q_tile] = acc / online_sum
```

The score_mod is arithmetic on a single scalar; it adds negligible FLOPs. The backward pass is auto-generated from the traced graph.

### 3. `block_mask` for block-sparsity

For patterns where large contiguous regions are always masked, `create_block_mask` pre-materializes a compact bitmask at the *tile* level:

```python
def causal_mask_fn(b, h, q_idx, kv_idx):
    return q_idx >= kv_idx                  # True = keep, False = mask

block_mask = create_block_mask(causal_mask_fn, B=1, H=1, SEQ_LEN=4096, TILE=128)
# block_mask is a [32, 32] bitmask (4096/128 tiles per axis)
# Lower-triangular blocks are kept; upper-triangular are skipped entirely
```

At runtime the kernel checks this bitmask before entering each tile, skipping tiles that are entirely masked. The result:
- Causal attention: ~½ of tiles skipped → ~½ the FLOPs and memory reads
- Sliding window (window=128 on seq_len=4096): ~97% tiles skipped → O(N) work
- Document masking: block-diagonal structure → each doc only reads its own KV tiles

`create_block_mask` calls the mask function only at tile granularity, not per element. Elements within a partly-masked tile are handled by `score_mod` setting scores to `-inf`.

### 4. Attention variants expressed concisely

Here is what the entire zoo of attention variants looks like in FlexAttention:

| Variant | Code |
|---|---|
| Causal | `return -inf if q < kv` |
| Causal + sliding window | `return -inf if q < kv or q-kv > W` |
| ALiBi | `return score - slope[h] * (q-kv)` |
| RoPE (score bias) | Handled in Q/K preprocessing (RoPE modifies Q,K before attention) |
| Document masking | `return -inf if doc_id[q] != doc_id[kv]` |
| Prefix attention (shared prefix, causal suffix) | `return -inf if q >= prefix_len and q < kv` |
| PagedAttention | Uses block_mask to skip non-contiguous physical KV pages |

### 5. Throughput numbers

From the blog post benchmarks on A100:
- Standard causal attention: ~95–99% of hand-tuned FlashAttention2 FLOP/s
- Sliding window (window=128, seq=2048): essentially identical to hand-tuned kernel
- ALiBi: ~95% of a specialized ALiBi kernel written in CUDA
- Writing any variant takes 10–15 lines of Python vs. 200–1000 lines of custom Triton/CUDA

---

## Where it breaks

**score_mod must be pure and tensor-friendly.** The function runs inside a vectorized Triton kernel. Data-dependent Python control flow based on the actual score values is not allowed; only pointwise arithmetic and index comparisons work. Anything that requires global state (e.g., "mask based on the top-k scores in this row") cannot be expressed.

**Non-standard QKV shapes need adaption.** GQA (grouped query attention) and MLA (multi-head latent attention, DeepSeek) compute K and V differently than standard attention — FlexAttention handles these through torch.compile's shape inference but the user must set up the Q/K/V tensors correctly beforehand.

**block_mask only helps with position-dependent sparsity.** If the sparsity pattern depends on the *content* of tokens (e.g., "attend only to tokens with high cosine similarity"), you can't express it in a pre-computed block_mask. You'd need score_mod to set many scores to -inf, but there's no block-level skip.

**Inference decode requires a different backend.** The original FlexAttention targets training and prefill (long queries). During autoregressive decode, the query length is 1, and a separate FlexDecoding backend is needed (covered in the follow-up post). The compilation path is different.

**compile overhead on first call.** Like all torch.compile usage, the first invocation triggers tracing and kernel generation. This adds latency not seen in hand-written CUDA kernels.

---

## Why it works

FlexAttention is an instance of the **Halide principle**: separate the *algorithm* (what values to compute and from what inputs) from the *schedule* (how to tile, parallelize, and move data).

In Halide, you write an image processing pipeline as a pure functional program (e.g., "output[x,y] = 0.25 × (input[x-1,y] + input[x+1,y] + input[x,y-1] + input[x,y+1])"). The Halide scheduler then decides how to tile, vectorize, parallelize, and fuse this into efficient machine code. The programmer controls *neither* — she only controls the split between "what" and "how" by choosing the schedule separately.

FlexAttention works the same way:
- **Algorithm (what)**: score_mod defines the mathematical specification of the attention score transformation; block_mask specifies which (q_tile, kv_tile) blocks are empty.
- **Schedule (how)**: torch.compile + Triton handles IO-aware tiling, online softmax accumulation, SRAM register allocation, memory coalescing.

The programmer writes the algorithm; the compiler writes the schedule.

This pattern appears throughout the best systems in this archive:

| System | Algorithm ("what") | Schedule ("how") |
|---|---|---|
| FlexAttention | `score_mod` Python function | torch.compile → Triton kernel with FlashAttention tiling |
| SQL | Declarative query | Optimizer-chosen join order, index scans, hash vs. merge join |
| BLAS `sgemm` | Matrix multiply C = αAB + βC | SIMD blocking, cache tiling, prefetch |
| Copy-and-Patch JIT | Stencil per opcode (what the opcode computes) | Hole-fill at JIT time (operand binding, register assignment) |
| MonetDB/X100 | Relational algebra expression | Vectorized tile execution, SIMD within tile |
| FlashAttention itself | Exact attention (full QKV attention) | Tile-level IO optimization (avoid materializing N×N matrix) |

The specific reason FlexAttention works so well is the same reason FlashAttention works at all: **attention is not compute-bound, it is memory-bound**. The FLOPs added by a score_mod (a few arithmetic operations per score) are negligible compared to the HBM reads/writes that dominate attention's runtime. So inlining score_mod into the kernel adds no material overhead — the kernel's cost is still dominated by the I/O bound, not by the extra arithmetic.

Generalized: whenever your custom operation adds **FLOP-cheap, IO-location-unchanged** computation at the innermost loop of an existing IO-optimal kernel, you can inline it for free. This is why fused kernel patterns (fused attention + rotary, fused layer-norm + linear) consistently win over chained separate kernels.

---

## Going deeper

1. **FlexAttention arXiv paper** — "Flex Attention: A Programming Model for Generating Optimized Attention Kernels" (He et al., Dec 2024): https://arxiv.org/abs/2412.05496 — the companion paper with formal analysis and ablations.

2. **Halide: A Language and Compiler for Optimizing Parallelism, Locality, and Recomputation in Image Processing Pipelines** (Ragan-Kelley et al., PLDI 2013): https://halide-lang.org/papers/halide_pldi13.pdf — the original paper for the algorithm/schedule separation principle FlexAttention applies.

3. **FlashAttention paper** (Dao et al., 2022): https://arxiv.org/abs/2205.14135 — the underlying IO-aware tiling kernel that FlexAttention builds on. Reading this first makes the FlexAttention design obvious.

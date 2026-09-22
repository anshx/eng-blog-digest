---
title: "DeepSeek-V2: Multi-Head Latent Attention"
source: https://arxiv.org/abs/2405.04434
author: DeepSeek-AI
company: DeepSeek
date_posted: 2024-05-06
date_digested: 2026-09-22
---

# DeepSeek-V2: Multi-Head Latent Attention

## What's new to learn

1. **Low-rank KV compression**: the KV cache memory bottleneck can be broken by projecting keys and values to a shared low-dimensional latent vector before caching — 512 floats instead of 40,960 per token per layer — then expanding back at attention time. This is LoRA's insight (updates live on a low-rank manifold) applied to *inference-time caching* rather than fine-tuning.

2. **Matrix absorption / operator folding**: when K = W_UK × c and Q = W_UQ × q_eff, the attention score Q^T K = q_eff^T (W_UQ^T W_UK) c can be computed entirely in the compressed space. The product W_UQ^T W_UK is weight-constant, so it is precomputed once offline and folded into Q's projection. Neither K nor the expanded Q is ever materialised at inference time.

3. **Decoupled RoPE**: rotary position embeddings cannot survive compression (the rotation angle depends on the token's position in the sequence — different at every step — so it cannot be baked into a shared latent). MLA handles this by splitting each head into a content part (compressible) and a small positional part (stored separately as a 64-dim shared key), then recombining for the attention score.

## Prerequisites

- Standard scaled dot-product attention: Q, K, V projection matrices; the softmax(QK^T/√d) V formula.
- **KV cache mechanics**: at each autoregressive decode step the model must attend over all past positions. Rather than recompute K and V for every past token, they are cached. For a 128-head, 128-dim-per-head model, each token occupies 128 × 128 × 2 (K+V) = 32 768 bfloat16 values per layer — roughly 64 KB per layer before adding positional dimensions.
- **Grouped-Query Attention** (archive 2026-08-19): sharing K/V heads across groups of query heads reduces the *count* of heads. MLA takes the orthogonal axis: fewer dimensions per cached representation, not fewer heads.
- **LoRA** (archive 2026-08-06): the claim that rank-r matrices suffice for fine-tuning updates motivates the analogous claim here that rank-512 suffices for the KV content.
- **RoPE** (archive 2026-09-21): rotating Q and K vectors by position-proportional angles encodes relative distance in the dot product.

## The core idea

GQA reduces KV memory by grouping: instead of 128 independent K/V sets, make 8 groups share a set — 16× fewer heads. The problem is that the 128 query heads that share one K/V head lose independent content; you pay in model quality.

MLA takes a fundamentally different axis. It keeps all 128 query heads independent, but *compresses the content that feeds all their keys and values* into a single 512-dimensional latent vector `c_KV`. Both K and V are linear functions of this same latent. At caching time you store only `c_KV` (and a tiny positional addendum); at attention time you expand back.

The memory saving is dramatic. For DeepSeek-V2:

| Representation | Floats per token per layer | Bytes (bf16) |
|---|---|---|
| Standard MHA (content+rope) | 40 960 | ~80 KB |
| MLA (c_KV + k_rope) | 576 | ~1.1 KB |
| **Ratio** | **71×** | **71×** |

And MLA does not pay in model quality: the DeepSeek-V2 ablations show MLA *outperforms* standard MHA, because the learned compression acts as a bottleneck that regularises the attention manifold.

The real trick is that you never actually expand `c_KV` to compute attention scores. The expansion matrix is folded into the query projection offline, so the hot path at inference time works entirely in 512-dimensional compressed space. This is the matrix absorption idea.

## Mechanics

### Training-time forward pass

Given hidden state **h** ∈ ℝ^{d_model} at position t:

**Query path (not cached):**
```
q_eff = W_DQ · h           # down-project to 1 536 dims
Q     = W_UQ · q_eff       # up-project to 128 heads × 128 dims
Q_r   = W_QR · h           # separate 128 heads × 64-dim RoPE part
Q_r   = apply_rope(Q_r, t)
Q_full = concat(Q, Q_r)    # 128 × (128+64) = 128 × 192
```

**KV path (cached):**
```
c_KV  = W_DKV · h          # down-project to 512 dims ← CACHE THIS
K     = W_UK · c_KV        # up-project: 128 heads × 128 dims
V     = W_UV · c_KV        # same
k_r   = W_KR · h           # shared RoPE key: 64 dims ← ALSO CACHE
k_r   = apply_rope(k_r, t)
K_full = concat(K, broadcast(k_r, 128 heads))
```

**Attention:**
```
scores = Q_full · K_full^T / sqrt(192)   # (128 heads × seq_len)
output = softmax(scores) · V             # (128 heads × 128)
```

### Inference-time absorption

At inference time you never call `W_UK · c_KV`. Instead:

```
# Precomputed once per layer (offline):
W_absorbed_K = W_UQ^T · W_UK    # shape: 1536 × 512
W_absorbed_V = W_UV              # shape: 128*128 × 512 (re-used as-is)

# Hot path for each new query at position i:
q_eff_i = W_DQ · h_i             # 1536-dim
# Score against cached c_KV_j at position j:
score_ij = q_eff_i^T · W_absorbed_K · c_KV_j  # bilinear form, no K expanded

# Output: attn-weighted sum of compressed V, then up-project
attn_v = sum_j(softmax_score_j · c_KV_j)       # 512-dim weighted sum
output = W_UV · attn_v                           # expand once
```

The insight: since K is a fixed linear function of c_KV, and Q is a fixed linear function of q_eff, Q^T K is a fixed bilinear form Q_eff^T (W_UQ^T W_UK) c_KV. All three of those matrices are constant — baked into the weights — so you can merge W_UQ^T W_UK into a single precomputed matrix. Neither the 128-head K nor the expanded Q ever exists in memory at inference time.

Similarly for the output: instead of materialising 128×128-dim V and then taking the attention-weighted sum, you accumulate the weighted sum in 512-dim compressed space, then apply W_UV once at the end.

### Decoupled RoPE: why and how

RoPE breaks absorption. The rotation matrix R(t) for position t is different for every t, so you cannot fold it into the constant W_absorbed_K. If you compress *after* rotating, each token's c_KV encodes a rotated vector — and the rotation cannot be factored out.

MLA's solution: reserve a small subspace (64 dimensions per query head, 64 dimensions for a *shared* key) that carries RoPE, and keep it *outside* the compressed latent.

At cache time, store both `c_KV` (content, 512-dim) and `k_rope` (position, 64-dim shared). The attention score splits:

```
score_ij = (content part via absorption) + (RoPE part: Q_r_i^T · k_r_j)
```

The content part benefits from full absorption. The positional part uses a conventional RoPE inner product between per-head query rotations and the single shared rotated key. The two are added before softmax.

Cost of the decoupled cache: 64 extra bfloat16 floats per token per layer — 128 bytes, a small constant overhead on top of the 1 024-byte content latent.

### Configuration numbers (DeepSeek-V2)

| Parameter | Value |
|---|---|
| Total parameters | 236 B |
| Activated per token | 21 B (MoE) |
| Attention heads (n_h) | 128 |
| Content head dim (d_h) | 128 |
| RoPE head dim (d_r) | 64 |
| KV latent dim (d_c) | 512 |
| Q latent dim (d_c') | 1 536 |
| KV cache per token per layer | ~1.1 KB |
| Speedup vs. DeepSeek 67B (throughput) | 5.76× |

## Where it breaks

**Extra FLOPs at every forward pass.** Each token pays two extra matrix multiplications (the DQ down-projection and the absorbed K inner product). For prefill — where you process a large prompt in one shot — this is measurable overhead compared to vanilla MHA.

**Absorbed weight matrices are large.** W_UQ^T W_UK has shape [1 536 × 512] per layer. Across 60 layers, that is 47 million extra float32 parameters loaded into registers. For a model serving millions of requests, these extra parameter loads add up in the kernel launch overhead.

**Quantisation asymmetry.** Standard KV-cache quantisation targets the cached K/V tensors. For MLA the "cached tensor" is c_KV (512-dim), and the "computed-from-cache tensor" is the bilinear result, which has different numerical properties. FlashMLA handles this by using FP8 for the KV cache while performing the absorbed matrix multiply in bf16 — requiring specialised kernels.

**No simple tensor-parallelism split.** In standard MHA you split heads across GPUs: device i handles heads i·(n_h/n_gpu) to (i+1)·(n_h/n_gpu). In MLA, all heads share the same c_KV, so you must either replicate c_KV on every device (communication) or restructure the absorbed multiply accordingly. This is solvable but requires non-trivial kernel engineering.

**Decoupled RoPE is a design tax.** Adding a separate positional pathway means two separate matrix projections and two attention score components per layer. If a future model abandoned RoPE in favour of a compression-friendly positional scheme, the decoupling complexity would disappear.

## Why it works

The core mechanism is **offline-online separation via operator folding** — a pattern that recurs across systems:

- **LoRA merging** (archive): after training, fold ΔW = BA back into W. At inference you pay no extra cost; the rank-r structure is entirely in the weights.
- **Maglev consistent hashing** (archive): precompute the full permutation table offline; at runtime it is a single array lookup.
- **Copy-and-Patch JIT** (archive): pre-compile opcode stencils offline; at runtime fill holes with literal operands (no codegen overhead on the hot path).
- **SQL prepared statements**: parse/plan once, execute many times.

In all these cases, the *constant part* of a computation is moved out of the inner loop. In MLA, the constant part is the linear maps W_UK and W_UV. Because K = W_UK · c and V = W_UV · c, the maps can be absorbed into Q and into the output projection respectively. Only the *variable part* — the per-position latent c_KV — stays on the hot path.

The second insight is **intrinsic low rank**: K and V projections across 128 independent heads are not actually 128-way independent. Different heads attend to structurally similar patterns (syntactic heads, coreference heads, positional heads); the 512-dimensional latent has enough capacity to represent all those patterns simultaneously. This is the same empirical observation that makes LoRA work: the weight space that a training process explores has much lower intrinsic dimension than the nominal matrix dimension.

A corollary is that *compression is regularisation*. By forcing all 128 K/V sets through a 512-dimensional bottleneck, MLA discourages heads from independently memorising the same content. This is why the ablations show MLA *better* than MHA: the compression acts like a shared-representation constraint that encourages heads to diversify.

The positional-content split in Decoupled RoPE reflects a deeper decomposition principle: identify the part of your signal that is **position-invariant** (what a token says, regardless of where it appears) versus **position-variant** (where it appears relative to the query). Content is compressible across heads; position is not, because every position is unique. This is the same separation underlying the locality hypothesis (archive 2026-07-30) and the reason HNSW (archive 2026-05-21) stores graph edges separately from vector content.

## Going deeper

1. **FlashMLA** — DeepSeek's open-source CUDA kernels for efficient MLA decoding (https://github.com/deepseek-ai/FlashMLA). Shows how the absorbed matrix multiply is fused with the RoPE step and KV cache reads into a single 660 TFLOPS kernel on H800 GPUs — the engineering complement to the mathematical idea here.

2. **"Palu: Compressing KV-Cache with Low-Rank Projection"** (arXiv:2407.21118, 2024). Applies the same low-rank insight *post-hoc* to already-trained models — without the absorption trick, using SVD decomposition of existing K/V weight matrices. Useful contrast: MLA learns the low-rank structure from scratch; Palu retrofits it.

3. **"Towards Economical Inference: Enabling DeepSeek's Multi-Head Latent Attention in Any Transformer-based LLMs"** (arXiv:2502.14837, 2025). Shows how to retrain a GQA-based model (e.g., Llama 3) to use MLA via a two-stage fine-tuning recipe, with quality recovery at 500 B tokens. Useful for understanding which parts of MLA require training from scratch vs. which can be retrofitted.

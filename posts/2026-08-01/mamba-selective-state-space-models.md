---
title: "Mamba: Linear-Time Sequence Modeling with Selective State Spaces"
source: https://arxiv.org/abs/2312.00752
author: Albert Gu, Tri Dao
company: Carnegie Mellon University / Princeton University
date_posted: 2023-12-01
date_digested: 2026-08-01
---

# Mamba: Linear-Time Sequence Modeling with Selective State Spaces

## What's new to learn

1. **Selective State Space Models**: A recurrent sequence model where the transition parameters are conditioned on the current input token, giving the model content-aware memory. Unlike fixed SSMs (same dynamics for every token) or attention (explicitly attends to all past tokens), a selective SSM can choose *how strongly* each token updates — or is later readable from — the compressed state.

2. **Parallel associative scan for variable-coefficient linear recurrences**: When SSM parameters are input-dependent, you can no longer precompute a fixed convolution kernel and apply FFT. But the recurrence is still *linear*, so its per-step (Ā, B̄x) pairs compose associatively. Blelloch's parallel prefix-scan algorithm — the same O(log L) depth primitive used for GPU prefix sums — computes all L hidden states in parallel during training.

3. **Δ as a learned per-token forget gate**: The discretization step size Δ_t controls the integration window. Large Δ → Ā_t = exp(ΔA) ≈ 0, resetting the state. Small Δ → Ā_t ≈ I, passing state through. Making Δ an input-dependent projection turns this into a learned, content-driven forget gate — mathematically equivalent to an LSTM cell gate but arising from first principles of signal discretization.

## Prerequisites

- **Recurrent neural networks**: hidden state, why sequential RNNs are hard to parallelize
- **LSTM gating**: cell state, input/forget/output gates; how learned gates address the vanishing gradient problem
- **Transformer attention**: Q/K/V matrices, scaled dot-product attention, why compute is O(L²) in sequence length
- **Parallel prefix scan** (see the July 15 archive entry): Blelloch's upsweep/downsweep algorithm and the key insight that any associative binary operator can be prefix-scanned in O(log L) depth — this is the exact algorithm Mamba reuses for its selective scan

## The core idea

Transformers are powerful because every output token can attend directly to every past input token — but this costs O(L²) time and memory. RNNs solve this by maintaining a fixed-size hidden state h_t that compresses all past tokens, giving O(1) memory per step. The compression is the problem: because the dynamics (the matrices A, B, C) are fixed after training, every token goes through the same "filter," and the model cannot selectively retain some inputs while discarding others based on content.

Mamba makes the SSM **selective**: learned projections of the current token x_t dynamically determine how to update the state (B_t), how to read from it (C_t), and how aggressively to forget the past (Δ_t). The model can now decide, token by token, whether to write this input into long-term memory or let the state coast. It passes a "selective copy" benchmark — repeat the flagged tokens, skip the rest — at 99.8% accuracy where the best prior SSMs achieved 57%.

The training complication is that input-dependent parameters break the convolution shortcut. Fixed SSMs compute training in O(L log L) with FFT. Mamba instead exploits that `h_t = Ā_t h_{t-1} + B̄_t x_t` has an **associative** composition structure and applies the same parallel prefix scan that computes GPU cumulative sums — getting O(log L) depth with O(L) total work.

At inference, the model reverts to a pure recurrence: update one N-dimensional state vector per token. No key/value cache that grows with context — just one fixed-size state, regardless of sequence length.

## Mechanics

### Step 1: The SSM framework

A state space model starts from a continuous-time linear system:

```
h'(t) = A h(t) + B x(t)
y(t)  = C h(t)
```

where h(t) ∈ ℝᴺ is the hidden state, A ∈ ℝᴺˣᴺ is the state transition, B ∈ ℝᴺˣ¹ projects the scalar input in, and C ∈ ℝ¹ˣᴺ reads the output out. N is the "state size" — typically 16 in Mamba.

For discrete tokens, this is **discretized** with step size Δ (using the zero-order hold or bilinear method):

```
Ā = exp(ΔA)
B̄ = (ΔA)⁻¹ (Ā − I) ΔB

h_t = Ā h_{t-1} + B̄ x_t
y_t = C h_t
```

**The convolutional shortcut for fixed SSMs**: If Ā and B̄ don't depend on t (i.e., A, B, C, Δ are fixed parameters), the recurrence unrolls to a convolution: `y = x ★ K` where the kernel is `K = (CB̄, CĀB̄, CĀ²B̄, ...)`. You can compute this with FFT in O(L log L) — as fast and parallelizable as any sliding-window operation.

**HiPPO initialization**: The A matrix is initialized as the "HiPPO matrix," which projects any continuous signal onto N Legendre polynomial basis functions. This gives S4-style SSMs provably optimal polynomial approximations to their input history. The state h_t literally represents the N-coefficient polynomial fit to all past inputs.

### Step 2: Why fixed SSMs fail at content-dependent tasks

With constant Ā, B̄, the model applies the same linear filter to every token. Consider the **selective copy** task: given input `A B _ _ C D _ _`, where underscores are filler, produce `A _ _ _ C _ _ _` (copy token iff it's not filler, based on a binary flag). A fixed filter cannot distinguish "copy" tokens from filler — it would need to suppress filler uniformly, but the same filter applies everywhere. Empirically, S4 and similar LTI-SSMs scored near-chance.

Attention handles this trivially: a sharp softmax over position flags picks exactly the tokens to attend to. The challenge is building SSM-like linear-time inference while gaining this selection ability.

### Step 3: The Mamba fix — making B, C, Δ input-dependent

Mamba parameterizes three SSM quantities as linear projections of the current input x_t ∈ ℝᴰ:

```
Δ_t = softplus(Linear_Δ(x_t))    ∈ ℝᴰ   (one per feature dim)
B_t = Linear_B(x_t)              ∈ ℝᴺ
C_t = Linear_C(x_t)              ∈ ℝᴺ
```

Then `Ā_t = exp(Δ_t ⊙ A)` and `B̄_t = diag(Δ_t) · B_t` (broadcast over N). The A matrix itself remains a fixed learnable parameter.

**What Δ does**: When Δ_t is large for some feature dimensions, Ā_t ≈ 0 in those dimensions — the new input B̄_t x_t completely overwrites the old state. When Δ_t is small, Ā_t ≈ I — the state barely changes. This is a soft, per-feature, per-token forget gate: the model learns to write incoming information aggressively or ignore it, based on content.

**What B, C do**: B_t controls the direction in state space that x_t writes into. C_t controls what direction to read out when forming y_t. Both vary per token, giving content-dependent write and read heads.

### Step 4: The parallel scan algorithm

With input-dependent Ā_t, B̄_t, the naive approach is sequential:

```
h_1 = Ā_1 h_0 + B̄_1 x_1
h_2 = Ā_2 h_1 + B̄_2 x_2
...
h_L = Ā_L h_{L-1} + B̄_L x_L
```

This is O(L) sequential steps — unparallelizable as written. But the recurrence is **linear**, so it has an associative structure. Define a composite element as a pair (a, b) where a is the "multiplicative coefficient" and b is the "additive offset":

```
(Ā_2, B̄_2 x_2) ⊗ (Ā_1, B̄_1 x_1) = (Ā_2 Ā_1,  Ā_2 B̄_1 x_1 + B̄_2 x_2)
```

Interpretation: "apply the recurrence of step 1 first, then step 2" = multiply the multipliers and accumulate the offsets. This operator is **associative**: `(p ⊗ q) ⊗ r = p ⊗ (q ⊗ r)` (matrix multiply is associative; offset accumulation composes the same way).

Since the operator is associative, Blelloch's parallel prefix scan applies directly:

1. **Upsweep (reduce)**: pair up adjacent (Ā, B̄x) elements and combine them, doubling stride each round. L elements → L/2 → L/4 → ... → 1 root. O(log L) rounds.
2. **Downsweep (distribute)**: propagate partial prefix products back down the tree. O(log L) rounds.
3. Result: for every position i, the output contains the composed operator for all positions 1..i, which is exactly h_i.

This is **the same algorithm as the GPU prefix sum in the July 15 archive entry** — the operator is affine composition `(a, b) ⊗ (c, d) = (ac, ad + b)` instead of addition `a + b`, but the tree structure and parallel complexity are identical. O(log L) depth, O(L) total work.

### Step 5: Hardware-aware kernel fusion

The scan produces intermediate (Ā, b̄) tuples at every tree level. Naively, materializing these in GPU HBM (main memory) would require O(B × L × D × N) bytes — for B=8, L=2048, D=2560, N=16, that's about 2GB just for intermediate scan states on a single batch.

Mamba uses the same kernel-fusion trick as FlashAttention:

- Compute Ā_t, B̄_t from x_t **inside SRAM** (fast on-chip GPU cache, ~20MB on A100)
- Run the entire scan pass without writing intermediate h_t states to HBM
- During backward pass, **recompute** the forward scan from checkpointed x_t rather than storing all h_t

This trades ~33% extra compute (one extra forward pass's worth of scan work) for a proportional reduction in HBM traffic — the bottleneck for long sequences. The custom CUDA kernel implementing this is ~2000 lines, with careful attention to thread coarsening, warp-level shuffles, and register pressure.

### Step 6: The Mamba block

Each Mamba layer wraps the selective SSM in a gated residual block:

```
input x ∈ ℝᴮˣᴸˣᴰ
  │
  ├─ Linear (expand to 2ED) ─── split ──► z ∈ ℝᴮˣᴸˣᴱᴰ
  │
  └─ Linear (expand to ED)
        │
        ├─ DepthwiseConv1d (kernel 4, local context)
        │
        ├─ SiLU
        │
        └─ Selective SSM
              │
              ▼
              y ∈ ℝᴮˣᴸˣᴱᴰ
              │
              × SiLU(z)   ← multiplicative gate (like SwiGLU)
              │
              └─ Linear (project back to D)
```

The expand factor E = 2 means each Mamba block internally processes at twice the model dimension, similar to the FFN expansion in transformers. The depthwise 1D convolution before the SSM provides local mixing (short context window of 4 tokens) before the SSM handles long-range dependencies — a local-then-global two-stage architecture.

### Results

Trained on The Pile (300B tokens), Mamba at 2.8B parameters matches a similarly sized Transformer in perplexity on standard NLP benchmarks. At inference:

- **State size**: N × D = constant per layer, independent of sequence length. No KV cache growth.
- **Throughput**: ~5× higher generation throughput vs. a similarly-sized Transformer at 2.8B scale, measured as tokens/second on a single A100.
- **Long-sequence tasks**: On pathologies like long DNA sequences (1M+ tokens), Mamba outperforms Transformers trained at the same compute budget, because the quadratic attention cost prohibits Transformer training at those lengths.

## Where it breaks

**Exact recall degrades over long context**: Mamba's state is a fixed-N vector regardless of context length. Attention stores every past token explicitly and retrieves any of them with a sharp softmax. On RULER and similar long-context recall benchmarks, Mamba at 2.8B shows measurably worse recall accuracy than comparably-sized Transformers once context exceeds a few thousand tokens, because the needed token's contribution may have been partially overwritten by Δ-gating.

**Poor tensor-core utilization**: The scan kernel operates on O(N) = O(16) dimensional elements with element-wise operations — far from the large matrix multiplications that A100/H100 tensor cores accelerate best. Measured hardware utilization is 10–15% of theoretical peak FLOPs vs. 80–90% for Transformer attention. This narrows Mamba's real-world throughput advantage to regimes where KV-cache bandwidth, not compute, is the bottleneck.

**Implementation complexity**: The hardware-aware scan kernel is intricate CUDA. The original codebase took months to stabilize. Debugging failures in a fused kernel that skips HBM materialization is significantly harder than debugging standard matrix operations.

**In-context learning**: Transformers exhibit remarkable ability to learn from in-context examples (few-shot learning). Mamba's in-context learning on tasks that require faithfully copying or manipulating input examples is weaker, likely because state compression loses exact copy fidelity. Many tasks studied are unaffected, but this is a regime to watch.

**Hybrid reality**: The industry response has been hybrid models (Jamba, Zamba, MambaFormer) that alternate Mamba and attention layers — Mamba for cheap bulk processing, attention for precise retrieval. This is pragmatic but signals Mamba is not yet a complete replacement.

## Why it works

**The unifying principle: any linear recurrence is a parallel prefix scan.** The recurrence `h_t = a_t h_{t-1} + b_t` is not inherently sequential — it is sequential only if you insist on computing left to right. The associativity of `(a, b) ⊗ (c, d) = (ac, ad+b)` means you can parenthesize the entire sequence in any tree structure. Blelloch's binary tree gives O(log L) depth.

This is the same algebraic insight across the archive:
- **GPU prefix sum** (July 15): addition is associative → O(log n) depth parallel scan
- **Ring AllReduce** (June 19): gradient sum is associative → pipeline over ring topology
- **MapReduce** (June 25): commutative monoid aggregations → scatter/gather over shards
- **Parallel scan** (Blelloch, 1990): any monoid/semigroup operation → prefix scan

Mamba's contribution is recognizing that the *variable-coefficient* recurrence — made variable by input-dependent Δ, B, C — still fits this mold because affine composition `(a, b) ⊗ (c, d) = (ac, ad+b)` is a semigroup. The selectivity breaks the LTI shortcut (FFT convolution) but not the algebraic structure (associativity).

The HiPPO connection reveals *what the state means*: it is the N-dimensional coefficient vector of a Legendre polynomial approximation to the entire input history. Selective gating (large Δ = reset; small Δ = preserve) adjusts the "bandwidth" of this polynomial approximation at each step. The model is learning to do adaptive compressed sensing on its own input stream — deciding, token by token, how much to update the polynomial fit.

The parallel in memory: Mamba's fixed-size state is like a lossy video codec. The SSM is a B-frame encoder: it stores only what changed (determined by Δ-gating). Attention is like raw uncompressed video: every frame stored, lookable by content. For most natural language the compression is fine. For byte-for-byte recall the lossless format wins.

## Going deeper

1. **"The Annotated S4"** — Sasha Rush and Albert Gu (https://srush.github.io/annotated-s4/): A literate Python/Jupyter notebook implementing S4 from scratch — deriving the HiPPO matrix, the convolutional view of fixed SSMs, and the efficient scan. Essential pre-reading before Mamba; it builds the entire SSM framework line by line.

2. **"Transformers are SSMs: Generalized Models and Efficient Algorithms Through Structured State Space Duality" (Mamba-2)** — Dao and Gu, 2024 (https://arxiv.org/abs/2405.21060): Proves that attention (at certain head sizes) and SSMs compute the same mathematical object from opposite directions — quadratic over sequence vs. linear recurrence. Mamba-2 exploits this duality to use tensor-core-friendly matrix multiplications, closing much of the hardware utilization gap.

3. **"RWKV: Reinventing RNNs for the Transformer Era"** — Peng et al., 2023 (https://arxiv.org/abs/2305.13048): An independent parallel attempt at linear-complexity sequence models using a different mathematical structure: time-mixing via exponential decay and a token-shift mechanism. Comparing Mamba's selective-scan design to RWKV's linear attention reformulation illuminates the design space of sub-quadratic sequence models.

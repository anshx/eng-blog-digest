---
title: "You Could Have Designed State of the Art Positional Encoding"
source: https://huggingface.co/blog/designing-positional-encoding
author: FL33TW00D-HF
company: HuggingFace
date_posted: 2024-11-25
date_digested: 2026-09-21
---

# You Could Have Designed State of the Art Positional Encoding

## What's new to learn

- **Permutation-equivariance as the root problem**: Self-attention computes a weighted sum of values where weights depend only on query-key dot products — there is no mechanism that distinguishes "dog at position 2" from "dog at position 8". Fixing this requires injecting position *before* the attention dot product.

- **Rotary Position Embedding (RoPE)**: Instead of *adding* position information to token embeddings, RoPE *rotates* query and key vectors by an angle proportional to their position. The inner product of two rotated vectors then depends only on their relative distance, not their absolute coordinates.

- **Geometric-frequency decomposition**: RoPE partitions the embedding into d/2 independent two-dimensional subspaces and rotates each at a different frequency. This creates a multi-scale positional signal — fast-rotating planes encode fine-grained local order, slow-rotating planes encode coarse long-range structure — with the same frequency design as the original sinusoidal position encoding from "Attention Is All You Need."

---

## Prerequisites

- **Scaled dot-product attention**: `Attention(Q, K, V) = softmax(QKᵀ/√d)V`. Know that the weight matrix comes from dot products of query and key vectors.
- **2D rotation matrices**: The matrix `[[cos θ, −sin θ], [sin θ, cos θ]]` rotates a 2D vector by angle θ. Complex multiplication achieves the same thing: multiplying complex number *z* by *e^(iθ)* rotates it by θ.
- **Why permutation equivariance matters**: If you permute the input token sequence before computing attention, you get the exact same output, just permuted. Position encoding breaks this symmetry.

Non-obvious: you do not need group theory or functional analysis, but the post is much richer if you recognize that "dot product preserved under common rotation" is the key geometric fact being exploited.

---

## The core idea

Self-attention is blind to token order. To fix this we need to modify the computation so that the attention weight between query *q* at position *m* and key *k* at position *n* becomes a function of *q*, *k*, and *m−n* only — not of *m* and *n* individually. Write this as the constraint:

```
⟨f(q, m), f(k, n)⟩ = g(q, k, m − n)
```

where *f* is whatever we do to inject position into the vectors before taking the dot product.

**Why subtraction, not absolute position?** Language is mostly locally relative. "The dog bit the man who trained it" — "it" refers back relative to where it is, not to the fact that it is token 8 out of 12.

**What mapping *f* satisfies this?** Rotate. In 2D, represent *q = q₁ + iq₂* as a complex number. Define:

```
f(q, m) = q · e^(imθ)
```

The inner product between two such encoded vectors (taking the real part of the complex dot product) is:

```
Re(f(q, m) · conj(f(k, n)))
  = Re(q · e^(imθ) · conj(k · e^(inθ)))
  = Re(q · k̄ · e^(i(m−n)θ))
```

The absolute positions *m* and *n* have collapsed into their *difference m−n*. The constraint is satisfied exactly.

For real-valued attention heads (which is what transformers use), this extends directly: partition the *d*-dimensional head into *d/2* pairs, treat each pair as a complex number, and rotate pair *i* at its own frequency θᵢ. The full dot product is a sum of *d/2* relative-position-dependent terms, one per frequency.

**Why rotate Q and K but not V?** The attention weight (`softmax(QKᵀ/√d)`) is computed from dot products of Q and K — that is where positional information enters. V is what gets retrieved after the softmax; rotating V would only add noise.

---

## Mechanics

### Frequency schedule

The *d/2* per-pair frequencies follow the same geometric sequence as the original sinusoidal encoding from Vaswani et al.:

```
θᵢ = base^(−2i / d),    i = 0, 1, …, d/2 − 1
```

With `base = 10,000` (the default in LLaMA-1/2):
- Pair 0 rotates at θ₀ = 1 radian per token step — one full revolution every 6 tokens.
- Pair d/2−1 rotates at θ_{d/2-1} ≈ 10⁻⁴ radians per step — one full revolution every ~62,000 tokens.

High-frequency pairs act as a fine-grained "local order" sensor; low-frequency pairs accumulate slowly enough to distinguish tokens far apart.

### Rotation matrix (full-dimensional)

Arrange the *d* components of query *q* as `[q₀, q₁, q₂, q₃, …, q_{d-2}, q_{d-1}]`. Pair them as `(q₀, q₁), (q₂, q₃), …`. Define the *d×d* block-diagonal rotation matrix for position *m*:

```
R(m) = block-diag(
  [[cos(m·θ₀),  −sin(m·θ₀)],
   [sin(m·θ₀),   cos(m·θ₀)]],

  [[cos(m·θ₁),  −sin(m·θ₁)],
   [sin(m·θ₁),   cos(m·θ₁)]],
  …
)
```

The rotated query is simply `R(m) · q`. The attention dot product becomes:

```
(R(m) · q)ᵀ (R(n) · k)
= qᵀ R(m)ᵀ R(n) k
= qᵀ R(n − m) k
```

because `R(m)ᵀ R(n) = R(n−m)` — the composition of two rotations subtracts angles. This is the geometric fact that makes RoPE work.

### Efficient implementation (no matrix multiply needed)

The rotation does not require materializing the full *d×d* block-diagonal matrix. For each pair `(q_{2i}, q_{2i+1})`:

```python
cos_m = cos(m * theta[i])
sin_m = sin(m * theta[i])

new_q_2i   = q_2i   * cos_m  −  q_{2i+1} * sin_m
new_q_2i+1 = q_2i   * sin_m  +  q_{2i+1} * cos_m
```

In practice, frameworks precompute a `(seq_len, d/2)` table of cos and sin values for all positions and frequencies, then apply the rotation as element-wise multiplication — O(seq_len · d) work, the same cost as adding sinusoidal embeddings.

### Where in the model does this happen?

In a standard transformer attention block:
1. Compute `Q = X · Wq`, `K = X · Wk`, `V = X · Wv`.
2. **Apply RoPE**: rotate each row of `Q` by the angle for its sequence position; same for `K`.
3. Compute attention: `softmax(QKᵀ/√d) · V` as usual.

No changes to the rest of the architecture. This is a drop-in replacement.

---

## Where it breaks

**Context length extrapolation is the principal weakness.** The fast-rotating frequencies (small *i*, θᵢ close to 1) complete many full cycles over a long sequence. Once the period of the fastest frequency is shorter than the training sequence, the model never sees a given relative offset at a given phase — so it cannot learn to extrapolate. Training on 4k tokens and inferring at 32k tokens breaks RoPE with `base = 10,000`.

The standard mitigation is **increasing the base**: LLaMA-3 uses `base = 500,000`. A larger base slows all frequencies proportionally, extending the effective context range roughly linearly with base — but requires retraining or at least fine-tuning on longer contexts.

More surgical fixes (YaRN, LongRoPE, RoPE ABF, NTK-aware scaling) apply a learned or analytic adjustment to the frequency table to "stretch" positions seen at training time to cover longer inference-time offsets. These achieve near-lossless long-context extension without full retraining.

**2D and 3D position generalization.** Extending RoPE to image patches (2D positions) or video frames (3D) requires choosing how to assign angles across dimensions. Naive flattening loses spatial structure; interleaved 2D RoPE (assigning half the pairs to x-position and half to y-position) is the current standard but is an engineering choice, not a principled derivation.

**Absolute position information is lost.** Because only relative positions appear in dot products, the model cannot directly distinguish "this sequence starts at position 0" from "this sequence starts at position 1000." For tasks that need absolute position (e.g., knowing you are near the beginning of a document), RoPE provides no signal. In practice this matters rarely for language tasks.

---

## Why it works

The deeper principle is the **Fourier shift theorem**: a translation in the time domain is a phase rotation in the frequency domain.

Formally: if `F(ω)` is the Fourier transform of `f(t)`, then `F{f(t − τ)}(ω) = F(ω) · e^(−2πiτω)`. A shift of `τ` multiplies each frequency component by `e^(−2πiτω)` — exactly a rotation in the complex plane.

RoPE is applying this theorem *inside* the attention mechanism:
- The token index *m* plays the role of the time-domain shift *τ*.
- Each frequency pair *i* (with frequency θᵢ) plays the role of a single Fourier basis component.
- Rotating query at position *m* and key at position *n* gives a relative rotation of `e^(i(m−n)θᵢ)` in each pair — the dot product captures only the difference, not the absolute values.

In other words: **encoding position as a rotation in a Fourier basis ensures that inner products measure relative position**. This is the same reason the DFT is used for cross-correlation (shift detection) in signal processing, and the same reason the phase of two complex numbers encodes the angular difference between them, not their individual orientations.

Contrast this with *additive* positional encoding (sinusoidal or learned):

```
attention(m, n) = (q + PE(m)) · (k + PE(n))
                = q·k  +  q·PE(n)  +  PE(m)·k  +  PE(m)·PE(n)
```

The cross-terms `q·PE(n)` and `PE(m)·k` mix absolute positions with content, forcing the model to learn "ignore this when you don't need it." With RoPE the content (magnitude/norm) and position (rotation angle) are *orthogonal channels* — the rotation affects angles, not norms — so there is no contamination.

A second lens: **relative-position encoding as OCC on keys**. Conventional attention lets any query "see" any key regardless of distance. Relative position encoding is a soft bias that says: "reduce attention to far-away keys." RoPE achieves this bias automatically because `Re(q̄k · e^(i(m−n)θ))` oscillates and, averaged over content diversity, decays in magnitude as `|m−n|` grows — it is a built-in decay kernel, without being a hard mask or a learned parameter.

---

## Going deeper

1. **YaRN: Efficient Context Window Extension of Large Language Models** (Peng et al., 2023) — derives a principled NTK-aware frequency rescaling to extend RoPE to 128k+ contexts without full retraining; good for understanding the extrapolation failure mode in detail. [arxiv:2309.00071]

2. **Roformer: Enhanced Transformer with Rotary Position Embedding** (Su et al., 2021) — the original paper proposing RoPE; contains the formal derivation and proof that the multiplicative rotation is the *unique* solution (up to scaling) satisfying the relative-position inner-product constraint. [arxiv:2104.09864]

3. **Extending the Context Window of Large Language Models via Positional Interpolation** (Chen et al., 2023) — the simpler (and widely-deployed) position interpolation approach that re-scales all positions to fit within the training window; complementary to YaRN and clarifies the trade-off between interpolation and extrapolation. [arxiv:2306.15595]

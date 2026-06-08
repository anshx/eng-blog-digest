---
title: "Looking Back at Speculative Decoding"
source: https://research.google/blog/looking-back-at-speculative-decoding/
author: Yaniv Leviathan, Matan Kalman, Yossi Matias
company: Google Research
date_posted: 2024-12-06
date_digested: 2026-06-08
---

# Looking Back at Speculative Decoding

## What's new to learn

1. **The memory-bandwidth bottleneck of autoregressive decoding.** At batch size 1, LLM decode uses less than 1% of a GPU's theoretical FLOP capacity—not because of inefficiency, but because every token generation requires loading all model weights from HBM while doing trivially little arithmetic. This makes the problem amenable to a completely different class of optimization than FLOP reduction.

2. **Draft-then-verify as a throughput multiplier.** A small "draft" model generates k candidate tokens cheaply; the expensive "target" model verifies all k in one parallel forward pass (same cost as verifying 1). When the draft is right, you pocket k tokens for the price of 1 target-model call.

3. **Rejection sampling for lossless output.** Each draft token is accepted with probability `min(1, p_target / p_draft)`. Tokens rejected are replaced by sampling from a "residual" distribution `normalize(max(0, p_target − p_draft))`. This classical trick, applied per-token in sequence, guarantees the output is distributed exactly as if the target model had generated every token itself—with no quality loss by proof, not heuristic.

## Prerequisites

- **Autoregressive LLM inference**: the model generates one token at a time; each step is one full forward pass over all model weights, conditioned on the KV cache of prior tokens.
- **Why decode is slow**: the GPU memory bandwidth ceiling, not arithmetic throughput, limits single-token generation. Transformers at batch size 1 are memory bound.
- **Rejection sampling (classical)**: if you have samples from distribution `q` and want samples from `p`, accept each sample `x` with probability `min(1, p(x)/q(x))`. The accepted samples are distributed as `p`. This technique predates LLMs by decades.
- **What a KV cache is**: previously computed key/value attention matrices are cached so each new token only computes attention over its own new position.

## The core idea

A 70B-parameter LLM in FP16 weighs ~140 GB. On an H100, peak HBM bandwidth is ~3.35 TB/s but peak compute is ~1,979 TFLOPS. Generating one token at batch size 1 requires loading all 140 GB of weights while doing only a few million multiply-adds—a severe memory-bandwidth bottleneck. The GPU is arithmetically idle more than 99% of the time.

Now here is the key observation: **loading those weights once is just as fast whether you verify 1 draft token or k draft tokens in that same pass.** The target model's forward pass over a sequence of length L+k (existing context plus k draft tokens) runs in the same number of memory reads as a forward pass over L+1, because attention is computed over the full sequence in one batched matrix multiply. So verifying k tokens has nearly the same memory cost as verifying 1.

Speculative decoding exploits this. A small, fast "draft" model (say, 7B parameters) generates k tokens autoregressively in k cheap steps. The large "target" model then runs a single forward pass to score all k positions at once. For each position the draft token is either accepted (matching the target's distribution) or replaced. In the best case—high draft accuracy—you receive k+1 tokens for the cost of one target-model call. In the worst case you get 1 token and fall back to exactly what vanilla decoding would have done.

The non-obvious part is the acceptance criterion: it must be mathematically tight enough to guarantee that the final token sequence is indistinguishable from the target model operating alone. That tightness comes from rejection sampling.

## Mechanics

### Step 1 — Draft

Run the draft model autoregressively from context `C`, producing k tokens `(x̃₁, x̃₂, ..., x̃ₖ)` and their probability distributions `q(·|C), q(·|C,x̃₁), ...`.

### Step 2 — Score

Run the target model once over `[C, x̃₁, x̃₂, ..., x̃ₖ]`. Because the target processes all positions in one parallel pass (standard transformer batched attention), it emits distributions `p(·|C), p(·|C,x̃₁), ..., p(·|C,x̃₁,...,x̃ₖ₋₁)` as a by-product of a single forward pass. This is the computational magic: one forward pass, k+1 distributions.

### Step 3 — Accept or replace

Walk the k tokens left to right. For token `x̃ᵢ`:

```
αᵢ = min(1,  p_target(x̃ᵢ | C, x̃₁,...,x̃ᵢ₋₁)
             ─────────────────────────────────── )
              p_draft(x̃ᵢ  | C, x̃₁,...,x̃ᵢ₋₁)
```

Draw `u ~ Uniform(0,1)`:
- If `u < αᵢ`: accept `x̃ᵢ`, move to position i+1.
- If `u ≥ αᵢ`: **reject**. Sample a replacement from the residual distribution:

```
p'(x) = normalize( max(0, p_target(x) − p_draft(x)) )
```

Discard `x̃ᵢ₊₁, ..., x̃ₖ`. Stop.

### Step 4 — Bonus token

If all k draft tokens were accepted, sample one final token from `p_target(·|C,x̃₁,...,x̃ₖ)` (already computed in Step 2). Total output: k+1 tokens.

### Expected throughput gain

Let `α` be the per-position acceptance rate (assumed i.i.d. for analysis). The expected number of tokens produced per target-model call is:

```
E[tokens] = (1 − αᵏ⁺¹) / (1 − α)
```

| α (acceptance rate) | k=4 | k=7 |
|---|---|---|
| 1.00 | 5.0 | 8.0 |
| 0.85 | ~3.7 | ~4.9 |
| 0.70 | ~2.8 | ~3.5 |
| 0.50 | ~1.9 | ~2.0 |

In practice, 70B-class models paired with well-matched 7B drafts achieve α ≈ 0.75–0.90 on common tasks, yielding 2–3× wall-clock speedup after accounting for draft overhead.

### Why the residual distribution works

The classical rejection sampling theorem guarantees that if you accept `x ~ q` with probability `min(1, p(x)/q(x))` and otherwise resample from `p`, the resulting samples are distributed as `p`. 

Speculative decoding refines this: instead of resampling from `p` (which would require another target-model forward pass), it samples from the "residual" `p' = normalize(max(0, p − q))`. The residual is exactly the leftover probability mass in `p` that `q` did not already cover. Together, the accepted samples from `q` and the resampled tokens from `p'` produce a combined distribution equal to `p`. This entire step is computed from the two probability vectors that Step 2 already materialized—no extra model calls needed.

## Where it breaks

**Low acceptance rate**: If the draft model is a poor approximation of the target (different training data, architecture family, or capability level), `α` drops below 0.5 and speculative decoding delivers less throughput than vanilla decoding while consuming extra memory and VRAM for the draft.

**Large batch sizes**: At batch size N, the target model processes N sequences in parallel, spreading the weight-read cost over N tokens. As N grows, the GPU shifts from memory-bandwidth-bound to compute-bound. The asymmetry that speculative decoding exploits disappears. The technique is primarily a latency optimization (single-request throughput), not a throughput-at-scale optimizer.

**Memory overhead**: You must fit both the draft and the target model on device simultaneously. For a 70B target and a 7B draft, this adds ~14 GB of VRAM—non-trivial.

**Vocabulary and architecture coupling**: Until 2024, draft and target models needed to share the same tokenizer and vocabulary so token-level probabilities could be compared directly. Newer work on universal assisted generation relaxes this constraint but adds complexity.

**Overhead on short sequences**: For a 5-token response, the draft overhead may be comparable to the gain. Speculative decoding pays off more on long-generation tasks.

## Why it works

### CPU speculative execution — the exact same pattern

Modern out-of-order CPUs encounter branches and cannot know which path will be taken until the condition is evaluated. Rather than stalling, they **speculatively execute** the predicted branch. If the prediction is correct, the work is committed and several instructions' worth of latency is hidden. If wrong, the speculative work is discarded and the correct path executes.

Speculative decoding is this pattern applied one level up:
- The **branch condition** → "what is the next token?" (resolved only by the target model)  
- **Speculative execution** → the draft model generates k token predictions without waiting
- **Commit or rollback** → accepted tokens are committed, rejected ones are discarded

The mathematical proof that rollback produces no distribution skew is speculative decoding's twist that CPUs lack: CPU speculation can give wrong results if squashing fails; speculative decoding has a proven algebraic guarantee.

### The general principle: "speculate-then-verify"

Any operation that is:
1. Memory-bandwidth-bound at small batch sizes, but
2. Parallelizable across a batch (verifying k items costs O(1) × verifying 1 item)

...is a candidate for speculate-then-verify acceleration. The trick requires a cheap approximator whose outputs can be scored by the expensive oracle cheaply and in bulk.

This is the same pattern underlying **hardware branch predictors** (CPUs), **optimistic concurrency control** (databases — also in this archive), and **prefetching** (storage systems): make a cheap bet, apply the expensive operation to confirm, roll back cheaply on miss. The quality of the approximator determines the speedup multiplier.

### Why rejection sampling gives losslessness

The key non-obvious claim is that `p'(x) = normalize(max(0, p − q))` completes the probability mass correctly. Consider the probability of emitting token `x` at a given position:

- Probability that `x` is drawn from `q` and accepted = `q(x) × min(1, p(x)/q(x))` = `min(q(x), p(x))`
- Probability that some other token `y ≠ x` is drawn and rejected, then `x` is drawn from `p'` = `(1 − Σ_y min(q(y), p(y))) × p'(x)`

Summing these:

```
P(emit x) = min(q(x), p(x)) + p'(x) × (1 − Σ_y min(q(y), p(y)))
           = min(q(x), p(x)) + max(0, p(x)−q(x))
           = p(x)
```

The algebra cancels perfectly. This is the heart of rejection sampling: the deficit in `q` relative to `p` is exactly `max(0, p−q)`, and sampling that residual fills the gap. No target model call needed.

## Going deeper

1. **Original paper**: Leviathan et al., "Fast Inference from Transformers via Speculative Decoding" (ICML 2023, arXiv:2302.01318). The concurrent DeepMind paper "Accelerating Large Language Model Decoding with Speculative Sampling" (Chen et al., arXiv:2302.01814) derives the same technique independently, providing additional analysis of the acceptance rate formula.

2. **Medusa** (Cai et al., 2024, arXiv:2401.10774): Eliminates the separate draft model by adding k lightweight "Medusa heads" on top of the target model's last hidden layer. Each head predicts tokens `+1, +2, ..., +k` ahead. Verification uses a tree-based attention mask so all candidates are evaluated in one pass. The insight: the draft model doesn't need to be a separate model—it just needs cheap access to the target's internal representation.

3. **HuggingFace "Assisted Generation"** blog post (Gante, May 2023, at huggingface.co/blog/assisted-generation): The most accessible engineering walkthrough of implementing speculative decoding in the Transformers library, covering vocabulary alignment, dynamic speculation length, and the interaction with greedy vs. sampling decoding modes.

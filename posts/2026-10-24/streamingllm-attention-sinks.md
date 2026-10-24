---
title: "Efficient Streaming Language Models with Attention Sinks"
source: https://arxiv.org/abs/2309.17453
author: Guangxuan Xiao, Yuandong Tian, Beidi Chen, Song Han, Mike Lewis
company: MIT / Meta AI / CMU / NVIDIA
date_posted: 2023-09-29
date_digested: 2026-10-24
tags: [transformers, attention, kv-cache, llm-inference, streaming, memory-efficiency]
---

# Efficient Streaming Language Models with Attention Sinks

**Source:** arXiv:2309.17453 (ICLR 2024)  
**Authors:** Guangxuan Xiao, Yuandong Tian, Beidi Chen, Song Han, Mike Lewis  
**Affiliation:** MIT / Meta AI / CMU / NVIDIA

---

## What's new to learn

Three things worth internalizing:

1. **Attention sinks are a universal emergent artifact.** Initial tokens in any transformer accumulate disproportionately high attention scores—not because their content is meaningful, but because softmax normalization *requires* somewhere for excess probability mass to go, and position 0 is structurally privileged.

2. **StreamingLLM algorithm.** Retain the first K=4 "sink" KV states permanently, plus a sliding window of W recent tokens. Bounded memory, infinite streaming generation, no retraining.

3. **Forced normalization creates structural drains.** Any system with a conservation law (outputs must sum to a constant) will develop "null device" positions that absorb excess. This pattern recurs across distributed systems, signal processing, and machine learning.

---

## Prerequisites

- **KV cache mechanics**: During autoregressive generation, each token computes key/value vectors once and caches them. Subsequent tokens attend over the full cache instead of recomputing.
- **Self-attention**: `Attention(Q,K,V) = softmax(QKᵀ/√d)·V`
- **Causal masking**: Position `i` can attend to positions `0..i` only (lower-triangular mask).
- **RoPE**: Positions are encoded by rotating Q/K by angle `θ·pos`. Relative position `i−j` is what the model actually sees, not absolute index.

---

## The problem: why sliding-window KV cache fails

The obvious way to make generation memory-bounded is to keep only the W most recent tokens in the KV cache—a pure sliding window. When the cache fills, evict the oldest token.

This catastrophically fails. Perplexity diverges. Generation degrades into incoherence.

The failure happens *even on inputs the model handles perfectly* when the full cache is available. The model is not running out of context. It's losing something structural.

---

## The surprising finding: attention sinks

When you visualize per-layer, per-head attention weight matrices for a well-trained LLM (Llama-2, MPT, Falcon, etc.), you see the same pattern everywhere, across all 32+ layers and all attention heads:

**The first 1–4 token positions receive anomalously high attention from nearly every subsequent token, regardless of their content.**

This is not sparsity in the usual sense—high attention going to the most semantically relevant prior token. The initial tokens receive high attention from tokens generating completely unrelated text, 4,000 positions later.

The paper calls these positions **attention sinks**.

A controlled ablation clinches the causal story: if you replace softmax with linear (unnormalized) attention, the sink pattern disappears entirely. The phenomenon is a softmax artifact, not an artifact of the data or architecture.

Another ablation: if you insert a fixed placeholder token (a period, a newline, an arbitrary special token) at position 0 before generation, it becomes the sink instead. The content at the sink position is semantically irrelevant. The position itself is what matters.

---

## Why softmax creates sinks

Softmax normalization forces attention weights to sum to 1:

```
attention_weights = softmax(q · kᵀ / √d)   # weights ∈ (0,1)ⁿ, sum = 1
```

This is a conservation law. The total probability budget is fixed. Every token in a sequence must allocate its attention budget across all accessible keys.

Now consider what happens when a query token has no semantically relevant context to attend to. The query might be generating a token where all prior context is equally irrelevant. But the budget must go somewhere. Softmax *cannot* output near-zero for everything; it always outputs a valid probability distribution.

During training, the model learns to route this "surplus" attention to designated drain positions. Those positions develop a specific learned representation: their **key vectors produce high dot products** (making them attract attention from almost any query) while their **value vectors are near-zero** (so attending there barely contributes to the output).

Position 0 is the natural drain candidate because:

1. **Universal visibility.** Causal masking makes position 0 visible to every single subsequent position. No other position has this property—position 5 is invisible to positions 1–4.
2. **Training pressure.** Across millions of sequences, the model learns a prior that "if nothing relevant is present, route attention to a learned null device." Position 0 is the one position that can absorb this gradient consistently across all positions.

The result: sink positions become learned **"I have no useful context"** signals. High attention to sinks encodes "this token is not contextually grounded in the current window," not "this initial token said something important."

---

## The StreamingLLM algorithm

The key insight from the sink analysis: evicting sink tokens collapses generation not because their *content* was needed, but because the model needs a valid target for structural probability mass. If you bring the sinks back—keep them in cache permanently—the sliding window works perfectly.

```
# KV cache layout:
sink_cache    = KV states of tokens [0, 1, ..., K-1]   # always retained, K=4
recent_cache  = KV states of the W most recent tokens   # sliding window

# Per-token generation step:
def streaming_step(new_token, sink_cache, recent_cache, W):
    cache = concat(sink_cache, recent_cache)
    output = attention(new_token, cache)
    recent_cache.append(new_token)
    if len(recent_cache) > W:
        recent_cache.pop(0)          # evict oldest non-sink token
    # sink_cache is never evicted
    return output
```

Cache size is bounded at K + W tokens regardless of sequence length. Generation can run indefinitely.

**The RoPE fix.** The one subtlety: RoPE encodes positions via rotation angle, so absolute positions matter to the model's internal arithmetic. If sink tokens are at positions 0–3 and the sliding window currently spans tokens 4,000–5,000, naively using the original position indices puts the recent window far out of distribution.

StreamingLLM re-indexes positions at each step: sinks get indices 0..K-1, the sliding window gets indices K..K+W-1. This keeps all tokens in-distribution with respect to the position encoding, regardless of how far into the sequence you are.

---

## Results

Tested on Llama-2 (7B/13B), MPT (7B/30B), Falcon (7B), Pythia (6.9B):

- **4 million tokens** generated without perplexity divergence (tested limit; no failure observed).
- **Perplexity is nearly identical** to full KV cache attention at equivalent window sizes.
- **22.2× speedup** over the "sliding window + recompute evicted tokens" baseline (recomputing from scratch at each step is the obvious-but-slow alternative).
- **K=4 sinks suffice.** Diminishing returns beyond 4; 1 sink is fragile, 2–3 are marginal, 4 is robust across all tested families.

---

## Where it breaks

**No long-range recall.** Evicted tokens are gone permanently. If a user references something from 10,000 tokens ago that isn't in the recent window, the model cannot retrieve it. StreamingLLM provides streaming coherence (the model continues generating plausibly), not infinite memory.

**Not a context-extension technique.** This is a common misread. StreamingLLM does not make the model "know" more context—it makes the model fail gracefully on arbitrarily long inputs instead of crashing. The model's effective context is still bounded by W.

**Position re-indexing is a heuristic.** Re-numbering positions at each step is not theoretically principled—it exploits the empirical fact that RoPE generalizes reasonably within-distribution. It works, but the gap between "it works empirically" and "we understand why" is real.

**K is not universal.** K=4 was established empirically across the tested model families. Different architectures or training regimes might require different K values; the paper does not derive K from first principles.

---

## The deeper pattern: forced normalization creates structural drains

Attention sinks are not a quirk of transformers. They're an instance of a recurring systems pattern:

> **In any system with a conservation law, "null device" positions emerge to absorb excess.**

- **Load balancers with default backends.** When no routing rule matches, traffic must go somewhere. Default backends become structural sinks for unclassified traffic—not because the destination matters, but because the routing table must be total.
- **RNN forget gates.** A recurrent network's hidden state has bounded capacity. Forget gates learn to route "irrelevant" inputs to near-zero weight; the gate saturates at 0 as a structural drain.
- **Hash tables and slack.** Open-addressing hash tables require empty slots; without them, probing loops. The empty slots are structural drains that absorb probe sequences, not "wasted space."
- **Probability distributions in general.** Any learned distribution over a fixed support will develop low-entropy "garbage collector" mass somewhere if the model is trained on data that doesn't fill the support uniformly.

The mental model: **when a system's outputs are forced to be normalized, learned "null positions" are thermodynamically inevitable.** They represent the model's learned prior for "no signal here."

This is why the StreamingLLM ablation works: inserting an arbitrary token at position 0 makes *that token* the null device, even though the model was never explicitly trained on it. The structural pressure from softmax normalization shapes the learned representation regardless of the token's identity.

---

## Connection to other posts in this archive

- **PagedAttention / vLLM**: Physical KV cache allocation and eviction policies. StreamingLLM's streaming algorithm is a complementary layer—it determines *what* to keep; PagedAttention determines *where* to put it in GPU memory.
- **GQA (Grouped-Query Attention)**: Reduces KV cache size by sharing keys/values across query heads. StreamingLLM reduces cache size by evicting old tokens. These compose: GQA per head, StreamingLLM across the sequence.
- **FlashAttention / FlashDecoding**: IO-efficient attention kernels. StreamingLLM's bounded-cache attention is a strict subset of flash-attention-compatible patterns (the cache is small and dense by construction).
- **Speculative Decoding**: Also a streaming inference optimization, targeting latency via draft/verify. Orthogonal to StreamingLLM; both can be applied simultaneously.
- **RoPE**: The position re-indexing trick in StreamingLLM is a direct consequence of RoPE's properties. ALiBi or learned absolute positions would require different strategies.

---

## Going deeper

1. **"When Attention Sink Emerges in Large Language Models"** (arXiv:2410.10781) — deeper mechanistic analysis of what causes sinks and whether they can be eliminated at training time.
2. **"Attention Sink in Transformers: A Survey on Utilization, Interpretation, and Mitigation"** (arXiv:2604.10098, 2025) — comprehensive survey; covers attempts to train sink-free models and why it's harder than it looks.
3. **LM-Infinite** (arXiv:2308.16137) — concurrent work on similar ideas; uses a Λ-shaped attention mask instead of full sink retention.
4. **MagicPIG** (arXiv:2410.16179) — extends streaming inference with LSH-based approximate retrieval to partially address the "no long-range recall" limitation.

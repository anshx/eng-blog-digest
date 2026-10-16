---
title: "In-Context Learning and Induction Heads"
source: https://transformer-circuits.pub/2022/in-context-learning-and-induction-heads/index.html
author: Catherine Olsson, Nelson Elhage, Neel Nanda, Nicholas Joseph, Nova DasSarma, Tom Henighan, et al.
company: Anthropic
date_posted: 2022-03-08
date_digested: 2026-10-16
---

# In-Context Learning and Induction Heads

## What's new to learn

1. **Induction heads**: a two-layer attention circuit that implements "prefix matching and copying" — for any input pattern `...A B...A`, an induction head predicts `B` by finding the earlier occurrence of `A` and copying its successor. It is the mechanistic implementation of in-context pattern completion inside a transformer.

2. **Phase transitions in capability**: in-context learning does not emerge gradually with scale; it appears suddenly at a specific point in training that coincides exactly with induction head formation — evidence that capabilities can require entire multi-layer circuits to co-form before they exist at all.

3. **In-context learning as distributed key-value retrieval**: ICL is, at a mechanistic level, a nearest-neighbor lookup against the token sequence in the context window, implemented by attention — transformers are re-running something structurally similar to retrieval-augmented generation inside their own forward pass.

## Prerequisites

- Transformer attention: queries Q, keys K, values V; attention score = softmax(QKᵀ/√d); output = weighted sum of V
- Residual stream: each attention head and FFN layer *adds* to a persistent vector that accumulates all contributions
- In-context learning (ICL): the observation that a frozen pre-trained transformer can perform new tasks just from a few labeled examples placed in the input — no weight updates required
- Softmax as soft argmax: the attention pattern is differentiable "find the most relevant past token"

## The core idea

Consider a transformer reading the sequence:

```
... groble → vrix ... groble → ?
```

An induction head solves this perfectly: it notices that `groble` appeared before, that `vrix` followed it, and predicts `vrix` now.

More formally: at position `t`, an induction head attends back to the position `t'` where the token at `t'-1` (the previous token there) most closely matches the token at `t-1` (the current previous token). Then it copies the value at `t'`, which is the token that followed the match.

The key insight is that this is **a two-layer operation**:
- A **previous-token head** (Layer 0) must first move information from position `t-1` into position `t`'s residual stream
- An **induction head** (Layer 1) can then use that enriched representation to attend back to positions that had the same "preceded by" token

Neither layer is sufficient alone. This is why the capability appears suddenly: both heads must co-form during training for either to be useful. Until the circuit is complete, zero gain is extracted from it.

## Mechanics

### The two-head circuit in detail

**Layer 0 — previous-token head:**
For each position `t`, this head attends primarily to position `t-1`:

```
attn_score(t, t') = Q(resid[t]) · K(resid[t'])
```

with weights heavily concentrated on `t' = t-1`. The output vector for position `t` therefore contains information about the token at `t-1`. This output is added to the residual stream at position `t`.

**Layer 1 — induction head:**
At position `t`, the query is formed from the current residual stream (which now includes the `t-1` token's info via Layer 0's addition). The key at position `t'` is formed from `t'`'s residual stream — which similarly includes `t'-1`'s token info after Layer 0.

The dot product `Q(t) · K(t')` is therefore high when the token at `t-1` resembles the token at `t'-1`. This means the induction head attends to positions where the same two-token context occurred before.

The value at position `t'` is information about the token at position `t'` itself. So the induction head's output at `t` is a soft copy of "the token that followed the previous occurrence of the current bigram."

### Measuring "inductiveness"

The authors define the **induction score** of an attention head as its average attention to position `t' = t - seq_len + 1` on a repeated random token sequence — i.e., given `ABCABC`, does position 5 (second `B`) attend to position 2 (first `B` + 1 = 2's successor)? High score = inductive behavior.

This metric allows them to identify induction-head-like attention heads in any transformer, including large ones where the precise circuit structure is harder to decompose.

### The phase transition

Training a 2-layer attention-only model on a language modeling loss, the following things happen *simultaneously* at a specific step:

1. The induction score for both the previous-token head and the induction head jumps from near-zero to near-maximum
2. The in-context learning score (measured as: how much better does the model predict the second copy of a token vs. the first copy in a repeated sequence?) jumps at the same moment
3. The loss on long-context examples drops sharply

Before the phase transition: neither head has formed, and the model cannot leverage context for pattern completion.  
After: both heads exist and the model gains substantial in-context learning ability — all at once, in a single training step window.

This is directly analogous to a percolation threshold: adding one more connection creates a spanning cluster that didn't exist before.

### Evidence in large models

For models up to 13B parameters, the authors find:
- **Correlation**: the fraction of attention heads with high induction scores correlates strongly (r > 0.9) with in-context learning ability across models
- **Activation patching**: patching induction-head activations from one forward pass to another transfers ICL performance
- **Ablation**: zero-ing out induction heads selectively degrades in-context learning far more than ablating random heads of equivalent size

The main caveat: this evidence is correlational for large models. The mechanistic proof (complete circuit decomposition) is only clean for 2-layer attention-only models.

## Where it breaks

**The 2-layer simplification.** The clean previous-token-head → induction-head circuit is provably present in 2-layer attention-only models. In full GPT-scale models with MLP layers and LayerNorm, the mechanism is qualitatively similar but more diffuse — multiple heads contribute, MLPs mediate, and there is no single clean two-head circuit to ablate.

**Simple pattern completion is not full ICL.** Induction heads explain how a model can repeat a pattern it has seen in context. But in-context few-shot learning on novel tasks — e.g., learning to translate from French to German from three examples — requires much more than bigram completion. The paper explicitly frames induction heads as implementing "copying and completion" and leaving open how more complex ICL works.

**The phase transition may not generalize.** The sharp training phase transition is clear in small attention-only models. In large models trained with data mixing, curriculum changes, and multi-epoch training, phase transitions blur or fragment across checkpoints.

**Causality gap.** The causal evidence (activation patching, ablation) in large models is still indirect. The model's behavior changes when induction heads are ablated, but this proves the heads *contribute*, not that they *implement* ICL via the described mechanism.

## Why it works

### The deeper principle: context-window retrieval is attention

The central insight the paper teaches is that **in-context learning is retrieval**, and **retrieval is exactly what attention already does**.

Standard attention: given a query about the current position, attend to all past keys and retrieve the corresponding values. If you tune the Q, K projections such that "query = what token just preceded me, keys = what token preceded each past position," you have exactly induction heads.

This unifies:
- Retrieval-augmented generation (RAG): look up relevant documents in an external DB to condition generation
- In-context few-shot learning: look up relevant patterns in the context window to condition generation
- Self-attention generally: at every position, look up the most relevant past representations to condition the next prediction

They are all the same operation: **nearest-neighbor retrieval over a key-value store, with the current query used to select past values**. The only difference is whether the KV store is external (RAG), in-context (ICL), or the full past sequence (standard attention).

### Why the phase transition

The phase transition is a **coordination problem** between two circuits that must co-exist to be useful. A previous-token head with no downstream induction head is wasted capacity — it shifts information that nobody reads. An induction head with no upstream previous-token head cannot form its matching key. Neither can be learned independently from the cross-entropy loss, because neither produces useful predictions alone.

This is directly analogous to why:
- Both a lock *and* a key must be manufactured before either is useful
- A MapReduce system must implement both map *and* reduce phases before any job completes
- A producer-consumer queue (LMAX Disruptor) needs both a producer and a consumer before any work flows through

The training dynamic is two simultaneously evolving gradients with a multiplicative interaction: the product (induction circuit) jumps from zero to nonzero only when both components are present. This is the mechanism behind emergent capabilities generally: abilities that require co-formed circuits will appear suddenly, not gradually, in loss curves.

### The abstraction: transformers are learned retrieval systems

This paper's deepest contribution is framing the transformer as a **learned, context-adaptive retrieval system**. Every attention head is implementing some form of "find past states that are relevant to the current query and aggregate their values." Different heads specialize into different relevance functions — syntactic agreement, coreference, topic similarity, bigram prediction. Induction heads specialize into temporal pattern repetition.

The "attention is soft nearest-neighbor retrieval" insight (from the "Attention Is All You Need" deep dive in this archive) is the mechanism. This paper shows that **the mechanism is powerful enough to implement in-context learning itself** — the model can learn *new* retrieval criteria from context, not just the fixed ones baked into its weights.

## Going deeper

1. **"A Mathematical Framework for Transformer Circuits"** (Elhage et al., 2021, transformer-circuits.pub) — the formal decomposition of transformers into composition of attention heads, introducing the residual stream, QK circuits, and OV circuits. The foundation for this paper.

2. **"Interpretability in the Wild: A Circuit for Indirect Object Identification"** (Wang et al., 2022, arXiv:2202.05262) — applies the circuit-finding methodology to a real task (IOI: "John gave Mary the ball; Mary gave [John] the ball"), finding a 26-head circuit with named roles. Shows the methodology scales to non-trivial tasks.

3. **"Grokking: Generalization Beyond Overfitting on Small Algorithmic Datasets"** (Power et al., 2022, arXiv:2201.02177) — the complementary view: phase transitions in *generalization* during training of algorithmic tasks. Paired with the induction head phase transition, argues that many capability jumps reflect circuit co-formation rather than smooth improvement.

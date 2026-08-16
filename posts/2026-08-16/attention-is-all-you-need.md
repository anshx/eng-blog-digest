---
title: "Attention Is All You Need"
source: https://arxiv.org/abs/1706.03762
author: Ashish Vaswani, Noam Shazeer, Niki Parmar, Jakob Uszkoreit, Llion Jones, Aidan N. Gomez, Łukasz Kaiser, Illia Polosukhin
company: Google Brain / Google Research
date_posted: 2017-06-12
date_digested: 2026-08-16
---

# Attention Is All You Need

## What's new to learn

1. **Scaled dot-product attention is differentiable nearest-neighbor lookup in a key-value database.** Q (query), K (key), and V (value) are three linear projections of the same input. Attention computes pairwise similarity between every query and every key, normalises with softmax, and returns a weighted average of values — exactly what you'd want from a fuzzy read from a hash table, but fully differentiable and end-to-end trainable.

2. **Multi-head attention is an ensemble of parallel information-retrieval specialists.** Rather than one fat attention with full width, you run h independent attentions in lower-dimensional subspaces and concatenate their outputs. Each head can specialise in a different relational pattern (syntactic, semantic, coreference) without any explicit supervision.

3. **Sinusoidal positional encoding injects order via frequency, not learned slots.** Because attention has no inherent notion of position — it treats tokens as a set, not a sequence — you add a vector encoding the token's position using sine/cosine functions at different frequencies, chosen so that each position's encoding is a linear function of nearby positions' encodings.

## Prerequisites

- What softmax does: maps a vector of real numbers to a probability distribution (values sum to 1, all non-negative), amplifying the largest component.
- Matrix multiplication basics: how a linear projection `xW` works, and that batched matrix multiplies handle all positions simultaneously.
- Roughly what seq2seq (encoder-decoder) means: an encoder reads the source, a decoder generates the target token by token.
- Why vanishing gradients are a problem: deep networks without residual connections lose gradient signal in early layers.
- Not required: prior familiarity with RNNs, LSTMs, or convolutional sequence models — though comparing against them helps.

## The core idea

Before transformers, sequence models (LSTMs, GRUs) processed tokens one at a time: step t could only see step t-1's hidden state, which was supposed to summarise all of t-1, t-2, ... 1. This created two problems:
1. Training was inherently sequential — you couldn't parallelise across positions.
2. Information from distant positions had to flow through hundreds of non-linear transformations, degrading or vanishing.

The transformer's answer: make each token attend directly to every other token in one step, with no sequential bottleneck.

For a sequence of n tokens, each converted to a d-model dimensional embedding, you compute three matrices by multiplying by learned weight matrices W^Q, W^K, W^V:

- **Q** ("what am I looking for?") — each row is a query vector for that position
- **K** ("what do I advertise?") — each row is a key vector for that position
- **V** ("what is my actual content?") — each row is a value vector for that position

Then:

```
Attention(Q, K, V) = softmax(Q K^T / √d_k) V
```

The term `Q K^T` is an n×n matrix of dot-product similarities between every query-key pair. After scaling and softmax, each row becomes a distribution over positions. Multiplying by V produces a weighted average of value vectors — effectively, for each query position, you're doing a soft lookup across the entire sequence and reading out a blend of values, weighted by relevance.

The result is a new n×d_v matrix where each position's output has been contextualised by the entire sequence — in a single layer, with no recurrence.

## Mechanics

### Scaled dot-product attention

```
Attention(Q, K, V) = softmax(QK^T / √d_k) V
```

**Why divide by √d_k?**  
When d_k is large (e.g. 64), dot products `q · k` grow in magnitude because they sum d_k terms. In the extreme, one dot product dominates and softmax produces a near-one-hot distribution — effectively zeroing out all but one position. This collapses gradients. Dividing by √d_k keeps the variance of the dot product at ~1, regardless of embedding dimension, keeping softmax in its high-gradient regime.

**Why not additive attention?**  
Bahdanau (2015) attention computes compatibility as `v^T tanh(W_1 q + W_2 k)`. Dot-product attention is computationally equivalent in theory but faster in practice because it's one batched matrix multiply instead of a feed-forward pass.

### Multi-head attention

Instead of one attention in d_model-dimensional space:

```
MultiHead(Q, K, V) = Concat(head_1, ..., head_h) W^O

head_i = Attention(Q W_i^Q, K W_i^K, V W_i^V)
```

The paper uses h=8 heads with d_model=512, so each head has d_k = d_v = 512/8 = 64. Total FLOPs are the same as a single-head attention at full dimension, but the ensemble property is very different: each head independently projects Q, K, V into a 64-d subspace and learns its own attention pattern.

**Why this matters:** Suppose head 1 learns to track syntactic subject-verb agreement, head 2 learns anaphora resolution (pronoun → antecedent), head 3 attends to sentence-final period to know when to stop. These patterns are mutually exclusive in any single attention's 512-d output, but composable as eight 64-d outputs concatenated together. You get specialisation for free, through training, with no explicit labels.

### The encoder layer (×6)

Each of the encoder's 6 identical layers applies:

1. **Multi-head self-attention** (all three Q, K, V come from the previous layer's output)
2. **Add & LayerNorm**: `LayerNorm(x + sublayer(x))`
3. **Position-wise Feed-Forward Network (FFN)**
4. **Add & LayerNorm** again

The FFN is applied independently to each position:

```
FFN(x) = max(0, x W_1 + b_1) W_2 + b_2
```

With d_model=512 and d_ff=2048 — a 4× expand then contract through a ReLU bottleneck. This gives each position a 2048-dimensional scratchpad to compute non-linear features between attention steps. (These FFN layers account for the majority of transformer parameters.)

**Residual connections** are load-bearing: `LayerNorm(x + sublayer(x))` means each sublayer only needs to learn a correction to its input, not a full re-representation. Without residuals, gradients through 6 × (attention + FFN) would vanish. Note that layer norm is used here rather than batch norm: layer norm normalises per-sample over the feature dimension, independent of batch size and sequence length, which is more stable for variable-length sequences.

### The decoder layer (×6)

Same structure as encoder, but with one extra sublayer:

1. **Masked multi-head self-attention** — decoder tokens can only attend to earlier positions. Future positions are masked to −∞ before softmax, ensuring the model doesn't "cheat" by reading the answer during training.
2. **Add & LayerNorm**
3. **Cross-attention** — Q comes from the decoder, but K and V come from the encoder's final-layer output. This is how the decoder "reads" the source.
4. **Add & LayerNorm**
5. **FFN**
6. **Add & LayerNorm**

### Positional encoding

Attention is permutation-equivariant: shuffle the input tokens and the output shuffles identically. For a language model this is catastrophic — "dog bites man" and "man bites dog" would produce the same representation. You must inject position information.

The paper adds a positional encoding vector to each token embedding before the first layer:

```
PE(pos, 2i)   = sin(pos / 10000^(2i / d_model))
PE(pos, 2i+1) = cos(pos / 10000^(2i / d_model))
```

This creates a unique d_model-dimensional vector for each position, with different embedding dimensions oscillating at different frequencies (like a Fourier series). The key geometric property: `PE(pos + k)` can be expressed as a fixed linear function of `PE(pos)`, independent of pos. This means the model can learn to compute relative offsets by learning a linear transformation of absolute positional encodings — without ever seeing that position during training.

The paper found learned positional embeddings performed similarly; sinusoidal encodings were chosen because they generalise to sequence lengths longer than any seen during training.

### Training setup and results

The base model (d_model=512, 6 layers, 8 heads, d_ff=2048) has 65M parameters. The big model (d_model=1024, 6 layers, 16 heads, d_ff=4096) has 213M parameters. Trained on 8 Nvidia P100 GPUs.

Key results on machine translation:

| Variant | En→De BLEU | En→Fr BLEU | Training cost |
|---|---|---|---|
| Transformer (base) | 27.3 | 38.1 | 12 hours |
| Transformer (big) | **28.4** | **41.0** | 3.5 days |
| Previous SOTA (ensemble) | 26.4 | 41.0 | weeks |

The big model matched SOTA on En→Fr at 1/10 the training cost, and exceeded SOTA on En→De.

### Ablations (selected findings)

- **Fewer heads → worse**: 1 head drops from 25.8 to 23.3 BLEU; too many (32 heads) also hurts (24.9). 8 is the sweet spot.
- **Smaller d_k → worse**: reducing key depth degrades quality, confirming that dot-product attention quality depends on key richness.
- **No dropout → worse**: 25.4 vs 25.8 — regularisation matters.
- **Replace LayerNorm with BatchNorm → worse**: not in original paper but well-established since.
- **Learned vs. sinusoidal PE → essentially same**: 25.8 vs 25.8 BLEU, so sinusoidal is preferred for generalisation.

## Where it breaks

**Quadratic cost in sequence length.** The attention matrix is n×n. For n=1024 tokens this is ~4M entries; for n=100K (book-length input) it's 40B entries. Memory is O(n²), compute is O(n² × d). This is why FlashAttention (in the archive) was necessary: it doesn't reduce asymptotic complexity but uses IO-aware tiling to avoid materialising the full n×n matrix in HBM.

**Absolute positional encoding is a hack.** Adding position as a one-time injection before layer 1 is fragile. The model must propagate positional signal through many layers of attention — and nothing forces it to. Later work (RoPE, ALiBi, xPos) learned that rotating Q and K vectors directly based on their positions is much more principled and generalises to longer sequences.

**Encoder-decoder is overengineered for pure generation.** GPT (2018) showed that decoder-only, with causal masking, suffices for language modelling and downstream tasks — eliminating the encoder and cross-attention. BERT (2018) showed encoder-only suffices for classification and extraction tasks. The original encoder-decoder is now mostly confined to translation and summarisation.

**FFN layers are black boxes.** Despite being the largest parameter block, the MLP sublayers are poorly understood compared to attention. Recent work (MoE transformers, including the Mixture of Experts post in this archive) addresses their density by routing tokens to specialist FFNs — but the fundamental opacity remains.

**Training stability requires careful tuning.** Without warm-up learning rate schedules (`lr ∝ d_model^{-0.5} × min(step^{-0.5}, step × warmup_steps^{-1.5})`), transformers diverge or converge slowly. This brittleness motivated LayerNorm placement research (Pre-LN vs. Post-LN) in subsequent years.

## Why it works

The deepest principle: **attention is differentiable relational reasoning over a set.**

A classical nearest-neighbor lookup is: given query q and a database of (key, value) pairs, return the value whose key is closest to q. Attention makes this continuous: instead of returning exactly one value, it returns a convex combination of all values, weighted by softmax of similarities. This is soft retrieval.

Making it differentiable means you can train the query, key, and value projections end-to-end — the model learns what questions to ask (W^Q), what features are worth advertising (W^K), and what content to pass along (W^V), all jointly optimised for the downstream loss.

**This pattern appears everywhere:**
- Content-addressable memory (Hopfield networks, 1982) — attention is a differentiable, multi-pattern Hopfield net.
- Database joins — attention is a soft join between query and key sets.
- Recommendation systems — a user embedding (query) attending over item embeddings (keys) to retrieve weighted item features (values).
- Pointer networks (Vinyals 2015) — attention used to output pointers into the input, a direct precursor.

Multi-head attention is just **ensemble learning applied to information retrieval subspaces.** The same principle as random forest feature subsampling, dropout (random zeroing is ensemble approximation), and ConvNet filter banks (multiple linear filters over the same input). Each head is a specialist with its own representation subspace.

Residual connections + layer norm together implement **gradient superhighways through depth.** The identity shortcut `x + f(x)` means the gradient signal flows unimpeded to layer 1 regardless of network depth. Layer norm stabilises the distribution at each layer boundary. Together, they're why you can stack 96 layers (GPT-3) and still train stably.

The positional encoding insight is that **order is just another feature dimension you inject.** Rather than building recurrence into the architecture (which forces sequential processing), you simply encode position as a vector and add it to content. Attention then handles both content-to-content and position-to-content relationships in the same mechanism.

## Going deeper

1. **"BERT: Pre-training of Deep Bidirectional Transformers for Language Understanding"** (Devlin et al., 2018) — How a bidirectional (encoder-only) transformer trained with masked language modeling unlocks representation learning without labeled data; the post-transformer paper that showed fine-tuning a pre-trained encoder beats task-specific architectures.

2. **"Roformer: Enhanced Transformer with Rotary Position Embedding"** (Su et al., 2021) — RoPE, the positional encoding used in LLaMA and most modern LLMs. Injects position by rotating Q and K in complex space rather than adding to embeddings, giving provably better generalisation to unseen lengths and cleaner relative attention semantics.

3. **FlashAttention** (already in this archive) — The IO-aware implementation of exact attention that makes transformers practical at longer sequence lengths by tiling the n×n attention matrix so it never fully materialises in slow GPU HBM memory.

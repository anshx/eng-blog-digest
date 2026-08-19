---
title: "GQA: Training Generalized Multi-Query Transformer Models from Multi-Head Checkpoints"
source: https://arxiv.org/abs/2305.13245
author: Joshua Ainslie, James Lee-Thorp, Michiel de Jong, Yanqi Zhou, Sumit Sanghai, Fei Sha
company: Google Research
date_posted: 2023-05-22
date_digested: 2026-08-19
---

# GQA: Training Generalized Multi-Query Transformer Models from Multi-Head Checkpoints

## What's new to learn

1. **KV cache bandwidth bottleneck** — Autoregressive LLM decoding is memory-bandwidth-bound, not compute-bound. Every forward pass must reload all past keys and values from GPU HBM; reducing that cache directly cuts latency and enables longer contexts or larger batches on the same hardware.

2. **Grouped Query Attention (GQA)** — A parameterized generalization where h query heads are divided into G groups, with each group sharing one K,V head pair. MHA (G=h independent heads) and MQA (G=1 fully shared) are the extremes; GQA sweeps the quality/bandwidth trade-off continuously.

3. **Mean-pool uptrain** — You can convert a pretrained MHA checkpoint to GQA by mean-pooling the K,V projection matrices within each group and then fine-tuning for ~5% of original pretraining compute — far cheaper than training GQA from scratch.

## Prerequisites

- **Multi-head self-attention (MHA)**: Q, K, V projections, scaled dot-product attention, concatenation and output projection.
- **KV cache**: during autoregressive generation, past keys and values are cached layer by layer to avoid recomputing them for the growing context.
- **GPU memory hierarchy**: HBM (high-bandwidth memory, the off-chip DRAM on the GPU) vs. on-chip SRAM. Most ML benchmarks focus on FLOPs, but memory bandwidth — bytes per second the GPU can read from HBM — is often the real bottleneck during inference.
- **Prefill vs. decode**: prefill processes all input tokens in parallel (compute-bound); decode generates tokens one at a time by attending to the full KV cache (bandwidth-bound). GQA targets decode.

## The core idea

Standard multi-head attention carries h independent key and value projections per layer. During training, you run attention in parallel over the whole sequence — no problem. During inference, every new token must do a forward pass that reads all h key heads and all h value heads for every past position. That's bandwidth, not compute, and for large models with long contexts it becomes the dominating cost.

Multi-query attention (Shazeer, 2019) goes to the extreme: one K head and one V head, shared by every query head. The KV cache shrinks h×, inference speeds up dramatically. But sharing so aggressively hurts quality — one pair of K,V cannot represent the range of what h different heads previously learned.

GQA is the interpolation. Divide the h query heads into G groups (1 ≤ G ≤ h). Within each group, the h/G query heads share exactly one K head and one V head. Set G=1 and you get MQA; set G=h and you recover MHA. In practice, G=8 for a 32- or 64-head model yields near-MHA quality with near-MQA bandwidth.

The reason you can get away with this: empirically, attention heads within a large MHA model naturally cluster into a small number of behavioral types (positional heads, syntactic heads, local-context heads, long-range heads). Forcing explicit groups approximates that natural redundancy.

## Mechanics

**Standard MHA attention for head i:**
```
Q_i  = X W_Q^i   ∈ R^{seq × d_k}
K_i  = X W_K^i   ∈ R^{seq × d_k}
V_i  = X W_V^i   ∈ R^{seq × d_v}

head_i = softmax(Q_i K_i^T / √d_k) V_i
```
KV cache per layer: `2 × h × L × d_k` floats (h heads, L cached tokens, d_k head dimension).

**GQA attention:**
```
Query heads:    Q_1 ... Q_h        (h distinct W_Q projections — unchanged)
KV groups:      (K_1, V_1) ... (K_G, V_G)   (G distinct W_K, W_V pairs)

head_i uses group g = floor(i · G / h):
head_i = softmax(Q_i K_g^T / √d_k) V_g
```
KV cache per layer: `2 × G × L × d_k` floats — a G/h fraction of MHA's footprint.

**Concrete numbers — Llama 2 70B (80 layers, h=64, G=8, d_k=128, fp16):**
| | KV cache at L=4096 |
|---|---|
| MHA (G=64) | ~10.7 GB |
| GQA (G=8) | ~1.3 GB |
| MQA (G=1) | ~167 MB |

The 8× GQA reduction means 70B can serve longer contexts on the same GPU, or batch more requests, without any change to model quality on most benchmarks.

**The uptrain recipe (converting an existing MHA checkpoint):**

1. **Initialize GQA K,V projections by mean-pooling**: for group g containing heads i₁ ... i_{h/G}:
   ```
   W_K^g = mean(W_K^{i_1}, ..., W_K^{i_{h/G}})  ∈ R^{d_model × d_k}
   W_V^g = mean(W_V^{i_1}, ..., W_V^{i_{h/G}})  ∈ R^{d_model × d_v}
   ```
2. **Keep all Q projections unchanged.**
3. **Uptrain on pretraining data for ≈5% of original steps.** The model re-adapts its Q projections to work with the now-coarser K,V representations.

Mean-pooling is a principled initialization: it preserves the "average attention target" of the group while discarding within-group head-to-head variation. Random initialization or truncation both perform worse as starting points before uptraining.

**Which heads go in which group?** The paper uses a simple index-based assignment — heads 0...(h/G)-1 form group 0, heads h/G...2(h/G)-1 form group 1, and so on. This is arbitrary but works well because adjacent heads in an MHA layer tend to be similar (they are initialized similarly and updated by similar gradient signals).

**Deployed examples:**
- Llama 2 70B: h=64, G=8 — trained with GQA from scratch
- Llama 3 (all sizes): GQA
- Mistral 7B: h=32, G=8
- Gemma: GQA
- Gemini (internal details unconfirmed but consistent with GQA)

## Where it breaks

**Quality floor at small G**: G=1 (MQA) can meaningfully regress, especially on long-context reasoning tasks where different heads need to capture different positional relationships. The paper shows GQA G≥4 (out of h=32) is usually within noise; G=1 is noticeably below.

**Uptrain cost is not free**: 5% of pretraining on a 70B model is still many GPU-days. Teams training from scratch can bake GQA in from the start and skip the uptrain. The uptrain path is primarily valuable for salvaging existing MHA checkpoints.

**Grouping is not learned**: the index-based grouping is a heuristic. There is likely a better grouping derivable from head similarity analysis (e.g., cluster heads by cosine similarity of their W_K matrices), but the paper does not explore this.

**Sensitive tasks regress more**: any task that requires fine-grained distinctions among many context positions — multi-hop QA over dense passages, needle-in-a-haystack retrieval — tends to show the largest gap between GQA and MHA, because those tasks most benefit from the diverse attention patterns MHA provides.

**Does not reduce prefill cost**: GQA does not change how much compute the prefill phase requires. FlashAttention targets prefill. GQA targets decode. They are complementary and both are used together in production (Llama 2 uses both).

## Why it works

The underlying principle is **amortized state loading via structured sharing**: when some data structure is expensive to load (the K,V tensors from HBM) and multiple independent computations (query heads) need it, sharing a copy across a group amortizes the load cost across the group's members — as long as the group members need similar enough data.

The same pattern appears throughout systems:

- **CPU cache set-associativity**: a direct-mapped cache (one slot per set, G=1) wastes capacity on conflicts; a fully-associative cache (G=h) is too expensive to search; N-way set-associative caches are the practical sweet spot — exactly the structure of GQA.

- **Database partitioned aggregation**: instead of one global accumulator (MQA), partition rows into G hash buckets each with its own state. Each bucket accumulates h/G rows. The partition → aggregate → merge pattern is the same as GQA's group → attend with shared K,V → concatenate.

- **SIMD vectorization**: one instruction (the K,V load) applied to multiple data streams (query heads within a group) — the hardware analogue of GQA's grouped sharing.

- **Peer-to-peer caching in CDNs**: instead of one global origin (MQA) or per-city caches (MHA), regional PoPs (GQA groups) cache content for the cities in their region. Each city still makes independent requests (Q heads) but shares the regional cache (K,V pair).

The attention-head level intuition: during training, heads naturally develop some redundancy in what they attend *to* (K,V) while maintaining diversity in what they *ask* (Q). GQA makes this implicit redundancy explicit and pays a structural cost only at initialization, which uptraining erases.

## Going deeper

1. **Shazeer, "Fast Transformer Decoding: One Write-Head is All You Need" (2019)** — the original MQA paper; sharply analyzes the bandwidth bottleneck and introduces the one-KV-head idea. Short (6 pages) and worth reading before GQA. arXiv:1911.02150

2. **Dao et al., "FlashAttention: Fast and Memory-Efficient Exact Attention with IO-Awareness" (2022)** — the complementary optimization: tiled SRAM computation for prefill. Already in this archive at posts/2026-05-26/. The two papers attack different phases of the same problem.

3. **Kwon et al., "Efficient Memory Management for Large Language Model Serving with PagedAttention" (2023)** — addresses KV cache *allocation* (fragmentation, sharing between requests) while GQA addresses KV cache *size* (per-head bandwidth). Together with GQA and FlashAttention, PagedAttention completes the KV cache optimization stack. Already in this archive at posts/2026-05-30/.

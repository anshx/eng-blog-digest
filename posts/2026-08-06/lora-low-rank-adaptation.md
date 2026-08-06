---
title: "LoRA: Low-Rank Adaptation of Large Language Models"
source: https://arxiv.org/abs/2106.09685
author: Edward J. Hu, Yelong Shen, Phillip Wallis, Zeyuan Allen-Zhu, Yuanzhi Li, Shean Wang, Lu Wang, Weizhu Chen
company: Microsoft
date_posted: 2021-06-17
date_digested: 2026-08-06
---

# LoRA: Low-Rank Adaptation of Large Language Models

## What's new to learn

1. **Intrinsic rank of fine-tuning updates**: When you fine-tune a large model, the weight update matrices ΔW that gradient descent learns are empirically low-rank — they concentrate most of their "information" in far fewer dimensions than the full d×k matrix. You don't need to train all d×k parameters; you only need to discover the small subspace they live in.

2. **Low-rank matrix factorization as a training primitive**: Parameterizing ΔW = BA (where B ∈ ℝ^{d×r} and A ∈ ℝ^{r×k}, with r ≪ min(d, k)) and training only B and A achieves full fine-tuning quality with 10,000× fewer trainable parameters for GPT-3 175B.

3. **Zero-cost adaptation at inference**: Because B and A are constant after training, you can pre-compute and fold ΔW = (α/r)BA back into the original weight matrix W before deployment — so LoRA adapters add exactly zero latency, zero extra memory, and zero extra compute at inference compared to the base model.

## Prerequisites

- How transformer attention works: query/key/value projections (Wq, Wk, Wv, Wo) and feed-forward sublayers.
- What fine-tuning is: starting from a pre-trained model and continuing gradient descent on a downstream task, updating all weights.
- Basic linear algebra: rank of a matrix, matrix multiplication, SVD (helpful but not required).
- Why full fine-tuning is expensive: storing a separate copy of all 175B weights for every task variant is memory-prohibitive; doing the backward pass through all layers requires proportionally more GPU memory than inference.

## The core idea

Full fine-tuning learns a dense update ΔW ∈ ℝ^{d×k} for each weight matrix in the model. LoRA's hypothesis: *these updates have low intrinsic rank*. That is, you don't need to freely adjust all d×k entries — the update that gradient descent would eventually converge to can be well-approximated by a matrix of rank r, where r is perhaps 4, 8, or 16, regardless of whether d and k are 1024, 4096, or 12288.

If that's true, then you can **parameterize the update as a product of two small matrices**:

```
ΔW = B · A
```

where B ∈ ℝ^{d×r} and A ∈ ℝ^{r×k}. Instead of training d×k parameters, you train d×r + r×k parameters — a reduction by a factor of roughly min(d, k) / (2r). At rank r = 8 with d = k = 4096, that's 512× fewer parameters for that layer alone.

The full modified weight during training is:

```
W_eff = W + (α/r) · B · A
```

where W is the frozen pre-trained weight and α is a scaling hyperparameter (typically set equal to r, making the effective scale factor 1, though it can be tuned). The `α/r` term normalizes the contribution so that changing r doesn't require retuning α.

**Initialization**: B is initialized to all zeros; A is initialized with random Gaussian values. This means ΔW = 0 at the start of training — the model starts from exactly the pre-trained checkpoint and learns deviations from there, preventing any abrupt distribution shift in the first forward pass.

**Where to apply**: LoRA is applied to the attention projection matrices — Wq and Wv (query and value) typically, sometimes all four of Wq, Wk, Wv, Wo. Feed-forward layers can also be adapted but the original paper found attending to query+value gave the best quality-per-parameter tradeoff.

## Mechanics

### The training loop

1. Load the pre-trained model. **Freeze all original weights** (set `requires_grad=False`).
2. For each target weight matrix W, inject two trainable matrices B (zeros) and A (Gaussian).
3. Replace the forward pass: `output = x @ W.T + x @ A.T @ B.T * (alpha / r)`.
4. Accumulate gradients only for A and B. The backward pass never touches W — so the optimizer states (Adam's m and v) exist only for A and B, not for W.
5. At end of training, materialize the merged weights: `W_merged = W + (alpha / r) * B @ A`. The merged model is identical in structure to the original — no extra modules remain.

### Parameter budget for GPT-3 175B

For GPT-3 (d_model = 12288, 96 attention layers, 4 attention matrices per layer):

- Rank r = 4, adapting Wq and Wv only:
  - Per matrix: 12288×4 + 4×12288 = 98,304 parameters
  - Total added: 2 matrices × 96 layers × 98,304 ≈ 18.9M parameters
  - As fraction of 175B: **0.01%**

- GPU memory: only A, B, their optimizer states, and activations for the LoRA path need to be held in training memory. The 175B base weights are frozen and can be kept in lower-cost storage. This trims training-time GPU memory by roughly **3×**.

### Comparison to adapter methods

Earlier parameter-efficient methods (e.g., Houlsby et al. 2019 adapters) insert small bottleneck MLP modules *sequentially* inside each transformer layer. This adds latency during inference: the adapter's computation sits on the critical path of every forward pass, and its small matrices can't be fused efficiently into large matrix multiplications.

LoRA's key architectural advantage: since ΔW is a low-rank matrix of the same shape as W, it can be **added** to W before inference rather than applied sequentially. The two paths (W and ΔW) run in parallel during training and merge into one at inference — zero added depth, zero added sequential operations.

### Serving multiple adapters from one base

Because the base weights W are frozen and shared, a single GPU can host the 175B base once and hot-swap LoRA adapters (just 18M–50M parameters each) for different tasks or customers. Systems like S-LoRA (2023) formalize this: they store hundreds of LoRA adapters in CPU memory and page them to GPU on demand, serving thousands of concurrent fine-tunes from a single base model deployment at 4× higher throughput than naive per-adapter deployment.

## Where it breaks

**Rank is a fixed hypothesis**: You choose r before training. If the actual update truly needs higher rank (e.g., for tasks very far from pre-training, like domain-specific code or structured output formats), low-rank LoRA can underfit. The paper shows quality starts declining below r = 1 for most tasks but this bound isn't universal.

**Transformer layers only, loosely**: LoRA is applied to weight matrices used in linear projections. Embedding layers and layer norm parameters are typically excluded. Some fine-tuning scenarios (e.g., adjusting the model's tokenization behavior) require updating these excluded layers.

**Not equivalent to full fine-tuning on distribution shifts**: A 2024 paper ("LoRA vs Full Fine-tuning: An Illusion of Equivalence") found that LoRA adapters and full fine-tuning span orthogonal subspaces of the weight space. LoRA pushes the model's representations toward the new task without forgetting pre-training, while full fine-tuning can consolidate representations more deeply. For tasks requiring large behavioral shifts, the gap can be significant.

**The "adapter collapse" failure mode**: If α is set too high or r too large relative to dataset size, LoRA can overfit before the base model's pre-trained knowledge stabilizes in the new context. Choosing r = 8 and α = 16–32 is empirically safe across most tasks.

**No free lunch on expressivity**: LoRA reduces trainable parameters by restricting ΔW to a low-rank subspace. The pre-trained W's null space remains unexplorable. For tasks requiring knowledge the pre-trained model has *no* latent representation of, you still need more data or higher rank — and eventually full fine-tuning.

## Why it works

The deeper principle is that **the fine-tuning problem has intrinsically lower dimensionality than the weight space suggests**.

Aghajanyan et al. (2020) first measured this empirically: for NLP tasks like SST-2 and MNLI, they found that full fine-tuning on BERT and RoBERTa can be matched by optimizing in a randomly-chosen 200-dimensional subspace of the full 125M-parameter weight space. The model doesn't need 125M degrees of freedom — it needs far fewer to steer its behavior.

LoRA takes this further: instead of a *random* low-dimensional subspace, it learns the subspace *structure* directly as part of training. The matrices A and B together define a specific rank-r subspace of ℝ^{d×k} — gradient descent finds which low-rank direction in weight space produces the desired output shift.

**The SVD lens**: Every matrix ΔW can be written as a sum of rank-1 outer products: ΔW = Σᵢ σᵢ uᵢ vᵢᵀ (its SVD). If most singular values σᵢ are near zero, the update is dominated by a few principal directions. LoRA is the hypothesis that fine-tuning updates have a fast-decaying singular value spectrum — and the trained BA decomposition is an approximation to the top-r singular components of the "ideal" update matrix.

This is the same insight as:
- **PCA**: most variance in data concentrates in a few principal components. You keep only those, discard the rest.
- **Matrix factorization for collaborative filtering**: a 10M-user × 1M-item rating matrix has rank ≈ 50 because user preferences and item properties live in a much smaller latent-factor space.
- **JPEG's DCT compression**: the 8×8 pixel blocks are well-approximated by a few low-frequency cosine components; most coefficient energy is in the top-left corner.
- **Low-rank signal recovery (compressed sensing)**: if the signal is known to be sparse in some domain, you need far fewer measurements than Shannon-Nyquist would suggest.

The common structure: the "object" you're representing (a fine-tuning update, a user-item interaction, an image patch) is **empirically constrained to a low-dimensional manifold** inside the full parameter space. The efficient algorithm exploits that constraint by working directly in the low-dimensional parameterization.

**Why pre-trained models specifically enable this**: A pre-trained LLM has already learned rich feature representations. Fine-tuning doesn't need to *build* representations from scratch — it needs to *select* and *reweight* existing ones. That selection is a much lower-dimensional operation than the full weight space. LoRA makes this structural assumption explicit and exploits it by design.

## Going deeper

1. **Intrinsic Dimensionality (Aghajanyan et al., 2020)** — [arXiv:2012.13255](https://arxiv.org/abs/2012.13255). The empirical foundation for LoRA's hypothesis: demonstrates that NLP fine-tuning can be done in random low-dimensional subspaces of parameter space, measuring the "intrinsic dimension" of various tasks.

2. **QLoRA (Dettmers et al., 2023)** — [arXiv:2305.14314](https://arxiv.org/abs/2305.14314). Combines LoRA with 4-bit quantization of the base weights (NF4 format). Fine-tunes a 65B model on a single 48 GB GPU. Shows that quantization and low-rank adaptation compose without quality loss, enabling consumer-hardware fine-tuning of frontier models.

3. **S-LoRA: Serving Thousands of Concurrent LoRA Adapters (Sheng et al., 2023)** — [arXiv:2311.03285](https://arxiv.org/abs/2311.03285). System engineering for multi-tenant LoRA deployment: unified paging for adapter weights, custom CUDA kernels for batched LoRA inference across heterogeneous rank configurations. The infrastructure counterpart to the training paper.

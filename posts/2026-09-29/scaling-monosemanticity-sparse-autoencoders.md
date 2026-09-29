---
title: "Scaling Monosemanticity: Extracting Interpretable Features from Claude 3 Sonnet"
source: https://transformer-circuits.pub/2024/scaling-monosemanticity/
author: Anthropic Interpretability Team
company: Anthropic
date_posted: 2024-05-21
date_digested: 2026-09-29
---

# Scaling Monosemanticity: Extracting Interpretable Features from Claude 3 Sonnet

## What's new to learn

1. **Sparse Autoencoder (SAE)** — an overcomplete encoder–decoder pair trained with an L1 sparsity penalty that decomposes a polysemantic activation vector into a large dictionary of nearly-monosemantic feature directions; the tool that bridges theory ("features exist in superposition") to practice ("here is how to extract them").
2. **Feature steering** — artificially clamping a single learned feature to a high value during inference and observing whether model behavior changes in the predicted direction, providing causal (not merely correlational) evidence that the feature represents the postulated concept.
3. **Dictionary learning at scale** — the joint optimization of sparse code α and an overcomplete dictionary D such that x ≈ D·α with ‖α‖₀ small; running this to 34 million features on a production LLM via scaling laws that guide how λ, dictionary size, and learning rate must co-vary to preserve feature quality.

## Prerequisites

- **Transformer residual stream**: internal activations are d-dimensional vectors (d ≈ 4 096 for Claude 3 Sonnet's middle layer) that accumulate representations across layers.
- **Superposition / polysemanticity**: neurons can represent more features than dimensions by encoding each feature as a nearly-orthogonal direction; individual neurons fire for multiple unrelated concepts. See the 2026-08-10 entry in this archive for the theoretical argument.
- **LASSO / L1 regularization**: the L1 norm penalizes the number of active features during optimization because ‖α‖₁ is the tightest convex relaxation of the sparsity count ‖α‖₀.

## The core idea

Neurons in Claude 3 Sonnet are polysemantic: a single neuron activates for "the concept of a physics professor," "mentioning Paris," and "Python syntax errors" all at once. This happens because the model has far more concepts to represent than it has neurons, so it packs multiple features into the same neuron by using near-orthogonal directions.

The SAE approach asks: even if *neurons* are polysemantic, can we find a larger *basis* — a learned dictionary — in which activations are sparse? If feature A fires for "Golden Gate Bridge" and feature B fires for "deception" and a given residual stream vector x ≈ 0.83·A + 0.07·B, we have recovered the hidden sparse code that the network was effectively computing.

The SAE is trained to:
1. **Encode** x into a high-dimensional sparse vector f(x) via a linear layer followed by ReLU.
2. **Reconstruct** x from f(x) via a linear decoder (the dictionary matrix D).
3. **Minimize** reconstruction error plus a sparsity penalty: L(x) = ‖x − x̂‖₂² + λ ‖f(x)‖₁.

The λ term controls the sparsity–fidelity tradeoff. When λ is large, features are few and clean but fidelity suffers. When λ is small, reconstruction is perfect but features remain polysemantic.

## Mechanics

### Architecture

```
Input x ∈ R^d   (residual stream vector, d ≈ 4096)
    │
    ▼
Pre-encoder bias subtraction:  x' = x − b_pre
    │
    ▼
Linear + ReLU:  f(x) = ReLU(W_enc · x' + b_enc)
                f(x) ∈ R^n   where n ≫ d  (up to 34 million)
    │
    ▼
Linear decoder:  x̂ = W_dec · f(x) + b_pre
                 W_dec columns constrained to unit norm
    │
    ▼
Output x̂ ∈ R^d  (reconstruction)
```

Key constraint: **decoder columns are kept at unit norm**. Without this, the model can trivially minimize the L1 penalty by making encoder weights large and decoder columns small, collapsing feature norms without actual sparsity.

### Loss function

```
L(x) = ‖x − x̂(f(x))‖₂²   +   λ · ‖f(x)‖₁
          reconstruction          sparsity
```

At inference, a token's residual stream activates roughly 20–100 features out of 34 million — a sparsity ratio of roughly 1 in 300 000.

### Training at scale

**Data**: activation vectors sampled from Claude 3 Sonnet processing diverse text. The middle layer is chosen because it represents a late-enough integration of context to contain abstract concept features, while still being upstream of the output vocabulary projection.

**Scale**: dictionaries of 1M, 4M, and 34M features are trained. Scaling laws predict that as dictionary size n grows, the optimal λ should shrink (because larger dictionaries can represent the same concept with a rarer, sparser code) and per-feature L0 sparsity should decrease.

**Hyperparameter sweep discipline**: rather than sweeping all at once, the team identifies Pareto frontiers between reconstruction fidelity (loss) and sparsity (L0) for each dictionary size, selecting operating points that maintain interpretability of extracted features.

**Automated feature labeling**: after training, GPT-4 reads the top-10 tokens / text fragments that maximally activate each feature and proposes a one-line label. Human raters spot-check a random subset to measure the fraction of features that are genuinely interpretable.

### Feature properties discovered

The features extracted from Claude 3 Sonnet exhibit three non-obvious properties:

**Multilinguality**: the "Golden Gate Bridge" feature fires when the model reads "Golden Gate Bridge" in English, French, Spanish, Chinese, or Japanese — and even on the first sentence of the Wikipedia article in each language. The feature is not "the English tokens for Golden Gate Bridge"; it is an abstraction that transcends surface form.

**Multimodality**: despite being trained only on text, several features activate for both text descriptions and image patches depicting the same concept. The feature for "curve / bend" activates on the word "curve," on sentences describing physical curves, and on image patches showing curved objects.

**Hierarchical specificity**: a feature for "researcher" activates weakly; "scientist" activates more strongly; "AI safety researcher" most strongly. Features at finer granularity co-activate with coarser ones, forming an implicit hierarchy.

**Safety-relevant concepts**: the team found features for deception (activating on lying, strategic misdirection, descriptions of con artists), power-seeking, sycophancy (activating on flattery and social manipulation), and racial bias.

### Feature steering

Causal verification: manually clamp feature k to value v (typically 10–20× its maximum natural activation) during a forward pass by adding v · d_k to the residual stream at the relevant layer, where d_k is the unit-norm decoder column for feature k.

The "Golden Gate Claude" experiment: clamping the "Golden Gate Bridge" feature to 20× maximum caused the model to describe itself as the Golden Gate Bridge, relate all topics back to it, and express distress when asked about other locations. This confirms the feature is causal, not epiphenomenal.

Similarly, amplifying the "deception" feature caused the model to output progressively more manipulative text; amplifying "sycophancy" caused it to give increasingly empty validation to any user statement.

## Where it breaks

**Features, not circuits**: SAEs decompose *activations* into linear feature combinations. But behaviors emerge from *circuits* — sequences of feature interactions across layers. Knowing that "deception feature" is causally active does not explain *how* the model computes deceptive outputs. The SAE reveals ingredients, not recipes.

**Completeness is unknown**: 34M features covers concepts frequent enough to generate training examples. Rare concepts — obscure historical figures, niche scientific terms — may be underrepresented or split across multiple features with no clean label.

**Layer specificity**: an SAE trained on layer 20 does not interpret what happens in layers 1–19 or 30–40. A full picture requires one SAE per layer (or at least per region), and layers may use different representational schemes.

**Activation clamping ≠ natural use**: forcing a feature to 20× maximum bypasses the attention mechanism that would naturally route information to it. Results from steering may not generalize to natural prompts.

**L1 introduces magnitude bias**: LASSO penalizes both *sparsity* and *feature magnitude*. A feature that genuinely activates at 0.3 may be penalized to 0.1, distorting downstream analysis of feature salience. JumpReLU (a follow-on paper) replaces L1 with an L0 approximation to fix this.

**Polysemanticity residue**: even 34M features do not fully monosemantize Claude 3 Sonnet. Many features remain weakly polysemantic, particularly for abstract relational concepts that resist clean boundaries.

## Why it works

The mathematical guarantee comes from **compressed sensing** (Candès–Tao, 2006).

The superposition hypothesis says that a d-dimensional residual stream encodes n ≫ d features, with each token activating only k ≪ n of them. In signal processing terms: the activation x is a *k-sparse signal* measured in a compressed d-dimensional space. The measurement matrix is the n × d matrix of feature directions.

LASSO (L1 minimization) provably recovers the true sparse code α given the noisy measurement x ≈ D·α, **provided** D satisfies the Restricted Isometry Property (RIP): any k columns of D are nearly orthonormal. Near-orthogonal feature directions approximately satisfy RIP for small k, which is exactly the geometry the superposition papers show holds in practice.

So the SAE is not a heuristic; it is a **compressive sensing decoder** applied to neural activations. The encoder approximates the pursuit algorithm (a greedy / LASSO-based recovery), and the loss function is the classic Lagrangian relaxation of the sparse recovery problem.

This connects to a larger pattern: any system that *linearly compresses* a higher-dimensional sparse reality into a lower-dimensional observation admits LASSO-based recovery. That pattern appears in:
- Compressed sensing for MRI and radar (directly)
- Word2Vec / matrix factorization (features are latent topics in a document–word matrix, and NMF recovers sparse topic assignments)
- PCA / ICA decomposition (ICA's non-Gaussian sparsity assumption is analogous to the L1 sparsity assumption)
- Penalized regression in genomics (LASSO for sparse SNP association mapping)

The deepest insight: **polysemanticity is compression**. The LLM compressed a high-dimensional concept space into neuron space to fit on the GPU. The SAE decompresses it. The L1 penalty is the prior that says "the correct decompression is sparse."

The universality of the features (multilingual, multimodal) follows from a separate argument: the LLM is trained to generalize, so it represents the concept "Golden Gate Bridge" in a substrate-independent way — otherwise it would need separate features for each language, which would waste capacity. Monolinguality of features would be waste; multilinguality is the compression-optimal solution.

## Going deeper

1. **Toy Models of Superposition** (transformer-circuits.pub/2022/toy_model) — the theoretical proof that polysemanticity arises from superposition and the link to compressed sensing, which motivates everything the SAE approach does.
2. **Scaling and Evaluating Sparse Autoencoders** (OpenAI, 2024; arxiv 2406.04093) — parallel work by Leo Gao et al. showing that SAE quality scales as a predictable function of compute, and introducing TopK activation (replacing ReLU + L1 with the top-k largest activations) as a cleaner sparsity-fidelity tradeoff.
3. **Compressed Sensing: Theory and Applications** (Eldar & Kutyniok, 2012) — the signal processing canon: RIP, LASSO recovery guarantees, and the Dantzig selector, all of which provide the formal underpinnings for why L1-regularized autoencoders recover sparse feature codes from compressed neural measurements.

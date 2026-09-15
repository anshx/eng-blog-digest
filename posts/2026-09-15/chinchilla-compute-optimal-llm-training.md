---
title: "Training Compute-Optimal Large Language Models"
source: https://arxiv.org/abs/2203.15556
author: Jordan Hoffmann, Sebastian Borgeaud, Arthur Mensch, et al.
company: DeepMind
date_posted: 2022-03-29
date_digested: 2026-09-15
---

# Training Compute-Optimal Large Language Models (Chinchilla)

## What's new to learn

- **Compute-optimal training**: For any fixed FLOPs budget there is a unique (model size, token count) pair that minimises loss — and it is far more data-heavy than the community assumed before 2022.
- **Balanced scaling of N and D**: Model parameters N and training tokens D should scale at equal rates with compute C (each roughly ∝ √C), yielding the "20 tokens per parameter" heuristic.
- **IsoFLOP profiling**: Fixing C and sweeping N — adjusting D = C/(6N) inversely — is the experimental design that cleanly isolates efficiency frontiers in training.

## Prerequisites

- Transformer decoder architecture and next-token prediction (cross-entropy) loss
- What "FLOPs" means in LLM training: approximately 6ND floating-point operations to train a model with N parameters on D tokens (2 flops/multiply-add × forward + backward ≈ 3 passes)
- The concept of empirical scaling laws: that LLM loss follows power-law relationships with model size and data size (from Kaplan et al. 2020)

## The core idea

Before Chinchilla, the field followed the Kaplan et al. (2020) scaling laws, which suggested that loss was much more sensitive to model size N than to training tokens D. The practical upshot: for a fixed compute budget, make the model as large as possible and train for as few tokens as you could get away with. This logic produced GPT-3 (175B params, 300B tokens), Gopher (280B params, 300B tokens), and PaLM (540B params, 780B tokens) — all enormous parameter counts trained on relatively thin data.

Chinchilla challenged this. The DeepMind team trained over 400 language models ranging from 70M to 16B parameters, spanning five decades of training-token counts from 5B to 500B, and systematically mapped the loss landscape across compute budgets. The question they asked was simple: for a given FLOPs budget C, what is the (N, D) pair that achieves the lowest validation loss?

Their answer: the optimal pair keeps D and N in approximately equal balance. For every doubling of model size, you should also double the number of training tokens. At the compute budgets current in 2022 (around 10²³ FLOPs), this corresponds to roughly **20 training tokens per model parameter**. Prior large models were off by a factor of 4–20× on the data side.

The empirical proof: Chinchilla, a 70B-parameter model trained on 1.4T tokens, consumed the same compute as Gopher (280B, 300B tokens) — but uniformly outperformed it on every benchmark tested, including a 7.5-point MMLU improvement (67.5% vs 60.0%) and state-of-the-art across common-sense reasoning, reading comprehension, and math.

## Mechanics

### The loss parametrisation

The paper fits loss as a sum of three components:

```
L(N, D) = E + A / N^α + B / D^β
```

where:

- **E** ≈ 1.69 nats — the irreducible entropy of the data; even a perfect model can't beat it
- **A / N^α** — the parameter-limited term: loss remaining because the model can't memorise the training distribution
- **B / D^β** — the data-limited term: loss remaining because the model hasn't seen enough examples

Fitted coefficients (Approach 3 / direct parametric fit): A = 406.4, B = 410.7, α = 0.34, β = 0.28.

The two decay exponents α ≈ 0.34 and β ≈ 0.28 are close but not identical, which turns out to be crucial.

### The compute constraint

Training an N-parameter transformer on D tokens costs approximately:

```
C ≈ 6 N D   FLOPs
```

This is the binding budget constraint. Given C, you must choose a point on the hyperbola D = C / (6N).

### Finding the optimal (N*, D*)

Minimise L(N, C/(6N)) over N. This is a constrained optimisation — you can state it as a Lagrangian:

```
minimise   E + A/N^α + B/D^β
subject to 6ND = C
```

Setting the partial derivatives equal via the Lagrange multiplier:

```
∂L/∂N :  α A / N^(α+1) = λ · 6D
∂L/∂D :  β B / D^(β+1) = λ · 6N
```

Dividing the two conditions to eliminate λ:

```
α A · N · D^(β+1)  =  β B · D · N^(α+1)
α A · D^β  =  β B · N^α
(D/N)^β/α  =  (βB) / (αA)     [not exact but illustrative]
```

More precisely, the ratio of exponents governs how D and N scale with C:

```
N* ∝ C^(β / (α+β))   ≈  C^0.452
D* ∝ C^(α / (α+β))   ≈  C^0.548
```

With α = 0.34 and β = 0.28, both exponents are close to 0.5 — hence "equal scaling". The oft-quoted 20:1 token-to-parameter ratio is this ratio evaluated at the compute budgets current in 2022:

```
D*/N* = (C* / 6)^((α-β)/(α+β))  ×  constant
```

This ratio is *slowly increasing* with C, so at larger compute (10²⁵+ FLOPs) the optimal ratio is somewhat higher than 20; at smaller compute it is lower.

### Three estimation approaches

The paper triangulates the result three ways:

1. **IsoFLOP profiles**: For each of several fixed compute budgets (6×10¹⁸ to 3×10²¹ FLOPs), train 5–9 models with N swept across two orders of magnitude, D adjusted as D = C/(6N). Fit a parabola to loss-vs-N in log-space; the minimum gives N*(C). This is the most direct approach.

2. **Global parametric fit**: Fit L(N, D) = E + A/N^α + B/D^β to all runs simultaneously using Adam optimisation with Huber loss (robust to outlier runs). Then analytically solve for the optimal (N*, D*) given any C.

3. **Direct power-law fit of the frontier**: Extract the observed (N*, D*) from Approach 1 across all compute budgets; fit N* = G₁ C^a and D* = G₂ C^b as power laws.

All three approaches agree within a factor of ~2 on the optimal token-to-parameter ratio across the range studied.

### Validation: Chinchilla vs. the field

| Model | Parameters | Tokens | MMLU |
|-------|-----------|--------|------|
| Gopher | 280B | 300B | 60.0% |
| **Chinchilla** | **70B** | **1.4T** | **67.5%** |
| Expert human | — | — | 89.8% |
| GPT-3 | 175B | 300B | 43.9% |

Chinchilla uses the same compute budget as Gopher, has 4× fewer parameters (cheaper inference), and outperforms it on every evaluated benchmark. It surpassed human expert performance on MMLU for the first time among language models.

## Where it breaks

**Inference cost is ignored.** Chinchilla minimises *training* loss per FLOPs. For a model deployed at massive scale, serving cost matters more than training cost. Meta's LLaMA series deliberately over-trains smaller models: LLaMA 3 8B used ~15T tokens (≈1,900 tokens/parameter), far beyond Chinchilla-optimal for training, because a smaller model is cheaper to run for billions of inference requests.

**The 20× ratio shifts with scale.** Epoch AI's replication (2022) found inconsistencies between the three approaches in the original paper and estimated the actual Chinchilla-optimal ratio might be 40–100× tokens per parameter at very large compute budgets (10²⁵ FLOPs). The community has converged on 50–100× in practice.

**Data quality is not modelled.** The scaling laws were fit on MassiveText, a Web-scraped corpus. High-quality curated datasets (The Pile, FineWeb, Dolma) shift the effective data efficiency; you may need fewer tokens to achieve the same loss on a cleaner dataset.

**Downstream task scaling diverges from loss scaling.** Cross-entropy loss on next-token prediction is a proxy for downstream performance. Some capabilities (multi-step reasoning, code, math) show phase-transition-like improvements at certain model scales regardless of Chinchilla optimality. Loss scaling is a smooth power law; task performance is not always.

**The parametric form assumes separability.** L(N,D) = E + f(N) + g(D) treats parameter-scaling and data-scaling as independent. In reality, a larger model trained on less data may learn different representations than a smaller model trained on more — the two effects interact at the distribution level.

## Why it works

The deeper principle is the **equimarginal principle applied to complementary power-law terms**.

Any budget-constrained optimisation over a separable objective — L = f(x) + g(y) subject to xy = C — has the same structure. At the optimum, the marginal loss reduction per unit of added "budget" must be equal for both resources:

```
|∂L/∂x| / (∂(xy)/∂x)  =  |∂L/∂y| / (∂(xy)/∂y)
```

With power-law marginals (∂L/∂x ∝ x^-(α+1), ∂L/∂y ∝ y^-(β+1)), this reduces to D^β / N^α = constant, which dictates the exponents of the scaling law.

When α ≈ β, the marginal returns from parameters and tokens are equally steep, so the budget splits evenly: √C to each. When one exponent dominates — say α >> β, as Kaplan believed — the optimal allocation tilts heavily toward parameters. Chinchilla showed the exponents are in fact near-equal, overturning the earlier assumption.

This is precisely the same logic as:

- **Amdahl's Law**: Serial fraction limits parallel speedup; you should parallelise until the serial bottleneck is as expensive as the parallelisable part — i.e., balance the two additive terms.
- **OLTP/OLAP storage design**: When both read I/O and write I/O matter, optimal log-structured merge settings balance the read-amplification term and the write-amplification term. RUM conjecture (in this archive) is the same structure.
- **Cache sizing**: Allocating cache capacity vs. cache associativity has diminishing returns in each direction; the optimal split equates marginal miss-rate improvements.

The "20 tokens per parameter" heuristic is the ML equivalent of "80/20 train-test split" — an empirically calibrated, approximately-correct budget rule that results from power-law marginal returns.

The further insight that Chinchilla added to the field: **what you are optimising matters**. Compute-optimal training (minimise loss per training FLOP) is not the same as inference-optimal training (minimise loss per inference FLOP at deployment scale), which is not the same as parameter-optimal fine-tuning (minimise adaptation cost). Each objective has a different optimal resource allocation. The same equimarginal logic applies; only the constraint changes.

## Going deeper

1. **Kaplan et al. 2020 "Scaling Laws for Neural Language Models"** (arXiv 2001.08361): The predecessor that established power-law scaling, but with α >> β — the paper Chinchilla refuted. Essential context for understanding what changed and why.

2. **"Beyond Chinchilla-Optimal: Accounting for Inference in Language Model Scaling Laws"** (arXiv 2401.00448): Derives the inference-optimal scaling laws showing that for deployed models, the optimal token:parameter ratio is 40–400×, formalising the LLaMA intuition.

3. **Epoch AI "Chinchilla Scaling: A Replication Attempt"** (2022): Finds a bug in Approach 1 of the original paper and provides cleaner estimates of the coefficients. Essential for anyone using the laws to make actual training decisions.

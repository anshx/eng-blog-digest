---
title: "The Algorithmic Foundations of Differential Privacy"
source: https://www.cis.upenn.edu/~aaroth/Papers/privacybook.pdf
author: Cynthia Dwork and Aaron Roth
company: Microsoft Research / University of Pennsylvania
date_posted: 2014
date_digested: 2026-11-06
---

# The Algorithmic Foundations of Differential Privacy

## What's new to learn

1. **The ε-differential privacy definition** — A mechanism M satisfies ε-DP if adding or removing any single person's record changes the probability of any output by at most a factor of e^ε. This converts the vague notion of "doesn't leak too much" into a crisp, quantitative, adversary-agnostic guarantee.

2. **Global sensitivity** — For any numeric query f, the most any one person can affect the answer is Δf = max_{adjacent D, D'} |f(D) − f(D')|. Adding Laplace noise scaled to Δf/ε makes f private; the sensitivity is what you're calibrating the noise to, not the value itself.

3. **The composition theorem** — Running k independent ε-DP mechanisms on the same dataset costs kε of total privacy budget. Privacy budgets add, just like money spent from a wallet, enabling a principled accounting framework for complex analytics pipelines.

## Prerequisites

- Basic probability (probability distributions, expected value, Bayes' theorem)
- What a database "query" means: a function from a dataset to a number or set of records
- The idea of an adversarial model (someone observing outputs trying to infer inputs)
- Familiarity with Gaussian and Laplace distributions helps but is not required

## The core idea

The classic approach to privacy—aggregation, anonymization, suppression—fails because an adversary who knows everything about you except whether you're in the dataset can often deduce whether you are, just by running enough queries. Differential privacy sidesteps this by defining privacy at the mechanism level, not the data level.

The key idea: a **mechanism** (randomized algorithm) M provides ε-differential privacy if for any two datasets D and D' that differ by exactly one person's record, and for any set of possible outputs S:

```
Pr[M(D) ∈ S] ≤ e^ε · Pr[M(D') ∈ S]
```

The factor e^ε bounds by how much the output distribution can shift when any single person is added or removed. If ε=0 the output is completely independent of your data (perfect privacy, zero utility). If ε is large, the output shifts a lot with your record (high utility, low privacy). Real deployments use ε between 0.1 and 10 depending on sensitivity.

The beautiful consequence: **ε-DP is adversary-agnostic**. It doesn't matter what the attacker already knows about you, what other data they have, or how clever they are. The guarantee holds universally.

**Randomized response**: The simplest DP mechanism predates the formal definition by four decades. Warner (1965) used it in surveys: to estimate the fraction answering "yes" to a sensitive question, each respondent flips a coin privately. If heads, they answer truthfully; if tails, they flip their answer. The analyst can correct for the bias and recover the true frequency, while no individual answer is verifiably true or false. With the right flip probability tied to ε, this achieves ε-local differential privacy for each record.

## Mechanics

### The Laplace Mechanism

For any numeric query f with global sensitivity Δf, the Laplace mechanism outputs:

```
M(D) = f(D) + Lap(Δf / ε)
```

where Lap(b) is Laplace-distributed noise with scale b (mean 0, variance 2b²). The key insight: the noise scale depends on the **query's sensitivity** (how much one person can move the answer), not on n, not on the values, not on the domain. Count queries ("how many rows satisfy X?") have sensitivity 1. Sum queries over values in [0, 1] have sensitivity 1. The mean query has sensitivity 1/n — adding noise of scale 1/(nε) gives a much tighter result.

**Why this works**: For adjacent D, D' the ratio of probability densities at any output z is:

```
p(z | D) / p(z | D') = exp(-|z - f(D)| / b) / exp(-|z - f(D')| / b)
        ≤ exp(|f(D) - f(D')| / b) ≤ exp(Δf / b) = e^ε
```

The Laplace density's exponential decay is precisely what makes the DP inequality tight.

### The Gaussian Mechanism and (ε, δ)-DP

The Laplace mechanism only works for ε ≤ ∞. For high-dimensional outputs (e.g., gradient vectors in machine learning), adding i.i.d. Laplace noise to each coordinate explodes variance. The Gaussian mechanism relaxes the guarantee to **(ε, δ)-DP**: the privacy guarantee holds with probability 1 − δ of not failing catastrophically:

```
M(D) = f(D) + N(0, σ²I)      where σ = Δ₂f · √(2 ln(1.25/δ)) / ε
```

Here Δ₂f is the L₂ sensitivity. The δ parameter is usually set to 1/n² so the failure probability is negligible; it must never be larger than 1/n or the mechanism is effectively broken.

### The Exponential Mechanism

For non-numeric output spaces (choosing an item from a set, selecting a model, responding to a query with a category), the **exponential mechanism** is the analog of the Laplace mechanism:

```
Pr[M outputs r] ∝ exp(ε · u(D, r) / (2Δu))
```

where u(D, r) is a "utility score" for output r on dataset D, and Δu is its sensitivity. This outputs the highest-scoring response with high probability while exponentially penalizing lower-scoring responses. The 2Δu denominator ensures the DP condition.

### Sequential and Parallel Composition

**Sequential composition**: Any sequence of k mechanisms each satisfying ε_i-DP together satisfies (Σ ε_i)-DP. Privacy budgets add like dollars: publish k statistics at ε each and you've spent kε total. This is both a bound and a warning—naively running 1000 ε=0.01 queries costs ε=10.

**Parallel composition**: If the k mechanisms operate on *disjoint* subsets of the dataset, the total cost is max(ε_i), not the sum. A histogram that partitions records into buckets and counts each bucket independently costs ε per bucket, not k·ε — each person appears in exactly one bucket, so their inclusion only affects one mechanism's output.

**Advanced composition and moments accountant**: Sequential composition is loose for many small-ε queries. The moments accountant (Abadi et al., 2016) tracks the cumulant-generating function (moments) of the privacy loss random variable and can show, e.g., that 10,000 gradient steps at ε=0.01 each cost only ε≈1 overall rather than ε=100. This technique makes differentially private SGD (DP-SGD) practical.

### Local vs. Central Differential Privacy

**Central DP**: A trusted curator collects raw records, runs M(D), and publishes the output. Each individual trusts the curator not to misuse raw data. Noise scales with 1/ε (excellent utility).

**Local DP**: Each individual applies a randomized mechanism to their own record before sending it. No trust in the curator required—not even a compromised server can violate privacy. The cost: noise scales with 1/(ε√n) to achieve the same accuracy, requiring n times more individuals for the same utility.

Apple uses local DP for telemetry (emoji frequency, Safari extension usage). Google uses local DP via RAPPOR (Randomized Aggregatable Privacy-Preserving Ordinal Response) for Chrome URL data, using two rounds of randomized response combined with Bloom filter encoding to handle high-cardinality domains.

### DP-SGD: Private Machine Learning

Training a neural network privately with DP-SGD (Abadi et al., 2016):

1. **Per-example gradient clipping**: Clip each training example's gradient to have L₂ norm ≤ C. This bounds the sensitivity of the sum: one example can change the gradient by at most 2C.
2. **Noise injection**: Add N(0, σ²C²I) Gaussian noise to the sum of clipped gradients before averaging.
3. **Privacy accounting**: Track the total (ε, δ) cost across all training steps using the moments accountant.

The clipping + noise pattern is universal: it's how DP is applied to any iterative algorithm. The clipping step bounds sensitivity; the noise step implements the Gaussian mechanism.

## Where it breaks

**Utility vs. privacy at small n**: The noise magnitude scales with sensitivity/ε, which is dataset-size-independent. For small datasets (n < 1000) the noise often swamps the signal entirely. DP is fundamentally better suited to large-scale aggregate statistics than to small studies.

**Group privacy degrades**: ε-DP for individuals implies (k·ε)-DP for groups of size k. If you want a family's data to be private as a group, you've already spent the budget k times over. Correlated records (spouses, families, same-employer) create implicit group privacy violations.

**The δ failure probability is easy to misuse**: (ε, δ)-DP with δ=0.01 is nearly meaningless—there's a 1% chance of complete privacy loss. δ must be cryptographically small (1/n² or smaller) to maintain the guarantee in practice. Many deployed systems get this wrong.

**ε calibration is an art**: What is an acceptable ε? There is no consensus. Apple uses ε=1–8 per day per metric. The US Census used ε=19.61 for the 2020 decennial census (later criticized as too high). Unlike, say, AES-256 which has a universally accepted security level, DP ε values are context-dependent and hard to compare across deployments.

**Local DP requires far more data**: To estimate a frequency with standard error σ in central DP requires n ≥ 1/(σ²ε²). In local DP (randomized response), the variance is roughly e^ε/(e^ε−1)² ≈ 1/ε² per person (at small ε), so to achieve error σ you need n ≥ 1/(σ²ε²) · e^ε/(e^ε−1)² — roughly e^(2ε) times more data. At ε=1 that's about 7× more. At ε=0.1, about 400× more.

**Differential privacy does not prevent reconstruction from multiple correlated queries**: The composition theorem guarantees that the total privacy loss is bounded by the sum of individual ε values, but sophisticated reconstruction attacks (like those used to show the census k-anonymity was broken) can still work if the analyst runs enough queries and the ε per query is large.

## Why it works

The deeper principle: **ε-DP is a bound on the log-likelihood ratio of any event between adjacent datasets**. Equivalently, it bounds the max-divergence D_∞(M(D) || M(D')) ≤ ε. Max-divergence is the strongest divergence measure — tighter than KL divergence, total variation, or Rényi divergence. So ε-DP implies all the weaker notions of distributional closeness.

This connects DP directly to **hypothesis testing**: the optimal test distinguishing "did the analyst observe M(D) or M(D')?" has advantage at most (1 − e^{-ε})/2. So ε-DP means: even the optimal adversary running the best possible distinguishing attack learns essentially nothing about whether a specific person's record was in the dataset. This is what "adversary-agnostic" means precisely.

The connection to **information theory**: ε-DP implies that the mutual information I(X; M(D)) ≤ ε (roughly), where X is a single record. The mechanism leaks at most ε bits of information about any individual in the worst case, which is the information-theoretic formulation of privacy.

The analogy that crystallizes it: **DP is to privacy what ACID is to transactions**. ACID isolates transactions from each other without specifying how (locking, MVCC, etc.). DP isolates an individual's privacy from analytics without specifying the query type or the adversary's side knowledge. Both are interface guarantees that compose and can be implemented multiple ways, but that give strong semantics at the boundary.

## Going deeper

1. **RAPPOR: Randomized Aggregatable Privacy-Preserving Ordinal Response** (Erlingsson et al., 2014, arXiv:1407.6981) — Google's local DP deployment for Chrome; covers the two-stage randomization that handles high-cardinality domains via Bloom filters and shows the utility-privacy tradeoff on real production data.

2. **Deep Learning with Differential Privacy** (Abadi et al., CCS 2016, arXiv:1607.00133) — Introduces DP-SGD and the moments accountant. The technique behind PyTorch Opacus, TensorFlow Privacy, and every company training private ML models.

3. **The U.S. Census Bureau's TopDown Algorithm** (Abowd et al., 2022, JPC) — The largest central-DP deployment: explains how the 2020 decennial census uses the geometric mechanism, the spine hierarchy, and post-processing constraints to satisfy ε=19.61 while preserving demographic summaries — the best real-world case study in the trade-off between utility and privacy at national scale.

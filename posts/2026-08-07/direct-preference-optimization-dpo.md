---
title: "Direct Preference Optimization: Your Language Model is Secretly a Reward Model"
source: https://arxiv.org/abs/2305.18290
author: Rafael Rafailov, Archit Sharma, Eric Mitchell, Stefano Ermon, Christopher D. Manning, Chelsea Finn
company: Stanford University
date_posted: 2023-05-29
date_digested: 2026-08-07
---

# Direct Preference Optimization: Your Language Model is Secretly a Reward Model

## What's new to learn

- **KL-constrained RL always has a Gibbs-distribution optimal policy.** When you maximize expected reward subject to a KL budget from a reference distribution, the solution is always π*(y|x) ∝ π_ref(y|x) · exp(r(y,x)/β). Most engineers know KL divergence; few know this closed form exists.
- **Inverting the optimal policy reveals an implicit reward.** Since the optimal policy determines the reward up to a per-prompt partition function Z(x), and Z cancels in pairwise comparisons, you can eliminate the reward model entirely—the language model encodes the reward implicitly.
- **The DPO loss.** Substituting the implicit reward into the Bradley-Terry preference model yields a binary cross-entropy loss over preference pairs that trains the policy directly, with no RL loop and no sampling from the model.

## Prerequisites

- Next-token language modeling (log-probability of a sequence is a sum over token log-probabilities).
- KL divergence: D_KL[π||π_ref] = E_π[log π(y) - log π_ref(y)]. Roughly, "how many nats does π deviate from π_ref on average?"
- Supervised fine-tuning (SFT): a starting checkpoint trained with standard cross-entropy on demonstration data.
- The Bradley-Terry model: a way to express a scalar preference probability from two scalar rewards—p(y_w > y_l) = σ(r(y_w) - r(y_l)).
- Basic PPO/RL vocabulary: policy, reward, trajectory—helpful but not required to follow the derivation.

## The core idea

The standard RLHF pipeline has three steps:

1. Collect human preference data: pairs (y_w, y_l) of responses to the same prompt x, where y_w is preferred.
2. Train a reward model: fit r_φ(x, y) to output high scores for preferred responses via the Bradley-Terry objective.
3. Fine-tune the LM with RL: maximize E_{y ~ π_θ}[r_φ(x, y)] subject to keeping π_θ near a reference policy π_ref (the SFT checkpoint), using PPO.

The pain is step 3: PPO requires generating responses from the current policy, scoring them with the reward model, computing advantages, and doing multiple gradient updates per batch. It's expensive, unstable, and hyperparameter-sensitive.

DPO asks: do we actually need the reward model at all?

The answer is no—because the constrained optimization in step 3 has a closed-form optimal solution, and that solution lets you express the reward directly in terms of the policy. Plug that expression into the Bradley-Terry model from step 2, and the reward model disappears. What remains is a classification loss you can minimize with plain backpropagation on the preference data.

## Mechanics

**The KL-constrained RL objective**

RLHF optimizes:

```
max_{π} E_{x~D, y~π}[r(x,y)]  −  β · D_KL[π(·|x) ∥ π_ref(·|x)]
```

The β term penalizes policies that drift too far from the reference. Without it, the model "reward hacks"—finds degenerate high-reward outputs that don't resemble natural text.

**Step 1: Find the closed-form optimal policy**

This is a standard result from variational inference. Writing out the Lagrangian and setting the functional derivative to zero gives:

```
π*(y|x)  =  π_ref(y|x) · exp(r(y,x)/β) / Z(x)
```

where Z(x) = Σ_y π_ref(y|x) · exp(r(y,x)/β) is the normalizing constant (the partition function). Z(x) ensures π* is a valid probability distribution, but is generally intractable to compute.

This is a Gibbs/Boltzmann distribution: responses with high reward get exponentially more probability mass relative to the reference. β is temperature—low β → nearly uniform over high-reward responses; high β → stays close to the reference.

**Step 2: Invert the equation to express the reward**

Rearranging for r(y,x):

```
r*(y,x)  =  β · log(π*(y|x) / π_ref(y|x))  +  β · log Z(x)
```

The reward decomposes into two terms: the log-ratio of optimal to reference policy, plus a prompt-dependent constant β·log Z(x).

**Step 3: Substitute into the Bradley-Terry preference model**

The probability that y_w is preferred over y_l is:

```
p*(y_w ≻ y_l | x)  =  σ(r*(y_w, x) − r*(y_l, x))
```

Substituting our expression for r* and computing the difference:

```
r*(y_w,x) − r*(y_l,x)
  =  [β·log(π*(y_w|x)/π_ref(y_w|x)) + β·log Z(x)]
   − [β·log(π*(y_l|x)/π_ref(y_l|x)) + β·log Z(x)]
  =   β·log(π*(y_w|x)/π_ref(y_w|x)) − β·log(π*(y_l|x)/π_ref(y_l|x))
```

The β·log Z(x) terms cancel exactly—they appear in both r*(y_w) and r*(y_l) since Z(x) depends only on the prompt, not the response. This cancellation is the crux of DPO: the partition function is the hard part of RL, and it simply vanishes when you compare two responses.

**Step 4: Replace π* with a trainable policy π_θ and maximize likelihood**

Instead of fitting a reward model, replace π* with π_θ and minimize the negative log-likelihood of the human preferences:

```
L_DPO(π_θ)  =  −E_{(x, y_w, y_l) ~ D} [
    log σ(
        β · log(π_θ(y_w|x) / π_ref(y_w|x))
      − β · log(π_θ(y_l|x) / π_ref(y_l|x))
    )
]
```

This is binary cross-entropy. The model is trained to increase the log-ratio for preferred responses and decrease it for rejected responses—exactly what step 2+3 of RLHF was trying to achieve, but without a separate reward model or RL.

**In practice**

```python
def dpo_loss(π_θ, π_ref, x, y_w, y_l, beta=0.1):
    # sequence log-prob = sum of token log-probs (auto-regressive)
    lp_w     = π_θ.log_prob(y_w | x)       # chosen under policy
    lp_l     = π_θ.log_prob(y_l | x)       # rejected under policy
    ref_lp_w = π_ref.log_prob(y_w | x)     # chosen under reference
    ref_lp_l = π_ref.log_prob(y_l | x)     # rejected under reference

    chosen_ratio   = lp_w - ref_lp_w       # log(π_θ/π_ref) for chosen
    rejected_ratio = lp_l - ref_lp_l       # log(π_θ/π_ref) for rejected

    return -F.logsigmoid(beta * (chosen_ratio - rejected_ratio)).mean()
```

Both π_θ and π_ref are forward passes through (often the same) model; π_ref is frozen. Training updates only π_θ. No sampling, no rollouts, no PPO clip ratio.

## Where it breaks

**Length bias.** The log-ratio of a response is a sum over token log-probabilities. Longer sequences accumulate more terms, so the model learns that generating longer responses increases chosen_ratio by default—even when length is irrelevant to quality. Benchmark scores can be inflated by response verbosity.

**Offline distribution shift.** DPO trains on static preference pairs collected from some data-generating policy (often the SFT checkpoint). If training drives π_θ far from that distribution, the preference labels become miscalibrated—a response that humans rated as "rejected" might look different under the new policy. PPO avoids this by continually sampling from π_θ.

**Overfitting on preference pairs.** A theoretical analysis (IPO, 2023) showed that DPO minimizing the log-σ loss doesn't actually converge to the KL-regularized optimum. The log-ratio can grow without bound, causing the model to concentrate probability mass on a handful of tokens that match the chosen responses rather than learning a smooth generalization.

**Reference model overhead.** DPO requires keeping π_ref loaded throughout training. At 70B parameters, that's another 140 GB of weights—a significant GPU memory overhead on top of the trainable π_θ.

**Multimodal preferences.** Bradley-Terry collapses human preferences to a single scalar. When different annotators have genuinely different ideals (some want brevity, some want detail), the model averages the signal, often learning nothing coherent.

## Why it works

**The deeper principle: every KL-constrained RL problem is secretly a variational inference problem.**

The connection is tight:
- In variational inference, the ELBO maximization yields a posterior q*(z) ∝ p(z) · exp(E_q[log p(x|z)]), a Gibbs distribution.
- In statistical mechanics, thermal equilibrium at temperature T is the distribution that maximizes entropy subject to fixed mean energy—also a Boltzmann/Gibbs distribution.
- In maximum entropy modeling, the distribution with highest entropy subject to a set of moment constraints is an exponential family—same form.

All three are instances of the same convex duality: "minimize divergence from prior subject to an expected-value constraint." The answer is always π*(y) ∝ π_prior(y) · exp(reward(y)/β). The β parameter is temperature, trading off between "stay near the prior" and "maximize reward."

DPO exploits the fact that in the pairwise comparison setting, the normalizing constant—the free energy of statistical mechanics, the evidence in variational inference—cancels. This is not specific to language models; it would apply anywhere you have a KL-constrained RL problem evaluated via pairwise preferences.

**A second framing: every LM is implicitly a reward model.**

Given a fine-tuned LM π_θ and a reference LM π_ref, you can always read off an implicit reward: r(y,x) = β · log(π_θ(y|x)/π_ref(y|x)). This isn't just a mathematical artifact—it means that SFT itself encodes preferences. A model fine-tuned to be "helpful and harmless" has, in a well-defined sense, learned a reward model. DPO makes this explicit and trains against it directly.

## Going deeper

1. **IPO — Identity Preference Optimization** (Azar et al., 2023, arxiv:2310.12036): Identifies that DPO's log-σ loss can overfit and proposes an MSE-based variant that converges to the true KL-regularized optimum. Shorter derivation, tighter theory.

2. **SimPO — Simple Preference Optimization** (Yu et al., 2024, arxiv:2405.14734): Drops the reference model entirely by using average log-probability (normalized by sequence length) as the implicit reward. Fixes length bias at the same time. Empirically competitive with DPO at fraction of the memory cost.

3. **GRPO — Group Relative Policy Optimization** (DeepSeek-R1, 2025, arxiv:2501.12599): Moves back toward online RL by sampling multiple completions per prompt, normalizing rewards within the group, and updating without a value model. Recovers the distribution-matching property of PPO with a loss that resembles DPO's pairwise objective. Used to train DeepSeek's reasoning models.

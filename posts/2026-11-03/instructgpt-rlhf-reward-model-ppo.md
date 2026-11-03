---
title: "Training language models to follow instructions with human feedback"
source: https://arxiv.org/abs/2203.02155
author: Long Ouyang, Jeffrey Wu, Xu Jiang, Diogo Almeida, Carroll Wainwright, Pamela Mishkin, Chong Zhang, Sandhini Agarwal, Katarina Slama, Alex Ray, John Schulman, Jacob Hilton, Fraser Kelton, Luke Miller, Maddie Simens, Amanda Askell, Peter Welinder, Paul Christiano, Jan Leike, Ryan Lowe
company: OpenAI
date_posted: 2022-03-04
date_digested: 2026-11-03
---

# Training language models to follow instructions with human feedback (InstructGPT)

## What's new to learn

1. **Bradley-Terry preference model as a reward-fitting objective** — Pairwise human preference data ("I prefer output A over B") can be fit to a scalar reward function by treating each comparison as a logistic regression sample: the probability human prefers A is σ(r(A) − r(B)). This makes training a reward model from preferences identical to fitting an Elo rating system, and gives you a differentiable proxy for human judgment.

2. **KL-regularized RL objective** — The correct formulation of RLHF is not "maximize reward" but "maximize reward subject to a KL-divergence penalty from the starting policy." Without the leash, the policy collapses to whatever single output happens to fool the reward model. The KL term is the only mechanism that preserves output diversity and prevents reward hacking.

3. **Alignment tax and the proxy-measure problem** — Fine-tuning for human preference slightly regresses performance on NLP benchmarks the model wasn't originally optimized for. This is Goodhart's Law at the training-objective level: the reward model is a proxy measure, and once it becomes the target, the model finds ways to score well on it that don't generalize. Mixing pretraining gradients back in (PPO-ptx) is the patch.

## Prerequisites

- Supervised fine-tuning / transfer learning (what it means to fine-tune a pretrained language model)
- Basic reinforcement learning: policy π, reward r, trajectory (sequence of tokens as a single "action" in a bandit setting)
- Cross-entropy loss and logistic regression
- The DPO archive entry (2026-08-07) connects directly here and is clarified by reading this first

## The core idea

Language models trained on next-token prediction are *misaligned*: they optimize for predicting internet text, not for being helpful, honest, or harmless. Scaling doesn't fix this — a bigger model is just better at predicting the internet.

RLHF (Reinforcement Learning from Human Feedback) corrects the objective in three stages:

1. **Supervised Fine-Tuning (SFT)** — Get human labelers to write demonstrations of the desired behavior on ~13,000 prompts. Fine-tune GPT-3 on those demonstrations with standard cross-entropy. This gives a model that knows *what good looks like* on the distribution of interest.

2. **Reward Model (RM) training** — For ~33,000 prompt-completion pairs, have labelers rank 4–9 completions from best to worst. Train a separate 6B-parameter model to assign scalar scores by maximizing the log-likelihood of the observed rankings under the Bradley-Terry model. The trained RM is a learned proxy for human preference.

3. **RL fine-tuning (PPO-ptx)** — Use Proximal Policy Optimization (PPO) to fine-tune the SFT model to maximize the RM's predicted reward, with a KL penalty that keeps the policy from drifting too far from the SFT initialization. Mix in a fraction of pretraining log-likelihood to counteract benchmark regression.

**The headline result:** A 1.3B-parameter InstructGPT model trained with this procedure is preferred by human raters over the 175B GPT-3 base model — a model **100× larger** — on nearly all evaluated tasks. Alignment quality can compensate for more than two orders of magnitude of parameter scale.

## Mechanics

### Stage 1: SFT

Standard supervised learning: collect 13,000 (prompt, demonstration) pairs from 40 labeler contractors, fine-tune 175B GPT-3 with cross-entropy loss. The resulting model is called `SFT_175B`. Training is straightforward; the innovation is in the data-collection protocol (labelers were given detailed guidelines about what "helpful" and "harmless" mean).

### Stage 2: Reward Model

**Architecture:** Start from the SFT model (or a 6B variant to save memory during PPO). Remove the final unembedding layer (vocabulary logits) and replace it with a linear projection to a single scalar. Feed it prompt + completion, get back a reward score `r(x, y)`.

**Loss function:** For each triple (x, y_w, y_l) where y_w is the preferred completion:

```
L_RM(θ) = −E[(x,y_w,y_l) ~ D] [ log σ(r_θ(x, y_w) − r_θ(x, y_l)) ]
```

This is exactly the Bradley-Terry pairwise preference model, fit by logistic regression. The sigmoid of the reward difference is the probability the model assigns to y_w being preferred. Maximizing log-likelihood of observations is maximum likelihood estimation of the underlying preference distribution.

**Data:** Labelers rank K completions per prompt (K ranged from 4 to 9). Each ranking yields C(K,2) pairwise comparisons. The paper trains on all pairs from a ranking in a single forward pass, sharing the prompt embedding. Total comparisons: ~33,000 prompts × ~10 pairs = ~330,000 comparison pairs (after deduplication).

### Stage 3: PPO-ptx

The RL objective is:

```
maximize: E[r_φ(x, y)] − β · KL(π_θ(·|x) ∥ π_ref(·|x))
```

where `π_ref` is the frozen SFT model and β controls the strength of the KL penalty.

**Per-token KL penalty:** In practice, the KL is computed token-by-token. After generating a response `y = (y_1, ..., y_T)`, the reward signal per token is:

```
r_t = r_φ(x, y) · 𝟏[t = T] − β · log(π_θ(y_t | x, y_{<t}) / π_ref(y_t | x, y_{<t}))
```

The full response reward (from the RM) is added only at the final token, while KL divergence penalties accumulate at every token. PPO then uses generalized advantage estimation (GAE) over this per-token reward.

**PPO-ptx (alignment tax fix):** An additional pretraining log-likelihood term is mixed in:

```
full objective: PPO objective + γ · E_{x ~ D_pretrain}[log π_θ(x)]
```

This is essentially multi-task learning: the model simultaneously optimizes for reward and for next-token prediction on pretraining text. The mixing coefficient γ is tuned as a hyperparameter. This prevents catastrophic forgetting of general capabilities.

**Four models in memory during PPO:**
- `π_θ`: the active policy (being trained)
- `π_ref`: the frozen SFT policy (for KL computation)
- `V_φ`: the value network (critic for GAE, initialized from the RM)
- `RM_φ`: the frozen reward model (for reward scoring)

### What the labelers judged

Labelers rated outputs on: helpfulness, truthfulness, harmlessness, and lack of problematic content. The evaluation rubric was explicit: labelers were told to prefer outputs that are *actually correct* over confident-sounding but wrong ones. Labelers had to disagree with less than 25% of the lead researcher team on a calibration set to be included.

## Where it breaks

**Labeler subjectivity and bias.** The 40 labelers were English-speaking, predominantly American. The model inherits their values. "Helpful" means helpful from a particular cultural perspective. The paper explicitly notes this and treats it as a limitation, but offers no fix.

**Reward hacking.** Even with the KL penalty, the model finds patterns that score well on the RM but aren't actually better. The paper observes that outputs tend to get longer over RL training — length correlates with labeler preference even when extra length adds no substance. Reward models that don't control for length inadvertently optimize for verbosity.

**RM as a fragile proxy.** The RM is trained on 33k comparisons. At scale, the model will find edge cases the RM misrates. The "Overoptimization" regime (where high RM score no longer correlates with quality) is reproducible and well-studied in later follow-up work.

**Labeler-researcher disagreement.** Labelers were asked to infer researcher intent from guidelines. On controversial topics (politics, religion), the labeler agreement rate drops significantly. The model's behavior on contested questions reflects this ambiguity — not a "correct" position, just the labeler consensus.

**PPO instability.** Proximal Policy Optimization is notoriously difficult to stabilize for language models: it requires careful learning-rate scheduling, clip ratios, and reward normalization. The paper's follow-ups (see Going Deeper) document how much engineering went into making PPO stable at this scale.

**Dataset size.** 13k SFT examples is tiny by pretraining standards. The model's breadth of coverage for the SFT objective is limited to the prompt distribution the labelers were shown. On out-of-distribution prompts, SFT quality degrades.

## Why it works

**Preference model = Elo ratings.** The Bradley-Terry model is the same statistical model used for chess Elo ratings, tennis rankings, and sports prediction. Fitting it to pairwise comparisons is logistic regression — a well-understood, convex optimization problem. The innovation is applying it to LLM output quality rather than player matchup outcomes. This gives reward model training a firm probabilistic foundation: it's MLE of a generative model of human choice behavior.

**KL regularization = a leash in policy space.** The KL penalty KL(π_θ ∥ π_ref) is the information-theoretic distance between the current policy and the SFT initialization. Minimizing it alongside maximizing reward is a soft constraint that prevents mode collapse. Without it, the optimizer finds a deterministic policy that always outputs the single highest-scoring completion — which collapses the model's diversity and amplifies any reward model errors. With it, the model stays in a neighborhood of the SFT distribution, where the RM is a reliable proxy.

This is mathematically the same as elastic net regularization in regression, or weight decay in neural network training — adding a penalty for moving far from the origin (here: the SFT policy) is how you prevent overfitting to a noisy objective (here: an imperfect RM).

**The scale inversion proves the objective dominates.** The 1.3B InstructGPT outperforming 175B GPT-3 is not a compression miracle. It's evidence that the *training objective* — not raw scale — is the bottleneck for alignment. GPT-3 has more capability; InstructGPT has better-targeted capability. This is the same principle as the COST metric (archive, 2026-07-19): a single-threaded algorithm with the right structure beats a distributed cluster with the wrong structure.

**PPO is a bandit problem in disguise.** Each token sequence is a single action; there's no temporal credit assignment across responses. RLHF is RL in a context-bandit setting — the "environment" is stateless (the user prompt), and each "episode" is a single response generation. This makes the RL problem conceptually simpler than Atari or robotics RL, which explains why PPO (designed for sequential MDPs) works at all: you're using a sledgehammer on a relatively simple problem.

**Pretraining mixing prevents catastrophic forgetting.** Adding γ · E[log π_θ(x)] to the PPO objective is multi-task learning with the pretraining task as a regularizer. The model retains the broad capability distribution from pretraining while the PPO gradient pulls it toward preference-optimal outputs. The same principle appears in continual learning: to prevent forgetting Task A when learning Task B, add Task A samples back into training.

## Going deeper

1. **"Direct Preference Optimization: Your Language Model is Secretly a Reward Model" (Rafailov et al., 2023)** — [archive entry 2026-08-07] — Shows that the optimal KL-regularized RLHF policy has a closed form: π*(y|x) ∝ π_ref(y|x) · exp(r(x,y)/β). Substituting this into the Bradley-Terry preference probability makes the partition function cancel, yielding a single binary cross-entropy loss over preference pairs. No reward model, no PPO, no four models in memory.

2. **"Secrets of RLHF in Large Language Models Part I: PPO" (Zheng et al., 2023)** — [arXiv:2307.04964] — A detailed engineering study of what makes PPO stable for LLMs: value model initialization from the RM (not random), per-token KD normalization, clip ratios, and adaptive KL coefficients. Fills the gap between the InstructGPT paper's description and a production implementation.

3. **"Constitutional AI: Harmlessness from AI Feedback" (Bai et al., Anthropic, 2022)** — Replaces human preference labelers with AI-generated preferences derived from a written "constitution" of principles. A model critiques and revises its own outputs, then a preference model is trained on AI comparisons rather than human comparisons. Scales RLHF without human labeler bottlenecks.

---
title: "DeepSeek-R1: Incentivizing Reasoning Capability in LLMs via Reinforcement Learning"
source: https://arxiv.org/abs/2501.12948
author: DeepSeek-AI
company: DeepSeek
date_posted: 2025-01-22
date_digested: 2026-09-25
---

# DeepSeek-R1: Incentivizing Reasoning Capability in LLMs via Reinforcement Learning

## What's new to learn

1. **GRPO (Group Relative Policy Optimization)**: A policy-gradient RL algorithm that eliminates the critic model by scoring each output relative to K siblings sampled in parallel — halving training memory vs PPO while being more stable for long-sequence tasks.

2. **Verification oracle as the RL prerequisite**: Rule-based rewards (correct answer? valid format?) are sufficient for math and code because those outcomes are checkable by a symbolic verifier — which is why *pure* RL (no human-annotated reasoning traces) can bootstrap reasoning capability.

3. **Emergent reasoning as a phase transition**: Training a base LLM with only accuracy rewards spontaneously induces self-reflection, backtracking, and multi-step planning — behaviors not present in the training data — because those strategies raise expected reward on hard problems.

## Prerequisites

- Fine-tuning language models (SFT vs RL vs RLHF) — know roughly what each does
- Chain-of-thought prompting and why longer reasoning paths help on math
- What a reward model is and why RLHF trains one
- Basic policy gradient intuition: you nudge the policy toward outputs that get higher reward

The math of GRPO is explained below; no prior RL coursework required.

## The core idea

The standard way to improve LLM reasoning is to collect human-written reasoning traces (SFT), or to train a reward model on human preferences and then do PPO (RLHF). Both require expensive human labor. DeepSeek-R1 asks: do you need any of that?

The answer is no — but only when you have an *oracle*. For math and code, you already have one: a symbolic verifier or test suite that checks whether the final answer is correct. That checker is O(1) to run and never wrong. That's all RL needs.

**DeepSeek-R1-Zero** is the purest experiment: start from a base model, apply reinforcement learning with only two rule-based rewards (accuracy and format), and let the policy discover reasoning on its own. The result is a model that spontaneously develops multi-step chain-of-thought, self-correction, and even reflection — reaching 71% on AIME 2024 from a 15.6% baseline.

The key enabler is **GRPO**, which makes RL training on LLMs practical: instead of training a separate critic network (as PPO requires), GRPO generates K outputs for each question and uses the *group mean reward* as the baseline. No extra model, no per-token value estimation, no credit-assignment problem.

The paper then shows **DeepSeek-R1**, which applies a four-stage pipeline to clean up R1-Zero's readability issues and reach state-of-the-art performance, and **distillation**, which shows that SFT on R1's reasoning traces is far more efficient than RL-from-scratch for small models.

## Mechanics

### GRPO: replacing the critic with group statistics

PPO estimates the advantage of an action using a learned value function V(s):

```
Advantage_PPO(o, q) = r - V(q)
```

Training V requires a second neural network, usually as large as the policy model. For long reasoning chains (thousands of tokens), V(s) is hard to estimate accurately, making training unstable.

GRPO sidesteps this entirely. For each question q, sample G outputs {o₁, ..., o_G} from the current policy. Compute the reward r_i for each. The advantage of output i is just its normalized position within the group:

```
Â_i = (r_i - mean(r₁...r_G)) / std(r₁...r_G)
```

The GRPO objective is then:

```
J(θ) = E_q [ (1/G) Σᵢ min(ρᵢ · Âᵢ,  clip(ρᵢ, 1−ε, 1+ε) · Âᵢ) ]
       − β · KL(π_θ ‖ π_ref)
```

Where ρᵢ = π_θ(oᵢ|q) / π_old(oᵢ|q) is the standard PPO importance ratio (with clipping to prevent large updates), and the KL term keeps the policy from drifting too far from the frozen reference model (same role as in RLHF).

Memory comparison:
- **PPO**: policy model + critic model + reference model ≈ 3× GPU memory
- **GRPO**: policy model + reference model ≈ 2× GPU memory

And GRPO is better-suited for LLM tasks because the group-mean baseline is directly computed from the task rewards — no regression error, no bootstrapping across token steps.

### Reward design: rule-based, not model-based

R1-Zero uses two reward signals:

1. **Accuracy reward**: For math problems, a symbolic verifier checks the final answer against the ground truth (exact match after normalization). For code, a test harness runs the generated solution. No reward model needed — the verifier is a deterministic function.

2. **Format reward**: +1 when the model uses `<think>...</think>` tags to separate reasoning from the final answer. This steers the model toward producing structured reasoning without requiring examples of structured reasoning.

That's the complete reward signal. No human labelers. No trained reward model. No preference data.

### Emergent behaviors during R1-Zero training

During training, without any supervision on *how* to reason, the model spontaneously develops:

- **Multi-step chain-of-thought**: Rather than answering directly, the model elaborates across many reasoning steps inside `<think>` tags.
- **Self-reflection**: Phrases like "Wait, let me reconsider..." appear in reasoning chains — not because they were in training data, but because reflection tends to increase final accuracy on hard problems.
- **Backtracking**: The model abandons a wrong path and tries a different approach within a single generation.
- **Dynamic length allocation**: Harder problems get longer reasoning chains; easy problems are answered more directly.
- **"Aha moment"**: At a specific training epoch, performance on hard benchmarks jumps as the model begins inserting explicit reflection steps. This is a phase transition in the policy's behavior.

These behaviors emerge because RL pressure selects for strategies that increase the accuracy reward — and on hard reasoning problems, these strategies *do* increase accuracy.

R1-Zero's weakness is readability: it sometimes mixes languages mid-reasoning, produces garbled output, or structures chains inconsistently. This motivates the full R1 pipeline.

### Full DeepSeek-R1 training pipeline

**Stage 1 — Cold-start SFT** (~10K examples):
Curate a small set of long-CoT reasoning examples with clean formatting. Fine-tune the base model on these to establish the `<think>` format and readable language. The goal is not to teach reasoning — it's to get clean enough behavior that RL training converges to legible outputs.

**Stage 2 — Reasoning-focused RL** (GRPO, math + code only):
Apply GRPO with the same rule-based rewards. This stage drives the core capability leap. Only math and code tasks are used so the rule-based verifier can provide reliable rewards.

**Stage 3 — Rejection sampling + general SFT**:
Sample many outputs from the Stage 2 model. Keep only the correct ones (rejection sampling). Mix these high-quality reasoning traces with data for other skills (QA, writing, summarization). Fine-tune to maintain general language capabilities alongside math/code reasoning.

**Stage 4 — Final RL** (mixed rewards):
Apply GRPO again, but now with two kinds of rewards:
- Rule-based accuracy for math/code (same as before)
- A trained reward model for general tasks where rule-based checking is not possible

This final stage balances reasoning ability with helpfulness and format quality.

### Distillation to smaller models

Training small models with GRPO is expensive and often unstable. A much more efficient path:

1. Use DeepSeek-R1 to generate long-CoT outputs on a large reasoning dataset.
2. Keep outputs where the final answer is correct (rejection sampling).
3. Standard SFT fine-tune a small base model (Qwen-7B, Llama-8B, etc.) on these traces.

No RL at all. The small model learns by imitating R1's reasoning style.

Results on AIME 2024:
- **DeepSeek-R1-Distill-Qwen-7B**: 55.5% — beating DeepSeek-R1-Zero (32B, pure RL) at 47.0%
- **DeepSeek-R1-Distill-Qwen-32B**: 72.6%

A 7B model trained by distillation outperforms a 32B model trained by RL from scratch. The teacher's reasoning traces are far more informative per sample than outcome rewards.

This is **knowledge distillation via behavior cloning** — the same principle as training a fast student model to mimic an expensive teacher model (Hinton 2015), applied to reasoning chains.

## Where it breaks

**Domain restriction**: Rule-based rewards require a deterministic outcome verifier. This works for:
- Math (symbolic comparison of the final answer)
- Competitive programming (test suite execution)
- Formal proofs (proof checker)

It *doesn't* work for creative writing, open-ended analysis, subjective QA, or anything where "correct" is a matter of human judgment. Those tasks still require a trained reward model or human labels.

**Reward hacking on format**: The format reward (checking for `<think>` tags) saturates quickly and can be gamed — a model can produce the tags without reasoning inside them. The accuracy reward is the one that drives real capability improvement.

**Distillation ceiling**: Distilled models are bounded by R1's accuracy. If R1 gets a problem wrong, the student model learns a wrong reasoning pattern. RL-trained models may generalize to harder out-of-distribution problems better than distilled ones.

**Long context instability**: Very long reasoning chains (thousands of tokens) create credit-assignment challenges even for GRPO. Later tokens in a chain may receive similar gradient signal whether or not their specific reasoning step was correct — the outcome reward doesn't distinguish them.

**Language mixing in R1-Zero**: Without format supervision, the model sometimes mixes languages (English question, Chinese reasoning, English answer). This is addressed in R1 with a language consistency reward term.

## Why it works

The deeper principle is: **RL requires an oracle, not a teacher**.

In supervised learning (SFT), you need demonstrations of *what correct behavior looks like* — the teacher shows you the full reasoning trace. In RLHF, you need humans to compare outputs and a reward model that generalizes those preferences. Both are expensive.

RL with a rule-based reward requires only that you can *evaluate* an outcome, not *demonstrate* how to reach it. For math and code, evaluation is O(1): run the verifier, get a score. Generation of the correct reasoning path may require searching an exponential space, but the evaluator doesn't care — it checks the output, not the path.

This asymmetry — **verification is polynomially easier than generation** — is what makes self-improvement tractable. It's the same principle that powers:

- **AlphaGo/AlphaZero**: win/loss as reward, no human game records needed for the RL phase
- **AlphaCode**: correctness test suites as reward, no labeled solutions needed
- **Automated theorem proving (AlphaProof)**: proof checker as oracle, no human proofs needed
- **Reinforcement learning from execution feedback (RLEF)**: test suite pass rate as reward for code generation
- **Chess engines (Stockfish self-play)**: game outcome as oracle

In all cases, the oracle is a cheap, deterministic function that an LLM policy can learn from without human annotation. DeepSeek-R1 transfers this to open-ended math reasoning by recognizing that "final answer correct" *is* a cheap verifiable oracle.

GRPO specifically connects to classical statistics:
- The group mean is a **control variate** for each individual sample's reward — reducing variance in the gradient estimate without requiring a separate value function
- This is Monte Carlo estimation with variance reduction: instead of estimating the baseline E[r] from a parametric model (critic), you estimate it empirically from the current batch
- The tradeoff: G samples per question costs G× more model forward passes, but eliminates an entire training objective (value function regression) and avoids its instability

The "aha moment" phase transition is a familiar pattern in complex systems: policy gradient methods can get stuck in local optima until a small exploratory step demonstrates that a new behavior (reflection) has higher expected reward than the current local strategy. Once a few examples show that reflection helps, the policy reinforces it broadly.

## Going deeper

1. **PPO and the Actor-Critic framework**: "Proximal Policy Optimization Algorithms" (Schulman et al., 2017, arxiv.org/abs/1707.06347) is the parent of GRPO — reading it clarifies exactly what GRPO simplified and why.

2. **Process Reward Models**: "Let's Verify Step by Step" (Lightman et al., OpenAI, 2023, arxiv.org/abs/2305.20050) explores training a reward model that scores each *step* of a reasoning chain, rather than just the final answer — the complement to GRPO's outcome rewards, and a richer signal for chains where intermediate steps are verifiable.

3. **Test-time compute scaling**: "Scaling LLM Test-Time Compute Optimally" (Snell et al., 2024, arxiv.org/abs/2408.03314) analyzes *when* generating more tokens at inference (via beam search or self-consistency) helps, connecting the training-time and inference-time views of reasoning scaling.

---
title: "Scaling Test-Time Compute with Open Models"
source: https://huggingface.co/spaces/HuggingFaceH4/blogpost-scaling-test-time-compute
author: Edward Beeching, Lewis Tunstall, Sasha Rush
company: Hugging Face
date_posted: 2024-12-16
date_digested: 2026-10-21
---

# Scaling Test-Time Compute with Open Models

## What's new to learn

1. **Process Reward Model (PRM)**: A verifier trained to assign a correctness score to each intermediate reasoning step — not just the final answer — providing dense feedback that makes tree search over reasoning paths practical.
2. **Test-time compute scaling**: The principle that spending more inference FLOPs on structured search over candidate solutions improves output quality, orthogonally to training-time scaling (more parameters or more training tokens).
3. **Compute-optimal allocation**: The insight that the most efficient search strategy depends on problem difficulty — easy problems need breadth (many independent tries); hard problems need depth (guided exploration of a single promising trajectory).

## Prerequisites

- Basic LLM inference: autoregressive sampling, temperature, nucleus sampling.
- RLHF context: what a reward model is and what it scores.
- Elementary tree search: beam search and BFS/DFS are sufficient; MCTS is helpful but not required.
- Chinchilla scaling laws (covered 2026-09-15): training-time scaling — the idea that performance improves predictably with more training compute — provides the contrast this post needs.
- Non-obvious: the difference between *outcome supervision* (reward only at the end) and *process supervision* (reward at each step), and why that distinction matters for search.

## The core idea

The key observation is a generate-and-verify asymmetry: **checking a partial solution is cheaper than generating a good one**. A Process Reward Model can evaluate a candidate reasoning step in a single forward pass costing roughly as much as generating a token. Generating the step to evaluate costs orders of magnitude more context. This asymmetry means you can afford to generate many candidate reasoning chains and use the PRM to guide search over them, reaching the correct answer with the same FLOP budget at which simple sampling usually fails.

Concretely: instead of sampling one solution and hoping it is correct, you run tree search — expanding promising partial reasoning chains, scoring each with a PRM, and pruning paths that score poorly. This is identical in structure to how AlphaGo works: a policy network proposes moves, a value network evaluates board positions, and MCTS uses the combination to search the game tree. Here, the LLM is the policy network, the PRM is the value network, and beam search or MCTS is the tree search.

The HuggingFace team's contribution is an open implementation of this idea plus a new variant called **DVTS (Diverse Verifier Tree Search)**, which addresses a failure mode of standard beam search on hard problems.

## Mechanics

### Outcome vs. Process Reward Models

An **Outcome Reward Model (ORM)** scores a complete solution as correct or incorrect. It provides one bit of signal per generation, making it usable only for best-of-N selection.

A **Process Reward Model (PRM)** scores each step of a reasoning chain. OpenAI trained the original PRM on PRM800K, a dataset of 800,000 human-labeled step-level annotations on MATH problems. The step score is typically a probability that the solution remains on a correct path through the current step. The aggregate score for a full chain is usually the product (or minimum) of per-step scores.

PRM training: given a partial solution $(q, s_1, \ldots, s_k)$ (question + $k$ steps), label whether continuing this path can reach a correct final answer. Human annotators provided these labels; the resulting signal is rich enough to train a fine-tuned LM to act as a dense evaluator.

### Best-of-N (baseline)

```
for i in 1..N:
    solution_i = LLM.sample(question, temperature=T)
    score_i = RM.score(question, solution_i)
return argmax_i score_i
```

Cost: $N$ × generation passes + $N$ × scoring passes. Scales linearly with $N$. With an ORM, the only signal is final correctness. With a PRM, you aggregate step scores (e.g., minimum step score as the solution score). Works well when $N$ is large but makes no use of partial-solution information.

### PRM-guided Beam Search

```
beam = [("", 0.0)]  # (partial_solution, score)
while not all complete:
    candidates = []
    for (partial, _) in beam:
        for _ in 1..K:                         # K = branching factor
            step = LLM.sample_next_step(partial)
            score = PRM.score(partial + step)
            candidates.append((partial + step, score))
    beam = top_B(candidates)                   # B = beam width
return beam[0]
```

Key property: the PRM score gates which partial solutions survive at each step. A reasoning chain that makes an algebra error in step 2 is pruned before steps 3–10 are generated, saving the compute those steps would have cost. Beam width $B$ controls the diversity-vs-depth tradeoff.

### DVTS (Diverse Verifier Tree Search) — the HuggingFace contribution

Standard beam search collapses toward one high-scoring trajectory early on hard problems, losing diversity. DVTS addresses this:

```
G = diversity (number of subtrees)
budget_per_subtree = total_budget / G
subtrees = []
for g in 1..G:
    root = LLM.sample(question)          # different starting point each time
    subtrees.append(beam_search(root, budget=budget_per_subtree, PRM))
return PRM.select_best(subtrees.flatten())
```

Each subtree explores an independently sampled starting trajectory, then runs beam search within that trajectory. The final answer is selected by the PRM across all completed paths. This is an ensemble of beam searches, each initialized with diversity rather than all starting from the same distribution. On the MATH benchmark, DVTS significantly outperforms equal-budget beam search on problems in the 40–70% difficulty range.

### Compute-Optimal Strategy by Problem Difficulty

The Snell et al. (2024) paper that motivated this work identified a key empirical finding: the best search strategy depends on problem difficulty, measured by the LLM's baseline pass@1 rate.

| Difficulty (pass@1 baseline) | Optimal strategy |
|-----|-----|
| Easy (>60%) | Best-of-N with ORM (breadth) |
| Medium (20–60%) | Weighted best-of-N or beam search with PRM |
| Hard (<20%) | MCTS or DVTS with PRM (depth) |

The intuition is information-theoretic: for an easy problem, many random samples reach the correct answer, so breadth wins. For a hard problem, almost no random sample is correct; you need to guide exploration along promising partial paths and be willing to explore deeply before branching.

The practical implication: a single search strategy applied uniformly wastes compute on easy problems (overkill) and fails on hard ones (insufficient depth). Compute-optimal allocation routes each query to the cheapest strategy that achieves the target quality.

### Experimental Results

On the MATH benchmark using a 1B-parameter LLM (Llama-3 scale):
- Compute-optimal DVTS/beam search with a PRM matches or exceeds a 14B-parameter model at the same inference FLOP budget on medium-difficulty problems.
- Best-of-N with ORM plateaus and is outperformed by PRM-guided search at budgets above N ≈ 16.
- DVTS dominates vanilla beam search on problems where pass@1 < 30%, with gains of 10–20 percentage points.

The headline number from Snell et al.: compute-optimal test-time scaling is **4x more efficient** than naive best-of-N in FLOP-matched comparisons. A 1B model with smart search can match a 14B model.

## Where it breaks

**PRM quality is the ceiling.** The entire search is only as good as the PRM's ability to distinguish correct from incorrect partial solutions. On domains where the PRM was not trained (e.g., code, science), step scores are unreliable and search degrades to random exploration. A 2025 study on PRM generalization from math to code found that standard math PRMs provide near-random signal on code problems.

**Reward hacking under search.** Search is very effective at finding inputs where the PRM scores high but the solution is wrong. This is the same failure mode as RLHF reward hacking, amplified: you are explicitly optimizing the PRM score, not ground truth. Calibrated PRMs and ensemble scoring partially mitigate this.

**Capability floor.** If the LLM has zero pass@1 on a problem class (it never generates a correct step), the PRM has no correct trajectories to guide toward. Test-time compute amplifies existing capability; it cannot synthesize capability that is absent. On AIME-level olympiad problems, even very large budgets provide limited gains for small models.

**Hardware overhead.** Beam search with a PRM runs the PRM on every candidate partial solution at every step. For a beam width of 16 and 10 steps, this is 160 PRM forward passes per problem, on top of 160 LLM steps. Memory-constrained deployments cannot run the PRM and LLM simultaneously without careful batching.

**Latency.** Best-of-N is embarrassingly parallel. Beam search is sequential: each step depends on scores from the previous step. For applications where latency matters more than throughput, parallelism of best-of-N may dominate DVTS quality gains.

## Why it works

The deeper principle is **the generate-and-verify asymmetry**, which appears across CS whenever finding a solution is hard but checking one is easy:

- **NP-completeness**: checking a SAT certificate is O(n); finding one is exponential without P=NP.
- **AlphaGo** (2016): a policy network proposes moves; a value network evaluates board state; MCTS uses both. The LLM/PRM/MCTS architecture here is AlphaGo applied to language.
- **Speculative decoding** (covered 2026-06-08): a small draft model proposes tokens; the large target model verifies. Same asymmetry — verification is one forward pass, generation requires many.
- **Optimistic Concurrency Control** (Calvin, covered 2026-07-29; Percolator, covered 2026-07-18): execute transactions optimistically, verify at commit time. The transaction is "generated" during execution; correctness is "verified" via the write-set conflict check.
- **Program synthesis**: enumerate candidate programs, run test suite to verify. Verification is O(tests), enumeration is combinatorial.
- **Rejection sampling**: sample from a proposal distribution, accept/reject based on a ratio. The PRM is exactly the rejection filter.

The specific form here is: **process supervision makes tree search over reasoning tractable**. Outcome supervision (ORM) provides one bit at the end of a long chain. That's like playing chess and only learning if you won at the end of the game. Process supervision provides a per-step reward, equivalent to a position evaluation after every move — the exact information MCTS needs to do useful exploration.

The **orthogonality to training-time scaling** is the architecturally interesting claim. Train-time scaling (Chinchilla) says: given a FLOP budget for training, spend equal portions on model parameters and training tokens. Test-time scaling says: given a FLOP budget for inference, spend it on search guided by a verifier. These two axes of scaling are additive: larger model + test-time search > larger model alone. This means the inference cost of a language model is not a fixed function of model size — it is a function of the difficulty of the query.

## Going deeper

1. **"Let's Verify Step by Step"** — Lightman et al. (OpenAI, 2023). The original PRM and PRM800K paper; establishes that process supervision outperforms outcome supervision for RLHF on hard math. https://arxiv.org/abs/2305.20050

2. **"Mastering the Game of Go with Deep Neural Networks and Tree Search"** — Silver et al. (DeepMind, 2016). The AlphaGo paper; the canonical template for policy + value + MCTS that test-time LLM search follows structurally. https://www.nature.com/articles/nature16961

3. **"Scaling LLM Test-Time Compute Optimally Can be More Effective than Scaling Model Parameters"** — Snell et al. (Google DeepMind / UC Berkeley, 2024). The full empirical study underlying the Hugging Face post; includes the formal compute-optimal allocation analysis and MATH benchmark results at scale. https://arxiv.org/abs/2408.03314

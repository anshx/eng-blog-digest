---
title: "Mixture of Experts Explained"
source: https://huggingface.co/blog/moe
author: Omar Sanseviero, Lewis Tunstall, Philipp Schmid, Sourab Mangrulkar, Younes Belkada, Pedro Cuenca
company: Hugging Face
date_posted: 2023-12-11
date_digested: 2026-06-26
---

# Mixture of Experts Explained

## What's new to learn

- **Conditional computation**: parameters can exist without being activated — a model can have 10× more weights than a dense counterpart while using only a fraction of them per inference token, breaking the proportional coupling between model capacity and compute cost.
- **Noisy top-k gating**: the routing algorithm that makes sparse expert selection tractable — add Gaussian noise to router logits before taking the top-k, so gradient signal reaches all experts during training and no single expert collapses into a monopoly.
- **Capacity factor**: a per-expert token budget that converts the soft desideratum "balanced routing" into a hard constraint, and directly exposes the bandwidth-vs-quality tradeoff in distributed MoE training.

## Prerequisites

- Transformer architecture at the level of "attention block + feed-forward block": you need to know what the FFN layer is to understand what MoE replaces.
- Softmax and argmax: the gating function is just a learned softmax over N experts, truncated to k outputs.
- Data parallelism and model parallelism: expert parallelism is a third axis, and the post's performance analysis only makes sense if you know what the first two axes are.

## The core idea

In a standard Transformer, every token passes through the same feed-forward network (FFN) on every layer. The FFN is two large linear projections: expand by 4×, apply an activation, then contract back. The weights are dense: every byte of them participates on every forward pass.

Mixture of Experts replaces that single FFN with N independent FFNs (the "experts") plus a learned router. For each token, the router picks the top-k experts (typically k=1 or k=2), runs the token only through those experts, and combines their outputs:

```
y = Σ  G(x)ᵢ · Eᵢ(x)    (sum over only the top-k chosen experts)
```

where G(x)ᵢ is the scalar routing weight for expert i (from a softmax over top-k), and Eᵢ(x) is the output of that expert's FFN.

The model now has N times as many FFN weights, but each token only touches k/N of them. A Mixtral 8x7B model has 46.7B parameters in memory but computes like a 12.9B dense model at inference — 8 experts, top-2 routing, where the attention and other shared layers deflate the effective size.

The phrase "conditional computation" captures the principle: which parameters you compute is a function of the input. Dense models are unconditional. MoE makes computation conditional.

## Mechanics

### The gating network

The router is a single learned linear layer: `W_g ∈ ℝ^(d_model × N_experts)`. Given a token representation x, the raw router logits are `x · W_g`, a vector of N scores. A naïve softmax + argmax would pick the single best-scoring expert — but that makes routing non-differentiable at the hard boundary, and more importantly causes **expert collapse**: the best expert trains faster, becomes better, gets picked more, trains even faster, and the other N-1 experts starve.

**Noisy top-k gating** solves both problems:

1. Add tunable Gaussian noise to each logit before selection:
   ```
   Hᵢ(x) = (x · W_g)ᵢ + StandardNormal() · Softplus((x · W_noise)ᵢ)
   ```
   `W_noise` is a second learned matrix; its outputs control the noise scale per expert per token.

2. Retain only the top-k logits; send the rest to −∞:
   ```
   KeepTopK(v, k)ᵢ = vᵢ  if vᵢ ∈ top-k(v), else −∞
   ```

3. Softmax over the surviving logits to get routing weights G(x).

The noise ensures that near-tied experts occasionally swap positions, so gradient signal propagates to experts that weren't quite selected. This is stochastic exploration of the routing space during training.

### Capacity factor: turning soft balance into a hard constraint

Even with noise, routing is still learned and may be uneven. If 30% of tokens all want expert 0, that expert either stalls waiting to process them (sequential) or overflows (parallel). The fix is a hard token budget per expert:

```
Expert Capacity = (tokens_per_batch / num_experts) × capacity_factor
```

With 512 tokens and 8 experts, the "fair share" is 64 tokens each. A capacity factor of 1.25 gives each expert room for 80 tokens. Tokens that would overflow their chosen expert either get routed to the second-choice expert (GShard) or are simply dropped, passing through the layer via the attention residual connection without any FFN computation.

The Switch Transformer showed that models tolerate dropping up to 10–15% of tokens without quality loss, because the attention sublayer still processes them and nearby tokens can carry the missing signal.

Capacity factor exposes a bandwidth-quality tradeoff: higher factor → fewer dropped tokens → better quality → more cross-device communication (tokens must physically travel to their expert's device).

### Load balancing with auxiliary loss

The capacity factor punishes overflow at runtime but does not prevent the router from learning to overload experts in the first place. The standard fix is an **auxiliary load-balancing loss**:

```
L_aux = α · Σᵢ fᵢ · Pᵢ
```

where fᵢ is the fraction of tokens dispatched to expert i in the current batch, and Pᵢ is the fraction of total router probability mass that went to expert i. Minimizing fᵢ · Pᵢ across all experts simultaneously pushes the model toward a uniform assignment.

The problem: L_aux is added to the primary language-modeling loss and the combined gradient distorts what the router is trying to learn. A strong α corrects balance but teaches the router to maximize uniformity rather than quality. The practical recommendation is α = 1e-2 (Switch Transformers) to α = 1e-3 — small enough to nudge but not dominate.

**Router Z-loss** (ST-MoE, 2022) adds a complementary stability term: penalize large logit magnitudes before the softmax. Large logits make the softmax saturate, causing roundoff errors in the exponential and training divergence. Z-loss costs almost nothing in quality but eliminates a major source of instability.

**DeepSeek-V3's approach** (2024) sidesteps the distortion entirely: instead of adding an auxiliary loss, it maintains per-expert **bias terms** that are added to routing scores during selection but excluded from the gating weight computation. The biases are updated by a heuristic (increase bias for underloaded experts, decrease for overloaded) separate from the main loss. Routing quality and load balance are now controlled by independent mechanisms with no cross-contamination.

### Expert parallelism

In a distributed training setup, MoE adds a third axis of parallelism alongside data and model parallelism. Each device holds one (or a few) experts. The forward pass becomes:

1. **Scatter**: each device sends its tokens to the devices hosting their assigned experts. (All-to-All communication, proportional to batch × hidden_dim.)
2. **Compute**: each device runs its expert's FFN on the tokens it received.
3. **Gather**: results travel back to their origin devices. (Second All-to-All.)

In non-MoE layers, expert parallelism degrades to plain data parallelism — each device just processes its own tokens with shared weights. The All-to-All happens only at MoE layers.

This is why capacity factor drives communication cost: a higher factor means more tokens could arrive at a device, increasing the worst-case message size. Conversely, dropped tokens shrink the scatter/gather payloads.

### Token choice vs expert choice

Standard routing is **token-choice**: each token scores N experts, picks top-k. The problem is tokens vote independently and may all prefer the same experts.

**Expert choice** (2022) flips the assignment: each expert independently selects the top-p tokens from the batch by their affinity score. This guarantees perfect load balance by construction — if an expert takes exactly p tokens, the math ensures uniform utilization. The tradeoff: some tokens may be selected by no expert (or by too many) and need special handling. Expert-choice routing is better studied in research than deployed in production models.

## Where it breaks

**Memory stays dense even when compute is sparse.** A Mixtral 8x7B model with only 2-of-8 experts active per token still requires all 46.7B parameters loaded into VRAM. The memory cost scales with total expert count; the compute cost scales with k. This means MoE's efficiency advantage only materializes at batch sizes large enough to keep experts busy — at batch size 1, you're paying full memory cost for 12.9B-equivalent compute.

**Fine-tuning is harder than pretraining.** At a fixed pretraining perplexity, sparse models underperform dense models on downstream benchmarks, especially reasoning-heavy tasks (SuperGLUE showed larger degradation than knowledge retrieval tasks like TriviaQA). Hypotheses: the routing distribution learned during pretraining may not generalize to fine-tuning data; fewer tokens per expert reduce each expert's effective fine-tuning batch size. Freezing non-expert weights and only training MoE layers partially mitigates this.

**Diminishing returns from more experts.** Quality gains flatten noticeably beyond 64–256 experts per layer. The routing space becomes too fragmented for the router to learn meaningful assignments, and expert specialization degrades as each expert sees too few training examples.

**Cross-device routing is expensive at scale.** The All-to-All communication that implements expert parallelism grows with the number of devices and the capacity factor. At very large cluster sizes, the communication overhead can erode the compute savings. This is why modern MoE deployments (Mixtral, DeepSeek) favor a moderate number of relatively large experts (8–64) rather than thousands of tiny ones.

## Why it works

The deeper principle is **specialized decomposition**: break a hard monolithic function into specializable subcomponents, then learn a routing policy that sends each input to the most capable component.

This is the same structure as:
- **Specialist hiring in organizations**: a company doesn't have every employee respond to every query; routing to the right expert is how human organizations scale without linear growth in coordination cost.
- **Database partitioning / sharding**: route each row to the shard that owns it; each shard specializes on a subset of the key space. The load-balancing problem in MoE is identical to the hot-partition problem in sharded databases.
- **The brain's cortical organization**: neuroscience shows that different regions of the cortex specialize on different modalities and tasks. MoE can be viewed as a learned cortical map.

The reason sparse MoE can match or exceed dense models of equivalent parameter count is that **specialization reduces the learning task per component**. A dense FFN must be a universal approximator for all inputs. An expert FFN only needs to be good at its assigned subset. Each expert can devote all its capacity to a narrower distribution, achieving lower per-subset error than a one-size-fits-all model.

The noise in noisy top-k gating is doing something analogous to epsilon-greedy exploration in reinforcement learning: occasionally route tokens to non-optimal experts to ensure every expert accumulates gradient signal and the routing distribution stays soft enough to improve.

The capacity factor is the key to making distributed MoE practical: without it, the system would need dynamic shape computation at runtime. With it, tensor shapes are static (pad to capacity, drop overflow), which is essential for GPU kernels that require fixed-size matrices. This is a direct trade of physical dropped tokens for hardware efficiency — a tradeoff that works because transformers are robust to moderate information loss.

## Going deeper

1. **Switch Transformers** (Fedus, Zoph, Shazeer 2021 — arxiv:2101.03961): the paper that made MoE practical at scale. Read sections 2–4 for the routing analysis, the capacity factor experiments, and the 4× pretraining speedup results against T5-XXL. The ablations showing model quality survives 10–15% dropped tokens are particularly instructive.

2. **Mixtral of Experts** (Mistral AI 2024 — arxiv:2401.04088): a concrete modern deployment. The architecture section shows how MoE integrates with grouped query attention and sliding window attention in a production open-weight model. Compare the 46.7B total vs. 12.9B active parameter breakdown against the Switch Transformer's analysis to see how the numbers change with k=2.

3. **Megablocks: Efficient Sparse Training with Mixture-of-Experts** (Gale et al. 2022 — arxiv:2211.15841): explains the GPU kernel problem. Standard matrix multiplication requires equal-sized inputs but dynamic routing means unequal expert assignments. Megablocks reframes expert computation as block-sparse matrix multiplication, eliminating dropped tokens entirely while achieving the same GPU efficiency as fixed-size batches. This is the "make the lower layer aware of the unit of variability" insight from systems design applied to ML hardware.

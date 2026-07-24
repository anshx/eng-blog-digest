---
title: "Training Deep Nets with Sublinear Memory Cost"
source: https://arxiv.org/abs/1604.06174
author: Tianqi Chen, Bing Xu, Chiyuan Zhang, Carlos Guestrin
company: University of Washington / CMU
date_posted: 2016-04-20
date_digested: 2026-07-24
---

# Training Deep Nets with Sublinear Memory Cost

## What's new to learn

- **Gradient checkpointing (O(√n) memory)**: By storing only every √n-th intermediate activation during the forward pass and recomputing between checkpoints on-demand during backpropagation, you can train an n-layer network with O(√n) memory instead of O(n), at the cost of roughly one extra forward pass (~33% more compute).
- **The three-way backpropagation strategy space**: There is a continuous family of strategies between "store everything" (O(n) memory, O(n) compute) and "store nothing" (O(1) memory, O(n²) compute). The √n checkpoint strategy sits at the exact geometric-mean optimum of this tradeoff.
- **Pebbling as a unifying lens**: Optimal checkpoint placement on a general computation DAG is an instance of the "pebble game" from circuit complexity theory: any algorithm that computes an n-step chain using M memory cells requires at least ⌈n/M⌉ additional computation steps, and the √n strategy achieves this lower bound exactly.

## Prerequisites

- **Backpropagation**: how gradients flow backward through a computation graph using the chain rule, and why each layer's gradient computation needs that layer's forward-pass activation.
- **Automatic differentiation**: how frameworks like PyTorch/TensorFlow retain tensors in a computation graph for `.backward()`.
- **GPU memory model**: why activations (not model weights) typically dominate peak memory during training — weights are constant per step, but activations scale with batch size and network depth.

## The core idea

During a training step, forward propagation computes activations a₁, a₂, ..., aₙ (one per layer). Backpropagation then flows gradients from the loss backward: to compute ∂L/∂aᵢ, it needs aᵢ itself (to compute the Jacobian of layer i+1). Standard training stores all n activations during the forward pass — O(n) peak memory.

Gradient checkpointing breaks this. You divide the n layers into √n segments of length √n each. During the forward pass, you store only the √n activations at segment boundaries — the "checkpoints" — discarding the rest. When backpropagation later needs an activation inside a segment, you re-run the forward pass for that segment from its checkpoint, producing at most √n additional activations before computing the gradient and freeing them.

Peak memory is thus O(√n) checkpoints + O(√n) recomputed activations within the current segment = O(√n) total. Each segment is computed twice (once during forward, once during backward recomputation), so the total compute is 2n instead of n+n — one extra forward pass, or about 33% more wall-clock time in practice.

For a 100-layer network: standard training stores 100 activations; gradient checkpointing stores 10 checkpoints plus at most 10 active recomputed values — 20× reduction in peak activation memory. The 2016 paper demonstrated this compressing an ImageNet training run from 48 GB to 7 GB in practice, enabling a 10× larger batch size on the same hardware.

## Mechanics

### Checkpoint placement for linear chains

For a network with n layers, select checkpoints at positions {√n, 2√n, 3√n, ...}.

**Forward pass:**
1. Compute a₁, a₂, ..., aₙ as normal.
2. At each position k·√n, save the activation. Discard all non-checkpoint activations immediately.

**Backward pass** (simplified, starting from loss at aₙ):
1. Find the nearest preceding checkpoint c = ⌊i/√n⌋ · √n for the current position i.
2. Re-run the forward pass from aᶜ to aᵢ, storing the segment's intermediate activations in a temporary buffer.
3. Compute ∂L/∂aᵢ using the now-available activation.
4. Free the temporary buffer.
5. Repeat moving backward through segments.

**Memory at any time:**
- √n stored checkpoints (always in memory)
- ≤ √n activations in the current recomputation buffer (temporary)
- Peak: O(√n)

### Why √n is the exact optimum

With k checkpoints uniformly placed, segment length is n/k. Memory = k + n/k. Differentiating:

    d/dk (k + n/k) = 1 - n/k² = 0  →  k = √n

By AM-GM: k + n/k ≥ 2·√(k · n/k) = 2√n, with equality at k = √n.

This is not a heuristic — it's a provable lower bound. No uniform checkpoint scheme can do better.

### PyTorch implementation

```python
from torch.utils.checkpoint import checkpoint

def forward(self, x):
    # Wrap each segment's forward function:
    x = checkpoint(self.segment1, x)  # activations NOT stored
    x = checkpoint(self.segment2, x)  # activations NOT stored
    x = self.final_layer(x)           # final activation IS stored
    return x
```

`torch.utils.checkpoint.checkpoint(fn, *inputs)` runs `fn` in a special context where:
- Intermediate tensors are freed immediately after the forward call.
- A custom backward hook re-runs `fn` when the gradient for its output is needed, regenerating the inputs to `fn`'s children.

The framework tracks no per-layer autograd graph inside the checkpoint; only the inputs to `fn` need to remain.

### Generalization to arbitrary DAGs

For non-sequential graphs (e.g., transformer blocks with skip connections and cross-layer attention), the paper formulates checkpoint selection as a **graph pebbling problem**: given a DAG where each node costs 1 memory unit, find the minimum-memory "pebbling strategy" — an ordering of node evaluations where a pebble is placed on a node when computed and removed when no longer needed as an input.

For general DAGs, optimal pebbling is PSPACE-complete. In practice, frameworks checkpoint at coarse block boundaries (e.g., each transformer layer block), sacrificing optimality for simplicity.

### Selective recomputation (2023 extension)

A 2023 Megatron-LM paper ("Reducing Activation Recomputation in Large Transformer Models," Korthikanti et al.) pushed this further for transformers: not all activations have the same memory/recompute cost profile.

- **Attention softmax weights** (seq_len × seq_len): very large (O(s²)), but cheap to recompute (a few matrix multiplies and a softmax).
- **Linear projection outputs**: small (O(s × d)), but expensive to recompute if d is large.

Selective recomputation stores cheap-to-store/expensive-to-recompute activations and discards large-but-cheap-to-recompute ones. Result: **5× memory reduction** with only **2.7% compute overhead** — vs. 30–40% for naïve full checkpointing. For GPT-3-scale models, this enables 70% reduction in activation memory at effectively no training throughput cost.

## Where it breaks

**Compute overhead is real.** The 33% extra compute is roughly constant regardless of hardware. At billion-dollar training runs (millions of GPU-hours), this overhead is significant. Selective recomputation (above) can bring this below 3%, but requires profiling which activations are cheap to recompute.

**Memory fragmentation.** Re-allocating temporary segment buffers during backpropagation interleaves allocations and frees in CUDA's memory allocator, causing fragmentation. In practice, this means the 7× theoretical memory reduction becomes 4–5× measured reduction due to fragmentation overhead.

**Non-sequential graphs require heuristics.** Transformer decoders with KV caches, multi-path architectures like UNets, and recurrent networks have computation graphs where optimal pebbling is non-trivial. Current frameworks use coarse block-level checkpointing, leaving efficiency on the table.

**Interaction with mixed precision.** Checkpointed activations are typically stored in FP32 (not BF16) to avoid precision loss during recomputation. If the model is trained in BF16, this doubles the checkpoint memory cost, partially negating the savings.

**Does not help with optimizer state memory.** Gradient checkpointing only addresses activation memory during the forward/backward pass. Optimizer states (Adam's first and second moments, ZeRO's parameter shards) are a separate, often larger concern covered by techniques like ZeRO and model parallelism.

## Why it works

The √n optimal checkpoint result is an instance of the **classical time-space tradeoff** in algorithm design: for any computation with a sequential structure, the optimal tradeoff between time T and space S satisfies T × S = Θ(n²), with the Pareto-optimal operating point at T = S = Θ(n). Choosing S = √n forces T = √n extra steps per segment, giving the √n/√n = O(1) amortized overhead per stored activation.

This same pattern appears across computer science wherever you must choose between precomputing and re-deriving:

- **Baby-step giant-step (BSGS)** for discrete logarithms: precompute √p values, then meet-in-the-middle in √p steps. Total cost O(√p) vs. O(p) brute force.
- **Two-level page tables**: split a 32-bit address space into two √(2³²) = 2¹⁶-entry tables, minimizing combined page directory + page table memory.
- **Floyd-Rivest / Pollard's rho cycle detection**: a cycle of expected length O(√|group|) is found by birthday paradox. Storing √n values to detect collisions is the exact same tradeoff.
- **Square root decomposition for range queries**: precompute block sums for √n blocks of √n elements each; point update in O(1), range query in O(√n).
- **Karatsuba multiplication**: recursively split numbers at √n digits to reduce asymptotic multiplications (though the exponent improvement here comes from the recursive structure, not just √n placement).

The unifying formalism is the **pebble game**: given an acyclic computation graph, what is the minimum number of pebbles needed to evaluate the output, subject to the rule that you can remove a pebble from a node only after all its children have been computed? The answer for an n-node path graph with M pebbles is Θ(n/M) extra computation steps — matching gradient checkpointing exactly.

The deeper principle: **any algorithm where you can trade one forward pass of computation for halving the memory footprint should repeatedly apply this tradeoff until compute becomes the binding constraint**. This is why gradient checkpointing is not applied uniformly in practice — selective recomputation applies it only where the memory-per-FLOP ratio is most favorable.

## Going deeper

1. **"Reducing Activation Recomputation in Large Transformer Models"** (Korthikanti et al., MLSys 2023, arXiv:2205.05198): The next step — selective recomputation that profiles which activations are expensive-to-store vs. expensive-to-recompute and checkpoints only the former. 5× memory reduction at 2.7% compute overhead for GPT-3-scale models.

2. **"The Reversible Residual Network"** (Gomez et al., NeurIPS 2017): Instead of recomputing activations from checkpoints, change the architecture to be *invertible*: given the output of a residual block, you can algebraically recover the input. No checkpointing needed — zero extra forward pass cost. (Tradeoff: architectural constraint; not universally applicable.)

3. **"Pebbling and Its Applications"** (Cook, 1974; and Lengauer & Tarjan, 1982): The theoretical foundation for the pebble game applied to circuit complexity. Shows that the space-time tradeoff for general DAGs is PSPACE-complete, explaining why optimal checkpoint placement remains a research problem for complex model architectures.

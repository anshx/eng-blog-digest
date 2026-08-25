---
title: "PipeDream: A More Effective Way to Train Deep Neural Networks Using Pipeline Parallelism"
source: https://www.microsoft.com/en-us/research/blog/pipedream-a-more-effective-way-to-train-deep-neural-networks-using-pipeline-parallelism/
author: Aaron Harlap, Deepak Narayanan, Amar Phanishayee, Vivek Seshadri, Nikhil Devanur, Gregory Ganger, Phil Gibbons
company: Microsoft Research / CMU / Stanford
date_posted: 2019-10-27
date_digested: 2026-08-25
---

# PipeDream: A More Effective Way to Train Deep Neural Networks Using Pipeline Parallelism

## What's new to learn

- **Pipeline parallelism (inter-batch parallelism)**: assigning different model layers to different GPUs and routing successive mini-batches through the pipeline simultaneously — distinct from data parallelism (all GPUs get the full model, different data) and tensor parallelism (a single layer is split across GPUs).
- **1F1B scheduling (One-Forward-One-Backward)**: the schedule that keeps every stage active in steady state by alternating forward passes for new micro-batches with backward passes for older ones, eliminating the periodic pipeline flush that dominates GPipe's bubble fraction.
- **Weight stashing**: maintaining one copy of the model weights per in-flight micro-batch so that the backward pass for micro-batch *k* uses the exact same weights that the forward pass saw, preserving gradient semantics despite concurrent weight updates at other stages.

## Prerequisites

- **Data parallelism**: each worker holds a full model copy and processes a distinct mini-batch; gradients are all-reduced across workers and weights are updated simultaneously everywhere.
- **AllReduce mechanics**: after a backward pass, all N workers sum their local gradients and distribute the result, so every worker updates identically (covered in the Ring AllReduce digest).
- **Gradient descent basics**: a single training step is forward(batch) → compute loss → backward(loss) → update weights. The backward pass produces gradients ∂loss/∂weights for every layer.
- **Why models don't always fit on one GPU**: a 70B-parameter model at bfloat16 needs ~140 GB of weights alone, before activations, optimizer states, and gradients — well beyond any single GPU's VRAM.

## The core idea

Data parallelism is the default approach to multi-GPU training: replicate the model N times, split the batch N ways, all-reduce gradients at the end of each step. It works well until:
1. The model itself doesn't fit on one GPU (so you can't replicate it).
2. The all-reduce communication dominates (parameter counts are too large relative to compute).

The alternative is to **pipeline the model across GPUs**: GPU 0 holds layers 1–L₁, GPU 1 holds layers L₁+1–L₂, and so on. A single micro-batch flows through each stage in sequence, exactly like a CPU instruction flowing through fetch → decode → execute → writeback stages.

The naive pipeline parallelism approach (GPipe) batches a set of micro-batches forward through all stages, then backward, then updates weights — a pipeline flush every step. This leaves most stages idle most of the time, since the forward pass must finish before any backward pass begins.

PipeDream's key insight: **there is no reason to synchronize forward and backward passes across micro-batches**. Instead of:

```
Stage 1: F1 F2 F3 F4 --- B4 B3 B2 B1 --- update
         [compute]   [idle]  [compute]   [idle]
```

PipeDream does:

```
Stage 1: F1 F2 F3 F4 B4 B3 B2 B1 (update each batch, no flush)
Stage 2:    F1 F2 F3 F4 B4 B3 B2 B1
Stage 3:       F1 F2 F3 F4 B4 B3 B2 B1
Stage 4:          F1 F2 F3 F4 B4 B3 B2 B1
```

Once the pipeline fills, every stage is always busy: it's either forwarding micro-batch *k* or backwarding micro-batch *k − (p−1)*. This is the 1F1B (one forward, one backward) schedule.

## Mechanics

### Stage assignment

PipeDream profiles each layer on a single GPU to collect:
- Compute time (forward + backward, in ms)
- Output activation size (bytes to send to the next stage)

A dynamic programming algorithm partitions the layers into *p* stages such that:
- All stages have equal compute time (balance the pipeline)
- Inter-stage communication is minimized (activation tensors between adjacent stages)
- Each stage's activations and weights fit in one GPU's memory

Stages with insufficient compute can be replicated (run data-parallel within that stage), using additional GPUs to match the rate of other stages.

### 1F1B schedule

The steady-state invariant: at any moment, each stage has at most one forward pass and one backward pass in-flight. In the startup ("fill") phase, stages are added one-by-one as micro-batches enter. Once all p stages have at least one micro-batch, every stage alternates:

1. Run backward pass on micro-batch *k* (the oldest active micro-batch for this stage).
2. Run forward pass on micro-batch *k + p* (the newest micro-batch arriving).
3. Update weights using the gradient from step 1.
4. Repeat.

**Pipeline bubble**: the fill phase (first p−1 micro-batches) and drain phase (last p−1 micro-batches) see idle stages. The idle fraction is approximately:

```
bubble fraction ≈ (p − 1) / (m + p − 1)
```

where m is the number of micro-batches per gradient update. For large m (many micro-batches), the bubble fraction → 0. PipeDream targets m = 4p or more.

### Weight stashing

The subtle problem: PipeDream updates weights *continuously*, not at a flush boundary. So Stage 1 may update its weights between forwarding micro-batches 1 and 2. When Stage 4 sends gradients back to Stage 1 for micro-batch 1, Stage 1's weights are now at version *w₁* not *w₀* — an inconsistency that degrades convergence.

PipeDream's fix: each stage **stashes** (saves) the weight version it used during the forward pass of each micro-batch. When the backward gradients arrive, the backward pass runs against the *stashed* weights, not the current weights. After the backward pass, the stashed copy is discarded.

With p stages and 1F1B scheduling, a stage has at most p different micro-batches in-flight at once, so it must keep at most p weight copies in memory. This is the dominant extra cost of PipeDream vs GPipe: p × model_shard_size additional memory per GPU.

```
Weight stashing at Stage 1, p=4 stages:
  After forward(micro-batch 1): stash w¹₀
  After forward(micro-batch 2): stash w¹₁  (w¹₁ = w¹₀ + Δ₁)
  After forward(micro-batch 3): stash w¹₂
  After forward(micro-batch 4): stash w¹₃
  On backward(micro-batch 1):   run backward using stashed w¹₀; discard it
  On backward(micro-batch 2):   run backward using stashed w¹₁; discard it
  ...
```

### Automatic partitioning

The DP algorithm runs in O(L² × p) time where L is the number of layers. It enumerates all possible ways to cut L layers into p contiguous groups and selects the cut that minimizes the maximum stage runtime (the pipeline's throughput bottleneck). With profiling on a single A100, this runs in minutes.

When bandwidth between GPUs is low (inter-node), PipeDream may choose fewer, larger stages to reduce activation transfer volume. When bandwidth is high (NVLink within a node), it can afford more stages.

### Performance

| Workload | GPUs | PipeDream vs data parallelism |
|---|---|---|
| VGG-16 image classification | 16 | 5.3× |
| Machine translation (GNMT) | 16 | 3.1× |
| Language model (PTB) | 16 | 4.3× |
| Video captioning | 4 | 3× |

The speedups come primarily from eliminating the AllReduce communication bottleneck: data parallelism must all-reduce all gradients each step (transferring the full model size), while pipeline parallelism only transfers the activation tensor at each stage boundary (much smaller).

## Where it breaks

**Memory overhead.** p weight copies per stage is expensive. A 7B-parameter model sharded across 8 stages, with 8 weight copies per stage, needs 8× the memory of a single model shard. This limits p in memory-constrained settings.

**Stale gradients.** Stashed weights are not the current weights — they're 1 to p−1 steps old. PipeDream shows empirically that this doesn't significantly hurt convergence (the variance introduced is small), but this is not guaranteed for all optimizers. Adaptive optimizers (Adam) interact with staleness differently than SGD.

**Rebalancing is expensive.** If layer compute times change during training (e.g., due to dynamic sequence lengths in transformers), the optimal partition changes. Re-profiling and repartitioning mid-training is not supported.

**Load balancing with residual blocks.** Transformers have uniform layer shapes (unlike VGG), so stage assignment is straightforward. But models with heterogeneous layer types (embedding layers, attention layers, MLP layers) require more care to prevent imbalance.

**Shared weights are broken.** If two layers share the same weight matrix (e.g., input embedding tied to output projection in language models), they must be on the same stage, constraining the partition.

**Not composable with tensor parallelism naively.** Combining pipeline parallelism and tensor parallelism (3D parallelism) requires careful scheduling of all-reduces within stages while forwarding/backwarding. Megatron-LM's interleaved 1F1B schedule solves this at the cost of higher memory.

## Why it works

### The CPU pipeline analogy

Pipeline parallelism in DNN training and instruction pipelining in CPUs are structurally identical problems with structurally identical solutions:

| CPU instruction pipelining | DNN pipeline parallelism |
|---|---|
| Instruction → stages (fetch/decode/execute/writeback) | Micro-batch → stages (layer groups) |
| Multiple instructions in-flight simultaneously | Multiple micro-batches in-flight simultaneously |
| Data hazard: instruction needs result of prior instruction | Weight hazard: backward needs same weights as forward |
| Solution: register renaming (keep old register values) | Solution: weight stashing (keep old weight copies) |
| Pipeline bubble during branch resolution | Pipeline bubble during fill/drain |
| Out-of-order execution (reorder backward/forward) | 1F1B schedule (reorder within a stage's queue) |

The **deeper principle**: whenever you pipeline a sequential computation, you introduce *structural dependencies* between in-flight instances that would clash if they shared a single version of mutable state. The universal solution is to **rename the state**: maintain one version per in-flight instance, tagged by the instance it belongs to. In CPUs this is register renaming; in PipeDream it is weight stashing; in MVCC databases it is row versioning.

### Why pipeline outperforms data parallelism at scale

In data parallelism, every gradient step requires an AllReduce of the full gradient tensor (size = model parameters × bytes/param). At 70B parameters × 2 bytes = 140 GB per step, this saturates inter-node InfiniBand even at 400 Gb/s. The AllReduce is on the critical path.

In pipeline parallelism, the inter-GPU communication is the activation tensor at each stage boundary — just the hidden dimension of the forward pass output, not the full gradient. For a transformer with hidden size 4096 and batch size 1, the activation tensor at each boundary is 4096 × seq_len × 2 bytes ≈ 16 MB per sequence. This is orders of magnitude smaller than 140 GB of gradients.

Pipeline parallelism **trades communication volume for computation overlap**: instead of communicating all gradients at the end, it communicates small activations continuously between stages. This is the same trade-off as systolic arrays vs. shared-memory BLAS: local, structured communication rather than global broadcast.

## Going deeper

1. **"Efficient Large-Scale Language Model Training on GPU Clusters Using Megatron-LM"** (Narayanan et al., SC 2021 — https://arxiv.org/abs/2104.04473) — combines tensor parallelism, pipeline parallelism, and data parallelism into a unified 3D strategy; introduces the interleaved 1F1B schedule (virtual pipeline stages) that cuts the bubble fraction by a factor of p while keeping memory overhead manageable.

2. **"PipeDream-2BW: Memory-Efficient Pipeline-Parallel DNN Training"** (Narayanan et al., MLSys 2021 — https://arxiv.org/abs/2102.03079) — replaces weight stashing with a double-buffered variant (2BW) that keeps at most 2 weight versions per stage (instead of p), drastically reducing memory overhead at the cost of allowing 1 step of weight staleness.

3. **GPipe paper: "GPipe: Efficient Training of Giant Neural Networks Using Pipeline Parallelism"** (Huang et al., NeurIPS 2019 — https://arxiv.org/abs/1811.06965) — the Google Brain alternative to PipeDream that uses synchronous pipeline-flush semantics (no weight stashing) and gradient checkpointing (recompute activations in the backward pass instead of storing them), trading throughput for memory and simplicity.

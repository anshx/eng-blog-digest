---
title: "ZeRO: Memory Optimizations Toward Training Trillion Parameter Models"
source: https://www.microsoft.com/en-us/research/blog/zero-deepspeed-new-system-optimizations-enable-training-models-with-over-100-billion-parameters/
author: Samyam Rajbhandari, Jeff Rasley, Olatunji Ruwase, Yuxiong He
company: Microsoft Research / DeepSpeed
date_posted: 2020-02-13
date_digested: 2026-06-18
---

# ZeRO: Memory Optimizations Toward Training Trillion Parameter Models

## What's new to learn

1. **The optimizer-state memory crisis**: For mixed-precision Adam, optimizer states (fp32 momentum, variance, and master weights) consume 12 out of 16 bytes per parameter — 75% of training memory — yet are fully replicated across every data-parallel GPU. Almost nobody knows this breakdown intuitively.

2. **ZeRO's three-stage partitioning ladder**: You can eliminate the N-fold replication in three escalating steps (optimizer states → gradients → parameters), and the first two stages cost zero extra communication bandwidth compared to standard data parallelism.

3. **Reduce-scatter + all-gather = all-reduce**: ZeRO replaces a single all-reduce with two smaller collectives. Together they move the same bytes as all-reduce, but each GPU only ever holds the fraction of state it "owns" — enabling the memory savings without a communication penalty.

## Prerequisites

- What **data parallelism** is: each GPU holds a full model copy, processes a disjoint data shard, then aggregates gradients with an all-reduce.
- What **mixed-precision training** means: forward/backward in fp16, weight updates in fp32, to balance speed and numerical stability.
- How **Adam** works: it tracks per-parameter first moment (momentum) and second moment (variance) alongside the weights. These are the optimizer states.
- What **all-reduce** does in a distributed context: aggregate a tensor across all workers so every worker ends up with the same sum.

## The core idea

Standard data parallelism replicates the entire training state — parameters, gradients, and optimizer states — on every GPU. With N GPUs you have N identical copies. That's N-1 copies that exist purely for availability, doing no unique computational work.

ZeRO's answer: **each GPU should own 1/N of the training state and be the single authority responsible for updating that shard.** When a GPU needs state it doesn't own (during a forward pass, or to read a finished gradient), it fetches it via a collective. Collectively, the N GPUs still cover 100% of the model; individually, each one holds only its fair share.

This is not a new idea in computing — it's exactly what RAID-0 (data striping) does with disk blocks, or what any sharded database does with rows. ZeRO applies it to ML training tensors.

## Mechanics

### Memory baseline: 16 bytes per parameter

In standard mixed-precision Adam training, each scalar parameter incurs:

| Tensor | Dtype | Bytes |
|---|---|---|
| fp16 parameters | float16 | 2 |
| fp16 gradients | float16 | 2 |
| fp32 master weights | float32 | 4 |
| fp32 Adam momentum m | float32 | 4 |
| fp32 Adam variance v | float32 | 4 |
| **Total** | | **16** |

For a 7.5-billion-parameter model this is 120 GB per GPU. A 175B-parameter model (GPT-3-class) is 2.8 TB per GPU. Every data-parallel replica is a full copy of all 16 bytes.

### Stage 1 (Pos): Partition optimizer states

Each GPU `i` is assigned responsibility for parameter indices in the slice `[i·Ψ/N, (i+1)·Ψ/N)` where Ψ is the total parameter count.

**Forward pass**: unchanged — every GPU runs the full model on its local data shard.

**Backward pass**: unchanged — every GPU computes the full gradient.

**Gradient aggregation**: instead of a full all-reduce, use **reduce-scatter**. GPU `i` accumulates the fully-reduced gradient only for its slice; the rest are discarded immediately after they're forwarded on to the responsible owner. Each GPU now holds a complete, reduced gradient for its shard.

**Parameter update**: GPU `i` applies Adam only to its shard, using its local fp32 optimizer states (momentum, variance, master weights). No other GPU participates.

**Re-synchronization**: an **all-gather** broadcasts the updated fp16 parameters from each owner to all GPUs. Everyone is now in sync for the next forward pass.

Communication volume: reduce-scatter + all-gather = 2·(N-1)/N·Ψ bytes ≈ 2Ψ, the same as a standard all-reduce. Memory cost per GPU: 2 (fp16 params) + 2 (fp16 gradients) + 12/N (optimizer states). For N = 64: ~4.2 bytes/param, a **~3.8× reduction** with no communication overhead increase.

### Stage 2 (Pos+g): Also partition gradients

After each layer's backward pass, the gradient for parameters in slice `[j·Ψ/N, (j+1)·Ψ/N)` is immediately reduce-scattered to GPU `j` and **discarded on all other GPUs**. By the time the full backward pass finishes, GPU `i` holds the fully-reduced gradient only for its shard.

Memory per GPU: 2 (fp16 params) + 2/N (gradients) + 12/N (optimizer states). For N = 64: ~2.2 bytes/param, an **~8× reduction** — still with no communication overhead increase vs. baseline.

### Stage 3 (Pos+g+p): Also partition parameters

The parameters themselves are partitioned. Each GPU holds only its 1/N slice in permanent storage.

**Forward pass**: before each transformer layer (or weight matrix), an all-gather collects the full weight from its N/N owners. After the layer computes, non-owners discard the gathered slice immediately (they don't have VRAM to spare).

Memory per GPU: 16/N bytes/param — **linear in N**, the theoretical optimum.

**Trade-off**: the extra all-gathers during the forward pass add ~50% communication volume compared to the baseline. This is acceptable when inter-GPU bandwidth is high (NVLink on a single node) but can hurt when training across slow inter-node links.

### Communication summary

| Stage | Memory per GPU | Extra communication |
|---|---|---|
| Baseline (no ZeRO) | 16 bytes/param | — |
| Stage 1 (optimizer) | ≈ 4 bytes/param (large N) | 0% |
| Stage 2 (+ gradients) | ≈ 2 bytes/param (large N) | 0% |
| Stage 3 (+ parameters) | 16/N bytes/param | +50% |

### Performance numbers from the paper

- 7.5B-parameter model on 64 V100 GPUs (Stage 2): 10× throughput improvement over Megatron-LM model parallelism at equal scale.
- 100B-parameter model on 400 V100 GPUs (Stage 3): 15 petaflops, 8× larger model with 10× higher throughput vs. prior state-of-the-art.
- Stage 3 overhead over Stage 2 is ≈6% at 1 GB/s inter-node bandwidth, justifying the extra communication for extreme-scale models.

## Where it breaks

**Slow inter-node links**: Stage 3's 50% communication overhead compounds across training steps. If inter-GPU bandwidth is the bottleneck (e.g., Ethernet-connected clusters vs. NVLink), Stage 3 regresses to below baseline throughput.

**Very small models**: If a model already fits in one GPU's memory, ZeRO's added collective operations are pure overhead. At N = 1, every stage degenerates to standard training with extra latency.

**Compute-bound layers**: ZeRO only addresses memory, not compute. A model that fits in memory but is compute-constrained (dense GEMM on large matrices) sees no benefit.

**Still needs model/tensor parallelism for the very largest models**: ZeRO eliminates redundancy across data-parallel replicas. A single layer that is too large to fit on any single GPU (even with Stage 3) still requires tensor parallelism or pipeline parallelism to split the computation. In practice, production LLM training uses 3D parallelism: data parallelism (ZeRO handles the memory), tensor parallelism (splits a single layer's weight matrix across GPUs), and pipeline parallelism (splits layers across stages).

**Stage 3 complicates checkpointing**: with parameters sharded, saving and loading checkpoints requires extra gather steps. This adds operational complexity.

## Why it works

The deeper principle is **redundancy elimination via partitioning**, and it has a direct analogy in storage systems:

- Standard data parallelism is **RAID-1** (full mirror): N copies of the data, any node can serve a full read, but storage efficiency is 1/N.
- ZeRO Stage 3 is **RAID-0** (striping): each node holds 1/N of the data. A full read requires reading from all N nodes simultaneously (→ the all-gather). Storage efficiency is 1.0×, with a coordination cost proportional to N.

The reason Stages 1 and 2 incur zero extra communication is more subtle. Recall that a standard all-reduce is logically decomposable: reduce-scatter (aggregate, giving each worker 1/N of the output) + all-gather (broadcast each 1/N to all workers) = all-reduce. Standard data parallelism uses all-reduce and then throws away the intermediate per-shard outputs — ZeRO simply keeps them. The computational work was always there; ZeRO just stops discarding the intermediate state.

The mental model to carry: **reduce-scatter is a "free" shard assignment**. Every distributed all-reduce already internally computes the per-shard reduced gradient; standard implementations discard it and reconstruct a full vector. ZeRO just intercepts the intermediate result and uses it to assign work.

This is also an instance of the broader principle behind MVCC, partial aggregation in columnar databases, and MapReduce: **push state closer to the worker that owns it, and use a collective only when the full state is strictly required.** In ZeRO, the full state (all parameters) is only required transiently during the forward pass of each layer — and it can be re-derived from the shards on demand.

## Going deeper

1. **ZeRO-Infinity** (Microsoft Research, 2021): extends ZeRO Stage 3 to spill shards to NVMe SSDs and CPU RAM, enabling models with trillions of parameters on commodity hardware. [https://arxiv.org/abs/2104.07857](https://arxiv.org/abs/2104.07857)

2. **Megatron-LM tensor parallelism** (NVIDIA, 2019): the orthogonal technique for splitting individual weight matrices across GPUs. Understanding ZeRO (which targets data-parallel redundancy) alongside Megatron (which targets intra-layer compute splitting) explains why modern training stacks use both simultaneously. [https://arxiv.org/abs/1909.08053](https://arxiv.org/abs/1909.08053)

3. **"Bringing HPC Techniques to Deep Learning"** (Baidu SVAIL, 2017): the original post explaining ring-allreduce as the bandwidth-optimal all-reduce implementation. Explains why `reduce-scatter + all-gather` achieves 100% network utilization on a ring topology, which is the communication primitive ZeRO relies on. [https://andrew.gibiansky.com/blog/machine-learning/baidu-allreduce/](https://andrew.gibiansky.com/blog/machine-learning/baidu-allreduce/)

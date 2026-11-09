---
title: "Decoupled DiLoCo: Resilient Distributed Pre-training"
source: https://deepmind.google/blog/decoupled-diloco/
author: Arthur Douillard et al.
company: Google DeepMind
date_posted: 2026-04-23
date_digested: 2026-11-09
---

# Decoupled DiLoCo: Resilient Distributed Pre-training

## What's new to learn

1. **Local SGD / pseudo-gradients**: Running H inner optimization steps per worker before synchronizing reduces inter-datacenter communication by roughly H×. The "pseudo-gradient" (Δθ = initial_weights − final_weights after H steps) is a sufficient summary of a worker's local contribution — you never ship per-step gradients across the WAN.

2. **Two-level optimization**: An inner optimizer (AdamW, running locally on each learner) handles per-token updates; an outer optimizer (Nesterov SGD, running once per round on the aggregated pseudo-gradients) captures the slow inter-round drift and applies momentum across synchronization boundaries. The two levels operate at fundamentally different timescales and need separate learning rates.

3. **Decoupled asynchronous outer sync**: Starting the next round of inner steps immediately — without waiting for the outer all-reduce to complete — converts the synchronization step from a synchronous barrier into background work. Each learner unit is fault-isolated; a failed unit doesn't stall the rest, and a recovered unit rejoins by simply loading the latest global model.

## Prerequisites

- **Ring AllReduce** (or any all-reduce primitive): how data-parallel training aggregates per-step gradients, and why it requires a high-bandwidth, low-latency fabric.
- **AdamW / Nesterov SGD**: what first- and second-moment estimates do, and what Nesterov momentum adds over plain SGD.
- **Data parallelism vs. model parallelism**: the difference between replicating the full model across workers (data-parallel) and sharding it (tensor/pipeline-parallel).

## The core idea

Standard data-parallel training is elegant but bandwidth-hungry: after every forward+backward pass, all workers all-reduce their gradients so every replica stays identical. For a 70B model trained on a cluster of 10,000 GPUs inside one datacenter, this works because NVLink / InfiniBand supplies hundreds of Gbits/s per GPU. Across datacenters, that number collapses to single-digit Gbits/s — sometimes lower. Standard all-reduce becomes the bottleneck, and a single node failure anywhere stalls the whole job.

DiLoCo's insight is to decouple *inner* optimization (per-worker, per-step AdamW on local data) from *outer* synchronization (global, once per H steps, using aggregated pseudo-gradients). Instead of shipping 100 GB of gradients every step, each worker ships one 100 GB weight-delta every H steps — the same bandwidth amortized over H× more compute. For H = 100, the communication falls from ~104 Gbits/s to ~1.7 Gbits/s.

Decoupled DiLoCo removes the remaining synchronous barrier: rather than halting all workers while the outer all-reduce happens, each learner immediately starts its next H inner steps. The outer sync for round k finishes sometime during round k+1, using a one-round-stale update. Because pseudo-gradients change slowly between consecutive rounds, this staleness barely affects convergence — the system gains ~88% goodput under high hardware failure rates vs. ~27% for standard data parallelism.

## Mechanics

**A single DiLoCo round:**

1. Each of the M learner units receives the current global model θ_global (or keeps the latest one it has in Decoupled mode).
2. Each unit trains locally for H steps with AdamW on its shard of data:  
   θ_local,i ← AdamW(θ_global, local_data_i, lr_inner, H_steps)
3. Each unit computes its pseudo-gradient:  
   Δθ_i = θ_global − θ_local,i  
   (note the sign: outer optimizer will *subtract* Δθ, so a positive Δθ means the model moved in that direction)
4. All-reduce the pseudo-gradients across all M units (average):  
   Δθ_avg = (1/M) Σ Δθ_i  
   Communication cost: 2 × |model_params| × sizeof(float) / round, same as one normal all-reduce but covering H steps.
5. Outer optimizer updates the global model:  
   θ_global ← NesterovSGD(θ_global, −Δθ_avg, lr_outer, momentum)  
   (Nesterov adds a look-ahead correction: apply momentum first, then gradient.)

**Decoupled extension:**

Steps 4–5 happen asynchronously alongside the *next* round's step 2. Each learner starts round k+1 immediately with the θ_global it already has; the step-5 update from round k arrives partway through and is applied to produce θ_global for round k+2. The one-round lag in the outer update is the price of full asynchrony.

**Fault tolerance mechanics:**

- A failed learner is excluded from the round-k all-reduce (the other M−1 learners proceed normally).
- When it recovers, it fetches the latest θ_global and resumes inner training from round k+j, where j is however many rounds it missed.
- Because each learner unit trains on its own data shard independently (no pipeline coupling), any unit can join or leave without protocol coordination — the syncer just adjusts M on the fly.

**Bandwidth numbers (from the paper):**
- Standard data-parallel all-reduce: ~104 Gbits/s inter-datacenter bandwidth needed.
- Decoupled DiLoCo in FP32: ~1.7 Gbits/s.
- Decoupled DiLoCo with int4 pseudo-gradient compression: ~0.43 Gbits/s.
- Reduction factor: ~60–240× depending on precision.

**Goodput at scale (1.2M chip simulation, one-year MTBF per chip):**
- Standard data-parallel: 27%
- Elastic data-parallel: 58%
- Decoupled DiLoCo: 88%

## Where it breaks

**Local divergence degrades aggregation quality.** More inner steps H means each worker's model diverges further from the global model before averaging, which makes the pseudo-gradients less aligned. Papers studying DiLoCo show diminishing returns beyond H ≈ 500 and loss degradation at H ≫ 1000. There is an H that maximizes bandwidth savings without hurting loss, and it is not "infinity."

**Two learning rates need independent tuning.** The inner lr_inner and outer lr_outer operate at different granularities. Setting lr_outer too high amplifies the inter-round variance; too low and the outer optimizer contributes nothing beyond plain FedAvg. Tuning both for a new model/data regime adds complexity.

**One-round staleness can matter on sharp loss landscapes.** If the loss landscape is highly non-stationary (e.g., the later stages of fine-tuning or in-context adaptation), the stale outer update may point in the wrong direction. The original DiLoCo (synchronous outer sync) handles this better at the cost of a stall.

**Memory overhead per learner unit.** Each unit must store: its local model (for AdamW states), plus a copy of θ_global (to compute the pseudo-gradient). For a 70B model this is a non-trivial overhead versus standard data-parallel which only stores one copy.

**Not a drop-in for intra-datacenter training.** When the fabric already delivers hundreds of Gbits/s, standard all-reduce is faster and simpler. DiLoCo / Decoupled DiLoCo is an engineering choice for the WAN case, not a universal improvement.

## Why it works

**The pseudo-gradient as a sufficient statistic.** After H steps of AdamW, the net weight change Δθ_i encodes everything the outer optimizer needs: the direction of descent aggregated over H micro-batches, with AdamW's adaptive per-parameter scaling already applied. You are not sending "which gradient the worker would have taken at this step" — you're sending "what direction H steps of optimization actually moved the model," which is a richer, lower-noise signal. The outer Nesterov optimizer then applies inter-round momentum on top of this already-aggregated signal.

**The amortization principle.** This is the same "batch your synchronization to reduce overhead" pattern that appears throughout distributed systems:

| System | Local work | Synchronization event |
|--------|------------|-----------------------|
| TCP Nagle algorithm | Accumulate small writes in socket buffer | Flush when buffer full or ack received |
| Kafka producer | Batch records per-topic | Flush every linger.ms ms or batch.size bytes |
| Calvin sequencer | Execute transactions locally | Agree on global order once per epoch |
| Ring AllReduce | Compute per-mini-batch gradients | All-reduce every step (baseline), vs. DiLoCo every H steps |
| DiLoCo outer sync | Run H local AdamW steps | Average pseudo-gradients once per round |

The deeper principle: **any algorithm where local computation is cheaper than global coordination should batch local steps to amortize the coordination cost**, subject to the constraint that accumulated local drift doesn't exceed the tolerance of the outer protocol's reconciliation step.

**Asynchrony safety from slow-moving pseudo-gradients.** The one-round staleness in Decoupled DiLoCo is safe for the same reason that Nesterov momentum is safe on smooth loss surfaces: pseudo-gradients between consecutive rounds correlate strongly (the model learns slowly relative to H steps), so the one-round-old outer update points in almost the same direction as the current-round update would. This mirrors why Kafka's async producer (`acks=1`) is acceptable for high-throughput analytics pipelines: the probability of losing one batch is low, and the batch is not the final word on correctness.

**Connection to federated learning's FedOpt.** DiLoCo is essentially FedOpt (Reddi et al. 2020) applied to LLM pre-training with one key observation: in federated learning the "outer" optimizer was introduced to handle heterogeneous client data and variable participation; in LLM pre-training the motivation is *network bandwidth*, but the math is the same. The outer Nesterov optimizer compensates for the bias introduced by local divergence, playing the role that server-side momentum plays in FedAdam/FedYogi.

## Going deeper

1. **Original DiLoCo paper**: Douillard et al. (2023), "DiLoCo: Distributed Low-Communication Training of Language Models," arXiv:2311.08105. The base algorithm and convergence analysis. Skip to Section 3 for the two-level optimization formulation.

2. **FedOpt** (Reddi et al. 2020), "Adaptive Federated Optimization," arXiv:2003.00295. The federated learning paper that introduced the outer-optimizer concept DiLoCo inherits. Reading this alongside DiLoCo makes clear that DiLoCo is "federated LLM training, with LR-schedule insights from pre-training."

3. **Local SGD** (Lin et al. 2018), "Don't Use Large Minibatches, Use Local SGD," arXiv:1808.07217. The convergence theory behind why averaging models after K local steps works, and what determines the optimal K.

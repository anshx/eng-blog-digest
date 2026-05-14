---
title: "Disaggregated Inference: 18 Months Later"
source: https://haoailab.com/blogs/distserve-retro/
author: Hao Zhang et al. (Hao AI Lab)
company: UC San Diego / Hao AI Lab
date_posted: 2025-11-03
date_digested: 2026-05-14
tags: [llm-serving, gpu, distributed-systems, inference]
---

# Disaggregated Inference: 18 Months Later

## What's new to learn

1. **The roofline asymmetry between prefill and decode.** Prefill (processing the prompt) is a large matrix-matrix multiply — compute-bound. Decode (generating each output token) is a matrix-vector multiply — memory-bandwidth-bound. These are fundamentally different bottlenecks on the same GPU, and collocating them forces you to optimize for both simultaneously, which means optimizing for neither.

2. **Goodput as a first-class serving metric.** Unlike throughput (requests/second, ignores latency) or raw latency (ignores utilization), goodput measures *how many requests per second can you satisfy while keeping TTFT and TPOT within SLO bounds*. It's the metric that forces you to reason about throughput and latency jointly, and it's what disaggregation actually optimizes.

3. **Disaggregation as the general pattern for heterogeneous bottleneck workloads.** When two phases of a pipeline have orthogonal resource bottlenecks, routing them to the same hardware pool causes mutual interference and prevents independent scaling. Separating them onto different hardware eliminates interference and unlocks per-phase optimization — a pattern that generalizes far beyond LLM serving.

## Prerequisites

- **Transformer KV cache**: During autoregressive generation, attention keys and values from all previous tokens are cached. This cache grows linearly with sequence length and must be loaded from GPU memory every decode step.
- **The roofline model**: A GPU operation is *compute-bound* if it's limited by FLOPS, *memory-bandwidth-bound* if it's limited by how fast data can move from DRAM to compute units. The transition point (the "ridge") is the arithmetic intensity (FLOPS per byte loaded).
- **Tensor parallelism (TP)**: Splitting a single operation (e.g., a matrix multiply) across multiple GPUs — increases compute throughput for a single request at the cost of all-reduce communication.
- **Pipeline parallelism (PP)**: Assigning different layers to different GPUs — increases the number of requests that can be in flight simultaneously.

## The core idea

Every LLM inference request passes through two distinct phases:

**Prefill**: The system processes the entire input prompt (say, 1,000 tokens) in a single forward pass. This is a batched matrix-matrix multiply: all query/key/value projections operate on a `[batch × seq_len, hidden_dim]` matrix. With realistic batch sizes and prompt lengths, arithmetic intensity is high — well above the GPU's ridge point. The GPU is compute-bound; adding more FLOPS helps.

**Decode**: The system generates one output token at a time. At each step it runs a forward pass for a *single new token*, loading the full model weight matrix just to produce one output vector. Arithmetic intensity collapses: loading 40 GB of weights to do a matrix-vector multiply for one token means the GPU spends most of its time waiting for memory. The GPU is memory-bandwidth-bound; adding more FLOPS doesn't help.

In every existing serving system circa 2023 (vLLM, TensorRT-LLM, FasterTransformer), prefill and decode requests were batched together on the same pool of GPUs. The result is **prefill-decode interference**: when a large prefill request enters a batch, it monopolizes the GPU for tens to hundreds of milliseconds. Decode requests in that batch must wait, causing TPOT (time per output token) to spike. Conversely, if the scheduler prioritizes decode to protect ITL, TTFT (time to first token) grows unbounded for new requests. There is no single batch composition that satisfies both SLOs.

DistServe's insight: **assign prefill and decode to separate GPU pools** and transfer the KV cache between them. Each pool now sees a homogeneous workload and can be tuned independently.

## Mechanics

### Instance topology

A DistServe cluster is split into two logical fleets:

- **Prefill instances**: each holds a full model replica. A new request arrives, all input tokens are processed in one forward pass, and the resulting KV cache (keys and values for every layer, for every input token) is produced.
- **Decode instances**: also hold a full model replica. They receive the KV cache from a prefill instance and run the autoregressive loop, generating output tokens one step at a time.

A central scheduler maintains two request queues — one per fleet — and routes incoming requests to the first available prefill instance.

### KV cache transfer

After prefill completes, the KV cache must move from the prefill GPU to the decode GPU. The size scales as:

```
KV cache bytes = 2 × num_layers × num_heads × head_dim × seq_len × element_size
```

For a 70B model with 2,048-token input using FP16: roughly 2 × 80 layers × 8 heads × 128 dim × 2,048 tokens × 2 bytes ≈ **850 MB**.

On NVLink (intra-node, 600 GB/s peak) this transfer takes ~1.4 ms. A single decode step at batch size 32 on an A100 takes ~10–20 ms. The transfer overhead is therefore well under one decode step and can be overlapped with queuing on the decode side.

On inter-node InfiniBand (800 Gbps), a 850 MB transfer takes ~8.5 ms — still comparable to a few decode steps. Placement matters: DistServe's placement algorithm tries to keep prefill and decode instances that serve the same requests on nodes with the highest available bandwidth.

### Independent parallelism strategies

Because the two phases have different bottlenecks, they benefit from different parallelism strategies:

| Phase | Bottleneck | Best parallelism | Why |
|-------|-----------|-----------------|-----|
| Prefill | Compute (FLOPS) | Tensor parallelism (TP) | Splits matrix multiplies across GPUs, reducing per-step latency |
| Decode | Memory bandwidth | Pipeline parallelism (PP) | Maximizes batch size per GPU, amortizing weight loads over more requests |

With colocation, you're forced to pick a single parallelism configuration for both. DistServe optimizes each independently — for example, TP=4 for prefill and PP=2 for decode on the same 8-GPU node.

### Goodput-driven resource allocation

Goodput is defined as the number of requests per second that complete within SLO bounds (e.g., TTFT < 500 ms, TPOT < 50 ms). DistServe uses a solver that, given a target goodput, finds the optimal prefill:decode GPU ratio and parallelism config by simulating queuing delay under realistic workload distributions.

A typical finding: code completion (long prompts, short outputs) benefits from 2:1 prefill-to-decode allocation; chat (short prompts, long outputs) shifts toward 1:2. The ratio is a tunable knob, not a fixed architecture constant.

### Production scheduling nuances

From 18 months of production experience: pure disaggregation requires **KV-cache-aware routing** on the decode side. If a user sends multiple turns of a conversation, the decode instance that already holds the KV cache from the previous turn should preferably handle the next turn (avoiding a re-prefill). Frameworks like vLLM's disaggregated mode and NVIDIA Dynamo address this via prefix-hash-based routing on the decode fleet.

## Where it breaks

**Throughput is not improved.** This is the most important non-intuitive result the retrospective emphasizes: disaggregation does *not* increase total tokens per second for a fixed GPU budget. The gains are purely in latency predictability and SLO compliance. If your only goal is maximum throughput and you have no latency SLO, colocation remains more hardware-efficient.

**Network is a hard dependency.** If prefill and decode instances are on separate nodes connected only by Ethernet (25–100 Gbps), KV cache transfer overhead becomes multi-second for long contexts. Disaggregation requires high-speed interconnects (NVLink intra-node or InfiniBand inter-node). This is a capital cost that not every deployment can absorb.

**Memory amplification.** While the KV cache is in-flight (between prefill completion and decode availability), it occupies GPU memory on both sides simultaneously. Peak memory usage is higher than with colocation.

**Operational complexity.** You now have two independently-scaling fleets, two queuing systems, a KV cache transfer layer, and a placement optimizer. The blast radius of a scheduling bug doubles.

**Long-context workloads stress the transfer.** A 128k-token context KV cache for a 70B model exceeds 50 GB. Even at NVLink speeds, transferring 50 GB takes ~83 ms, which is now material relative to decode step latency. Context-parallel prefill (splitting the prompt across multiple prefill GPUs) is the emerging mitigation.

## Why it works

The deeper principle is **roofline-driven specialization**: when two workloads operate on opposite sides of the ridge point on the same hardware, neither achieves its maximum throughput. The hardware can't be both maximally-compute-utilized and maximally-bandwidth-utilized at the same time. Forcing them to share a GPU means whichever phase is "wrong" for the current hardware state causes the other to stall or underutilize.

This is exactly the same reason we separate OLTP and OLAP databases: OLTP is random I/O (latency-bound), OLAP is sequential scan (bandwidth-bound). Colocation on the same storage engine forces compromises that make both worse. The solution — in both cases — is to recognize the orthogonal bottlenecks and route each workload to hardware optimized for it.

Other instances of the same pattern:
- **Thread pool design**: separate CPU-bound and I/O-bound thread pools in an application server. Mixing them causes I/O-blocked threads to consume CPU slots, starving compute.
- **Storage tiering**: hot random-access data on NVMe SSD, cold sequential-scan data on HDD — same logic.
- **Network control vs. data plane**: control-plane packets (small, bursty, latency-sensitive) vs. data-plane forwarding (large, throughput-oriented). Processed on different hardware (see also the Netflix Lightbulb post in this archive).

The generalization: whenever a pipeline has two stages with measurably different resource profiles (compute intensity, memory access pattern, parallelism structure), ask whether colocation is causing the "wrong" stage to set the pace. If so, disaggregate and connect with a fast transfer channel.

## Going deeper

1. **Original DistServe paper (OSDI 2024)**: https://arxiv.org/abs/2401.09670 — the full technical treatment with formal SLO models, the placement algorithm derivation, and complete evaluation. Sections 3 and 4 cover the resource profile analysis and system design in detail.

2. **NVIDIA Dynamo announcement**: https://developer.nvidia.com/blog/introducing-nvidia-dynamo-a-low-latency-distributed-inference-framework-for-scaling-reasoning-ai-models/ — the production-grade open-source implementation with context-parallel prefill and NIXL (NVIDIA Inference Transfer Library) for high-speed KV migration across nodes.

3. **Sarathi-Serve (OSDI 2024)**: an alternative approach that keeps prefill and decode colocated but uses *chunked prefill* — splitting long prompts into sub-chunks that interleave with decode steps. Provides some of the TTFT improvement without disaggregation overhead. Understanding both papers clarifies the design space.

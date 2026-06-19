---
title: "Bringing HPC Techniques to Deep Learning: Ring AllReduce"
source: https://andrew.gibiansky.com/blog/machine-learning/baidu-allreduce/
author: Andrew Gibiansky
company: Baidu Research (context)
date_posted: 2017-02-21
date_digested: 2026-06-19
---

# Bringing HPC Techniques to Deep Learning: Ring AllReduce

## What's new to learn

- **Ring AllReduce**: A collective communication algorithm that averages gradient tensors across N workers using O(G) bytes of bandwidth per node regardless of cluster size — the same throughput as a single data transfer, no matter how many workers you add.
- **Bandwidth-optimality**: A formal lower bound on how much data each node must receive in any all-reduce algorithm (exactly G bytes for a G-byte gradient), and why ring allreduce hits within a factor of 2 of that bound.
- **The COST of a parameter server**: Why a centralized gradient aggregation server has O(N·G) bandwidth on the master node, turning the cheapest node into the bottleneck as you scale — and why this is not a tuning problem but a topology problem.

## Prerequisites

- Data-parallel distributed training: each GPU runs the same model on a different mini-batch, computes local gradients, and needs a globally averaged gradient before the optimizer step.
- What a gradient is: a vector of partial derivatives, typically one float per parameter — for modern LLMs, hundreds of GB of floats.
- Basic familiarity with collective operations: all-reduce = every participant ends with the sum/average of all participants' inputs.

## The core idea

In data-parallel training, N workers must exchange and reduce their gradient vectors after every forward-backward pass. The naive approach — a central parameter server — creates a bottleneck: the server's network link must handle N incoming gradient writes and N outgoing parameter broadcasts. Double the workers, double the server's traffic. The server becomes the ceiling.

Ring AllReduce eliminates the central node entirely. Workers are arranged in a logical ring; each worker only ever talks to its two immediate neighbors. The gradient vector is cut into N equal chunks. The algorithm runs in two phases:

**Phase 1 — Scatter-Reduce** (N−1 rounds): Each worker sends its current chunk to the right neighbor and receives the previous chunk from the left neighbor, accumulating the sum. After N−1 rounds, each worker holds the fully-reduced (summed) version of exactly one chunk — its "home" chunk.

**Phase 2 — All-Gather** (N−1 rounds): Each worker sends its fully-reduced home chunk to the right neighbor and receives the next chunk from the left. After N−1 more rounds, every worker has received every chunk and thus holds the complete, fully-reduced gradient vector.

The result: every worker ends up with the gradient average, and no single node handled more traffic than any other.

## Mechanics

**Setup**: N workers, each holding a gradient vector of G bytes. Partition the vector into N chunks of G/N bytes each. Worker j owns chunks indexed `[0, N)`, starting at its "home" chunk `j`.

**Phase 1 — Scatter-Reduce**:

```
for round in range(N-1):
    send_chunk_index = (my_rank - round) % N
    recv_chunk_index = (my_rank - round - 1) % N
    send chunk[send_chunk_index] to right_neighbor
    accumulate received data into chunk[recv_chunk_index]
```

After N−1 rounds, worker `j` has:
```
chunk[j] = sum(worker_0.chunk[j], worker_1.chunk[j], ..., worker_{N-1}.chunk[j])
```
— a fully reduced partial result, covering a disjoint slice of the gradient.

**Phase 2 — All-Gather**:

```
for round in range(N-1):
    send_chunk_index = (my_rank - round + 1) % N
    recv_chunk_index = (my_rank - round) % N
    send chunk[send_chunk_index] to right_neighbor
    overwrite chunk[recv_chunk_index] with received data
```

After N−1 rounds, every worker has the fully-reduced version of all N chunks.

**Bandwidth accounting**:
- Each worker sends N−1 chunks of size G/N per phase → (N−1)·G/N ≈ G bytes per phase.
- Two phases → ~2G bytes sent per worker, regardless of N.
- Compare to **parameter server**: server receives N·G bytes and sends N·G bytes → bandwidth at server = 2N·G. Doubles with every new worker.
- Compare to **binary tree reduce**: O(G log N) per node. Better than parameter server but still grows with N.
- Ring AllReduce reaches the **lower bound**: any algorithm requires each node to receive at least G bytes (the full averaged gradient). Ring AllReduce uses 2G, within a factor of 2 of optimal.

**Latency**: Ring AllReduce takes 2(N−1) rounds of one chunk each. For large clusters or very small gradients, latency (not bandwidth) dominates. Recursive halving-doubling algorithms achieve O(log N) rounds at the same 2G bandwidth, trading algorithmic complexity for fewer round trips.

**Modern stack**: In practice, NCCL (NVIDIA Collective Communications Library) implements ring allreduce and is the backend for PyTorch `DistributedDataParallel` (DDP). NCCL adds two refinements:
1. **Double-buffering**: one chunk is being sent while the previous received chunk is being accumulated — overlapping compute and communication.
2. **Hierarchical allreduce**: within a machine, NVLink bandwidth is ~600 GB/s; across machines, InfiniBand is ~100 Gbps. NCCL runs a ring within each node (fast), then a ring across nodes (slow) with only node-reduced chunks — reducing inter-node traffic by N_local× compared to a flat ring.

## Where it breaks

**Latency-bound small models**: For models with few parameters (e.g., a 10 MB embedding table), the round-trip latency of 2(N−1) messages across InfiniBand (~1–2 µs each) dominates over the bandwidth savings. Recursive halving-doubling (log N rounds) wins here. Ring allreduce's constant bandwidth advantage only pays off when the gradient tensor is large enough that transmission time exceeds latency overhead.

**Gradient compression breaks the algorithm**: Ring AllReduce operates on exact float32/float16 sums. Error-feedback gradient compression (Top-K sparsification, PowerSGD) changes the mathematical structure — compressed gradients aren't additively decomposable the way ring allreduce requires. These methods need custom distributed protocols.

**Memory isn't helped at all**: Ring AllReduce doesn't reduce per-GPU memory. Every worker still stores a full copy of gradients, optimizer states, and model weights. This is precisely the gap that ZeRO fills: ZeRO stages 1–3 partition those tensors across workers, while still using (a modified) allreduce for communication. Ring AllReduce provides bandwidth efficiency; ZeRO provides memory efficiency; they are orthogonal optimizations layered on top of each other.

**Topology mismatch**: A logical ring may not match physical network topology. Modern GPU clusters have a fat-tree InfiniBand topology; a ring that routes across different spine switches wastes links. Production systems (NCCL, MSCCL) use topology-aware scheduling to keep ring segments within the same network stratum.

## Why it works

Ring AllReduce is an instance of **pipeline parallelism applied to collective communication**. The gradient vector is treated as N independent work items (chunks), and N workers form a pipeline where each stage processes one item per clock (round). This is identical to:

- **Instruction-level pipelining in CPUs**: fetch → decode → execute → writeback stages overlap in time, achieving one instruction per clock in the steady state.
- **Systolic arrays for matrix multiplication**: data flows through a grid of PEs, each doing one multiply-accumulate per cycle — bandwidth-optimal for GEMM.
- **Double-buffered DMA**: while one buffer is being consumed, the next is already being filled — bandwidth saturated, latency hidden.

In all of these, the key insight is: **break a large sequential operation into N equal work items and pipeline them through N stages**. Throughput becomes bandwidth-limited (each stage is always busy) rather than latency-limited (no stage waits for all others). The ring topology enforces one peer per step, which is exactly the pipelining constraint.

The deeper principle: **collective communication is a scheduling problem**. The parameter server schedules all communication through one bottleneck; the ring distributes the schedule evenly. This is the same insight behind consistent hashing (uniform load distribution), SFQ (fair queuing via hash-bucketed round robin), and work stealing — replacing a single hot node with a distributed schedule.

## Going deeper

1. **NCCL documentation and source code** — NVIDIA's open-source collective communications library; the actual implementation engineers interact with. See especially the ring algorithm implementation and topology-aware scheduling: https://github.com/NVIDIA/nccl
2. **ZeRO: Memory Optimizations Toward Training Trillion Parameter Models** (Rajbhandari et al., 2020, Microsoft DeepSpeed) — covers how ring allreduce is modified when optimizer states are partitioned across workers (the AllReduce becomes a Reduce-Scatter + AllGather, which is structurally identical to ring allreduce Phase 1 + Phase 2, just with different owners per chunk).
3. **"Optimization of Collective Communication Operations in MPICH"** (Thakur, Rabenseifner, Gropp, 2005, IJHPCA) — the academic origin of the ring scatter-reduce/all-gather formulation; explains the space of allreduce algorithms (recursive doubling, recursive halving-doubling, ring, pairwise exchange) and when each wins.

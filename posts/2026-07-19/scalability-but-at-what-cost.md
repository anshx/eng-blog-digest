---
title: "Scalability! But at What COST?"
source: https://www.usenix.org/conference/hotos15/workshop-program/presentation/mcsherry
author: Frank McSherry, Michael Isard, Derek G. Murray
company: Microsoft Research
date_posted: 2015-06-15
date_digested: 2026-07-19
---

# Scalability! But at What COST?

## What's new to learn

1. **COST (Configuration that Outperforms a Single Thread)**: A metric defined per system for a problem — the number of machines or cores a distributed system needs before it beats a well-written single-threaded implementation. A system with infinite COST is worse than a laptop at every scale.

2. **Hilbert curve edge ordering**: Interleaving the bits of a graph edge's source and destination vertex IDs places nearby edges together along a space-filling curve, giving cache locality in *both* endpoints simultaneously — unlike CSR format, which only provides locality in source vertices. On memory-bound graph algorithms, this layout difference is worth a 10x+ speedup.

3. **Parallelizable overhead vs. true parallelism**: When distributed systems compare themselves only against each other, they measure how efficiently they divide overhead, not whether that overhead was necessary in the first place. A system can "scale to 512 cores" and still lose to one Rust binary running on your laptop.

## Prerequisites

- Basic parallel computing concepts: weak vs. strong scaling, worker coordination, network shuffle
- Graph algorithm fundamentals: PageRank (iterative matrix-vector product), connected components (union-find)
- Awareness that DRAM random access (~100 ns, ~100M words/sec) is dramatically slower than L2/L3 cache access (~5 ns), and that this gap dominates memory-bound algorithms
- Familiarity with CSR (Compressed Sparse Row) as a standard graph storage format

## The core idea

The standard way distributed graph processing systems demonstrated value was by comparing themselves to each other: system A showed "3x speedup over Hadoop on 32 nodes" while system B showed "10x speedup over Spark on 64 nodes." The implicit baseline was other distributed systems, never a competent sequential program.

McSherry et al. wrote single-threaded Rust implementations of two graph algorithms — PageRank and graph connectivity — totaling around 100 lines of code. They ran these on a laptop against two massive public graphs (Twitter follower graph: 1.47B edges; UK web crawl: 3.7B edges). Then they compared against published benchmark numbers from the leading distributed graph processing systems of the day: Spark, GraphLab, GraphX, PowerGraph, and Naiad.

The results were stark. For graph connectivity on the Twitter graph, the laptop single-thread finished in **15 seconds**. GraphLab running on **128 cores** took **242 seconds** — 16x slower than one machine.

For PageRank on the same graph, the laptop (with Hilbert curve ordering) finished in **110 seconds**. Naiad needed at least 16 cores to beat that. GraphLab needed 512 cores. GraphX never beat it at any reported configuration.

This is what COST makes precise: the hardware you must throw at a distributed system before the distribution starts helping at all, relative to the optimum you could achieve without it.

## Mechanics

### Datasets and benchmarks

Two graphs serve as ground truth throughout the paper:

- **twitter_rv**: 1,468,365,182 directed edges (about 1.47 billion), ~60 million vertices, Twitter's public follower graph
- **uk-2007-05**: 3,738,733,648 edges, UK web crawl from 2007

Two algorithms:
- **PageRank** (20 iterations): Each iteration iterates over every edge and accumulates source rank into destination rank accumulators. Memory-bound: with 60M vertices, the per-vertex state is gigabytes; updating a destination vertex is a random DRAM access.
- **Graph connectivity** (union-find): Find all connected components via path compression + union by rank. Also memory-bound on large graphs.

### The Hilbert curve layout

A naive sequential implementation stores edges in CSR order: all outgoing edges from vertex `v` are contiguous. When computing PageRank, you read source ranks sequentially (great cache behavior) but write to destination accumulators randomly (cache-thrashing for graphs that don't fit in L3).

The key insight is that a **Hilbert curve** is a space-filling curve — a 1D order over a 2D grid that preserves 2D locality. For a grid where one axis is the source vertex and the other is the destination vertex, the Hilbert index of edge `(u, v)` is computed by interleaving the bits of `u` and `v`. Sorting edges by this index means:

- Edges with close Hilbert indices touch nearby source vertices *and* nearby destination vertices.
- Both the reads (source rank) and writes (destination accumulator) now hit nearby memory.
- For a 40 GB working set (60M vertices × rank values), this converts random DRAM access into mostly sequential access, unlocking full memory bandwidth.

The concrete representation splits each edge into a 32-bit upper part (coarse cell on the Hilbert curve) and two 16-bit offsets (fine position within cell), stored in separate files. The upper file determines which 64K×64K square you're in; the lower file gives the exact position. The loop over a square's edges touches a contiguous 64K-entry source array and a 64K-entry destination array — both fit in L3 cache.

With this layout, RAM-based PageRank on twitter_rv completes in **110 seconds** on a single thread, beating multi-core clusters running less cache-aware implementations.

### The COST table

The paper presents a table of COST values for each system × algorithm combination. Some values:

| System | PageRank (twitter_rv) | Connected Components (twitter_rv) |
|---|---|---|
| Naiad | 16 cores | ≤ 1 core (competitive) |
| GraphLab | 512 cores | > 128 cores |
| GraphX | ∞ (never) | ∞ (never) |
| PowerGraph | high | high |

The range from Naiad (COST 16) to GraphX (COST ∞) reflects engineering quality, not just algorithmic choice. Naiad was purpose-built for low-overhead dataflow; GraphX ran on the JVM via Spark, adding serialization, GC pauses, and coordination overhead that compounded at every scale.

### Why distributed systems pay so much overhead

Four overhead sources compound in distributed graph processing:

1. **Serialization**: Every edge update that crosses a machine boundary must be serialized (and deserialized). At 1.47B edges per PageRank iteration, even 10 nanoseconds per edge totals 15 seconds of pure serialization overhead.

2. **Network communication**: Even with 40 GbE, sending billions of small per-vertex messages per iteration saturates the interconnect. Graph algorithms' irregular access patterns don't batch well.

3. **Load imbalance**: Graphs follow power-law degree distributions. High-degree vertices (celebrities on Twitter) receive updates from millions of sources; the worker assigned that vertex becomes a straggler on every iteration.

4. **JVM GC pauses**: Systems built on Spark or Hadoop inherit JVM garbage collection. GC pauses force synchronization barriers to wait, turning occasional 200 ms pauses into full-iteration slowdowns.

None of these overheads disappear with more machines. Serialization cost grows with partition count. Coordination cost grows with cluster size. A system that scales from 32 to 128 cores with 3x speedup looks good internally, but if it started 20x slower than a laptop, 3x improvement still leaves it 6x behind.

## Where it breaks

**Dataset doesn't fit on one machine.** The entire argument assumes the problem fits in RAM. For the twitter_rv graph (about 12 GB of edges + 3 GB of vertex state), a 64 GB laptop handles it fine. For truly web-scale graphs (trillions of edges), COST flips: the single-thread baseline is ∞ because it simply can't run. Distribution is necessary, not a choice.

**Real-time latency requirements.** A 110-second offline PageRank run is fine for batch analytics but useless for serving a social network recommendation in 50 ms. Distributed systems pre-shard and pre-compute so that individual queries are fast; a sequential scan cannot match this.

**Many concurrent small queries.** The COST critique is sharpest for batch analytics where you run one big computation. For OLTP — thousands of concurrent point queries against a large dataset — adding shards genuinely helps by partitioning the key space.

**"Competent" is load-bearing.** COST is defined against a "competent" single-threaded implementation. The Hilbert curve ordering took domain expertise. Most engineers writing a sequential PageRank would use CSR order, which loses to GraphLab at a few dozen cores. The baseline requires expertise to construct, limiting its use as an everyday measurement tool.

**Hardware has changed.** The 2015 laptop had a 48 GB RAM ceiling. Modern workstations have 1-4 TB of RAM. Problems that required distribution in 2015 may now fit on one machine. The direction of the argument has strengthened over time, not weakened.

## Why it works

The paper is an application of a single principle: **the correct baseline is the best feasible alternative, not the least-bad alternative you compared against.**

This principle recurs throughout computing:

- **Bélády's algorithm** is the optimal cache replacement policy. A cache with 95% hit rate sounds impressive until you learn the optimal offline policy achieves 99%. COST-style reasoning says: before claiming your LRU variant is good, measure against Bélády.
- **Shannon entropy** is the information-theoretic compression floor. Any lossless compression scheme's quality should be measured against the entropy of the source, not against gzip.
- **Amdahl's Law** generalizes the idea: the maximum parallel speedup is bounded by the serial fraction. The COST paper reveals a more subtle version — when the "serial fraction" equivalent is *per-operation overhead* that is parallelizable in the sense that each worker pays it, adding workers doesn't help because each new worker also pays the overhead tax.

The reason the *specific result* is so extreme (laptop beats 128 machines) is the intersection of two known facts:

1. **Memory bandwidth, not compute, is the bottleneck.** PageRank does one floating-point multiply-add per edge, but accesses memory randomly. Modern CPUs can do 10 billion multiply-adds per second but only ~100 million random DRAM accesses per second. Adding more CPUs does not add more DRAM bandwidth to the same memory (or adds network bandwidth as a new bottleneck instead).

2. **Distributed systems convert random DRAM into random network.** Instead of randomly accessing local DRAM for destination vertex state, each worker sends a message across the network for every cross-partition edge. This is equally random, but slower. The Hilbert curve layout sidesteps both, restructuring the computation into sequential memory access — which scales to full DRAM throughput regardless of core count.

The "oh, so X is just an instance of Y" insight here: apparent scalability = real parallelism + parallelizable overhead. When overhead dominates real work, a cluster "scales" efficiently while remaining slower than a single thread. COST is the instrument that separates these two components.

## Going deeper

1. **McSherry's follow-up: "Bigger data; same laptop"** — extends the analysis to the uk-2007-05 graph (3.7B edges), where disk-based single-threaded implementations using SSD still compete favorably with distributed systems. The frankmcsherry/COST GitHub repository contains the Rust code; reading it alongside the paper is instructive.

2. **"Making Caches Work for Graph Analytics" (Beamer et al., 2015)** — a detailed empirical study of cache behavior in graph algorithms, with profiling breakdowns showing how cache miss rates at each level determine runtime. Provides the hardware evidence that explains *why* the Hilbert curve trick works so well. Available at arXiv:1608.01362.

3. **Timely Dataflow / Differential Dataflow** (McSherry's subsequent work) — the same authors built Naiad's successor with explicit attention to avoiding the overheads identified in the COST paper. The design choices (shared memory within a worker, minimal per-record allocation, batched communication) are direct responses to the overhead taxonomy above. The Differential Dataflow paper (already in this archive) shows the payoff.

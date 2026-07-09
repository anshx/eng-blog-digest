---
title: "Morsel-Driven Parallelism: A NUMA-Aware Query Evaluation Framework for the Many-Core Age"
source: https://dl.acm.org/doi/10.1145/2588555.2610507
author: Viktor Leis, Peter Boncz, Alfons Kemper, Thomas Neumann
company: TU Munich / CWI Amsterdam
date_posted: June 2014
date_digested: 2026-07-09
---

# Morsel-Driven Parallelism: A NUMA-Aware Query Evaluation Framework for the Many-Core Age

## What's new to learn

1. **Morsels as the unit of parallel work**: A morsel is a fixed horizontal slice of a relation (~100K rows). Treating morsels as tasks — rather than assigning whole partitions statically at plan time — is what allows elastic, balanced parallelism.

2. **Work-stealing dispatch applied to databases**: A central dispatcher hands morsels to idle worker threads on demand, automatically balancing load across cores without knowing anything about data distribution at plan time.

3. **NUMA as an explicit scheduling constraint**: The dispatcher tracks which NUMA socket holds each morsel and prefers to schedule it on a thread pinned to that socket, converting a hardware topology fact into a scheduling preference that cuts cross-socket memory traffic roughly in half.

## Prerequisites

- **Volcano iterator model**: Pull-based query execution where each operator exposes `next()`, and a parent calls `child.next()` one tuple at a time. Covered in the MonetDB/X100 post.
- **Hash join basics**: Two phases — build (scan the smaller table into a hash table), then probe (scan the larger table, look up each row in the hash table).
- **NUMA hardware**: Multi-socket servers where each CPU socket has local DRAM. Accessing RAM on a remote socket adds ~100 ns of latency compared to ~50 ns for local. Any dual-socket server is NUMA.
- **Work-stealing schedulers (helpful, not required)**: The idea that in Cilk or Intel TBB, idle worker threads steal tasks from busy workers' queues, achieving near-perfect load balance without central coordination.

## The core idea

Classical parallel query execution grafts parallelism into the query plan at compile time by adding **exchange operators** — Volcano's "parallel push." An exchange operator partitions a relation horizontally into exactly `P` fragments and dedicates one thread to each. The degree of parallelism (DOP) is frozen before execution starts.

This model has three compounding problems:

1. **Static load imbalance.** If partition 3 happens to have twice as many qualifying rows as the others due to data skew, its thread runs twice as long while the other P−1 threads sit idle.

2. **NUMA blindness.** Partitions are assigned to CPUs without regard to where the data physically lives on the memory bus. A thread on socket 0 may repeatedly fetch pages from socket 1's DRAM at 2× the latency.

3. **Rigid elasticity.** When a second query arrives, you cannot dynamically scale down the first query's DOP to free resources. The plan hardcoded its thread count.

Morsel-driven parallelism fixes all three with one change of abstraction:

> Instead of *assigning entire partitions to threads at plan time*, divide each relation into small, fixed-size **morsels** of ~100K rows and maintain a global queue of pending **(pipeline, morsel)** jobs. Worker threads pull jobs from the queue; a central **dispatcher** chooses which thread gets which morsel.

The dispatcher is NUMA-aware: it tracks the socket on which each morsel's data resides and prefers to assign that morsel to a worker thread pinned to that socket. Because jobs are small and granular, even skewed data self-corrects: a heavy-hitter morsel just takes a little longer, but every other worker is busy on another morsel rather than sitting idle.

Parallelism becomes *elastic*: the dispatcher can increase or reduce the number of workers assigned to a query at any moment — something a hardcoded DOP plan cannot do.

## Mechanics

### Plan analysis: pipelines and pipeline breakers

Before execution the system makes one pass over the operator tree to identify **pipelines**: maximal sequences of pipelineable operators (scan, filter, projection, hash-probe) separated by **pipeline breakers** — operators that must consume all their input before producing any output, such as the build phase of a hash join, sorting, or a blocking aggregation.

Example query:

```sql
SELECT c.name, SUM(l.quantity)
FROM lineitem l
JOIN customer c ON l.custkey = c.custkey
GROUP BY c.name
```

Yields:

| Phase | Pipeline | Operators | Breaker |
|-------|----------|-----------|---------|
| 0 | Build | scan `customer` → hash-build | creates `HT_c` |
| 1 | Probe + partial agg | scan `lineitem` → hash-probe `HT_c` → thread-local partial GROUP BY | — |
| 2 | Merge | merge thread-local aggregates per key range | output |

Phase 0 must complete before phase 1 begins; phase 1 before phase 2. The dispatcher enforces these precedence edges by simply not releasing phase-N+1 jobs until a completion counter for phase N reaches zero.

### The dispatcher

The dispatcher is a lightweight bookkeeping thread — not on the critical path. Workers don't block waiting for it; they spin on a per-NUMA-socket lock-free job queue. The dispatcher's loop is:

```
while jobs remain:
    pick the next ready morsel job
    if an idle worker on job's NUMA socket exists:
        assign there          # prefer local
    else:
        assign to any idle worker   # cross socket rather than sit idle
    record assignment
```

Worker threads execute the full pipeline for their morsel and immediately return to the dispatcher for the next one. This is the classic **work-stealing scheduler contract**, just with a central dispatcher instead of per-worker queues.

### Parallel hash join

**Build phase (phase 0)**: Parallel and granular.

1. `customer` is partitioned into K morsels (each ~100K rows, located on a NUMA socket).
2. Workers concurrently build their morsel into a **partitioned hash table** `HT_c`, whose buckets are divided into B stripes. Worker on socket 0 writes only to stripe range [0, B/2); worker on socket 1 writes only to [B/2, B). No row-level locking — each stripe is owned by one socket's workers.
3. The dispatcher watches the build completion counter. When the last build morsel completes, probe morsels are released.

**Probe phase (phase 1)**: Each morsel of `lineitem` goes through the full pipeline atomically on one thread: scan 100K rows, probe each against `HT_c`, compute partial GROUP BY into a thread-local hash map. No shared mutable state during this phase.

### Parallel aggregation

Each worker during phase 1 accumulates a **thread-local partial aggregate map** `{name → partial SUM}`. After all probe morsels complete, phase 2 launches:

- The key space is range-partitioned among workers.
- Each worker merges only the portion of the key space it "owns" across all thread-local maps.
- The final aggregated results are ready as soon as each worker writes its partition of the output.

No worker ever holds a global lock; the only coordination is the phase completion counter.

### Sort

Sort is the hardest pipeline breaker. The morsel approach:

1. Each worker independently sorts its morsel (using in-place pdqsort or introsort) — no synchronization.
2. After all morsels are locally sorted, a **parallel k-way merge** produces the final output. Workers own disjoint key-range partitions of the output, so they merge from the thread-local sorted runs in parallel with no locks.

For very large sorts, a third step (two-level merging) avoids creating a merge fan-in proportional to the thread count.

### NUMA-aware memory allocation

The hash table built in phase 0 must be probed in phase 1. If probe morsels on socket 1 must reach across to socket 0's DRAM for `HT_c` every tuple, latency doubles. The system allocates each stripe of `HT_c` on the NUMA socket where its assigned probe morsels will run, using `numa_alloc_onnode()`. Similarly, intermediate results are allocated on the socket that will consume them next. On a 4-socket machine, this reduces remote memory accesses by ~50%.

### Elastic DOP

Because the dispatcher controls morsel assignment at runtime, adjusting DOP costs just a policy change in the dispatcher:

- **New query arrives**: reduce the morsel-assignment rate for existing queries (let their workers finish the current morsel and then don't hand them more).
- **Query finishes**: workers become available for any pending query immediately.

No query plan needs to be recompiled or replanned.

## Where it breaks

**Pipeline breaker serialization.** No matter how many cores build `HT_c` in parallel, zero probe morsels can start until the last build morsel finishes. For extremely wide build-side relations, this serial rendezvous sets a hard latency floor.

**Single-morsel tables.** A relation with 80K rows fits in one morsel. The entire operation runs single-threaded, regardless of available cores.

**Severe data skew within a morsel.** If 90% of the keys in one morsel map to the same hash bucket during aggregation, that morsel takes 10× longer than average with no load-balancing remedy — you can't subdivide a morsel further.

**Memory bandwidth saturation.** On scans of very wide columnar tables, all threads compete for DRAM bandwidth from the same socket. Adding cores past the bandwidth ceiling yields zero speedup; more threads just increase contention.

**OLTP transactions.** A transaction touching 10 rows has nothing to morsel-ize. Morsel-driven parallelism only pays off when the working set is large enough to produce many morsels (millions of rows).

**Dispatcher as bottleneck at extreme thread counts.** The paper's experiments run up to 32 cores. At 256+ cores (rare in 2014, common now), dispatcher throughput may become a bottleneck if morsel assignment rate can't keep up.

## Why it works

### Work-stealing is Cilk for databases

Morsel-driven parallelism is **Cilk's work-stealing scheduler** applied to relational query execution. In Cilk, tasks are function calls; workers steal from each other's task queues. In morsel-driven execution, tasks are `(pipeline, morsel)` pairs; a central dispatcher hands them to idle workers. The structural contract is identical:

> *Any computation decomposable into independent, similarly-sized tasks achieves near-optimal parallel speedup via work-stealing.*

The three necessary conditions, and why queries satisfy them:

1. **Independence**: No data dependency between concurrent morsel tasks (morsels are disjoint rows). ✓
2. **Uniform task size**: Each morsel is ~100K rows, so tasks take similar wall-clock time. ✓
3. **Enough tasks**: A 10M-row table becomes 100 morsels — far more than 32 cores. ✓

The 100K-row morsel size is calibrated so that at typical analytic query throughput (~5 billion row-operations/sec per core), each task takes ~20 ms. Dispatcher overhead (<1 µs) is then negligible, and 100 tasks per 32 cores leaves enough slack to absorb the longest-tail morsel.

### Brent's theorem: theoretical bound

Brent's theorem from parallel computing states: on a P-worker greedy scheduler, any computation with total work T₁ and critical-path length T∞ runs in time at most T₁/P + T∞.

For a morsel-driven hash join on a table with N rows, P cores, and build-phase cost C_build:

- T₁ = N × (cost per row through the pipeline)
- T∞ ≈ C_build + (N/P) × (cost per row in probe) ≈ dominated by the serial build rendezvous for many cores

The dispatcher is greedy (any ready morsel is assigned to an idle worker immediately), so the actual runtime achieves this bound. The speedup limit is set by T∞ — the critical path, not by how many morsels exist.

### NUMA as an explicit memory tier

Classical databases manage the disk↔buffer-pool hierarchy explicitly. Morsel-driven execution makes the **NUMA socket** explicit as a scheduling constraint, treating it as an "L4 cache" with 100 ns latency instead of nanoseconds (L1–L3) or microseconds (NVMe). This is the same principle as cache-oblivious algorithms and BLAS tiling: make the memory hierarchy explicit in the algorithm, match data access to the fastest available memory tier.

### The deeper pattern: separate task definition from task assignment

The exchange-operator model collapses these two concerns into the query plan: the plan simultaneously *defines* the tasks (partitions) and *assigns* them (partition i → thread i). Morsel-driven separation — define tasks lazily at runtime, assign dynamically — is the exact insight behind every high-performance task-parallel runtime: Cilk, Intel TBB, Go's goroutine scheduler, Rust's Tokio. The pattern recurs because static assignment always overfits the assignment to the conditions at plan/compile time, while dynamic assignment adapts to the actual runtime.

## Going deeper

1. **"Umbra: A Disk-Based System with In-Memory Performance"** — Neumann & Freitag, VLDB 2020. How the TUM group (same team as this paper) evolved morsel-driven execution into a disk-backed system with pipelined sort-merge joins and adaptive radix sort, keeping OLAP performance at in-memory speed even when data spills to NVMe.

2. **DuckDB source: `src/parallel/task_scheduler.cpp`** and `src/execution/pipeline.cpp` — the open-source implementation of morsel-driven execution. `Pipeline::Execute()` maps to the "pipeline job" concept; `TaskScheduler` maps to the dispatcher. Readable without deep C++ expertise and shows exactly how the paper's ideas translate to ~3,000 lines of production code.

3. **"Scheduling Multithreaded Computations by Work Stealing"** — Blumofe & Leiserson, JACM 1999. The formal proof that work-stealing achieves Brent's bound in expectation. If you want to understand *why* morsel dispatch is near-optimal (not just that it is), this is the mathematical foundation.

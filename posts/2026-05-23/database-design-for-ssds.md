---
title: "What Does a Database for SSDs Look Like?"
source: https://brooker.co.za/blog/2025/12/15/database-for-ssd.html
author: Marc Brooker
company: AWS
date_posted: 2025-12-15
date_digested: 2026-05-23
---

# What Does a Database for SSDs Look Like?

## What's new to learn

1. **Sequential-vs-random I/O gap as the original design constraint.** Almost every "obvious" piece of database internals — WALs, multi-megabyte buffer pools, large page sizes, sequential-scan preference — exists to paper over the 1000× latency penalty of random I/O on spinning disks. That constraint no longer holds on NVMe SSDs.

2. **The 32 KB SSD transfer sweet spot.** NVMe SSDs transition from IOPS-bound to throughput-bound at roughly 32 KB per request. This single hardware number cascades into concrete recommendations for page sizes and I/O coalescing that differ significantly from Postgres's 8 KB and InnoDB's 16 KB defaults.

3. **Distributed log as a replacement for local WAL.** Committing to a multi-AZ distributed log instead of a local write-ahead log simultaneously upgrades durability (multi-machine fault tolerance vs. single-node crash survival) and eliminates the need to size write paths around sequential-write optimization — the distributed log is the more capable primitive.

## Prerequisites

- Basic familiarity with how a relational database persists data: pages, buffer pool, the idea of a write-ahead log, and crash recovery.
- What B-trees and LSM-trees are (not their internals, just that B-trees favor reads and LSM-trees favor writes by buffering in memory).
- A rough sense of what IOPS means and why spinning disk seek time (5–15 ms) differs from SSD latency (~100 µs).

## The core idea

Traditional database engines were co-designed with spinning magnetic disks in the 1990s and early 2000s. Spinning disks have two dominant performance characteristics: (1) sequential reads and writes are one to two orders of magnitude faster than random access because the disk head has to physically seek to a different track, and (2) even a fast disk can only serve ~150–200 random I/Os per second because each seek takes 5–15 ms.

These two constraints jointly drove almost every architectural decision:

- **WAL (Write-Ahead Log):** All writes — inserts, updates, deletes — get appended sequentially to the WAL before touching the data pages. Sequential writes are fast; random writes are not. The WAL converts the random write pattern of a B-tree into a sequential one, then data pages are flushed lazily in the background.
- **Large buffer pool:** If every cache miss costs 10 ms, you need a pool large enough to make cache misses rare. A 95% hit rate on a 10 ms disk is still 500 µs average latency; you need a 99.9% hit rate to get below 10 µs. Buffer pools on production OLTP servers grew to hold gigabytes — often most of available RAM.
- **Large page sizes:** Reading 8 KB vs. 4 KB incurs the same seek overhead; might as well get more data per seek. B-tree leaf nodes are one page; fat pages mean shorter trees and fewer seeks per lookup.
- **Sequential scans over index lookups:** For large table scans, a full sequential read is faster than many random index lookups. Query planners often prefer sequential scans at surprisingly small selectivity thresholds because of the disk seek cost.

A modern NVMe SSD breaks every one of these assumptions:

| Metric | Spinning HDD | NVMe SSD |
|---|---|---|
| Sequential read | ~150 MB/s | ~7 GB/s |
| Random read latency | 5–15 ms | ~100 µs |
| Random IOPS | ~200 | ~1,000,000 |
| Sequential/random ratio | ~1000× | ~2–5× |

The gap between sequential and random I/O has collapsed from 1000× to less than 10×. The architectural consequences are dramatic.

## Mechanics

### Rethinking the WAL

On SSDs, a random write and a sequential write land within 2–5× of each other. The WAL's original purpose — converting random to sequential — is far less valuable. But the *form* of the WAL matters too: a local WAL protects against single-node crashes. It says nothing about the machine failing entirely, the storage device dying, or AZ-level failures.

Brooker argues the right primitive is a **distributed log** — a replicated, multi-AZ append-only log that multiple compute nodes can write to and replay from. This is the model used in Aurora DSQL, Spanner, and similar systems: the transaction commits when a quorum of the distributed log acknowledges it. Any replica can replay the log and serve reads.

The upgrade is twofold:
- *Durability*: Multi-machine, multi-AZ vs. single-node crash survival.
- *Simplicity of recovery*: Crash recovery is "replay the distributed log onto a fresh replica" — no local checkpoint files, no ARIES-style redo/undo passes. Any peer replica can be rebuilt from the log.

### Page size: the 32 KB sweet spot

NVMe SSDs have a transfer-size curve that resembles network interfaces: there is an IOPS ceiling and a throughput ceiling, and which one you hit depends on transfer size.

- **Below ~32 KB**: The device is IOPS-bound. Each transfer costs ~100 µs regardless of size. You want large transfers to amortize that per-I/O overhead.
- **Above ~32 KB**: The device is throughput-bound (~7 GB/s). Larger transfers don't improve latency; they just block the interface longer and hurt cache efficiency through false sharing (loading a large page to touch one row evicts other useful pages).

This puts the sweet spot at roughly **32 KB**. Postgres's 8 KB pages are IOPS-limited (each read wastes the remaining transfer bandwidth). InnoDB's 16 KB pages are better but still below the knee. A database designed for SSDs would target 32 KB pages.

For network-attached storage (e.g., cloud block volumes), the sweet spot shifts down to ~8 KB because network round-trip dominates and the bandwidth ceiling is lower — the same analysis applied to different hardware parameters.

### Right-sizing the buffer pool

On spinning disk, the buffer pool had to achieve a ~99.9% hit rate to get sub-millisecond average latency because a miss cost 10 ms. On SSDs, a miss costs ~100 µs — roughly what a round-trip to DRAM takes in a NUMA system. The break-even hit rate drops dramatically.

Brooker's recommendation: **size the buffer pool for 30 seconds to 5 minutes of the hot working set**, not for the entire dataset. Cache the frequently-accessed pages, tolerate occasional SSD reads. This frees RAM for other uses and avoids the pathology of giant buffer pools that warm slowly after a restart and are mostly cold pages.

The exact sizing depends on SSD IOPS budget vs. query latency SLA. If you have 500 K IOPS available and your P99 latency target is 1 ms, you can afford ~500 misses per second; size the cache so the miss rate stays below that budget.

### What to keep

Brooker explicitly keeps the entire relational stack: relational model, SQL, atomicity, isolation, strong consistency, and interactive transactions. These are not artifacts of disk hardware — they are the semantic contracts that make databases useful. The hardware is the implementation detail; the semantics are the product.

### Data structure choices: it's complicated

The B-tree vs. LSM-tree tradeoff shifts on SSDs but doesn't resolve cleanly:

- **B-trees** do random writes (in-place updates). On HDDs this was expensive; on SSDs it's much cheaper. However, B-tree writes still need careful concurrency control and can cause write amplification at both the database level (WAL + modified pages) and the SSD level (erase-before-write in NAND flash).
- **LSM-trees** convert random writes to sequential by buffering in memory and merging in the background. The sequential-write advantage is largely gone on SSDs, but LSM-trees still offer excellent write throughput for high-cardinality workloads and good space amplification control through compaction.

Brooker declines to pick a winner — the right choice still depends on workload read/write mix and access patterns. The important shift is that the HDD-era "B-tree needs WAL to avoid killing write performance" assumption relaxes, so the design space opens up.

## Where it breaks

**SSDs still have internal write amplification.** NAND flash must erase a large block before rewriting even a single byte. The SSD firmware handles this via garbage collection and wear leveling, but every logical write can trigger multiple physical writes internally. Write-heavy workloads still benefit from write-coalescing strategies like LSM compaction; the discipline is just at the SSD-firmware boundary rather than the OS-sequential-I/O boundary.

**NVMe is still ~1000× slower than DRAM.** At 100 µs, SSDs are fast enough to tolerate smaller caches, but they are nowhere near main memory (~100 ns). For in-memory workloads, this analysis is irrelevant; for datasets that truly fit in RAM (many OLTP transactional systems), SSDs are only in the fault-tolerance path.

**Distributed log adds network latency to the write path.** If the distributed log is cross-AZ, each commit incurs a network round-trip (~1–5 ms). This is actually *higher* latency than a local `fsync` to NVMe (~100 µs). You trade write latency for multi-AZ durability; whether that's the right tradeoff depends on your requirements. Aurora DSQL mitigates this by committing to a log within the same AZ first and replicating asynchronously for the additional replicas.

**SSD pricing per GB.** SSDs are still 5–10× more expensive per GB than HDDs for large data tiers. The analysis applies fully to the performance tier; for cold storage (terabytes of rarely-accessed data), the cost argument pushes back toward HDD or object storage.

**Not all SSDs are equal.** Consumer NVMe SSDs have different durability characteristics (TBW ratings) and latency profiles than enterprise NVMe. A database designed for enterprise NVMe (e.g., AWS io2) may not perform as expected on a laptop's consumer SSD with its lower sustained write endurance.

## Why it works

The deeper pattern here is: **hardware constraints are the hidden variable in every architecture document**. When hardware improves by 1000×, the software built to work around its limitations should be systematically re-examined.

The sequential/random I/O gap is the canonical example, but the same reasoning applies across systems:

- **Network reliability improved** → TCP's retransmit-on-loss became a bottleneck for latency-sensitive apps → QUIC, HTTP/2 with independent stream retransmit were designed instead of just tuning TCP.
- **Network bandwidth improved** → RPC started beating batch data-movement patterns; the "reduce data movement" constraint relaxed enough that iterative algorithms on remote data became practical.
- **DRAM capacity grew** → In-memory databases (Redis, VoltDB) became viable where before you had to assume data wouldn't fit.
- **CPU core counts grew** → The single-threaded programming model hit a wall; concurrency had to become first-class.

In each case, the correct move is: *identify which hardware constraint the design was compensating for, measure whether that constraint still holds, and redesign the component if it doesn't.*

For databases, the audit looks like:
1. WAL → compensates for random-write penalty → SSD shrinks that penalty → consider distributed log for better semantics.
2. Large buffer pool → compensates for slow random reads → SSD makes random reads cheap → size cache for working set, not for hit rate maximization.
3. Large page size → compensates for fixed seek cost → SSD has no seek → tune to the transfer-size knee.
4. Sequential scan preference → compensates for seek cost of many random reads → SSD makes random reads cheap → the planner's selectivity thresholds should shift.

This is also an instance of **"question the axioms" thinking**: every design choice was an answer to a question at a point in time. Write down the question, then check whether the question still applies.

## Going deeper

1. **Marc Brooker's Aurora DSQL series** (brooker.co.za, Dec 2024 – Apr 2025): The distributed log and distributed transaction model that this post references is exactly what Aurora DSQL implements. The "Decomposing Aurora DSQL" and "DSQL Vignette: Transactions and Durability" posts show the live system. Especially relevant: how physical clocks let you read from any replica without coordination.

2. **"Closing the B-tree vs. LSM-tree Write Amplification Gap on Modern Storage Hardware with Built-in Transparent Compression"** (VLDB 2021, Hao et al.): An academic analysis of how SSD hardware compression changes the B-tree/LSM tradeoff, with concrete measurements. Directly extends the data-structure discussion Brooker leaves open.

3. **"The Design and Implementation of Modern Column-Oriented Database Systems"** (Abadi, Boncz, Harizopoulos, 2013 – still the best reference): Explains how the columnar storage model was itself a response to hardware — specifically, how column groups exploit DRAM bus bandwidth and CPU SIMD differently than row-oriented storage. A complement to this post's SSD-focused analysis: both are examples of co-designing storage with the hardware's dominant bandwidth bottleneck.

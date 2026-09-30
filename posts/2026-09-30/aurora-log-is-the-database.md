---
title: "Amazon Aurora: Design Considerations for High Throughput Cloud-Native Relational Databases"
source: https://dl.acm.org/doi/10.1145/3035918.3056101
author: Alexandre Verbitski, Anurag Gupta, Debanjan Saha, Murali Brahmadesam, et al.
company: Amazon Web Services
date_posted: 2017-05-14
date_digested: 2026-09-30
---

# Amazon Aurora: Design Considerations for High Throughput Cloud-Native Relational Databases

## What's new to learn

1. **"The log is the database"** — In a distributed storage system, you can ship only redo log records (what changed, ~10–100 bytes) rather than modified data pages (8 KB), reducing write traffic across the network by 7.7×. The storage tier reconstructs pages lazily from the log.

2. **Active storage nodes** — Making storage nodes compute participants rather than passive block devices — they apply redo logs, gossip with peers to fill sequence gaps, and coalesce records into pages — moves work out of the write critical path into the background, decoupling write latency from storage materialization latency.

3. **Segmentation as a MTTR dial** — Partitioning a database volume into fixed 10 GB segments caps the blast radius of any single failure: you repair 10 GB, not an entire database volume, reducing Mean Time to Repair from hours to minutes and making the durability guarantee easier to reason about than an all-or-nothing MTTF calculation.

## Prerequisites

- **WAL (Write-Ahead Logging)**: before changing a page in-memory, write a redo log record describing the change to durable storage. On crash, replay the log to rebuild committed changes.
- **Data page**: the fixed-size unit (8 KB in InnoDB) used by the storage engine to read/write data. WAL records describe deltas to pages.
- **ARIES crash recovery**: after a crash, replay the redo log from the last checkpoint forward, then undo any transactions that were uncommitted at crash time.
- **Quorum replication**: if you have N replicas, write to W and read from R. When W + R > N, at least one reader always sees the latest write.

## The core idea

Traditional MySQL on top of Amazon EBS (Aurora's predecessor setup) suffered from write amplification. A single committed transaction generated writes for: the redo log, the binary log, the modified data pages themselves, a double-write buffer (to detect half-written pages on crash), and metadata files like FRM. Multiply this over a 6-replica EBS setup and you get roughly 7.4 I/Os per transaction.

Aurora's diagnosis: **network I/O is the new bottleneck**, not compute or disk. And the key observation is that of all these write paths, only the redo log is strictly necessary. Everything else is either:
- Redundant given the redo log (data pages can be reconstructed by applying redo records)
- Needed only for MySQL's specific durability model (double-write buffer, which becomes unnecessary if writes are atomically durable at the storage level)
- Derivable from redo (the binary log can be materialized on demand)

So Aurora ships *only* redo log records across the network to its distributed storage tier. The result: **0.95 I/Os per transaction** — a 7.7× reduction. The storage tier is responsible for constructing page images from redo records, asynchronously and lazily.

The slogan the Aurora team arrived at: *The log is the database.*

## Mechanics

### Protection Groups and Quorum

Aurora's data volume is partitioned into 10 GB fixed-size segments. Each segment belongs to a **Protection Group (PG)** of exactly 6 replicas, spread across 3 Availability Zones — 2 nodes per AZ. Every write goes to all 6 nodes, but is considered durable once **4 out of 6** acknowledge (the write quorum Vw = 4). Reads use Vr = 3.

Since Vw + Vr = 7 > 6, a read quorum always overlaps with a write quorum — you always find the most recent data.

This specific quorum (4 writes, 3 reads, 2-per-AZ layout) was chosen to survive the worst plausible correlated failure: the loss of an entire AZ (2 nodes), plus one additional independent node failure, without data loss. Specifically:

- **AZ failure + 1 node loss = 3 nodes remaining** → still meets Vr = 3, so reads succeed
- **AZ failure + 2 nodes lost = 2 nodes remaining** → still has 2 of 6 replicas; no new writes accepted, but data is not lost

### Storage Node Processing Pipeline

When a storage node receives a redo log record from the database:

1. **Receive and enqueue** — add to an in-memory queue
2. **Persist and acknowledge** — write to local disk, send acknowledgment to the DB instance ← *this is the only foreground step*
3. [Background] **Sort and organize** — group records by log sequence number (LSN) and segment
4. [Background] **Detect gaps** — identify holes in the LSN space (records that haven't arrived yet)
5. [Background] **Gossip with peers** — request missing log records from the other 5 nodes in the PG to fill gaps
6. [Background] **Coalesce** — apply accumulated redo records to older page images to materialize new page versions
7. [Background] **Garbage collect** — discard page versions that no active database snapshot needs

The write latency as seen by the database is determined only by steps 1–2: receive and acknowledge. Everything else is deferred. This is why the "active storage" insight matters — the storage nodes do significant work, but they decouple that work from the write critical path.

### Crash Recovery Without Redo Replay

In traditional MySQL, restarting after a crash requires replaying the redo log from the last checkpoint forward — potentially many minutes for a busy database.

In Aurora, the database compute layer doesn't hold the canonical version of data; the storage tier does. On restart:

1. The database queries storage to determine the **Volume Complete LSN (VCL)** — the highest LSN for which all prior log records are present across a write quorum of segments
2. Any log records beyond the VCL are truncated (they weren't durably committed)
3. The database instance begins accepting new connections

No redo replay. No checkpoint-driven recovery. Restarts complete in seconds.

This works because the storage nodes have already been applying redo records in the background continuously. When the database comes back up, the pages are already up-to-date. The database just needs to agree on where the log ends.

### Eliminating 2-Phase Commit

Conventional multi-replica databases use 2-phase commit (2PC) to coordinate write durability — prepare, then commit, with coordinator overhead on every write.

Aurora eliminates 2PC for writes entirely. The 4/6 quorum write *is* the commit. The redo records are small enough to be acknowledged quickly by 4 storage nodes without coordination overhead. Each storage node independently acknowledges based on its local write; the database waits for 4 responses and then considers the write durable — no coordinator, no second round trip.

## Where it breaks

**Read amplification on cold or heavily-written pages.** When a page hasn't been recently materialized and many redo records have accumulated since the last checkpoint, a read requires the storage node to apply all those records before returning the page. Under heavy write workloads, first reads of infrequently accessed pages can see high latency.

**Undo log stays on the compute tier.** Aurora ships only redo log to storage. Undo log — needed to roll back uncommitted transactions and to serve MVCC snapshots — remains on the database compute node. This means compute nodes are not truly stateless; they hold transactional undo state that would be lost on a hard failure. (Aurora handles this by writing undo records into the redo stream so they can be recovered, but the complexity is real.)

**Single-writer architecture.** The original Aurora design has exactly one write master. A single source of truth for the redo log makes "the log is the database" clean — there's one authoritative sequence of changes. Aurora Multi-Master complicates this significantly because you need to resolve write-write conflicts that are invisible to each writer's local redo stream.

**Not a speedup for reads.** Aurora's innovation targets write I/O over the network. Read I/Os still fetch full 8 KB pages. Read-heavy, write-light workloads see little benefit over replicated EBS.

**Gossip latency on storage-node recovery.** When a storage node falls behind (rebooting, delayed), it must gossip with peers to fill the LSN gap before it can rejoin the quorum. The time to catch up is proportional to how far behind it fell. If many nodes fall behind simultaneously — perhaps due to a shared network partition — catchup time can create a write stall.

## Why it works

The deeper principle: **don't replicate state — replicate operations.**

In event sourcing, the append-only event log is the system of record. The current application state is a derived projection of events. This gives you compact writes (events are semantically minimal), free audit history, and the ability to rebuild state by replaying the log.

Aurora applies this pattern to storage replication:
- **Redo log records = events** (minimal causal descriptions of what changed)
- **Data pages = projections** (materialized views of accumulated events applied to a base image)
- **Storage nodes = projection builders** (apply events lazily on demand)

What makes this work is that the redo log is *already computed* — ARIES mandates writing it before modifying pages. Aurora just redirects that pre-computed log: instead of writing it locally and shipping pages, ship the log and build pages remotely. The "cost" is already paid; you're just choosing where to deliver the pre-computed artifact.

The ARIES insight (1992) was: "the log is ground truth, pages are cache." Aurora's insight is: "since the log is ground truth, ship it — not the pages."

This same operation-over-state pattern appears throughout distributed systems:

| System | Operations (the log) | State (the projection) |
|---|---|---|
| Aurora | Redo log records | Data pages |
| Kafka | Append-only log segments | Consumer materialized views |
| Git | Commits (tree diffs) | Working tree checkout |
| CDC pipelines | Binlog / WAL records | Downstream database tables |
| CRDT operation-based | Operation messages | Replica current value |
| vLLM KV cache eviction | Eviction decisions | GPU memory state |

In each case: the operation is small, causal, and immutable; the state is large, derived, and reproducible from the operation log. Replicating operations instead of state gives you network efficiency, reproducibility, and the ability to replay from any point.

## Going deeper

1. **"Amazon Aurora: On Avoiding Distributed Consensus for I/Os, Commits, and Membership Changes"** (SIGMOD 2018) — the follow-up paper covering Aurora's hardest problem: how to handle storage membership changes (nodes joining/leaving) without distributed consensus. It introduces epoch-based leases and explains how the quorum read/write protocol handles concurrent membership transitions. The 2017 paper explains steady-state behavior; this one handles the edge cases that make the design production-safe. https://dl.acm.org/doi/10.1145/3183713.3196937

2. **"ARIES: A Transaction Recovery Method Supporting Fine-Granularity Locking and Partial Rollbacks Using Write-Ahead Logging"** (Mohan et al., ACM TODS 1992) — the foundational paper establishing that redo logs are ground truth and pages are derivable cache. Aurora is best understood as "take ARIES seriously and apply it to the network boundary." Understanding ARIES makes Aurora's key simplification feel inevitable rather than clever. https://dl.acm.org/doi/10.1145/128765.128770

3. **"Designing Data-Intensive Applications"** by Martin Kleppmann, Chapters 5 and 11 — for the full context of "the log is the database" as a distributed systems design principle: Chapter 5 covers replication models; Chapter 11 extends the idea to stream processing and CDC, showing that Aurora's storage insight is the same pattern that motivates Kafka, event sourcing, and the Lambda/Kappa architectures. https://dataintensive.net/
